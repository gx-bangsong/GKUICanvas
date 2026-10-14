<!--
     SPDX-FileCopyrightText: The LineageOS Project
     SPDX-License-Identifier: Apache-2.0
-->

# GKUI 移植功能在 Canvas 中的 UI 落位方案

> Abstract (EN): UI placement plan for porting GKUI Photos editing features into LineageOS Canvas. It maps every existing Screen, bottom-bar container and reusable Composable (with file paths and line numbers), measures bottom-bar capacity at 360dp and 320dp from the real layout code and Material 3 tokens, and places the ported features (crop ratios and guides, MediaStore write request, privacy watermark, temperature, invert filter, EXIF on overwrite) by insertion only, without refactoring existing screens. Export options were removed after review as toolbox-domain; see the Errata in the sibling port plan. Documentation only; no code changes.
>
> 文档性质：UI 落位方案，作为后续编码任务的输入。**本文档不包含任何代码改动**。
>
> 姊妹文档：[`GKUI_EDIT_PORT_PLAN.md`](GKUI_EDIT_PORT_PLAN.md)（下称“移植计划”）。功能规格（Action 模型、算法、SPDX、任务编号 P0-1…P2-2）以移植计划为准；本文只回答“放在哪、怎么交互、复用什么”。凡与移植计划“UI 重做说明”冲突之处，以本文 §4 为准；两份文档都以移植计划顶部的 Errata 节为最高优先级。
>
> 基线：Canvas `lineage-24.0` @ `115bed9`（本分支 `arena/bcdbc086-gkuicanvas` 仅多出文档提交）。依赖版本：`compose-material3 = 1.5.0-alpha16`、`material-icons-extended = 1.7.8`、`navigation3 = 1.2.0-alpha01`（`gradle/libs.versions.toml:11-16`）。
>
> 下文路径省略前缀 `app/src/main/java/org/lineageos/canvas/`。
>
> **引用规则**：文中行号对应基线 `115bed9` 与撰写时的 androidx-main，可能漂移。实现时以符号名与语义定位为准，行号仅作线索；发现不一致时更新文档而非信任行号。
>
> **修订**：2026-10-14 评审后做了勘误与规格冻结（errata / spec-freeze）。修订记录集中在移植计划顶部的 **Errata** 节（下称“Errata 第 N 条”），本文正文已原地同步。主要变化：§2.4 导出选项整体删除（工具箱域）；§2.2 明确 `writeRequestDenied` 的存活范围并补充 Snackbar 遮挡风险；§2.3 冻结颜色模式（不做）和 `initialText`（做），并明确阴影归属；§2.5 冻结 `centered`（本轮不做）。
>
> **状态说明**：需求要求阶段一完成后暂停确认。本执行环境无法在一次交付中途暂停，因此 §1 最初作为“中间结论”交付，§2–§3 在此基础上继续给出。之后的评审以本文为基础拍板了修订（见上），没有否决 §1 的结论。

---

## 0. 对任务背景描述的勘误（附代码证据）

任务背景描述中有 4 处与代码不符。下文全部按代码事实展开：

| # | 背景描述 | 代码事实 | 证据 |
|---|---|---|---|
| E1 | 现有工具是“裁剪、亮度、对比度、灰度、复古” | 共有 **8 个 `EditMode`**：CROP、ROTATION、MARKER、HIGHLIGHTER、TEXT、BRIGHTNESS、CONTRAST、FILTERS。灰度和复古**不是独立工具**，而是 FiltersScreen 内 `FilterChoice` 的两个选项 | `models/EditMode.kt:15-50`；`ui/screens/FiltersScreen.kt:65-83` |
| E2 | 底栏是平铺的一排工具按钮，加 3 个后有 8 个 | 底栏**已经是两级分组**：首页一级是 3 个分类按钮（变换 / 绘制 / 调整），点开后二级才是该分类的工具。每个分类目前最多 3 个工具 | `ui/composables/CanvasBottomBar.kt:158-191`（`RootHomeToolbar`）、`:203-237`（`CategoryToolbar`）；`EditMode.ofCategory()`（`EditMode.kt:52-56`） |
| E3 | 没有文字输入先例 | 有：`TextEditorOverlay`（全图层遮罩 + `TextField` + 样式开关 + 颜色），供 TextScreen 使用 | `ui/composables/TextEditorOverlay.kt:63`；`ui/screens/TextScreen.kt:104` |
| E4 | 新增一个工具要改 7 处 | 实际是 **8 处**：移植计划 §2 ② 列出了 7 处，但漏掉了 `CanvasTopAppBar` 标题的**穷尽式 `when`**。不补这一分支会编译失败 | `ui/composables/CanvasTopAppBar.kt:65-74` |

E4 也是对移植计划 §2 ② 的勘误：P0-3（水印）和 P1-2（色温）的“修改的接入文件”都要加上 `ui/composables/CanvasTopAppBar.kt`。移植计划正文已同步（Errata 第 1 条）。

---

## 1. 阶段一：Canvas UI 地图（中间结论）

### 1.1 Screen 导航树

导航由 `ui/CanvasApp.kt` 中的 `navigationBackStack = mutableStateListOf<Screen>(Screen.Home)`（`:51`）驱动。它由 `ui/navigation/CanvasNavDisplay.kt` 的 `NavDisplay`（`:137-145`）渲染。`Screen` 只有两种：`Home` 和 `Edit(editMode)`（`ui/navigation/Screen.kt`），所以返回栈深度**最多为 2**。

```text
MainActivity (ACTION_EDIT only)                                   MainActivity.kt:29
└─ CanvasTheme → CanvasApp(editViewModel, onClose, onSave, onSaveAs, onShare)   ui/CanvasApp.kt:31
   └─ Scaffold                                                    CanvasApp.kt:55
      ├─ topBar: CanvasTopAppBar(canClose, onClose, currentScreen, isWritable, onSave, onSaveAs, onShare)
      │    ├─ navigationIcon: Close（仅当 backStack.size == 1）→ onClose → RESULT_CANCELED + finish   :80-92 / MainActivity.kt:108-111
      │    ├─ title: Edit 页显示工具名（穷尽 when）                    :61-78
      │    └─ actions（仅 Home）: SplitButtonLayout                       :95-196
      │         ├─ 主按钮: isWritable ? 保存(Save) : 分享(Share)          :100-127
      │         └─ ▾ DropdownMenu: [分享（仅 isWritable）] [另存为…]       :163-194
      └─ content: CanvasNavDisplay(navigationBackStack, …)                ui/navigation/CanvasNavDisplay.kt:32
         ├─ Screen.Home → HomeScreen                                      ui/screens/HomeScreen.kt:28
         │    ├─ CanvasImage + CanvasBottomBar { HomeBottomBarContent }    :44 / :53
         │    │    ├─ 一级 RootHomeToolbar: 撤销 | [变换][绘制][调整] | 重做
         │    │    └─ 二级 CategoryToolbar: 关闭FAB | [该分类的 EditModeButton…]
         │    ├─ 点工具 → backStack.add(Screen.Edit(mode))                CanvasNavDisplay.kt:57-59
         │    └─ 系统返回: 已选分类 → 清空分类（BackHandler :39）；否则 → Activity 默认返回（finish）
         └─ Screen.Edit(mode)                                             CanvasNavDisplay.kt:63-133
              ├─ CROP        → CropScreen          ui/screens/CropScreen.kt:45
              ├─ ROTATION    → RotationScreen      ui/screens/RotationScreen.kt:47
              ├─ MARKER      → MarkerScreen        ui/screens/MarkerScreen.kt:44  ─┐ 共用私有 StrokeScreen :88
              ├─ HIGHLIGHTER → HighlighterScreen   ui/screens/MarkerScreen.kt:67  ─┘
              ├─ TEXT        → TextScreen          ui/screens/TextScreen.kt:41
              │                  └─ 子状态: TextEditorOverlay（点图片空白处打开）  :104
              ├─ BRIGHTNESS  → BrightnessScreen    ui/screens/BrightnessScreen.kt
              ├─ CONTRAST    → ContrastScreen      ui/screens/ContrastScreen.kt:29
              └─ FILTERS     → FiltersScreen       ui/screens/FiltersScreen.kt:102
                                 └─ 子状态: FilterRail ⇄ FilterSlider（再点已选滤镜进入）  :125-142
```

