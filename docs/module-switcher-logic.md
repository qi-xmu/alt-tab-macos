# 切换器逻辑

## 模块职责

切换器逻辑模块是 AltTab 应用的核心交互层，负责从用户按下快捷键到窗口切换完成的整个流程。具体包括：

- **搜索系统**：基于分层匹配的窗口搜索，支持精确匹配、前缀匹配、子串匹配、缩写匹配和模糊匹配（含 Damerau-Levenshtein 编辑距离），并在 UI 上高亮命中区域。
- **会话管理**：追踪每次切换器调用从打开到关闭的完整生命周期，管理选择状态、修饰键状态和搜索查询。
- **快捷键系统**：支持最多 9 组独立的快捷键配置，每组包含 hold 修饰键 + next 窗口键的组合，以及其他动作快捷键。
- **键盘重复**：通过 `DispatchSourceTimer` 模拟 macOS 系统键盘重复率，在列表边界处停止而非循环。
- **预览面板**：独立的 `NSPanel`，用于在主面板旁显示选中窗口的大尺寸截图预览。

## 核心组件

| 文件 | 类/结构 | 职责 |
|------|---------|------|
| `Search.swift` | `Search` | 搜索入口，协调归一化和分层匹配，缓存结果到 `Window` |
| `SearchTestable.swift` | `SearchTestable` | 可测试的搜索算法实现（归一化、分层匹配、Damerau-Levenshtein） |
| `SwitcherSession.swift` | `SwitcherSession` | 单次切换器调用的全部可变状态 |
| `ATShortcut.swift` | `ATShortcut` | 单个快捷键的匹配逻辑、状态机和作用域判断 |
| `ShortcutAction.swift` | `ShortcutAction` / `ShortcutActions` | 快捷键到动作的映射表和执行分发 |
| `KeyRepeatTimer.swift` | `KeyRepeatTimer` | 匹配系统键盘重复率的 GCD 定时器 |
| `PreviewPanel.swift` | `PreviewPanel` | 窗口预览的浮层面板 |

## 搜索系统

### 架构概览

搜索系统分为两层：

1. **`Search`**（`Search.swift`）：面向 `Window` 对象的高级 API，负责缓存管理、应用名/标题双字段评分合并。
2. **`SearchTestable`**（`SearchTestable.swift`）：纯算法层，无 UI 依赖，可直接在单元测试中调用。

### 文本归一化 — `SearchTestable.normalize(_:)`

归一化将原始字符串转换为 `Normalized` 结构体，包含：

- `text: String` — 去除空白、应用 Unicode 折叠（`folding(options: [.diacriticInsensitive, .caseInsensitive])`）后的文本
- `chars: [Character]` — 归一化字符数组
- `toOriginal: [Int]` — 每个归一化字符到原始字符串索引的映射
- `isWordStart: [Bool]` — 包含 camelCase 边界的词首标记（用于 T3/T5）
- `isHardWordStart: [Bool]` — 仅非字母数字边界的词首标记（用于 T6）

词首检测由 `computeIsWordStart` 和 `computeIsHardWordStart` 两个方法实现。前者检测 camelCase 转换（`isLower(prev) && isUpper(c)`），后者仅在非字母数字字符后触发。folding 操作可能将一个字符展开为多个（如德语 `ß` → `ss`），因此 `toOriginal` 映射是一对多的，每个折叠后的字符都指向同一个原始索引，只有首个折叠字符标记为词首。

### 分层匹配 — `SearchTestable.tierMatch(query:text:)`

核心搜索算法，按优先级从高到低尝试 6 个匹配层级。一旦某层匹配成功即返回，不会继续尝试更低层级：

**Tier 1 — 精确匹配** (`tierExactBase = 1000`)
- 归一化后的 query 与 text 长度相等且字符完全相同
- 例：`"safari"` 匹配 `"Safari"`

**Tier 2 — 文本前缀** (`tierPrefixBase = 800`)
- query 长度 < text 长度，且 text 的前缀与 query 完全相同
- 例：`"saf"` 匹配 `"Safari"`

**Tier 3 — 单词前缀** (`tierWordPrefixBase = 600`)
- query 匹配某个单词（含 camelCase 分词）的前缀
- 跳过 `word.lowerBound == 0` 的情况（已被 T2 覆盖）
- 例：`"ex"` 匹配 `"Google Chrome — Expert Mode"` 中的 `"Expert"`

