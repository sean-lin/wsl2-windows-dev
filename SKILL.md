---
name: wsl2-windows-dev
description: 在 WSL2 中开发任意 Windows 项目的通用经验与踩坑指南：开工前系统设置（UTF-8 内码、开发者模式）、首选 Windows-MCP 通道（安装 / 登录自启 / Qoder 接入 / 按任务选工具）、interop 执行模型（引号静默丢失、.ps1 编码、PowerShell 退出码）、UNC/9P 路径上的 Win32 API 行为、从 WSL2 调起常驻/GUI/控制台程序、batch 与 MSBuild 陷阱、Windows 盘仓库的 git 与 .ssh ACL、mirrored 网络的端口与进程身份、WinAppDriver 点击的 z-order 约束、挂死进程的诊断（断言窗 / 控制台抓取 / socket 残影）。凡从 WSL2 构建、部署、运行、自动化 Windows 程序时使用。
---

# WSL2 下开发 Windows 项目通用经验

> 与具体项目无关的模式与陷阱，均为实测结论。项目/本机特有的约定见各项目的 AGENTS.md 与本机 AGENTS.local.md。

正文按域拆入 `references/` 分册（本页留开工前设置、首选通道与速查表）；遇到问题先按「现象」查速查表拿定位，再 Read 对应分册。

## 开工前：一次性系统设置（自己的开发机先做）

> 本 skill 主要给自己用。系统级设置先做对，后面整类问题直接消失；其余条目主要处理「设置正确之外」的坑。

### 1. 系统内码切 UTF-8（强烈建议）

**设置**：设置 → 时间和语言 → 语言和区域 → 管理语言设置 → 更改系统区域设置 → 勾选「Beta: Use Unicode UTF-8 for worldwide language support」→ 重启。区域保持 `zh-CN` 即可，不必改成英文。

**验证**：

```bash
cmd.exe /d /c "cd /d %USERPROFILE% && chcp"                              # Active code page: 65001
powershell.exe -NoProfile -Command "[Console]::OutputEncoding.WebName"  # utf-8
```

注册表 `HKLM\SYSTEM\CurrentControlSet\Control\Nls\CodePage` 的 `ACP` / `OEMCP` 均应为 `65001`。**判据：两项里任一不是 `65001` / `utf-8`，就说明系统还没切，先改再做别的。**

**解决**（均为实测）：PowerShell/CLI 中文输出乱码、UTF-8 无 BOM `.ps1` 解析失败、Python stdout 默认编码、WSL interop 中文往返、Windows 侧 MCP 工具输出乱码。

**不解决**：Defender 首跑拦截、interop 引号丢失、环境变量不传递、UNC 路径、PowerShell CLM——这些仍按下文分册处理。

**风险**：依赖 GBK 的旧程序、GBK 编码的文本配置会反向乱码。自己的开发机接受这个交换；遇到旧工具中文异常，先想到这里，必要时回退该 Beta 选项。

### 2. 开发者模式（做 GUI 自动化前）

设置 → 隐私和安全性 → 开发者模式 → 开。WinAppDriver 依赖它；未开时报 `Developer mode is not enabled`，详见 `references/gui-automation.md` §8。

## 首选通道：Windows-MCP（SSE）

**凡需要在 Windows 上执行的操作——命令、进程、窗口、桌面、注册表、文件——先走 Windows-MCP**；本 skill 各分册里的 WSL 直调方案，默认视为它不可用时的回退路径。WSL 内部的纯文件操作不受此限。

一次性安装（完整脚本与说明见 `references/windows-mcp.md`）：

```powershell
uv python install 3.14
uv tool install windows-mcp==0.8.8 --python 3.14
```

然后配置 `%USERPROFILE%\.windows-mcp\config.toml`（SSE、`127.0.0.1:8765`、`auth_key`），用**纯 ASCII** 的 `start-server.cmd` 启动，并注册登录自启；WSL/Qoder 侧用 `qodercli mcp add-json windows-mcp '<json>' --scope user` 接入，`/mcp reload` 后生效。

**边界**：它服务开发机上的桌面 / 系统操作；被测应用的 UI 自动化仍走项目自己的框架（WinAppDriver / Appium / Insight）。

## 分册地图