**进入方式**：所有 Edit 页都只能从底栏二级 `EditModeButton` 进入。顶栏菜单（分享 / 另存为）只触发动作，不会进入任何 Screen。

**退出方式**（逐页核对）：

| Screen | 确认（✓ FAB）做了什么 | 取消（✕ FAB） | 系统返回 |
|---|---|---|---|
| CropScreen | `cropRect != imageBounds` 时提交 `Action.Transformation.Crop`，然后出栈（`CropScreen.kt:84-89`；`CanvasNavDisplay.kt:68-71`） | `removeLastOrNull` | NavDisplay 默认 `onBack` = `removeLastOrNull`，等同取消（navigation3 `NavDisplay.kt:255-259`） |
| RotationScreen | 非 `ROT_0` 时提交 `Transformation.Rotation`（`RotationScreen.kt:100-107`） | 同上 | 同上 |
| Marker / Highlighter | 把 `pendingActions` **逐条** `onAddAction`（N 笔就是 N 个撤销步骤），然后 `onCancel()` 出栈（`MarkerScreen.kt:213-217`） | 丢弃所有笔画 | 同取消；**没有 BackHandler**，未提交的笔画会被静默丢弃 |
| TextScreen | 同上，逐条提交 `Drawing.Text`（`TextScreen.kt:126-130`）；编辑器打开时点 ✓ FAB，编辑器内尚未确认的文字会丢失 | 丢弃 | 编辑器打开时先关闭编辑器（BackHandler `:53-55`），否则等同取消 |
| Brightness | `brightness != 0f` 时提交 `Adjustment.Brightness`（`BrightnessScreen.kt:59-61`） | 丢弃 | 同取消 |
| Contrast | `contrast != 1f` 时提交 `Adjustment.Contrast`（`ContrastScreen.kt:58-62`） | 丢弃 | 同取消 |
| FiltersScreen | 提交 `selection.action`；ORIGINAL 或强度为 0 时为 null（`FiltersScreen.kt:144-146`、`:70-83`） | 丢弃 | 滑块打开时先回到滤镜列表（BackHandler `:112`），否则等同取消 |

**统一的预览模型**：工具页内的所有改动都是本地 `remember` 状态，预览靠 `CanvasImage(colorFilter=…)` 或叠在图上的 Compose `Canvas`。点击确认时才推入线性历史 `HistoryList`，之后由 `EditViewModel.adjustedBitmapWithActions`（`EditViewModel.kt:185-213`）在全分辨率下重放。`cropRect` 只是“当前结果的整幅边界”（`:223-225`），所以 CropScreen 每次进入都从整幅开始。

### 1.2 底栏结构与容量（最关键）

**结构**：底栏没有固定的按钮数，而是由三种容器组合而成：

| 容器 | 代码 | 布局方式 | 子项 |
|---|---|---|---|
| 外壳 `CanvasBottomBar(modifier, onCancel?, onConfirm?, content)` | `CanvasBottomBar.kt:95-117` | `Row`：`padding(bottom=16dp)`、`height(64dp)`、`fillMaxWidth()`、`padding(horizontal=16dp)`。依次为 [✕ FAB]、`Box(weight 1f, Center){content}`、[✓ FAB] | FAB 是 M3 `FloatingActionButton`，56×56dp（`FabBaselineTokens.ContainerWidth/Height = 56dp`） |
| 首页一级 `RootHomeToolbar` | `:158-191` | `Row(fillMaxWidth, SpaceEvenly)`：撤销 `IconButton`、`HorizontalFloatingToolbar(weight(1f, fill=false))` 里放 **`LazyRow`**、重做 `IconButton` | `EditMode.Category.entries`，目前 3 个 |
| 首页二级 `CategoryToolbar` | `:203-237` | `Row(fillMaxWidth, spacedBy(8dp, CenterHorizontally))`：关闭 FAB、`HorizontalFloatingToolbar(weight(1f, fill=false))` 里放 **`LazyRow`**（无额外间距） | `EditMode.ofCategory(category)`；目前变换 2、绘制 3、调整 3 |
| Edit 页中间槽 | 例如 `RotationScreen.kt:98-116` → `RotationBottomBarContent`（`:119-144`）；FiltersScreen 放提示文字（`:147-161`） | `HorizontalFloatingToolbar(expanded=true)` 里直接放 `IconButton` | Rotation 有 2 个 |

**结论 1（布局类型）**：两级工具列表都是 **`LazyRow`**，放在 `HorizontalFloatingToolbar` 中。超出时**可横向滚动并被裁切**：LazyRow 自身会裁切，toolbar 的 `graphicsLayer` 也设置了 `clip = true`（M3 `FloatingToolbar.kt:1498-1510`）。**不会截断丢项，也没有溢出菜单**。唯一的“提示”是最后一个按钮只露出一部分。

**度量来源**（M3 androidx-main 源码；Canvas 使用的 1.5.0-alpha16 需编译时复核数值）：

- `IconButton` 带 `.minimumInteractiveComponentSize()`，再叠加 `size(smallContainerSize())`（`IconButton.kt:247-248`），所以布局占位是 **48dp**（视觉容器 40dp，`SmallIconButtonTokens.ContainerHeight = 40dp`）。
- `HorizontalFloatingToolbar` 的 `contentPadding` 默认为 `FloatingToolbarDefaults.ContentPadding`，左右各 **8dp**（`ContainerLeadingSpace / ContainerTrailingSpace = 8dp`），最小高度 64dp（`FloatingToolbarTokens.ContainerHeight`）。
- FAB：**56dp**。

**360dp 宽度下的容量计算**（竖屏、手势导航，水平 inset 为 0；外壳内容宽 = 360 − 2×16 = **328dp**）：

| 层级 | 计算 | toolbar 内 LazyRow 可用宽度 | 能完整显示的 48dp 按钮数 | 现状 → 移植后 |
|---|---|---|---|---|
| 首页一级 | 328 − 撤销 48 − 重做 48 = 232；减去 toolbar 内边距 16 | **216dp** | **4**（192） | 3 个分类 → **3**（不新增分类） |
| 首页二级 | 328 − 关闭 FAB 56 − 间距 8 = 264；减去 16 | **248dp** | **5**（240） | 2 / 3 / 3 → **2 / 4 / 4**（绘制 + 水印，调整 + 色温） |
| Edit 页中间槽（有 ✕✓ 两个 FAB） | 328 − 56 − 56 = 216；减去 16 | **200dp** | **4**（192） | Rotation 2 → 裁剪 1、水印 2（§2） |

**压力测试：320dp**（小屏手机把“显示大小”调到较大时）。外壳内容宽 288dp：

- 首页一级：192 − 16 = 176 → 3 个按钮。
- 二级：224 − 16 = 208 → **4 个按钮**。
- 中间槽：176 − 16 = 160 → 3 个按钮。

移植后每个分类最多 4 个工具，**在 320dp 下仍能完整显示**。如果新增第 4 个分类，在 320dp 下首页一级就放不下了（4×48 = 192 > 176），这也是 §3 Q1 不新增分类的依据之一。

**结论 2（8 个工具能否放下）**：

- 背景描述设想的“8 个工具平铺一行”在 Canvas 中**并不存在**（E2）。
- 退一步按假设计算：即使把 8 个工具放进一个二级 LazyRow，360dp 下也只能完整显示 5 个，另外 3 个要靠滚动。
- 而真实结构下，移植后单个分类最多只有 4 个工具，**360dp 和 320dp 都无需滚动**。

**结论 3（平铺还是分组）**：

- 底栏工具**已按分类分组**（`EditMode(val category)` + `groupBy`）。
- 滤镜（灰度 / 复古）在 FiltersScreen 内部再分了一层。
- 无需改造任何底栏代码：新工具只要在 `EditMode` 中声明正确的 `Category`，就会自动出现在对应二级栏里。顺序由枚举声明顺序决定，因为 `groupBy` 会保持原有顺序。

### 1.3 可复用组件清单

