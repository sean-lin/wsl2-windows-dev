## 3. 从 WSL2 调起 Windows 程序

### 3.1 常驻 / GUI / 需交互的程序用 Start-Process

`cmd.exe /c "start \"\" app.exe"` 或管道启动都有坑：实例随命令返回或管道关闭而终止。**需要进程在命令返回后存活时，用 PowerShell `Start-Process`**：

```bash
powershell.exe -NoProfile -Command "Start-Process -FilePath '<dir>\app.exe' -ArgumentList '<arg1>','<arg2>' -WorkingDirectory '<dir>'"
```

典型场景：常驻 GUI（`cmd start` 拉起的实例短暂出现后即消失，命令本身不报错）；控制台服务（避免 stdin 随 WSL2 管道关闭——如 WinAppDriver 打印 `Press ENTER to exit` 后退出）。

启动后用 Windows 侧进程列表（`tasklist` / `Get-Process`）确认存活，并用程序自身日志确认初始化完成。**停止 WSL2 侧的包装命令/后台任务不保证终止已派生的 Windows 进程**——状态确认与停止都以 Windows 侧为准（目标端口或进程列表）。

**后台任务里经 `cmd start` 拉起的 GUI 程序与任务同命——完成通知是迟到记账**（2026-10-03 实测）：一个"超时后被移入后台"、内部 `cmd start` 发起 GUI 程序的任务，会一直显示运行中（实测约 20 分钟）——直到该程序退出，任务才记账完成、才送达完成通知。收到迟到完成通知先核对进程实态（`tasklist` / `netstat`）再下结论，勿当成"有人刚启动/关闭了什么"；启动常驻程序一律 `Start-Process` 脱离式，"超时转后台"不是可依赖的启动姿势。

### 3.2 控制台输出

GUI 子系统程序（WinMain）默认无控制台。`FreeConsole` + `AttachConsole(ATTACH_PARENT_PROCESS)`（失败则 `AllocConsole`）+ `freopen("CONOUT$", "w", stdout/stderr)` 之后：

| 启动场景 | 效果 |
|------|------|
| 原生 Windows 终端 | 输出直接可见 |
| WSL2 bash | 经管道继承，可捕获 |
| Explorer 双击 | AllocConsole 独立窗口 |
| `> file.txt` 重定向 | **不生效**（CONOUT$ 写控制台而非文件） |

**文件日志始终是最可靠的输出方式**（跨所有启动方式），WSL2 侧用 `tail -f` 实时查看。

**捕获输出的编码**：Windows 控制台程序按代码页（中文系统 = GBK/936）输出字节流，落进 WSL 侧管道后与 UTF-8 混排——`grep`/`head` 会把它当二进制（报 `binary file matches` 并吞掉匹配行），加 `-a` 强制按文本处理；要人读干净输出用 `iconv -f GBK -t UTF-8` 转码，或让 bat 开头 `chcp 65001`（对非 cmd 自带工具的输出不总是生效，`iconv` 兜底）。

### 3.3 工具定位：PATH 是 Windows 的，npm 全局 bin 未必在内

从 WSL2 调 Windows 工具时，"在交互式 cmd 里能直接敲命令"不等于"脚本里也能"——看到的 PATH 不同：

- Windows 工具按 **Windows 的 PATH**（系统 + 用户级）解析，WSL2 侧 PATH 里的同名 Linux 工具不参与；
- **npm 全局 bin（`%APPDATA%\npm`）默认未必在 PATH 里**（实测：`where appium` 找不到，而完整路径 `%APPDATA%\npm\appium.cmd` 可用）——npm 安装的全局 CLI 一律按完整路径调用，`%APPDATA%` 在 `.bat` 内展开可靠；
- 同样地，改 PATH 或环境变量后**必须重启调用方进程**（常驻服务如 Appium）才生效——能按完整路径调用时，比改 PATH 更省事。

### 3.4 新编译的 exe 首次执行被云裁定拦截——隔几分钟重试即通

**现象**：刚从 WSL2 编译（或从 WSL2 写入 Windows 盘）的未签名 exe，**首次执行被静默阻断**——interop 直跑报 `Invalid argument`（EINVAL）；经 `cmd.exe` 跑拿到退出码 1（历史上也见过 786）且**无任何输出**（连加载器报错都没有）；WMI `Win32_Process.Create` 发起返回 0（进程创建成功）但进程即刻消失。**同机老文件**（已存在一段时间的工具）不受影响——被拦的是「首次见到的未知二进制」。

