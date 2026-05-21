# macOS API 封装

## 模块职责

本模块是 AltTab 与 macOS 操作系统交互的基础层，封装了公开 API、私有 API 和系统框架的调用。核心职责包括：

1. **私有 API 声明**：通过 `@_silgen_name` 桥接 SkyLight 窗口服务器私有框架，获取公开 API 无法提供的窗口管理能力。
2. **Accessibility API 封装**：对 `AXUIElement` 系列调用进行超时控制、错误恢复和批量属性获取。
3. **AX 调度器**：`AXCallScheduler` 实现了 4 阶段状态机、双队列并发控制和指数退避重试，是应对 macOS AX 子系统不稳定的核心性能组件。
4. **系统权限管理**：处理辅助功能（Accessibility）和屏幕录制两项隐私权限的检测、轮询和回调。
5. **辅助工具**：日志系统、Shell 命令执行、系统控制接口、Mission Control 状态检测、轻量计时器等。

**为什么需要私有 API？** macOS 公开 API（`CGWindowListCopyWindowInfo`、`NSWorkspace` 等）存在以下不足：

- **Space 管理**：公开 API 无法获取窗口所在的 Space ID、无法将窗口移动到指定 Space。`CGSCopyManagedDisplaySpaces`、`CGSCopySpacesForWindows` 等私有 API 弥补了这一空白。
- **窗口截图**：公开的 `CGWindowListCreateImage` 对最小化窗口和跨 Space 窗口截图效果不佳，而 `CGSHWCaptureWindowList` 可以稳定捕获。
- **窗口层级**：公开 API 无法获取窗口的 z-index 排序信息，`CGSCopyWindowsWithOptionsAndTags` 按层级返回窗口列表。
- **进程前置**：`_SLPSSetFrontProcessWithOptions` 和 `SLPSPostEventRecordTo` 提供了精确控制窗口焦点的能力，公开 API `NSRunningApplication.activate()` 粒度不足。
- **系统快捷键**：`CGSSetSymbolicHotKeyEnabled` 允许动态启用/禁用 Command+Tab 等系统快捷键。
- **屏幕录制权限**：`SLSRequestScreenCaptureAccess` 在首次调用时向用户弹出授权提示。

私有 API 通过 SDK 中的符号名逆向工程获得签名，来源包括 WebKit 仓库、CGSInternal 项目和社区逆向分析。详见 `src/macos/api-wrappers/README.md`。

---

## 核心组件

### 文件总览

| 文件 | 职责 |
|------|------|
| `SkyLight.framework.swift` | SkyLight 私有框架函数声明，Window Server 交互 |
| `AXUIElement.swift` | Accessibility API 封装，批量属性获取，窗口枚举 |
| `CGWindow.swift` | CGWindow 字典类型别名和查询扩展 |
| `CGWindowID.swift` | CGWindowID 扩展，标题/层级/Space 查询 |
| `ApplicationServices.HIServices.framework.swift` | HIServices 私有 API 桥接，常量补充 |
| `HelperExtensions.swift` | 通用扩展方法（NSColor、CALayer、NSView 等） |
| `HelperExtensionsTestable.swift` | 测试辅助：闭包 Action、CGImage 缩放 |
| `Logger.swift` | 4 级日志系统，ANSI 彩色输出 |
| `Markdown.swift` | 简易 Markdown 转 NSAttributedString 渲染器 |
| `Bash.swift` | Shell 命令同步执行 |
| `Sysctl.swift` | BSD sysctl 系统参数查询封装 |
| `LightweightTimer.swift` | 基于 `mach_absolute_time` 的轻量计时器 |
| `MissionControl.swift` | Mission Control 活跃状态检测 |
| `AXCallScheduler.swift` | AX 调用调度器（核心性能组件） |
| `LoginItem.swift` | 登录项（开机启动）管理 |
| `SystemPermissions.swift` | 系统权限检测与轮询 |

---

## 私有 API 概览

### 声明方式

AltTab 通过 Swift 的 `@_silgen_name` 属性声明私有 API，绕过标准框架链接，直接以 C 函数符号名调用。例如：

```swift
@_silgen_name("CGSMainConnectionID")
func CGSMainConnectionID() -> CGSConnectionID
```

所有私有 API 函数的签名来自逆向工程，属于 best-effort 还原，Apple 不提供官方文档。

### 核心类型

