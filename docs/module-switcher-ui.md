# 切换器 UI

## 模块职责

切换器 UI 模块负责 AltTab 核心交互界面——一个浮动面板（`NSPanel`），以网格形式展示当前打开的窗口/应用缩略图、标题和图标，供用户通过键盘快捷键或鼠标进行窗口切换。该模块完全采用纯 Swift + AppKit 手动布局实现，不使用 Interface Builder、SwiftUI 或 Auto Layout，所有视图定位和尺寸计算由自定义两遍布局算法完成，并配合视图回收池、CALayer 隐式动画禁用等技术手段来确保高性能和低延迟。

## 核心组件

### TilesPanel (`TilesPanel.swift`)

`TilesPanel` 是整个切换器的顶层窗口，继承自 `NSPanel`，配置为：

- `styleMask: .nonactivatingPanel` — 不抢占应用焦点
- `isFloatingPanel = true` — 始终浮动在最前
- `level = .popUpMenu` — 窗口层级仅次于屏幕保护程序，能覆盖上下文菜单
- `animationBehavior = .none` — 禁用系统默认窗口动画
- `collectionBehavior = .canJoinAllSpaces` — 在 Space 切换动画期间也能正常显示
- `backgroundColor = .clear` — 透明背景，由内部的 `EffectView` 提供视觉效果
- `setAccessibilitySubrole(.unknown)` — 将自身从窗口列表中排除，避免递归捕获
- `setAccessibilityLabel(App.name)` — 为 VoiceOver 提供无障碍标签

核心方法：

| 方法 | 职责 |
|------|------|
| `updateContents(_:)` | 在 `caTransaction` 中调用 `TilesView.updateItemsAndLayout`，然后调整面板尺寸和位置；事务结束后调用 `TilesView.clearNeedsLayout()` 阻止后续 AppKit 布局传递 |
| `show()` | 设置外观（`updateAppearance`）、将 `alphaValue` 恢复为 1、调用 `makeKeyAndOrderFront`，并开启上下文菜单和光标事件监听 |
| `orderOut(_:)` | 支持两种消失方式：渐隐动画（`Preferences.fadeOutAnimation`）或立即隐藏（先 `alphaValue = 0` 再 `super.orderOut`，避免 WindowServer 滞后闪烁） |
| `repositionOrFreeze()` | 搜索模式下冻结面板顶部位置（`frozenTopCenter`），避免面板大小随搜索结果变化时跳动；非搜索模式下调用 `NSScreen.preferred.repositionPanel` 居中 |
| `updateAppearance()` | 根据 `Appearance.currentTheme` 设置阴影（`Appearance.enablePanelShadow`）和 `NSAppearance`（`.vibrantDark` 或 `.vibrantLight`） |

作为 `NSWindowDelegate`，它还处理：

- `windowDidResignKey(_:)` — 当切换器激活时，被其他窗口抢走焦点会重新夺回（通过 `DispatchQueue.main.async` 延迟调用 `makeKeyAndOrderFront`），同时恢复主菜单
- `windowDidBecomeKey(_:)` — 隐藏菜单栏防止 Command+Q 误操作，搜索模式下恢复编辑菜单，并延迟 0.25 秒刷新窗口列表

静态属性 `maxPossibleThumbnailSize` / `maxPossibleAppIconSize` 通过遍历所有屏幕取最大值（考虑 `backingScaleFactor`），用于缩略图/图标的预分配尺寸。静态方法 `maxThumbnailsWidth` 在 `.titles` 样式下考虑 `comfortableReadabilityWidth` 保证可读性。

### TilesView (`TilesView.swift`)

`TilesView` 不是视图子类，而是一个纯静态协调类，管理整个切换器的视图层次和布局逻辑。它持有：

- `scrollView: ScrollView` — 自定义 `NSScrollView`，内含 `TilesDocumentView` 作为 documentView
- `contentView: EffectView` — 面板背景效果视图（毛玻璃或 Liquid Glass）
- `currentEffectViewKind: EffectViewKind?` — 当前使用的效果视图类型
- `cachedEffectViews: [EffectViewKind: EffectView]` — 缓存已创建的效果视图，避免每次切换重建
- `recycledViews: [TileView]` — 视图回收池，预分配 20 个 `TileView`
- `rows: [[TileView]]` — 当前布局的行信息
- `lastRowSignature: [Int]` — 行签名缓存，仅在实际行结构变化时更新 tile 的行内位置属性
- `thumbnailUnderLayer: TileUnderLayer` — 全局高亮底层
- `thumbnailOverView: TileOverView` — 全局悬浮控件层
- `layoutCache: LayoutCache` — 缓存 `labelHeight`、`iconWidth`、`iconHeight`、`comfortableReadabilityWidth`
- `searchField: NSSearchField` — 搜索栏
- `noWindowLabel: NSTextField` — "No Window" 空状态提示

