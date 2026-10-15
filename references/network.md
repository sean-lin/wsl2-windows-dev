## 7. mirrored 网络：端口与进程身份

### 7.1 同名端口：WSL2 侧遮蔽 Windows 侧

mirrored 模式下 WSL2 与 Windows 共享 localhost。**两侧都绑 `127.0.0.1:<port>` 时，WSL2 侧的监听遮蔽 Windows 侧**——从 WSL2 连过去命中的是本地的那个，"永远看不到 Windows 侧服务"。典型触发：WSL2 侧某工具自动拉起本地 server。判据与处置：

```bash
ss -ltnp | grep <port>     # 看监听进程属于哪一端；是本地的就清掉
pkill -f '<工具> -m'       # 清掉后 client 直连 Windows server 即恢复
```

不要只看"端口 OPEN"就断定 Windows 侧在跑——必须看监听进程。

### 7.2 刚释放的 loopback 端口短暂仍可拨通

关闭绑在 `127.0.0.1` 的监听后，该端口**几毫秒内仍会接受 connect**（握手成功、一写数据即 RST），属 host 侧释放延迟——此刻 WSL2 netns 内 `ss -ltn` 已查不到监听。实测衰减：关闭后立即 70/300 拨通、20ms 后 1/300、100ms 后 0/300；从未绑定过的随机端口 0/800。

**对测试的影响**：`Close()` 后立刻断言"再拨必被拒"的用例会间歇性失败。判据要落在**库自身状态**（关闭标志、accept 循环退出），或给拨号留 ≥100ms 窗口；不要加重试掩盖。

### 7.3 跨端 server：管理命令会跨端生效

WSL 侧 client 直连 Windows 侧 server 时，client 发出的**管理命令跨端作用于 Windows server**。实测：从 WSL 发 `adb kill-server` 把 Windows 侧 server 一并杀掉，且 WSL 侧起不回（`start-server` 恒报 `Address already in use`，数十秒不缓解）。恢复：**从 Windows 侧把 server 起回来**（`adb.exe start-server`），无线设备再 `adb connect <host:port>`。

推广：任何"client 自动拉起 / 管理 server"的工具（adb、hdc 等），先问**这条命令打的是哪一端的 server**；不要从 WSL 侧对 Windows 持有的 server 发 kill / restart 类命令。根治方向见 §7.5：让 WSL 侧默认命令就是 Windows exe，就不存在"哪一端"的歧义。

### 7.4 Windows 侧 EXE 的新副本 = 全新身份

授权 / 配对 / 登录状态按**该 EXE 副本自己的身份目录**记账，不按机器。重解压一份 EXE 首次启动即生成全新身份目录，对端设备因从未见过而判未授权——**在别的副本上授权过不算数**。处置：对端重新确认授权；状态未翻转则重启该 EXE 的 server 重新握手。

### 7.5 双物种工具：让 WSL 默认命令就是 Windows exe（adb 实例）

同一工具两侧都有原生版本、且**自带 client→server 自拉起行为**时（adb 最典型：client 连不上
`127.0.0.1:5037` 会自行 spawn server），WSL 侧默认用 Linux 版会持续制造混乱：

- Windows server 缺席时，WSL client 自起的 **Linux server** 占住共享 loopback 端口——之后 Windows 侧起 server 前要先杀掉它，且该 server 看不到 Windows 的 USB 设备（§7.1 同类掩蔽）；
- WSL 侧管理命令跨端作用到 Windows server（§7.3）。

**根治：`ln -s /mnt/<drive>/<path>/<tool>.exe ~/bin/<tool>`**（`~/bin` 在 PATH 里先于两侧原生安装目录），让 WSL 裸跑该命令就是 Windows exe——client 与它自起的 server 同物种（都是 Windows 进程），"WSL 侧 server"这一物种不复存在；管理命令的语义也随之明确（`kill-server` 就是管 Windows server）。实测（adb 37.0.1，2026-09-28）：symlink 到 `.exe` 可直接执行（binfmt_misc 按 PE magic 匹配，文件名后缀无关）；无 server 时裸跑 `adb devices` 起的是 Windows server（Windows `tasklist` 可见 `adb.exe`，WSL `ss` 无 5037 监听——Windows 侧 socket 本就不在 Linux socket 表里）。需要 Linux 版原生能力（如读取 Linux 文件路径的参数）时，显式用其全路径调用。

### 7.6 停掉 WSL2 侧监听后，TIME_WAIT 会挡另一侧的重绑

停掉 WSL2 侧的监听服务时，它与对端**仍存在的已建立连接**会让服务端端口留在 `TIME_WAIT`（Linux 最长 60s）。窗口内 **Windows 侧 bind 同一 `127.0.0.1:<port>` 会失败**（`bind: Only one usage of each socket address …`），症状是"Windows 侧明明空闲却绑不上"。判据与处置：在 WSL2 `ss -tan | grep <port>`，看到 `TIME-WAIT … <port>` 行就等它自然消亡（几十秒）再启动 Windows 侧服务，不要立刻重试轰炸。实测（2026-10-01）：kill 掉仍连着 agent 的 WSL 侧 broker → Windows 侧同端口 broker 报 `Only one usage`；等 `ss -tan` 清空后 bind 成功。

### 7.7 查 Windows 端口清单用 `netstat -ano`，别用 `Get-NetTCPConnection`

`Get-NetTCPConnection` 走 `MSFT_NetTCPConnection` CIM 类，实测（Windows 11，2026-10-01）一次 `-State Listen` 查询要 **25–30s**（当时监听仅 57 条；同机 `Get-CimInstance Win32_Process` 3s、`netstat -ano` **0.15s**），足以打穿 10s 级程序超时。查监听 / 连接清单用 `netstat -ano`：数据行形如 `<proto> <local> <foreign> LISTENING <pid>`，`TCP` 与 `LISTENING` 关键字在任何语言版本的 Windows 上都不本地化（只有表头与汇总行本地化），可按它们安全过滤；地址列 IPv6 形如 `[::]:135`，解析时去括号。从 WSL2 直接调 `netstat.exe` 可用（interop 解析到 `C:\Windows\system32\netstat.exe`）。