| 类型 | 定义 | 用途 |
|------|------|------|
| `CGSConnectionID` | `UInt32` | 与 WindowServer 的连接标识符 |
| `CGSSpaceID` | `UInt64` | macOS Space 的唯一标识符 |
| `CGSWindowCaptureOptions` | `OptionSet<UInt32>` | 窗口截图选项 |
| `CGSCopyWindowsOptions` | `OptionSet<Int>` | 窗口列表查询选项 |
| `CGSCopyWindowsTags` | `OptionSet<Int>` | 窗口标签过滤 |
| `CGSSpaceMask` | `enum Int` | Space 查询掩码（current=5, other=6, all=7） |
| `SLPSMode` | `enum UInt32` | 进程前置模式 |
| `CGSSymbolicHotKey` | `enum Int` | 系统快捷键标识 |

### 连接管理

```swift
let CGS_CONNECTION = CGSMainConnectionID()
```

应用启动时通过 `CGSMainConnectionID()` 获取与 WindowServer 的主连接，该连接 ID 在后续所有 SkyLight API 调用中作为第一个参数传入。

---

## SkyLight 框架

SkyLight 是 macOS WindowServer 的私有通信框架，位于 `/System/Library/PrivateFrameworks/SkyLight.framework`。AltTab 声明了其中的 14 个核心函数。

### 窗口截图：CGSHWCaptureWindowList

```swift
@_silgen_name("CGSHWCaptureWindowList")
func CGSHWCaptureWindowList(_ cid: CGSConnectionID,
    _ windowList: UnsafeMutablePointer<CGWindowID>,
    _ windowCount: UInt32,
    _ options: CGSWindowCaptureOptions) -> Unmanaged<CFArray>
```

- **功能**：返回指定窗口的 `CGImage` 截图数组。
- **特性**：支持最小化窗口、跨 Space 窗口；HW 前缀暗示硬件加速，性能优于公开 API。
- **限制**：实测仅处理窗口列表中的第一个窗口 ID。
- **截图选项**：
  - `ignoreGlobalClipShape`（`1 << 11`）：忽略全局裁剪形状
  - `nominalResolution`（`1 << 9`）：标准分辨率（Retina 屏幕上为物理分辨率的 1/4）
  - `bestResolution`（`1 << 8`）：最高分辨率
  - `fullSize`（`1 << 19`）：Stage Manager 下获取完整尺寸截图

### Space 管理

**获取所有 Space 信息**：

```swift
@_silgen_name("CGSCopyManagedDisplaySpaces")
func CGSCopyManagedDisplaySpaces(_ cid: CGSConnectionID) -> CFArray
```

返回值为 `CFArray`，每个元素是一个显示器字典，包含 `"Current Space"`、`"Display Identifier"`、`"Spaces"` 键。每个 Space 字典包含 `id64`（Space ID）、`uuid`、`type` 字段。

**重要**：仅在系统偏好设置 > Mission Control > "显示器具有单独的 Space" 勾选时返回完整的多显示器信息；未勾选时只返回一个显示器的信息。

**获取指定显示器的当前 Space**：

```swift
@_silgen_name("CGSManagedDisplayGetCurrentSpace")
func CGSManagedDisplayGetCurrentSpace(_ cid: CGSConnectionID, _ displayUuid: ScreenUuid) -> CGSSpaceID
```

**窗口与 Space 的关系**：

```swift
@_silgen_name("CGSCopySpacesForWindows")
func CGSCopySpacesForWindows(_ cid: CGSConnectionID, _ mask: CGSSpaceMask.RawValue, _ wids: CFArray) -> CFArray

@_silgen_name("CGSAddWindowsToSpaces")
func CGSAddWindowsToSpaces(_ cid: CGSConnectionID, _ windows: NSArray, _ spaces: NSArray) -> Void

@_silgen_name("CGSRemoveWindowsFromSpaces")
func CGSRemoveWindowsFromSpaces(_ cid: CGSConnectionID, _ windows: NSArray, _ spaces: NSArray) -> Void
```

`CGSAddWindowsToSpaces` / `CGSRemoveWindowsFromSpaces` 仅在 macOS 10.10-12.2 可用，12.3 之后 Apple 移除了这些 API。

### 窗口列表查询：CGSCopyWindowsWithOptionsAndTags

```swift
@_silgen_name("CGSCopyWindowsWithOptionsAndTags")
func CGSCopyWindowsWithOptionsAndTags(_ cid: CGSConnectionID, _ owner: Int,
    _ spaces: CFArray, _ options: Int,
    _ setTags: UnsafeMutablePointer<Int>,
    _ clearTags: UnsafeMutablePointer<Int>) -> CFArray
```

返回指定 Space 中的窗口 ID 数组，按 z-index 排序。通过 options 和 tags 参数可以过滤不可见窗口、桌面图标窗口、屏保窗口等。

### 窗口属性：CGSCopyWindowProperty

