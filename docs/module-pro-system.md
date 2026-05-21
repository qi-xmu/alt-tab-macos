# Pro 系统

## 模块职责

Pro 系统是 AltTab 的商业化层，负责许可证管理、功能门控、试用期到付费的转化漏斗以及所有 Pro 相关的 UI 组件。它由三个子系统组成：

- **License 子系统** (`src/pro/license/`) -- 管理许可证状态的获取、持久化、远程验证和版本限制检查。
- **调度子系统** (`src/pro/scheduling/`) -- 驱动一个 35 天 7 触点的转化漏斗，使用纯函数状态机决策逻辑。
- **UI 子系统** (`src/pro/ui/`) -- 提供所有 Pro 提示窗口、气泡、徽章、渐变按钮等共享 UI 组件。

设计上，调度逻辑与 UI 完全解耦：`ProTransitionManager` 通过 `ProPromptAction` 枚举发出动作请求，`ProPromptHost` 负责将抽象动作映射到具体的窗口/气泡类，使协调器不依赖任何 AppKit 类型。

---

## 许可证管理

### 四种 License 状态

`LicenseState` 枚举定义了四种互斥的许可证状态：

```swift
enum LicenseState: Equatable {
    case trial(daysRemaining: Int)   // 14 天试用期内
    case pro                          // 已激活 Pro
    case proExpired                   // Pro 许可证已过期（版本限制型或验证失败）
    case trialExpired                 // 试用期已过且未购买
}
```

状态转换规则（`LicenseManager.computeState()`）：

1. **存在 licenseKey（Keychain）** -> 检查 `lastValidationResult`：
   - `false` -> `.trialExpired`（远程验证标记为无效）
   - `true` -> 检查版本限制（`versionLimitedVariants` 映射），若当前版本超过限制 -> `.proExpired`，否则 -> `.pro`
2. **无 licenseKey** -> 调用 `computeTrialState()`：
   - 首次运行时记录 `trialStartDate`
   - `daysSinceTrialStart < 14` -> `.trial(daysRemaining: 14 - daysSinceTrialStart)`
   - 否则 -> `.trialExpired`

关键属性：
- `isProAvailable`：`.trial` 和 `.pro` 时为 `true`（功能可用）
- `isProLocked`：`.proExpired` 和 `.trialExpired` 时为 `true`（功能降级）

### Keychain + UserDefaults 混合存储策略

许可证数据分散存储在两个位置：

**Keychain**（`SystemKeychain`，服务名 `$(bundleId).license`）：
- `licenseKey` -- 许可证密钥
- `instanceId` -- 激活实例 ID
- `variantId` -- 变体标识（如 `pro_lifetime`）
- `machineFingerprint` -- 机器指纹回退值

**UserDefaults**（suite `$(bundleId).license`）：
- `trialStartDate` -- 试用期开始时间戳
- `lastValidation` -- 上次远程验证时间戳
- `lastValidationResult` -- 上次验证结果（Bool）
- `customerEmail` -- 客户邮箱

### 机器指纹

`MachineFingerprint.get(keychain:)` 的三级回退策略：

1. **首选**：通过 IOKit 读取 `IOPlatformExpertDevice` 的 `kIOPlatformUUIDKey`，获取硬件 UUID
2. **回退**：从 Keychain 读取之前存储的指纹值
3. **兜底**：生成随机 UUID 并写入 Keychain，保证后续验证一致性

使用 `IOServiceGetMatchingService(0, ...)` 传入 0 避免 `kIOMasterPortDefault` 弃用警告。

### 远程验证 API

`LicenseAPI` 协议定义三个操作：

- `activate(_ licenseKey:)` -> `ActivateResult`（返回 instanceId、variantId、customerEmail）
- `validate(_ licenseKey:, instanceId:)` -> `ValidateResult`（返回 valid、variantId）
- `deactivate(_ licenseKey:, instanceId:)` -> `Void`

`RemoteLicenseClient` 是具体实现，与 `alt-tab.app/v1/license/*` 后端通信。激活时附带 `fingerprint` 和可选的 `trial_started_at`（用于后端计算转化延迟）。`LicenseAPIError` 枚举覆盖所有错误场景：`invalidKey`、`seatLimitExceeded`（附带活跃实例列表）、`keychainWriteFailed` 等。

