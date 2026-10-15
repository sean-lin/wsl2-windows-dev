## 1. Interop 执行模型

WSL2 通过 `binfmt_misc` 注册 Windows PE 执行器：bash 里运行 `.exe` 由 Windows 侧进程加载器启动，文件系统经 9P 桥接。

### 1.1 Windows 进程眼中的 WSL2 路径

从 WSL2 ext4 启动的 Windows EXE，其 CWD 是 UNC 形式的 9P 路径（`\\wsl.localhost\<发行版>\home\<user>\...`），而非 Linux 路径——Windows 进程模型不认识 Linux 路径。影响：

- `GetModuleFileNameA` / `GetCurrentDirectoryA` 返回 UNC 路径；
- 基于 CWD 的相对路径解析都在 UNC 命名空间进行；
- 路径解析要同时容忍 `\` 与 `/`（Windows API 返回 `\`，WSL2 侧传入的常是 `/`）：
  ```cpp
  size_t pos = full.find_last_of("\\/");
  ```

### 1.2 从 WSL2 调用 cmd.exe / powershell.exe

cmd.exe 继承 UNC 形式的 CWD 时会打印警告（`CMD.EXE 不支持 UNC 路径作为当前目录，默认值为 Windows 目录`）——不影响执行，但干扰输出解析。

**内联 `set` 不可靠（换引号层级救不了）**：`cmd.exe /d /c 'cd /d D:\x && set "VAR=value" && tool.exe'` 这类内联命令，变量**静默没设上**——工具以默认值运行、无任何报错（实测：`go mod download` 无视刚设置的 GOPROXY 仍去拨默认代理，直到超时才发现）。三种写法实测（2026-09-27）：外层单引号+内层双引号 ✗、外层双引号+`\"` 转义 ✗、完全无引号 ✓（但无引号只能承载不含空格的值）。**要设环境变量就写进 `.bat` 文件执行**（文件内引号不受 interop 影响），或用工具自身的持久配置（`go env -w`、`git config` 等）。

同类：把复杂 PowerShell 内联进 `cmd.exe /d /c "powershell.exe -Command \"...\""` 时，bash 与 cmd 两层引号互相吞掉，报 `无效查询`、`Cannot find module 'D:\...\"..."'` 之类怪错。**一律落成 `.ps1` 文件，用 `powershell.exe -NoProfile -ExecutionPolicy Bypass -File script.ps1` 执行**，参数以 `-Name value` 追加。`-ExecutionPolicy Bypass` 不可省：Windows 客户端默认 Restricted 策略，缺它时 `.ps1` 文件被直接拒绝执行（`-Command` 内联不受执行策略限制，这也是内联看似能跑、落文件后反而报拒的原因之一）。

**给 `.cmd`/`.bat` 传参不要经 `cmd.exe /d /c` 拼接（含引号时引号字面化）**：把参数引号拼进字符串再交给 `cmd.exe /d /c`，引号会**原样字面化**传进批处理——实测 `wechatide.cmd "-c" Qoder ...` → 批处理收到带引号的 `"-c"` → 报 `unknown tool: "\"-c\""`。无内层引号的整串调用（`cmd.exe /d /c "tool.cmd -a b --flag c"`，不能承载含空格参数）可用。**要传含空格/任意参数：绕过 cmd，直接 exec 目标 `.exe` 并以独立 argv 传参**（interop 对 argv 做标准转义）；若官方入口是 .cmd，把它的内部逻辑（环境变量、bootstrap 参数）照抄进 shim 脚本，行为保持一致。

### 1.3 PowerShell 脚本：退出码与编码

**退出码 = 最后一条语句的 `$?`**，与脚本中间是否发生非终止错误无关。`Get-Process -ErrorAction SilentlyContinue` 未命中会把 `$?` 置为 false；若随后的 `if` 没有 `else`，脚本以退出码 1 结束，宿主误报 "PowerShell command failed"。规则：**所有分支都要以成功语句收尾**——查询类补 `else { "" }`，整体以显式 `exit 0` 兜底；真实失败路径靠 `-ErrorAction Stop` 保留非零退出码（用于区分"操作失败"与"预期内未命中"）。

