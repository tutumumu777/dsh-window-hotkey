# dsh-window-hotkey

用全局快捷键开关 DeepSeek Harness 窗口（默认 `Ctrl+Alt+Q`），无论应用是否在前台；窗口已显示但被别的窗口盖住时，按热键会先置顶到前台，而不是直接最小化。
**改键直接在「通用设置 → 通用快捷键」里改，改完立刻生效。**

> **本 fork 的差异**：新增 `minimizeTo` 配置，可让热键把窗口**收进系统托盘**（任务栏按钮消失），而不是只最小化到任务栏。默认键改为 `Alt+Shift+I`。详见下方「收起方式：任务栏 vs 托盘」。

## 收起方式：任务栏 vs 托盘

`cordis.patch.yml` 里：

```yaml
config:
  enabled: true
  hotkey: alt+shift+i
  windowTitle: DeepSeek Harness
  # tray: 收进系统托盘（任务栏按钮消失，托盘图标常驻）
  # taskbar: 最小化到任务栏（上游默认行为）
  minimizeTo: tray
```

| 取值 | 行为 | 日志 |
| --- | --- | --- |
| `taskbar`（缺省） | `SC_MINIMIZE`，窗口留在任务栏 | `minimize ... iconic=True` |
| `tray` | `ShowWindow(SW_HIDE)`，任务栏按钮消失 | `hide-to-tray ... hidden=True` |

`tray` 分支之所以安全，是因为**收起之后仍然走原有的 `WakeViaSecondInstance` 唤醒路径**：
`second-instance` → `focusPrimaryWindow()` → Electron 自己的 `window.show()`。渲染进程是真被唤醒的，
不会出现"画面完整但点哪都没反应"的静态死页面。（那个问题是"用 `ShowWindow` 去**唤醒**"才有的；
"用 `SW_HIDE` 去**收起**"不受影响，已实测。）

实测（DSH Desktop 0.2.2-rc.3 / Windows 11）：`hide-to-tray hidden=True` 后按热键，
**261ms** 内唤回且 `woken=True`。

`SW_HIDE` 不经过 DSH 的 `close` 流程，因此不会弹一次性的「可在系统托盘重新打开」提示，
也不写 `background-close-confirmed` 标记 —— 热键收起不该弹对话框。

## 两个半边

| 半边 | 文件 | 作用 |
| --- | --- | --- |
| Host | `index.js` + `hotkey.js` | 起一个 PowerShell helper，用 Win32 `RegisterHotKey` 注册系统级热键，按键时最小化 / 还原 / 从隐藏唤醒窗口 |
| Client | `client.js` | 把命令注册进「通用设置 → 通用快捷键」，让这一项在 UI 里可见、可改键 |

Host 进程是 `ELECTRON_RUN_AS_NODE` 启动的，没有 BrowserWindow，也没有任何宿主服务暴露窗口句柄，
所以窗口只能经 Win32 直接驱动。helper 子进程故意不 detached，生命周期由脚本内的 `WatchParent`
线程盯住宿主 pid：宿主一死，helper 立刻注销热键并退出，不会留下占着热键的僵尸进程。

## 改键是怎么生效的

快捷键面板是 Electron 自己的编辑器，它只写 `keybindings.json`、只移动**应用内** binding；
真正的系统级热键由 helper 进程持有。两条链路本来互不相通，所以这里做了桥接，**两条路都通**：

1. **快路** — `client.js` 订阅 `shortcuts.catalog`，面板一改就 `POST /dsh-window-hotkey/bind`，
   Host 立刻换 helper 并回 `{ok, hotkey, label}`；机器抢不到该组合时返回 409，
   Client 用 `shortcuts.edit()` 把面板那一行改回实际注册的键（面板不会显示一个假键）。
2. **兜底** — Host 每 1 秒重读一次 `keybindings.json`（`%APPDATA%\@deepseek-ai\dsh-desktop\keybindings.json`）。
   即使页面还挂着旧 bundle、或你手改了文件，改键照样生效。

改完的选择缓存进 `hotkey-state.json`（插件目录内），重启后仍用这个键，不会被 `cordis.patch.yml` 覆盖。
在面板里清空这一行 = 主动解绑，会缓存成 `null`，重启也不会复活默认键。

两键和弦（如 `Ctrl+K Ctrl+S`）无法用 `RegisterHotKey` 表达，会被跳过并保持原注册不动。