初始化流程（`initialize()`）：

1. 配置搜索栏（`configureSearchField`）和空状态标签（`configureNoWindowLabel`）
2. 创建背景视图（`updateBackgroundView`）
3. **预热 20 个 `TileView`** 到回收池（`(1...20).forEach { _ in TilesView.recycledViews.append(TileView()) }`）
4. 缓存尺寸信息（`updateCachedSizes`）

搜索功能支持三种模式（`SearchMode` 枚举）：`.off` / `.editing` / `.locked`

- `.editing`：搜索栏获得焦点，用户输入实时过滤窗口列表（`sendsSearchStringImmediately = true`）
- `.locked`：锁定搜索结果，方向键可继续导航，但不再响应文本输入
- 切换搜索模式的方法：`enableSearchEditing()` / `disableSearchMode()` / `lockSearchMode()` / `toggleSearchModeFromShortcut()`
- 搜索键事件处理（`handleSearchEditingKeyDown`）返回 `SearchKeyResult` 枚举：`.handled`（已处理）、`.passToField`（传递给搜索框）、`.passToShortcuts`（传递给快捷键系统）
- 搜索查询通过 `Windows.updateSearchQuery` 更新，触发 `App.refreshUi(true)` 刷新界面

背景视图管理：

- `updateBackgroundView()` — 完全重建背景视图层次，创建新的 `ScrollView`
- `swapBackgroundViewIfNeeded()` — 当效果视图类型变化时（如跨快捷键切换样式），无缝替换背景视图而不重建整个层次
- `cachedEffectView(for:)` — 从 `cachedEffectViews` 字典获取或创建效果视图

`reset()` 方法在外观设置变化时完全重建所有组件：更新屏幕偏好、外观参数、最大缩略图/图标尺寸、背景视图，替换所有 20 个回收视图，重建 `TileUnderLayer` 和 `TileOverView`，清空缓存。

`highlight(_:)` 方法同时更新 `TileView.drawHighlight()`（处理 appIcons 模式下的标签可见性）和 `TileUnderLayer.updateHighlight`（更新聚焦/悬浮高亮图层）。

`navigateUpOrDown(_:allowWrap:)` 实现行间导航：根据当前选中 tile 的水平中心位置，在目标行中找到最近的 tile，支持 RTL 布局方向。

`windowIdsInViewport()` 返回当前可见区域内的窗口 ID 集合，用于优化截图更新。

`clearNeedsLayout()` 在 `updateContents` 完成后，手动将 `contentView`、`scrollView` 及其子视图的 `needsLayout`、`needsDisplay`、`needsUpdateConstraints` 全部置为 `false`。

### ScrollView (`TilesView.swift`)

自定义 `NSScrollView` 子类，特性：

- `isCompatibleWithResponsiveScrolling` 覆写为 `true`，确保响应式滚动
- `isCurrentlyScrolling` 标记当前是否在滚动中，影响悬浮控件的显示（`TileOverView.updateHover` 会检查此标志）
- `scrollWheel(with:)` 覆写：将 Shift+滚轮的水平滚动转换为垂直滚动，避免与快捷键冲突
- 配置：`drawsBackground = false`、`verticalScrollElasticity = .none`（禁用弹性滚动）、`scrollerStyle = .overlay`、`scrollerKnobStyle = .light`

### TilesDocumentView (`TilesView.swift`)

继承自 `FlippedView`，作为 ScrollView 的 documentView，支持拖放操作：

- 注册 URL 类型拖放（`kUTTypeURL`）
- `dragOperation(_:)` — 将文件/URL 拖到 tile 上时打开对应应用
- 内置拖放定时器（`dragAndDropTimer`），2 秒延迟后触发 `mouseUpCallback`（即选中该窗口）
- `dragAndDropTimerResetDistance = 5` — 移动超过 5 点才重置定时器
- `wantsPeriodicDraggingUpdates` 覆写为 `false`，减少不必要的拖放更新

### TileView (`TileView.swift`)

`TileView` 继承自 `FlippedView`（翻转坐标系），是单个窗口/应用的可视化单元。它包含以下子视图和图层：

- `thumbnail: LightImageLayer` — 缩略图图层
- `appIcon: LightImageLayer` — 应用图标图层
- `appIconHighlight: CALayer` — 搜索匹配时的图标高亮（通过 `noAnimation { CALayer() }` 创建，禁用隐式动画）
- `label: TileTitleView` — 标题文本
- `statusIcons: StatusIconsView` — 状态图标（Space 编号、隐藏/全屏/最小化）
- `dockLabelIcon: TileFontIconView` — Dock 徽章（红点/数字）
- `windowlessAppIndicator: WindowlessAppIndicator` — 无窗口应用指示器

关键设计：