| 组件 | 签名（原样） | 约定与现有调用方 | 移植中的用途 |
|---|---|---|---|
| `AdjustmentControls` | `fun AdjustmentControls(@StringRes label: Int, value: Float, onValueChange: (Float) -> Unit, valueRange: ClosedFloatingPointRange<Float>, modifier: Modifier = Modifier)`（`ui/composables/AdjustmentControls.kt:26-32`） | `Surface(RoundedCornerShape(16dp), surfaceContainerHigh)`，上方一行是“标签 + 百分比”，百分比显示 `value * 100`，格式为 `adjustment_percent = "%1$.0f%%"`（`strings.xml:42`），下方是 M3 `Slider`。调用方：Brightness `-1f..1f`（恒等值 0）、Contrast `0f..2f`（恒等值 1，`ContrastScreen.kt:48-52`） | 色温、水印字号 |
| └ 双向（bipolar）支持 | — | **没有专门的双向模式**，但 `valueRange = -1f..1f` 已在亮度中使用：轨道从左端开始填充，百分比显示 −100%…100%。M3 提供的 `SliderDefaults.CenteredTrack(sliderState, …)`（`Slider.kt:2051`）可以让轨道从中点填充，但需要改 `AdjustmentControls` 的实现。**冻结：本轮不做**，标注为“未来上游 patch 候选（亮度与色温共同受益）”（Errata 第 10 条） | 色温直接用 `-1f..1f`，**与亮度保持一致**（§2.5） |
| `FilterSlider`（私有） | `private fun FilterSlider(selection, onValueChange, onBack)`（`FiltersScreen.kt:232`） | 带返回箭头的 0–1 滑块，只能在 FiltersScreen 内使用 | 反色强度（自动获得） |
| `StrokeColorPalette` | `fun StrokeColorPalette(selectedColor: Color, onColorSelected: (Color) -> Unit, modifier: Modifier = Modifier)`（`ui/composables/StrokeColorPalette.kt:50-54`） | 胶囊形 `Surface(CircleShape)` 内放可横滚的 `Row`，共 8 色（`:38-47`），每个颜色是一个 `IconButton`，语义为 RadioButton。**唯一调用方**是 `StrokeScreen`（`MarkerScreen.kt:205-211`） | 水印可选的颜色条（§2.3） |
| `TextEditorOverlay` | `fun TextEditorOverlay(onDismiss: () -> Unit, onConfirm: (String, TextStyle) -> Unit, modifier: Modifier = Modifier)`（`TextEditorOverlay.kt:63-67`） | **文字输入的唯一先例**。黑色 80% 遮罩；顶行是 ✕ / ✓；B/I/U/S 四个 `FilledIconToggleButton`（`:149-172`）；一行 8 色 `TextColorButton`（`:175-192`，颜色集合与 StrokeColorPalette 相同）；下面是 `TextField`（`:194+`），占位提示为 `enter_text`。基线中**没有初始文本参数**；水印任务会新增 `initialText: String = ""`（冻结为“做”，见 §2.3、Errata 第 10 条）。它放在图片区域的 `Box` 内，而不是全屏，所以底栏仍然可见（`TextScreen.kt:58-124`） | 水印的自定义文字和“用途”输入（§2.3） |
| `CanvasImage` + `ImageInformation` | `fun CanvasImage(imageBitmap, cropRect, modifier, colorFilter, onImageInformation)`（`CanvasImage.kt:173`）；映射函数有 `originalBitmapRectToViewRect`、`viewRectToOriginalBitmap`、`bitmapOffsetToView` 等（`:50-160`） | 所有 Edit 页的底图，带 sharedElement 过渡 | 全部视觉工具 |
| `CanvasBottomBar` 的 `content` 槽 | 见 §1.2 | 先例：Rotation 放工具条，Filters 放提示文字 | 裁剪的人脸参考开关、水印的模式开关 |
| `ToolbarTooltip` | `fun ToolbarTooltip(text, position = Above, content)`（`ToolbarTooltip.kt:19`） | 所有图标按钮都包一层 | 所有新增图标按钮 |

**裁剪层结构（比例锁定和参考线的叠加位置）**：

```text
CropScreen.Column                                          CropScreen.kt:57
├─ Box(weight 1f)                                          :58
│  ├─ CanvasImage(cropRect = null, padding 16dp) → onImageInformation   :63-69
│  └─ CropOverlay(imageBounds=imageViewRect, cropRect(view), onCropRectChange)   :72-79
│       └─ Canvas  modifier:
│            renderInSharedTransitionScopeOverlay(z=1) + animateEnterExit
│            graphicsLayer(CompositingStrategy.Offscreen)           :123  ← 挖洞依赖它
│            pointerInput: detectDragGestures                       :124-161
│              onDragStart → getHandleForOffset(8 个手柄 + CENTER)   :234-260
│              onDrag      → updateRectWithDrag(rect, handle, drag, bounds, 100dp)   :172-232   ← ① 比例锁定在此
│         onDraw: drawCropOverlay(rect, showHandles = true, style)  :164-168
│              ext/DrawScope.kt:28-50:
│                1. drawRect(dimColor)                  整体压暗
│                2. drawRect(BlendMode.Clear)           挖出裁剪框
│                   ← ② 三分线 / 人脸椭圆画在这里（挖洞之后、边框之前）
│                3. drawRect(Stroke)                    边框
│                4. handleOffsets(rect) 8 个圆点         前 4 个是角点（:18-21），后 4 个是边中点
└─ CanvasBottomBar(onConfirm, onCancel)                    :83-92   ← ③ 中间槽目前为空
```

`CropOverlayStyle`（`ui/theme/CropOverlayStyle.kt`）包含 `dimColor`、`borderColor`、`handleColor`、`strokeWidth = 4dp`、`handleRadius = 10dp`，参考线颜色可以加在这里。

**代码库中不存在的组件**（`grep` 结果为 0）：`ModalBottomSheet`、`AlertDialog`、`Snackbar`/`SnackbarHost`、`Toast`、任何进度指示器、`FilterChip`/`SuggestionChip`、`SegmentedButton`、设置页或 Preference。新增这些组件时需要单独说明理由（见 §2）。

### 1.4 保存 / 分享 UI

| 入口 | 呈现 | 调用链 | 结束方式 |
|---|---|---|---|
| 保存 | 只在 Home 顶栏出现，作为 `SplitButtonLayout` 的主按钮，**仅当 `isWritable` 时显示**（`CanvasTopAppBar.kt:100-127`） | `onSave` → `resultOpsViewModel.saveImage(finalBitmap)`（`MainActivity.kt:112-116`）→ `"wt"` 截断写回（`ResultOpsViewModel.kt:103`） | `EditStatus.Saved` → `setResult(RESULT_OK)` + `finish()`（`MainActivity.kt:65-68`） |
| 分享 | 不可写时作为主按钮；可写时放在 ▾ 菜单中（`:167-180`） | `shareImage` → `EditStatus.Shared` → `ACTION_SEND` chooser（`MainActivity.kt:70-82`） | Activity 留在前台 |
| 另存为… | 始终在 ▾ 菜单中（`:182-193`） | `onSaveAs` → 按**源格式**选择 3 个 `CreateDocument` 启动器之一（`MainActivity.kt:95-103`、`:117-126`）→ `onDocumentCreated` → `saveImageToUri` | 同“保存”，`Saved` 后会 **finish** |
| `isWritable` 来源 | 调用方是否授予 `FLAG_GRANT_WRITE_URI_PERMISSION`（`MainActivity.kt:45`），由 `CanvasApp` 透传（`:39`、`:62`） | | |

**导出选项可以挂载的位置**（现状记录。P1-1 已整体删除，见 Errata 第 4 条，**本段不再对应任何落位任务**）：

- **(a)** 放在“另存为…”菜单项和 `CreateDocument` 启动之间，也就是 `MainActivity` 的 `onSaveAs` lambda。
- **(b)** 放在 DropdownMenu 内，作为“另存为 JPEG / PNG / WEBP”三个子项。只能选格式。
- **(c)** 系统文件选择器返回之后：**不可行**。`CreateDocument(mimeType)` 的 MIME 在注册启动器时就固定了（这正是代码里有 3 个启动器的原因），所以格式必须在启动选择器**之前**确定。

**进度与反馈的现状（缺口）**：

