# Kit UI 组件

## 模块职责

`src/kit/` 是 AltTab 的轻量级 UI 组件库，为整个应用提供可复用的自定义控件和视图。该模块完全基于纯 AppKit 和 CALayer 构建，不使用 Interface Builder（`.xib`/`.storyboard`）和 SwiftUI。所有组件采用程序化初始化、Auto Layout 手动约束以及直接 CALayer 操作。

该模块的设计动机有二：
1. **标准 AppKit 控件的局限性** — 原生 `NSButton`、`NSTextField` 等在特定场景下无法满足需求（例如自定义绘制红绿灯按钮、高性能缩略图渲染、快捷键冲突检测等）。
2. **统一风格与行为** — 通过封装统一的初始化模式（`translatesAutoresizingMaskIntoConstraints = false`、`fit()` 自动尺寸适配）和交互模式（基于闭包的 `onAction`），减少样板代码并保证一致性。

## 核心组件

### Button（`Button.swift`）

`Button` 是对 `NSButton` 的极简封装，提供基于闭包的回调机制。

```swift
class Button: NSButton
```

**设计要点：**
- 使用 `convenience init(_ title: String, _ action: ActionClosure?)` 将标题和点击回调一体化传入
- `onAction` 属性（定义在 `NSControl` 扩展中）通过 Objective-C 关联对象（`objc_setAssociatedObject`）和 `SelectorWrapper` 桥接，将 target-action 模式转换为现代闭包风格
- 调用 `fit()` 方法（`NSView` 扩展）将视图尺寸约束到其 `fittingSize`，实现自动适配
- 默认禁用 `translatesAutoresizingMaskIntoConstraints`，与项目手动布局策略一致

### Switch（`Switch.swift`）

`Switch` 是对 `NSSwitch`（macOS 10.15+）的兼容性封装，在旧系统上回退到 `NSButton` 的 `.switch` 类型。

```swift
class Switch: NSButton
```

**实现细节：**
- **双模式架构**：在 macOS 10.15+ 上，创建原生 `NSSwitch` 作为子视图嵌入；在更早版本上回退到 `setButtonType(.switch)`
- `switchButton` 属性持有对内嵌 `NSSwitch` 的引用，所有状态变更同步到子视图
- **`setSilently(_:)` 方法**：绕过 `state` 的 `didSet` 观察器（直接调用 `super.state`），用于重新同步 UI 状态到全局值时避免触发 `UserDefaults` 写入。这在"全局设置覆盖单项设置"的场景中至关重要
- `switchToggled(_:)` 作为 `NSSwitch` 的 target-action 中间件，将子视图状态变化传播到父 `Switch` 的 `state` 属性

**回退机制的条件编译：** 通过 `if #available(macOS 10.15, *)` 检查，而非编译期条件，保证同一二进制在不同系统版本上都能正确运行。

### ImageTextButtonView（`ImageTextButtonView.swift`）

图文组合的选项卡按钮，用于设置界面中的视觉风格选择。由图片按钮和文字标签纵向排列组成的 `NSStackView`。

```swift
class ImageTextButtonView: NSStackView
```

**内部类结构：**
- `ImageButton`：`NSButton` 子类，自定义焦点环绘制（圆角矩形路径），重写 `mouseDown` 实现按压状态反馈（通过 `isPressed` 标志触发 `updateStyle()`）
- `ClickableLabel`：`NSTextField` 子类，通过 `hitTest` 返回 `nil` 使点击穿透到父视图；同时设置 `mouseDownCanMoveWindow = false` 防止误触发窗口拖拽

**状态驱动的样式系统（`updateStyle()` 方法）：**
- 选中时边框使用 `systemAccentColor`（活动窗口）或 `unemphasizedSelectedContentBackgroundColor`（非活动窗口）
- 未选中时使用 `NSColor.lightGray.withAlphaComponent(0.3)` 半透明边框
- 按压状态通过 `alphaValue = 0.7` 实现视觉反馈
- 选中标签文字加粗（`NSFont.boldSystemFont`），未选中使用常规字重
- 监听 `NSWindow.didBecomeKeyNotification` / `didResignKeyNotification` 实时更新样式

**图片加载策略：** 目前硬编码加载 `_light` 后缀的图片资源（如 `name_light`），注释表明主题系统尚未实现。

## 图层与视图

### LightImageLayer（`LightImageLayer.swift`）

