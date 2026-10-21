## 4. Batch 与 MSBuild 陷阱

> 构建命令优先经 Windows-MCP 的 `PowerShell` 执行（原生 cwd / env / 退出码）；本节问题在两种通道下都可能出现。

### 4.1 Batch

- **`pushd` UNC 的临时盘符映射随 `popd` 消失**：`pushd \\wsl.localhost\...` 会建 `Z:` 映射，`popd` 后路径失效（后续用它找文件会报找不到）。解析脚本自身相对路径用 `for %%I in ("%~dp0..\..") do set "REPO=%%~fI"`，不要 pushd/popd。
- **`::` 注释在 `setlocal enabledelayedexpansion` + 特定 code page（如 936）下可能被当命令执行**：统一用 `REM`。
- **参数值以 `\` 结尾且紧跟 `"` 时**，`\"` 被解释为转义引号（典型：`/p:SolutionDir=...\msvc\`）——**尾部反斜杠双写** `\\` 避免。
- **cmd 里的裸 `bash` 不是 Git Bash**：装了 WSL 的机器上 `where bash` 命中 `C:\Windows\System32\bash.exe`（WSL bash）——批处理里裸调 `bash script.sh` 会掉进 WSL：cwd 经 `/mnt` 映射、用 WSL 侧 git 操作 NTFS 盘，CRLF 脚本还会报 `$'\r'` 类语法错。**要跑 Git Bash 脚本一律显式全路径**：`set "GITBASH=%ProgramFiles%\Git\bin\bash.exe"`，并前置 `set "PATH=%ProgramFiles%\Git\cmd;%PATH%"`（配 `--noprofile --norc` 时不靠 profile 补 PATH；git-for-windows 的 git 经自身 exec-path 找到 ssh，无需 Git\usr\bin）。脚本侧可加 WSL 守卫：`grep -qi microsoft /proc/version` 命中即退出并提示双击 bat。
- **bat 的 CWD 不可依赖**：Explorer 双击时 CWD=bat 所在目录，经 `cmd /c`、计划任务调用时不是——开头一律 `cd /d "%~dp0"`，脚本内相对路径（含传给 bash 的脚本名）才稳定。
- **bat 与 bash 的编码分工**：bat 内容保持纯 ASCII（cmd 按当前代码页解析，非 ASCII 易出怪错，同 `interop.md` §1.3 的 `.ps1`/`.vbs`）；要让 bash 侧 UTF-8 中文在 cmd 窗口正常显示，bat 先 `chcp 65001 >nul`。**Windows 盘上的 `.sh` 必须 LF**：从 Linux 侧写入/同步后核对行尾（`file`/`od`），CRLF 用 `sed -i 's/\r$//'` 归一——Git Bash 对 CRLF 脚本报 `syntax error near unexpected token $'\r'`。
- **从 WSL2 非交互调用带 `pause` 的 bat**：`cmd.exe /d /c "X.bat <nul"`——`pause` 读到 nul 立即返回，末尾/失败分支的 pause 不再挂死；需要传参避开 pause 的脚本（`if "%~1"=="" pause` 形态）可传一个无害参数（如工程根目录）。配套给脚本内的 git/ssh 设 `GIT_TERMINAL_PROMPT=0`（cmd 内联 `set GIT_TERMINAL_PROMPT=0&&...`，无引号无空格的值可安全内联）——密码/主机指纹提示直接失败而非无限等输入（WSL 侧看不见交互窗口，挂了只能等超时）。
- **CMD 内联 `set` 含空格值的规避**：`set PATH=C:\Program Files\...;%PATH%` 这类含空格路径的内联 set 在 WSL interop 下不可靠（历史实测设不上）——需要临时 PATH 时写进 `.bat` wrapper（`set "VAR=..."` 在 bat 内可靠），再 `cmd /c wrapper.bat`；这也是本项目 winbuild.bat / MSBuild wrapper 模式的通用理由。

### 4.2 MSBuild

- **从 WSL2 调用的推荐方式**：`cmd.exe /c "$(wslpath -w path/to/build.bat)"`，由 build.bat 内部解析绝对路径。不要 pushd 映射盘符后调用（映射不稳定）、不要在 bash 里手拼 MSBuild 命令（引号嵌套易错）。
- **常用参数覆盖**：`/p:SolutionDir=<repo>\...\\`（注意尾部双反斜杠）、`/p:WindowsTargetPlatformVersion=<版本>`、`/p:PlatformToolset=<工具集>`——覆盖 `.vcxproj` 里可能缺失或与本机不匹配的默认值（具体版本值随项目）。
- **重建被占用的 EXE**：链接阶段报 `LNK1168: 无法打开 <path>.exe 进行写入`，伴随一批 DLL"共享冲突（复制了 0 个文件）"。先 `taskkill /IM <name>.exe`（不带 `/F` 对 GUI 程序是 WM_CLOSE 优雅关闭），确认退出后重跑。
- **up-to-date 构建也照样打印输出行——"OK" 不等于重新链接**：MSBuild `/v:m` 在项目判定 up-to-date 时仍打印 `xxx.vcxproj -> ...\xxx.exe` 与 post-build 拷贝步骤并回 `[build] OK`，但**产物时间戳不变**（同一 build.bat 连跑三次，exe 仍是上次构建时刻）。要证明改动真进了产物，对产物做字节级校验；既不要凭 OK 行下结论，也不要凭时间戳未变反推"构建失败"。