```swift
@_silgen_name("CGSCopyWindowProperty") @discardableResult
func CGSCopyWindowProperty(_ cid: CGSConnectionID, _ wid: CGWindowID,
    _ property: CFString, _ value: UnsafeMutablePointer<CFTypeRef?>) -> CGError
```

获取指定窗口的某个属性值。`CGWindowID` 扩展中用它来获取窗口标题（属性名 `"kCGSWindowTitle"`）。

### 窗口层级

```swift
@_silgen_name("CGSGetWindowLevel") @discardableResult
func CGSGetWindowLevel(_ cid: CGSConnectionID, _ wid: CGWindowID,
    _ level: UnsafeMutablePointer<CGWindowLevel>) -> CGError
```

返回窗口层级值（对应 `CGWindowLevel.h` 中的定义）。AltTab 用它来判断窗口是否在标准层级（normal level）。

### 进程控制

**前置进程**：

```swift
@_silgen_name("_SLPSSetFrontProcessWithOptions") @discardableResult
func _SLPSSetFrontProcessWithOptions(_ psn: UnsafeMutablePointer<ProcessSerialNumber>,
    _ wid: CGWindowID, _ mode: SLPSMode.RawValue) -> CGError
```

通过 `ProcessSerialNumber` 和窗口 ID 将指定应用/窗口前置。模式选项：

| 模式 | 值 | 含义 |
|------|----|------|
| `allWindows` | `0x100` | 显示应用的所有窗口 |
| `userGenerated` | `0x200` | 模拟用户触发的操作 |
| `noWindows` | `0x400` | 不显示任何窗口 |

**发送事件**：

```swift
@_silgen_name("SLPSPostEventRecordTo") @discardableResult
func SLPSPostEventRecordTo(_ psn: UnsafeMutablePointer<ProcessSerialNumber>,
    _ bytes: UnsafeMutablePointer<UInt8>) -> CGError
```

向 WindowServer 发送原始字节事件，用于模拟窗口操作（如关闭按钮点击）。底层机制参见 Hammerspoon 项目的分析。

### 系统快捷键控制

```swift
@_silgen_name("CGSSetSymbolicHotKeyEnabled") @discardableResult
func CGSSetSymbolicHotKeyEnabled(_ hotKey: CGSSymbolicHotKey.RawValue, _ isEnabled: Bool) -> CGError
```

动态启用/禁用系统快捷键。AltTab 使用的快捷键 ID：

| 快捷键 | ID | 说明 |
|--------|-----|------|
| `commandTab` | 1 | Command+Tab 原生切换器 |
| `commandShiftTab` | 2 | Command+Shift+Tab |
| `commandKeyAboveTab` | 6 | Command+键盘中Tab上方的键 |

通过 `setNativeCommandTabEnabled(_:)` 函数统一控制。**注意**：该设置在应用退出后仍然持久化。

### 屏幕录制权限

```swift
@_silgen_name("SLSRequestScreenCaptureAccess") @discardableResult
func SLSRequestScreenCaptureAccess() -> UInt8
```

返回屏幕录制权限状态（1=已授权，0=未授权），首次调用未授权时弹出系统授权对话框。**限制**：返回值在应用生命周期内不会更新，不会反映用户在系统偏好设置中的实时变更。

### 活跃菜单栏显示器

```swift
@_silgen_name("CGSCopyActiveMenuBarDisplayIdentifier")
func CGSCopyActiveMenuBarDisplayIdentifier(_ cid: CGSConnectionID) -> ScreenUuid
```

返回当前活跃菜单栏所在显示器的 UUID（其他显示器的菜单栏会变暗）。

---

## AXCallScheduler 调度器

`AXCallScheduler`（`src/macos/AXCallScheduler.swift`）是 AltTab 应对 macOS Accessibility 子系统不稳定性的核心组件。macOS 的 AX API 经常因目标应用无响应而阻塞线程，调度器通过状态机、并发控制和退避重试来保证应用整体流畅性。

### 设计动机

macOS 的 `AXUIElement` API 存在以下问题：

1. **阻塞调用**：AX 调用可能因目标应用挂起而长时间阻塞（默认超时 6 秒）。
2. **不返回错误**：某些失败场景下 AX 调用不抛出异常，而是返回零值对象。
3. **瞬态失败**：窗口属性查询可能暂时失败，稍后重试即可成功。
4. **进程级传染**：一个无响应应用的 AX 调用会影响对该应用的所有后续调用。

### 4 阶段状态机

每个调度 key（字符串标识符）维护独立的 `KeyState`，经历以下 4 个阶段：