### 30 天静默重验证

`LicenseManager.scheduleAsyncRevalidationIfNeeded()` 在每次 `initialize()` 时检查距上次验证的时间间隔。若超过 `revalidationInterval`（30 天 = 2592000 秒），则自动调用 `revalidateWithServer()`。网络失败时静默忽略，下次启动重试。

验证成功时更新 `lastValidation` 时间戳和 `lastValidationResult`。验证失败则直接设为 `.trialExpired`。

### 版本限制型 License

`LicenseManager.versionLimitedVariants` 是一个 `[String: String]` 字典，将变体 slug 映射到最大支持版本号（如 `"pro_v1": "2.0.0"`）。在 `computeState()` 中，若当前 app 版本（`currentAppVersion`）超过此限制，状态变为 `.proExpired`。当前该字典为空，预留为未来版本限制预留机制。

`lifetimeVariants` 集合包含 `"pro_lifetime"`，表示终身许可证，不受版本限制影响。

### Cookie 同步

`syncLicenseCookie(state:)` 在每次状态变更时写入一个 `license` Cookie 到 `alt-tab.app` 域名，值为 `"pro"`、`"proExpired"` 或空字符串。Sparkle 的 appcast 请求携带此 Cookie，使服务端能按许可证层级返回对应的更新信息。

### 时钟抽象

`Clock` 协议和 `SystemClock` 实现将 `Date()` 调用抽象为可注入的依赖，使测试可以控制时间流逝。`LicenseManager` 通过 `clock` 属性读取当前时间。

---

## 功能门控

### 功能定义

`ProFeature` 枚举定义了所有 Pro 功能，分为两大类：

**可降级（Degradable）功能** -- 有对应的存储偏好设置：
- `.appIconsAndTitlesStyle` -- 应用图标和标题样式（Pro: `.appIcons` / `.titles`；Free: `.thumbnails`）
- `.autoSize` -- 自动尺寸（Pro: `.auto`；Free: `.medium`）
- `.searchOnReleaseShortcut` -- 释放时搜索快捷键（Pro: `.searchOnRelease`；Free: `.doNothingOnRelease`）

**硬门控（Hard-Gated）功能** -- 无存储偏好，使用时拦截：
- `.extraShortcut(index: Int)` -- 额外快捷键（索引 >= 1）
- `.searchInSwitcher` -- 切换器内搜索
- `.lockSearchInSwitcher` -- 锁定搜索

### 门控类型 (GateKind)

每个功能有三种门控类型：

| GateKind | 含义 | 适用功能 |
|---|---|---|
| `.degradable` | 仅静默降级，不触发升级 UI | `.autoSize` |
| `.hardGated` | 仅使用时门控，无偏好存储 | `.extraShortcut`、`.searchInSwitcher`、`.lockSearchInSwitcher` |
| `.degradableAndHardGated` | 降级 + 首次过期后触发 [C] | `.appIconsAndTitlesStyle`、`.searchOnReleaseShortcut` |

### 运行时门控流程

`ProFeature.attemptUse()` 是运行时门控的统一入口：

1. **Pro 或试用期可用** -> 直接返回 `true`
2. **Free-Pass 会话激活中** -> 直接返回 `true`（用户正处于一次性 Pro 体验中）
3. **硬门控功能** -> 调用 `ProTransitionManager.shared.attemptHardGatedFeature(self)` 进入 Free-Pass 阶梯
4. **可降级功能** -> 直接返回 `true`（已在偏好写入时降级）

### PreferenceDefinition + PreferenceGate 偏好降级机制

`PreferenceDefinition<T>` 将每个可降级偏好声明为一等公民，包含：

- `key` -- UserDefaults 存储键
- `default` -- Free 层默认值
- `gate: PreferenceGate<T>?` -- 可选的门控策略

`PreferenceGate<T>` 包含三个要素：
- `freeEquivalent` -- Free 层替代值
- `rememberedKey` -- 快照存储键（如 `"rememberedAppearanceStyle"`）
- `isProValue` -- 判断某个枚举值是否属于 Pro

