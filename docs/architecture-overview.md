# 架构总览

## 模块职责

### 应用入口 — `main.swift`

`main.swift` 是整个 AltTab 的入口文件，执行顺序严格为：

1. **CLI 命令检测**：通过 `CliClient.detectCommand()` 检查命令行参数。若检测到 CLI 命令（如 `--list`、`--focus=`、`--show=`），则调用 `CliClient.sendCommandAndProcessResponse(command)` 向已运行的 AltTab 实例发送 `CFMessagePort` 消息并输出结果后退出，不会进入 GUI 循环。这使得用户可以通过 `open AltTab.app --args --list` 执行 CLI 操作。

2. **信号处理**：注册 `SIGTERM` 和 `SIGTRAP` 信号处理器。`SIGTERM` 在 Activity Monitor 强制退出时触发；`SIGTRAP` 在 Swift 代码崩溃（如意外 nil）时触发。`SIGKILL` 无法拦截。信号处理器调用 `emergencyExit()` 函数。

3. **NSException 拦截**：通过 `NSSetUncaughtExceptionHandler` 捕获 Objective-C 层面的异常（如 unrecognized selector）。

4. **启动应用**：`App.shared.run()` 进入 Cocoa 事件循环。此调用之后定义的 `printStackTrace()`、`emergencyExit()`、`makeSureAllCapturesAreFinished()` 为辅助函数。

`emergencyExit()` 的行为：恢复原生 Command+Tab（`setNativeCommandTabEnabled(true)`），打印日志和调用栈，等待所有截图操作完成后调用 `exit(0)`。

`makeSureAllCapturesAreFinished()` 使用自旋等待（最多 5 秒超时），轮询 `ActiveWindowCaptures.value()` 确保无截图操作正在进行。这是为了避免一个 macOS bug：应用退出时若有截图操作未完成，系统会弹出权限对话框。

### 中央协调器 — `App.swift`

`App` 类继承自 `AppCenterApplication`（Obj-C 混编类，集成 AppCenter 崩溃报告），是整个应用的中央协调器。它同时担任 `NSApplicationDelegate` 角色。

**静态属性**：
- `activity`：通过 `ProcessInfo.beginActivity` 阻止 App Nap，保持应用响应性
- `bundleIdentifier`、`bundleURL`、`name`、`version`、`licence`：应用元数据
- `isTerminating`：全局终止标志，在 `applicationShouldTerminate` 和 `emergencyExit` 中设为 `true`
- `isVeryFirstSummon`：首次召唤标志，控制 `Windows.sortByLevel()` 的执行
- `refreshOpenUiThrottler`：200ms 主线程 UI 刷新节流器

**生命周期方法**（`NSApplicationDelegate`）：

- `applicationDidFinishLaunching`：初始化崩溃报告（`AppCenterCrash`）、日志系统（`Logger`）、偏好设置（`Preferences`）、许可证管理器（`LicenseManager`）、后台工作预启动（`BackgroundWork.preStart()`），最后调用 `SystemPermissions.ensurePermissionsAreGranted()` 触发权限检查流程
- `applicationShouldHandleReopen`：Dock 图标点击时显示设置窗口
- `applicationShouldTerminate`：等待所有截图完成（`makeSureAllCapturesAreFinished()`）
- `applicationWillTerminate`：恢复原生 Command+Tab
- `application(_:open:)`：处理自定义 URL Scheme（`bundleId://activate?license_key=xxx`），用于许可证激活

**核心方法**：

- `continueAppLaunchAfterPermissionsAreGranted()`：权限授予后的完整初始化链，依次启动后台线程、创建 UI 组件、注册事件观察者、发现运行中的应用、启动 Sparkle 更新器、显示首次启动引导等
- `showUi(_:)` / `hideUi(_:)`：显示/隐藏切换器面板
- `showUiOrCycleSelection(_:_:)`：统一入口，首次触发显示 UI，重复触发循环选择窗口
- `focusSelectedWindow(_:)`：隐藏 UI 并聚焦选定窗口，可选移动光标到窗口中心（`CGWarpMouseCursorPosition`）
- `refreshOpenUiAfterExternalEvent(_:)`：外部事件触发 UI 刷新，带 200ms 节流
- `refreshUi(_:)`：刷新 UI 内容（更新选中窗口、面板内容、预览、徽章）
- `buildUiAndShowPanel()`：构建 UI 并显示面板，处理延迟显示逻辑
- `checkIfShortcutsShouldBeDisabled(_:_:)`：根据应用例外列表禁用/启用全局快捷键