**Tier 4 — 连续子串** (`tierSubstringBase = 400`)
- query 作为连续子串出现在 text 中任意位置
- 使用 `findSubarray` 进行朴素字符串搜索
- 例：`"ari"` 匹配 `"Safari"`

**Tier 5 — 缩写匹配** (`tierAcronymBase = 200`)
- query 的每个字符依次匹配各单词的首字母
- 使用 `matchAcronym` 方法实现
- 例：`"gc"` 匹配 `"Google Chrome"`（G→Google, C→Chrome）

**Tier 6 — 模糊匹配** (`tierFuzzyBase = 100`)
- 仅当 `qLen >= 3` 时启用
- 使用 `wholeWordSpans`（仅按非字母数字分词，不含 camelCase）分割文本
- 对每个完整单词的前缀（长度在 `[qLen - maxEdits, qLen + maxEdits]` 范围内）执行 Damerau-Levenshtein 距离计算
- `maxEdits`：`qLen <= 9` 时为 1，否则为 2
- 扣分：`25 * editDistance + partialPenalty`（未匹配部分每字符扣 5 分，上限 30）
- 例：`"gthub"` 可匹配 `"GitHub"`（1 次编辑距离）

### Damerau-Levenshtein 距离 — `SearchTestable.damerauLevenshtein(_:_:k:)`

实现了带阈值 `k` 的 O(n*m) Damerau-Levenshtein 算法，支持插入、删除、替换和相邻交换四种操作。关键优化：

- 带状 DP（banded DP）：内层循环限制在 `[i-k, i+k]` 范围内，跳过不可能满足阈值的单元格
- 提前剪枝：如果某行的最小值超过 `k`，立即返回 `nil`
- 长度预检：`abs(n-m) > k` 时直接返回 `nil`

### 评分与加成 — `makeResult` 和 `computeBonuses`

每层匹配的基础分被限制在上限 `nextTierBase - 1`（防止低层级得分超过高层级基础分）。加成包括：

| 加成项 | 最大值 | 说明 |
|--------|--------|------|
| 位置加成 | 60 | `max(0, 60 - startIdx)`，越靠前越高 |
| 词首边界加成 | 15 | 匹配起始位置在词首 |
| 完整词加成 | 10 | 起止都在词边界（仅 T1-T4） |
| 长度比加成 | 30 | `min(30, 30 * qLen / tLen)` |
| 大小写精确加成 | 30 | `min(30, 5 * caseExactCount)`（仅 query 含大写字母且 T1-T5） |

### 搜索缓存 — `Search.ensureCache`

`Search` 类使用 `Window` 对象上的 `lastSearchQuery` 字段作为缓存键（格式：`normalizedQuery + "|3"`）。如果缓存命中，跳过重复计算。缓存失效通过查询变更自然触发（不同查询产生不同 cacheKey）。

缓存结果存储在 `Window` 的以下属性：
- `swAppResults: [SWResult]` — 应用名匹配结果
- `swTitleResults: [SWResult]` — 窗口标题匹配结果
- `swBestSimilarity: Double` — 最佳相似度（`max(appScore * 1.02, titleScore)`）

应用名得分乘以 1.02 的微小加权，在两者分数接近时优先选择应用名匹配。

### 搜索匹配判定 — `Search.matches` 和 `Search.relevance`

- `matches(_:query:)`：`swBestSimilarity > 0` 即为匹配
- `relevance(for:query:)`：直接返回 `swBestSimilarity` 用于排序
- 空查询始终匹配且相关度为 0

### 高亮显示 — `TileView.applySearchHighlight`

`TileView` 中的 `applySearchHighlight()` 方法负责将搜索结果可视化：

1. **应用图标高亮**：如果应用名有匹配（`swAppResults` 非空），在图标周围绘制高亮光晕（`appIconHighlight`），颜色使用 `Appearance.searchMatchHighlightColor`。
2. **标题文本高亮**：
   - 从 `searchSpanRanges()` 获取所有匹配范围（根据标题显示模式处理应用名/窗口标题的偏移）
   - 使用 `highlightedIndexes` 将范围转换为逐字符索引集合
   - 通过 `truncatedDisplay` 进行 Unicode 感知的文本截断（支持 `.byTruncatingHead`、`.byTruncatingMiddle`、`.byTruncatingTail` 三种模式）
   - 使用 `visibleHighlightRanges` 将原始索引映射到截断后的显示索引
   - 如果高亮区域被截断隐藏，则将省略号 `…` 也标记为高亮色
