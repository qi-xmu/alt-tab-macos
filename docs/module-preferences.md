# 偏好系统

## 模块职责

偏好系统（`src/preferences/`）是 AltTab 应用的配置中枢，负责：

1. **偏好持久化** — 基于 `UserDefaults` 的类型安全存取层，支持宏枚举、快捷键、JSON 对象等多种数据类型
2. **运行时缓存** — 通过 `ConcurrentMap` 提供零分配的高频读取路径，写入时自动失效
3. **Pro 功能门控** — Pro 功能锁定时自动降级偏好值，解锁时从快照恢复，支持"免费通行"会话
4. **Per-Shortcut 覆盖** — 每个快捷键可独立覆盖全局外观/行为设置
5. **版本迁移** — 跨版本偏好结构变化时自动迁移
6. **设置 UI** — 模拟 macOS 系统设置风格的侧边栏 + 分区布局，含模糊搜索和高亮

## 核心组件

### Preferences（主入口）

`Preferences` 类是所有偏好读写的唯一入口，定义于 `src/preferences/Preferences.swift`。

**初始化流程** (`Preferences.initialize()`)：
1. `PreferencesMigrations.removeCorruptedPreferences()` — 清除已知的损坏数据
2. `PreferencesMigrations.migratePreferences()` — 执行版本间迁移
3. `registerDefaults()` — 通过 `UserDefaults.standard.register(defaults:)` 注册默认值

**默认值** (`defaultValues`)：一个静态字典，包含所有偏好的默认值。其中：
- 布尔值存为字符串 `"true"` / `"false"`
- 宏枚举存为索引字符串 `"0"` / `"1"` / `"2"`...
- 快捷键存为 `[String: Any]` 字典，包含 `"string"` 和 `"secureData"` 两个字段
- 异常列表 (`exceptions`) 存为 JSON 编码字符串
- Per-shortcut 偏好通过 `indexToName(_:_:)` 生成带数字后缀的键名（如 `appsToShow` → `appsToShow2`）

**写入路径** (`Preferences.set(_:_:_:)`)：
```
1. 写入 UserDefaults.standard
2. CachedUserDefaults.removeFromCache(key) — 移除该键的缓存
3. invalidateAllCache() — 清空 cachedAll 快照
4. PreferencesEvents.preferenceChanged(key) — 通知观察者
```

**快捷键写入** (`Preferences.setShortcut(_:_:_:)`)：额外逻辑是将 `Shortcut` 对象通过 `NSKeyedArchiver` 序列化为 `Data`，与字符串表示一同存入字典。

**全量快照** (`Preferences.all`)：从 `UserDefaults.standard.persistentDomain(forName:)` 读取，过滤只包含 `ownedKeys` 中的键，结果缓存于 `cachedAll`。每次写入操作（`set` / `setShortcut` / `remove` / `resetAll`）都会通过 `invalidateAllCache()` 清除此缓存。

**异常条目** (`ExceptionEntry`)：`Codable` 结构体，包含 `bundleIdentifier`、`hide`（`ExceptionHidePreference`）、`ignore`（`ExceptionIgnorePreference`）和 `windowTitleContains: [String]?`。解码器兼容旧版单字符串格式和新版数组格式。

### CachedUserDefaults（缓存层）

`CachedUserDefaults` 是一个纯静态工具类，为 `UserDefaults` 的读取提供线程安全的内存缓存。

**核心数据结构**：
```swift
static var cache = ConcurrentMap<String, Any>()
```

`ConcurrentMap` 是一个基于锁的线程安全字典，确保在多线程环境下的安全访问。

**缓存策略**：
- **命中**：直接返回缓存值，无 `UserDefaults` I/O
- **未命中**：从 `UserDefaults` 读取，转换为目标类型，写入缓存
- **失效**：
  - `removeFromCache(_:)` — 写入时移除单个键
  - `invalidateAllCache()` — 清空 `Preferences.cachedAll`（不是 `CachedUserDefaults.cache`）

