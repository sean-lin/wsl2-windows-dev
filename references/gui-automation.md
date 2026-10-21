## 8. GUI 自动化：WinAppDriver 点击的 z-order 约束

> 桌面级窗口 / 点击 / 置前优先用 Windows-MCP（`App` / `Click`）；被测应用的 UI 自动化仍走 WinAppDriver / Appium / Insight，本节讲其 z-order 约束。

WinAppDriver 的 `click` 是**物理注入**：光标移到元素屏幕坐标后注入按下/抬起，命中由该坐标处 **z-order 最顶**的窗口决定，且**不校验效果**——注入完成即返回 success。目标被其他窗口覆盖时（如测试期间切去浏览器/终端），点击**静默打给覆盖窗口**且仍报 success："点击成功但界面没反应"。非 topmost 弹出窗（自绘控件常见，如窗口关闭按钮）尤其脆弱：只在创建/位置变化时置顶，任何后激活的窗口都会压过它。

**诊断（WindowFromPoint hit-test）**：Windows Python + ctypes，先 `SetProcessDpiAwarenessContext(-4)`（PER_MONITOR_AWARE_V2）对齐物理像素，再对目标坐标做命中测试：

```python
import ctypes
from ctypes import wintypes
u = ctypes.windll.user32
u.SetProcessDpiAwarenessContext(-4)
hwnd = u.WindowFromPoint(wintypes.POINT(x, y))
root = u.GetAncestor(hwnd, 2)          # GA_ROOT
buf = ctypes.create_unicode_buffer(256)
u.GetClassNameW(hwnd, buf, 256)        # 命中窗口类名
pid = ctypes.c_ulong()
u.GetWindowThreadProcessId(hwnd, ctypes.byref(pid))
```

再按类名 `EnumWindows` 枚举目标窗口 rect、对中心点逐个 hit-test，即可判断"点击会打给谁"。

**处置**：发 native 点击前保证目标窗口在点击坐标处位于 z-order 顶部；对"成功但无效果"先做 hit-test 诊断，不要相信 click 返回值。该约束只存在于 Windows 原生点击路径——WebView/CDP 一类的输入注入不经过屏幕命中，不受影响。

### 8.1 坐标注入（WAD touch）与被遮挡窗口激活（2026-10-02 实测）

- **WinAppDriver `/actions` 只支持 pen / touch 指针类型**：mouse 类型被直接拒绝（"Currently only pen and touch pointer input source types are supported"，microsoft/WinAppDriver#1727，2022 至今未支持）。**touch 是唯一的 W3C 坐标注入通道**：`pointerMove(x,y,origin=viewport)` + `pointerDown/pointerUp`；桌面（Root）会话下坐标 = **屏幕绝对物理像素**（实测命中任务栏按钮与任意窗口内容坐标）。与元素级 `element.click()`（WAD 内部走鼠标注入、按 UIA 元素定位）是两条不同路径——后者的鼠标能力不受 `/actions` 限制牵连。
- **被遮挡窗口的激活（不写任何工具）**：物理注入只能命中坐标处 z-order 最顶窗口。窗口被其他窗口盖住时，touch 点击其**任务栏按钮**——任务栏在其条带处天然最顶；分组按钮会先弹**预览浮窗**，再点目标窗口缩略图即激活。此路径同时绕开"CLM 禁 PowerShell P/Invoke（Add-Type/COM 全禁）、本机无 Windows Python"这类环境限制。
- **更直接的激活通道（2026-10-03 实测）**：`cscript //nologo x.vbs` 跑 VBScript `CreateObject("WScript.Shell").AppActivate(<PID 或标题前缀>)` 一步把窗口带到前台——**WSH 不受 PowerShell CLM 约束**（CLM 只限 PowerShell 宿主），比"点任务栏 → 预览浮窗 → 点缩略图"更直接。要点：`.vbs` 放 Windows 盘、路径用 Windows 形式（`cscript.exe //nologo 'D:\...\x.vbs'` 经 interop 直调或 `cmd.exe /d /c` 包裹均可）；**PID 动态取、别写死**——进程重启即换号，写死的脚本会静默失效；用 WMI 枚举（`GetObject("winmgmts:\\.\root\cimv2").ExecQuery("SELECT ProcessId FROM Win32_Process WHERE Name='x.exe'")` 逐个 `AppActivate`，首个成功即止）；标题含非 ASCII 字符时纯标题匹配不可靠（实测全角标题 False），用 **PID** 更稳；返回 `False` 可重试。**`.vbs` 必须纯 ASCII（含注释）**——WSH 恒按 ANSI 解析，含 UTF-8 中文注释会吞行报假错（`缺少对象`），无 BOM 解法，见 `interop.md` §1.3。**用法样例**：物理注入型测试开跑前若担心目标窗被前台窗口遮挡，先 `AppActivate` 再注入；配合 Root 会话 `/screenshot` 确认已在前台。
- **`/screenshot` 是现场真值**：Root 会话截图 = 整屏（含任务栏与所有窗口），窗口位置、遮挡关系、命中效果一目了然；比任何"命令返回成功"的判断可靠。坐标类自动化每步注入前后各截一张，是最省事的取证法。
- **会话与端点坑**：Appium 会话默认约 60s 无命令即被回收（命令间隔长的流程显式设 `appium:newCommandTimeout`，否则中途 `invalid session id`）；桌面会话的 `/window_handles`、`/source` 枚举病态慢（可到分钟级直至超时，同 §8 前文的 Root session 教训）；`POST /window`（SetWindow）跨进程句柄会被拒（"Window handle does not belong to the same process/application"）；同一 WAD 实例**并发处理请求**（慢请求挂起时其他请求照常快速响应）。
- **坐标标定**：`屏幕 = client 原点 + 内容坐标 × 系统 DPI 缩放`。**不要假设标题栏高度**：取已知内容的 1–2 个边缘点反解原点与缩放（例：目标 UI 的已知逻辑 bounds 与截图中其像素边界对照，解出缩放与 client 原点；多点交叉验证到 ≤1px 即可按中心注入）。