3. **结果输出**：构造 `NSMutableAttributedString`，在匹配范围上添加 `searchHighlightBackgroundKey`（背景色）和 `.foregroundColor`（前景色）属性。

### Unicode 感知截断 — `TileView.truncatedDisplay`

该方法正确处理 Unicode 字符（如 emoji、组合字符），返回三元组 `(text, visibleToOriginal, ellipsisIndex)`：

- `text`：截断后的显示文本
- `visibleToOriginal: [Int?]`：显示字符到原始字符索引的映射（省略号位置为 `nil`）
- `ellipsisIndex: Int?`：省略号在显示文本中的位置

三种截断模式使用二分搜索找到最大可显示字符数，确保文本宽度不超过 `maxWidth`。

## 会话管理

### SwitcherSession 类

`SwitcherSession` 是一个轻量的 `final class`，以静态属性 `current` 管理当前活跃会话：

```swift
static var current: SwitcherSession?
static var isActive: Bool { current != nil }
static var activeShortcutIndex: Int { current?.shortcutIndex ?? 0 }
```

### 会话生命周期

会话由 `App.showUiOrCycleSelection` 创建，由 `App.hideUi` 销毁：

1. **创建**：当 `SwitcherSession.current == nil` 时，`showUiOrCycleSelection` 创建新实例：
   - 记录 `initialPid = Applications.frontmostPid`（使用缓存值避免阻塞 IPC）
   - 设置 `isFirstSummon = true`
   - 在首次召唤时执行 `Windows.sortByLevel()` 排序

2. **存活期间**：
   - `isFirstSummon` 在首次显示后置为 `false`
   - `shortcutIndex` 记录当前使用的快捷键组
   - `holdMask` 从配置中读取当前 hold 快捷键的修饰键掩码
   - `selectedIndex` 追踪当前选中的窗口索引
   - `hoveredIndex` 追踪鼠标悬停的窗口索引
   - `searchQuery` 保存当前搜索查询字符串
   - `forceDoNothingOnRelease` 标记在修饰键释放时是否应跳过聚焦动作

3. **销毁**：`App.hideUi` 将 `SwitcherSession.current = nil`，同时清理搜索状态、上下文菜单、光标事件等。

### holdMask 追踪与修饰键释放检测

- `holdMask` 在 `showUiOrCycleSelection` 中从 `ControlsTab.shortcuts` 获取当前 hold 快捷键的 `modifierFlags`
- 修饰键释放检测在 `cgEventHandler`（`KeyboardEvents`）中进行：当 `flagsChanged` 事件的当前修饰键不再完全满足 `holdMask` 时，判定为释放
- 释放事件通过 `CGEventPostToPid` 重新投递给 `initialPid` 对应的应用，确保原始应用收到匹配的 press/release 对

### forceDoNothingOnRelease

此标志用于控制释放修饰键时的行为：

- 设为 `true` 的场景：
  - 搜索模式为 `searchOnRelease` 时（避免在搜索输入中意外聚焦窗口）
  - `showUiOrCycleSelection` 被调用且 `forceDoNothingOnRelease_` 参数为 `true` 时
  - 搜索编辑模式启用时（`TilesView.enableSearchEditing`）
- `ATShortcut.shouldTrigger()` 中检查此标志：仅当 `!session.forceDoNothingOnRelease && Preferences.effectiveShortcutStyle == .focusOnRelease` 时才在释放时触发聚焦

### selectedTarget

`selectedTarget` 存储当前选中窗口的 `id`（字符串标识），用于跨刷新周期跟踪选择状态。窗口列表更新时，通过此 ID 重新定位选中位置。

## 快捷键系统

### ATShortcut 类

`ATShortcut` 封装单个快捷键的完整生命周期，核心属性：

- `shortcut: Shortcut` — ShortcutRecorder 库的快捷键对象
- `id: String` — 快捷键标识符（如 `"holdShortcut0"`、`"nextWindowShortcut1"`）
- `scope: ShortcutScope` — `.global`（全局，通过 CGEventTap 捕获）或 `.local`（应用内，通过 NSEvent monitor 捕获）
- `triggerPhase: ShortcutTriggerPhase` — `.down`（按下触发）或 `.up`（释放触发）
- `state: ShortcutState` — 当前物理状态 `.down` / `.up`
- `index: Int?` — 快捷键组索引（仅 holdShortcut 和 nextWindowShortcut 有）

