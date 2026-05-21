# 事件系统

## 模块职责

`src/events/` 目录包含 AltTab 的全部事件订阅与分发逻辑。该模块是应用与 macOS 系统之间的桥梁，负责：

- 拦截全局键盘快捷键并触发窗口切换器
- 监听辅助功能（Accessibility）通知，追踪窗口的创建、销毁、移动、缩放、标题变更及焦点转移
- 感知系统环境变化：Space 切换、屏幕增减、深色模式切换、输入法变更、睡眠/唤醒
- 处理触控板手势、光标交互、滚轮事件等输入
- 提供 CLI 进程间通信（IPC）通道
- 监听偏好设置变更并触发相应的副作用

事件系统采用专用 RunLoop 线程处理底层事件，通过 `DispatchQueue.main.async` 将 UI 更新调度回主线程，配合多层节流机制保证性能和响应性。

## 核心组件

### 专用线程与队列（`BackgroundWork`）

事件处理分布在多个专用线程和操作队列上，定义于 `src/util/BackgroundWork.swift`：

| 线程/队列 | 类型 | 用途 |
|---|---|---|
| `accessibilityEventsThread` | `BackgroundThreadWithRunLoop` | AX Observer 回调（辅助功能通知） |
| `keyboardAndMouseAndTrackpadEventsThread` | `BackgroundThreadWithRunLoop` | 键盘 CGEventTap、触控板手势、滚轮事件 |
| `missionControlThread` | `BackgroundThreadWithRunLoop` | Dock Mission Control 状态通知 |
| `cliEventsThread` | `BackgroundThreadWithRunLoop` | CLI CFMessagePort 事件 |
| `screenshotsQueue` | `LabeledOperationQueue` (并发=8) | 窗口截图捕获 |
| `accessibilityCommandsQueue` | `LabeledOperationQueue` (并发=4) | AX 焦点/关闭/最小化等命令 |

`BackgroundThreadWithRunLoop` 继承自 `Thread`，在 `main()` 中调用 `CFRunLoopRun()` 进入运行循环，并通过一个空操作 `CFRunLoopSource` 防止 RunLoop 在无源时退出。线程启动使用信号量（`threadStartSemaphore`）实现同步等待，确保 `runLoop` 属性在构造完成后即可用。

### 节流基础设施

#### Throttler（`Throttler`）

基于时间戳的简单节流器，默认在主线程上运行。`throttleOrProceed(_:)` 检查距上次执行是否超过指定延迟；若未超过，安排一个延迟 10ms 的尾部调度以确保最终执行。用于 `SpacesEvents`（200ms）、`ScreensEvents`（200ms）等场景。

#### ThrottlerWithKey（`ThrottlerWithKey`）

带键的多实例节流器，内部使用 `ConcurrentMap<String, ThrottleState>` 存储每个键的独立状态。支持 `tailScheduled` 尾部调度模式，确保同键事件最终被执行一次。用于 `Applications.appListUpdateThrottler`（200ms）、`Applications.windowListUpdateThrottler`（200ms）、`Applications.captureThrottler`（200ms）。

`ConcurrentMap` 使用 `os_unfair_lock` 实现轻量级线程安全（比 NSLock 快约 10 倍的无竞争路径），仅保护字典的查找或赋值操作。

#### AXCallScheduler（`AXCallScheduler`）

AX 调用专用调度器（单例），提供节流 + 重试 + 并发管理三层能力：

- **节流**：`throttleDelayNs = 200ms`，同一 key 的调用间隔至少 200ms
- **重试**：失败后按指数退避重试 `[200ms, 1s, 2s, 5s, 5s, ...]`，60 秒后放弃
- **无响应 PID 隔离**：失败的 PID 被标记为 `unresponsivePids`，其后续调用路由到 `retryQueue` 而非 `fastQueue`
- **阶段状态机**：每个 key 有 `idle -> throttled -> executing -> retrying` 四个阶段