```
idle ──→ throttled ──→ executing ──→ retrying ──→ executing (重试)
  ↑           │             │             │             │
  └───────────┴─────────────┴─────────────┴─────────────┘
                      （成功/超时后回到 idle）
```

| 阶段 | 含义 | 行为 |
|------|------|------|
| **idle** | 空闲，无进行中的操作 | 检查节流间隔，立即执行或进入 throttled |
| **throttled** | 正在等待节流间隔到期 | 新请求替换 pending block（coalescing） |
| **executing** | block 正在执行中 | 新请求替换 pending block，等待完成后处理 |
| **retrying** | 执行失败，正在退避重试中 | 新请求替换 pending block 并设置 `cancelRetries` 标志取消当前重试 |

### KeyState 数据结构

```swift
private struct KeyState {
    var phase: Phase = .idle
    var lastExecutionTime: UInt64 = 0     // 上次完成执行的纳秒时间戳
    var retryStartTime: UInt64 = 0        // 首次重试开始的纳秒时间戳
    var retryCount = 0                     // 当前重试次数
    var pendingBlock: (() throws -> Void)? // 等待执行的 block（coalescing）
    var pendingPid: pid_t?                 // block 关联的进程 ID
    var pendingContext: String?            // 调试上下文信息
    var cancelRetries = false              // 是否取消当前重试序列
}
```

### 双队列并发控制

调度器管理两个 `LabeledOperationQueue`：

| 队列 | 名称 | QoS | 最大并发 | 用途 |
|------|------|-----|---------|------|
| `fastQueue` | `axCallsFast` | `.userInteractive` | 16 | 正常应用的 AX 调用 |
| `retryQueue` | `axCallsRetry` | `.userInteractive` | 8 | 无响应应用的 AX 调用重试 |

**队列选择策略**：通过 `unresponsivePids` 集合判断目标进程是否无响应。无响应进程的调用被路由到 `retryQueue`，避免占用 `fastQueue` 的并发槽位。当调用成功后，进程从 `unresponsivePids` 中移除，后续调用回到 `fastQueue`。

### 指数退避策略

```swift
private static let backoffStepsNs: [UInt64] = [200_000_000, 1_000_000_000, 2_000_000_000, 5_000_000_000]
```

重试延迟随次数递增：200ms -> 1s -> 2s -> 5s -> 5s（封顶）：

```
retryCount=0 → delay = backoffSteps[0] = 200ms
retryCount=1 → delay = backoffSteps[1] = 1s
retryCount=2 → delay = backoffSteps[2] = 2s
retryCount≥3 → delay = backoffSteps[3] = 5s（封顶）
```

使用 `min(retryCount, backoffStepsNs.count - 1)` 确保索引不越界。

### 60 秒超时放弃

```swift
private static let giveUpAfterSeconds: Float = 60.0
```

从首次重试开始计时，超过 60 秒仍未成功的调用将被放弃。`retryStartTime` 记录首次重试的时间戳，每次重试前检查是否超时：

```swift
let elapsed = Float(DispatchTime.now().uptimeNanoseconds - retryStartTime) / 1_000_000_000
if elapsed >= Self.giveUpAfterSeconds {
    // 放弃，清理状态
}
```

### Per-key 节流

```swift
private static let throttleDelayNs: UInt64 = 200_000_000  // 200ms
```

每个 key 的两次执行之间至少间隔 200ms。在 `schedule()` 和 `drainPending()` 中，如果距上次执行不足 200ms，则进入 throttled 阶段，设置延迟触发。

### Pending Block Coalescing

当某个 key 已有正在执行或等待中的操作时，新提交的 block 会**替换**旧的 pending block，而非排队。这意味着：

- 只有最新的一次请求会被执行，中间的请求被丢弃。
- 在 `throttled`、`executing`、`retrying` 阶段，新 block 都会覆盖旧 pending block。
- 在 `retrying` 阶段，新 block 还会设置 `cancelRetries = true`，取消正在进行的重试序列。

这种 coalescing 策略适用于窗口属性轮询场景：窗口标题、位置等属性只需要最新值，历史请求可以安全丢弃。

### 调度流程详解

1. **`schedule(key:pid:block:)`**：入口方法。获取 key 对应的 KeyState，根据当前 phase 决定行为。
2. **`submitToQueue(key:pid:block:)`**：将 block 提交到 fastQueue 或 retryQueue，包装为 `attemptBlock` 调用。
3. **`attemptBlock(key:pid:retryStartTime:block:)`**：执行 block，成功则调用 `onComplete`，失败则计算退避延迟并提交到 retryQueue。
4. **`onComplete(key:)`**：更新 `lastExecutionTime`，重置 phase 为 idle，调用 `drainPending`。
5. **`drainPending(key:)`**：检查是否有 pending block，如果有则再次进入执行流程（可能先经过 throttle）。
6. **`fireThrottled(key:)`**：throttle 定时器到期后，取出 pending block 并执行。