- 覆写 `_windowChangedKeyState()` 和 `_layoutSubtreeWithOldSize(_:)` 为空方法，**阻止 AppKit 对子视图树的递归遍历**
- 覆写 `canBecomeKeyView` / `acceptsFirstResponder`，仅当 `window` 属性非空（即视图在当前层级中）时返回 `true`，避免 AppKit key-view 循环遍历到已回收的视图
- `applyCurrentStyle()` 根据 `AppearanceStylePreference` 切换子视图的可见性和对齐方式（`thumbnail.isHidden`、`statusIcons.isHidden`、`label.alignment`）
- `updateRecycledCellWithNewContent(_:_:_:)` 是回收复用的核心方法，调用链：`applyCurrentStyle` → `updateValues` → `updateSizes` → `updatePositions` → `applySearchHighlight`
- `highlightFrame` 计算属性：appIcons 样式下仅覆盖图标区域（`appIcon.frame.height + edgeInsetsSize * 2`），其他样式覆盖整个 cell
- `drawHighlight()` 在 appIcons 样式下管理标签的显示/隐藏，确保标签 frame 在取消隐藏前已更新到正确位置，避免"标题从图标右侧滑到下方"的视觉故障

搜索高亮系统：

- `applySearchHighlight()` 处理标题文本中的搜索匹配高亮
- `appIconHighlight` 在搜索匹配到应用名时显示黄色高亮背景
- 标题高亮使用 `TileTitleView.searchHighlightBackgroundKey` 自定义属性，配合 `drawRoundedSearchHighlights()` 绘制圆角高亮背景
- `searchSpanRanges()` 根据偏好设置（仅应用名 / 应用名+窗口标题 / 仅窗口标题）计算匹配范围
- `truncatedDisplay(_:maxWidth:mode:attributes:)` 实现截断后的高亮范围映射，支持头部/中间/尾部三种截断模式，确保被截断部分的高亮信息在省略号上显示

阴影系统：

- `makeShadow(_:)` — 通用阴影（`blurRadius = 1`，`offset = .zero`）
- `makeAppIconShadow(_:)` — 图标阴影（`blurRadius = 2`，`offset = (0.1, 1)`，40% 透明度）
- `makeThumbnailShadow(_:)` — 缩略图阴影（`blurRadius = 3`，`offset = (0.8, 2.2)`，40% 透明度）

appIcons 样式下的标签布局（`updateAppIconsLabelFrame`）：

- 标签居中于图标下方，宽度为标题实际宽度与 `maxAllowedLabelWidth` 的较小值
- 行首 tile 的标签向左延伸，行末 tile 的标签向右延伸，中间 tile 的标签向两侧均分延伸
- `getMaxAllowedLabelWidth()` 计算基于 tile 在行中的位置和行内 tile 数量的最大允许标签宽度

## 布局系统

### 手动布局 + 视图回收池（20 个预热 TileView）

`TilesView.recycledViews` 在 `initialize()` 时预创建 20 个 `TileView` 实例。每次布局通过 `updateRecycledCellWithNewContent` 复用已有视图。超出 `Windows.list` 数量的视图调用 `thumbnail.releaseImage()` / `appIcon.releaseImage()` / `window_ = nil` 释放内存。当外观变化需要完全重建时，`TilesView.reset()` 替换所有 20 个视图。

### 两遍布局算法（dry run → real layout → row centering）

布局由 `TilesView.updateItemsAndLayout(_:)` 驱动，流程如下：

```
1. 如果尺寸偏好为 .auto → resolveAutoSize() 进行 dry run
   ├─ 从 .large → .medium → .small 逐级尝试
   ├─ dryRunLayoutTileViews() 计算布局高度（不修改视图 frame）
   └─ 选取第一个能让内容适配屏幕高度的尺寸
2. layoutTileViews() — 正式布局
   ├─ 逐个遍历 recycledViews，为每个视图调用 updateRecycledCellWithNewContent 设置内容和 frame
   ├─ 根据累计宽度判断是否换行（needNewLine）
   ├─ 收集行信息到 rows 数组和行签名（rowSignature）
   ├─ 将活跃 tile 作为 scrollView.documentView 的子视图
   ├─ 添加 thumbnailOverView 和 thumbnailUnderLayer 到 documentView
   └─ 返回 (maxX, maxY, labelHeight, rowSignature)
3. layoutParentViews() — 设置父容器尺寸
   ├─ 计算 thumbnailsWidth / thumbnailsHeight
   ├─ 设置 contentView、scrollView、searchField 的 frame
   ├─ appIcons 样式下调整 frameHeight（减去标题高度）
   ├─ RTL 模式下裁剪并偏移 documentView 子视图的 X 坐标
   └─ 空状态时显示 noWindowLabel
4. centerRows() — 行居中
   └─ 计算每行实际宽度与 maxX 的差值，将 tile 向中间偏移
5. 行签名变化时更新每个 tile 的 numberOfViewsInRow / isFirstInRow / isLastInRow / indexInRow
6. highlightStartView() — 高亮起始窗口或清除高亮
7. 可选：restoreScrollOrigin(_:) — 恢复之前的滚动位置
```