内部使用 `NSLock` 保护 `keyStates` 字典，所有状态转换均为原子操作。

## 键盘事件系统

键盘事件系统采用三层拦截架构，从最底层到最高层依次为：

### 第一层：CGEventTap（`KeyboardEvents.addCgEventTap()`）

这是最早的拦截点，使用 `CGEvent.tapCreate(tap: .cghidEventTap, ...)` 创建。关键参数：

- **Tap 位置**：`.cghidEventTap` — HID 级别，早于 `cgSessionEventTap`，能击败 macOS 26 的 Game Overlay（issue #5585）
- **插入位置**：`.headInsertEventTap` — 队头插入，最先处理
- **选项**：`.defaultTap` — 可吸收（吞掉）事件
- **事件掩码**：`flagsChanged` + `keyDown`
- **运行线程**：`BackgroundWork.keyboardAndMouseAndTrackpadEventsThread.runLoop`

回调 `cgEventHandler` 处理三种事件类型：

1. **`.flagsChanged`** — 修饰键状态变化。核心逻辑是 Hold-Modifier-Then-Tab 模式：
   - 检测 `SwitcherSession.current?.holdMask`，当修饰键不再完全满足 hold 掩码时
   - 通过 `cgEvent.copy()?.postToPid(session.initialPid)` 将 release 事件回投到原始应用
   - 返回 `nil` 吸收系统路由的副本，避免目标应用收到重复 release

2. **`.keyDown`** — 仅处理 Esc 键（当 `anyShortcutUsesEscape && SwitcherSession.isActive`）。在 cghid 级别吸收 Esc 事件，先于 Game Overlay 的下游 hook 拦截。

3. **`.tapDisabledByUserInput / .tapDisabledByTimeout`** — 自动重新启用 event tap。

### 第二层：Carbon RegisterEventHotKey（`KeyboardEvents.addGlobalHandlerIfNeeded()`）

使用 Carbon HIToolbox 的 `RegisterEventHotKey` API 注册全局快捷键：

- **事件目标**：`GetEventDispatcherTarget()` — 不需要辅助功能权限
- **签名**：`"altt"` 转为 `OSType` 四字符码
- **事件处理器**：分别安装 `kEventHotKeyPressed` 和 `kEventHotKeyReleased` 两个处理器
- **映射**：通过 `EventHotKeyID.id` 整数标识符映射到具体的快捷键 ID

`KeyboardEventsTestable.globalShortcutsIds` 维护控件 ID 到整数的映射，`nextWindowShortcut{0..N}` 映射 `0..N`，`holdShortcut{0..N}` 映射 `N+1..2N`。

快捷键的注册/注销通过 `registerHotKeyIfNeeded` / `unregisterHotKeyIfNeeded` 动态管理，支持 `toggleGlobalShortcuts` 全局启用/禁用。

### 第三层：NSEvent Local Monitor（`KeyboardEvents.addLocalMonitorForKeyDownAndKeyUp()`）

使用 `NSEvent.addLocalMonitorForEvents(matching: [.keyDown, .keyUp])` 监听应用内的键盘事件。仅当 AltTab 面板是 key window 时有效。返回 `nil` 吞掉匹配的事件，返回 `event` 放行。

### 统一调度：`handleKeyboardEvent()`

所有三层拦截最终汇聚到全局函数 `handleKeyboardEvent(_:_:_:_:_:_:)`（定义于 `KeyboardEventsTestable.swift`），执行流程：

1. **搜索编辑拦截**：如果 Switcher 处于活动状态、搜索框正在编辑，优先让 `TilesView.handleSearchEditingKeyDown` 处理，返回 `.handled`（吞掉）、`.passToField`（传递给搜索框）或 `.passToShortcuts`（继续快捷键匹配）
2. **日志记录**：`logKeyboardEvent` 输出调试信息
3. **快捷键匹配**：遍历 `ControlsTab.shortcuts`，每个快捷键调用 `matches()` 检查是否匹配当前事件，匹配则执行 `executeAction()`
4. **hold 快捷键透传**：`holdShortcut` 类型匹配时不设置 `someShortcutTriggered = true`，确保 alt-up 等释放事件能传递给活动应用

