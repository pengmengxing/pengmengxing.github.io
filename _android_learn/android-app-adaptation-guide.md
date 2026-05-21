---
layout: private_post
title: "Android 应用适配完全指南：分辨率与版本兼容"
---

## 一、基础概念

### 1.1 px、dp、sp 是什么

| 单位 | 全称 | 含义 |
|------|------|------|
| **px** | Pixel | 物理像素，屏幕上的一个实际发光点 |
| **dp** | Density-independent Pixel | 密度无关像素，用于布局尺寸 |
| **sp** | Scale-independent Pixel | 缩放无关像素，用于文字大小（会跟随系统字体设置缩放） |

#### 为什么不能直接用 px？

同样写 `100px`，在不同屏幕上物理大小完全不同：

```
低密度屏 (160dpi)：100px ≈ 1.59cm
高密度屏 (480dpi)：100px ≈ 0.53cm   ← 缩小了 3 倍！
```

同样的像素数，高密度屏上看起来小得多。

#### dp 的设计思路

Android 把 **160dpi** 定为基准密度（density = 1），换算公式：

```
px = dp × (dpi / 160)
px = dp × density
```

设定按钮宽度为 `100dp`：

```
160dpi 手机 → density=1 → 100×1 = 100px → 物理宽度 ≈ 1.59cm
320dpi 手机 → density=2 → 100×2 = 200px → 物理宽度 ≈ 1.59cm
480dpi 手机 → density=3 → 100×3 = 300px → 物理宽度 ≈ 1.59cm
```

**像素数不同，但物理尺寸相同** — 这就是"密度无关"的含义。

#### dp 和 sp 的区别

```
dp → 只跟屏幕密度有关
sp → 跟屏幕密度 + 用户字体设置都有关
```

```
用户把系统字体调到 1.5 倍：
  16dp 的文字 → 还是 16dp 对应的像素  （不变）
  16sp 的文字 → 变成 24dp 对应的像素  （放大了）
```

使用规则：
- **布局尺寸**（宽高、间距、圆角）→ 用 `dp`
- **文字大小** → 用 `sp`

### 1.2 像素密度与分辨率

#### 分辨率 (Resolution)

屏幕上**总共有多少个像素点**，用 `宽 × 高` 表示：

```
1080 × 1920  → 横向 1080 个像素，纵向 1920 个像素
1440 × 3200  → 横向 1440 个像素，纵向 3200 个像素
```

分辨率只告诉你像素的**数量**，不告诉你屏幕的**大小**。

#### 像素密度 (PPI / DPI)

每英寸能塞下**多少个像素点**，衡量像素的**紧密程度**。

```
PPI = Pixels Per Inch（每英寸像素数）
DPI = Dots Per Inch（每英寸点数，Android 中与 PPI 等价）
```

计算公式：

```
                √(宽像素² + 高像素²)
像素密度 PPI = ─────────────────────
                  屏幕对角线英寸数
```

#### 关键区别

两块屏幕，**分辨率相同**，但**大小不同**：

```
手机：  1080×1920，5.5 英寸  → PPI ≈ 401（密集，清晰）
平板：  1080×1920，10.1 英寸 → PPI ≈ 218（稀疏，颗粒感）
```

```
┌──────────┐          ┌─────────────────────┐
│ :::::::::: │          │ :  :  :  :  :  :  :  │
│ :::::::::: │          │ :  :  :  :  :  :  :  │
│ :::::::::: │  5.5寸   │ :  :  :  :  :  :  :  │  10.1寸
│ :::::::::: │  401ppi  │ :  :  :  :  :  :  :  │  218ppi
│ :::::::::: │          │ :  :  :  :  :  :  :  │
└──────────┘          └─────────────────────┘
  像素点密集               像素点稀疏
```

#### 三者的关系

```
分辨率（像素数量）
─────────────── = 像素密度（PPI）
屏幕物理尺寸
```

| | 分辨率 | 像素密度 | 屏幕尺寸 |
|--|--------|---------|---------|
| 描述 | 总共多少像素 | 像素有多密 | 屏幕多大 |
| 类比 | 一块地里种了多少棵树 | 树种得多密 | 地有多大 |
| 单位 | px × px | PPI (dpi) | 英寸 |

知道任意两个，就能算出第三个。

#### 常见密度等级

| 密度等级 | dpi | density | 1dp = ?px | 代表设备 |
|---------|-----|---------|-----------|---------|
| mdpi | 160 | 1.0 | 1px | 早期低端机 |
| hdpi | 240 | 1.5 | 1.5px | |
| xhdpi | 320 | 2.0 | 2px | 主流中端机 |
| xxhdpi | 480 | 3.0 | 3px | 主流旗舰 |
| xxxhdpi | 640 | 4.0 | 4px | 高端旗舰 |