### 布局参数

布局方向由 `App.shared.userInterfaceLayoutDirection` 决定：

- **LTR**：起始 X = `interCellPadding`，向右累加
- **RTL**：起始 X = `widthMax - interCellPadding`，向左递减

三个关键的布局辅助函数：

| 函数 | 作用 |
|------|------|
| `projectedWidth(_:_:)` | 计算添加一个 tile 后的 X 坐标投影（LTR: 累加；RTL: 递减） |
| `needNewLine(_:_:)` | 判断是否需要换行（LTR: `projectedX > widthMax`; RTL: `projectedX < 0`） |
| `localizedCurrentX(_:_:)` | RTL 下将 X 转换为 frame.origin.x（`currentX - width`） |

### RTL 布局支持

整个布局系统完全支持从右到左（RTL）的文字方向（阿拉伯语、希伯来语）：

- 布局起始点从右侧开始（`startingX = widthMax - interCellPadding`）
- 换行判断方向反转（`projectedX < 0` 而非 `projectedX > widthMax`）
- 应用图标在标题头部右对齐（`updatePositions` 中根据布局方向计算 `appIcon.frame.origin.x`）
- 状态图标从右侧开始排列（`StatusIconsView.layoutIcons` 中 `isLTR` 分支）
- 行居中偏移方向反转（`centerRows` 中 `isLTR ? offset : -offset`）
- `layoutParentViews` 中对 RTL 模式裁剪 documentView 子视图的 X 坐标（`croppedWidth` 偏移）

## Tile 视图层次

### 层次结构（从底到顶）

```
TilesPanel (NSPanel)
└── contentView (EffectView: FrostedGlassEffectView / LiquidGlassEffectView)
    ├── scrollView (ScrollView)
    │   └── documentView (TilesDocumentView : FlippedView)
    │       ├── TileUnderLayer (CALayer) — z-index 0, 全局高亮
    │       │   ├── focusedLayer (CALayer)
    │       │   └── hoveredLayer (CALayer)
    │       ├── TileView[0..N] (FlippedView)
    │       │   ├── [CALayer] appIconHighlight
    │       │   ├── [LightImageLayer] appIcon
    │       │   ├── [LightImageLayer] thumbnail
    │       │   ├── StatusIconsView
    │       │   ├── TileTitleView (NSTextField)
    │       │   ├── WindowlessAppIndicator
    │       │   └── TileFontIconView (dockLabelIcon)
    │       └── TileOverView (FlippedView) — 全局悬浮控件
    │           ├── TrafficLightButton (quit)
    │           ├── TrafficLightButton (close)
    │           ├── TrafficLightButton (minimize)
    │           └── TrafficLightButton (maximize)
    ├── NSSearchField
    └── NSTextField (noWindowLabel)
```

### TileUnderLayer (`TileUnderLayer.swift`)

`TileUnderLayer` 是一个 `CALayer`，包含两个子图层：

- `focusedLayer` — 键盘选中窗口的高亮（使用 `highlightFocusedBackgroundColor` / `highlightFocusedBorderColor`）
- `hoveredLayer` — 鼠标悬浮窗口的高亮（使用 `highlightHoveredBackgroundColor` / `highlightHoveredBorderColor`）

两者均通过 `noAnimation { CALayer() }` 创建（自动设置 `delegate = NoAnimationDelegate.shared`）。初始化时两个图层都设为 `isHidden = true` 并通过 `addSublayer` 添加。

核心方法 `updateHighlight(focusedView:hoveredView:)` 更新两个高亮图层：

- `updateLayer(_:for:isFocused:)` 根据目标 `TileView` 的 `highlightFrame` 属性计算 frame
- 设置 `cornerRadius = Appearance.cellCornerRadius`
- 设置 `borderWidth = Appearance.highlightBorderWidth`
- 聚焦使用 `systemAccentColor`（20% 透明度背景 + 不透明边框），悬浮使用 10% 透明度背景 + 70% 透明度边框
- 当目标视图 frame 为 `.zero` 时隐藏对应高亮图层

### TileOverView (`TileOverView.swift`)

`TileOverView` 是一个覆盖在所有 tile 之上的透明视图，`layer!.masksToBounds = false` 允许按钮溢出显示。

**鼠标交互**：

