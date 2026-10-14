<!--
     SPDX-FileCopyrightText: The LineageOS Project
     SPDX-License-Identifier: Apache-2.0
-->

# GKUI Photos 编辑特色功能 → Canvas 移植要点

> 文档性质：改进要点，作为后续编码任务的输入。**本文档不包含任何代码改动**。
>
> 状态：阶段一表格已于 2026-10-14 经需求方确认，并按反馈做了三处调整：
> ① F4 暖/冷色按源码事实移植为“色温”调节；
> ② F12 GIF 编辑器**不移植**；
> ③ 额外纳入 arena 分支上的“隐私水印”（F14）。PDF 导出属于 GKUI 图库工具箱，不纳入。

## 0. 调查基线与勘误

### 0.1 差分基线

| 项 | 值 |
|---|---|
| GKUI 仓库 | `gx-bangsong/GKUIphotos`，主线分支 `lineage-23.2`，HEAD `5158d58` |
| 上游 | `LineageOS/android_packages_apps_Glimpse`，分支 `lineage-23.2`，HEAD `c7b5e8c` |
| `git merge-base HEAD upstream/lineage-23.2` | `c7b5e8cfbb4e941473f3179322ec8513d83b4ca9`（GKUI 在 `5158d58` 合入了上游最新提交，所以分叉点等于上游 HEAD；GKUI 第一个自研提交 `eca6f1b` 的父提交是上游的 `160c5b6`） |
| 主线上 GKUI 独有的非 merge 提交 | 15 个（`eca6f1b` … `a83caa8`，2026-08-02 至 08-05） |
| 差分规模 | `git diff --stat upstream/lineage-23.2...HEAD`：50 个文件，+4567 / −86 |
| 额外纳入的分支（需求方指定） | `origin/arena/01a0b551-gkuiphotos`。它与上游的 merge-base 同样是 `c7b5e8c`，且主线 HEAD 是它的祖先（线性追加）。纳入其中的提交 `b6265b7`（隐私水印）；同一分支上的 `175d2c2`（PDF 导出、分辨率档位）和 `cc40a3e`（批量工具箱）属于图库工具箱，不纳入 |
| 第三方依赖变化 | `git diff c7b5e8c <ref> -- '*.gradle*' '*.toml'` 对主线和 arena 分支**输出都为空**，即 GKUI 没有引入 uCrop、GPUImage、glide-transformations 等任何新依赖 |
| Canvas 基线 | `LineageOS/android_packages_apps_Canvas`，分支 `lineage-24.0`，HEAD `115bed9`。本仓库 `gx-bangsong/GKUICanvas` 与它 `diff -r` 完全一致 |

主线 GKUI 独有提交（`git log --oneline --no-merges c7b5e8c..5158d58`）及其归属：

```
a83caa8 fix(toolbox): make video->GIF not OOM/corrupt                 → F11/F12（GifCodec）
c89ba43 i18n(zh-rCN): refine the General settings strings             → 非编辑
3b2ddaa fix(viewer): actually hide the toolbox capsule over videos    → 工具箱入口 UI（非编辑）
fb5efed i18n(zh): add Simplified/Traditional Chinese ...              → 非编辑
46042be fix(viewer): keep the toolbox off the video progress bar      → 工具箱入口 UI（非编辑）
fff1100 feat(video): replace redundant speed chip with press-and-hold → 非编辑（视频）
f4f2961 fix(toolbox): give each action button a distinct label        → F9/F10
1b2e6ee fix: share EXIF-stripped copy via FileProvider + tidy toolbox → F13、F9/F10
2e9487b fix(editor): use setDataAndType so the image editor receives  → F1–F8
38be2a1 / c52df98 fix(ThumbnailAdapter): ... blur RenderEffect        → 非编辑（只修复上游已有的模糊效果）
ff8741d fix: drop ExifInterface.TAG_HOST_COMPUTER                     → F13
09fb7ad / 5d02a0a Add GitHub Actions workflow                         → CI
eca6f1b feat: offline productivity suite for Glimpse (7 modules)      → F1–F13 的主体
（arena 分支）b6265b7 feat: add Xiaomi-style privacy watermark ...    → F14
```

### 0.2 已排除：上游原有功能，或与编辑无关

| 排除项 | 证据 |
|---|---|
| 发送 `ACTION_EDIT` 打开外部编辑器 | `ext/Intent.kt` 中的 `buildEditIntent()` 在上游 `lineage-23.2` 和 `lineage-24.0` 中写法相同，且只带 `FLAG_GRANT_READ_URI_PERMISSION` |
| 锁屏下编辑需先解锁 | 上游提交 `3246358` |
| 缩略图选中时的模糊 `RenderEffect` | 上游 `ThumbnailAdapter.kt:211` 原本就有；GKUI 只改成懒加载 |
| 设为壁纸 / “用作” | 上游原有功能 |
| 画中画、长按快进、倍速 | GKUI 自研，但属于视频播放 |
| 智能相册（`SourceAlbumAggregator`） | GKUI 自研，但属于图库功能 |
| PDF 导出、批量工具箱 | arena 分支上的图库工具箱功能，需求方明确不纳入 |

### 0.3 勘误：Canvas 目前没有“白平衡”工具

按 `115bed9` 的源码核实：

- `models/EditMode.kt` 中只有 `CROP`、`ROTATION`、`MARKER`、`HIGHLIGHTER`、`TEXT`、`BRIGHTNESS`、`CONTRAST`、`FILTERS`。
- `models/Action.kt` 中 `Action.Adjustment` 下只有 `Brightness`、`Contrast`、`Sepia`、`Grey`。
- 滤镜只有原图、灰度、怀旧三种，强度可调。
- 在源码中 grep `white|balance|temperature|tint` 没有结果，上游 86 个提交的 `git log --grep` 也没有。

此外，Canvas 的裁剪**没有比例锁定，也没有参考线**：`ext/DrawScope.kt` 的 `drawCropOverlay()` 只画遮罩、边框和 8 个手柄。

---

## 1. GKUI 特色编辑功能总表（阶段一结果，已确认）