`read()` 方法的三个分支：
1. Free-Pass 会话激活且有 `remembered*` 值 -> 返回记住的 Pro 值
2. 存储值仍是 Pro 值（lock 与 `onProLockEngaged()` 之间瞬态） -> 返回 Free 等价物
3. 正常情况 -> 返回存储值（已被降级）

`snapshotAndDowngrade()` 将当前 Pro 值写入 `remembered*` 键，同时将存储值改为 Free 等价物。

`restore(from:)` 在购买 Pro 后将 `remembered*` 值恢复到存储位置（`notify: false` 避免触发升级 Tab 跳转）。

`ProGatedPreferences` 注册了 6 个门控偏好（3 个全局 + 3 个快捷键 0 覆写），通过 `all` 数组和 `forPreferenceKey(_:)` 查找。

### Free-Pass 一次性体验

Free-Pass 是给过期用户的一次性 Pro 功能体验，通过 `ProTransitionManager.isFreePassSessionActive` 控制：

**触发方式**（二选一，仅首次）：
1. 过期后首次打开切换器 -> `onSwitcherShown()` 设置 `freePassUsed = true` + `isFreePassSessionActive = true`
2. 首次尝试硬门控功能 -> `attemptHardGatedFeature()` 进入 `.freePass` 分支

**效果**：`PreferenceDefinition.read()` 检测到 `isFreePassSessionActive` 后返回 `remembered*` 值，使切换器渲染用户之前的 Pro 选择。切换器关闭时（`onSwitcherDismissed()`）结束 Free-Pass 会话，恢复 Free 读取。

**后续**：Free-Pass 结束后延迟 1 秒弹出 [C] Full Upgrade 窗口（通过 `pendingDismissAction` 机制）。

### Ghost UI（灰化 Pro 选项）

过期后，Settings UI 在之前选择的 Pro 选项上显示灰化标记：

- `rememberedAppearanceStyle` -> 在风格选择器上显示灰色轮廓
- `rememberedAppearanceSize` -> 在 `.auto` 分段控件上显示灰色覆盖
- `rememberedShortcutStyle` -> 在下拉菜单项上显示灰色背景

点击这些灰化选项会被拦截并跳转到 Upgrade Tab（通过 `PreferencesEvents.preferenceChanged` 中的 `isStoredValuePro` 检查）。

---

## 转化漏斗

### 纯状态机决策逻辑

`ProTransitionManagerTestable` 是一个零副作用的纯函数状态机。它接收一个 `State` 快照，返回决策结果，不访问任何单例、日期或 UI。

**State 快照**包含 13 个布尔/整数字段：
- `isPro`、`isTrialActive`、`daysSinceTrialStart`（0-indexed）
- `isInTimeWindow`（10:00-11:30 或 15:30-17:00）
- 8 个 `hasSeen*` / `has*` 标志

**三个决策函数**：

1. `evaluateTimedAction(_:)` -> `TimedAction` -- 调度器触发时调用
2. `evaluateHardGate(_:)` -> `HardGateAction` -- 硬门控功能尝试时调用
3. `evaluateSwitcherOpen(_:)` -> `SwitcherOpenAction` -- 切换器打开时调用

### 35 天 7 触点漏斗

| 触点 | 代号 | 时机 | UI 类型 | 触发条件 |
|---|---|---|---|---|
| Day 1 欢迎信 | [A] | 首次启动 | NSWindow (560x520) | `!hasSeenWelcome` |
| Day 4 功能导览 | [H] | Day 4 首次打开切换器 | NSPopover (280x110) | `daysSinceTrialStart == 3 && !hasSeenDay4Tour` |
| Day 12 到期预告 | [B] | Day 12 时间窗口内 | NSPopover (300x100) | `daysSinceTrialStart == 11 && !hasSeenDay12 && isInTimeWindow` |
| Day 15 升级窗口 | [C] | 首次硬门控或首次切换器 | NSWindow (440x340) | Free-Pass 消费后延迟 1s |
| Day 15 主动弹窗 | [D] | Day 15+ 时间窗口内 | NSWindow (380x280) | `daysSinceTrialStart >= 14 && !hasSeenProactiveDay15 && !hasSeenFullUpgrade` |
| Day 15 硬门控气泡 | [E] | [C] 后每次硬门控尝试 | NSPopover (280x100) | `hasSeenFullUpgrade && freePassUsed` |
| Day 21 提醒 | [F] | Day 21-34 时间窗口内 | NSPopover (300x140) | `daysSinceTrialStart 20..<34 && !hasSeenDay21 && !hasSeenDay35` |
| Day 35 最后机会 | [G] | Day 35-48 时间窗口内 | NSWindow (380x280) | `daysSinceTrialStart 34..<48 && !hasSeenDay35` |