### 菜单栏管理 — `MainMenu.swift`

`MainMenu` 类负责创建 macOS 标准菜单栏（非状态栏图标菜单）。没有 `MainMenu` 时，标准快捷键如复制粘贴（Cmd+C/V）不可用。

**菜单结构**：
- **AltTab**：Preferences…（Cmd+,）、Services、Show All、Quit AltTab（Cmd+Q）
- **File**：Close（Cmd+W）、Page Setup…（Cmd+Shift+P）、Print…（Cmd+P）
- **Edit**：Undo（Cmd+Z）、Redo（Cmd+Shift+Z）、Cut/Copy/Paste/Select All、Find 子菜单、Spelling and Grammar、Substitutions、Transformations、Speech
- **Format**：Font 子菜单（Show Fonts、Bold、Italic 等）、Text 子菜单（对齐、书写方向等）
- **Window**：Minimize（Cmd+M）、Zoom、Bring All to Front
- **Help**：AltTab Help（Cmd+?）

**快捷键动态管理**：
- `toggle(_:)`：全局启用/禁用所有菜单项快捷键（切换器显示时禁用，避免冲突）
- `toggleEditMenu(_:)`：单独控制 Edit 菜单快捷键（搜索模式时启用，允许文字编辑）
- 内部通过 `menuItemsWithShortcut` 字典记录每个菜单项的原始快捷键，禁用时将 `keyEquivalent` 设为空字符串

### 菜单栏 UI — `Menubar.swift`

`Menubar` 类管理 macOS 状态栏图标及其下拉菜单。

**状态栏图标**：
- `statusItem`：`NSStatusItem`，支持左键点击弹出菜单、右键/其他点击显示切换器
- 图标偏好通过 `applyMenubarIconPreferences()` 应用，支持显示/隐藏和多种图标样式
- 可选橙色圆点徽章（`CALayer` 覆盖），用于 Pro 转化提示
- macOS 26+ 使用 SF Symbols 作为菜单项图标

**菜单项定义**（从上到下）：
1. Show（切换器）
2. --- 分隔线 ---
3. Settings…（Cmd+,）
4. Check for updates…
5. Check permissions…
6. --- 分隔线 ---
7. About AltTab
8. Debug tools
9. Send feedback…
10. Get Pro / My Account（根据许可证状态动态切换）
11. Support this project（红色心形图标，Pro 过期/试用过期时显示）
12. --- 分隔线 ---
13. Quit AltTab（Cmd+Q）

**许可证状态菜单项动态更新**：
- `refreshLicenseMenuItems()` 根据 `LicenseManager.shared.state` 切换显示
- `.trial`：显示 Get Pro（含剩余天数），隐藏 Support 和 Account
- `.pro`：隐藏 Get Pro 和 Support，显示 Account
- `.proExpired`：显示 Get Pro、Support、Account
- `.trialExpired`：显示 Get Pro、Support，隐藏 Account

**`UpgradeMenuItemView`**：自定义 NSView 作为 Get Pro 菜单项的视图，包含渐变背景（`ProGradient`）、星形图标、鼠标悬停时的光泽动画效果。

**`PermissionCallout`**：权限提示条，当 Screen Recording 权限未授予时在菜单顶部显示紫色提示栏。

## 核心组件

### MVC + 事件驱动架构

AltTab 采用 **MVC + 事件驱动** 的混合架构：

**Model 层**（`src/switcher/state/`）：
- `Application`：封装运行中的应用信息（PID、bundleIdentifier、图标、Dock 徽章等）
- `Window`：封装窗口信息（CGWindowID、标题、位置、大小、空间、最小化状态等）
- `Applications`：应用列表管理，包含节流器和窗口发现逻辑
- `Windows`：窗口列表管理，窗口排序、选中、循环选择
- `Spaces`：macOS 空间管理
- `Screens`：屏幕管理