---

## AXUIElement 封装

`AXUIElement.swift` 对 macOS Accessibility API 进行了高层封装，提供批量属性获取、窗口枚举和错误恢复。

### 全局超时设置

```swift
private static let globalMessagingTimeoutInSeconds = Float(1)
static func setGlobalTimeout() {
    AXUIElementSetMessagingTimeout(AXUIElementCreateSystemWide(), globalMessagingTimeoutInSeconds)
}
```

将 AX 调用的全局超时从默认 6 秒降低到 1 秒。这防止无响应应用阻塞过多线程，但代价是某些合法的慢操作可能被提前中断。

### 错误处理模式

```swift
private func throwIfNotSuccess(_ result: AXError) throws -> Void {
    if result == .cannotComplete {
        throw AxError.runtimeError
    }
    // .success 和其他错误不抛出
}
```

仅当 AX 返回 `.cannotComplete`（目标应用无响应）时抛出异常。其他错误（如属性不存在）静默忽略，由调用者检查可选值。

### 批量属性获取：attributes(_:)

```swift
func attributes(_ keys: [String]) throws -> AXAttributes
```

通过 `AXUIElementCopyMultipleAttributeValues` 一次性获取多个属性值，比逐个查询高效。返回值封装为 `AXAttributes` 结构体，包含 15 个可选字段：

- **窗口属性**：`title`、`role`、`subrole`、`isMinimized`、`isFullscreen`、`statusLabel`
- **层级属性**：`parent`、`children`、`focusedWindow`、`mainWindow`、`closeButton`、`windows`
- **几何属性**：`position`（CGPoint）、`size`（CGSize）
- **应用属性**：`appIsRunning`
- **URL 属性**：`url`

**类型安全转换**：`castSafely<T>()` 方法根据 `CFTypeID` 分派转换逻辑：

| CFTypeID | 目标类型 |
|----------|---------|
| `AXValueGetTypeID()` | 通过 `AXValueGetType` 分派 `.cgSize` / `.cgPoint` 等 |
| `AXUIElementGetTypeID()` | `AXUIElement` |
| `CFArrayGetTypeID()` | `[AXUIElement]` |
| `CFStringGetTypeID()` | `String` |
| `CFBooleanGetTypeID()` | `Bool` |
| `CFURLGetTypeID()` | `URL` |

**`.axError` 处理**：当 `AXUIElementCopyMultipleAttributeValues` 使用 `.stopOnError` 选项时，缺失属性返回 `.axError` 占位值而非 nil。`castSafely` 将其映射为 `nil`，避免将零值 AXUIElement 误认为有效对象。

### 双路径窗口枚举：allWindows(_:)

```swift
func allWindows(_ pid: pid_t) throws -> [AXUIElement]
```

结合两种策略枚举应用的所有窗口：

1. **标准路径** `windows()`：通过 `kAXWindowsAttribute` 获取。**限制**：不返回其他 Space 的窗口。
2. **暴力路径** `windowsByBruteForce(_:)`：通过 `_AXUIElementCreateWithRemoteToken` 构造 AXUIElement，遍历 ID 0-999。**限制**：应用刚启动时可能遗漏窗口。

两种路径的结果通过 `Set` 去重合并，确保覆盖所有场景。

**Remote Token 构造**：token 为 20 字节的 `Data`，结构为：

| 偏移 | 长度 | 内容 |
|------|------|------|
| 0-3 | 4 字节 | PID |
| 4-7 | 4 字节 | `0` |
| 8-11 | 4 字节 | `0x636f636f`（"coco"） |
| 12-19 | 8 字节 | AXUIElementID |

暴力路径使用 `LightweightTimer` 限制总耗时不超过 100ms，在性能和窗口发现率之间取得平衡。

### OS 级标签页检测：tabGroupInfo(_:)

```swift
static func tabGroupInfo(_ children: [AXUIElement]?) -> [String]?
```

遍历窗口的 children，查找 `role == "AXTabGroup"` 的子元素，提取其中 `subrole == "AXTabButton"` 的标签页标题。仅在标签页数量 >= 2 时返回结果。

### HIServices 桥接

`ApplicationServices.HIServices.framework.swift` 补充了 Apple 框架中缺失的常量和函数：

**缺失的私有函数**：

- `_AXUIElementGetWindow`：从 AXUIElement 获取 CGWindowID（10.10+）
- `_AXUIElementCreateWithRemoteToken`：从二进制 token 创建 AXUIElement

