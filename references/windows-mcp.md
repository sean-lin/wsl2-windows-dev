## 10. Windows-MCP（SSE）：Windows 侧操作的首选通道

> **优先级**：凡需要**在 Windows 上执行**的操作——命令、进程、窗口、桌面、注册表、文件——先走本通道。`interop.md` / `processes.md` / `git.md` / `network.md` / `gui-automation.md` 里的 WSL 直调方案，默认视为它不可用时的回退路径。WSL 内部的纯文件操作不受此限。
>
> **边界**：本通道服务**开发机上的桌面 / 系统操作**；被测应用的 UI 自动化仍走项目自己的框架（WinAppDriver / Appium / Insight），两者不要混用。

### 10.1 安装（一次性）

前置：Windows 10/11；用 uv 管理 Python 3.14，不污染系统 Python。uv 用官方 Windows 发行包解压到 NTFS 目录（如 `D:\tools\uv`），确保 `uv.exe` 在 PATH 或用完整路径。

```powershell
# Windows PowerShell 内执行
uv python install 3.14
uv tool install windows-mcp==0.8.8 --python 3.14
```

`auth_key` 可本地生成（不要把值提交或外发）：

```powershell
[guid]::NewGuid().ToString('N')
```

`%USERPROFILE%\.windows-mcp\config.toml`：

```toml
[server]
transport = "sse"
host = "127.0.0.1"
port = 8765
auth_key = "<随机生成，只在本机配置与用户级 MCP 配置里，勿外传>"
```

`%USERPROFILE%\.windows-mcp\start-server.cmd` —— **纯 ASCII**（中文用户名 + UTF-8 脚本会被 cmd 按 GBK 解析而报「找不到指定的路径」）：

```bat
@echo off
setlocal
set "ANONYMIZED_TELEMETRY=false"
cd /d "%USERPROFILE%"
"%APPDATA%\uv\tools\windows-mcp\Scripts\python.exe" -m windows_mcp serve --transport sse --host 127.0.0.1 --port 8765 1>>"%USERPROFILE%\.windows-mcp\server.log" 2>>"%USERPROFILE%\.windows-mcp\server.error.log"
```

注册登录自启（必须交互式会话；真正的 Windows 服务在 session 0，无法做 UI 自动化）：

```powershell
$action  = New-ScheduledTaskAction -Execute "$env:USERPROFILE\.windows-mcp\start-server.cmd"
$trigger = New-ScheduledTaskTrigger -AtLogOn -User $env:USERNAME
Register-ScheduledTask -TaskName "windows-mcp-server" -Action $action -Trigger $trigger `
  -User $env:USERNAME -LogonType Interactive -RunLevel Limited -Force
```

验证：

```bash
cmd.exe /d /c "cd /d %USERPROFILE% && netstat -ano | findstr :8765"   # LISTENING
```

注意：`windows-mcp install --force` 会重新生成 UTF-8、含中文路径的启动脚本，需要改回上面的纯 ASCII 版本再重启任务。

### 10.2 WSL / Qoder 接入

- mirrored 网络下 WSL 直接连 `127.0.0.1:8765`，无需 proxy。
- Qoder（qodercli）用户级 MCP 实际读 `~/.qoder/settings.json` 的 `mcpServers`；`~/.qoder/mcp.json` 当前不被读取（仅可作备份）。

```bash
qodercli mcp add-json windows-mcp '{"type":"sse","url":"http://127.0.0.1:8765/sse","headers":{"Authorization":"Bearer <auth_key>"}}' --scope user
# 然后 /mcp reload
qodercli mcp get windows-mcp     # Status: Connected
```

- 自写客户端：先 `initialize` + `notifications/initialized`，再 `tools/call`；否则报 `-32602`。

### 10.3 按任务选工具

| 任务 | 工具 | 要点 |
|---|---|---|
| 执行 Windows 命令 | `PowerShell` | 一组动作合并成一次调用；默认超时 30s，长任务显式设 `timeout` |
| Win32 / 窗口枚举 / 取矩形 | Python + `ctypes` 或 `pywin32` | 解释器 `%APPDATA%\uv\tools\windows-mcp\Scripts\python.exe`；不再编译 exe |
| 启动 GUI / 常驻进程 | `App`（`launch_executable`） | 秒返，进程独立存活 |
| 移动 / 置前窗口 | `App`（`resize` / `switch`） | `window_loc` + `window_size`；截图核验 |
| 列进程 | `Process`（`list`） | 按名筛选有子串 / 大小写怪癖 |
| 杀进程 | `PowerShell` | `Get-Process -Name X \| Stop-Process -Force` |
| 截图 / 读屏 | `Screenshot` / `Snapshot` | `region` 可只抓一块屏；本机 `Snapshot` 窗口枚举不可靠，以截图 + `Get-Process` 为准 |
| 读写文本 | `FileSystem` | UTF-8 往返已验证 |
| 键鼠 / 剪贴板 | `Click` / `Type` / `Shortcut` / `Clipboard` | 物理注入，注意门控与 z-order |
| 注册表 | `Registry` | 原生读写 |

### 10.4 能力边界（实测）

**解决 / 绕过**：

- interop 引号互吞、环境变量不传递、UNC 工作目录；
- 新 exe 首跑的 interop EINVAL（原生创建进程）；Defender 按**内容信誉**拦截真新二进制仍在，所以优先用 Python 而不是新编译 exe；
- CLM 禁 `Add-Type` / COM → 用 Python（ctypes / pywin32）替代编译探针；
- GUI 启动挂起 / 随 WSL 父进程死；
- 窗口移动 / 置前 / z-order；
- 退出码不可靠（`Status Code` 原样返回）；
- 中文输出（系统切 UTF-8 内码后，见 `SKILL.md` 开工前设置）。

**仍有限制**：

- **桌面门控**：用户 10s 内有键鼠活动时返回 `USER_CONTROL`；把动作合并，等空闲后重试；
- `Process` 按名 kill 不可靠（`notepad` 杀不掉 `Notepad.exe`）；
- `Snapshot` 的窗口枚举在本机返回空（用截图 + PowerShell）；
- 被测应用的 UI 自动化不归它管。

### 10.5 回退

MCP 服务未启动、门控无法满足，或操作本质是 WSL 内部文件处理时，回退到：

- 命令 / 进程：`processes.md` §3.1–§3.5；
- 引号 / env / 脚本编码：`interop.md` §1；
- 构建 / git / 网络 / GUI 自动化：对应分册。
