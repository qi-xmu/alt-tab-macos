# 基础设施

## 模块职责

基础设施层为 AltTab 提供底层支撑能力，涵盖后台线程与队列管理、事件节流去重、使用统计、调试工具、API 端点与密钥、第三方集成、次要窗口以及实验性代码。这些模块大多不直接面向用户，而是为上层的窗口切换器、应用管理和事件处理提供运行时环境。

本层的设计遵循几个核心原则：

- **RunLoop 线程优先于 GCD**：当 API（如 `CGEvent.tapCreate`、`AXObserverGetRunLoopSource`、`CFMessagePortCreateRunLoopSource`）需要 RunLoop 时，使用专用线程；否则使用 `OperationQueue` 以获得更好的并行控制。
- **线程预算管理**：macOS 进程软限制为 64 个线程，AltTab 通过 `addPotentialThreadCount` 断言总线程数不超过 45。
- **最小化网络权限**：AppCenter 的 `networkRequestsAllowed` 默认关闭，仅在发送崩溃报告时短暂开启，发送完毕后立即关闭。

---

## 后台工作系统

`BackgroundWork`（`src/util/BackgroundWork.swift`）是 AltTab 并发架构的核心，管理两类执行载体：**RunLoop 专用线程** 和 **OperationQueue**。

### RunLoop 专用线程

4 个 `BackgroundThreadWithRunLoop` 实例在 `start()` 中创建，每个都运行独立的 `CFRunLoop`：

| 线程名 | QoS | 用途 | RunLoop Source |
|--------|-----|------|----------------|
| `axEvents` | `.userInteractive` | 接收 Accessibility 通知（窗口创建/销毁/标题变更等） | `AXObserverGetRunLoopSource`（每窗口、每应用各一个） |
| `inputDevices` | `.userInteractive` | 监听键盘、鼠标、触控板事件 | `CFMachPortCreateRunLoopSource`（来自 `CGEvent.tapCreate`） |
| `missionControl` | `.userInteractive` | 追踪 Mission Control 状态变化 | `AXObserverGetRunLoopSource`（Dock 应用的 AXObserver） |
| `cliMessages` | `.userInteractive` | 接收 CLI 命令（`CFMessagePort` 跨进程通信） | `CFMessagePortCreateRunLoopSource` |

**线程启动机制**：`BackgroundThreadWithRunLoop.init()` 中调用 `start()` 后，通过 `DispatchSemaphore` 阻塞初始化线程，直到新线程的 `main()` 方法完成 RunLoop 初始化并 `signal()`，确保构造完成后 RunLoop 已就绪。

**保活机制**：每个线程添加一个空的 `CFRunLoopSource`（`perform` 回调为空函数），防止 RunLoop 在实际 Source 注册前退出。

### OperationQueue 队列

6 个 `LabeledOperationQueue` 实例分布在 `preStart()` 和 `start()` 两个阶段：

| 队列名 | QoS | 最大并发 | 用途 |
|--------|-----|---------|------|
| `permissionsCheckOnTimer` | `.userInteractive` | 1 | 定时器触发的权限检查 |
| `permissionsSystemCalls` | `.userInteractive` | 1 | 权限 API 调用（串行化以减轻系统压力） |
| `screenshots` | `.userInteractive` | 8 | 窗口截图捕获（高并发，并行渲染缩略图） |
| `axCommands` | `.userInteractive` | 4 | Accessibility 命令（focus、close、minimize 等），超时不重试 |
| `repeatingKey` | `.userInteractive` | 1 | 按键重复计时（串行，结果回主线程处理） |
| `crashReports` | `.utility` | 1 | 崩溃报告上传（低优先级） |

**`LabeledOperationQueue`** 继承自 `OperationQueue`，增加了：
- `strongUnderlyingQueue`：强引用底层 `DispatchQueue`（防止被 ARC 回收）
- `activeCallbacks`：通过 `OSAtomicAdd32` 原子计数器追踪正在执行的回调数量
- `addOperationAfter(deadline:block:)`：基于 `DispatchQueue.asyncAfter` 的延迟操作调度
- `trackCallbacks(_:)`：内联标记回调执行中的计数（用于 `DebugMenu` 的队列监控）