### 时间窗口

所有定时触点仅在两个时间窗口内触发（`isInTimeWindow(hour:minute:)`）：
- 上午 10:00 - 11:30（totalMinutes 600-690）
- 下午 15:30 - 17:00（totalMinutes 930-1020）

### 调度器 (ProTransitionScheduler)

`ProTransitionScheduler` 管理一个 `DispatchWorkItem`，使用 `DispatchQueue.main.asyncAfter` 定时触发。核心逻辑：

- `onAppLaunchComplete()` -- 检查是否有错过的唤醒时间，如有则立即触发；然后调度下一个
- `scheduleNext()` -- 调用 `computeNextFireDate()` 获取下一个触发时间，设置定时器
- `computeNextFireDate()` -- 遍历所有未展示的定时触点，找到最早的触发时间
- `cancel()` -- 购买 Pro 时取消所有调度

调度器在 `nextScheduledDate` 键中持久化下一次触发时间，防止应用关闭时丢失。

### 菜单栏图标徽章

Day 12-14（`daysSinceTrialStart 12-13`）期间，菜单栏图标显示一个小圆点徽章（`shouldShowBadgeDot`），提醒用户即将到期。Day 15+ 移除徽章。

### Hard-Gate 阶梯

`evaluateHardGate(_:)` 的四级阶梯：

1. `.allow` -- Pro 或试用期活跃
2. `.freePass` -- 首次硬门控 -> 允许执行一次，然后展示 [C]
3. `.showFullUpgrade` -- 首次 Free-Pass 消费后 -> 展示 [C] 全屏升级窗口
4. `.showHardGatePopover` -- [C] 之后每次硬门控尝试 -> 展示 [E] 气泡

### 上下文化 Hard-Gate 原因

`HardGateReason` 枚举携带触发原因，经 `resolved` 属性解析为 `ResolvedReason`：

优先级排序：`extraShortcut` > `search` > `appIconsStyle` / `titlesStyle` > `nonEngaged`

这使得 [C] 和 [E] 能展示针对性的标题文案（如 "Unlock Search with Pro" 而非通用的 "Get more from AltTab with Pro"）。

### 使用统计驱动的个性化文案

`ProConversionCopy` 提供两个个性化方法：

- `day12Subtitle()` -- 根据已使用的 Pro 功能数量生成不同文案（0 个通用、1-2 个列举名称、3+ 个显示计数）
- `day21Body()` -- 根据总触发次数和 Pro 功能使用次数组合文案

`UsageStatHeroView` 在 [C]、[D]、[G] 窗口中渲染使用统计英雄块：
- 两列布局（总切换次数 + Pro 功能使用次数），Pro 数字用渐变色渲染
- 单列回退（仅切换次数），用于从未使用 Pro 功能的用户
- 零使用时整个块省略

`supportingLine` 属性根据用户是否使用过 Pro 功能展示不同的辅助文案。

### 状态完成判定

`isSchedulingComplete(_:)` 在以下条件满足时停止调度：
- 用户已购买 Pro
- 用户选择退出且已看过 [G]
- 超过 Day 49（`daysSinceTrialStart >= 48`）
- 所有定时触点均已展示

### PendingDismissAction 延迟机制

某些动作需要在切换器关闭后执行（如 [C] 和 [H]），通过 `pendingDismissAction` 实现：

1. 在 `showUi` 期间将动作入队（如 `.showFullUpgrade(reason)`）
2. 切换器关闭时（`onSwitcherDismissed()`）取出动作
3. 延迟 1 秒后执行，确保前台的焦点窗口先回到前台

---

## Pro UI 组件

### ProPromptWindow