**View 层**（`src/switcher/main-window/`）：
- `TilesPanel`：主面板窗口（NSPanel），承载所有 Tile 视图
- `TilesView`：滚动视图，管理 Tile 的布局和回收
- `TileView`：单个窗口卡片视图，包含缩略图、标题、应用图标、Dock 徽章等
- `TilesPanelBackgroundView`：面板背景视图，支持毛玻璃效果
- `PreviewPanel`：预览面板，显示选中窗口的大图

**Controller 层**：
- `App`：中央协调器，连接 Model 和 View
- `SwitcherSession`：单次切换器会话状态（从触发到释放）

### SwitcherSession — 会话生命周期

`SwitcherSession` 是 `final class`（引用类型）会话对象，生命周期由 `App` 管理：

- **创建时机**：`showUiOrCycleSelection` 中，当 `SwitcherSession.current == nil` 时创建新实例
- **销毁时机**：`hideUi()` 中将 `current` 设为 `nil`
- **关键属性**：
  - `shortcutIndex`：当前使用的快捷键索引（支持多组快捷键）
  - `isFirstSummon`：是否首次召唤（区分"打开面板"和"循环选择"）
  - `forceDoNothingOnRelease`：释放修饰键时不执行聚焦（搜索模式、首次显示时使用）
  - `initialPid`：触发时前台应用的 PID，用于将修饰键释放事件重投递给初始应用
  - `holdMask`：保持快捷键的修饰符掩码
  - `selectedIndex`、`hoveredIndex`、`searchQuery`：交互状态

### Preferences — 偏好设置系统

偏好设置系统位于 `src/preferences/`：
- `Preferences`：核心偏好管理类，基于 `UserDefaults`
- `PreferenceDefinition`：偏好项定义
- `PreferencesMigrations`：版本迁移
- `ShortcutConfiguration`：快捷键配置
- `MacroPreferences`：宏偏好设置

设置窗口（`src/preferences/settings-window/`）包含多个 Tab：
- `GeneralTab`：常规设置（更新策略、菜单栏图标、启动项等）
- `ControlsTab`：快捷键控制
- `AppearanceTab`：外观设置
- `ExceptionsTab`：应用例外
- `AboutTab`：关于页面
- `UpgradeTab`：Pro 升级
- `AcknowledgmentsTab`：致谢

## 技术要点

### 模块间通信机制

AltTab 使用多种通信机制：

1. **NotificationCenter**：
   - `NSWorkspace.notificationCenter`：监听空间切换（`activeSpaceDidChangeNotification`）
   - `NSWindow.willCloseNotification`：监听窗口关闭（如 Day1 欢迎窗口关闭后显示设置）
   - 自定义通知：`ProTransitionManager.proLockStateDidChangeNotification`

2. **Delegates**：
   - `App` 作为 `NSApplicationDelegate`
   - `MenubarMenuDelegate` 作为 `NSMenuDelegate`（菜单打开前刷新许可证状态）
   - `SparkleDelegate` 作为 `SPUStandardUpdaterController` 的代理

3. **Closures**：
   - `LicenseManager.shared.onStateChanged`：许可证状态变更回调
   - `LicenseManager.shared.onBeforeProUnlock`：Pro 解锁前回调
   - `ProTransitionManager.shared.onAction`：Pro 转化动作回调
   - `Throttler.throttleOrProceed(_:)`：节流回调

4. **SwitcherSession.current**：
   - 全局可变的会话状态单例，作为各模块间的共享状态载体
   - 通过 `SwitcherSession.isActive`（即 `current != nil`）判断切换器是否处于活动状态

5. **CGEvent Tap 回调**：
   - `KeyboardEvents.cgEventHandler`：低级别键盘事件回调
   - `AccessibilityEvents.axObserverCallback`：AX 通知回调

6. **CFMessagePort**：
   - `CliEvents` 通过 `CFMessagePortCreateLocal` 监听 CLI 命令
   - `CliClient` 通过 `CFMessagePortCreateRemote` 发送 CLI 命令

### 4 层节流保护

系统设计了 4 层节流来保护性能和系统资源：