## 关闭（不退出）后按热键为什么能活过来

点窗口 X 走的是 Electron 的 `window.hide()`，不是 Win32 隐藏，进程还在。纯 Win32
`ShowWindowAsync(SW_SHOW)` 只能让 HWND 变可见，Electron 主进程并不知道窗口"已显示"，
Chromium 的输入路由没被唤醒 —— 画面完整但点哪都没反应（就是之前那个"静态死页面"）。

所以隐藏分支改成 `WakeViaSecondInstance()`：再启动一次本 app 的 exe，让 Electron 自己走
`second-instance` → `focusPrimaryWindow()` → `window.show()` 把渲染进程真正唤醒。
启动前必须从环境里移除 `ELECTRON_RUN_AS_NODE`，否则新进程会跑成 node、拿不到单实例锁。

## 最小化后按一次热键为什么能直接唤醒

窗口最小化（`iconic`）分支走 Win32 `SC_RESTORE`。`SC_RESTORE` 返回时还原动画仍在异步进行，
立刻发 `SetForegroundWindow` 会撞上还原中的瞬态、被 Windows 前台锁静默拒绝 —— 表现为
「按第一次窗口还原了但没抢到前台、按第二次才成功」。

所以还原后会先轮询等 `IsIconic` 真正落地、再稍作延迟，然后才调 `Focus()`。`Focus()` 也不再
只看 `SetForegroundWindow` 的返回值，而是带验证的重试循环（最多 5 次）：先
`AttachThreadInput` 附着当前前台线程（后台进程抢前台的标准 workaround），再
`BringWindowToTop` + `SetForegroundWindow`，每轮用 `GetForegroundWindow()` 核对前台是否
真变成目标窗口，命中就成功返回。结果会写进 `hotkey.log`（`focus ok attempt=N` /
`focus gave up fg=… target=…`），不再出现「看着成功其实没抢到」的假象。

## 窗口已显示、只是被别的窗口盖住时

窗口可见但并不在前台（`GetForegroundWindow()` 拿到的是别的窗口）。此时按热键会先把它
**置顶到前台**，而不是顺手最小化：`Toggle()` 在展开分支里先取前台窗口，`foreground != hWnd`
就调 `Focus(hWnd)` 并写日志 `focus (was behind)`，然后直接返回。只有窗口已经处于前台时，
才走原来的最小化逻辑（日志 `minimize`）。

这样「按一下窗口不见了」只发生在窗口本来就可见且在最前面的情况，不会把用户已经看得见的
窗口藏到任务栏后面。

## 配置

`cordis.patch.yml` 里：

```yaml
config:
  enabled: true
  hotkey: alt+shift+i
  windowTitle: DeepSeek Harness
  minimizeTo: tray
```

- `enabled` — `false` 时完全不加载 helper。
- `hotkey` — 首次启动用的组合（`ctrl` / `alt` / `shift` / `win` 加一个主键：字母、数字、`F1`–`F24` 或具名键）。
  一旦在面板里改过，以 `hotkey-state.json` 为准。
- `windowTitle` — 按标题**后缀**匹配窗口；应用标题形如 `会话名 — DeepSeek Harness`，后缀匹配能命中。
- `minimizeTo` — `tray` 收进系统托盘，`taskbar`（缺省）最小化到任务栏。非法值按 `taskbar` 处理。

## 排障

`%LOCALAPPDATA%\Temp\dsh-hotkey\hotkey.log` 记录每次注册与开关：

```
[plugin] helper started pid=21184 hotkey=Alt+Shift+I
helper start pid=21184 parent=4184 thread=3180 minimizeTo=tray
registered mods=0x5 vk=0x49 id=1
hide-to-tray hwnd=0x00040532 hidden=True visible=False
focus (was behind) hwnd=0x00040532 fg=0x00040816 fronted=True title="DeepSeek Harness"
restore (was hidden) hwnd=0x00040532 woken=True
minimize hwnd=0x00040532 sent=True iconic=True
```

`registered mods=… vk=…` 出现 = 热键真的拿到手了；`register failed after … error=1409` = 被别的程序占着。
`minimizeTo=tray` 出现在 `helper start` 那行，说明配置已生效；收起时看 `hide-to-tray … hidden=True`。

`resolve()` 故意是空操作：窗口获得焦点时系统已经把按键交给了 helper，应用内再执行一次会连开关两下。