- **U-1**：DropdownMenu 的菜单项 `onClick` **没有**把 `checked` 置回 `false`（`CanvasTopAppBar.kt:171`、`:185`），从系统选择器或分享面板返回后菜单仍然展开。这是独立缺陷，修复只需 2 行；P1-1 删除后，它**不再绑定任何移植任务**，仅作记录。
- **U-2**：`EditStatus.Saving` 和 `EditStatus.Sharing` 会被赋值（`ResultOpsViewModel.kt:99`、`:127`），但**没有任何 UI 消费**：没有进度指示，按钮也不会禁用，可以重复点击。这是独立缺陷，仅作记录，**不绑定任何移植任务**（原先与 P1-1 绑定的进度指示建议已随 P1-1 删除）。
- **U-3**：`EditStatus.Error("Failed")`（`:116`、`:156`）落入 `else -> {}`（`MainActivity.kt:84`），**失败时用户看不到任何提示**。这是独立缺陷，不属于本轮移植任务；修复时可以直接复用 §2.2 引入的 `SnackbarHost`。
- **U-4**：成功唯一的反馈就是 Activity 结束、回到调用方。

### 1.5 现有 Screen 的复杂度上限（§3 Q2 的依据）

| Screen | 行数 | 图片区域的叠加层 | 图片与底栏之间的控件条 | 底栏中间槽 | 页内子状态 | BackHandler |
|---|---|---|---|---|---|---|
| Home | 65 | 图片 | — | 两级工具条 | 分类选择 | ✓ |
| Crop | 260 | 图片 + 手势 Canvas | — | 空 | — | — |
| Rotation | 144 | 图片（旋转） | — | 工具条（2 个按钮） | — | — |
| Marker / Highlighter | 221 | 图片 + 绘制 Canvas | 调色板 | — | — | — |
| Text | 134 + 293 | 图片 + 文字预览 + **编辑遮罩** | — | — | 编辑器开 / 关 | ✓ |
| Brightness / Contrast | 65 | 图片 | 滑块 | — | — | — |
| Filters | 281 | 图片 | **列表 ⇄ 滑块（原地切换）** | 提示文字 | 列表 / 滑块 | ✓ |

**上限规则**（取每一列的最大值）：

- 图片区域最多 3 层叠加（Text）。
- 同一时刻**最多 1 条**控件条；需要多种控件时原地切换（Filters）。
- 中间槽最多放一个工具条（Rotation）。
- 页内子状态最多为“1 个遮罩 + 1 组二选一”。

**没有**向导、分步或多页的先例：返回栈只有 Home 和 Edit 两层。

---

## 2. 阶段二：功能落位方案（原 7 项，§2.4 已删除，现 6 项）

来源说明沿用移植计划 §0 的差分基线。下列各项**全部是 GKUI 新增**，上游 Glimpse 中没有。§2.4 已按评审裁决删除，保留编号是为了不打乱交叉引用。

### 2.1 裁剪：比例锁定 + 证件照预设 + 三分线 / 人脸椭圆（P0-1，来源 F6）

**放置位置**：

- ① 比例选择条：放在 `CropScreen.Column` 中 `Box(weight 1f)` 和 `CanvasBottomBar` 之间（`CropScreen.kt:81` 与 `:83` 之间）。这是其他页面放控件条的同一个位置，见 §1.5。
- ② 人脸参考开关：放在 `CanvasBottomBar` 的 `content` 槽（`:83-92`，目前为空）。外层沿用 `RotationBottomBarContent` 的 `HorizontalFloatingToolbar(expanded = true)`，里面放 1 个 `FilledIconToggleButton`，样式沿用 `TextPropertyButton`（`TextEditorOverlay.kt:242`）。只有选中证件照比例时才显示。
- ③ 参考线：画在 `drawCropOverlay` 的第 2 步（挖洞）和第 3 步（边框）之间，见 §1.3 的层结构。
- ④ 比例锁定：按移植计划 P0-1，把私有的 `updateRectWithDrag`（`:172`）挪到新文件 `ext/CropGeometry.kt`，锁定比例时的拖动逻辑也在那里实现；让 `getHandleForOffset`（`:234`）在锁定时只检测角点和 CENTER。

**交互流程**：

1. 进入裁剪时，比例默认是“自由”，框是整幅图（现状不变）。
2. 点某个比例：以当前框的中心为基准，在图像范围内取该比例下**最大的居中矩形**。计算在 bitmap 坐标系中进行，`ContentScale.Fit` 是等比缩放，所以视图坐标下比例相同。“原图”比例取 `imageBitmap.width / height`。传入的是 `bitmapWithOverlayActions`，已经包含了之前的旋转。
3. 锁定比例后：只有 4 个角点手柄和 CENTER（平移）响应拖动，边手柄不显示也不响应（Errata 第 11 条）；`drawCropOverlay` 只画 `handleOffsets(rect).take(4)`，`getHandleForOffset` 只检测角点和 CENTER。拖角点时以对角点为锚，按主拖动轴推算宽高，再按 `bounds` 和最小 100dp（`minimumCropSizePx`，`CropScreen.kt:110`，作用于较短边）收缩。拖 CENTER 平移的逻辑不变。
4. 三分线在拖动期间显示（`activeHandle != null`；`activeHandle` 已经是 `CropOverlay` 内的 state，读到后会自动重绘）。选中证件照比例时自动显示人脸椭圆，可以用中间槽的开关隐藏。
5. 确认和取消的逻辑**不变**。比例只是页面本地状态，不进入历史。

**新建与复用**：

- 新建 `ui/composables/PresetRail.kt` 中的 `PresetRail`（与 §2.3 共用，见下）。
- 修改私有的 `CropOverlay`，它改为调用 `ext/CropGeometry.kt`，`updateRectWithDrag` 也挪到那个文件里（移植计划 P0-1）。
- 给 `drawCropOverlay` 增加带默认值的参数 `showGrid: Boolean = false`、`showFaceGuide: Boolean = false`、`cornerHandlesOnly: Boolean = false`，命名与移植计划 P0-1 一致（Errata 第 11 条）。它只有 CropScreen 一个调用方，属于增量修改。
- `CropOverlayStyle` 增加 `gridColor`、`guideColor`、`guideStrokeWidth`。
- 复用 `CanvasBottomBar` 的 content 槽、`HorizontalFloatingToolbar`、`FilledIconToggleButton`、`ToolbarTooltip`。
- **`PresetRail` 规格（唯一的新 UI 组件）**：
  - `@Composable fun <T> PresetRail(items: List<T>, selected: T, label: @Composable (T) -> String, onSelect: (T) -> Unit, modifier: Modifier = Modifier, icon: ((T) -> ImageVector)? = null)`。
  - 外层是 `LazyRow(fillMaxWidth, padding(horizontal 16dp, vertical 8dp), spacedBy(8dp))`，与 `FilterRail` 相同（`FiltersScreen.kt:171-176`）。
  - 每项是 `Surface(onClick, height 48dp, shape = CircleShape)`，颜色沿用 FilterRail 的选中 / 未选中配色 `primaryContainer` / `surfaceContainerHighest`（`:193-202`），文字为 `labelLarge`，并设置 `semantics { selected = … }`。
  - 胶囊外形取自 `StrokeColorPalette` 的 `CircleShape`。
  - 不用 88dp 方块：裁剪需要放 6 项，水印需要放 7 项。用 88dp 方块会让控件条高 104dp；48dp 胶囊只需 64dp，给图片多留 40dp。
  - **冻结：采用 `PresetRail`**。不用 M3 `FilterChip`，因为代码库中目前没有 Chip，用它会再多出一种组件类型；`PresetRail` 的配色和间距都沿用现有的 `FilterRail`。
- 通用比例和证件照比例之间，在 LazyRow 中插入一个 `VerticalDivider` 项。

**一致性**：

- 预览：拖动时实时显示，与现状一致。
- 应用：确认后推入一个 `Transformation.Crop`。
- 撤销：一步。比例和参考线都不进入历史，重新进入裁剪时回到“自由”。

**文案与图标**：

| 项 | 字符串键（`values/strings.xml`，建议英文原文） | 图标（Material Symbols 名 → Compose 常量） |
|---|---|---|
| 自由 | `crop_aspect_free` = "Free" | `crop_free` → `Icons.Filled.CropFree` |
| 原图 | `crop_aspect_original` = "Original" | `crop_original` → `Icons.Filled.CropOriginal` |
| 1:1 | `crop_aspect_square` = "Square" | `crop_square` → `Icons.Filled.CropSquare` |
| 一寸 | `crop_aspect_id_one_inch` = "1-inch ID" | `badge` → `Icons.Filled.Badge` |
| 二寸 | `crop_aspect_id_two_inch` = "2-inch ID" | 同上 |
| 护照 | `crop_aspect_passport` = "Passport" | 同上 |
| 人脸参考开关 | `crop_face_guide` = "Face guide" | `face` → `Icons.Filled.Face` |