### 线程调试工具

`logThreadsAndQueuesOnRepeat()` 在开发阶段提供线程与队列的实时监控：
- `logThreads()` 使用 `task_threads` Mach API 枚举所有线程，通过 `pthread_from_mach_thread_np` + `pthread_getname_np` 获取线程名
- `logQueues()` 统计每个 OperationQueue 中正在执行的 operation 数量
- `DebugMenu`（`src/secondary-windows/DebugMenu.swift`）提供实时队列活动图表面板

---

## 节流器

### Throttler（无键节流）

`Throttler`（`src/util/Throttler.swift`）是最简形式的节流器，无键区分。通过 `throttleOrProceed(_:)` 方法：

1. 检查距离上次执行是否超过 `delayInNanoseconds`
2. 超过则立即执行并更新时间戳
3. 未超过则调度一个延迟回调（额外加 10ms 缓冲），通过 `nextScheduled` 标志防止重复调度

**约束**：`dispatchPrecondition(condition: .onQueue(.main))` 确保只在主线程调用。

### ThrottlerWithKey（带键去重节流）

`ThrottlerWithKey` 是按 key 独立节流的变体，内部使用 `ConcurrentMap<String, ThrottleState>` 管理每个 key 的状态：

```swift
struct ThrottleState {
    let time: UInt64
    var tailScheduled: Bool
}
```

**尾部调用（tail coalescing）**：当某个 key 的请求在节流窗口内到达时，不会立即丢弃，而是将 `tailScheduled` 标记为 `true`，并在窗口剩余时间后调度一次"尾部"执行。这确保高频事件中最后一个请求不会被丢失。

**队列感知**：`throttleOrProceed(key:queue:priority:_:)` 支持传入 `LabeledOperationQueue`，将执行体包装为 `BlockOperation` 并设置优先级。不传队列时在调用方队列上执行。

**清理方法**：`removeEntry(withKey:)`、`removeEntries(withSuffix:)`、`removeEntries(withPrefix:)` 用于在窗口/应用被移除时清理对应的节流状态。

### ConcurrentMap 和 ConcurrentArray

这两个泛型容器使用 `os_unfair_lock` 实现轻量级线程安全：

- 分配在堆上的 `os_unfair_lock`（通过 `UnsafeMutablePointer`），在 `deinit` 中手动释放
- `withLock(_:)` 方法提供范围锁，返回闭包的计算结果
- 比传统 `NSLock` 快约 10 倍（单次原子 CAS，无 ObjC dispatch 开销）

### AXCallScheduler 的关系

`AXCallScheduler`（`src/macos/AXCallScheduler.swift`）是专门为 Accessibility API 调用设计的调度器，与 `ThrottlerWithKey` 互补：

- **双队列架构**：`fastQueue`（16 并发）处理正常 AX 调用，`retryQueue`（8 并发）处理无响应进程的重试
- **状态机**：每个 key 有 4 个阶段（`idle` → `throttled` → `executing` → `retrying`），200ms 节流窗口
- **指数退避重试**：200ms → 1s → 2s → 5s → 5s，60 秒总超时后放弃
- **无响应进程隔离**：一旦某个 PID 的 AX 调用失败，后续调用自动路由到 `retryQueue`，避免阻塞 `fastQueue`

---

## 使用统计

### UsageStats

`UsageStats`（`src/util/UsageStats.swift`）追踪用户的功能使用行为，为 Pro 功能转化提供数据基础。

**存储**：使用独立的 `UserDefaults` suite（`\(App.bundleIdentifier).usage`），通过 `writeQueue`（`DispatchQueue`，QoS `.utility`）串行化写入，避免阻塞主线程。

**追踪指标**（6 个 key）：

| Key | 含义 |
|-----|------|
| `triggers` | 窗口切换器触发次数 |
| `triggersAppIcons` | 使用 App Icons 外观的触发 |
| `triggersTitles` | 使用 Titles 外观的触发 |
| `triggersAutoSize` | 使用 Auto Size 的触发 |
| `triggersExtraShortcuts` | 使用额外快捷键的触发 |
| `searches` | 搜索功能使用次数 |