| 层级 | 节流器 | 间隔 | 位置 | 作用 |
|------|--------|------|------|------|
| Layer 0 | `Applications.manualRefreshThrottler` | 1000ms | `Applications.swift` | 全局手动刷新节流，限制 `manuallyRefreshAllWindows()` 的调用频率 |
| Layer 1 | `AXCallScheduler` | 200ms | `AXCallScheduler.swift` | AX IPC 调用节流+重试+并发控制，按 key（pid 或 wid+事件桶）去重 |
| Layer 2 | `Throttler` / `ThrottlerWithKey` | 200ms | 多处 | 主线程 UI 更新节流（`refreshOpenUiThrottler`、`appListUpdateThrottler`、`windowListUpdateThrottler`） |
| Layer 3 | `Applications.captureThrottler` | 200ms | `Applications.swift` | 单窗口截图节流，按 WID 去重，优先级调度 |

**Layer 0 详细说明**：
- `manualRefreshThrottler`（1000ms）：`Applications.manuallyRefreshAllWindows()` 的全局限流，防止短时间内的多次完整窗口列表同步
- `badgesThrottler`（1000ms）：Dock 徽章刷新节流

**Layer 1 详细说明**：
- `AXCallScheduler` 是一个复杂的调度器，管理对 Accessibility API 的所有调用
- 每个 AX 调用通过 `schedule(key:pid:context:block:)` 提交
- 内部维护 `KeyState` 状态机（`idle` → `throttled` → `executing` → `retrying` → `idle`）
- 200ms 节流间隔（`throttleDelayNs = 200_000_000`）
- 失败时自动重试，指数退避：200ms → 1s → 2s → 5s → 5s...
- 60 秒后放弃（`giveUpAfterSeconds = 60.0`）
- 双队列设计：`fastQueue`（16 并发）处理正常进程，`retryQueue`（8 并发）处理无响应进程
- 无响应 PID 集合（`unresponsivePids`）：失败过的 PID 路由到 `retryQueue`

**Layer 2 详细说明**：
- `refreshOpenUiThrottler`（200ms，`Throttler`）：`App.refreshOpenUiAfterExternalEvent` 的主线程 UI 刷新节流
- `appListUpdateThrottler`（200ms，`ThrottlerWithKey`）：按 PID 节流应用列表更新
- `windowListUpdateThrottler`（200ms，`ThrottlerWithKey`）：按 WID+事件桶（focus/geometry/generic）节流窗口列表更新
- `SpacesEvents.throttler`（200ms）：空间切换事件节流
- `ScreensEvents.throttler`（200ms）：屏幕配置变更节流
- `Throttler` 保证在节流期间至少执行最后一次调用（tail scheduling）

**Layer 3 详细说明**：
- `captureThrottler`（200ms，`ThrottlerWithKey`）：按 `capture-wid-{WID}` 键节流每个窗口的截图频率
- 支持优先级调度（`prioritizedIds`）：视口内窗口截图优先级更高
- 截图请求在 `BackgroundWork.screenshotsQueue`（8 并发）上执行

### 事件桶分桶策略

Accessibility 事件通过 `bucket(for:)` 函数分为三类桶：
- **focus**：`kAXMainWindowChangedNotification`、`kAXFocusedWindowChangedNotification`
- **geometry**：`kAXWindowResizedNotification`、`kAXWindowMovedNotification`
- **generic**：其他所有事件

这确保了 burst 性质的事件（如拖动窗口产生的连续 resize 事件）不会覆盖焦点变更事件（后者更新 `Window.lastFocusOrder`）。

## 线程架构

### 4 个专用 RunLoop 线程

AltTab 使用 4 个专用 `BackgroundThreadWithRunLoop` 线程，用于需要 RunLoop 的 API：

| 线程 | 名称 | 用途 | 启动时机 |
|------|------|------|----------|
| `accessibilityEventsThread` | "axEvents" | AX Observer 事件源（`AXObserverGetRunLoopSource`） | `BackgroundWork.start()` |
| `keyboardAndMouseAndTrackpadEventsThread` | "inputDevices" | CGEvent Tap（`CGEvent.tapCreate`） | `BackgroundWork.start()` |
| `missionControlThread` | "missionControl" | Mission Control 状态监控 | `BackgroundWork.start()` |
| `cliEventsThread` | "cliMessages" | CFMessagePort 事件源（`CFMessagePortCreateRunLoopSource`） | `BackgroundWork.start()` |

**`BackgroundThreadWithRunLoop`** 实现细节：
- 继承 `Thread`，在 `main()` 中初始化 `CFRunLoopGetCurrent()` 并调用 `CFRunLoopRun()`
- 使用信号量（`threadStartSemaphore`）确保初始化同步完成
- 添加 dummy RunLoop Source 防止 RunLoop 在无真实 Source 时立即退出