**读取方法**：
| 方法 | 返回类型 | 说明 |
|------|---------|------|
| `string(_:)` | `String` | 直接读取字符串 |
| `shortcut(_:)` | `Shortcut?` | 反序列化快捷键数据，用 `NSNull()` 标记 `nil` |
| `int(_:)` | `Int` | 字符串转整数 |
| `bool(_:)` | `Bool` | 字符串转布尔 |
| `double(_:)` | `Double` | 字符串转双精度 |
| `macroPref<A>(_:_)` | `A` | 字符串索引转宏枚举值 |
| `intFromMacroPref(_:_:)` | `Int` | 宏枚举转索引 |
| `json<T>(_:_)` | `T` | JSON 解码 |

**损坏恢复** (`getThenConvertOrReset`)：当字符串无法转换为目标类型时，移除该键（回退到注册默认值），再重新读取默认值。

## 宏枚举偏好

宏枚举偏好（MacroPreference）是整个偏好系统的核心抽象，定义于 `src/preferences/MacroPreferences.swift`。

**协议层次**：
```
MacroPreference (协议)
  └── localizedString: LocalizedString { get }

SfSymbolMacroPreference (协议, 继承 MacroPreference)
  └── symbol: Symbols { get }

ImageMacroPreference (协议, 继承 MacroPreference)
  └── image: WidthHeightImage { get }
```

**存储格式**：所有宏枚举偏好以 **枚举索引的字符串表示** 存储。例如 `AppearanceStylePreference.thumbnails`（index=0）存为 `"0"`，`.appIcons`（index=1）存为 `"1"`。这通过 `CaseIterable` 的 `allCases` 数组的索引实现。

**扩展属性**：
- `indexAsString: String` — 枚举索引的字符串形式
- `index: Int` — 枚举在 `allCases` 中的位置

**已定义的宏枚举（26 个）**：

| 枚举类型 | Case 数 | 协议 | 用途 |
|---------|---------|------|------|
| `MenubarIconPreference` | 3 | MacroPreference | 菜单栏图标样式 |
| `GesturePreference` | 5 | MacroPreference | 触控板手势 |
| `LanguagePreference` | 21 | MacroPreference | 界面语言 |
| `ShortcutStylePreference` | 3 | SfSymbolMacroPreference | 释放按键后行为 |
| `ShowHowPreference` | 3 | MacroPreference | 显示/隐藏/末尾显示 |
| `WindowOrderPreference` | 4 | MacroPreference | 窗口排序方式 |
| `AppsToShowPreference` | 3 | MacroPreference | 显示哪些应用 |
| `SpacesToShowPreference` | 3 | MacroPreference | 显示哪些空间 |
| `ScreensToShowPreference` | 2 | MacroPreference | 显示哪些屏幕 |
| `ShowOnScreenPreference` | 3 | MacroPreference | 在哪个屏幕显示 |
| `TitleTruncationPreference` | 3 | MacroPreference | 标题截断位置 |
| `ShowAppsOrWindowsPreference` | 2 | MacroPreference | 按应用/窗口分组 |
| `GroupTabsPreference` | 2 | MacroPreference | 标签页分组方式 |
| `CursorFollowFocus` | 3 | MacroPreference | 光标跟随焦点 |
| `ShowTitlesPreference` | 3 | MacroPreference | 标题显示内容 |
| `AppearanceStylePreference` | 3 | ImageMacroPreference | 外观样式（缩略图/图标/标题）|
| `AppearanceSizePreference` | 4 | SfSymbolMacroPreference | 外观尺寸 |
| `ThemePreference` | 2 | ImageMacroPreference | 主题（macOS/Windows 10）|
| `AppearanceThemePreference` | 3 | SfSymbolMacroPreference | 明暗主题 |
| `UpdatePolicyPreference` | 3 | MacroPreference | 更新策略 |
| `CrashPolicyPreference` | 3 | MacroPreference | 崩溃报告策略 |
| `ExceptionHidePreference` | 4 | MacroPreference, Codable | 异常隐藏行为 |
| `ExceptionIgnorePreference` | 3 | MacroPreference, Codable | 异常忽略行为 |

## Pro 门控偏好

Pro 门控系统允许 Free 用户看到 Pro 功能的 UI（灰显/徽章），但阻止其选择 Pro 值。核心定义在 `src/preferences/PreferenceDefinition.swift`。

### PreferenceGate<T>

```swift
struct PreferenceGate<T: MacroPreference & CaseIterable & Equatable> {
    let freeEquivalent: T        // Free 用户的等价值
    let rememberedKey: String    // 快照存储键
    let isProValue: (T) -> Bool  // 判断是否为 Pro 值的闭包
}
```