GKUI 路径相对于 `GKUIphotos/app/src/main/java/org/lineageos/glimpse/`；Canvas 路径相对于 `app/src/main/java/org/lineageos/canvas/`。

| # | 功能名称 | 入口位置 | 核心算法文件 / 关键类 | UI 耦合度 | 外部依赖（许可证） | 与 Canvas 重叠度 | commit | 结论 |
|---|---|---|---|---|---|---|---|---|
| F1 | 90° 旋转 | `ViewActivity` 的 `adjustButton`（`:294`）进入 `ImageEditorActivity`，点击 `rotateButton` | `ImageEditorActivity.rotate()`（`:333`） | 算法 + Activity | 平台 API | **完全重复**（`RotationScreen.kt`、`ext/ImageBitmap.kt` 中的 `rotateBy()`） | `eca6f1b` | 放弃，直接用 Canvas 原生 |
| F2 | 灰度、怀旧滤镜 | `ImageEditorActivity.setupFilterChips()`（`:141`） | `utils/media/ImageFilter.kt` 中的 `GRAYSCALE`、`SEPIA` | 纯算法 | 平台 API | **完全重复**（`Action.Adjustment.Grey/Sepia`，强度可调，比 GKUI 更完整） | `eca6f1b` | 放弃，直接用 Canvas 原生 |
| F3 | 反色滤镜 | 同上 | `ImageFilter.INVERT` | 纯算法 | 平台 API | 无重叠 | `eca6f1b` | **P2** |
| F4 | 暖色 / 冷色 | 同上 | `ImageFilter.WARM`（R×1.2、G×1.05、B×0.8）和 `COOL`（R×0.8、B×1.2） | 纯算法 | 平台 API | 按源码无重叠（见 §0.3） | `eca6f1b` | **P1**，做成连续可调的“色温” |
| F5 | 自由裁剪（固定框、拖动图片） | `ImageEditorActivity` 的 `ratioChipGroup` 第一项 | `ui/views/CropImageView.kt` 中的 `crop()`（`:213`） | 算法 + 自定义 View | 平台 API | **完全重复**（`CropScreen.kt`、`Action.Transformation.Crop`） | `eca6f1b` | 放弃，直接用 Canvas 原生 |
| F6 | 证件照比例锁定裁剪 + 三分线 + 人脸对齐椭圆 | `ImageEditorActivity.setupRatioChips()`（`:165`）和 `faceGuideButton` | `CropImageView.computeCropRect()`（`:142`）、`drawGuides()`（`:157`） | 算法 + 自定义 View | 平台 API | 部分重叠（Canvas 有裁剪，但无比例锁定和参考线） | `eca6f1b` | **P0** |
| F7 | 覆盖保存时的写权限回退（`createWriteRequest`） | `ImageEditorActivity` 的覆盖保存路径 | `ImageEditorActivity.overwrite()`（`:236`）、`writeRequestContract`（`:75`） | 算法 + Activity | AndroidX Activity（Apache-2.0） | 部分重叠（Canvas 只在调用方授予写权限时才能保存，见 `MainActivity.kt:45`） | `eca6f1b`、`2e9487b` | **P0** |
| F8 | 保存默认行为设置 + 另存到 `Pictures/Glimpse` | `ImageEditorActivity.onSave()`（`:205`）；设置页 | `utils/EditSaveBehavior.kt`；`ext/SharedPreferences.kt`；`res/xml/root_preferences.xml`；`saveAsNew()`、`writeToMediaStore()`（`:262`、`:271`） | **深度耦合 GKUI**（设置页） | androidx.preference、AppCompat（Apache-2.0） | 部分重叠（Canvas 已分开提供“保存”和“另存为…”） | `eca6f1b` | 放弃：Canvas 现有的保存 / 另存为（SAF）已覆盖，且 Canvas 没有设置页 |
| F9 | 图片压缩（质量档位 / ≤500 KB） | `ViewActivity` 的 `toolboxButton`（`:317`）或菜单 `R.id.toolbox`（`:244`），打开 `ImageToolboxBottomSheet` | `utils/media/ImageToolbox.kt`：`compress()`（`:73`）、`encodeToTargetSize()`（`:226`） | 纯算法（与 MediaStore 写入耦合） | 平台 API | 无重叠（Canvas 固定质量 100） | `eca6f1b`、`1b2e6ee`、`f4f2961` | **P1**，与 F10 合并为“导出选项” |
| F10 | 格式转换（JPEG/PNG/WEBP） | 同上 | `ImageToolbox.convert()`（`:103`）、`ImageTargetFormat` | 纯算法 | 平台 API | 部分重叠（Canvas 有三种编码器，但“另存为”只能用源格式） | 同上 | **P1**，与 F9 合并 |
| F11 | 视频转 GIF | 工具箱的视频区 | `ImageToolbox.videoToGif()`（`:137`）+ `GifCodec.encode()` | 纯算法 | 平台 API | 无重叠，但输入是视频，超出 Canvas 范围 | `eca6f1b`、`a83caa8` | 不移植，保留在 GKUI Photos |
| F12 | GIF 编辑器 | `adjustButton`，当 MIME 为 `image/gif` 时 | `GifEditorActivity.kt`、`utils/media/GifCodec.kt` | 编辑器 UI 深度耦合；GifCodec 是纯算法 | 平台 API | 无重叠（Canvas 不接收 `image/gif`） | `eca6f1b`、`a83caa8` | **不移植**（需求方决定），保留在 GKUI Photos |
| F13 | 分享前剥离 EXIF | `ViewActivity` 的 `shareButton`（`:272`），打开 `SharePrivacyBottomSheet` | `utils/media/ExifStripper.kt`：`strip()`（`:91`）、`LOCATION_TAGS`（`:42`）、`DEVICE_AND_TIME_TAGS`（`:66`） | 纯算法 + BottomSheet | androidx.exifinterface、androidx.core（Apache-2.0） | 按效果完全重复（Canvas 分享时重新编码，本来就不带 EXIF，见 `ResultOpsViewModel.kt:138-140`） | `eca6f1b`、`ff8741d`、`1b2e6ee` | 分享面板放弃；标签分类复用到 **P2** 的“保存时元数据策略” |
| F14 | **隐私水印**（−30° 斜向交错平铺、阴影、“仅供…使用”模板与快捷预设） | `ImageEditorActivity` 的 `watermarkButton`，打开 `showPrivacyWatermarkDialog()`（`:233`）；保存时在 `:216` 烘焙进图片 | `utils/media/ImageToolbox.kt` 中的 `applyWatermark()`（`:462`）；`CropImageView.drawWatermark()`（`:156`，实时预览） | 算法 + 自定义 View + BottomSheet | Material Components（Apache-2.0，仅 UI） | 无重叠（Canvas 的文字工具只能放单个文字标签） | `b6265b7`（arena 分支） | **P0** |