轻量级 `CALayer` 子类，作为 `NSImageView` 的高性能替代品，用于主窗口中的缩略图渲染。

```swift
class LightImageLayer: CALayer
```

**与 NSImageView 的性能差异：**

| 特性 | NSImageView | LightImageLayer |
|------|-------------|-----------------|
| 继承层次 | NSView > NSResponder > NSObject | CALayer > NSObject |
| 布局开销 | 完整的 Auto Layout 递归、布局标记传播 | 直接设置 `frame.size` |
| 响应链 | 参与完整响应链（mouseDown、keyDown 等） | 无响应链开销 |
| 拖拽支持 | 内置拖放支持 | 无拖放开销 |
| 隐式动画 | 默认启用（位置、边界变化有动画） | 通过 `NoAnimationDelegate` 显式禁用 |
| 内存占用 | 完整的 NSView 栈 | 仅一个 CALayer |

**关键配置：**
- `contentsGravity = .resize`：图片自动缩放以填满图层
- `magnificationFilter` / `minificationFilter = .trilinear`：三线性过滤，在缩略图缩放时提供更平滑的效果
- `shouldRasterize = false`：禁用光栅化缓存，因为缩略图内容频繁更新
- `delegate = NoAnimationDelegate.shared`：通过 `CALayerDelegate` 的 `action(for:forKey:)` 返回 `NSNull()`，阻止所有隐式动画（如位置、尺寸变化时的过渡效果），确保帧间更新无延迟

**`updateContents(_:size:)` 方法** 支持两种内容源：
1. `.cgImage(CGImage?)` — 来自截图 API 的静态图像
2. `.pixelBuffer(CVPixelBuffer?)` — 来自屏幕录制流的像素缓冲区，通过 `IOSurface` 桥接到 CALayer contents

**`releaseImage()` 方法** 将 `contents` 置为 `nil`，在缩略图不可见时释放底层图像内存。

### LightImageView（`LightImageView.swift`）

`LightImageLayer` 的 `NSView` 包装器，用于需要 `NSView` 接口的场景（如作为 `contentView`、Auto Layout 锚点、`NSStackView` 子视图等）。

```swift
class LightImageView: NSView
```

**实现方式：**
- 在 `init` 中设置 `wantsLayer = true`，将 `LightImageLayer` 作为子图层添加到视图的 `layer` 上
- `layerContentsRedrawPolicy = .never`：告知 AppKit 不需要为图层内容重绘调用 `draw(_:)`，因为内容通过直接操作 `CALayer.contents` 更新
- `layout()` 中同步 `imageLayer.frame = bounds`，确保图层与视图大小一致

**`CALayerContents` 枚举** 统一了两种图像源的接口，提供 `size()` 方法获取图像尺寸，用于缩略图布局计算。

### WindowlessAppIndicator（`WindowlessAppIndicator.swift`）

无窗口应用的视觉指示器，绘制为半透明圆角矩形小标记。

```swift
class WindowlessAppIndicator: NSView
```

**设计目的：** 在应用切换器中，某些应用（如后台服务）没有可见窗口。该指示器以小色块形式显示在应用图标旁，提示用户此应用当前无窗口。

**`AppearanceParameter` 结构体** 根据当前显示风格和尺寸级别提供不同的参数：
- `thumbnails` / `appIcons` 风格：较大尺寸（宽度 10-12pt，高度 5pt，圆角 2pt）
- 其他风格：较小尺寸（宽度 6-8pt，高度 3pt，圆角 1pt）
- `large` 和普通尺寸进一步区分

**绘制方式：** 在 `draw(_:)` 中使用 `Appearance.fontColor.withAlphaComponent(0.5)` 填充圆角矩形路径，颜色随主题自动适配。

## 控件

### CustomRecorderControl（`CustomRecorderControl.swift`）

基于第三方库 `ShortcutRecorder` 的 `RecorderControl` 的自定义快捷键录制控件，是 AltTab 设置界面中最复杂的交互组件。

```swift
class CustomRecorderControl: RecorderControl
```

**核心功能：**
- 允许用户录制键盘快捷键（支持纯修饰键组合）
- 实时检测快捷键冲突（与已有快捷键、macOS 保留快捷键）
- 提供冲突解决对话框（取消或覆盖已有绑定）