三个 Pro 门控偏好：
| 偏好 | freeEquivalent | Pro 值判定 |
|------|---------------|-----------|
| `appearanceStyle` | `.thumbnails` | `!= .thumbnails` |
| `appearanceSize` | `.medium` | `== .auto` |
| `shortcutStyle` | `.doNothingOnRelease` | `== .searchOnRelease` |

### PreferenceDefinition<T>

```swift
struct PreferenceDefinition<T: MacroPreference & CaseIterable & Equatable> {
    let key: String
    let `default`: T
    let gate: PreferenceGate<T>?
}
```

**读取逻辑** (`read()`)：
1. 通过 `CachedUserDefaults.macroPref` 读取存储值
2. 若无门控或 Pro 未锁定 → 直接返回
3. 若 **免费通行会话** 激活且有记忆索引 → 返回记忆的 Pro 值
4. 若存储值仍是 Pro 值 → 返回 `freeEquivalent`
5. 否则 → 返回已降级的存储值

**快照与降级** (`snapshotAndDowngrade()`)：
1. 读取当前存储值
2. 若为 Pro 值 → 用 `freeEquivalent` 覆盖写入，返回原始索引
3. 调用方将索引存入 `rememberedKey`

**恢复** (`restore(from:)`)：从记忆索引恢复 Pro 值，写入时 `notify: false` 避免观察者反弹到升级页面。

**类型擦除** (`AnyProGatedPreference`)：将 `PreferenceDefinition<T>` 擦除为统一接口，只暴露 `Int` 索引和 `Bool` 返回值。`ProTransitionState` 在锁定/解锁时遍历 `ProGatedPreferences.all` 数组。

**注册的 Pro 门控偏好（6 个）**：
- `appearanceStyle` / `appearanceSize` / `shortcutStyle`（全局）
- `appearanceStyleOverride0` / `appearanceSizeOverride0` / `shortcutStyleOverride0`（快捷键 0 的覆盖）

### Pro 锁定 UI 交互

在设置界面中，Pro 门控偏好通过以下机制体现：
- **分段控件**：Pro 值段上叠加 `ProBadgeView`，点击时拦截并导航到升级页
- **单选按钮组**：Pro 选项显示 Pro 徽章，点击时恢复已存储的选择并导航到升级页
- **窗口 key 状态**：徽章的图标/文字颜色响应窗口 key 状态变化（`onWindowKeyChanged`）

## Per-Shortcut 覆盖

Per-Shortcut 覆盖系统允许每个快捷键拥有独立于全局的外观和行为设置。

**5 个可覆盖偏好** (`Preferences.appearanceOverrideBaseNames`)：
```
"appearanceStyleOverride"    → 覆盖 "appearanceStyle"
"appearanceSizeOverride"     → 覆盖 "appearanceSize"
"appearanceThemeOverride"    → 覆盖 "appearanceTheme"
"shortcutStyleOverride"      → 覆盖 "shortcutStyle"
"previewFocusedWindowOverride" → 覆盖 "previewFocusedWindow"
```

**索引键名规则** (`indexToName(_:_:)`)：
- `index == 0` → 无后缀（如 `appearanceStyleOverride`）
- `index == 1` → 后缀 `2`（如 `appearanceStyleOverride2`）
- `index == 2` → 后缀 `3`，以此类推

**覆盖判定** (`hasOverride(_:_:)`)：通过 `Preferences.all`（读取 `persistentDomain`）检查，排除了注册默认值，因此未设置的覆盖不会被误判为已覆盖。

**有效值读取** (`effectiveXxx(_:)`)：优先级为 覆盖值 > 全局值。对于 Pro 门控的覆盖（index==0），通过 `ProGatedPreferences.xxxOverride0.read()` 读取。

**覆盖点击决策** (`OverrideClickResolver`)：
```
override SET:  displayed = storedOverride
override UNSET: displayed = global

click on displayed value → .skip (不做任何操作)
click on any other value → .write(value) (写入覆盖)
```

**取消覆盖** (`removeOverride(_:_:)`)：删除键并通知 `ProTransitionState` 清除记忆索引。