### 1.1 已确认的 GKUI 源码缺陷（移植时必须修复，不能照搬）

| # | 缺陷 | 位置 | 影响到的移植任务 |
|---|---|---|---|
| D1 | GIF 帧延迟单位错误：编码时把毫秒值写进 1/100 秒字段，导出的 GIF 慢 10 倍 | `GifCodec.kt:399`（解码 `:94` 有 ×10 换算） | 不移植（F11/F12），但应反馈给 GKUI |
| D2 | LZW 字典用 `HashMap<List<Byte>,Int>`，每个像素都分配新 List | `GifCodec.lzwEncode()` | 同上 |
| D3 | 目标体积压缩：`:242` 是死代码，分辨率最多只缩小一次，无法保证 ≤ 目标值 | `ImageToolbox.encodeToTargetSize()` | P1-1 |
| D4 | 编辑时把图片降到长边 2048 像素，所以覆盖保存会降低原图分辨率 | `ImageEditorActivity.kt:323` | 全部任务（Canvas 必须保持全分辨率） |
| D5 | 水印字号按绝对像素截断：预览限制在 24–48 px，导出限制在 24–96 px。在 Canvas 的全分辨率图片上，预览和导出大小会不一致，大图上水印也会变得很小 | `CropImageView.kt:156` 起；`ImageToolbox.kt:462` 起 | P0-3 |
| D6 | 水印在保存时才烘焙，不进历史，无法撤销 | `ImageEditorActivity.kt:216` | P0-3 |
| D7 | “仅供…使用”模板靠在字符串里查找 `……` 或 `...` 来选中占位符，无法正常本地化 | `showPrivacyWatermarkDialog()` | P0-3 |
| D8 | `IdPhotoPreset` 没有被任何地方引用，是死代码 | `ImageFilter.kt` 末尾 | P0-1 |

### 1.2 GKUI Photos 需要配合的改动（不属于 Canvas，记录备查）

GKUI 在 `ViewActivity.kt:294-311` 改写了编辑按钮，静态图片和 GIF 都走自己的编辑器。要实现“编辑归 Canvas”，需要恢复成上游的 `startActivity(Intent.createChooser(buildEditIntent(it), null))`。上游 `buildEditIntent()` 只授予读权限，所以 P0-2 是避免功能退化的前提。

---

## 2. Canvas 架构接入点分析

**① 启动模型。** `AndroidManifest.xml` 只为 `MainActivity` 声明了一个 intent-filter：`android.intent.action.EDIT` + `DEFAULT`，MIME 限于 `image/jpeg`、`image/png`、`image/webp`。**没有 `MAIN/LAUNCHER`**。`MainActivity.onNewIntentListener` 会再校验一次 action 和 `ImageFormat.fromMimeTypeOrNull()`，不符合就直接 `finish()`。“是否可写”只看调用方有没有授予 `FLAG_GRANT_WRITE_URI_PERMISSION`（`MainActivity.kt:45`）。本方案所有任务都不改动这套启动模型，也不新增 MIME 类型。

**② 工具抽象。** Canvas 没有 `Tool` 接口，而是用 `enum EditMode(category)` + `sealed interface Action` + `Screen.Edit(editMode)`，再加上若干**穷尽式 `when`** 组成。新增一个工具需要改动以下文件，编译器会对遗漏的分支报错：

1. `models/EditMode.kt`：新增枚举项并指定 `Category`。
2. `models/Action.kt`：如果是新的位图操作，新增 `Adjustment`、`Transformation` 或 `Drawing` 子类型。
3. `viewmodels/EditViewModel.kt`：在 `applyAction()`（`:264`）和 `drawAction()`（`:277`）中加分支。
4. `ui/screens/<Xxx>Screen.kt`：新建工具页面。
5. `ui/navigation/CanvasNavDisplay.kt`：在 `when (it.editMode)` 中加分支。
6. `ui/composables/CanvasBottomBar.kt`：在 `EditModeButton` 中加文案和图标分支（新增 Category 时还要改 `CategoryButton`）。
7. `res/values/strings.xml`：新增 `edit_mode_xxx`。

如果新工具复用已有的 Action（例如比例锁定裁剪继续用 `Transformation.Crop`），第 2、3 步可以省略。

**③ 位图管线与撤销栈。**

- 解码：`EditViewModel.decodeBitmap()` 用 `BitmapFactory.decodeStream` 以**全分辨率**解码（`:110`，没有设置 `inSampleSize`），再把 EXIF 方向烘焙进像素。
- 撤销栈：`HistoryList<Action>` 是线性撤销/重做，插入新动作时丢弃可重做部分，通过 `snapshot` StateFlow 驱动 UI。
- 渲染：`adjustedBitmapWithActions`（`:185`）在 `Dispatchers.IO` 上把所有生效的 Action 依次重放到源图上。`RenderCache` 只复用“追加”场景的前缀；撤销后会从源图整体重放。`finalResultBitmap` 就是同一个 Flow 的别名（`:218`）。
- 预览与应用：工具页里的预览是绘制时给 `CanvasImage(colorFilter=…)` 加的 GPU ColorFilter，或者叠在图上的 Compose `Canvas`；点击确认后才把 Action 推入历史，并在全分辨率下用 `CanvasDrawScope` 烘焙。所以“预览”和“全尺寸应用”已经分开，但**没有降采样的代理图**。
- 由此得出三条接入约束：新算法必须是 `(ImageBitmap, params) -> ImageBitmap` 的纯函数；几何参数必须使用“前序动作产出图像”的像素坐标（见 `Action.kt` 的注释）；滑块拖动时只更新预览，不能每帧都烘焙全图。