### Hold-Modifier-Then-Tab 模式

`SwitcherSession` 中维护两个关键状态：

- `initialPid: pid_t?` — 触发切换器时前台应用的 PID
- `holdMask: NSEvent.ModifierFlags` — 当前 hold 快捷键的修饰键掩码

当用户按住修饰键触发切换器后松开修饰键时，CGEventTap 的 `flagsChanged` 回调检测到 `!modifiers.isSuperset(of: session.holdMask)`，将释放事件通过 `postToPid` 回投到原始应用，使原始应用看到完整的 press-release 序列。

## 辅助功能事件系统

### 通知订阅

Application 和 Window 各维护一个通知列表：

**Application 通知**（`Application.notifications`）：
- `kAXApplicationActivatedNotification` — 应用激活
- `kAXMainWindowChangedNotification` — 主窗口变化
- `kAXFocusedWindowChangedNotification` — 焦点窗口变化
- `kAXWindowCreatedNotification` — 窗口创建
- `kAXApplicationHiddenNotification` — 应用隐藏
- `kAXApplicationShownNotification` — 应用显示

**Window 通知**（`Window.notifications`）：
- `kAXUIElementDestroyedNotification` — 窗口销毁
- `kAXTitleChangedNotification` — 标题变更
- `kAXWindowMiniaturizedNotification` — 最小化
- `kAXWindowDeminiaturizedNotification` — 取消最小化
- `kAXWindowResizedNotification` — 缩放
- `kAXWindowMovedNotification` — 移动

### 渐进式订阅策略

订阅采用先尝试第一个通知、成功后再依次订阅后续通知的模式：

1. 通过 `AXObserverCreate` 创建观察者
2. 将第一个通知的订阅提交到 `AXCallScheduler`（key: `"sub-win-{wid}"` 或 `"sub-app-{pid}"`）
3. 若第一个订阅成功（`subscribeToNotification` 返回 `true`），再将其余通知逐个提交到 `AXCallScheduler`
4. 将 `AXObserverGetRunLoopSource` 添加到 `BackgroundWork.accessibilityEventsThread.runLoop`

这种渐进式策略的原因是：部分应用在启动阶段尚未完全初始化，第一个订阅可能返回 `.cannotComplete`；`AXCallScheduler` 会自动按退避策略重试，直到应用准备好。一旦首个订阅成功，说明应用已就绪，可以安全订阅其余通知。

`subscribeToNotification` 方法（`AXUIElement` 扩展）处理三种结果：
- `.success` / `.notificationAlreadyRegistered` — 返回 `true`
- `.notificationUnsupported` / `.notImplemented` — 返回 `false`（永远不重试）
- 其他错误 — `throw AxError.runtimeError`（触发 AXCallScheduler 重试）

### 事件回调与桶式节流

`AccessibilityEvents.axObserverCallback` 是统一的 AX 回调入口，所有通知都汇聚到这里：

```
AX Observer callback -> AXCallScheduler.submit -> handleEvent(type, element)
```

**桶式节流（Bucket Throttling）** 通过 `bucket(for:)` 方法将通知分为三桶：

| 桶名 | 包含的通知 | 节流键模式 |
|---|---|---|
| `"focus"` | `kAXMainWindowChangedNotification`, `kAXFocusedWindowChangedNotification` | `"wid-{wid}-focus"` |
| `"geometry"` | `kAXWindowResizedNotification`, `kAXWindowMovedNotification` | `"wid-{wid}-geometry"` |
| `"generic"` | 其余所有通知 | `"wid-{wid}-generic"` |

同一桶内的通知被视为可互换的（interchangeable），只处理最新的一次。focus 桶独立出来是为了防止 geometry/title 突发通知覆盖 `Window.lastFocusOrder` 的更新（issue #5492 / #5580）。