### 快捷键匹配 — `ATShortcut.matches`

匹配逻辑支持两种输入源：

1. **通过全局快捷键 ID 匹配**：从 `KeyboardEventsTestable.globalShortcutsIds` 字典查找
2. **通过修饰键 + 键码匹配**：将 Carbon 修饰键标志与快捷键配置比较

状态翻转检测（`flipped`）是关键：仅在状态实际发生变化时才触发 `.up` 阶段，避免误匹配。

### 修饰键匹配 — `ATShortcut.modifiersMatch`

三种匹配策略：

1. **holdShortcut**：使用超集匹配 — `modifiersCleaned == (modifiersCleaned | shortcutModifiersCleaned)`，允许用户在按住 hold 键的同时按其他键
2. **nextWindowShortcut（面板打开时）**：额外匹配"裸基础键"（去掉 holdShortcut 修饰键后的组合），支持用户在面板打开后松开 hold 键再按方向键
3. **其他快捷键**：精确匹配或精确匹配 + holdShortcut 修饰键

### 多组快捷键配置

系统支持 1-9 组快捷键（`Preferences.minShortcutCount = 1`, `maxShortcutCount = 9`，默认 `shortcutCount = 2`）：

- 每组包含：`holdShortcut{N}` + `nextWindowShortcut{N}`（N 从 0 开始）
- 命名使用 `Preferences.indexToName(_:_)` 方法，如 `indexToName("holdShortcut", 0)` → `"holdShortcut0"`
- 每组可独立配置过滤规则、外观样式、排序方式等覆盖（override）
- 第 2 组起（`index >= 1`）为 Pro 功能，通过 `ProFeature.extraShortcut(index:).attemptUse()` 门控

### 快捷键注册流程 — `ControlsTab.shortcutChangedCallback`

当用户在设置面板中修改快捷键时：

1. **holdShortcut 变更**：注册为 `.up` 阶段触发的 `.global` 快捷键，并联动更新同组的 `nextWindowShortcut`（通过 `restrictModifiers` 限制其修饰键必须包含 hold 键）
2. **nextWindowShortcut 变更**：通过 `combineHoldAndNextWindow` 将 hold 键和 next 键合并为完整快捷键组合
3. **其他快捷键**：直接注册，holdShortcut 相关的修饰键限制不适用

`combineShortcuts` 方法将两个 `Shortcut` 的 `modifierFlags` 合并，确保 nextWindowShortcut 始终包含 holdShortcut 的修饰键。

### 安全网措施 — `ATShortcut.redundantSafetyMeasures`

由于键盘事件可能乱序或丢失（来自不同事件源），此方法提供冗余保护：

1. **holdShortcut 释放检测**：如果当前物理修饰键状态已不满足 holdShortcut，但应用内状态仍为 `.down`，强制触发 holdShortcut 动作（关闭面板）
2. **计时器清理**：如果快捷键状态为 `.up`，停止对应的重复计时器

### shouldTrigger — 触发条件判断

- **全局 `.down`**：无活跃会话时（打开面板），或会话的快捷键索引匹配时
- **全局 `.up`**：有活跃会话，索引匹配，`!forceDoNothingOnRelease`，快捷键样式为 `.focusOnRelease`
- **本地**：有活跃会话且索引匹配

## 快捷键动作

### ShortcutAction 结构

```swift
struct ShortcutAction {
    let id: String
    let perform: () -> Void
}
```

### 动作注册表 — `ShortcutActions.all`

所有动作通过静态数组定义，按 ID 查找执行：

| ID | 动作 | 说明 |
|----|------|------|
| `focusWindowShortcut` | `App.focusTarget()` | 聚焦选中窗口 |
| `previousWindowShortcut` | `App.previousWindowShortcutWithRepeatingKey()` | 选择上一个窗口（带键盘重复） |
| `→` / `←` / `↑` / `↓` | `App.cycleSelection(_:)` | 方向键导航 |
| `vimCycleRight/Left/Up/Down` | `App.cycleSelection(_:)` | Vim 风格导航（h/j/k/l） |
| `cancelShortcut` | 取消搜索或隐藏面板 | 有搜索时关闭搜索，否则隐藏面板 |
| `closeWindowShortcut` | 关闭选中窗口 | `window.close()` |
| `minDeminWindowShortcut` | 最小化/恢复窗口 | `window.minDemin()` |
| `toggleFullscreenWindowShortcut` | 切换全屏 | `window.toggleFullscreen()` |
| `quitAppShortcut` | 退出应用 | `application.quit()` |
| `hideShowAppShortcut` | 隐藏/显示应用 | `application.hideOrShow()` |
| `searchShortcut` | 切换搜索模式 | 仅在面板打开时有效 |
| `lockSearchShortcut` | 锁定搜索模式 | Pro 功能，需搜索模式已开启 |