**④ 保存流程。**

- **保存**：`ResultOpsViewModel.saveImageToUri()` 使用 `contentResolver.openOutputStream(targetUri, "wt")`（`:103`）。**已确认带 truncate**，可以防止 ACropalypse 那类旧数据残留。编码固定为质量 100，WEBP 使用 `WEBP_LOSSLESS`。
- **另存为**：通过 SAF `CreateDocument` 选择位置，但格式只能是源格式（`MainActivity.kt:96-102`），之后同样走 `"wt"`。
- **分享**：写到 `filesDir/images/share_temp.<ext>`（`FileOutputStream` 默认会截断），再通过 FileProvider 用 `ACTION_SEND` 发送，并附带 `clipData` 和读权限。
- **元数据**：三条路径都不写任何 EXIF。分享时这对隐私有利；覆盖保存时却会丢掉拍摄时间等信息（见 P2-2）。
- **错误反馈**：`EditStatus.Error("Failed")` 在 `MainActivity` 的 `LaunchedEffect` 中没有处理，用户看不到失败提示。这是一项独立的观察，建议另行修复。

**⑤ 构建与合规。** `app/Android.bp` 的 `static_libs` 由 `generateBp` 插件生成。`availableInAOSP` 只放行 `androidx.*`、`org.jetbrains*`、`com.google.android.material` 和 `com.google.guava`，其他依赖都会让 AOSP 构建失败。Gerrit 检查流程（`.github/workflows/gerrit.yml`）会运行 REUSE lint。翻译通过“Automatic translation import”（Crowdin）导入，所以新文案**只写在 `values/strings.xml`**，GKUI 已有的中文译文应通过 Crowdin 提交。

---

## 3. 移植任务清单（按优先级排序）

> 通用规则（适用于每一项）：GKUI 原有的 View、XML 布局、BottomSheet、Chip 面板**一律不照搬**，只保留算法和参数。交互层按 Canvas 现有的 Compose + Material 3 Expressive 风格重写：底部使用 `CanvasBottomBar`（取消/确认 FAB），横向选项使用与 `FiltersScreen` 一致的 `LazyRow`，工具入口放在 `HorizontalFloatingToolbar` 中，配色使用动态取色的 `MaterialTheme.colorScheme`。

### 汇总

| 优先级 | 任务 | 来源 | 新增 / 修改文件 | 预估行数 |
|---|---|---|---|---|
| **P0-1** | 裁剪比例锁定 + 证件照预设 + 三分线 / 人脸参考椭圆 | F6 | 新增 2 / 修改 4 + strings | ~350 |
| **P0-2** | 覆盖保存的写权限回退（`createWriteRequest`） | F7 | 新增 0 / 修改 5 | ~100 |
| **P0-3** | 隐私水印工具 | F14 | 新增 2 / 修改 5 + strings | ~450 |
| **P1-1** | 导出选项：格式转换 + 质量 / 目标体积 | F9 + F10 | 新增 3 / 修改 2 + strings | ~360 |
| **P1-2** | 色温调节 | F4 | 新增 1 / 修改 6 + strings | ~120 |
| **P2-1** | 反色滤镜 | F3 | 新增 0 / 修改 4 + strings | ~50 |
| **P2-2** | 保存时的元数据策略（复用 ExifStripper 的标签分类） | F13 | 新增 1 / 修改 1 | ~160 |
| 放弃 | F1、F2、F5（Canvas 原生已覆盖）；F8（改用保存 / 另存为）；F13 的分享面板（分享本来就不带 EXIF）；F11、F12（超出范围或需求方决定） | — | 0 | 0 |

合计：新增约 9 个文件，修改约 15 个不重复的文件，总计约 1600 行（不含测试）。

---

### P0-1 裁剪比例锁定 + 证件照预设 + 参考线（F6）

- **落地方式**：继续使用 `EditMode.CROP`，**复用 `Action.Transformation.Crop(rect)`**，不新增 Action，所以撤销/重做和重放管线都不用改。
  - 新建 `models/CropAspectRatio.kt`：`enum class CropAspectRatio(val ratio: Float?, val faceGuide: Boolean)`，取值为 `FREE(null)`、`ORIGINAL`（运行时取图片宽高比）、`SQUARE(1f)`、`ID_ONE_INCH(25f/35f, true)`、`ID_TWO_INCH(35f/49f, true)`、`PASSPORT(35f/45f, true)`。比例数值来自 GKUI 的 `setupRatioChips()`（`:165`），GKUI 中 `IdPhotoPreset` 的死代码不迁移。
  - 新建 `ext/CropGeometry.kt`（纯函数，便于单元测试）：
    - 把 `CropScreen.kt:172` 中私有的 `updateRectWithDrag()` 挪到这里。
    - 新增 `fitRectToAspect(rect, ratio, bounds)`：切换比例时，以当前框的中心为准，求出符合比例的最大矩形。
    - 新增锁定比例下的拖动逻辑：拖角手柄时固定对角，按比例缩放；拖边手柄时沿该轴缩放，另一条边以中心对称调整；`CENTER` 手柄只平移；结果始终限制在图像范围内，并满足最小尺寸 `minimumCropSizePx`。
  - 修改 `ext/DrawScope.kt`：`drawCropOverlay()` 增加 `showGrid` 和 `showFaceGuide` 参数。三分线和椭圆的几何参数照搬 GKUI 的 `drawGuides()`：椭圆高为裁剪框的 0.6 倍，宽为椭圆高的 0.75 倍，上沿距裁剪框顶部 12%；虚线使用 `PathEffect.dashPathEffect(floatArrayOf(14f, 10f))`。
  - 修改 `ui/theme/CropOverlayStyle.kt`：增加 `gridColor`、`guideColor` 和 `guideStrokeWidth`。
  - 修改 `ui/screens/CropScreen.kt`：增加比例状态和选择行；手柄拖动改为调用 `CropGeometry`。