**记录机制**：每个事件记录 Unix 时间戳（秒级整数），存储为 `[Int]` 数组。`triggers` 与 `triggersAppIcons`/`triggersTitles`/`triggersAutoSize`/`triggersExtraShortcuts` 共享同一时间戳（在同一个 `recordTrigger` 调用中记录）。

**Pro 功能会话计数**：`usedProFeaturesSessionCount` 通过 `UsageStatsTestable.proFeatureSessionCount` 计算。算法将所有 Pro 功能的时间戳合并到一个 `Set<Int>` 中，与 `triggers` 取交集，得到"使用了至少一项 Pro 功能的触发次数"。搜索的时间戳通过二分查找（`mostRecentTrigger`）映射回最近的触发时间戳。

**查询 API**：
- `usedAppIconsOrTitles()` / `usedSearch()` / `usedAutoSize()` / `usedExtraShortcuts()` — 判断是否曾使用某功能
- `usedProFeatureNames()` — 返回已使用的 Pro 功能名称列表（用于个性化文案）
- `count(_:since:)` — 计算自某日期以来的事件数量

**数据清理**：`prune()` 移除超过 365 天的时间戳数据。

### UsageStatsTestable

`UsageStatsTestable`（`src/util/UsageStatsTestable.swift`）将 Pro 功能判定逻辑抽为纯函数，便于单元测试。核心方法 `proFeatureSessionCount` 接受原始时间戳数组作为参数，不依赖任何全局状态。

---

## 调试工具

### BenchmarkRunner

`BenchmarkRunner`（`src/debug/Benchmark.swift`）通过命令行参数 `--benchmark <mode>` 驱动自动化性能测试：

- **`launch` 模式**：启动后等待 `startupDelay`（5 秒）+ `launchDuration`（10 秒），然后终止进程，用于测量冷启动时间
- **`showUi <count>` 模式**：交替调用 `App.showUi` / `App.hideUi`，每次显示 `showDuration`（500ms）、隐藏 `hideDuration`（500ms），执行指定次数后终止，用于测量 UI 循环性能

### DebugWindow

`DebugWindow`（`src/secondary-windows/DebugWindow.swift`）是开发者的瑞士军刀，提供两大功能模块：

**日志查看器**：
- 通过 `Logger.setTap` 拦截所有日志输出，按级别（Debug/Info/Warning/Error）分色显示
- 级别过滤（`NSSegmentedControl`）和 `WindowDiscriminator` 专项过滤
- 自动滚动（用户手动滚动时暂停，滚动到底部时恢复）
- 窗口打开时 `Logger.minLevel` 强制设为 `.debug`，关闭时恢复

**窗口检查器（Inspect）**：
- 以 100ms 间隔读取鼠标位置下的窗口信息
- 三列并排显示：App（名称、BundleID、PID）、CG API（标题、WID、层级、尺寸、透明度）、AX API（标题、角色、子角色、最小化、全屏）
- 点击任意位置停止检查

### DebugMenu

`DebugMenu`（`src/secondary-windows/DebugMenu.swift`）是浮动面板，以 100ms 采样率绘制 `LabeledOperationQueue` 的实时活动曲线图：
- 60 秒滑动窗口
- 自动缩放 Y 轴
- Core Text 渲染坐标轴和图例
- 按队列名着色（HSL 均匀分布）

### QAMenu

`QAMenu`（`src/debug/QAMenu.swift`）仅在 `#if DEBUG` 条件下编译，是 QA 团队的交互式测试面板：

**基础功能**：
- 语言切换下拉框（覆盖 `AppleLanguages` UserDefaults 后重启）
- "Settings..."按钮、"Open on launch"复选框、Quit 按钮

**Pro Transition 测试区**（可折叠）：
- Mock Day 按钮（1/13/15/21/35/Pro）：模拟试用期的天数推进，自动标记已看过的引导步骤
- Show window/popover 按钮：触发 Day1 Welcome（新安装/升级两种路径）、Day4 Tour、Day12 HeadsUp、Day15 FullUpgrade/Proactive/HardGate、Day21 Reminder、Day35 Final
- Reset 按钮：Free Pass、Day4 Tour、Switcher Trigger、Toggle Opt-Out、Revalidate、Mock fresh install
- `mockFreshInstall()` 清除所有 UserDefaults suite 和 Keychain 条目，模拟全新安装