**快捷键录制流程：**
1. 用户点击录制控件进入录制模式
2. 按下键组合触发 `recorderControl(_:canRecord:)` 委托方法
3. 调用 `CustomRecorderControlTestable.isShortcutAcceptable()` 进行合法性检验
4. 根据返回的 `ShortcutAcceptance` 枚举值执行不同操作：
   - `.accepted`：直接保存
   - `.modifiersOnlyButContainsKeycode`：拒绝（holdShortcut 必须为纯修饰键）
   - `.conflictWithExistingShortcut`：弹出冲突对话框
   - `.reservedByMacos`：弹出 macOS 保留快捷键对话框

**冲突检测逻辑（`CustomRecorderControlTestable`）：**

AltTab 的快捷键系统有独特的"组合"语义：`holdShortcut`（修饰键）+ 本地快捷键 组合成最终的全局快捷键。例如 `holdShortcut = ⌥` + `nextWindowShortcut = Tab` = `⌥Tab`。

- `newCombinationsFromCandidate`：计算候选快捷键与所有现有快捷键的笛卡尔积组合
- `oldCombinationsExcludingTargetOfCandidate`：获取所有现有组合（排除被替换的）
- `isAlreadyUsedByAnotherShortcut`：逐一比较新旧组合是否冲突
- `isReservedByMacos`：检查是否与 macOS 强制保留快捷键冲突（`⌘⌥⎋`、`⌘⌥⇧⎋`、`⌘⌥⇧⌃⎋`）
- `combinedModifiersMatch`：用于比较两个修饰键集合在加入 holdShortcut 后是否等价

**`ShortcutAcceptance` 枚举** 使用关联值传递冲突信息（如 `.conflictWithExistingShortcut(shortcutAlreadyAssigned: String)`），为冲突对话框提供具体信息。

**修饰键限制：** `allowedModifiers` 限定为 `⌘⌃⌥⇧` 四个标准修饰键，通过 `restrictModifiers(_:)` 方法可进一步排除特定修饰键。

### PopupButtonLikeSystemSettings（`PopupButtonLikeSystemSettings.swift`）

模仿 macOS 系统设置中弹出按钮外观和行为的 `NSPopUpButton` 子类。

```swift
class PopupButtonLikeSystemSettings: NSPopUpButton
```

**核心问题与解决方案：** 标准 `NSPopUpButton` 的 `intrinsicContentSize` 取所有菜单项中最长的标题来计算宽度。当某个菜单项同时有标题和图片时，系统不会将图片计入尺寸。该类通过创建临时的 `NSPopUpButton` 来精确测量当前选中项（含图片）的实际所需宽度。

**`intrinsicContentSize` 覆写逻辑：**
1. 获取当前选中项
2. 创建一个临时 `NSPopUpButton`，仅添加选中项的标题和图片
3. 复制当前按钮的 `bezelStyle`、`arrowPosition`、`imagePosition` 等样式属性
4. 调用临时按钮的 `sizeToFit()` 后取其 `intrinsicContentSize`

这确保了按钮宽度随选中项内容动态变化，与系统设置中的行为一致。

### TrafficLightButton（`TrafficLightButton.swift`）

完全自定义绘制的 macOS 窗口控制按钮（红绿灯按钮），模拟原生关闭、最小化、全屏、退出按钮。

```swift
class TrafficLightButton: NSButton
```

**`TrafficLightButtonType` 枚举** 定义四种按钮类型：`.quit`、`.close`、`.miniaturize`、`.fullscreen`。

**为什么不用系统按钮：** 源码注释记录了尝试使用 `NSWindow.standardWindowButton` 的实验。放弃的原因包括：
- zoom 按钮有不可移除的弹出提示
- 系统按钮外观依赖于父窗口状态（如需要设置 `collectionBehavior = .fullScreenPrimary` 才显示全屏按钮）
- 无法完全控制悬停行为和绘制细节

**自定义绘制架构（`draw(_:)` 方法）：**

1. **`colors()`** — 根据 `NSColor.currentControlTint` 和按钮类型返回三层颜色：
   - `diskBackgroundColor`（渐变 `NSGradient`）
   - `diskStrokeColor`（描边色）
   - `symbolColor`（符号色）
   - 支持 Graphite 主题（灰色单色调）和标准彩色主题
   - 每种类型有独特的渐变色：全屏=绿色、最小化=黄色、关闭=红色、退出=紫色