- **UI 重做说明**：不移植 GKUI“固定框、拖动或缩放图片”的交互（`CropImageView` 的手势），保留 Canvas 的手柄模型。比例选择放在图片和 `CanvasBottomBar` 之间，用一行 `FilterChip` 横向滚动；通用比例和证件照比例之间用分隔线隔开。选中证件照比例时自动显示人脸椭圆；`CanvasBottomBar` 中间放一个 `IconToggleButton`（`Icons.Filled.Face`）用来开关椭圆。三分线只在拖动时显示，与系统相机和相册的习惯一致。
- **依赖处理**：只用 Compose UI Graphics（Apache-2.0），零新增依赖。
- **新增文件与 SPDX 头**：

  | 文件 | SPDX 头 |
  |---|---|
  | `app/src/main/java/org/lineageos/canvas/models/CropAspectRatio.kt` | `/* SPDX-FileCopyrightText: The LineageOS Project` / `SPDX-License-Identifier: Apache-2.0 */` |
  | `app/src/main/java/org/lineageos/canvas/ext/CropGeometry.kt` | 同上 |

- **修改的接入文件**：`ui/screens/CropScreen.kt`、`ext/DrawScope.kt`、`ui/theme/CropOverlayStyle.kt`、`res/values/strings.xml`（约 8 条：`crop_aspect_free`、`crop_aspect_original`、`crop_aspect_square`、`crop_aspect_id_one_inch`、`crop_aspect_id_two_inch`、`crop_aspect_passport`、`crop_face_guide` 等）。
- **预估改动量**：新增 2 个文件约 180 行，修改 4 个文件约 170 行。
- **验收标准**：选择一寸比例后，确认得到的 `Crop.rect` 宽高比与 25:35 的误差不超过 1 像素；撤销后恢复到上一步；旋转后再裁剪，比例依然正确。

### P0-2 覆盖保存的写权限回退（F7）

- **背景**：上游 Glimpse 唤起编辑时只授予读权限（§1.2），Canvas 因此只显示“分享”，GKUI 用户原有的“覆盖保存”能力会丢失。
- **落地方式**：
  - `viewmodels/ResultOpsViewModel.kt`：
    - 新增 `canRequestWrite`：仅当 `uri.authority == MediaStore.AUTHORITY` 时为真。
    - 新增 `createWriteRequest(): IntentSender`，内部调用 `MediaStore.createWriteRequest(contentResolver, listOf(uri)).intentSender`。
    - 写入仍然走 `saveImageToUri(…, "wt")`，**不另开写入路径**。
  - `MainActivity.kt`：在 `setContent` 中用 `rememberLauncherForActivityResult(StartIntentSenderForResult())` 注册结果回调；结果为 `RESULT_OK` 时先标记为可写，再调用 `saveImage()`。
  - `viewmodels/EditViewModel.kt`：新增 `markWritable()`。
  - `ui/CanvasApp.kt`、`ui/composables/CanvasTopAppBar.kt`：主按钮的逻辑改为：已有写权限时直接“保存”；能请求写权限时，“保存”先发起授权请求；两者都不满足时才显示“分享”。溢出菜单保持“分享”和“另存为…”。
- **不移植的部分**：GKUI 的做法是先尝试写入，捕获异常后再回退（`overwrite()`）。Canvas 已经能从 intent flag 知道是否可写，不需要靠异常驱动流程。
- **UI 重做说明**：不新增界面，系统授权对话框由框架提供。顶栏图标沿用 `Icons.Default.Save`。
- **依赖处理**：只用平台 API `MediaStore.createWriteRequest`（API 30+，Canvas 的 minSdk 是 36）和已有的 AndroidX Activity。**不得新增任何权限**，包括 `READ_MEDIA_IMAGES` 和 `MANAGE_MEDIA`。⚠️ 需要在真机上验证：只持有调用方给的临时读权限时，`createWriteRequest` 能否正常弹出授权。如果不能，保持现状（分享 / 另存为），并改为推动 GKUI 端授予写权限。
- **新增文件**：无。
- **修改的接入文件**：`MainActivity.kt`、`viewmodels/ResultOpsViewModel.kt`、`viewmodels/EditViewModel.kt`、`ui/CanvasApp.kt`、`ui/composables/CanvasTopAppBar.kt`。
- **预估改动量**：修改 5 个文件约 80–120 行。

### P0-3 隐私水印工具（F14）

- **落地方式**：
  - `models/Action.kt`：在 `Action.Drawing` 下新增

    ```kotlin
    data class Watermark(
        val text: String,
        val color: Color,
        val alpha: Float,              // 默认 0.35，对应 GKUI 的 argb(90, …)
        val textSizeFraction: Float,   // 相对图像短边的比例，替代 GKUI 的绝对像素截断（修复 D5）
        val angleDegrees: Float = -30f,
    ) : Drawing
    ```

    因为它属于 `Action.Drawing`，`EditViewModel.applyAction()` 现有的 `is Action.Drawing ->` 分支会自动创建新位图，只需在 `drawAction()` 中补一个分支。水印也就自然进入了历史，可以撤销和重做（修复 D6）。
  - 新建 `ext/Watermark.kt`：实现纯函数 `DrawScope.drawTiledWatermark(layout: TextLayoutResult, …)`。算法照搬 GKUI 的 `ImageToolbox.applyWatermark()`（`:462`）：
    - 绕中心旋转 `angleDegrees`，覆盖范围取图像对角线长度；
    - `stepX = textW + 3.2·base`，`stepY = textH + 3.8·base`，奇数行偏移 `stepX/2`；
    - 字号 `base = textSizeFraction × min(w, h)`；
    - 阴影通过 Compose `TextStyle(shadow = Shadow(...))` 实现。

    文本布局只用 `TextMeasurer` 测量一次，再对每块平铺调用 `drawText(layoutResult, topLeft)`，与 `EditViewModel` 中 Text 动作的做法一致。
  - 新建 `ui/screens/WatermarkScreen.kt`：用 `CanvasImage(onImageInformation=…)` 获取视图与图像之间的映射，在上面叠一层 Compose `Canvas`，以视图坐标调用**同一个** `drawTiledWatermark`，确保预览和导出所见即所得（参照 TextScreen 中 `imagePixelsPerSp` 的做法，以及上游提交 `67467e7`“Match text overlay preview and export size”）。
  - `models/EditMode.kt`：新增 `WATERMARK(Category.DRAWING)`。
  - `ui/navigation/CanvasNavDisplay.kt`、`ui/composables/CanvasBottomBar.kt`：补分支，图标用 `Icons.Filled.BrandingWatermark`。