### 8 个 OperationQueue

| 队列 | 名称 | 并发数 | QoS | 用途 |
|------|------|--------|-----|------|
| `screenshotsQueue` | "screenshots" | 8 | .userInteractive | 窗口截图捕获 |
| `accessibilityCommandsQueue` | "axCommands" | 4 | .userInteractive | AX 命令（聚焦/关闭/最小化等窗口操作） |
| `AXCallScheduler.fastQueue` | "axCallsFast" | 16 | .userInteractive | AX API 调用（正常进程） |
| `AXCallScheduler.retryQueue` | "axCallsRetry" | 8 | .userInteractive | AX API 调用（无响应进程重试） |
| `repeatingKeyQueue` | "repeatingKey" | 1 | .userInteractive | 按键重复定时器（串行） |
| `crashReportsQueue` | "crashReports" | 1 | .utility | 崩溃报告发送 |
| `permissionsCheckOnTimerQueue` | "permissionsCheckOnTimer" | 1 | .userInteractive | 权限检查定时器 |
| `permissionsSystemCallsQueue` | "permissionsSystemCalls" | 1 | .userInteractive | 权限 API 调用（串行，减轻系统压力） |

**`LabeledOperationQueue`** 实现细节：
- 继承 `OperationQueue`，强持有底层 `DispatchQueue`（防止被释放）
- 使用 `OSAtomicIncrement32/Decrement32` 跟踪活跃回调数
- 支持 `addOperationAfter(deadline:block:)` 延迟执行
- 线程计数保护：`totalPotentialThreadCount` 上限 45（macOS 进程软限制 64 线程）

### 事件流转路径

**键盘事件流**：
1. CGEvent Tap 在 `keyboardAndMouseAndTrackpadEventsThread` 上捕获
2. `cgEventHandler` 回调处理 `flagsChanged` / `keyDown` 事件
3. 通过 `DispatchQueue.main.async` 投递到主线程
4. 主线程执行 `handleKeyboardEvent` → `App.showUiOrCycleSelection` / `App.focusSelectedWindow`

**AX 事件流**：
1. AX Observer 在 `accessibilityEventsThread` 上接收通知
2. `axObserverCallback` 通过 `AXCallScheduler.shared.submit` 提交到 fastQueue
3. 在 fastQueue 线程上执行 AX 属性查询
4. 通过 `DispatchQueue.main.async` 投递到主线程
5. 主线程通过 `ThrottlerWithKey` 节流后更新 Model 和 View

**截图事件流**：
1. `WindowThumbnails.refreshAsync` 在主线程触发
2. 截图操作投递到 `screenshotsQueue`（8 并发）
3. 优先使用 ScreenCapture Kit（macOS 14+），回退到私有 API `CGSHWCaptureWindowList`
4. `Applications.captureThrottler` 按 WID 节流（200ms）
5. 截图完成后通过 `DispatchQueue.main.async` 更新 View

## 性能与优化

### App Nap 阻止

通过 `ProcessInfo.processInfo.beginActivity(options: .userInitiatedAllowingIdleSystemSleep)` 阻止 App Nap，保持应用响应性。同时允许系统空闲睡眠。

### 线程安全

- **`os_unfair_lock`**：`ConcurrentMap` 和 `ConcurrentArray` 使用轻量级 `os_unfair_lock`（比 `NSLock` 快约 10 倍）
- **`OSAtomic`**：`ActiveWindowCaptures` 和 `LabeledOperationQueue` 使用 `OSAtomicIncrement32/Decrement32` 进行原子计数
- **`NSLock`**：`AXCallScheduler` 使用 `NSLock` 保护 `keyStates` 字典
- **主线程 Model 访问**：所有 Model 数据的读写都在主线程进行，后台线程通过 `DispatchQueue.main.async` 回调更新

### 延迟初始化

- `SettingsWindow`、`AboutWindow`、`FeedbackWindow`、`DebugWindow`、`PermissionsWindow` 均为懒加载，首次访问时创建
- `SPUStandardUpdaterController` 延迟 30 秒启动，避免启动时的资源竞争

### 截图优化