**根因（2026-10-03 定案）**：Windows Defender 云保护（Block at first sight——未知文件首见送检）——首次执行触发云端裁定，裁定期间阻止运行；**裁定为 clean 后放行**。实测：被拦的 hello world 约 10 分钟后重试即正常运行、此后稳定；换构建方式、换编译器（Go / .NET / mingw 一样）、换发起者（WMI 直发也一样）均无差别——唯一变量是「文件是新的」。

**处置**：

- **重试即可，不要换构建/执行方式**——写完新工具先跑一次（预期失败，视作"预热送检"），隔几分钟回来再跑同一文件；
- 排障时先区分「新文件被拦」与「程序自身失败」：用同目录已知能跑的老 exe（或复制一份老 exe 改名跑）做对照；注意 `Get-MpThreatDetection` 与 Defender 操作日志**可能都不记录**此拦截，不要因"日志干净"排除此因；
- 需要立即运行时：请人在 Windows 侧交互式启动一次（Explorer 双击有安全交互），或等裁定窗口（通常几分钟）过去。

### 3.5 读另一个进程的控制台文本（AttachConsole）

`-with_console` / 控制台程序打到 stdout / stderr 的内容**只在控制台里**（不在日志文件），从 WSL2 侧想取证时读不到。读取路径（纯 syscall，Go / C 几十行即可）：

`FreeConsole()`（先脱离自己的控制台）→ `AttachConsole(pid)`（附加到目标进程的控制台）→ `CreateFileW("CONOUT$", GENERIC_READ, ...)`（打开其输出缓冲）→ `GetConsoleScreenBufferInfo` 拿缓冲尺寸 → `ReadConsoleOutputCharacterW` 逐行读文本 → `CloseHandle`。注意：`ReadConsoleOutputCharacterW` 返回 UTF-16，转 UTF-8 后落文件再回 WSL 侧读。

用途举例：游戏 `-with_console` 输出里的调试日志 / C++ 断言前的最后几行——常能把「疑似上层工具问题」一眼拨正到「native 层问题」（实测案例见 §9）。参考实现：`D:\linux_tmp\console_dump.go`（约百行，无 cgo；首次执行可能有 Defender 云裁定延迟，见 §3.4）。

## 9. 挂死进程：断言窗、不可杀与 socket 残影（2026-10-10 实证）

**现象**：进程「活着但什么都不响应」——服务不接受连接（服务侧 hello 超时 / 连接被拒）、日志停写、`taskkill /F` 与 `Stop-Process -Force` 杀不动（`taskkill` 报「无法终止……没有此任务的实例在运行」，`Stop-Process` 甚至打印成功但进程仍在）。

**真因之一 = C++ 断言窗**：Debug 构建的 `assert` 失败会弹出 CRT 对话框（"Microsoft Visual C++ Runtime Library — Assertion failed!"）；弹窗不处理，命中线程一直阻塞（弹窗在屏幕上等按键），整个进程表现为冻结、且对 `taskkill` 呈现不可杀态。**第一动作：看屏幕有没有断言 / 对话框弹窗**（无人值守 / 长跑场景最易漏看）；**弹窗上的 Abort 即终止进程**——清理这类「不可杀」实例最可靠的一招。

**诊断读法**：结合控制台文本（§3.5）看弹窗前的最后输出；本次即靠「控制台只剩 CRIWARE 音频 stall 告警、无上层 Lua 错误」+ 屏幕断言窗对上号（断言在游戏侧适配层 dll，与当时排查的注入链无关）。

**socket 残影遮蔽端口**：挂死进程持有的监听端口——其他进程 `bind` **不报错**（netstat 可见双 LISTENING）但连接全被拒 / 超时（残影赢路由）；连独占占位也进不去（EADDRINUSE）。真正释放要等挂死进程死掉（重启机器最稳）；期间新实例换显式端口绕过。

**推论纪律**：纯用户态 / 纯脚本层的组件**构造不出**「杀不动」的进程状态——遇到杀不掉的进程先怀疑 native / 驱动层（断言窗、内核态等待、设备独占），别先怀疑自己的上层工具。