### 事件分发流程

1. **应用级事件**（`handleEventApp`）：
   - 通过 `Applications.appListUpdateThrottler`（200ms 按键节流）调度
   - 处理 `kAXApplicationActivatedNotification` → 更新 `frontmostPid`、检查快捷键是否应禁用
   - 处理隐藏/显示通知 → 200ms 延迟后刷新 UI（等应用 UI 就绪）

2. **窗口级事件**（`handleEventWindow`）：
   - 通过 `Applications.windowListUpdateThrottler`（200ms 按键+桶节流）调度
   - 一次性读取所有 AX 属性（title, subrole, role, size, position, fullscreen, minimized, children）
   - 调用 `Windows.findOrCreate` 查找或创建窗口
   - 根据通知类型调用 `focusedWindowChanged`、`windowResizedOrMoved` 或通用刷新

3. **窗口销毁事件**（`windowDestroyed`）：
   - 直接在主线程处理，不经过节流
   - 通过 `isEqualRobust` 匹配 AXUIElement 或 CGWindowID
   - 调用 `Windows.removeWindows` 移除窗口

### Dock Mission Control 事件（`DockEvents`）

通过 AX Observer 监听 Dock 进程的 Mission Control 状态通知（`MissionControlState.allCases`），在 `BackgroundWork.missionControlThread.runLoop` 上运行。使用 `AXCallScheduler` 管理订阅。

## 系统事件处理

### 应用启动/退出（`RunningApplicationsEvents`）

使用 KVO 观察 `NSWorkspace.shared.runningApplications` 属性变化（`[.old, .new]` 选项），而非 `NSWorkspace.didLaunchApplicationNotification`，因为后者仅对部分 GUI 应用生效。通过比较 `newValue` 和 `oldValue` 分别获取启动和退出的应用列表。

### Space 变化（`SpacesEvents`）

监听 `NSWorkspace.activeSpaceDidChangeNotification`，使用 `Throttler(200ms)` 节流。处理流程：

1. `Windows.updateIsFullscreenOnCurrentSpace()` — 修复 Safari 全屏视频的 workaround
2. 检查前台应用的焦点窗口是否需要禁用快捷键
3. 如果 UI 在 Space 切换期间保持打开，刷新所有窗口列表

### 屏幕变化（`ScreensEvents`）

监听 `NSApplication.didChangeScreenParametersNotification`，使用 `Throttler(200ms)` 节流。屏幕通知通常成组到达，节流避免重复处理。处理流程：

1. `Spaces.refresh()` + `Screens.refresh()` — 刷新 Space 和屏幕缓存
2. `App.resetPreferencesDependentComponents()` — 屏幕增减或分辨率变化可能破坏布局
3. `App.refreshOpenUiAfterExternalEvent(Windows.list)` — 刷新所有窗口

### 睡眠/唤醒（`SleepWakeEvents`）

监听 `NSWorkspace.didWakeNotification`。采用**双重重试策略**：

1. **立即重试**：`reEnableAllTaps()` — 唤醒后立刻重新启用所有 CGEventTap
2. **延迟重试**：`DispatchQueue.main.asyncAfter(deadline: .now() + 3)` — 3 秒后再次尝试

`reEnableAllTaps()` 依次调用：
- `TrackpadEvents.reEnableTapIfNeeded()`
- `ScrollwheelEvents.reEnableTapIfNeeded()`
- `KeyboardEvents.reEnableTapIfNeeded()`
- `CursorEvents.reEnableTapIfNeeded()`

双重重试的原因是：唤醒后系统可能尚未完全恢复 HID 子系统，第一次重试可能失败，3 秒后再次尝试可覆盖这种情况。

### 系统外观变化（`SystemAppearanceEvents`）

监听分布式通知 `AppleInterfaceThemeChangedNotification`（macOS 10.14+），读取 `UserDefaults.standard.string(forKey: "AppleInterfaceStyle")` 判断深色/浅色模式，调用 `App.resetPreferencesDependentComponents()` 重置组件布局。