### DebugProfile

`DebugProfile`（`src/secondary-windows/DebugProfile.swift`）生成标准化的调试信息报告，附加在崩溃报告和用户反馈中：

- 应用信息：名称、版本、许可证状态
- 偏好设置：所有键值对的完整转储
- 运行时状态：应用数量、窗口数量
- 操作系统：架构、语言、输入法、Space 数量、暗色模式、独立空间设置
- 硬件：型号、屏幕（含 DPI）、CPU、内存
- 资源占用：通过 `top` 命令获取 CPU/内存/线程数

---

## API 与第三方集成

### Endpoints

`Endpoints`（`src/api/Endpoints.swift`）集中管理所有 URL，域名从 `Info.plist` 的 `Domain`/`ApiDomain` 键读取（xcconfig 构建时注入）：

| 属性 | 用途 |
|------|------|
| `website` | 官网 `https://<domain>` |
| `appcastUrl` | Sparkle 更新源的 appcast XML |
| `supportUrl` | 支持页面 |
| `checkoutUrl` | 定价/购买页面 |
| `accountUrl` | 用户账户管理页面 |
| `licenseApiBaseUrl` | 许可证验证 API `https://<apiDomain>/v1/license` |
| `feedbackUrl` | 反馈提交 API `https://<apiDomain>/v1/feedback` |

### Secrets

`Secrets`（`src/api/Secrets.swift`）从 `Info.plist` 读取 `AppCenterSecret`，由 xcconfig 在构建时注入，不在源码中硬编码。

### AppCenter 崩溃报告

`AppCenterCrash`（`src/vendors/AppCenterCrashes.swift`）封装 Microsoft AppCenter 的崩溃报告功能：

**初始化流程**：
1. 注册 `NSApplicationCrashOnExceptions = true` 捕获主线程未处理异常
2. 设置 `AppCenter.networkRequestsAllowed = false`（默认禁止网络）
3. 先注册 `Crashes.delegate` 和 `userConfirmationHandler`，再调用 `AppCenter.start`（因为 `start` 内部同步处理待发送的崩溃报告）

**崩溃报告发送策略**：
- `confirmationHandler` 返回 `true`（承诺会弹窗询问用户），但实际弹窗延迟到下一个 RunLoop tick（避免在启动 UI 未完成时触发嵌套模态循环导致的 `_NSDetectedLayoutRecursion`）
- 在 `crashReportsQueue`（`.utility` QoS）中切换 `networkRequestsAllowed`，调用 `Crashes.notify`
- 发送成功/失败后立即重新关闭网络权限

**附件**：`attachments(with:for:)` 自动附加 `DebugProfile.make()` 生成的 `debug-profile.md` 文件。

**用户策略**：`crashPolicy` 偏好支持 "每次询问"/"总是发送"/"从不发送"三种模式。

### Sparkle 自动更新

`SparkleDelegate`（`src/vendors/SparkleDelegate.swift`）实现 `SPUUpdaterDelegate`：

**会话级缓存**：`cachedResult` 存储最近一次更新检查结果（`.updateAvailable(SUAppcastItem)` 或 `.upToDate`），仅存在于内存中，每次启动重新检查。

**一次性回调**：`onNextCheckCompletion` 用于驱动 UI（如反馈窗口的加载指示器），触发后清空。

**feed 参数**：`feedParameters(for:sendingSystemProfile:)` 附加版本号、macOS 版本、CPU 架构、首选语言到更新检查请求。

---

## 次要窗口

### FeedbackWindow

`FeedbackWindow`（`src/secondary-windows/FeedbackWindow.swift`）是用户反馈提交窗口，支持 Bug 报告和功能建议两种类型。