**缺失的常量**：

- `kAXDocumentWindowSubrole = "AXDocumentWindow"`
- `kAXFullscreenAttribute = "AXFullScreen"`
- `kAXStatusLabelAttribute = "AXStatusLabel"`

**MissionControlState 枚举**（也是缺失的 AX 通知常量）：

```swift
enum MissionControlState: String, CaseIterable {
    case showAllWindows = "AXExposeShowAllWindows"
    case showFrontWindows = "AXExposeShowFrontWindows"
    case showDesktop = "AXExposeShowDesktop"
    case inactive = "AXExposeExit"
}
```

**已废弃但仍然可用的 Process Manager 函数**（10.9-10.15）：

- `GetProcessInformation(_:_:)`：通过 PSN 获取进程信息
- `GetProcessForPID(_:_:)`：通过 PID 获取 PSN

---

## CGWindow 操作

### CGWindow 类型

```swift
typealias CGWindow = [CFString: Any]
```

`CGWindowListCopyWindowInfo` 返回的字典类型别名。提供以下便捷访问方法：

- `windows(_:)`：调用 `CGWindowListCopyWindowInfo`，排除桌面元素
- `isNotMenubarOrOthers()`：通过 `layer() == 0` 过滤非窗口 UI 元素
- `id()`、`layer()`、`bounds()`、`ownerPID()`、`ownerName()`、`title()`：类型安全的属性访问

### CGWindowID 扩展

`CGWindowID` 上的扩展方法直接调用 SkyLight 私有 API：

- `title()` → `CGSCopyWindowProperty`（key: `"kCGSWindowTitle"`）
- `level()` → `CGSGetWindowLevel`
- `spaces()` → `CGSCopySpacesForWindows`（返回窗口所属的所有 Space ID）

---

## MissionControl 状态检测

`MissionControl.swift` 提供两种检测策略，按 macOS 版本分支：

### macOS 12+（Monterey 及以后）

通过 AX 私有通知（`MissionControlState` 枚举值）直接获取状态，准确度高。使用 `NSLock` 保护内部状态变量：

```swift
private static let stateLock = NSLock()
private static var state_ = MissionControlState.inactive

static func setState(_ state: MissionControlState) {
    stateLock.lock()
    defer { stateLock.unlock() }
    state_ = state
    Logger.info { state }
}
```

通知由 `missionControl` RunLoop 线程上的 AXObserver 接收，回调中更新 `state_`。

### macOS 12 以下

通过 **Dock 进程窗口侧效应** 推断：遍历 `CGWindowListCopyWindowInfo(.optionOnScreenOnly)` 的结果，检查是否存在 `ownerName == "Dock"` 且 `title == nil` 且 `layer != 500` 的窗口。条件 `layer != 500` 排除了用户从 Dock 文件夹拖拽文件时的误判（issue #706）。

---

## 系统权限处理

`SystemPermissions.swift` 管理两项 macOS 隐私权限：**辅助功能**（Accessibility）和**屏幕录制**（Screen Recording）。

### 权限检测

**辅助功能权限**：

```swift
AXIsProcessTrustedWithOptions([kAXTrustedCheckOptionPrompt.takeRetainedValue(): false] as CFDictionary)
```

使用标准 API 检测，参数 `false` 表示不自动弹出系统授权对话框。

**屏幕录制权限**：

由于 `CGPreflightScreenCaptureAccess` 和 `SLSRequestScreenCaptureAccess` 的返回值在应用生命周期内不更新，AltTab 使用以下策略：

| macOS 版本 | 检测方法 |
|-----------|---------|
| 12.3+ | `SCShareableContent.getExcludingDesktopWindows`：如果回调返回有效内容则权限已授予 |
| 10.15-12.2 | `CGDisplayStream` 创建：在每个屏幕上尝试创建 1x1 像素的 DisplayStream，成功则权限已授予 |
| < 10.15 | 无需屏幕录制权限 |

两种方法都通过 `runWithTimeout` 包装，设置 6 秒超时防止阻塞。

### 权限轮询策略

使用 `DispatchSourceTimer` 进行分级轮询：

| 场景 | 轮询间隔 | 说明 |
|------|---------|------|
| 启动前（权限未授予） | 0.5 秒 | 快速检测用户授权 |
| 启动后（权限已授予，无分布式通知监听） | 5 秒 | 中等频率 |
| 启动后（权限已授予，有分布式通知监听） | 60 秒 | 稀疏后备 |
| 权限窗口可见 | 0.5 秒 | 实时反馈 |