**快捷键配置打包** (`ShortcutConfiguration`)：将一个快捷键的所有 9 个过滤/排序/样式偏好打包为一个值对象，替代了之前 9 个并行数组的索引方式。

**覆盖指示器**：在全局外观设置行末尾，若某偏好被快捷键覆盖，显示 `arrow.triangle.branch`（旋转 180 度）图标按钮，提示哪些快捷键覆盖了该值，点击导航到对应快捷键的外观面板。

## 偏好迁移

`PreferencesMigrations` 类处理跨版本偏好结构变化，定义于 `src/preferences/PreferencesMigrations.swift`。

**版本追踪**：键 `"preferencesVersion"` 存储上次迁移的应用版本号。

**迁移流程** (`migratePreferences()`)：
1. 读取当前版本号
2. `ProTransitionState.markFreshInstallIfUnknown()` — 标记全新安装
3. 若版本低于当前应用版本 → 执行 `updateToNewPreferences()`
4. 更新版本号为当前应用版本

**迁移规则** (`shouldRun`)：当 `versionInPlist < versionThreshold` 时运行迁移。

**已注册的迁移（按版本排序）**：

| 版本 | 迁移方法 | 说明 |
|------|---------|------|
| 6.18.1 | `migrateMaxSizeOnScreenToWidthAndHeight` | `maxScreenUsage` 拆分为 `maxWidthOnScreen` + `maxHeightOnScreen` |
| 6.18.1 | `migrateMenubarIconFromCheckboxToDropdown` | 隐藏菜单栏图标复选框 → 下拉菜单 |
| 6.18.1 | `migrateShowWindowsCheckboxToDropdown` | 显示最小化/隐藏/全屏窗口复选框 → 下拉菜单 |
| 6.18.1 | `migrateDropdownsFromTextToIndexes` | 下拉菜单从英文文本 → 索引数字 |
| 6.18.1 | `migrateNextWindowShortcuts` | 移除 nextWindowShortcut 中与 holdShortcut 重叠的修饰键 |
| 6.23.0 | `migrateShowWindowsFrom` | "Active Space" 选项移除后的重新映射 |
| 6.27.1 | `migrateLoginItem` | 从 Login Items API 迁移（已废弃 API） |
| 6.28.1 | `migrateMinMaxWindowsWidthInRow` | `0` → `1` 的最小值修正 |
| 6.43.0 | `migrateExceptions` | 两个独立黑名单合并为统一异常列表 |
| 7.0.0 | `migratePreferencesIndexes` | spacesToShow / titleTruncation 索引重对齐 |
| 7.8.0 | `migrateMenubarIconWithNewShownToggle` | 菜单栏图标"隐藏"选项拆分为独立开关 |
| 7.13.1 | `migrateGestures` | 手势从 3 选项扩展为 5 选项 |
| 7.25.0 | `migrateHideWindowlessApps` | 全局 → per-shortcut |
| 7.26.0 | `migrateShowWindowlessApps` | 新增 `show` 选项，索引重映射 |
| 7.27.0 | `migrateCursorFollowFocus` | 布尔 → 三值枚举 |
| 9.0.0 | `migrateShortcutIndexes` | 第 3 快捷键索引从 3→9 |
| 10.12.0 | `migrateBlacklistToExceptions` | `blacklist` → `exceptions` 重命名 |
| 10.12.0 | `migrateLanguagePreferenceIndex` | 语言列表从 59→21 项，索引重映射 |
| 10.12.0 | `migrateExceptionsTitleArray` | `windowTitleContains: String?` → `[String]?` |
| 10.13.0 | `migrateGroupingToPerShortcut` | `showAppsOrWindows` / `showTabsAsWindows` 从全局 → per-shortcut |

**损坏修复** (`removeCorruptedPreferences`)：清除 `holdShortcut` 系列键的空字符串值。

## 设置窗口

设置窗口模拟 macOS 系统设置的设计风格，基于纯 AppKit（无 SwiftUI / Interface Builder），定义于 `src/preferences/settings-window/SettingsWindow.swift`。

### 窗口结构