- **UI 重做说明**：不照搬 GKUI 的 BottomSheet 和 `TextInputEditText`。新界面分为三部分：
  - 上方是实时预览。
  - 中间是单行 `OutlinedTextField`，只让用户填写“用途”。模板改为可翻译的字符串资源 `watermark_template_used_for = "仅供%1$s使用，他用无效"`，不再靠查找省略号来定位占位符（修复 D7）。
  - 下方用一行 `SuggestionChip` 提供快捷预设：身份核验、业务办理、租房核验、样本/复印件、他用无效（来自 GKUI 的 `strings.xml`）。再往下是字号和不透明度两个滑块（沿用 `AdjustmentControls` 的样式），以及复用的 `StrokeColorPalette(selectedColor, onColorSelected)`（GKUI 只有白色，在白底证件上看不清）。
  - 最底部是 `CanvasBottomBar` 的取消和确认按钮。
- **UX 建议（需评审）**：水印通常用于把证件“分享出去的副本”。建议历史中存在 Watermark 动作时，顶栏主按钮优先显示“另存为 / 分享”，避免用带水印的图片覆盖原始证件照。
- **依赖处理**：使用 Compose UI Text/Graphics（Apache-2.0）。GKUI 用到的 Material Components（`BottomSheetDialog`、`Chip`、`TextInputEditText`）**不引入**：它们虽然是 Apache-2.0 且在 AOSP 中可用，但属于 View 体系，与 Canvas 纯 Compose 的架构不符。
- **新增文件与 SPDX 头**：

  | 文件 | SPDX 头 |
  |---|---|
  | `app/src/main/java/org/lineageos/canvas/ext/Watermark.kt` | `The LineageOS Project` / `Apache-2.0`（块注释） |
  | `app/src/main/java/org/lineageos/canvas/ui/screens/WatermarkScreen.kt` | 同上 |

- **修改的接入文件**：`models/Action.kt`、`models/EditMode.kt`、`viewmodels/EditViewModel.kt`（只改 `drawAction`）、`ui/navigation/CanvasNavDisplay.kt`、`ui/composables/CanvasBottomBar.kt`、`res/values/strings.xml`（约 12 条）。
- **预估改动量**：新增 2 个文件约 380 行，修改 6 个文件约 70 行。
- **验收标准**：同一张图在 1080p 和 8000 px 两种尺寸下，水印占画面的比例一致；预览截图与导出结果的平铺位置偏差不超过 1 个 tile；撤销后水印消失。

### P1-1 导出选项：格式转换 + 质量 / 目标体积（F9 + F10）

- **落地方式**：
  - 新建 `models/ExportOptions.kt`：`data class ExportOptions(val format: ImageFormat, val quality: Int = 100, val maxBytes: Long? = null)`。
  - 新建 `utils/ImageEncoder.kt`（纯函数）：
    - `encode(bitmap, format, quality)`。
    - `encodeToMaxBytes(bitmap, format, maxBytes)`：先在质量 1–95 之间二分查找；仍然超出时，按 0.85 倍逐步缩小分辨率，直到满足目标或达到最小边长（修复 D3）。
    - PNG 不支持质量参数，只能通过缩小分辨率来满足目标体积；有损 WEBP 使用 `Bitmap.CompressFormat.WEBP_LOSSY`。
  - 修改 `viewmodels/ResultOpsViewModel.kt`：`saveImageToUri()` 增加 `options` 参数；先编码成 `ByteArray`，再通过 `"wt"` 写入。
  - 修改 `MainActivity.kt`：`onSaveAs` 先弹出导出选项，再用所选格式调用**现有的**三个 `CreateDocument` 启动器之一；文件名使用现有的 `suggestedFilename(format)`。
  - 新建 `ui/composables/ExportOptionsSheet.kt`。
- **范围约束**：覆盖保存**保持源格式和质量 100**（不能更改已有文档的 MIME，也不应在用户不知情时有损重压缩）。导出选项只出现在“另存为…”中，分享是否也加入留待 P2 讨论。GKUI 那种直接插入 MediaStore、写到 `Pictures/Glimpse` 的做法不移植，统一使用 SAF，无需任何权限。
- **UI 重做说明**：使用 M3 `ModalBottomSheet`。格式用 `SingleChoiceSegmentedButtonRow` 选择 JPEG / PNG / WEBP；质量预设为“高 / 中 / 低”（85 / 70 / 50，来自 GKUI）和“≤ 500 KB”，用横向 `FilterChip` 排列，选择 PNG 时禁用质量档位；底部放确认按钮。不照搬 GKUI 的 `RadioGroup` 面板。
- **依赖处理**：平台 `Bitmap.compress`，零新增依赖。
- **新增文件与 SPDX 头**：`models/ExportOptions.kt`、`utils/ImageEncoder.kt`、`ui/composables/ExportOptionsSheet.kt`，都使用 `The LineageOS Project` / `Apache-2.0`。
- **修改的接入文件**：`MainActivity.kt`、`viewmodels/ResultOpsViewModel.kt`、`res/values/strings.xml`（约 10 条）。
- **预估改动量**：新增 3 个文件约 300 行，修改 3 个文件约 60 行。

### P1-2 色温调节（F4）