### 滚动条样式变化（`SystemScrollerStyleEvents`）

监听 `NSScroller.preferredScrollerStyleDidChangeNotification`，强制将 `TilesView.scrollView.scrollerStyle` 设为 `.overlay`，确保 overlay 滚动条样式。

### 输入法切换（`InputSourceEvents`）

通过 Carbon TIS（Text Input Source）API 监听输入法变化：

- 通知名：`kTISNotifySelectedKeyboardInputSourceChanged`
- 分布式通知中心：`DistributedNotificationCenter.default()`
- 挂起行为：`.deliverImmediately`（不等待应用激活）
- 回调中调用 `ControlsTab.inputSourceChanged()` 更新快捷键匹配状态
- `currentInputSource()` 通过 `TISCopyCurrentKeyboardInputSource()` + `TISGetInputSourceProperty(kTISPropertyLocalizedName)` 获取当前输入法名称

## 输入事件处理

### 触控板手势（`TrackpadEvents`）

使用 `CGEvent.tapCreate(tap: .cghidEventTap, ...)` 拦截手势事件（`NSEvent.EventTypeMask.gesture`），运行在 `keyboardAndMouseAndTrackpadEventsThread` 上。

**手势检测器层次**：

1. **`TriggerSwipeDetector`** — 检测触发手势（打开切换器的滑动）
   - 最小滑动距离：`MIN_SWIPE_DISTANCE = 0.015`（触控板面积的百分比）
   - 最大偏移容差：`MAX_SWIPE_DISTANCE_IN_WRONG_DIRECTION = 0.1`
   - 支持水平和垂直方向（根据 `Preferences.nextWindowGesture`）
   - 触发后执行 `App.showUiOrCycleSelection`，打开触觉反馈

2. **`NavigationSwipeDetector`** — 在切换器打开时检测导航滑动
   - 最小滑动距离：`MIN_SWIPE_DISTANCE = 0.03`
   - 支持 4 个方向（上下左右）
   - 触发后执行 `App.cycleSelection`，循环选择窗口

3. **`NonFreshGestureDetector`** — 检测"不新鲜"的手势（已有其他手势发生后的额外手势），避免误触发

4. **`GestureTracker`** — 底层位置追踪器，记录每个触点的起始位置，计算移动距离

**触觉反馈**：当 `Preferences.trackpadHapticFeedbackEnabled` 时，通过 `NSHapticFeedbackManager.defaultPerformer.perform(.generic, ...)` 提供。

**动态启用/禁用**：`TrackpadEvents.toggle(_:)` 根据 `Preferences.nextWindowGesture != .disabled` 动态启用或禁用 event tap。

### 光标事件（`CursorEvents`）

使用 `CGEvent.tapCreate(tap: .cgSessionEventTap, ...)` 拦截鼠标事件，运行在**主线程**的 RunLoop 上（因为需要访问 UI 坐标）。

监听的事件类型：`leftMouseDown/Up`、`rightMouseDown/Up`、`otherMouseDown/Up`、`mouseMoved`。

**事件处理逻辑**：

- **左键按下**：检查搜索框标记文本、搜索框内点击 → 放行；UI 内点击 → 吞掉并记录目标（按钮或 Tile）；UI 外点击 → 吞掉
- **左键释放**：匹配按下时的目标，执行 `button.onClick()` 或 `target.mouseUpCallback()`
- **右键/中键**：右键在 UI 外被吞掉（防止触发系统右键菜单）；中键在 Tile 上释放时关闭窗口或退出应用
- **鼠标移动**：通过死区（dead zone）机制过滤触控板滑动产生的微小移动（阈值 25pt）

**死区机制**：`deadZoneInitialPosition` 记录初始位置，只有移动距离超过 25pt 才设置 `isAllowedToMouseHover = true`，避免触控板手势时的误悬停。

### 滚轮事件（`ScrollwheelEvents`）