`ProPromptWindow` 是所有非模态 Pro 窗口的基类（[A]、[C]、[D]、[G]），统一处理：
- 隐藏标题栏和红绿灯按钮
- `fullSizeContentView` 样式
- `hidesOnDeactivate = false`（不会因失去焦点而隐藏）
- `isReleasedWhenClosed = false`（关闭时不释放，复用单例）
- ESC 键关闭（`cancelOperation` 覆写）

### ProPromptPopover

`ProPromptPopover` 是菜单栏气泡的共享工具（[B]、[E]、[F]、[H]），提供：
- `make()` -- 创建瞬态 NSPopover
- `present()` -- 激活应用、锚定到菜单栏图标下方、设为 key window

### ProPromptHost

`ProPromptHost` 是 `ProPromptAction` 枚举的分发器，将抽象动作映射到具体 UI：

```
.showWelcome           -> Day1WelcomeLetterWindow.show()
.showDay4Tour          -> Day4TourPopover.show()
.showDay12HeadsUp      -> Day12HeadsUpPopover.show() + 刷新菜单栏图标
.showDay15Proactive    -> Day15ProactiveWindow.show()
.showDay15FullUpgrade  -> Day15FullUpgradeWindow.show(for:)
.showDay15HardGatePopover -> Day15HardGatePopover.show(for:)
.showDay21Reminder     -> Day21ReminderPopover.show()
.showDay35Final        -> Day35FinalWindow.show()
.dismissAllProWindows  -> 关闭所有单例窗口
.refreshBadge          -> 刷新菜单栏图标
```

### ProPromptHeader

`ProPromptHeader` 是所有 Pro 窗口的品牌头部组件（图标 + 标题横向排列）：
- 两种尺寸：`.large`（48pt 图标，22pt 粗体）和 `.compact`（32pt 图标，16pt 粗体）
- 自动将标题中最后一个 "Pro" 子串替换为渐变文字附件

### ProBadgeView

`ProBadgeView` 是一个多层 Core Animation 渲染的 Pro 徽章：
- 三层 CAGradientLayer：填充（alpha 0.1）、边框（alpha 0.7）、文字（alpha 1.0）
- 选中+key window 时切换为白色覆盖模式
- 监听窗口 key 状态变化自动刷新颜色
- `hitTest` 返回 `nil`，让点击穿透到父视图

`ProBadgeView.SegmentOverlay` 提供将徽章叠加到 `NSSegmentedControl` 最后一个分段的完整方案，处理布局计算、颜色同步和选择状态。

### ProGradient

`ProGradient` 枚举定义了 Pro 品牌渐变（粉红 `#FF4488` -> 蓝 `#4488FF` -> 天蓝 `#66CCFF`），36 度角方向。提供：
- `makeLayer()` -- 创建 CAGradientLayer
- `makeGradientTextImage()` -- 用 Core Text 渲染渐变文字图片
- `makeProTextAttachment()` -- 用于 NSAttributedString 的渐变文字附件
- `makeFullProBadgeImage()` -- 将完整徽章渲染为 NSImage（用于 NSMenuItem.image）
- `drawGradientFill()` -- 在 NSBezierPath 中绘制渐变填充

### ProGradientButton

`ProGradientButton` 是渐变背景按钮，支持：
- 渐变背景层 + 阴影
- 鼠标悬停时播放光泽动画（`playShineAnimation()`）
- 按下时降低渐变不透明度到 0.82
- 圆角 7pt

### NotAdvisedButton

`NotAdvisedButton` 是用于次要操作的小号灰色无框按钮（如 "Not now"、"Maybe later"），视觉上弱化以引导用户选择主要操作。

### DynamicColorImageView / DynamicColorTextField

这些视图在每次 `viewWillDraw` 时通过闭包重新计算颜色，确保颜色在不同窗口状态（选中/未选中、key/non-key）、外观模式和禁用状态下正确同步。

### ProDropdownItemView

`ProDropdownItemView` 是带 Pro 徽章的下拉菜单行视图（`NSMenuItem.view`），处理高亮绘制（使用系统强调色）、文字颜色切换和鼠标事件转发。

---

## 文件清单

### 根文件