- [`references/windows-mcp.md`](references/windows-mcp.md)：**何时读**：要在 Windows 上执行命令、操作进程、移动/置前窗口、操作桌面——这是首选通道。**里面有什么**：§10 安装与登录自启、Qoder/WSL 接入、按任务选工具、能力边界与回退（含桌面门控与实测结论）。
- [`references/interop.md`](references/interop.md)：**何时读**：Windows-MCP 不可用时，从 WSL2 直调 cmd / PowerShell（引号静默丢失、脚本编码、退出码）、给 Windows 进程传环境变量、排查启动路径形式导致的配置/状态目录错位。**里面有什么**：§1 Interop 执行模型（1.1 Windows 进程眼中的 WSL2 路径 / 1.2 cmd 与 PowerShell 调用 / 1.3 退出码与编码 / 1.4 WSLENV 与 execPath）。
- [`references/filesystem.md`](references/filesystem.md)：**何时读**：在 `/mnt` 上做文件操作（解压 / 批量复制 / 搜索遍历）、Win32 API 在 UNC 路径上行为异常、junction / symlink 选择、路径解析约定。**里面有什么**：§2 UNC/9P 路径上的文件与 Win32 行为（2.1 API 差异 / 2.2 大批量文件操作走 Windows 侧 / 2.3 Junction 与 Symlink / 2.4 用 Git for Windows 的 GNU 工具搜 NTFS 树）+ §5 路径解析约定。
- [`references/processes.md`](references/processes.md)：**何时读**：Windows-MCP 不可用时，从 WSL2 启动常驻 / GUI / 控制台程序、捕获其输出、定位 Windows 工具（PATH 差异）、新编译 exe 首次执行被拦、读另一个进程的控制台文本、诊断挂死不可杀进程。**里面有什么**：§3 从 WSL2 调起 Windows 程序（3.1 Start-Process / 3.2 控制台输出 / 3.3 工具定位 / 3.4 Defender 云裁定拦截 / 3.5 AttachConsole 读控制台 / 3.6 首选 MCP 索引）+ §9 挂死进程（断言窗 / socket 残影）。
- [`references/build.md`](references/build.md)：**何时读**：写 `.bat`、从 WSL2 调 MSBuild 构建、构建信号可疑（LNK1168 / up-to-date 仍打印 "OK"）。**里面有什么**：§4 Batch 与 MSBuild 陷阱（4.1 Batch / 4.2 MSBuild）。
- [`references/git.md`](references/git.md)：**何时读**：操作 NTFS 盘上的 git 仓库、git 报 `Bad owner or permissions` 或 `Host key verification failed`。**里面有什么**：§6 Windows 盘仓库用 Windows git（6.1 .ssh 宽 ACL / 6.2 ssh 身份分歧）。
- [`references/network.md`](references/network.md)：**何时读**：跨端端口不通或被遮蔽、管理命令跨端生效、TIME_WAIT 挡重绑、查 Windows 端口清单。**里面有什么**：§7 mirrored 网络：端口与进程身份（7.1 端口遮蔽 / 7.2 释放延迟 / 7.3 跨端 server / 7.4 EXE 副本身份 / 7.5 双物种工具 / 7.6 TIME_WAIT / 7.7 netstat -ano）。
- [`references/gui-automation.md`](references/gui-automation.md)：**何时读**：Windows GUI 自动化（WinAppDriver / Appium）——点击报成功但界面无反应、被遮挡窗口激活、坐标注入与标定、会话与端点坑。**里面有什么**：§8 WinAppDriver 点击的 z-order 约束（含 8.1 坐标注入与被遮挡窗口激活）。

## 常见错误速查

> 下表只列 **Windows-MCP 不可用 / 不适用** 的坑；Windows 侧执行优先走上面的 MCP 通道，分册里的直调方案都是回退路径。