图标规则（冻结）：`PresetRail` 只在传入 `icon` 时显示图标。裁剪传入上表的图标，水印传 `icon = null`，只显示文字（§2.3）。

### 2.2 覆盖保存的写权限流程（P0-2，来源 F7）

**放置位置**：不新增任何界面元素，只改顶栏主按钮的判定条件和点击后的流程。

- `ui/CanvasApp.kt:62`：传给 `CanvasTopAppBar` 的 `isWritable` 改为 `isWritable || (canRequestWrite && !writeRequestDenied)`，三个值都从 `editViewModel` 收集，因为 `CanvasApp` 只持有它。这样 **`CanvasTopAppBar` 的签名和实现都不用改**：主按钮显示“保存”，▾ 菜单自动出现“分享”（`:167` 的条件是同一个参数）。
- `MainActivity.kt:95-103`：在 3 个 `CreateDocument` 启动器旁边，新注册 `rememberLauncherForActivityResult(StartIntentSenderForResult())`。
- `MainActivity.kt:112-116` 的 `onSave` 改为：已有写权限时直接 `saveImage`；否则启动 `IntentSenderRequest(resultOpsViewModel.createWriteRequest())`。
- 授权被拒后的提示：`Scaffold`（`CanvasApp.kt:55`）新增 `snackbarHost = { SnackbarHost(snackbarHostState) }`，`snackbarHostState` 从 `MainActivity` 作为新参数传入，这是增量修改。
- **`writeRequestDenied` 的存活范围（冻结）**：放在 ViewModel 字段 `EditViewModel.writeRequestDenied: StateFlow<Boolean>` 中，由 `markWriteRequestDenied()` 设置，在 `setUri()` 中随 `isWritable` 一起重置；**不用 `rememberSaveable`**。理由：
  - 授权回调在 `MainActivity` 的启动器中（组合之外），可以直接写 ViewModel；`rememberSaveable` 的状态在 `CanvasApp` 内部，需要再上提一层。
  - 与 `isWritable`、`markWritable()`、`canRequestWrite` 放在一起，只有一个来源。
  - 配置变化后保留，**不会重复弹窗**；新的 `ACTION_EDIT` intent 会重置它，`rememberSaveable` 感知不到这种情况。
  - 已知边界：进程被杀后重建会丢失，最多再弹一次。
  - 详见 Errata 第 8 条。
- **实现注意：Snackbar 遮挡风险**（Errata 第 9 条）。Canvas 的底栏 `CanvasBottomBar` 在 Scaffold 的 **content 内**，不在 `bottomBar` 槽里（§1.2；`HomeScreen.kt:53`）。所以 Scaffold 不会把 Snackbar 抬到底栏上方，Snackbar 可能压住首页底栏（高 64dp，外加底部 16dp 间距，见 `CanvasBottomBar.kt:103-104`），包括中间的 `HorizontalFloatingToolbar`（`:178`、`:227`），以及两侧的取消/确认 FAB（`CancelActionFloatingButton` / `ConfirmActionFloatingButton`，`:124`、`:142`）和返回 FAB（`:216`）。Snackbar 只在首页弹出，但新增工具页同样使用这根底栏，规则一致。实现时必须实测验证，必要时给 `SnackbarHost` 加 `Modifier.padding(bottom = …)`，至少要让出底栏高度和间距共 80dp。

**交互流程**（时机选择的论证见 §3 Q3）：

```text
Home 顶栏 [保存]
 ├─ isWritable == true ─────────────────────────→ saveImage → Saved → RESULT_OK + finish（现状）
 └─ canRequestWrite（MediaStore uri）且未被拒
      → 系统“允许 Canvas 修改此照片？”对话框（框架提供）
          ├─ 允许 → editViewModel.markWritable() → saveImage → 同上
          └─ 拒绝 / 返回 → editViewModel.markWriteRequestDenied()（writeRequestDenied = true）
                → 顶栏主按钮回到现在的“分享”形态（本次会话内不再反复请求）
                → Snackbar: "Can't overwrite the original"  [另存为…] → onSaveAs()（按源格式直接打开系统文件选择器，与现状一致）
```

**新建与复用**：

- 系统对话框由框架提供。
- `SnackbarHost` 是代码库中第一次使用（M3 标准组件）。
- 不改造任何 Screen 或组件。
- **降级预案（仅限一种情况）**：如果实测发现 Snackbar 遮挡底栏，且加 padding 也解决不了，就改用 `Toast` 提示，不改 `CanvasApp` 的签名。降级路径仍然成立，因为拒绝后主按钮会回到“分享”，▾ 菜单中始终有“另存为…”。代价是 Toast 无法带操作按钮。

**一致性**：不进入撤销历史。授权请求是 Activity 结果流程，与已有的 `CreateDocument` 启动器处于同一层，即 `MainActivity.setContent`。配置变化后，`rememberLauncherForActivityResult` 会自动恢复回调；位图保存在 ViewModel 的 StateFlow 中，不会丢失。

**文案与图标**：

- `save_write_denied` = "Can't overwrite the original"。
- Snackbar 的操作按钮复用 `save_as`。
- 图标沿用 `Icons.Default.Save`，无新增。

### 2.3 隐私水印（P0-3，来源 F14，arena 分支 `b6265b7`）

**放置位置**：

- 底栏：在 `EditMode` 的 `TEXT(Category.DRAWING)` 之后加入 `WATERMARK(Category.DRAWING)`，绘制分类变成 4 个工具；§1.2 已验证 360dp 和 320dp 都能放下。
- 需要补分支的 4 处穷尽式 `when`：
  - `CanvasNavDisplay.kt:64`，参照 TextScreen 的接线：使用 `finalResultBitmap` 和 `cropRect`，确认时 `onConfirm` 后出栈。
  - `CanvasBottomBar.kt` `EditModeButton` 中的文案（`:312-321`）和图标（`:329-338`）两处。
  - `CanvasTopAppBar.kt:65-74` 的标题。
- 新页面 `ui/screens/WatermarkScreen.kt`，布局如下，每一层都对应一个现有先例：

```text
WatermarkScreen.Column(fillMaxSize)
├─ Box(weight 1f)                                    ← TextScreen.kt:58
│  ├─ CanvasImage(imageBitmap, cropRect, padding 16dp) { imageInformation = it }
│  ├─ Canvas(fillMaxSize) { clipRect(imageViewRect) { drawTiledWatermark(…视图坐标…) } }   ← MarkerScreen.kt:122 的叠加 Canvas
│  └─ if (editing) TextEditorOverlay(onDismiss, onConfirm, fillMaxSize, initialText)    ← TextScreen.kt:104，复用 + 增量参数 initialText
├─ 控件条（同一时刻只显示一个，原地切换）               ← FiltersScreen.kt:125-142
│  ├─ mode = PRESETS: PresetRail([自定义…][仅供…使用][身份核验][业务办理][租房核验][样本/复印件][他用无效], icon = null)
│  └─ mode = SIZE:    AdjustmentControls(label = watermark_size, value, 0f..1f)
└─ CanvasBottomBar(onCancel, onConfirm) {           ← RotationScreen.kt:98-116
       HorizontalFloatingToolbar(expanded = true) {
           [TextFields 开关: PRESETS] [FormatSize 开关: SIZE]     // 2 个按钮，需 112dp ≤ 中间槽 216dp
       }
   }
```

**交互流程**：