---

## 二、不同分辨率设备适配

### 2.1 多套资源目录

```
res/
├── layout/            # 默认布局
├── layout-sw360dp/    # 最小宽度 360dp
├── layout-sw600dp/    # 平板 7寸
├── layout-sw720dp/    # 平板 10寸
├── drawable-mdpi/     # 160dpi
├── drawable-hdpi/     # 240dpi
├── drawable-xhdpi/    # 320dpi
├── drawable-xxhdpi/   # 480dpi
├── drawable-xxxhdpi/  # 640dpi
└── values-sw600dp/    # 平板尺寸值
```

### 2.2 布局方案

- 优先用 `ConstraintLayout`，用比例约束替代固定尺寸
- 用 `match_parent`、`wrap_content`、`0dp`(match_constraint) 而非固定值
- 列表类用 `RecyclerView` + `GridLayoutManager`，动态计算列数：

```kotlin
val columns = (screenWidthDp / 180).coerceAtLeast(2)
recyclerView.layoutManager = GridLayoutManager(context, columns)
```

### 2.3 图片适配

- 矢量图优先用 `VectorDrawable`（SVG），天然适配所有密度
- 位图提供 2-3 套（xhdpi + xxhdpi 通常够用）
- 使用 Glide/Coil 等库根据 View 尺寸加载对应大小

### 2.4 今日头条适配方案（按设计稿等比缩放）

```kotlin
// 核心思路：修改 density 使屏幕宽度固定为设计稿宽度(如 360dp)
fun adaptScreen(activity: Activity, designWidthDp: Int = 360) {
    val appMetrics = activity.resources.displayMetrics
    val targetDensity = appMetrics.widthPixels.toFloat() / designWidthDp
    val targetDensityDpi = (targetDensity * 160).toInt()
    appMetrics.density = targetDensity
    appMetrics.scaledDensity = targetDensity * (appMetrics.scaledDensity / appMetrics.density)
    appMetrics.densityDpi = targetDensityDpi
}
```

---

## 三、不同 Android 版本适配

### 3.1 编译配置

```groovy
android {
    compileSdk 35        // 编译用最新
    defaultConfig {
        minSdk 24        // 最低支持 Android 7.0
        targetSdk 35     // 目标版本
    }
}
```

### 3.2 运行时版本判断

```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    // Android 13+ 专属逻辑
} else {
    // 兼容逻辑
}
```

### 3.3 AndroidX / Jetpack 兼容库

| 问题 | 兼容方案 |
|------|----------|
| 通知 | `NotificationCompat` |
| 权限 | `ActivityResultContracts.RequestPermission` |
| Fragment | `androidx.fragment` |
| 窗口适配 | `WindowCompat` / `WindowInsetsCompat` |
| 后台限制 | `WorkManager` 替代 Service |

### 3.4 各版本重点适配项

```
Android 8  (26) — 通知 Channel 必须创建
Android 10 (29) — 分区存储(Scoped Storage)
Android 11 (30) — 包可见性(package visibility), MANAGE_EXTERNAL_STORAGE
Android 12 (31) — 精确闹钟权限, PendingIntent 必须声明 mutability
Android 13 (33) — 通知运行时权限 POST_NOTIFICATIONS, 细化媒体权限
Android 14 (34) — 前台服务类型必须声明, 隐式 Intent 限制
Android 15 (35) — Edge-to-edge 强制全屏, 16KB page size 对齐
```

### 3.5 权限适配示例（通知权限）

```kotlin
val launcher = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { granted ->
    if (granted) { /* 发通知 */ }
}

fun requestNotification() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        launcher.launch(Manifest.permission.POST_NOTIFICATIONS)
    } else {
        // 13 以下默认有权限，直接发
    }
}
```

### 3.6 存储适配示例

```kotlin
fun saveImage(context: Context, bitmap: Bitmap) {
    val values = ContentValues().apply {
        put(MediaStore.Images.Media.DISPLAY_NAME, "photo_${System.currentTimeMillis()}.jpg")
        put(MediaStore.Images.Media.MIME_TYPE, "image/jpeg")
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            put(MediaStore.Images.Media.RELATIVE_PATH, "Pictures/MyApp")
        }
    }
    val uri = context.contentResolver.insert(
        MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values
    )
    uri?.let { context.contentResolver.openOutputStream(it)?.use { os ->
        bitmap.compress(Bitmap.CompressFormat.JPEG, 90, os)
    }}
}
```