- `hitTest(_:)` 仅对 `TrafficLightButton` 响应（返回按钮），其他区域返回 `nil`（点击穿透）
- `updateHover()` — 核心悬浮检测方法，将面板鼠标坐标转换为本地坐标，通过 `findTarget(_:)` 找到对应的 `TileView`
- `findTarget(_:)` 遍历 documentView 的所有 `TileView` 子视图，判断鼠标点是否落在 tile 的扩展 frame 内（RTL 下左边缘扩展 1 点）
- `findButton(_:)` — 查找鼠标位置下的 `TrafficLightButton`
- `updateButtonHover(_:)` — 管理按钮的悬浮状态（`isMouseOver` 属性）

**窗口控制按钮**：

4 个 `TrafficLightButton`（`quitButton` / `closeButton` / `minimizeButton` / `maximizeButton`），仅在缩略图模式下且未隐藏彩色圆圈时显示。

- `showWindowControls(for:)` — 根据窗口能力约束控制按钮可见性（不可退出的应用隐藏退出按钮，不可关闭的窗口隐藏关闭按钮等）
- 按钮定位基于缩略图的左上角（`thumbnailOrigin`），初始偏移 `(3, 2)`，按 `TrafficLightButton.size + TrafficLightButton.spacing` 间距排列
- 超出缩略图宽度时自动换行（xOffset 回到 3，yOffset 递增）
- `resetHoveredWindow()` — 清除悬浮状态并隐藏所有按钮

### LightImageLayer

`LightImageLayer` 是一个轻量级 `CALayer` 子类（定义在 `src/kit/LightImageLayer.swift`），用于显示缩略图和应用图标，避免 NSView 的开销：

- `contentsGravity = .resize` — 自动缩放
- `magnificationFilter = .trilinear` / `minificationFilter = .trilinear` — 三线性过滤保证缩放质量
- `delegate = NoAnimationDelegate.shared` — 全局禁用隐式动画
- `updateContents(_:_:)` 支持 `CGImage` 和 `CVPixelBuffer`（通过 IOSurface）两种内容源
- `applyShadow(_:)` 接受 `NSShadow` 参数为缩略图/图标添加阴影
- `centerInSuperlayer(x:)` — 水平居中于父图层
- `releaseImage()` 将 `contents` 置为 `nil` 释放图像内存

### TileTitleView (`TileTitleView.swift`)

`TileTitleView` 继承自 `NSTextField`，用于显示窗口/应用标题：

- 覆写 `intrinsicContentSize` 返回 `noIntrinsicMetric`，避免 AppKit 自动布局干涉
- 实现 `action(for:forKey:)` 作为 `CALayerDelegate` 方法，对 `position`/`bounds`/`frame`/`hidden`/`opacity`/`transform` 键返回 `NSNull()`，禁用所有隐式动画（注意：此方法不是 `override`，因为 Swift 不将 Obj-C 的 `CALayerDelegate` 方法暴露为可覆写；在 Swift 层面提供该方法会被运行时发现并调用）
- 支持三种截断模式（`byTruncatingHead` / `byTruncatingMiddle` / `byTruncatingTail`），通过 `getTruncationMode()` 根据 `Preferences.titleTruncation` 选择
- `setWidth(_:)` 带缓存（`currentWidth`），仅在宽度实际变化时更新
- `fixHeight()` 将高度设置为 `cell!.cellSize.height`

**搜索高亮**：

- 使用自定义 `drawRoundedSearchHighlights()` 方法（在 `draw(_:)` 中调用 `super.draw` 之前执行），绘制圆角矩形高亮背景
- 通过 `NSTextStorage` / `NSLayoutManager` / `NSTextContainer` 精确定位高亮范围的字形位置
- 自定义属性键 `searchHighlightBackgroundKey`（`NSAttributedString.Key`）标记需要高亮的文本范围
- `pixelAligned(_:)` 确保高亮矩形在 Retina 屏幕上像素对齐（考虑 `backingScaleFactor`）
- 高亮圆角半径为 `min(4, rect.height * 0.35)`

### StatusIconsView (`StatusIconsView.swift`)

`StatusIconsView` 继承自 `FlippedView`，直接在 `draw(_:)` 中绘制四个状态图标：

| 索引 | 常量 | 符号 | 含义 |
|------|------|------|------|
| 0 | `spaceIdx` | 圆圈数字 / 星号 | Space 编号 |
| 1 | `hiddenIdx` | `circledSlashSign` | 应用已隐藏 |
| 2 | `fullscreenIdx` | `circledPlusSign` | 窗口已全屏 |
| 3 | `minimizedIdx` | `circledMinusSign` | 窗口已最小化 |