1. 进入后：文字为空，控件条显示预设，没有预览。此时点 ✓ 会 `onConfirm(null)`，与亮度为 0 时一样，不产生动作，直接出栈。
2. 点固定预设（身份核验等）：文字立即生效，开始实时平铺预览。
3. 点“仅供…使用”：打开 `TextEditorOverlay`，底栏中间槽临时显示提示“Enter the purpose”，写法与 FiltersScreen 中间槽的提示文字相同。确认后 `text = getString(watermark_template_used_for, purpose)`，通过 `%1$s` 占位，修复移植计划中的缺陷 D7。页面记住最近一次的 `purpose`；再次点“仅供…使用”时，作为 `initialText` 回填到遮罩中。
4. 点“自定义…”：打开同一个遮罩，确认后直接使用输入的文字；再次打开时，用最近一次的自定义文字作为 `initialText` 回填。遮罩返回的 `TextStyle` 中，颜色、粗体、斜体、下划线、删除线都会用于水印。固定预设沿用最近一次在遮罩中确认的样式，**默认是白色**。
   - **样式不回填**：`initialText` 只回填文字。遮罩的颜色和 B/I/U/S 每次打开都从遮罩自身的默认值开始，不新增 `initialStyle` 参数，这样对 `TextEditorOverlay` 的改动只有一个参数（Errata 第 10 条）。
   - **阴影**：由 `drawTiledWatermark` 内部固定添加，**不依赖 `TextStyle.shadow`**（传入的 `style.shadow` 会被忽略），所以无论选什么颜色，阴影都在，效果与 GKUI 的默认相同（`setShadowLayer(3f, 1.5f, 1.5f, argb(140,0,0,0))`，arena 分支 `ImageToolbox.kt:478`；Errata 第 7 条）。
   - **颜色**：只能在遮罩的 8 色行中选择，**不做**单独的“颜色”模式（见下文冻结项）。
5. 点 FormatSize：控件条原地切换为字号滑块，显示 0–100%，映射到 `textSizeFraction`；默认值按 GKUI 的默认字号标定。系统返回或再点 TextFields 可回到预设（BackHandler 的写法与 `FiltersScreen.kt:112` 相同）。
6. 遮罩打开时，系统返回会先关闭遮罩（与 `TextScreen.kt:53` 相同）。
7. 确认：提交**一个** `Action.Drawing.Watermark`，然后出栈。取消：直接出栈。

**为什么不提供位置参数**：

- 水印是覆盖全图的 −30° 交错平铺。GKUI 源码（`ImageToolbox.applyWatermark()`，`:462`）中**没有**位置参数。
- 需要“在某处放一行字”的场景已经由文字工具覆盖。
- 所以本方案只提供字号，不提供位置。不透明度固定为 0.35，与 GKUI 的 `argb(90,…)` 一致，不提供滑块。

**新建与复用**：

- 新建：`WatermarkScreen` 和 `ext/Watermark.kt`（移植计划已列出）；共用 §2.1 的 `PresetRail`。
- 原样复用：`AdjustmentControls`、`CanvasImage`、`CanvasBottomBar`、`HorizontalFloatingToolbar`、`ToolbarTooltip`。
- 增量复用：`TextEditorOverlay` 新增 `initialText: String = ""`，作为 `rememberTextFieldState` 的初值。这是增量参数，不算重构，现有调用方 TextScreen 不受影响。**冻结为“做”**，是水印任务的正式组成部分（Errata 第 10 条）。
- **对移植计划的修订**：
  - `Action.Drawing.Watermark` 的 `color: Color` 改为 `style: TextStyle`，与 `Action.Drawing.Text` 一致，这样遮罩返回的样式可以原样保存（已写入移植计划 P0-3 和 Errata 第 3 条）。阴影不放在 `style` 中，见上文第 4 步。
  - 不再使用 `OutlinedTextField` 和 `SuggestionChip`：前者有 `TextEditorOverlay` 作为先例，后者由 `PresetRail` 代替。
- **冻结：不做第 3 个“颜色”模式**。理由：遮罩里已经可以改色，固定预设沿用最近一次确认的样式；为了这点收益突破 §1.5 的复杂度上限（页内子状态不超过 2 个）不值得。中间槽保持 2 个按钮（Errata 第 10 条）。

**一致性**：

- 预览：叠加的 Canvas 与导出共用同一个 `drawTiledWatermark`，字号以及阴影的半径和偏移都按图像短边的比例计算，所以所见即所得。
- 应用：确认时推入历史。
- 撤销：一个 Action 对应一步。
- 确认后不能再编辑水印。这与 Canvas 所有工具的模型一致：要修改，先撤销，再重新进入工具。
- 历史中的几何都是“相对于该动作所在时刻的图像状态”（`EditViewModel.kt:181-183`）。水印之后再做裁剪或旋转，会作用在已经烘焙好的水印上，这是预期行为。

**文案与图标**：

- 工具：`edit_mode_watermark` = "Watermark"，`branding_watermark` → `Icons.Filled.BrandingWatermark`。
- 模式开关：`watermark_presets` = "Text"（`text_fields` → `Icons.Filled.TextFields`）、`watermark_size` = "Size"（`format_size` → `Icons.Filled.FormatSize`）。
- 预设：
  - `watermark_custom` = "Custom…"
  - `watermark_template_label` = "For … only"
  - `watermark_template_used_for` = "For %1$s only. Invalid for other uses."
  - `watermark_purpose_hint` = "Enter the purpose"
  - `watermark_preset_identity` = "Identity verification"
  - `watermark_preset_business` = "Business use"
  - `watermark_preset_rental` = "Rental verification"
  - `watermark_preset_copy` = "Copy"
  - `watermark_preset_invalid_other` = "Invalid for other uses"
- 中文译文来自 GKUI 的 `strings.xml`，通过 Crowdin 提交（移植计划 §2 ⑤）。

### 2.4 ~~“另存为”中的导出选项（P1-1，来源 F9 + F10）~~：已删除

- **状态**：**整体删除**。GKUI 源码证据显示，F9（质量档位、目标体积）和 F10（格式转换）的入口只在查看器和相册的工具箱（`ImageToolboxBottomSheet`），编辑器保存流程固定按源格式、质量 95 编码，没有任何选项，属于**工具箱域**。裁决、证据表和域边界判据见移植计划 Errata 第 4、5 条。
- **结果**：Canvas 维持跟随源格式、质量 100 的现状。“另存为…”仍然直接按源格式启动 `CreateDocument`（`MainActivity.kt:117-126`），中间没有面板。
- **本节原有的全部内容作废**：导出面板、格式/质量/大小三组选择、按目标体积编码、配套文案、最小备选方案，以及原先绑定的进度提示。
- **保留为现状记录**：§1.4 的挂载点 (a)(b)(c)，以及缺陷 U-1（菜单点击后不收起）、U-2（保存/分享时没有忙碌态），**都不再绑定任何移植任务**。

### 2.5 色温滑块（P1-2，来源 F4）

**放置位置**：

- 在 `EditMode` 中按 `CONTRAST` → **`TEMPERATURE(Category.ADJUSTMENTS)`** → `FILTERS` 的顺序插入。调整分类的二级栏变为“亮度、对比度、色温、滤镜”，共 4 个，§1.2 已验证可以放下。
- 补 4 处 `when`：`CanvasNavDisplay`（照抄 BRIGHTNESS 分支 `:104-112`）、`EditModeButton` 中的文案和图标两处、`CanvasTopAppBar` 的标题。
- 新页面 `ui/screens/TemperatureScreen.kt` 以 `BrightnessScreen.kt` 为模板逐行对应：`CanvasImage(colorFilter = temperatureColorFilter(t))`、`AdjustmentControls(label = edit_mode_temperature, valueRange = -1f..1f)`、`CanvasBottomBar(onConfirm = t.takeIf { it != 0f }?.let(::Temperature))`。

**交互流程**：与亮度完全相同。左侧（负值）偏冷，右侧（正值）偏暖，百分比显示 −100%…100%。

**双向滑块**：

- `AdjustmentControls` 没有双向模式，但 `-1f..1f` 已经由亮度验证过（§1.3）。为了保持一致，**不改 `AdjustmentControls`**。
- **冻结：`centered` 本轮不做**，标注为“未来上游 patch 候选（亮度与色温共同受益）”。设想是给 `AdjustmentControls` 加 `centered: Boolean = false`，打开时用 `SliderDefaults.CenteredTrack`。它改的是上游共享组件，适合单独向 LineageOS 提交，不随本轮移植；届时需要先确认依赖的 material3 版本中有这个 API（Errata 第 10 条）。
- 已知局限：拖回恰好 0 很难，用户需要点取消才能回到恒等状态。这一点**与亮度相同**，本次不处理。

**新建与复用**：新建 1 个 Screen，属于模板复制；其余全部复用。

**一致性**：预览用 `colorFilter`，确认后推入一个 `Adjustment.Temperature`，撤销一步。

**文案与图标**：`edit_mode_temperature` = "Temperature"；`thermostat` → `Icons.Filled.Thermostat`。