2. **`drawDisk()`** — 绘制圆形底盘（0.5pt 偏移 + 1px 缩小的椭圆），填充渐变色 + 0.5pt 描边

3. **`drawSymbol()`** — 根据按钮类型绘制不同的 `NSBezierPath` 符号：
   - `.fullscreen`：两个三角形（全屏时面对面，非全屏时背对背）
   - `.miniaturize`：水平线（关闭抗锯齿以确保 1px 精确度）
   - `.close`：X 形交叉线
   - `.quit`：带竖线的圆弧（类似吃豆人 ghost）

4. **`drawDimming()`** — 根据交互状态叠加半透明黑色：
   - `isHighlighted`（按下）：50% 不透明度
   - `isMouseOver`（悬停）：25% 不透明度

**强制外观：** `appearance = NSAppearance(named: .aqua)` 确保按钮始终使用亮色系渲染，不受系统暗色模式影响。这与 macOS 原生红绿灯按钮行为一致。

**点击处理：** `onClick()` 根据按钮类型调用 `Window` 的不同方法（`toggleFullscreen()`、`minDemin()`、`close()`）或 `Application.quit()`。

### Tooltips（`Tooltips.swift`）

工具提示管理工具，解决主窗口隐藏时残留 tooltip 的清理问题。

```swift
enum Tooltips
```

**`hideAll()` 方法** 使用私有 API `NSSelectorFromString("abortAllToolTips")` 强制关闭所有显示中的 tooltip。这是一个已知的 AppKit 问题 —— 当窗口通过 `orderOut` 隐藏时，某些 tooltip 不会被自动关闭。通过 `NSApp.responds(to:)` 进行安全检查，避免未来系统版本移除该 API 时崩溃。

## 布局容器

### GridView（`GridView.swift`）

基于 `NSGridView` 的设置页面网格布局容器。

```swift
class GridView: NSGridView
```

**统一间距常量：**
- `padding = 20pt`：外边距
- `interPadding = 10pt`：行列间距

`convenience init` 接受二维 `NSView` 数组，自动设置 `yPlacement = .fill`（垂直填充）和 `rowAlignment = .firstBaseline`（基线对齐）。首尾行列和列的 padding 确保内容不紧贴边缘。

### StackView（`StackView.swift`）

基于 `NSStackView` 的简化封装，提供统一的初始化接口。

```swift
class StackView: NSStackView
```

**特殊处理：** 包含 `RecorderControl` 的水平 StackView 会有额外的 `fittingSize.height`（约 7pt），通过硬编码减去该值来修正布局。这属于对 ShortcutRecorder 库布局行为的 workaround。

**边距支持：** 通过 `edgeInsets` 参数提供四个方向的边距控制。

### TableView（`TableView.swift`）

滚动视图相关组件，处理嵌套 `NSScrollView` 之间的滚动事件转发。

**`ForwardingVerticalScrollView`：** 当内部滚动视图到达边界（顶部或底部）时，将垂直滚动事件转发给父级 `NSScrollView`。这在设置页面中有多个可滚动区域嵌套的场景中至关重要。

**实现原理：**
1. `wantsForwardedScrollEvents(for:)` 返回 `true`（仅垂直轴）
2. `scrollWheel(with:)` 中通过 `constrainBoundsRect` 计算有效滚动范围
3. 判断是否已在边界，若是则调用 `parentScrollView()?.scrollWheel(with:)`

**`ForwardingVerticalDocumentView`：** 基于 `FlippedView`（翻转坐标系，原点在左上角）的文档视图，配合滚动视图使用。

### SheetWindow（`SheetWindow.swift`）

模态 Sheet 窗口基类，用于设置页面中的弹出配置面板。

```swift
class SheetWindow: NSWindow
```

**结构：**
- 固定宽度 `500pt`
- `WindowContentView`（`NSStackView` 子类）：纵向排列内容视图、分隔线、完成按钮
- 分隔线使用 `NSColor.tableSeparatorColor` 绘制
- 完成按钮使用 `NSColor.controlAccentColor`（macOS 10.14+），回车键作为等效键

**可扩展设计：** `makeContentView()` 返回空 `NSView`，子类覆写该方法提供实际内容。`cancel(_:)` 方法通过 `sheetParent!.endSheet(self)` 关闭 Sheet，同时支持 Escape 键关闭。

## 文本组件