**双状态设计**：
- **State A — Kind Picker**：显示 Bug 和 Enhancement 两张卡片（`FeedbackKindCard`），底部有 GitHub Issues 链接
- **State B — Form**：标题输入框、正文文本域、"Create GitHub issue"按钮、"Go back"按钮

**草稿管理**：每种类型（bug/enhancement）维护独立的 `Draft`，在窗口关闭/返回选择器时自动保存，仅在提交成功（HTTP 201）后清除。

**Bug 报告更新检查**：选择 Bug 类型时触发 Sparkle 更新检查：
- 缓存命中则立即判断
- 缓存未命中则将 Bug 卡片变形为加载指示器，等待 Sparkle 回调或 5 秒超时
- 如果有更新可用，弹出提示建议先更新（"你遇到的 Bug 可能已修复"）
- 使用 `updateCheckGeneration` 计数器使过期的回调失效

**提交**：POST 到 `Endpoints.feedbackUrl`，请求体包含 title、body、kind 和 `DebugProfile.make()` 生成的 debug-profile。

**键盘支持**：Cmd+Return 提交（`performKeyEquivalent` 拦截），Escape 关闭窗口。

### PermissionsWindow

`PermissionsWindow`（`src/secondary-windows/permission-window/PermissionsWindow.swift`）是首次启动时的权限请求窗口。

**两个权限卡片**：
- **Accessibility**（必需）：用于切换后聚焦窗口
- **Screen Recording**（macOS 10.15+，可选）：用于显示窗口缩略图和预览，附带"跳过"复选框

**关闭守卫**：`windowShouldClose` 在权限未通过时阻止关闭并终止应用（`App.shared.terminate`），确保用户必须授权。

### PermissionView

`PermissionView`（`src/secondary-windows/permission-window/PermissionView.swift`）是单个权限的视觉组件：

- SF Symbol 图标 + 标题 + 理由说明 + "Open Settings"按钮 + 状态指示
- 三种状态：`.granted`（绿色背景）、`.notGranted`（红色背景）、`.skipped`（黄色背景）
- 圆角卡片式布局

### MoveToApplicationsFolder

`MoveToApplicationsFolder`（`src/util/MoveToApplicationsFolder.swift`）替代了原 LetsMove CocoaPod（约 600 行 Obj-C），以约 190 行纯 Swift 实现相同功能：

**流程**：
1. 检查 `suppressKey` UserDefaults（用户选择不再提示）
2. 检查是否已在 Applications 文件夹中
3. 检测是否运行在 DMG 上（通过 `statfs` + `hdiutil info` 匹配设备节点）
4. 检查 `/Applications` 写权限
5. 弹出 NSAlert（含"不再提示"复选框）
6. 如目标已有运行实例，`open` 它并退出自己
7. 复制 bundle，strip `com.apple.quarantine` xattr，relaunch
8. 如在 DMG 上运行，relaunch 后 5 秒延迟 detach DMG

**弃用**：不支持 `AuthorizationExecuteWithPrivileges`（macOS 10.7 已弃用），也不支持 AppleScript trash fallback。

---

## 实验性代码

`src/experimentations/` 目录包含 4 个实验性文件，记录了功能探索的历史过程。这些文件大多不参与编译，而是作为调查文档和原型代码保存。

### GhostWindowDetection.swift

**背景**："幽灵窗口"是存在于 macOS API（AX 返回标准窗口且有合法 CGWindowID）但对用户不可见的窗口。常见来源：
- Microsoft Outlook 提醒（alpha=0 的 NSWindow）
- Electron `BrowserWindow({show: false})` / `.hide()`
- WeChat / Teams / DingTalk 通过 alpha=0 + 尺寸操作隐藏窗口

**检测结果**：使用 `Spaces.windowsInSpaces` 的两次查询（包含/排除 invisible 标记）产生两种强度的幽灵信号：
1. **最强信号**：WID 同时缺失于两个查询（CGS 完全丢失了窗口，如 Joplin 的不可见窗口）
2. **弱信号**：WID 在 allCgs 中但不在 visibleCgs 中（如 Outlook 的 alpha=0 窗口）