| 现象 | 定位 |
|------|------|
| `GetPrivateProfileStringA` 返回空 | INI 在 UNC 路径上 → filesystem.md §2.1 |
| MSBuild 报"项目文件不存在" | `pushd/popd` 移除了临时盘符 → build.md §4.1 |
| Batch 注释被当命令执行 | `::` + delayedexpansion + code page → build.md §4.1 |
| `mklink /J` 报"需要 NTFS 卷" | 目标在 ext4/9P → filesystem.md §2.3 |
| `/p:` 参数解析失败 | 尾部 `\` 转义了引号 → 双写 `\\` → build.md §4.1 |
| 重建时报 `LNK1168` / "复制了 0 个文件" | 目标 EXE 在运行 → 先 taskkill → build.md §4.2 |
| GUI 程序无输出 | 无控制台 → 用文件日志（processes.md §3.2） |
| IDE（Qoder 等）拉起的 MCP server / 子进程读不到项目配置（.env 等） | 子进程 cwd 是 IDE 安装目录而非项目目录，依赖相对路径找配置静默落空 → 给工具显式传绝对路径（如 --env-file） |
| `.ps1` 报 `MissingPropertyName`/引号附近语法错 | 系统内码是 GBK → 先做开头的一次性设置（切 UTF-8）；未切时纯 ASCII 或 BOM → interop.md §1.3 |
| `.vbs` 报 `缺少对象` 但该行看着正常 | ANSI 解析吞行（UTF-8 中文注释） → 纯 ASCII → interop.md §1.3 |
| `.ps1` 被拒绝执行（execution policy / Restricted） | 加 `-ExecutionPolicy Bypass` 运行 → interop.md §1.2 |
| Windows npm 全局工具"找不到" | 全局 bin 未必在 PATH → 用 `%APPDATA%\npm\<tool>.cmd` 完整路径 → processes.md §3.3 |
| Windows 中文输出乱码 / 无 BOM UTF-8 `.ps1` 解析失败 | 系统内码是 GBK → **先做开头的一次性设置（切 UTF-8）**；不能改系统时命令只输出 ASCII / UTF-8 文件交换 → interop.md §1.3 |
| 解压到 `/mnt` 后工具报缺文件 | 9P 写万级小文件静默不完整 → Windows `tar` → filesystem.md §2.2 |
| `git.exe` 报 `Bad owner or permissions` | `.ssh` 宽 ACL → icacls 收紧 → git.md §6.1 |
| `git.exe` 报 `Host key verification failed` 但裸 `ssh` 能通 | git 用 Git 自带 ssh、吃 WSL 的 HOME 找不到 known_hosts → `-c core.sshCommand=C:/Windows/System32/OpenSSH/ssh.exe` → git.md §6.2 |
| Windows 侧服务"看不见"但端口被占 | WSL2 侧同名端口遮蔽 → 看监听进程 → network.md §7.1 |
| Windows 侧 bind 报 `Only one usage` 但本侧无监听 | WSL2 侧停服后的 TIME_WAIT 未消 → 等 `ss -tan` 清空 → network.md §7.6 |
| PowerShell 查 Listen 端口要 25s+ | `Get-NetTCPConnection` 走 CIM → 换 `netstat -ano` → network.md §7.7 |
| 刚关闭的 loopback 端口仍能拨通 | host 释放延迟 → 判据落库自身状态 → network.md §7.2 |
| `adb devices` 报 `ADB server didn't ACK` | 曾从 WSL 发过 kill-server 杀掉 Windows server → Windows 侧重启 → network.md §7.3 |
| WSL 裸跑工具自动起了本地 server，顶掉/掩蔽 Windows server | 双物种混用 → 默认命令 symlink 到 Windows exe → network.md §7.5 |
| 重解压的 EXE 显示未授权 | 新副本 = 新身份 → 对端重新授权 → network.md §7.4 |
| WAD 点击 success 但界面无反应 | z-order 覆盖 → hit-test 诊断 → gui-automation.md §8 |
| Appium/WAD 起不来（`server process is not running` / `Developer mode is not enabled`） | WAD 不在 PATH / 未开开发者模式（32 位 WAD 读 WOW6432Node 注册表视图） | 开开发者模式；设 `APPIUM_WAD_PATH` 后重启 Appium；仍报则补写 32 位注册表 |
| 从 WSL 连续快拨 Windows 侧单客户端服务被 reset | 转发时序竞速 → 间隔 ≥0.3s 或保持长连接 |
| `taskkill` 报「没有此任务的实例在运行」/ `Stop-Process` 报成功但进程仍在 | 进程被断言窗或内核态等待阻塞 → 看屏幕弹窗（Abort 即终止）→ processes.md §9 |
| 服务进程活着但不接受连接；新进程 bind 同端口不报错却连不通 | 挂死进程 socket 残影遮蔽端口 → 杀挂死进程 / 重启 → processes.md §9 |
| 想读某进程控制台里的输出（不在日志文件） | AttachConsole + ReadConsoleOutputCharacterW → processes.md §3.5 |
| 新编译的 exe 首次执行静默失败（EINVAL / rc=1 无输出） | 优先用 Windows-MCP 原生执行或 Python 脚本；真新二进制仍可能被 Defender 信誉拦截 → processes.md §3.4 |
| C4819 编译警告 | 源文件含非当前 code page 字符 → 不影响功能，可忽略 |