| 文件 | 职责 |
|---|---|
| `src/pro/ProFeature.swift` | Pro 功能注册表：门控类型、运行时拦截、偏好映射、营销文案 |
| `src/pro/ProConversionCopy.swift` | 转化文案：Day 12/21 个性化文案，基于 UsageStats 驱动 |
| `src/pro/ProFeatureCopy.swift` | 功能文案：4 条 Pro 功能的营销描述 |

### License 子系统

| 文件 | 职责 |
|---|---|
| `src/pro/license/LicenseState.swift` | 4 种许可证状态枚举 + `isProAvailable` / `debugProfileLabel` |
| `src/pro/license/LicenseManager.swift` | 中央管理器：状态计算、激活/停用、远程重验证、调试 mock |
| `src/pro/license/LicenseAPI.swift` | 验证 API 协议 + 结果类型 + 错误枚举 |
| `src/pro/license/RemoteLicenseClient.swift` | 后端 HTTP 客户端实现，JSON POST + Decodable 解析 |
| `src/pro/license/LicenseCookie.swift` | Sparkle appcast Cookie 同步函数 |
| `src/pro/license/Keychain.swift` | Keychain CRUD 封装（基于 SecItem* API） |
| `src/pro/license/MachineFingerprint.swift` | IOKit UUID + Keychain 回退机器指纹 |
| `src/pro/license/Clock.swift` | 时钟抽象协议 + 系统时钟实现 |

### 调度子系统

| 文件 | 职责 |
|---|---|
| `src/pro/scheduling/ProTransitionManager.swift` | 转化协调器：单例、Free-Pass 管理、调度器调度、QA 重置 |
| `src/pro/scheduling/ProTransitionManagerTestable.swift` | 纯函数状态机：`evaluateTimedAction` / `evaluateHardGate` / `evaluateSwitcherOpen` |
| `src/pro/scheduling/ProTransitionState.swift` | 持久化状态：hasSeen* 标志、remembered* 索引、快照/恢复 |
| `src/pro/scheduling/ProTransitionScheduler.swift` | 时间调度器：计算下次触发时间、DispatchWorkItem 管理 |
| `src/pro/scheduling/Day1WelcomeLetterWindow.swift` | [A] Day 1 欢迎信窗口（新装/升级两种文案） |
| `src/pro/scheduling/Day4TourPopover.swift` | [H] Day 4 Pro 功能导览气泡 |
| `src/pro/scheduling/Day12HeadsUpPopover.swift` | [B] Day 12 到期预告气泡 |
| `src/pro/scheduling/Day15FullUpgradeWindow.swift` | [C] Day 15 全屏升级窗口（上下文标题 + 使用统计） |
| `src/pro/scheduling/Day15HardGatePopover.swift` | [E] Day 15 硬门控气泡（[C] 后每次触发） |
| `src/pro/scheduling/Day15ProactiveWindow.swift` | [D] Day 15 主动弹窗（时间驱动，非用户触发） |
| `src/pro/scheduling/Day21ReminderPopover.swift` | [F] Day 21 使用统计提醒气泡 |
| `src/pro/scheduling/Day35FinalWindow.swift` | [G] Day 35 最后机会窗口（可选退出） |
| `src/pro/scheduling/ProTransitionSpec.md` | 完整 35 天漏斗规格文档（Mermaid 状态图） |

### UI 子系统

| 文件 | 职责 |
|---|---|
| `src/pro/ui/ProPromptWindow.swift` | 非模态 Pro 窗口基类 |
| `src/pro/ui/ProPromptPopover.swift` | 菜单栏气泡共享工具 |
| `src/pro/ui/ProPromptHost.swift` | ProPromptAction -> 具体 UI 的分发器 |
| `src/pro/ui/ProPromptHeader.swift` | 图标+标题品牌头部组件 |
| `src/pro/ui/ProBadgeView.swift` | Pro 徽章视图 + ProGradient + ProDropdownItemView + NotAdvisedButton |
| `src/pro/ui/ProGradientButton.swift` | 渐变背景按钮（光泽动画 + 按压反馈） |
| `src/pro/ui/UsageStatHeroView.swift` | 使用统计英雄块（双列/单列渐变数字） |