### 执行分发 — `ShortcutActions.execute(_:)`

执行逻辑包含 Pro 功能门控：

1. 对于 `holdShortcut` 和 `nextWindowShortcut` 前缀的 ID，检查索引是否 >= 1，若是则通过 `ProFeature.extraShortcut(index:).attemptUse()` 验证
2. 在 `all` 数组中查找精确匹配的 action 并执行
3. 未找到匹配时，根据 ID 前缀执行默认行为：
   - `holdShortcut` 前缀 → `App.focusTarget()`（修饰键释放时聚焦）
   - `nextWindowShortcut` 前缀 → `App.showUiOrCycleSelection(index, false)`

`cancelShortcut` 的行为有条件分支：如果搜索模式开启且快捷键样式非 `.searchOnRelease`，则关闭搜索模式而非隐藏面板。

## 键盘重复

### KeyRepeatTimer 类

`KeyRepeatTimer` 使用 GCD `DispatchSourceTimer` 在后台队列（`BackgroundWork.repeatingKeyQueue`，优先级 `.userInteractive`）上模拟系统键盘重复行为。

### 与系统设置同步

通过读取系统 UserDefaults 获取用户配置的重复率：

- **初始延迟**：`CachedUserDefaults.globalString("InitialKeyRepeat")` → `ticksToSeconds` 转换
- **重复间隔**：`CachedUserDefaults.globalString("KeyRepeat")` → `ticksToSeconds` 转换
- **Ticks 到秒**：Apple 硬编码 60 ticks = 1 秒（`Double(appleNumber)! / 60`），与刷新率无关

每次启动计时器时重新读取，确保用户在系统设置中修改后立即生效（注释说明 `NSEvent.keyRepeatInterval` 不会实时更新，所以使用 UserDefaults 方案）。

### 计时器生命周期

1. **启动** — `startTimerForRepeatingKey`：
   - 前置条件：`timerIsSuspended`，快捷键状态非 `.up`，全局快捷键需 hold 修饰键仍按下
   - 设置 `schedule(deadline: .now() + initialDelay, repeating: repeatRate)`
   - leeway 设为 `repeatRate * 100ms / 10`（即重复间隔的 1/10），在精度和功耗间平衡

2. **触发** — `handleEvent`：
   - 在主线程异步执行
   - 检查快捷键是否已释放或 hold 键已松开，若是则停止计时器
   - 否则执行传入的动作闭包

3. **停止** — `stopTimerForRepeatingKey`：
   - 仅当传入的快捷键名称与当前活跃计时器匹配时才停止
   - 挂起计时器并重置状态

### 边界行为 — 不循环

键盘重复在到达列表末尾时不循环。`Windows.cycleSelectedWindowIndex` 方法中的判断逻辑：

```swift
if ((step > 0 && nextIndex < session.selectedIndex) ||
    (step < 0 && nextIndex > session.selectedIndex)) &&
    (!allowWrap || ATShortcut.lastEventIsARepeat || !KeyRepeatTimer.timerIsSuspended) {
    return // 不循环
}
```

当检测到以下条件同时满足时，停止而不是循环：
- 计算出的下一个索引"绕回"了（`nextIndex < selectedIndex` 或 `nextIndex > selectedIndex`）
- 这是键盘重复事件（`ATShortcut.lastEventIsARepeat`）或计时器仍在运行

手动按键（非重复）时，`allowWrap` 为 `true` 且非重复事件，因此会正常循环。

### 两种重复启动器

- `startRepeatingKeyPreviousWindow`：为 `previousWindowShortcut`（通常是 Shift+Tab 等有实际键码的快捷键）启动计时器。仅在快捷键无键码时启动（有键码的快捷键由系统自动重复）。
- `startRepeatingKeyNextWindow`：为 `nextWindowShortcut` 启动计时器，始终启动（因为 nextWindowShortcut 通常是无键码的修饰键组合）。

### hold 修饰键释放检测

