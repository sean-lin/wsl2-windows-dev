## 2. UNC/9P 路径上的文件与 Win32 行为

### 2.1 Win32 API 差异

| API | 在 UNC/9P 路径上 | 处置 |
|---|---|---|
| `GetPrivateProfileStringA`（INI） | **静默返回空** | 提供基于 `fopen` 的 INI 解析 fallback |
| `GetFullPathNameA` | 只做字符串规范化，**不解析 junction/symlink 目标** | 不能靠 junction 做路径跳转；配置里的相对路径要直指真实目录 |
| `_wfopen` / `fopen` | 完全正常（读/写/建目录） | — |
| `GetModuleFileNameA` | 返回 9P UNC 路径 | 见 `interop.md` §1.1 分隔符处理 |

### 2.2 大批量文件操作走 Windows 侧

WSL2 的 `unzip`、「批量复制」经 9P 往 `/mnt/<盘符>` 写**上万级小文件**时极慢，且实测**可能静默不完整**（命令返回、无报错、缺文件；症状如 `package X is not in std`）。判据：`find <dir> -type f | wc -l` 与预期量级不符。

- 解压改用 Windows 侧：`cmd.exe /d /c "cd /d D:\path && tar -xf dl\pkg.zip"`（Win10 1803+ 自带 bsdtar，支持 zip）；
- 大批量复制/删除同理走 Windows 原生工具（`cmd.exe` / `powershell.exe`）；
- 单文件小量 `cp` 走 `/mnt` 无碍，可直接用。

### 2.3 Junction 与 Symlink

| 特性 | NTFS | ext4 (9P/WSL2) |
|------|------|----------------|
| Junction（`mklink /J`） | 可用 | 不可用（报"需要 NTFS 卷"） |
| symlink（`ln -s`） | Windows 可经 9P 读 | 可用 |
| 硬链接 | 可用 | 可用 |

跨环境部署脚本应备两套并在环境不匹配时报错指路（ext4 侧 `ln -s`、NTFS 侧 `mklink /J`）。

### 2.4 搜索/遍历 NTFS 树：用 Git for Windows 自带的 GNU 工具

WSL2 侧的一切文件工具（`grep -r` / `find` / `rg`）遍历 `/mnt/<盘符>` 都吃 9P 逐文件延迟：目录越深越慢，整树搜索动辄超时（实测同一目录同一关键字：Windows 侧 `grep.exe` ~10s vs WSL `grep` 120s+ 未跑完）。这是文件系统层的问题，「换工具」不解决——除非换成 Windows 原生程序。

**不用装**：Git for Windows 自带整套 GNU 工具，`C:\Program Files\Git\usr\bin\` 下有 `grep.exe` / `find.exe` / `sed.exe` / `awk.exe` / `sort.exe` / `xargs.exe`。从 WSL 直接 exec（**不经 `cmd.exe /c` 包装**——直接 exec 的参数传递更可靠）：

```bash
# 路径参数用 Windows 形式（正斜杠亦可）；输出路径同样是 C:/... 形式
"/mnt/c/Program Files/Git/usr/bin/grep.exe" -rn --include="*.ts" "pattern" "C:/proj/src"
```

判据：扫 `/mnt` 大目录卡住/超时即改用此法；输出是 Windows 路径形式，管道后处理时注意。仓内找代码也可用 Windows `git.exe grep`（快车道，只含 tracked 文件）。

## 5. 路径解析约定

- **相对路径优于绝对路径**：程序配置中的路径相对于 EXE 所在目录，团队保持目录结构一致即可直接使用。
- **绝对路径判断**：`path.size() > 1 && path[1] == ':'` 判 `C:\` 形式；其余按相对路径与程序目录拼接后 `GetFullPathNameA` 规范化。此判断不含 UNC 绝对路径（`\\server\...`），需要时额外检查前两字符。
- **分隔符容忍**：见 `interop.md` §1.1。