### 2.6 反色滤镜（P2-1，来源 F3）

**放置位置**：在 `FiltersScreen.kt:65-83` 的 `FilterChoice` 末尾、`SEPIA` 之后加入 `INVERT(R.string.filter_invert, Icons.Filled.InvertColors, 1f)`。**底栏不变**。

**交互流程**：与灰度和复古完全相同。点击后预览；再点一次进入强度滑块，范围 0–100%；确认后提交 `Adjustment.Invert(amount)`。现有提示文案 `filter_select_hint` 和 `filter_tap_again_to_adjust` 直接复用。

**容量说明**：

- FilterRail 的方块是 88dp，间距 8dp，左右各留 16dp 内边距。放 4 项需要 4×88 + 3×8 = 376dp，所以屏幕宽度至少要 **408dp** 才能全部显示。
- 360dp 屏幕上，第 4 项只露出 40/88，需要横向滚动（LazyRow 原生支持）。411dp 的设备（如 Pixel 7）可以完整显示。
- 这符合上游 LazyRow 的设计预期，**不建议**为此缩小方块，那会改变现有 Screen 的视觉。

**新建与复用**：全部复用，只新增 1 个枚举项。

**一致性**：同 Filters：预览用 `colorFilter`，确认后推入一个 `Adjustment`，撤销一步。

**文案与图标**：`filter_invert` = "Invert"；`invert_colors` → `Icons.Filled.InvertColors`。

### 2.7 覆盖保存时保留 EXIF（P2-2）

**无 UI 改动。** 理由如下：

1. **策略是确定的**：覆盖保存时保留 EXIF（同一文件，不应改变拍摄时间）；另存为和分享时不带 EXIF（维持现状）。这里不存在需要用户做的选择。
2. **覆盖保存路径上没有可以挂 UI 的位置**：点击“保存”后立即执行写入（`MainActivity.kt:112-116`），中间没有任何界面。插入确认对话框会改变现有流程，而且代码库中没有 `AlertDialog` 先例。
3. **没有设置页基础设施**：代码库中没有设置页、Preference 或 DataStore（`grep` 结果为 0）。为一个开关新建持久化和设置页，违反“优先插入”的原则。
4. 原先唯一可能的位置是 §2.4 的导出面板，现已随 P1-1 删除。即使它还在，也只用于“另存为”，而另存为的策略本来就是不带 EXIF，与“覆盖保存时保留”无关。

字符串和图标：无。

---

## 3. 三个必须论证的问题

### Q1 底栏拥挤：滚动、分组还是溢出菜单？

**结论：选“分组”。分组已经存在，需要的改动最少：每个新工具只在 `EditMode` 中加 1 个枚举项，底栏代码 0 处改动；`EditModeButton` 中的 `when` 分支是任何方案都必须加的。**

论据：

1. **前提不成立**：底栏并非平铺（§0 E2）。新增的 3 项中，反色进入 FiltersScreen，不占底栏；水印和色温分别进入已有分类，二级栏最多 4 个。
2. **容量充足**：二级栏在 360dp 下可完整显示 5 个、在 320dp 下可显示 4 个（§1.2 表）。4 个工具在两种宽度下都不需要滚动。
3. **滚动是天然的兜底**：二级栏本身就是 `LazyRow`。将来某个分类超过 5 个时会自动变成可滚动，不需要任何改动。
4. **不选溢出菜单**：底栏没有溢出菜单的先例；需要在 `HorizontalFloatingToolbar` 中新加 `DropdownMenu`，会改变现有交互，增加改动，也违反“不改整体结构”的约束。
5. **不新增第 4 个分类（如“隐私”）**：需要改 `CategoryButton` 的两处 `when`、加新文案和新图标；而且首页一级在 320dp 下只能放 3 个分类（§1.2）。

### Q2 水印交互：单屏还是分步？

**结论：单屏，加原地模式切换。不做分步向导。**

论据（以 §1.5 的复杂度上限为准）：

| 维度 | 现有上限（来源） | WatermarkScreen | 是否超限 |
|---|---|---|---|
| 图片区域叠加层 | 3 层（Text：图片、预览、遮罩） | 3 层（图片、平铺预览、`TextEditorOverlay`） | 持平 |
| 控件条 | 同一时刻 1 条，原地切换（Filters：列表 ⇄ 滑块） | 同一时刻 1 条（预设 ⇄ 字号） | 持平 |
| 底栏中间槽 | 工具条（Rotation，2 个按钮） | 工具条（2 个开关） | 持平 |
| 页内子状态 | 遮罩开关（Text）、二选一（Filters） | 遮罩开关 + 二选一 | 等于两者之和，每一部分都有先例 |
| BackHandler 层级 | 1 层（Text / Filters） | 2 层（先关遮罩，再回到预设） | 多 1 层，但规则与已有写法相同 |

- **不选分步**：Canvas 没有分步、向导或步骤指示器的先例。返回栈只有 Home 和 Edit 两层（§1.1）。分步需要新的导航层级或步骤组件，违反“不改变整体导航结构”的约束。
- **单屏堆叠也不可行**：移植计划原来的设想是把文本框、建议芯片、两个滑块和调色板同时堆在一屏上。这会出现 4 条控件条，**超出上限**，因此在 §4 中修订掉。

### Q3 写权限请求时机：进入编辑器时，还是点保存时？

**结论：在点“保存”时请求。被拒后的降级路径是：Snackbar 上的“另存为…”操作，加上顶栏回到“分享 + 另存为”的现有形态。**

论据：

1. **只有保存需要写权限**：`isWritable` 在 UI 中唯一的使用方是顶栏的保存 / 分享按钮（`CanvasApp.kt:39` → `CanvasTopAppBar.kt:101-117`、`:167`）。所有 Edit 页都与写权限无关，所以进入时请求对编辑过程没有任何好处。
2. **大部分会话可能不需要写权限**：用户可能只是分享、另存为，或者直接关闭（`onClose` → `RESULT_CANCELED`，`MainActivity.kt:108-111`）。进入时就弹系统对话框，会在用户还没看到图片之前打断他们，而且很多时候这个请求根本用不上。
3. **在保存时请求，请求和用户意图一一对应**：“保存”按钮本身就说明了用途，用户拒绝的含义也很明确，就是“不要覆盖原图”。所以降级到“另存为副本”顺理成章。
4. **技术上可行**：授权结果回调和已有的 `CreateDocument` 回调处在同一层（`MainActivity.setContent`）。最终位图和 `writeRequestDenied` 都保存在 ViewModel 中，对话框期间或拒绝之后发生配置变化，既不会丢失，也不会重复弹窗（§2.2）。
5. **降级路径**：
   - 拒绝后调用 `editViewModel.markWriteRequestDenied()`，主按钮回到现在不可写时的“分享”形态，本次会话内不再反复请求。
   - Snackbar 的操作按钮直接打开“另存为…”，按源格式启动系统文件选择器。
   - ▾ 菜单中始终保留“另存为…”。
   - **不会**自动弹出文件选择器：用户刚刚拒绝过一次，再自动弹窗过于强势。
6. **前置风险**（移植计划 P0-2 已标注）：需要在真机上验证，只持有临时读权限时 `createWriteRequest` 能否正常弹出。如果不能，`canRequestWrite` 恒为 false，UI 自动退回现状，不需要任何额外处理。

---

## 4. 对移植计划“UI 重做说明”的修订