使用 `CGEvent.tapCreate(tap: .cghidEventTap, ...)` 拦截滚轮事件，运行在 `keyboardAndMouseAndTrackpadEventsThread` 上。

仅拦截连续（trackpad）滚动（`scrollWheelEventIsContinuous != 0`），放行离散（鼠标滚轮）滚动。默认禁用，在切换器打开且检测到多指触控时由 `TrackpadEvents` 动态启用。

### 右键菜单事件（`ContextMenuEvents`）

监听 `NSMenu.didBeginTrackingNotification` / `NSMenu.didEndTrackingNotification`，维护 `openMenuCount` 计数器。当菜单打开时，光标事件处理器检查 `ContextMenuEvents.isMenuOpen` 放行所有鼠标事件，避免干扰菜单操作。

## CLI 事件系统（`CliEvents`）

基于 `CFMessagePort` 的进程间通信（IPC）机制：

- **端口名**：`"{bundleIdentifier}.cli"`（如 `com.lwouis.alt-tab-macos.cli`）
- **运行线程**：`BackgroundWork.cliEventsThread.runLoop`
- **回调**：`CFMessagePortCallBack`，接收 UTF-8 编码的命令字符串

**支持的命令**（`CliServer`）：
- `--list` — 返回 JSON 格式的窗口列表（id + title）
- `--detailed-list` — 返回完整窗口信息（app、space、fullscreen、minimized 等）
- `--focus={id}` — 按 CGWindowID 聚焦窗口
- `--focusUsingLastFocusOrder={n}` — 按 lastFocusOrder 聚焦窗口
- `--show={index}` — 显示指定快捷键索引的切换器

命令在 `DispatchQueue.main.sync` 中执行（因为需要访问 UI 状态），通过 `JSONEncoder` 编码响应数据。

`CliClient` 用于命令行启动模式，通过 `CFMessagePortSendRequest` 发送命令并处理响应。

## 偏好设置事件系统

### 偏好变更分发（`PreferencesEvents`）

`preferenceChanged(_:)` 作为偏好变更的中心分发器：

- **菜单栏图标**：`"menubarIcon"`, `"menubarIconShown"` → 调用 `Menubar.menubarIconCallback`
- **触控板手势**：`"nextWindowGesture"` → 调用 `TrackpadEvents.toggle`
- **开机启动**：`"startAtLogin"` → 调用 `LoginItem.applyCurrentPreference`
- **更新策略**：`"updatePolicy"` → 调用 Sparkle 更新配置
- **外观相关**：`"appearanceStyle"`, `"appearanceSize"`, `"appearanceTheme"`, `"showOnScreen"` + 覆盖键 → 调用 `App.resetPreferencesDependentComponents`
- **Pro 功能检查**：如果 Pro 功能被锁定且用户修改了 Pro 偏好，导航到升级页面

### UserDefaults 监听（`UserDefaultsEvents`）

使用 KVO 监听 Sparkle 的自动更新偏好键：
- `"SUAutomaticallyUpdate"` — 自动下载安装
- `"SUEnableAutomaticChecks"` — 自动检查更新

当 Sparkle UI 修改这些键时，`UserDefaultsEvents` 将变化同步到 AltTab 自己的 `"updatePolicy"` 偏好，保持 UI 一致性。使用 `GeneralTab.policyLock` 防止循环更新。

## 窗口截图事件（`WindowCaptureEvents`）

### ScreenCaptureKit 截图（`WindowCaptureScreenshots`，macOS 14+）

使用 `SCScreenshotManager.captureSampleBuffer` 进行窗口截图：

- **缓存层**：`cachedSCWindows: ConcurrentArray<SCWindow>` 缓存 `SCShareableContent` 查询结果
- **优先级排序**：优先处理有 `prioritizedIds` 标记的窗口
- **节流**：通过 `Applications.captureThrottler`（200ms 按 WID 节流）
- **活跃计数**：`ActiveWindowCaptures` 使用原子操作 `OSAtomicIncrement32/Decrement32` 跟踪进行中的截图数