**分布式通知优化**：启动后注册 `com.apple.accessibility.api` 分布式通知，当 Apple 撤销辅助功能权限时触发回调重启应用。该通知名称未公开文档化，行为不完全可靠，因此保留 60 秒后备轮询。

**权限撤销处理**：

- 辅助功能权限被撤销 → 记录日志并调用 `App.restart()` 重启应用
- 屏幕录制权限被撤销 → UI 提示用户重新授权

---

## LoginItem 登录项管理

`LoginItem.swift` 通过 launchd plist 实现"开机启动"功能，不使用已废弃的 `LSSharedFileList` API。

### launchd Plist 结构

```swift
private static let launchAgentPlist: NSDictionary = [
    "Label": App.bundleIdentifier,
    "Program": Bundle.main.executablePath,
    "RunAtLoad": true,
    "LimitLoadToSessionType": "Aqua",
    "AssociatedBundleIdentifiers": App.bundleIdentifier,
    "ProcessType": "Interactive",
    "LegacyTimers": true,
]
```

写入路径：`~/Library/LaunchAgents/<bundleIdentifier>.plist`。`ProcessType: "Interactive"` 避免系统对后台进程施加 CPU 和 I/O 限制。`LegacyTimers` 确保定时器精度。

### 旧登录项迁移

`removeLoginItemIfPresent()` 使用已废弃的 `LSSharedFileList` API 检查并移除旧版登录项（macOS 10.11 之前的 API），通过 `AvoidDeprecationWarnings` 协议隔离废弃警告。如果发现旧登录项但用户未开启新设置，自动开启开机启动。

---

## 辅助工具

### Logger 日志系统

4 级日志系统（debug / info / warning / error），特性包括：

- **编译期过滤**：`emit()` 方法在 `@inline(__always)` 标记下，当 `level < minLevel` 时闭包不会被调用，零性能开销。
- **ANSI 彩色输出**：4 种 xterm-256 颜色（绿/青/黄/粉），与 SwiftyBeaver 默认配色一致。
- **异步写入**：格式化和 I/O 在 `writeQueue`（`.utility` QoS）上执行，不阻塞调用线程（通常是主线程）。
- **函数名清理**：`cleanFunctionName()` 移除 Swift `#function` 中的匿名参数占位符（如 `init(_:_:_:_:)` → `init()`）。
- **日志级别**通过命令行参数 `--logs=<level>` 配置，默认 `.error`。
- **Tap 机制**：`setTap()` 允许 DebugWindow 等组件订阅日志输出（无 ANSI 颜色）。

输出格式：`HH:mm:ss.SSS LEVEL file.swift:line function [thread] message`

### Bash Shell 命令执行

```swift
class Bash {
    static func command(_ command: String) -> String?
}
```

同步执行 `/bin/bash -c <command>`，返回 stdout 内容。用于执行系统级命令（如 `defaults read` 等）。

### Sysctl 系统参数查询

```swift
class Sysctl {
    static func run(_ name: String) -> String
    static func run<T>(_ name: String, _ type: T.Type) -> T?
    static func run(_ keys: [Int32]) -> String?
}
```

封装 BSD `sysctl` 系统调用，支持三种使用模式：

1. **字符串查询**：按名称查询，返回字符串值（如 `"kern.osrelease"`）
2. **类型查询**：按名称查询，返回指定类型（如 `Int.self`）
3. **OID 查询**：直接使用 `Int32` OID 数组查询

内部通过 `sysctlnametomib` 将名称转换为 OID，两步调用（先查大小、再取数据）确保缓冲区正确。

### LightweightTimer 轻量计时器

```swift
class LightweightTimer {
    init()
    var elapsedMilliseconds: Double { get }
    func hasElapsed(milliseconds: Double) -> Bool
}
```

基于 `mach_absolute_time()` 的高精度计时器，通过 `mach_timebase_info` 将绝对时间转换为纳秒。用于暴力枚举窗口时的 100ms 超时控制，比 `DispatchTime` 更轻量（无需 GCD 调度）。

### Markdown 渲染器

```swift
class Markdown {
    static func toAttributedString(_ markdown: String) -> NSAttributedString
}
```

轻量 Markdown 渲染器，支持以下语法：

| 语法 | 渲染效果 |
|------|---------|
| `## 标题` | 粗体 15pt |
| `**粗体**` | 粗体 13pt |
| `[文本](URL)` | 蓝色下划线链接 |
| `* 列表项` | 20pt 缩进 + 项目符号 |

通过 `NSRegularExpression` 正则匹配，逐行处理。用于应用内 "关于" 和更新日志等 UI 文本的渲染。

### HelperExtensions 通用扩展

提供大量跨模块使用的扩展方法：