- **落地方式**：
  - `models/Action.kt`：新增 `Action.Adjustment.Temperature(val value: Float)`，取值范围 [-1, 1]。
  - `ext/ImageAdjustments.kt`：新增 `temperatureColorFilter(t)`，按 GKUI 的 WARM/COOL 增益做连续插值：`R = 1 + 0.2t`，`G = 1 + 0.05·max(t, 0)`，`B = 1 − 0.2t`。t = 1 时等于 GKUI 的 WARM，t = −1 时等于 COOL，t = 0 时不改变图像。再新增 `ImageBitmap.adjustTemperature()`，复用文件里已有的私有函数 `applyColorFilter()`。
  - `viewmodels/EditViewModel.kt`：在 `applyAction()` 和 `drawAction()` 中各加一个分支。
  - `models/EditMode.kt`：新增 `TEMPERATURE(Category.ADJUSTMENTS)`。
  - 新建 `ui/screens/TemperatureScreen.kt`：以 `BrightnessScreen.kt` 为模板，用 `AdjustmentControls(valueRange = -1f..1f)`，预览通过 `colorFilter`。
  - `CanvasNavDisplay.kt`、`CanvasBottomBar.kt`：补分支，图标用 `Icons.Filled.Thermostat`。
- **前提条件**：如果在动工前上游已经合入白平衡或色温工具，本任务降级为“放弃”。
- **UI 重做说明**：与亮度、对比度页面保持一致，不保留 GKUI 的两个固定 Chip。
- **依赖处理**：平台 ColorMatrix（Compose `ColorFilter.colorMatrix`），零新增依赖。
- **新增文件与 SPDX 头**：`ui/screens/TemperatureScreen.kt`，使用 `The LineageOS Project` / `Apache-2.0`。
- **修改的接入文件**：`models/Action.kt`、`models/EditMode.kt`、`ext/ImageAdjustments.kt`、`viewmodels/EditViewModel.kt`、`ui/navigation/CanvasNavDisplay.kt`、`ui/composables/CanvasBottomBar.kt`、`res/values/strings.xml`。
- **预估改动量**：新增 1 个文件约 65 行，修改 7 个文件约 55 行。

### P2-1 反色滤镜（F3）

- **落地方式**：
  - `ui/screens/FiltersScreen.kt`：在 `FilterChoice` 中新增 `INVERT(R.string.filter_invert, Icons.Filled.InvertColors, 1f)`。
  - `models/Action.kt`：新增 `Action.Adjustment.Invert(val amount: Float)`。
  - `ext/ImageAdjustments.kt`：新增 `invertColorFilter(amount)`，按强度在单位矩阵和 GKUI 的 INVERT 矩阵（−1 / +255）之间插值。
  - `viewmodels/EditViewModel.kt`：补两个分支。
- **UI 重做说明**：直接融入现有的滤镜列表，交互与灰度、怀旧相同（再次点击可调强度）。
- **依赖处理**：零新增依赖。
- **新增文件**：无。
- **修改的接入文件**：`ui/screens/FiltersScreen.kt`、`models/Action.kt`、`ext/ImageAdjustments.kt`、`viewmodels/EditViewModel.kt`、`res/values/strings.xml`。
- **预估改动量**：修改 5 个文件约 50 行。

### P2-2 保存时的元数据策略（复用 F13 的标签分类）

- **问题**：Canvas 覆盖保存后不带任何 EXIF，拍摄时间等信息会丢失，可能导致图库时间线位置变化（需验证 MediaStore 重新扫描后的 `DATE_TAKEN` 行为）。
- **落地方式**：新建 `utils/ExifPolicy.kt`，迁移 GKUI `ExifStripper.kt` 中的两份标签清单 `LOCATION_TAGS`（`:42`）和 `DEVICE_AND_TIME_TAGS`（`:66`）。
  - **覆盖保存**：从源 Uri 读取 EXIF，复制到输出。`Orientation` 重置为 `NORMAL`（像素已经转正），更新 `PixelXDimension` 和 `PixelYDimension`，丢弃缩略图。
  - **另存为 / 分享**：保持现在“不带 EXIF”的行为。
  - **写入步骤**：先编码到 `cacheDir` 中的临时文件，用 `ExifInterface(file)` 设置属性并调用 `saveAttributes()`，再以 `"wt"` 复制到目标 Uri，最后删除临时文件。不能对目标 Uri 原地调用 `saveAttributes`，因为它需要可 seek 的文件描述符，而且会绕开 truncate 约束。
- **UI 重做说明**：不移植 GKUI 的分享隐私面板。初版不加界面，以后如需开关，放在“另存为”的导出选项里。
- **依赖处理**：`androidx.exifinterface` 已经在 Canvas 的依赖中（Apache-2.0），零新增。
- **新增文件与 SPDX 头**：`utils/ExifPolicy.kt`，使用 `The LineageOS Project` / `Apache-2.0`。
- **修改的接入文件**：`viewmodels/ResultOpsViewModel.kt`。
- **预估改动量**：新增 1 个文件约 120 行，修改 1 个文件约 40 行。
- **备注**：这项会改变 Canvas 的保存语义，建议先在上游 Gerrit 单独讨论。

### 放弃 / 不移植（附理由）

| 功能 | 理由 |
|---|---|
| F1 旋转 | Canvas 已有 `RotationScreen`，并支持逆时针旋转 |
| F2 灰度 / 怀旧 | Canvas 的版本强度可调，比 GKUI 更完整 |
| F5 自由裁剪 | Canvas 的 `CropScreen` 已经覆盖；P0-1 只补比例锁定和参考线 |
| F8 保存默认行为设置 | Canvas 定位为轻量编辑器，没有设置页；“保存”和“另存为…”分开已经覆盖“每次询问”的场景 |
| F13 分享隐私面板 | Canvas 分享时重新编码，本来就不带任何元数据 |
| F11 视频转 GIF | 输入是视频，需要 `video/*` intent-filter，超出 Canvas 的定位 |
| F12 GIF 编辑器 | 需求方决定不移植；如果 GKUI 继续保留，建议先修复 D1 和 D2 |

---

## 4. 统一工程约束检查单

每个移植 CL 提交前逐项勾选：

**启动模型与范围**

- [ ] `AndroidManifest.xml` 中没有新增 `MAIN/LAUNCHER`、新 Activity 或新的 MIME 类型。调试入口如有需要，只能放在 `app/src/debug/AndroidManifest.xml`，并标注为 **debug-only**。
- [ ] 没有新增任何 `uses-permission`（P0-2 只用 `createWriteRequest` 的用户授权流程）。
- [ ] 没有引入网络访问、云端或 ML 能力，所有处理都在本地完成。