**历史意义**：v10.9 曾用此信号同时检测标签页（`isTabbed = !visibleCgsWindowIds.contains(wid)`），但因为"invisible"桶是幽灵窗口、非活动标签页、最小化窗口、其他 Space 窗口、隐藏应用的超集，导致大量误报。v10.10 引入 AX-based `AXTabGroup` 后（见 `TabbedWindowDetection.swift`），标签页检测变得确定性，幽灵窗口检测才安全地单独实施。

**消歧表**（代码顺序）：`wid not in allCgs` → 幽灵 → `wid in visibleCgs` → 不是幽灵 → `isMinimized` → 不是幽灵 → `application.isHidden` → 不是幽灵 → `isTabbed` → 不是幽灵 → `spaceIds 非空但与 visibleSpaces 无交集` → 不是幽灵 → 否则 → 幽灵。

### TabbedWindowDetection.swift

**背景**：macOS 的窗口标签功能（OS-level tabs）没有公共 API 来检测某个窗口是否是非活动标签页。

**实验总结**：测试了 10 种方法，均失败：
1. `CGWindowListCopyWindowInfo` — 非活动标签根本不出现
2. `CGSCopyWindowsWithOptionsAndTags` — 无法区分标签与隐藏窗口
3. `SCShareableContent`（macOS 13+）— 无区分信号
4. `SLSWindowIterator`（parentId, tags, attributes）— 标签窗口的 parentId 始终为 0
5. `SLSGetWindowTags`（64-bit）— 位模式与隐藏/其他 Space 窗口相同
6. `SLSGetWindowSubLevel` — 全部为 0
7. `CGSCopyWindowGroup`（movementGroup, orderingGroup, tabGroup）— 全部为空
8. `CGSCopyWindowProperty`（~40 个推测 key）— 只有 `kCGSWindowTitle` 有值
9. `SLSCopyAssociatedWindows` — 只返回自身
10. 基于帧的聚类 — 不可靠（标签化前的位置不固定）

**最终方案**：AX Accessibility API 的 `AXTabGroup`：
- 查询窗口 `AXUIElement` 的 `kAXChildrenAttribute`
- 找到 `kAXRoleAttribute == "AXTabGroup"` 的子元素
- 其子元素中 `kAXSubroleAttribute == "AXTabButton"` 的是标签按钮
- `kAXValueAttribute` 为 1 表示活动标签，0 表示非活动标签
- 非标签窗口没有 `AXTabGroup` 子元素

**实现**：见 `AXUIElement.tabGroupInfo()`（`src/api-wrappers/AXUIElement.swift`）。

### IOKit.swift

`IOKitPrototype` 是对 IOKit HID（Human Interface Device）API 的原型实验：

- 通过 `IOHIDManagerCreate` 创建 HID 管理器
- 匹配键盘设备（`kHIDPage_GenericDesktop` + `kHIDUsage_GD_Keyboard`）
- 注册 `IOHIDValueCallback` 到 `keyboardAndMouseAndTrackpadEventsThread` 的 RunLoop
- `keyCodeToString` 使用 `TISCopyCurrentKeyboardLayoutInputSource` + `UCKeyTranslate` 将扫描码转为字符
- `keyCodeToStringUsingCG` 使用 `CGEvent` 的 `keyboardGetUnicodeString` 方法

**限制**：TIS API 不是线程安全的，必须在主线程调用（代码中通过 `DispatchQueue.main.sync` 桥接）。

### PrivateApis.swift

记录了大量 macOS 私有 API 的探索结果，每个 API 都标注了可用版本和实验结论：

**Space 管理**：
- `CGSSpaceGetType` — 获取 Space 类型（user/system/fullscreen）
- `CGSMoveWorkspaceWindowList` / `CGSMoveWindowsToManagedSpace` — 移动窗口到指定 Space（不支持全屏窗口）
- `CGSManagedDisplaySetCurrentSpace` — 切换当前 Space（不触发动画，导致图形故障）
- `CGSShowSpaces` / `CGSHideSpaces` — 覆盖显示/隐藏 Space（窗口仅视觉存在，AX 无法访问）