```
SettingsWindow (NSWindow)
├── NSSplitViewController
│   ├── sidebarContainer (175pt 固定宽度)
│   │   ├── SidebarSearchField — 搜索框
│   │   ├── SidebarTableView — 侧边栏导航
│   │   ├── UpgradeButton — Pro 状态/升级按钮
│   │   └── QuitButton — 退出按钮
│   └── contentContainer
│       └── rightScrollView → sectionsDocumentView
│           └── sectionsStack — 垂直排列的 section 容器
│               ├── section "Appearance" → AppearanceTab.initTab()
│               ├── section "Controls" → ControlsTab.initTab()
│               ├── section "General" → GeneralTab.initTab()
│               └── section "Exceptions" → ExceptionsTab.initTab()
```

**窗口尺寸**：宽度 = 175（侧边栏）+ 710（内容区）+ 5 + 5 + 30（左右边距）= 925pt。高度默认 570pt，最小 400pt，最大不限。窗口位置通过 `setFrameAutosaveName("SettingsWindow")` 自动持久化。

### 侧边栏 (SidebarList / SettingsSidebarCellView)

侧边栏使用 `NSTableView` 的 `.sourceList` 样式，每行显示一个 section 的图标和标题。选中行有系统蓝色高亮效果。

**Tab 导航**：`SidebarSearchField` 和 `SidebarTableView` 通过 `nextValidKeyView` / `previousValidKeyView` 重写实现搜索框 ↔ 侧边栏的闭合 Tab 循环，阻止 AppKit 发现内部子控件。

**滚动联动**：内容区滚动时，根据 `sectionSelectionTriggerRatio`（向下滚动 0.4，向上 0.6）自动更新侧边栏选中项。点击侧边栏项则反向滚动内容区。

### 搜索系统 (SettingsSearch)

搜索系统定义于 `src/preferences/settings-window/SettingsSearch.swift`，实现模糊匹配算法：

**匹配流程**：
1. **分词** — 按非字母数字字符分割，每个 token 规范化（去重音、去大小写、去全角半角差异）
2. **Token 匹配** — 对每个查询 token，在所有文本 token 中找最佳匹配
3. **评分** — 综合以下指标加权：
   - Damerau-Levenshtein 编辑距离（权重 0.64）
   - 公共前缀长度（权重 0.23）
   - 最长公共子序列覆盖率（权重 0.13）
4. **阈值** — 按查询长度动态调整最低匹配分数（3 字符 → 0.74，8+ 字符 → 0.56）
5. **高亮** — 匹配范围用圆角矩形黄色背景高亮

**搜索范围**：遍历所有 section 的视图树，收集 `NSTextField`、`NSPopUpButton`、`NSSegmentedControl`、`NSButton`、`ClickHoverImageView`、`NSTextView` 的文本内容。

**Sheet 搜索**：打开的 sheet 窗口也会应用搜索高亮，通过 `beginSheetWithSearchHighlight(_:)` 实现。

### 标签页

#### AppearanceTab（外观）

定义于 `src/preferences/settings-window/tabs/appearance/AppearanceTab.swift`。

**控件**：
- **Style** — 3 个图片单选按钮（Thumbnails / App Icons / Titles），非 Thumbnails 的带 Pro 徽章
- **Size** — 分段控件（Small / Medium / Large / Auto），Auto 段有 Pro 徽章和锁定拦截
- **Theme** — 分段控件（Light / Dark / System）
- **After keys are released** — 分段控件（Focus / Hold / Search），Search 段有 Pro 徽章
- **Preview selected window** — 开关（当 Style 不是 Thumbnails 时禁用）
- **Show on** — 下拉菜单（多屏幕设置）
- **Customize more…** — 打开 `CustomizeStyleSheet` sheet
- **Animations…** — 打开 `AnimationsSheet` sheet

**Pro 锁定刷新**：监听 `ProTransitionManager.proLockStateDidChangeNotification`，调用 `refreshProLockUi()` 重新同步 3 个 Pro 感知控件。

**覆盖指示器**：每个全局设置行末尾有一个小图标按钮，当某快捷键覆盖了该设置时显示，提示覆盖的快捷键编号，点击导航到该快捷键的外观面板。

#### ControlsTab（控件）

定义于 `src/preferences/settings-window/tabs/controls/ControlsTab.swift`。

**布局**：侧边栏 + 编辑器的分栏布局。

**侧边栏**：
- 每个快捷键一行（`SidebarListRow`），显示标题和快捷键组合
- 手势设置行（分隔线隔开）
- +/- 按钮调整快捷键数量（最多 9 个，最少 1 个）
- 快捷键 2+ 带 Pro 徽章