`holdModifierIsReleased()` 方法作为额外的安全网，通过 `ModifierFlags.current` 轮询硬件修饰键状态：

- 将当前修饰键与 holdShortcut 的修饰键进行按位与操作
- 如果结果不包含全部 hold 修饰键，判定为已释放
- 这解决了主线程繁忙时事件更新延迟的问题

## 预览面板

### PreviewPanel 类

`PreviewPanel` 继承自 `NSPanel`，是一个浮动的、不可激活的面板，用于显示选中窗口的大尺寸截图预览。

### 初始化配置

- **StyleMask**：`.nonactivatingPanel` + `.titled` + `.fullSizeContentView`
- `isFloatingPanel = true`：始终浮动在其他窗口之上
- `animationBehavior = .none`：禁用系统动画
- `hidesOnDeactivate = false`：应用失焦时不隐藏
- `titleVisibility = .hidden` + `titlebarAppearsTransparent = true`：隐藏标题栏
- `backgroundColor = .clear`：透明背景
- `constrainFrameRect` 被重写为直接返回 `frameRect`，允许面板的 `origin.y = 0` 时覆盖菜单栏上方
- `collectionBehavior = .canJoinAllSpaces`：跟随 Space 切换
- `setAccessibilitySubrole(.unknown)`：在窗口枚举时被过滤，避免自身出现在缩略图中

### 内容视图层级

- `previewView: LightImageView` — 主内容视图，持有 `CALayerContents`（通常是 `IOSurface` 截图）
- `borderView: BorderView` — 绘制 5pt 圆角边框的子视图，使用系统强调色（`NSColor.systemAccentColor.withAlphaComponent(0.5)`）

### 显示逻辑 — `PreviewPanel.show`

```swift
static func show(_ id: CGWindowID, _ preview: CALayerContents, _ position: CGPoint, _ size: CGSize)
```

1. **位置和尺寸更新**：调用 `repositionAndResize`，将 Quartz 坐标（Y 轴原点在左下）转换为 Cocoa 坐标（Y 轴原点在左上）
2. **内容更新**：仅当窗口 ID 变化时更新 `previewView` 内容，避免重复渲染
3. **显示动画**：如果 `Preferences.previewFadeInAnimation` 为 `true`，使用 `NSAnimationContext` 执行 0.3 秒淡入
4. **层级管理**：
   - `order(.below, relativeTo: TilesPanel.shared.windowNumber)` — 预览始终在主面板下方
   - 显式设置 `level = TilesPanel.shared.level - 1`，解决多显示器场景下的 Z 轴排序闪烁问题

### 增量更新 — `updateIfShowing`

当预览面板已显示且 ID 匹配时，仅更新位置和内容，不重新排序。用于窗口位置/大小变化时实时更新预览。

### 清理 — `clearIfShowing`

当窗口被移除时，如果预览正在显示该窗口，释放 `previewView` 中的 `IOSurface`。避免已关闭窗口的全分辨率截图残留在内存中。

### 坐标转换 — `repositionAndResize`

```swift
frame.origin.y = NSScreen.screens.first!.frame.maxY - frame.maxY
```

始终以主屏幕为参考进行 Y 轴翻转，因为 Quartz 使用左下角原点而 Cocoa 使用左上角原点。

## 文件清单

| 文件路径 | 行数 | 核心类型 | 说明 |
|----------|------|----------|------|
| `src/switcher/Search.swift` | 38 | `Search` | 搜索入口，缓存管理，双字段评分合并 |
| `src/switcher/SearchTestable.swift` | 431 | `SearchTestable`, `SWOp`, `SWResult`, `MatchResult` | 搜索算法实现（归一化、分层匹配、DL 距离） |
| `src/switcher/SwitcherSession.swift` | 33 | `SwitcherSession` | 会话状态（选择、搜索、修饰键追踪） |
| `src/switcher/ATShortcut.swift` | 145 | `ATShortcut`, `ShortcutTriggerPhase`, `ShortcutState`, `ShortcutScope` | 快捷键匹配、状态机、修饰键清洗 |
| `src/switcher/ShortcutAction.swift` | 71 | `ShortcutAction`, `ShortcutActions` | 动作注册表与执行分发 |
| `src/switcher/KeyRepeatTimer.swift` | 79 | `KeyRepeatTimer` | GCD 定时器模拟系统键盘重复 |
| `src/switcher/PreviewPanel.swift` | 91 | `PreviewPanel`, `BorderView` | 预览浮层面板 |