- 使用 `TileFontIconView.symbolCache` 缓存 `NSAttributedString`，通过 `SF Pro Text` 字体渲染 SF Symbol 字符
- `cachedAttrString(for:)` 静态方法从缓存获取或创建属性字符串
- `iconCellSize` 在 `init` 时计算并缓存，供 `LayoutCache` 使用
- `totalWidth` 计算属性返回可见图标的总宽度（`visibleCount * TilesView.layoutCache.iconWidth`）
- `update(isHidden:isFullscreen:isMinimized:showSpace:)` 根据窗口状态和偏好设置更新图标可见性
- `setSpaceStar()` / `setSpaceNumber(_:)` — 设置 Space 图标为星号（在所有 Space 上）或数字
- `symbolForSpace(_:)` — 静态方法，通过 Unicode 标量运算将基础符号（`circledNumber0` 或 `circledNumber10`）偏移生成 1-20 的圆圈数字
- `layoutIcons(hWidth:hHeight:edgeInsets:)` — 设置 frame 尺寸和位置，RTL 下从左侧开始
- `ensureTooltipsInstalled()` — 使用 `addToolTip` 为可见图标安装工具提示，通过 `tooltipsDirty` 标志避免重复安装
- 覆写 `_windowChangedKeyState()` / `_layoutSubtreeWithOldSize(_:)` 为空方法

### TileFontIconView (`TileFontIconView.swift`)

`TileFontIconView` 继承自 `NSView`，支持两种渲染模式（`Rendering` 枚举）：

- **`.symbol`**：渲染单个 SF Symbol 字符（如状态图标），使用 `SF Pro Text` 字体
- **`.badge`**：渲染 Dock 徽章（红色圆角矩形 + 白色数字/文字），使用系统字体

关键特性：

- 静态缓存 `symbolCache: [SymbolCacheKey: NSAttributedString]` 避免重复创建属性字符串
- `SymbolCacheKey` 由 `symbol`、`size`、`colorKey`（颜色的 `description`）组成
- `warmCaches(symbols:extraStrings:size:color:)` 方法在初始化时预填充缓存
- 徽章模式支持最多 4 位数字（`BadgeSizing.maxDigits`），超过限制时降级为填充星号（`setFilledStar()`）
- `BadgeSizing` 结构体定义容器与图标的比例关系：`containerFromIconRatio = 0.43`、`textFromContainerRatio = 0.57`
- `badgeBaseSize(forIconSize:)` 静态方法计算徽章基础尺寸
- `paragraphStyle` 静态属性设置 `lineHeightMultiple = 0.85` 紧凑行高
- `isOpaque` 覆写为 `false`，支持透明背景

`Symbols` 枚举定义了项目中使用的所有 SF Symbol 字符的 rawValue 映射，包括切换器状态图标、偏好设置侧边栏图标、权限窗口图标等。

## 外观系统

### 3 种外观模式: thumbnails / titles / appIcons

通过 `AppearanceStylePreference` 枚举定义三种视觉模式，由 `Appearance.applyConcreteSize` 分发：

| 样式 | 方法 | 特点 |
|------|------|------|
| `.thumbnails` | `thumbnailsSize` | 显示缩略图（`hideThumbnails = false`）；`rowsCount` 根据屏幕方向（水平 3-5 行，垂直 6-8 行）和尺寸级别变化；支持多行网格布局 |
| `.titles` | `titlesSize` | 隐藏缩略图（`hideThumbnails = true`）；单行布局（`rowsCount = 1`）；`windowMinWidthInRow = 0.6`、`windowMaxWidthInRow = 0.9` 保证标题可读 |
| `.appIcons` | `appIconsSize` | 大图标模式；单行（`rowsCount = 1`）；`hideThumbnails = true`；`iconSize` 从 70（small）到 150（large）；`edgeInsetsSize = 5` 紧凑内边距 |

### Appearance 类的实现

`Appearance` 是一个纯静态类（`src/switcher/Appearance.swift`），管理所有视觉参数。分为三个维度：

**样式（Style）** — 由 `currentStyle` 计算（`Preferences.effectiveAppearanceStyle(SwitcherSession.activeShortcutIndex)`，支持按快捷键覆盖）

**尺寸（Size）** — 通过 `updateSize()` / `applyConcreteSize` 根据样式分发到对应方法：

每个尺寸级别（`.small` / `.medium` / `.large`）有不同的参数：

| 参数 | small | medium | large |
|------|-------|--------|-------|
| `iconSize` (thumbnails) | 16 | 26 | 28 |
| `iconSize` (appIcons) | 70 | 110 | 150 |
| `fontHeight` | 13 | 14 | 16 |
| `rowsCount` (水平) | 5 / 4 / 3 | — | — |

macOS 26 (Tahoe) 上使用更大的圆角和间距以适配 Liquid Glass 风格：`windowPadding = 28`、`windowCornerRadius = 43`、`cellCornerRadius = 18`。

自动尺寸（`.auto`）由 `resolveAutoSize` 在布局时动态计算：从 `.large` 开始尝试，如果内容高度超出屏幕则降级。

**主题（Theme）** — 由 `currentTheme` 计算（`Preferences.effectiveAppearanceTheme`，`.system` 时跟随 `NSAppearance.current.getThemeName()`）：