**编辑器**：每个快捷键/手势有独立的编辑器视图，包含：
- **Trigger** — Hold + Next Window 快捷键录制器
- **三个 Tab**（分段控件切换）：
  - **Filtering** — 7 行下拉菜单（应用/空间/屏幕/最小化/隐藏/全屏/无窗口应用）
  - **Appearance** — 5 行覆盖控件（Style/Size/Theme/ShortcutStyle/PreviewSelectedWindow），每行有取消链接按钮
  - **Ordering & Grouping** — 3 行下拉菜单（分组方式/标签页分组/排序方式）

**快捷键管理**：
- `addShortcutSlot()` — 新增快捷键（Pro 锁定时导航到升级页）
- `removeShortcutSlot()` — 删除快捷键并平移后续偏好
- `copyShortcutPreferences(_:_)` — 在删除时平移偏好数据
- `shortcutChangedCallback(_:)` — 快捷键变更回调，处理 Hold + Next Window 的组合逻辑

**覆盖控件同步**：当全局偏好变更时，`syncOverrideControlsToGlobal()` 将所有未覆盖的控件重新同步到全局值。

#### GeneralTab（通用）

定义于 `src/preferences/settings-window/tabs/GeneralTab.swift`。

**控件**：
- Start at login — 开关
- Menubar icon — 下拉菜单 + 显示开关（支持拖拽移除）
- Capture windows in the background — 开关（附详细说明）
- Language — 下拉菜单（21 种语言，变更需重启）
- Updates policy — 下拉菜单 + "Check for updates now…" 按钮
- Crash reports policy — 下拉菜单
- Export settings… / Import settings… / Reset settings and restart…

#### ExceptionsTab（异常）

定义于 `src/preferences/settings-window/tabs/ExceptionsTab.swift`。

**布局**：侧边栏 + 编辑器的分栏布局。

**侧边栏**：每个异常条目一行，显示应用图标和 Bundle ID，+/- 按钮管理条目。

**编辑器** (`ExceptionEditorView`)：编辑选中条目的隐藏行为（`ExceptionHidePreference`）、忽略行为（`ExceptionIgnorePreference`）和窗口标题匹配模式。

#### UpgradeTab（升级）

定义于 `src/preferences/settings-window/tabs/UpgradeTab.swift`。

当用户点击侧边栏的 Upgrade 按钮时，内容区切换到升级页面（隐藏 section 堆栈，显示升级视图）。包含 Pro 状态显示、用量统计、功能列表、激活/管理许可证等。

### 基础 UI 组件

#### TableGroupView

定义于 `src/preferences/settings-window/TableGroupView.swift`。模拟 macOS 系统设置的圆角矩形分组容器。

- **行结构**：左标签 + 右控件 + 可选副标题 + 分隔线
- **圆角**：首行上圆角、末行下圆角、单行四圆角
- **悬停效果**：行背景变为 `tableHoverColor`，分隔线扩展至全宽
- **点击**：macOS 风格 mouseUp 触发
- **常量**：`cornerRadius = 5`，`borderWidth = 1`，`padding = 10`，`spacing = 10`

#### TableGroupSetView

`TableGroupView` 的容器，将多个 `TableGroupView` 和其他视图按垂直方向排列，连续的 `TableGroupView` 会紧密排列（5pt 间距），非 `TableGroupView` 元素水平排列。

#### LabelAndControl

定义于 `src/preferences/settings-window/LabelAndControl.swift`。控件工厂类，提供统一的控件创建接口：

| 方法 | 控件类型 |
|------|---------|
| `makeDropdown` | `NSPopUpButton` |
| `makeSegmentedControl` | `NSSegmentedControl` |
| `makeSwitch` | `Switch`（自定义开关）|
| `makeCheckbox` | `NSButton`（复选框）|
| `makeLabelWithRecorder` | `CustomRecorderControl`（快捷键录制器）|
| `makeLabelWithSlider` | `NSSlider` |
| `makeLabelWithRadioButtons` | `NSButton`（单选按钮组）|
| `makeImageRadioButtons` | `ImageTextButtonView`（图片单选按钮组）|
| `makeInfoButton` | `ClickHoverImageView`（信息提示按钮）|
| `makeLabelWithCheckboxAndInfoButton` | 复选框 + 信息按钮组合 |