**编码**：WSL2 侧写出的 `.ps1` 默认 UTF-8 无 BOM，而 PowerShell 5.1 对无 BOM 脚本按 ANSI code page（中文系统 = GBK）解析，中文字面量变乱码，**解析期**报 `MissingPropertyName` / `UnexpectedToken`（报错位置常落在被吞掉的引号附近，易误判为逻辑错误）。规避：脚本源码保持**纯 ASCII**，或存为 **UTF-8 with BOM**。

**`.vbs` 同病且无 BOM 解法**（2026-10-03 实测）：WSH 恒按 ANSI 解析 `.vbs`，UTF-8 中文注释的字节被 GBK 误码后**可吞掉行分隔符**——注释吞并下一行代码，报错却是"对象未定义"（如 `缺少对象: 'sh'`，指向看似正常的 `Set sh = ...` 行，极易误判为逻辑错误）。规避：`.vbs` 一律**纯 ASCII**（含注释）。

**用户目录变量**：从 WSL2 调 cmd.exe 时 `%USERPROFILE%` 可能不展开，且中文用户名经 cmd 往返会以 GBK 字节进入 PowerShell、拼出的路径不存在。PowerShell 内统一用 `$env:USERPROFILE`。

### 1.4 WSLENV 传环境变量；execPath 保留启动路径形式

**WSL 侧环境变量默认不传**：interop 启动的 Windows 进程拿到的是 Windows 侧环境，WSL 侧新设的变量不会自动过去。要传必须用 `WSLENV` 声明（冒号分隔；`VAR/p` 后缀表示同时做路径转换）：

```bash
ELECTRON_RUN_AS_NODE=1 WSLENV=ELECTRON_RUN_AS_NODE "/mnt/c/.../app.exe" -e "console.log(process.env.ELECTRON_RUN_AS_NODE)"
# → 1（实测：Electron 应用以 ELECTRON_RUN_AS_NODE 模式运行）
```

**execPath 保留调用时使用的路径写法（8.3 短名 vs 完整路径）**：interop 启动的 Windows 进程，其 `process.execPath`（及 `GetModuleFileName`）**保留启动路径的原样形式**——用 `/mnt/c/PROGRA~2/...` 启动则得到 `C:\PROGRA~2\...`，用完整路径启动则得到完整形式。若目标程序用 execPath 派生状态目录名（如对路径做哈希），**两种形式会算到不同目录**，症状是"读不到配置 / 服务连不上"这类费解故障。

实测案例（微信开发者工具 CLI）：其 User Data 目录 hash = `md5(dirname(execPath) + "\resources\app.asar")`。用完整路径调用 → 算到旧目录 → 读不到 IDE 写在另一个目录的 `.ide` 端口文件 → 报 `Failed to connect`；换 8.3 短名路径启动即恢复正常。调用方也可用反证法自查：分别以两种路径形式启动，打印 `process.execPath` 对比。

**`Start-Process` 会把 8.3 短名规范化为长名**（2026-09-30 实测）：`Start-Process -FilePath 'C:\PROGRA~2\...\app.exe'` 启动的进程，execPath 是完整长名——按路径形式派生状态目录的程序会写到**另一个**目录（同一 IDE：Start-Process 启动写长名 hash 目录、WSL 直接 exec 写短名 hash 目录）。**需要保留路径形式时不要用 `Start-Process`**（`processes.md` §3.1 的"保活"需求改用 WSL 直接 exec + `setsid nohup`）；反之不在意路径形式、只需保活时 Start-Process 仍是首选。另一条稳妥路径：让程序自带的 CLI/引导流程自己拉起宿主进程（实测 IDE 的 CLI 自动启动落到正确的短名目录）。