| 主题 | 字体颜色 | 图像阴影 | 毛玻璃材质 |
|------|----------|----------|------------|
| `.dark` | 白色 85% 透明度 | 灰色 80% 透明度 | `.dark` |
| `.light` | 黑色 80% 透明度 | 灰色 80% 透明度 | `.mediumLight` |

高亮颜色使用 `NSColor.systemAccentColor`：
- 聚焦：20% 透明度背景 + 不透明边框
- 悬浮：10% 透明度背景 + 70% 透明度边框
- 搜索匹配：50% 透明度黄色背景 + 深灰色（12% 白度）前景

`updateTheme()` 还处理 Liquid Glass 下的面板阴影：macOS 26+ appIcons 样式且私有 API 可用时 `enablePanelShadow = false`。

`updateFont()` 在 macOS 26+ 使用 `.semibold` 字重（appIcons 样式）或 `.medium` 字重（其他样式），旧版本使用常规字重。

`maxHeightOnScreen = 0.8`（常量），`interCellPadding = 1`，`intraCellPadding = 5`，`appIconLabelSpacing = 2`。

### 面板背景 (`TilesPanelBackgroundView.swift`)

背景效果视图有三种实现，通过 `EffectViewKind` 枚举选择：

| 类型 | 类 | 平台 |
|------|----|------|
| `.frosted` | `FrostedGlassEffectView` (NSVisualEffectView) | macOS < 26 |
| `.liquidGlassRegular` | `LiquidGlassEffectView` (NSGlassEffectView) | macOS 26+ |
| `.liquidGlassClear` | `LiquidGlassEffectView` (NSGlassEffectView, clear style) | macOS 26+ appIcons 样式 |

`EffectView` 协议要求实现 `updateAppearance()` 方法。

**LiquidGlassEffectView**（macOS 26+）：

- 继承自 `NSGlassEffectView`（Apple 私有框架）
- 使用私有 API `set_variant:`（通过 `NSSelectorFromString("set_variant:")` 动态查找）设置透明变体（`variant = 3`）
- `canUsePrivateLiquidGlassLook()` 静态方法在初始化时缓存检查 `set_variant:` 方法是否存在
- `clear` 样式：`style = .clear` + `safeSetVariant(3)`（透明玻璃效果）
- `regular` 样式：`style = .regular`
- `layer!.masksToBounds = true` 防止角落出现异常阴影

**FrostedGlassEffectView**（macOS < 26）：

- 继承自 `NSVisualEffectView`
- `blendingMode = .behindWindow`、`state = .active`
- 自定义 `updateRoundedCorners(_:)` 使用 `NSBezierPath` + 九宫格拉伸（`maskImage` + `capInsets` + `resizingMode = .stretch`）实现抗锯齿圆角，而非直接使用 `layer.cornerRadius`（后者会导致锯齿）

`requiredEffectViewKind()` 函数根据系统版本和样式偏好决定使用哪种效果。`makeEffectView(for:)` / `makeAppropriateEffectView()` 工厂函数创建对应的效果视图实例。

## 性能与优化

### 1. CALayer 隐式动画禁用 (nullActions)

多层防护确保无动画：

- **`NoAnimationDelegate`**（定义在 `HelperExtensions.swift`）：全局单例，`action(for:forKey:)` 始终返回 `NSNull()`，赋给 `LightImageLayer`、`TileUnderLayer` 及其子图层
- **`noAnimation` 闭包工厂**（定义在 `HelperExtensions.swift`）：`noAnimation { CALayer() }` 创建图层时自动设置 `delegate = NoAnimationDelegate.shared`
- **`caTransaction`**（定义在 `HelperExtensions.swift`）：在 `TilesPanel.updateContents` 中包裹整个布局过程，内部调用 `CATransaction.setDisableActions(true)`
- **`nullActions` 字典**：在 `TileView.setupSharedSubviews` 中，对 `label`、`statusIcons`、`windowlessAppIndicator`、`dockLabelIcon` 的 layer 设置 `actions` 字典，将 `position`/`bounds`/`frame`/`hidden`/`opacity`/`transform` 映射为 `NSNull()`
- **`TileTitleView.action(for:forKey:)`**：作为 `CALayerDelegate` 方法，拦截 AppKit 布局传递触发的动画查找

这种多层防护是因为 `NSWindow.setContentSize` 会触发 AppKit 的后续布局传递，该传递发生在 `caTransaction` 之外——此时 layer 的默认 action 字典会触发 position/bounds 动画，产生"标题从图标右侧滑到下方"等视觉故障。

### 2. 视图回收

- 初始化时预创建 20 个 `TileView` 实例，存储在 `TilesView.recycledViews` 数组中
- 每次布局只更新内容和 frame，不创建/销毁视图
- 超出窗口列表长度的视图调用 `releaseImage()` 释放 `CALayer.contents` 和窗口引用
- 当外观变化需要完全重建时，`TilesView.reset()` 替换所有 20 个视图

