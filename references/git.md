## 6. Git：Windows 盘仓库用 Windows git

> Windows 盘仓库的 git 命令优先经 Windows-MCP 的 `PowerShell` 执行（原生 PATH 与 cwd）；本节讲直调时的坑。

`/mnt/<盘符>` 经 9P 访问 Windows 仓库虽可行，但慢且可能触发索引格式/换行符/文件锁的隐式问题：

- **Windows NTFS 盘上的仓库一律用 Windows 原生 `git.exe`**（`cmd.exe /c "cd /d D:\repo && git.exe ..."`）；禁止 `git -C /mnt/d/...` 操作它们；WSL2 ext4 上的仓库照常用 WSL git；
- 多行 commit message 从 WSL2 传入时走 `-F <文件>`（内层引号在 cmd 里会丢）。

### 6.1 Windows OpenSSH 报 "Bad owner or permissions"

根因：`.ssh` 下文件的**宽 ACL**——继承自 Home 目录的 `Everyone` / `BUILTIN\Users` 条目（带 `(I)`），或 WSL 侧改写文件时引入的 AppContainer 包 SID；Windows OpenSSH 严格检查只允许 owner + SYSTEM + Administrators 访问。`icacls <file>` 看到带 `(I)` 的 Everyone/Users 条目即确诊。

修复（目录与所有密钥/配置都要收紧；主体用 SID 避免本地化组名问题）：

```powershell
$ssh = "$env:USERPROFILE\.ssh"
$me  = "$env:USERDOMAIN\$env:USERNAME"
icacls "$ssh" /inheritance:r /grant:r "${me}:(OI)(CI)F" "*S-1-5-18:(OI)(CI)F" "*S-1-5-32-544:(OI)(CI)F"
foreach ($f in @("config", "id_rsa", "known_hosts")) {
    icacls "$ssh\$f" /inheritance:r /grant:r "${me}:F" "*S-1-5-18:F" "*S-1-5-32-544:F"
}
```

要点：`/inheritance:r` 先切断继承；**目录也要修**（否则新建文件继续继承宽 ACL、问题复发）；owner 不对时先 `takeown`。验证：`git.exe fetch` 恢复、`icacls` 复查只剩三条不带 `(I)` 的条目。

### 6.2 git 报 `Host key verification failed`，但裸 `ssh` 却能通

同一 cmd 环境里 `ssh -T git@<host>` 正常（有 Welcome 横幅），而 `git.exe fetch/push/ls-remote` 报 `Host key verification failed`——**两者用的 ssh 不是同一个**：

- git 网络操作起的是 **Git-for-Windows 自带的 ssh**（`<Git>\usr\bin\ssh.exe`）：Unix 风格工具，读 `.ssh` 的路径规则与 Windows 用户目录脱节——**两个触发面**：① WSL interop 继承了 `HOME=/home/sean` 时，它去 `/home/sean/.ssh/known_hosts`（Windows 视角不存在）查主机指纹；② **HOME 未传递（为空）时同样失败**——`%USERPROFILE%` 含非 ASCII 用户名（如中文）时，它把用户目录读成不可访问的路径（2026-10-13 实测：`GIT_SSH_COMMAND=ssh -v` 的 verbose 里身份文件与 known_hosts 的查找路径为 `/c/Users/<用户名 GBK 字节>/...`，逐项 `No such file`——中文用户名的 MSYS 路径编码缺陷）。两面的共同点：**Git 自带 ssh 读不到该 Windows 用户的 `.ssh`** → 必然失败；
- 裸 `ssh` 通常命中 PATH 第一顺位 = 系统 OpenSSH（`C:\Windows\System32\OpenSSH\ssh.exe`）：原生程序，按 Windows 规矩用 `%USERPROFILE%\.ssh`（keys / known_hosts 齐）→ 正常。

解法（不动物理 config）：网络操作显式指定 ssh——

```cmd
cmd.exe /c "cd /d D:\repo && git.exe -c core.sshCommand=C:/Windows/System32/OpenSSH/ssh.exe fetch --depth 1 origin <branch>"
```

（等价持久化：对该仓库写一次 `core.sshCommand` local config。）**push 实场景已验证（2026-10-13）**：`-c "core.sshCommand=<System32 ssh> [-o 保活参数…]" push` 全链走通——注意原 global `core.sshCommand` 带的连接保活参数会被 `-c` 覆盖，需要就一并带上。判据：**「Host key verification failed 但裸 ssh 能通」= 此坑**，与 §6.1 的 ACL 坑（报 Bad owner or permissions）区分。