所有控件通过 `setupControl(_:_:_:)` 统一设置 `identifier` 和 `onAction` 回调，`onAction` 中调用 `controlWasChanged` 将新值写入偏好系统。

#### SidebarListRow

定义于 `src/preferences/settings-window/SidebarList.swift`。可点击/悬停的列表行，用于 ControlsTab 和 ExceptionsTab 的侧边栏。

特性：
- 图标 + 标题 + 摘要 + 箭头的水平布局
- 选中/悬停/普通三种视觉状态
- Pro 徽章（`ProBadgeView`）
- 窗口 key 状态响应（选中时文字颜色变化）
- 应用图标异步加载（通过 `resolvedToken` 去重）

## 文件清单

| 文件路径 | 核心类型 | 职责 |
|---------|---------|------|
| `src/preferences/Preferences.swift` | `Preferences`, `CachedUserDefaults`, `ExceptionEntry` | 偏好读写、缓存、异常条目模型 |
| `src/preferences/MacroPreferences.swift` | 26 个枚举 + `MacroPreference` 协议 | 宏枚举偏好定义 |
| `src/preferences/PreferenceDefinition.swift` | `PreferenceGate`, `PreferenceDefinition`, `AnyProGatedPreference`, `ProGatedPreferences` | Pro 门控偏好定义 |
| `src/preferences/PreferencesMigrations.swift` | `PreferencesMigrations` | 跨版本偏好迁移 |
| `src/preferences/ShortcutConfiguration.swift` | `ShortcutConfiguration` | 快捷键配置打包 |
| `src/preferences/settings-window/SettingsWindow.swift` | `SettingsWindow`, `UpgradeButton`, `SettingsSidebarCellView` | 设置窗口主框架 |
| `src/preferences/settings-window/SidebarList.swift` | `SidebarListRow`, `SidebarListContainer` | 侧边栏列表行 |
| `src/preferences/settings-window/LabelAndControl.swift` | `LabelAndControl` | 控件工厂 |
| `src/preferences/settings-window/SettingsSearch.swift` | `SettingsSearch` | 模糊搜索算法 |
| `src/preferences/settings-window/TableGroupView.swift` | `TableGroupView`, `TableGroupSetView`, `ClickHoverStackView` | 分组容器和行 |
| `src/preferences/settings-window/tabs/appearance/AppearanceTab.swift` | `AppearanceTab`, `IllustratedImageThemeView`, `Popover` | 外观设置标签页 |
| `src/preferences/settings-window/tabs/appearance/AnimationsSheet.swift` | `AnimationsSheet` | 动画设置 Sheet |
| `src/preferences/settings-window/tabs/appearance/CustomizeStyleSheet.swift` | `CustomizeStyleSheet` | 自定义样式 Sheet |
| `src/preferences/settings-window/tabs/controls/ControlsTab.swift` | `ControlsTab` | 控件设置标签页 |
| `src/preferences/settings-window/tabs/controls/AdditionalControlsSheet.swift` | `AdditionalControlsSheet` | 附加控件 Sheet |
| `src/preferences/settings-window/tabs/controls/ShortcutsWhenActiveSheet.swift` | `ShortcutsWhenActiveSheet` | 激活时快捷键 Sheet |
| `src/preferences/settings-window/tabs/controls/OverrideClickResolver.swift` | `OverrideClickResolver`, `OverrideClickDecision` | 覆盖点击状态机 |
| `src/preferences/settings-window/tabs/GeneralTab.swift` | `GeneralTab` | 通用设置标签页 |
| `src/preferences/settings-window/tabs/ExceptionsTab.swift` | `ExceptionsTab` | 异常设置标签页 |
| `src/preferences/settings-window/tabs/ExceptionEditorView.swift` | `ExceptionEditorView` | 异常条目编辑器 |
| `src/preferences/settings-window/tabs/UpgradeTab.swift` | `UpgradeTab` | 升级/Pro 管理标签页 |
| `src/preferences/settings-window/tabs/AboutTab.swift` | `AboutTab` | 关于标签页 |
| `src/preferences/settings-window/tabs/AcknowledgmentsTab.swift` | `AcknowledgmentsTab` | 致谢标签页 |