**许可证与 REUSE**

- [ ] 所有新增文件都带 SPDX 头。`.kt` / `.kts`：`/*\n * SPDX-FileCopyrightText: The LineageOS Project\n * SPDX-License-Identifier: Apache-2.0\n */`；`.xml`：使用 `<!-- … -->` 形式的同样两行；`.md`：使用 HTML 注释。
- [ ] 本地运行 `reuse lint` 通过（`requirements.txt` 已指定 `reuse`）。
- [ ] 从 GKUI 迁移的代码：GKUI 的源文件本身就标注 `The LineageOS Project` / `Apache-2.0`，与 Canvas 的许可证一致。提交前需取得原作者 gx-bangsong 的同意或 `Signed-off-by`。
- [ ] 🔴 **GPL 等不兼容许可证检查：本次调查没有发现任何 GPL、LGPL、AGPL 代码，也没有任何第三方依赖**（GKUI 的差分中 gradle/toml 没有变化，编辑相关文件只 import 了 `android.*`、`androidx.*`、`com.google.android.material.*`，均为 Apache-2.0）。如果后续编码引入外部代码片段，必须重新做这项检查，并在 CL 描述中写明来源和许可证。
- [ ] 不引入 `com.google.android.material`（View 体系）。如果确实要引入任何新依赖，必须在 `generateBp.availableInAOSP` 的白名单内，并重新生成 `Android.bp`。手动编辑 `DO NOT EDIT` 区域会被拒绝。

**位图管线**

- [ ] 新算法是 `ImageBitmap → ImageBitmap` 的纯函数，放在 `ext/` 或 `utils/` 下，不依赖 Context 或 UI。
- [ ] 几何参数使用“前序动作产出图像”的像素坐标；需要与图像大小挂钩的尺寸（例如水印字号）存为相对比例。
- [ ] 保持全分辨率处理，**禁止**沿用 GKUI 降到 2048 px 的做法（D4）。
- [ ] 滑块拖动时只更新预览（ColorFilter 或叠加层 Canvas），确认后才 `addAction`。
- [ ] 新 Action 在 `applyAction()` 和 `drawAction()` 的穷尽式 `when` 中都有分支，撤销/重做验证通过。

**保存与隐私**

- [ ] 所有写入已有 Uri 的路径都使用 `openOutputStream(uri, "wt")`，禁止使用默认的 `"w"`（ACropalypse 类问题）。
- [ ] 覆盖保存不改变源 MIME 类型；改变格式只能通过“另存为”。
- [ ] 分享产物仍然不带 EXIF；如果改动了元数据策略（P2-2），需单独评审。
- [ ] 临时文件只放在 `cacheDir` 或 `filesDir/images/`（FileProvider 的 `filepaths.xml` 只暴露 `images/`），用完清理。

**UI 与国际化**

- [ ] 交互层使用 Compose + M3 Expressive（`CanvasBottomBar`、`HorizontalFloatingToolbar`、`ToolbarTooltip`、动态取色），没有照搬 GKUI 的 XML 布局或 View。
- [ ] 每个图标按钮都有 `ToolbarTooltip` 和 `contentDescription`。
- [ ] 新文案只写在 `values/strings.xml`，模板使用 `%1$s` 占位符，不靠查找字符串来定位；中文译文通过 Crowdin 提交，不直接提交 `values-zh-r*`。
- [ ] 不会直接翻译的固定格式（例如 `adjustment_percent`）标注 `translatable="false"`。

**测试（建议）**

- [ ] `CropGeometry`、`ImageEncoder`、颜色矩阵函数补 JVM 单元测试。测试依赖优先使用 `org.jetbrains.kotlin:kotlin-test`（Apache-2.0）。如果使用 JUnit 4（EPL-1.0），只能放在 `testImplementation` 中，不会进入 APK 和 `Android.bp`。

---

## 附录 A：交付前自查

| 自查项 | 结果 |
|---|---|
| 表格中每个功能都能在 `git log` 差分范围内找到对应 commit？ | ✅ F1–F13 对应主线 `c7b5e8c..5158d58` 中的提交（见表格的 commit 列）；F14 对应需求方指定纳入的 arena 分支提交 `b6265b7`（`c7b5e8c..origin/arena/01a0b551-gkuiphotos`），已在 §0.1 单独注明 |
| 每条移植建议都指明了 Canvas 侧的具体接入文件？ | ✅ P0-1 到 P2-2 都列出了“新增文件”和“修改的接入文件”，路径精确到文件 |
| 与 Canvas 重复的功能（裁剪、白平衡、基础滤镜）都已标记为放弃？ | ✅ F1 旋转、F2 灰度/怀旧、F5 自由裁剪已标记放弃。白平衡：经核实 Canvas 源码中**不存在**（§0.3），经需求方确认后 F4 按 P1“色温”移植，并附带“上游先合入白平衡则放弃”的前提条件 |
| 所有第三方依赖都做了许可证与可替换性评估？ | ✅ GKUI 没有新增第三方依赖；用到的 AndroidX、Material Components 均为 Apache-2.0。View 体系的组件全部改用 Compose M3 替代，算法只用平台 `ColorMatrix`、`Bitmap.compress`、`MediaStore`、`ExifInterface` API。没有发现 GPL |

## 附录 B：复现命令

```bash
git clone https://github.com/gx-bangsong/GKUIphotos && cd GKUIphotos
git remote add upstream https://github.com/LineageOS/android_packages_apps_Glimpse
git fetch upstream
MB=$(git merge-base HEAD upstream/lineage-23.2)            # c7b5e8c
git log --oneline --no-merges $MB..HEAD                     # 15 个 GKUI 提交
git diff --stat $MB HEAD                                    # 50 个文件
git diff $MB HEAD -- '*.gradle*' '*.toml'                   # 空：没有新依赖
git log --oneline --no-merges $MB..origin/arena/01a0b551-gkuiphotos   # 包含 b6265b7
git show b6265b7 --stat                                     # 隐私水印
```