**窗口操作**：
- `CGSCaptureWindowsContentsToRectWithOptions` — 截图（比 `CGWindowListCreateImage` 略快，但画质低）
- `CGSGetWindowBounds` — 获取窗口尺寸（比 AX 快）
- `CGSOrderWindow` — 改变窗口层级（`relativeToWindow=0` 时无效）
- `windowManagerDeferWindowRaise` / `windowManagerActivateWindow` / `windowManagerDeactivateWindow` — 从 Hammerspoon 移植的窗口管理器事件注入

**事件监听**：
- `SLSRegisterConnectionNotifyProc` — 注册 WindowServer 事件回调，有效事件范围涵盖 100-2099

**进程管理**：
- `_SLPSGetFrontProcess` — 获取前台进程 PSN
- `CGSGetWindowOwner` + `CGSGetConnectionPSN` — WID 到 PSN 的映射

---

## 文件清单

### 工具层 (`src/util/`)

| 文件 | 行数 | 核心类/结构体 | 职责 |
|------|------|-------------|------|
| `BackgroundWork.swift` | 177 | `BackgroundWork`, `BackgroundThreadWithRunLoop`, `LabeledOperationQueue`, `ConcurrentMap`, `ConcurrentArray` | 后台线程与队列管理、并发安全容器 |
| `Throttler.swift` | 157 | `Throttler`, `ThrottlerWithKey`, `ConcurrentMap`, `ConcurrentArray` | 事件节流去重 |
| `UsageStats.swift` | 83 | `UsageStats` | 使用统计记录与查询 |
| `UsageStatsTestable.swift` | 47 | `UsageStatsTestable` | 使用统计的纯函数测试辅助 |
| `MoveToApplicationsFolder.swift` | 189 | `MoveToApplicationsFolder` | 首次启动移至 Applications 文件夹 |

### 调试层 (`src/debug/`)

| 文件 | 行数 | 核心类/结构体 | 职责 |
|------|------|-------------|------|
| `Benchmark.swift` | 74 | `BenchmarkRunner`, `BenchmarkConfig` | 命令行驱动的性能基准测试 |
| `QAMenu.swift` | 252 | `QAMenu` | QA 交互测试面板（仅 DEBUG） |

### API 层 (`src/api/`)

| 文件 | 行数 | 核心类/结构体 | 职责 |
|------|------|-------------|------|
| `Endpoints.swift` | 13 | `Endpoints` | URL 和 API 端点集中管理 |
| `Secrets.swift` | 7 | `Secrets` | API 密钥管理（xcconfig 注入） |

### 第三方集成 (`src/vendors/`)

| 文件 | 行数 | 核心类/结构体 | 职责 |
|------|------|-------------|------|
| `AppCenterCrashes.swift` | 106 | `AppCenterCrash` | AppCenter 崩溃报告集成 |
| `SparkleDelegate.swift` | 54 | `SparkleDelegate` | Sparkle 自动更新代理 |

### 次要窗口 (`src/secondary-windows/`)

| 文件 | 行数 | 核心类/结构体 | 职责 |
|------|------|-------------|------|
| `DebugWindow.swift` | 410 | `DebugWindow` | 日志查看器与窗口检查器 |
| `DebugProfile.swift` | 117 | `DebugProfile` | 调试信息报告生成 |
| `DebugMenu.swift` | 220 | `DebugMenu`, `QueueGraphView` | 队列活动实时监控图表 |
| `FeedbackWindow.swift` | 569 | `FeedbackWindow`, `FeedbackKindCard` | 用户反馈提交窗口 |
| `permission-window/PermissionView.swift` | 73 | `PermissionView`, `PermissionStatus` | 单个权限的视觉组件 |
| `permission-window/PermissionsWindow.swift` | 111 | `PermissionsWindow` | 权限请求窗口 |

### 实验性代码 (`src/experimentations/`)

| 文件 | 行数 | 职责 |
|------|------|------|
| `GhostWindowDetection.swift` | 107 | 幽灵窗口检测调查文档 |
| `TabbedWindowDetection.swift` | 103 | 标签窗口检测调查文档 |
| `IOKit.swift` | 102 | IOKit HID API 原型实验 |
| `PrivateApis.swift` | 262 | macOS 私有 API 探索记录 |