- **缓存 SCShareableContent**：`WindowCaptureScreenshots.cachedSCWindows` 缓存窗口列表，避免每次截图都调用 `SCShareableContent.getExcludingDesktopWindows`
- **视口优先**：`TilesView.windowIdsInViewport()` 返回可见窗口 ID，截图时优先处理
- **按需分辨率**：仅在启用预览时使用全分辨率截图，否则按缩略图最大尺寸缩放
- **僵尸窗口回收**：`removeZombieWindows()` 通过 `CGWindowListCreateDescriptionFromArray` 检测已销毁但未收到通知的窗口

### 窗口列表同步

`Applications.manuallyRefreshAllWindows()` 执行四步同步：
1. `removeZombieWindows()`：移除不存在的窗口
2. `addMissingWindows()`：添加未发现的窗口
3. `reviewExistingWindows()`：刷新已知窗口的属性
4. `refreshIsInvisible()`：检测幽灵窗口（AX 报告存在但实际不可见）

### 日志系统

`Logger` 支持多级别日志（debug/info/warning/error），日志级别可通过 CLI 参数 `--logs=` 配置。在 `#if DEBUG` 模式下有额外的 QA 菜单和调试窗口。

## 文件清单

### 入口与协调
| 文件 | 职责 |
|------|------|
| `src/main.swift` | 应用入口，CLI 检测，信号处理，异常捕获 |
| `src/App.swift` | 中央协调器，NSApplicationDelegate，UI 显示/隐藏/刷新控制 |
| `src/MainMenu.swift` | macOS 标准菜单栏构建和快捷键管理 |
| `src/Menubar.swift` | 状态栏图标和下拉菜单，许可证状态 UI |

### 切换器 UI（`src/switcher/main-window/`）
| 文件 | 职责 |
|------|------|
| `TilesPanel.swift` | 主面板窗口（NSPanel） |
| `TilesPanelBackgroundView.swift` | 面板背景（毛玻璃效果） |
| `TilesView.swift` | 滚动视图，Tile 布局和回收 |
| `TileView.swift` | 单个窗口卡片视图 |
| `TileTitleView.swift` | 窗口标题标签 |
| `TileOverView.swift` | 悬停覆盖层 |
| `TileUnderLayer.swift` | 底层装饰 |
| `TileFontIconView.swift` | 字体图标视图 |
| `StatusIconsView.swift` | 状态图标（最小化、全屏等） |

### 状态管理（`src/switcher/state/`）
| 文件 | 职责 |
|------|------|
| `Application.swift` | 应用模型 |
| `Applications.swift` | 应用列表管理，4 层节流定义 |
| `Window.swift` | 窗口模型 |
| `Windows.swift` | 窗口列表管理，排序和选择 |
| `WindowThumbnails.swift` | 截图调度和预览 |
| `Spaces.swift` | 空间管理 |
| `Screens.swift` | 屏幕管理 |
| `TabGroup.swift` | 标签页组管理 |
| `ApplicationDiscriminator.swift` | 应用过滤器 |
| `WindowDiscriminator.swift` | 窗口过滤器 |

### 切换器辅助（`src/switcher/`）
| 文件 | 职责 |
|------|------|
| `SwitcherSession.swift` | 切换器会话状态 |
| `Appearance.swift` | 外观主题管理 |
| `PreviewPanel.swift` | 预览面板 |
| `Search.swift` | 搜索功能 |
| `ATShortcut.swift` | 快捷键抽象 |
| `ShortcutAction.swift` | 快捷键动作绑定 |
| `KeyRepeatTimer.swift` | 按键重复定时器 |

### 事件系统（`src/events/`）
| 文件 | 职责 |
|------|------|
| `AccessibilityEvents.swift` | AX Observer 事件处理 |
| `KeyboardEvents.swift` | 全局键盘事件（CGEvent Tap + Carbon HotKey） |
| `TrackpadEvents.swift` | 触控板手势事件 |
| `CursorEvents.swift` | 光标事件（悬停、死区） |
| `ContextMenuEvents.swift` | 右键菜单事件 |
| `CliEvents.swift` | CLI 命令通信（CFMessagePort） |
| `SpacesEvents.swift` | 空间切换事件 |
| `ScreensEvents.swift` | 屏幕配置变更事件 |
| `RunningApplicationsEvents.swift` | 应用启动/退出事件 |
| `WindowCaptureEvents.swift` | 窗口截图事件 |
| `PreferencesEvents.swift` | 偏好设置变更事件 |
| `SystemAppearanceEvents.swift` | 系统外观变更 |
| `SystemScrollerStyleEvents.swift` | 滚动条样式变更 |
| `InputSourceEvents.swift` | 输入法切换 |
| `SleepWakeEvents.swift` | 睡眠/唤醒事件 |
| `ScrollwheelEvents.swift` | 滚轮事件 |
| `DockEvents.swift` | Dock 事件 |
| `UserDefaultsEvents.swift` | UserDefaults KVO |