### 3. 禁用 AppKit 递归

在 `TileView`、`TileOverView`、`TilesDocumentView`、`StatusIconsView` 上覆写两个 Objective-C 私有方法为空实现：

```swift
@objc func _windowChangedKeyState() {}
@objc func _layoutSubtreeWithOldSize(_ oldSize: NSSize) {}
```

`_windowChangedKeyState` 会递归标记所有 `NSControl` 子视图为 `needsDisplay`，`_layoutSubtreeWithOldSize` 会触发 Auto Layout 传递。由于切换器使用手动布局，这些递归是纯开销，空覆写将其截断在 tile 边界。

### 4. `assignIfDifferent` 条件赋值

所有 frame 更新通过 `assignIfDifferent(&view.frame.origin, newValue)` 函数（定义在 `HelperExtensions.swift`），仅在值实际改变时才赋值，减少不必要的标记和重绘。

### 5. 缓存机制

- `LayoutCache`：缓存 `labelHeight`、`iconWidth`、`iconHeight`、`comfortableReadabilityWidth`
- `TileFontIconView.symbolCache`：缓存 `NSAttributedString` 避免重复创建，通过 `SymbolCacheKey`（symbol + size + colorKey）索引
- `cachedEffectViews`：缓存背景效果视图（`[EffectViewKind: EffectView]`）
- `cachedSearchBarHeight`：缓存搜索栏高度
- `lastRowSignature`：缓存行签名（`[Int]`），仅在行结构变化时更新 `numberOfViewsInRow` / `isFirstInRow` / `isLastInRow` / `indexInRow` 属性
- `TileTitleView.currentWidth`：缓存标签宽度，避免重复设置
- `StatusIconsView.tooltipsDirty`：避免重复安装工具提示

### 6. 面板出现/消失动画

- **出现**：直接设置 `alphaValue = 1` + `makeKeyAndOrderFront`，无动画，保证即时响应
- **消失**：两种模式。`Preferences.fadeOutAnimation = true` 时使用 `NSAnimationContext.runAnimationGroup` 渐隐 `alphaValue` 到 0，完成后调用 `super.orderOut`；否则立即设置 `alphaValue = 0` 再调用 `orderOut`（先隐藏可避免 WindowServer 滞后导致的闪烁）

### 7. 清除布局标记

`TilesView.clearNeedsLayout()` 在 `updateContents` 完成后，手动将 `contentView`、`scrollView`、`scrollView.contentView`、`scrollView.documentView`、`noWindowLabel` 的 `needsLayout`、`needsDisplay`、`needsUpdateConstraints` 全部置为 `false`，阻止 AppKit 在后续 run loop 中发起不必要的布局传递。

## 文件清单

| 文件路径 | 核心类/结构 | 职责 |
|----------|-------------|------|
| `src/switcher/main-window/TilesPanel.swift` | `TilesPanel` | 顶层 NSPanel，窗口生命周期管理、面板定位和焦点管理 |
| `src/switcher/main-window/TilesView.swift` | `TilesView`, `ScrollView`, `TilesDocumentView`, `FlippedView`, `Direction`, `SearchMode`, `SearchKeyResult` | 布局协调器、滚动容器、拖放处理、搜索功能、方向枚举 |
| `src/switcher/main-window/TileView.swift` | `TileView` | 单个窗口/应用的可视化单元，包含缩略图/图标/标题/状态图标/搜索高亮 |
| `src/switcher/main-window/TileOverView.swift` | `TileOverView` | 全局悬浮层（鼠标交互 + 窗口控制按钮：退出/关闭/最小化/全屏） |
| `src/switcher/main-window/TileUnderLayer.swift` | `TileUnderLayer` | 全局高亮底层（聚焦高亮 + 悬浮高亮） |
| `src/switcher/main-window/TileTitleView.swift` | `TileTitleView` | 标题文本渲染（截断模式 + 搜索匹配圆角高亮 + 像素对齐） |
| `src/switcher/main-window/TileFontIconView.swift` | `TileFontIconView`, `Symbols` | SF Symbol 字体图标渲染（symbol 模式）和 Dock 徽章渲染（badge 模式），含静态符号缓存 |
| `src/switcher/main-window/StatusIconsView.swift` | `StatusIconsView` | 状态图标（Space 编号/星号、隐藏、全屏、最小化），直接 draw 渲染 |
| `src/switcher/main-window/TilesPanelBackgroundView.swift` | `FrostedGlassEffectView`, `LiquidGlassEffectView`, `EffectView`, `EffectViewKind` | 面板背景效果（毛玻璃 / Liquid Glass），含工厂函数和动态 API 检测 |
| `src/switcher/Appearance.swift` | `Appearance` | 外观尺寸（small/medium/large/auto）和主题（dark/light）参数管理 |