所有文本组件位于 `src/kit/text/` 子目录，针对不同使用场景对 `NSTextField` 进行特化。

### TextField（`TextField.swift`）

基础文本标签，用于设置页面的标签和只读文字显示。

```swift
class TextField: NSTextField
```

**RTL 处理：** `forceLeftToRight(_:)` 静态方法强制所有 `NSAttributedString` 使用从左到右书写方向。这对于显示快捷键符号（`⌘⌥⇧` 等技术符号）至关重要——这些符号在 RTL 语言环境下可能被错误地反转显示。实现方式是创建 `NSMutableParagraphStyle` 并设置 `baseWritingDirection = .leftToRight`。

**去除默认内边距：** `alignmentRectInsets` 返回零边距，消除 `NSTextField` 默认的 2px 左右内边距，确保精确布局。

### BoldLabel（`BoldLabel.swift`）

粗体标签，用于设置分组标题等强调文字。

```swift
class BoldLabel: NSTextField
```

使用 `NSFont.boldSystemFont(ofSize: NSFont.systemFontSize)` 将默认系统字号加粗。

### TitleLabel（`TitleLabel.swift`）

大标题标签，使用 `wrappingLabelWithString` 构造器支持多行换行。

```swift
class TitleLabel: NSTextField
```

字号为 `NSFont.labelFontSize * 2`（约 26pt），是标准标签的两倍大小，用于 Sheet 窗口顶部标题。

### DynamicColorTextField（`DynamicColorTextField.swift`）

支持动态颜色更新的文本标签，颜色随外部状态实时变化。

```swift
class DynamicColorTextField: NSTextField
```

**设计动机：** 某些标签的颜色需要跟踪窗口焦点状态、选中状态等频繁变化的外部条件。传统做法需要在每个状态变更点手动更新颜色，容易遗漏。该组件通过闭包延迟求值的方式，在每次绘制前自动计算正确颜色。

**`colorProvider` 闭包：** 在 `viewWillDraw()` 中调用，这是 AppKit 绘制周期中最后的修改机会。闭包通常返回动态系统颜色（如 `.controlTextColor`），由 AppKit 自动适配 Light/Dark Mode。

### HyperlinkLabel（`HyperlinkLabel.swift`）

可点击的超链接标签，支持 URL 跳转和自定义点击回调两种模式。

```swift
class HyperlinkLabel: NSTextField
```

**两种初始化模式：**
1. `init(_ string:, _ urlString:)` — 点击时通过 `NSWorkspace.shared.open(url)` 打开链接
2. `init(_ string:, onClick:)` — 点击时执行自定义闭包

**视觉样式：** 使用 `NSColor.linkColor` 和 `NSFont.labelFont` 渲染。`resetCursorRects()` 中添加手指光标（`.pointingHand`）。设置 `isSelectable = false` 确保标签不可选，仅响应点击。

### TextArea（`TextArea.swift`）

多行可编辑文本区域，支持行数限制、字符数上限和自定义内边距。

```swift
class TextArea: NSTextField
```

**布局计算：** 宽度基于 `(font.xHeight * nCharactersWide + padding * 2)`，高度基于 `(systemFontSize * interLineFactor * nLinesHigh + padding * 2)`，`interLineFactor = 1.6` 提供舒适的行间距。

**`TextFieldCell`（`NSTextFieldCell` 子类）：** 唯一目的是通过覆写 `drawingRect(forBounds:)` 添加 5pt 四周内边距。同时修正 AppKit 默认对齐方向 bug（文档声称默认 `.natural`，实际为 `.left`）。

**回车键处理策略：**
- 单行模式：回车结束编辑，触发窗口默认按钮（标准 macOS 提交行为）
- 多行模式：回车插入换行符（`insertNewlineIgnoringFieldEditor`），匹配 Mail/Notes 等原生应用的行为

**字符数限制（UTF-16 精确计数）：** `maxLength` 以 UTF-16 code unit 为单位（与后端 JavaScript 的 `String.length` 一致），按 grapheme cluster 逐字符截断，避免在 Unicode 代理对或组合序列中间切割。这在处理 emoji 等复杂字符时尤为重要。

## 设计考量

### 为什么自定义组件而非标准 AppKit 组件