---

## 四、跨版本 UI 显示兼容

### 4.1 核心框架选择

| 方案 | 兼容性 | 适用场景 |
|------|--------|---------|
| **Jetpack Compose** | 不依赖系统版本，UI 一致性最好 | 新项目首选 |
| **AndroidX AppCompat** | Material 控件向下兼容 | 传统 View 体系 |
| **原生 View** | 随系统版本变化大 | 不推荐 |

```kotlin
// Activity 继承 AppCompatActivity，而非 Activity
class MainActivity : AppCompatActivity() { ... }
```

```xml
<!-- 主题用 MaterialComponents / Material3 -->
<style name="AppTheme" parent="Theme.Material3.Light.NoActionBar">
```

### 4.2 Edge-to-Edge（全面屏适配）

Android 15 强制 edge-to-edge，内容会延伸到状态栏和导航栏下方。

```kotlin
WindowCompat.setDecorFitsSystemWindows(window, false)

ViewCompat.setOnApplyWindowInsetsListener(rootView) { view, insets ->
    val bars = insets.getInsets(WindowInsetsCompat.Type.systemBars())
    view.setPadding(bars.left, bars.top, bars.right, bars.bottom)
    insets
}
```

### 4.3 刘海屏 / 挖孔屏（Android 9+）

```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
    window.attributes.layoutInDisplayCutoutMode =
        WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES
}

ViewCompat.setOnApplyWindowInsetsListener(view) { v, insets ->
    val cutout = insets.displayCutout
    cutout?.let {
        val topSafe = it.safeInsetTop
        // 避免内容被刘海遮挡
    }
    insets
}
```

### 4.4 深色模式（Android 10+）

```kotlin
// 统一用 AppCompatDelegate 管理，兼容到 Android 5
AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_FOLLOW_SYSTEM)
```

```xml
<!-- 提供两套颜色资源 -->
res/values/colors.xml           <!-- 亮色 -->
res/values-night/colors.xml     <!-- 暗色 -->
```

### 4.5 圆角屏幕（Android 12+）

```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
    view.setOnApplyWindowInsetsListener { v, insets ->
        val radius = insets.roundedCorner(RoundedCorner.POSITION_TOP_LEFT)?.radius ?: 0
        // 根据圆角半径调整布局边距
        insets
    }
}
```

### 4.6 字体缩放（Android 14 非线性缩放）

```kotlin
// Android 14+ 字体缩放可达 200%，且是非线性的
// 使用 sp 单位会自动适配，但要注意布局溢出
// 关键：不要给文本容器设固定高度
```

```xml
<!-- 错误 -->
<TextView
    android:layout_height="48dp"
    android:textSize="16sp" />

<!-- 正确 -->
<TextView
    android:layout_height="wrap_content"
    android:minHeight="48dp"
    android:textSize="16sp" />
```

### 4.7 屏幕尺寸获取兼容工具

```kotlin
object CompatUtils {
    fun setStatusBarColor(window: Window, color: Int) {
        WindowInsetsControllerCompat(window, window.decorView).apply {
            isAppearanceLightStatusBars = ColorUtils.calculateLuminance(color) > 0.5
        }
        window.statusBarColor = color
    }

    fun getScreenSize(activity: Activity): Pair<Int, Int> {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            val bounds = activity.windowManager.currentWindowMetrics.bounds
            bounds.width() to bounds.height()
        } else {
            val dm = DisplayMetrics()
            @Suppress("DEPRECATION")
            activity.windowManager.defaultDisplay.getMetrics(dm)
            dm.widthPixels to dm.heightPixels
        }
    }
}
```

---

## 五、测试验证

```bash
# 重点测试版本（行为变化最大）：
API 26 (8.0)   — 通知 Channel、自适应图标
API 29 (10)    — 深色模式、手势导航
API 31 (12)    — Material You、动态颜色
API 34 (14)    — 预测性返回动画
API 35 (15)    — 强制 Edge-to-Edge
```

---

## 六、总结

| 维度 | 核心策略 |
|------|---------|
| **分辨率** | dp/sp + ConstraintLayout + 多套资源 + sw 限定符 |
| **版本** | compileSdk 最新 + `Build.VERSION.SDK_INT` 判断 + AndroidX 兼容库 |
| **UI 显示** | AppCompat/Compose + WindowInsetsCompat + Material3 主题 |

一句话：**布局用约束不用固定值，API 用 Jetpack 不用原生旧接口。**