| 移植计划原文 | 本文修订 | 理由 |
|---|---|---|
| P0-1：比例选择用 `FilterChip` 一行；中间放 `IconToggleButton` | 比例选择改用 `PresetRail`（48dp 胶囊，配色取自 FilterRail）；中间槽改为 `HorizontalFloatingToolbar` 加 `FilledIconToggleButton` | 沿用现有组件的用法（FilterRail、Rotation、`TextPropertyButton`）；已冻结为 `PresetRail`，不保留 `FilterChip` 备选 |
| P0-1：三分线只在拖动时显示 | 不变；此外明确：锁定比例时只有 4 个角点和 CENTER（平移）响应拖动；`drawCropOverlay` 参数为 `showGrid` / `showFaceGuide` / `cornerHandlesOnly` | 边手柄在锁定比例时无法单独定义；参数命名已与移植计划统一（Errata 第 11 条） |
| P0-3：`OutlinedTextField` + `SuggestionChip` + 字号和不透明度两个滑块 + `StrokeColorPalette`，全部堆叠在一屏 | `TextEditorOverlay` 复用（加 `initialText`）+ `PresetRail` + 一个字号滑块，原地切换；不透明度固定为 0.35；**不做**颜色模式；`TextEditorOverlay` 增加 `initialText` 回填 | 符合 §1.5 的上限（Q2）；复用文字输入的先例；冻结项见 Errata 第 10 条 |
| P0-3：`Watermark(color: Color, …)` | `Watermark(style: TextStyle, …)` | 原样保存遮罩返回的样式，与 `Action.Drawing.Text` 一致 |
| P0-3：阴影通过 `TextStyle(shadow = …)` 实现 | 阴影由 `drawTiledWatermark` 内部固定添加，`style.shadow` 被忽略 | 遮罩返回的样式不带阴影，换色后阴影仍然存在（Errata 第 7 条） |
| P0-3 / P1-2 的接入文件 | 加上 `ui/composables/CanvasTopAppBar.kt` | §0 E4 |
| ~~P1-1：导出选项面板~~ | **整个任务已删除**，§2.4 作废 | 工具箱域（Errata 第 4、5 条） |
| P1-2：双向滑块 | `AdjustmentControls` 不改；`centered` 本轮不做 | 未来上游 patch 候选（亮度与色温共同受益） |
| P0-2：“不新增界面” | 拒绝后用 Snackbar 提示，并提供“另存为”操作；`writeRequestDenied` 和 `canRequestWrite` 放在 `EditViewModel`；只有 Snackbar 遮挡底栏且加 padding 也解决不了时，才降级为 Toast | 落实降级路径（Q3）；配置变化后不重复弹窗（Errata 第 8、9 条） |

---

## 5. 改动汇总与重构标记

### 5.1 新增 UI 文件（都需要 SPDX 头）

本表只列 UI 文件。非 UI 的新文件（`models/CropAspectRatio.kt`、`ext/CropGeometry.kt`、`ext/Watermark.kt` 等）及其 SPDX 头以移植计划各任务的“新增文件与 SPDX 头”为准。

| 文件 | 来源任务 | SPDX 头（与仓库现有 Kotlin 文件相同，块注释） |
|---|---|---|
| `ui/composables/PresetRail.kt` | P0-1、P0-3 共用（**本文新增**，移植计划中没有） | 版权方 `The LineageOS Project` / 许可证 `Apache-2.0` |
| `ui/screens/WatermarkScreen.kt` | P0-3（移植计划已列出） | 同上 |
| `ui/screens/TemperatureScreen.kt` | P1-2（移植计划已列出） | 同上 |

所有组件都来自 `androidx.compose.material3`（Apache-2.0，属于 `availableInAOSP` 已放行的 `androidx.*`）。**不引入任何第三方或 GPL 代码**。

### 5.2 对现有代码的改动（全部是插入或增量，没有重构）

| 位置 | 改动 | 性质 |
|---|---|---|
| `CropScreen.kt`：`CropOverlay`、`getHandleForOffset` | 增加比例和参考线参数；锁定时只检测角点和 CENTER | 私有函数，增量修改 |
| `CropScreen.kt`：`updateRectWithDrag` | 按移植计划 P0-1 挪到 `ext/CropGeometry.kt`，逻辑不变，锁定比例的分支在那里新增 | **函数搬迁**（移植计划 P0-1 决定，目的是 JVM 单测）。不搬迁的备选：留在 `CropScreen.kt` 原地扩展，代价是无法单测 |
| `ext/DrawScope.kt`：`drawCropOverlay` | 增加带默认值的参数 `showGrid`、`showFaceGuide`、`cornerHandlesOnly` | 增量修改，只有 1 个调用方 |
| `ui/theme/CropOverlayStyle.kt` | 增加 `gridColor`、`guideColor`、`guideStrokeWidth` | 增量修改 |
| `models/EditMode.kt` | 加 2 个枚举项 | 插入 |
| `CanvasBottomBar.kt`、`CanvasTopAppBar.kt`、`CanvasNavDisplay.kt` | 补穷尽式 `when` 分支 | 插入 |
| `FiltersScreen.kt` | 在 `FilterChoice` 中加 1 项 | 插入 |
| `viewmodels/EditViewModel.kt` | 加 `canRequestWrite`、`writeRequestDenied: StateFlow<Boolean>`、`markWritable()`、`markWriteRequestDenied()`；在 `setUri()` 中重置 | 增量修改 |
| `viewmodels/ResultOpsViewModel.kt` | 加 `createWriteRequest()` | 插入 |
| `CanvasApp.kt` | 改 `isWritable` 表达式（从 `editViewModel` 读取三个值）；`Scaffold` 加 `snackbarHost`；加 `snackbarHostState` 参数 | 增量修改 |
| `MainActivity.kt` | 只加授权启动器（`StartIntentSenderForResult`）及其回调 | 插入 |
| `TextEditorOverlay.kt` 的 `initialText` | 增量参数 `initialText: String = ""` | 增量修改；**做**，是水印任务的正式组成部分 |
| `AdjustmentControls.kt` 的 `centered` | 不改 | **本轮不做**；未来上游 patch 候选（亮度与色温共同受益） |
| `CanvasTopAppBar.kt` 的 U-1、U-2 | 不改 | 现状缺陷，仅在 §1.4 记录，不绑定任何移植任务 |

**重构标记**：唯一接近重构的是 `updateRectWithDrag` 的函数搬迁（逻辑不变，已给出不搬迁的备选）。其余现有 Screen 都不需要拆分、改结构或改导航。上表各行都已冻结为“做”或“本轮不做”，没有待定项。

---

## 6. 自查

- [x] 导航树覆盖了全部 9 个 Screen 入口：Home + 8 个 EditMode（Marker 和 Highlighter 共用 StrokeScreen），以及 2 个页内子状态（TextEditorOverlay、FilterSlider）。每页的确认、取消和系统返回都有行号。
- [x] 容量结论来自 `CanvasBottomBar.kt:95-237` 的真实布局，加上 M3 token 数值（IconButton 48dp、FAB 56dp、toolbar 内边距 8dp），按 360dp 和 320dp 逐级计算。数值取自 androidx-main，实现时需要在 1.5.0-alpha16 上复核。
- [x] 复用组件已列全（§1.3）。新增的 UI 文件只有 1 个本文提出的组件（`PresetRail`）和 2 个移植计划中已有的 Screen（§5.1）。
- [x] Q1、Q2、Q3 都有代码证据（§3）。
- [x] 没有重构，唯一的函数搬迁已标记并给出备选；§5.2 中没有待定项（§2.3、§2.5 的冻结项已落实）。
- [x] F7（EXIF）判定为“无 UI 改动”，并给出了 4 条理由（§2.7）。
- [x] 没有改动启动模型（仍然只有 `ACTION_EDIT`），没有新增 `uses-permission`。
- [x] 本文自身带 SPDX 头，可以通过 REUSE 检查。
- [ ] 需要编译验证：`Icons.Filled.CropFree`、`CropOriginal`、`CropSquare`、`Badge`、`Face`、`FormatSize`、`BrandingWatermark`、`Thermostat`、`InvertColors` 在 material-icons-extended 1.7.8 中存在。符号名已经在 `google/material-design-icons` 仓库中核对过。
- [ ] 需要实机验证：Snackbar 不遮挡底栏和 FAB，必要时加 bottom padding（§2.2、Errata 第 9 条）。

## 附录：复现命令

```bash
# 底栏结构与容量
sed -n 95,237p app/src/main/java/org/lineageos/canvas/ui/composables/CanvasBottomBar.kt
# 组件调用点
grep -rn "AdjustmentControls(\|StrokeColorPalette(\|TextEditorOverlay(\|HorizontalFloatingToolbar" app/src/main/java
# 确认不存在的组件
grep -rn "ModalBottomSheet\|AlertDialog\|Snackbar\|Toast\|ProgressIndicator\|FilterChip\|Preference" app/src/main/java
# M3 token（androidx-main）
curl -s -H "Accept: application/vnd.github.raw" \
  "https://api.github.com/repos/androidx/androidx/contents/compose/material3/material3/src/commonMain/kotlin/androidx/compose/material3/tokens/FloatingToolbarTokens.kt?ref=androidx-main"
```