- **动画控制**：`NoAnimationDelegate`（禁用 CALayer 隐式动画）、`caTransaction(_:)`（无动画事务）
- **NSColor**：主题感知的表格颜色（边框/背景/分隔线/悬停）、系统强调色、十六进制转换
- **CALayer**：居中、阴影应用
- **NSView**：约束管理（`fit()`/`fit(_:_:)`）、子视图管理、标题栏内边距包装
- **NSImage**：`initCopy()`（强制创建独立实例）、`fromSymbol()`（SF Symbol 渲染为模板图像，裁剪到字形墨迹边界）、`tinted()`（着色）
- **CGImage**：缩放、命名资源加载、最佳尺寸匹配
- **pid_t**：`isZombie()` 通过 sysctl 检测僵尸进程
- **String**：`StringInterpolation` 扩展消除 Optional 插值警告
- **CGEvent**：`toNSEvent()` 线程安全的 CGEvent → NSEvent 转换
- **Collection**：`subscript(safe:)` 安全下标访问
- **ModifierFlags**：当前修饰键状态
- **NSPoint**：算术运算符重载

### HelperExtensionsTestable

测试和生产共用的辅助代码：

- **SelectorWrapper<T>**：将闭包包装为 Objective-C selector，用于 NSControl 的 action 回调
- **CGImage.resizedCopyWithCoreGraphics**：使用 CoreGraphics 进行高质量图像缩放，支持位图信息修复

---

## 文件清单

| 文件路径 | 行数 | 核心类型/函数 |
|---------|------|-------------|
| `src/macos/api-wrappers/SkyLight.framework.swift` | 202 | `CGS_CONNECTION`, `CGSHWCaptureWindowList`, `CGSCopyManagedDisplaySpaces`, `CGSCopyWindowsWithOptionsAndTags`, `_SLPSSetFrontProcessWithOptions`, `SLPSPostEventRecordTo`, `CGSSetSymbolicHotKeyEnabled`, `SLSRequestScreenCaptureAccess` |
| `src/macos/api-wrappers/AXUIElement.swift` | 240 | `AXAttributes`, `AxError`, `attributes(_:)`, `allWindows(_:)`, `windowsByBruteForce(_:)`, `tabGroupInfo(_:)`, `castSafely(_:)` |
| `src/macos/api-wrappers/CGWindow.swift` | 53 | `CGWindow` typealias, `windows(_:)`, `isNotMenubarOrOthers()`, `id()`, `bounds()`, `ownerPID()`, `title()` |
| `src/macos/api-wrappers/CGWindowID.swift` | 24 | `title()`, `level()`, `spaces()`, `cgProperty(_:_:)` |
| `src/macos/api-wrappers/ApplicationServices.HIServices.framework.swift` | 42 | `_AXUIElementGetWindow`, `_AXUIElementCreateWithRemoteToken`, `GetProcessInformation`, `GetProcessForPID`, `MissionControlState`, `kAXFullscreenAttribute` |
| `src/macos/api-wrappers/HelperExtensions.swift` | 434 | `NoAnimationDelegate`, `caTransaction`, NSColor 主题色, NSView 约束, NSImage.fromSymbol, pid_t.isZombie, CGEvent.toNSEvent |
| `src/macos/api-wrappers/HelperExtensionsTestable.swift` | 61 | `SelectorWrapper<T>`, `CGImage.resizedCopyWithCoreGraphics` |
| `src/macos/api-wrappers/Logger.swift` | 121 | `Logger` (debug/info/warning/error), `LogLevel`, ANSI 彩色输出, 异步写入 |
| `src/macos/api-wrappers/Markdown.swift` | 73 | `Markdown.toAttributedString`, 标题/粗体/链接/列表渲染 |
| `src/macos/api-wrappers/Bash.swift` | 15 | `Bash.command` |
| `src/macos/api-wrappers/Sysctl.swift` | 54 | `Sysctl.run` (字符串/类型/OID 查询) |
| `src/macos/api-wrappers/LightweightTimer.swift` | 27 | `LightweightTimer`, `mach_absolute_time` 计时 |
| `src/macos/api-wrappers/MissionControl.swift` | 39 | `MissionControl.state()`, `setState(_:)`, `isActive()` |
| `src/macos/AXCallScheduler.swift` | 263 | `AXCallScheduler`, 4 阶段状态机, 双队列, 指数退避 |
| `src/macos/LoginItem.swift` | 82 | `LoginItem.applyCurrentPreference`, launchd plist 管理 |
| `src/macos/SystemPermissions.swift` | 219 | `SystemPermissions`, `AccessibilityPermission`, `ScreenRecordingPermission` |