1. **性能优先**：`LightImageLayer` 避免了 `NSView` 栈的完整开销。AltTab 的主窗口可能同时显示数十个缩略图，每个缩略图每帧都需要更新。使用 `CALayer` 直接操作比 `NSImageView` 快数倍。

2. **精确控制**：`TrafficLightButton` 需要在非标准窗口中绘制完全一致的 macOS 红绿灯按钮，系统 API 无法在浮动面板中正确渲染这些按钮。

3. **一致接口**：所有组件使用统一的初始化模式（`translatesAutoresizingMaskIntoConstraints = false` + `fit()`）、统一的回调机制（`onAction` 闭包），减少样板代码。

4. **跨版本兼容**：`Switch` 同时支持 macOS 10.15+ 的原生 `NSSwitch` 和旧版本的回退方案，确保应用在所有支持的系统版本上表现一致。

### RTL 和本地化考虑

- `TextField.forceLeftToRight()` 确保快捷键符号在所有语言环境下正确显示
- `TextArea.TextFieldCell` 显式设置 `alignment = .natural` 修正 AppKit 默认值 bug
- `TextArea` 的 UTF-16 字符计数与后端 JavaScript 保持一致

### 动画与渲染优化

- `NoAnimationDelegate` 单例通过 `CALayerDelegate` 返回 `NSNull()` 禁用所有隐式动画，在 60fps 缩略图更新场景中消除不必要的过渡效果
- `LightImageLayer` 使用 `.trilinear` 过滤在缩放时保持平滑，但禁用光栅化避免频繁更新时的缓存失效开销
- `ForwardingVerticalScrollView` 实现滚动事件穿透，解决嵌套滚动区域的用户体验问题

### 私有 API 的谨慎使用

- `Tooltips.hideAll()` 使用 `abortAllToolTips` 私有 API，但通过 `responds(to:)` 安全检查确保前向兼容性

## 文件清单

| 文件路径 | 类名 | 行数 | 职责 |
|---------|------|------|------|
| `src/kit/Button.swift` | `Button` | 10 | 基于闭包的按钮封装 |
| `src/kit/Switch.swift` | `Switch` | 87 | 兼容性开关控件 |
| `src/kit/GridView.swift` | `GridView` | 20 | 网格布局容器 |
| `src/kit/CustomRecorderControl.swift` | `CustomRecorderControl` | 124 | 快捷键录制控件 |
| `src/kit/CustomRecorderControlTestable.swift` | `CustomRecorderControlTestable` | 185 | 录制控件逻辑与冲突检测 |
| `src/kit/ImageTextButtonView.swift` | `ImageTextButtonView` | 190 | 图文选项卡按钮 |
| `src/kit/LightImageLayer.swift` | `LightImageLayer` | 41 | 高性能图像图层 |
| `src/kit/LightImageView.swift` | `LightImageView` | 49 | 图层视图包装器 |
| `src/kit/PopupButtonLikeSystemSettings.swift` | `PopupButtonLikeSystemSettings` | 35 | 动态宽度弹出按钮 |
| `src/kit/SheetWindow.swift` | `SheetWindow` | 76 | 模态 Sheet 窗口基类 |
| `src/kit/StackView.swift` | `StackView` | 19 | 堆栈视图封装 |
| `src/kit/TableView.swift` | `ForwardingVerticalScrollView`, `ForwardingVerticalDocumentView` | 50 | 滚动事件转发 |
| `src/kit/Tooltips.swift` | `Tooltips` | 12 | 工具提示清理 |
| `src/kit/TrafficLightButton.swift` | `TrafficLightButton`, `TrafficLightButtonType` | 220 | 自定义红绿灯按钮 |
| `src/kit/WindowlessAppIndicator.swift` | `WindowlessAppIndicator` | 55 | 无窗口应用指示器 |
| `src/kit/text/BoldLabel.swift` | `BoldLabel` | 9 | 粗体标签 |
| `src/kit/text/DynamicColorTextField.swift` | `DynamicColorTextField` | 18 | 动态颜色文本框 |
| `src/kit/text/HyperlinkLabel.swift` | `HyperlinkLabel` | 38 | 超链接标签 |
| `src/kit/text/TextArea.swift` | `TextArea`, `TextFieldCell` | 79 | 多行文本区域 |
| `src/kit/text/TextField.swift` | `TextField` | 28 | 基础文本标签 |
| `src/kit/text/TitleLabel.swift` | `TitleLabel` | 9 | 大标题标签 |