### 后台工作（`src/util/`）
| 文件 | 职责 |
|------|------|
| `BackgroundWork.swift` | 线程和队列管理器，`BackgroundThreadWithRunLoop`、`LabeledOperationQueue` 定义 |
| `Throttler.swift` | 节流器（`Throttler`、`ThrottlerWithKey`、`ConcurrentMap`、`ConcurrentArray`） |
| `UsageStats.swift` | 使用统计 |
| `MoveToApplicationsFolder.swift` | 移动到 Applications 文件夹提示 |

### macOS API 封装（`src/macos/`）
| 文件 | 职责 |
|------|------|
| `AXCallScheduler.swift` | AX API 调用调度器（节流、重试、退避） |
| `SystemPermissions.swift` | 系统权限检查 |
| `LoginItem.swift` | 开机启动项管理 |
| `api-wrappers/AXUIElement.swift` | AXUIElement 扩展 |
| `api-wrappers/CGWindow.swift` | CGWindow API 封装 |
| `api-wrappers/SkyLight.framework.swift` | 私有 SkyLight 框架调用 |
| `api-wrappers/Logger.swift` | 日志系统 |
| `api-wrappers/MissionControl.swift` | Mission Control 状态查询 |

### 偏好设置（`src/preferences/`）
| 文件 | 职责 |
|------|------|
| `Preferences.swift` | 偏好设置核心 |
| `PreferenceDefinition.swift` | 偏好项定义 |
| `PreferencesMigrations.swift` | 版本迁移 |
| `ShortcutConfiguration.swift` | 快捷键配置 |
| `settings-window/SettingsWindow.swift` | 设置窗口 |
| `settings-window/tabs/GeneralTab.swift` | 常规设置 Tab |
| `settings-window/tabs/ControlsTab.swift` | 快捷键控制 Tab |
| `settings-window/tabs/appearance/` | 外观设置 Tab |
| `settings-window/tabs/ExceptionsTab.swift` | 例外设置 |
| `settings-window/tabs/UpgradeTab.swift` | Pro 升级 Tab |

### 二级窗口（`src/secondary-windows/`）
| 文件 | 职责 |
|------|------|
| `DebugWindow.swift` | 调试窗口 |
| `FeedbackWindow.swift` | 反馈窗口 |
| `permission-window/` | 权限窗口 |

### Pro 系统（`src/pro/`）
| 文件 | 职责 |
|------|------|
| `license/LicenseManager.swift` | 许可证管理 |
| `license/LicenseState.swift` | 许可证状态枚举 |
| `license/LicenseAPI.swift` | 许可证 API |
| `scheduling/ProTransitionManager.swift` | Pro 转化管理器 |
| `scheduling/ProTransitionScheduler.swift` | Pro 转化调度（Day1/4/12/15/21/35 提示） |
| `ui/ProPromptHost.swift` | Pro 提示宿主 |
| `ui/ProGradientButton.swift` | 渐变按钮 |

### 工具库（`src/kit/`）
| 文件 | 职责 |
|------|------|
| `StackView.swift` | 自定义堆栈视图 |
| `Button.swift` | 自定义按钮 |
| `Tooltips.swift` | 提示气泡 |
| `TrafficLightButton.swift` | 红绿灯按钮 |
| `GridView.swift` | 网格视图 |
| `Switch.swift` | 开关控件 |
| `text/` | 文本相关组件 |

### 第三方集成（`src/vendors/`）
| 文件 | 职责 |
|------|------|
| `AppCenterApplication.h/m` | AppCenter 崩溃报告 Obj-C 桥接 |
| `AppCenterCrashes.swift` | AppCenter 崩溃报告配置 |
| `SparkleDelegate.swift` | Sparkle 自动更新代理 |