### 私有 API 截图（`WindowCaptureScreenshotsPrivateApi`）

使用 `CGSHWCaptureWindowList` 私有 API（通过 `@_silgen_name` 声明），优势是可以截取最小化窗口的截图。

### 截图尺寸计算（`SCStreamConfiguration.forWindow`）

根据是否启用预览选择窗口（`anyPreview`）决定尺寸：
- **有预览**：使用原始分辨率（`width * scaleFactor`）
- **无预览**：缩放到最大缩略图尺寸（`min(1.0, maxSize/size)`）

## 节流策略

整个事件系统采用多层节流策略，从底层到顶层依次为：

| 层级 | 机制 | 延迟 | 作用域 |
|---|---|---|---|
| AX IPC | `AXCallScheduler` | 200ms 节流 + 指数退避重试 | 每个 AX 调用 key |
| 应用列表更新 | `Applications.appListUpdateThrottler` | 200ms | 每个 PID |
| 窗口列表更新 | `Applications.windowListUpdateThrottler` | 200ms | 每个 WID+桶 |
| 截图捕获 | `Applications.captureThrottler` | 200ms | 每个 WID |
| Space 变化 | `Throttler` | 200ms | 全局 |
| 屏幕变化 | `Throttler` | 200ms | 全局 |
| 桶式分类 | `bucket(for:)` | — | focus / geometry / generic |
| 搜索编辑 | `shouldAbsorbSearchEditingKeyDown` | — | 活跃切换器会话 |

## 文件清单

| 文件 | 行数 | 核心职责 |
|---|---|---|
| `KeyboardEvents.swift` | 182 | 三层键盘拦截：CGEventTap (.cghidEventTap) + Carbon RegisterEventHotKey + NSEvent Local Monitor |
| `KeyboardEventsTestable.swift` | 67 | 全局快捷键 ID 映射、统一键盘事件处理函数 `handleKeyboardEvent` |
| `AccessibilityEvents.swift` | 159 | AX 事件回调、桶式节流（focus/geometry/generic）、窗口与应用事件处理 |
| `WindowCaptureEvents.swift` | 337 | 窗口截图协调：ScreenCaptureKit 截图 + 私有 API 截图 + 活跃计数 |
| `SpacesEvents.swift` | 29 | Space 变化监控，200ms 节流 |
| `ScreensEvents.swift` | 24 | 屏幕参数变化监控，200ms 节流 |
| `SleepWakeEvents.swift` | 20 | 睡眠唤醒监控，双重重试策略（立即 + 3 秒延迟） |
| `TrackpadEvents.swift` | 258 | 触控板手势拦截、TriggerSwipe/NavigationSwipe/NonFreshGesture 检测器 |
| `CursorEvents.swift` | 190 | 鼠标事件拦截（.cgSessionEventTap），死区机制，Tile/按钮点击处理 |
| `ScrollwheelEvents.swift` | 53 | 滚轮事件拦截，仅过滤连续（trackpad）滚动 |
| `RunningApplicationsEvents.swift` | 24 | KVO 监听应用启动/退出 |
| `SystemAppearanceEvents.swift` | 16 | 深色模式切换监听 |
| `SystemScrollerStyleEvents.swift` | 14 | 滚动条样式变化，强制 overlay 样式 |
| `InputSourceEvents.swift` | 26 | 输入法切换监听，Carbon TIS API |
| `ContextMenuEvents.swift` | 44 | 右键菜单状态跟踪（openMenuCount） |
| `CliEvents.swift` | 160 | CLI IPC：CFMessagePort 服务端 + 命令执行 + 客户端 |
| `PreferencesEvents.swift` | 89 | 偏好变更中心分发器 |
| `UserDefaultsEvents.swift` | 39 | KVO 监听 Sparkle 自动更新偏好 |
| `DockEvents.swift` | 35 | Dock Mission Control 状态通知 |
