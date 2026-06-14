---
layout: private_post
title:  "图形渲染笔记:纹理、GPU 采样与 SurfaceFlinger 合成"
date:   2026-06-14 00:00:00 +0800
categories: android
tag: SurfaceFlinger
description: 从"纹理是什么"出发,串起 GPU 采样算法、OpenGL/Vulkan 渲染链路、Android 各层采样,以及高斯/Kawase 模糊在 AOSP 16 的真实实现,并落到 SurfaceFlinger 合成。
---

> 主题:从"纹理是什么"出发,串起 GPU 采样算法、OpenGL/Vulkan 渲染链路、Android 各层采样,以及高斯/Kawase 模糊在 AOSP 的真实实现。
> 场景:AOSP 16 / Android 图形栈。

---

## 1. 纹理(Texture)是什么

纹理本质上就是**GPU 能高效采样的一块图像数据**。它和普通"图片"在数据上很像(都是像素数组),但组织方式不同。

| | 普通图片(PNG/JPEG 文件) | 纹理(Texture) |
|---|---|---|
| 存储格式 | 压缩编码,或线性排列的裸像素 | GPU 专用布局,常被 swizzle/tiled(块状重排) |
| 位置 | CPU 内存 / 磁盘 | 显存(或 GPU 可访问内存) |
| 访问方式 | CPU 逐字节读 | GPU 通过采样器(sampler)硬件并行读取,支持过滤、mipmap、wrap |

**结论**:纹理不是神秘的东西,就是"为了让 GPU 采样硬件快速读取而专门组织好的图像"。

### GPU 能直接读 PNG/JPEG 吗?——不能

1. **PNG/JPEG 是压缩编码**,GPU 采样单元是固定功能硬件,只认 RGBA8、ETC2/ASTC 等格式,不会解码。必须先在 CPU(或专用解码硬件)解码成裸像素。
2. **内存布局不对**,线性排列对 GPU 缓存不友好。上传成纹理时驱动会重排成 tiled 布局。

典型流程:

```
PNG 文件 → CPU 解码 → 裸像素(线性) → glTexImage2D/上传 → 驱动重排 → 纹理(显存) → GPU 采样
```

---

## 2. 纹理与 SurfaceFlinger 的关系(纠正常见误解)

**纹理不是 SurfaceFlinger "创建"出来给 GPU 的。** 准确图景:

1. **每个 App 自己画自己的窗口**:通过 BufferQueue 申请一块 **GraphicBuffer**(底层 gralloc 分配的图形内存,常对应 `AHardwareBuffer` / dma-buf),用 GPU(GL/Vulkan)把内容画进去,再 `queueBuffer` 提交。
2. **SurfaceFlinger 是合成器(compositor)**,把所有窗口的 buffer 合成成一帧。两条路径:
   - **GPU 合成(Client Composition)**:把每个 Layer 的 GraphicBuffer **当作纹理绑定**,用 GPU 一层层画到一起。零拷贝直接当纹理采样。
   - **硬件合成(HWC)**:显示控制器(DPU)直接处理各图层,**完全绕过 GPU**,更省电。

> SurfaceFlinger 不"造纹理",而是在 GPU 合成时把各 App 已画好的 GraphicBuffer **当作纹理来采样**。

---

## 3. GPU 采样(纹理过滤)算法

核心问题:纹理被放大/缩小/旋转/透视后,屏幕像素采样坐标常落在纹理"格子之间"。

### 3.1 基础过滤

- **最近邻(Nearest / Point)**:取最近 1 个 texel。最快;放大有马赛克。
- **双线性(Bilinear)**:取周围 4 个 texel 加权平均。放大平滑,最常用;硬件几乎免费。

### 3.2 缩小时的多级渐远(Mipmap)

- **三线性(Trilinear)**:在两个相邻 mip 层各做 bilinear(8 texel)再层间插值,消除 mip 接缝。
- **各向异性(Anisotropic)**:斜视平面覆盖区狭长,沿该方向多次采样(2x~16x),避免过度模糊。

### 3.3 对比

| 算法 | 采样 texel 数 | 用途 | 成本 |
|---|---|---|---|
| Nearest | 1 | 像素风、查表纹理 | 最低 |
| Bilinear | 4(1 层) | 放大、通用 | 低 |
| Trilinear | 8(2 层) | 缩小、消除 mip 接缝 | 中 |
| Anisotropic Nx | 最多 N×8 | 斜视平面 | 高 |

### 3.4 广义采样(shader 手写)

PCF(阴影)、Bicubic(16 texel)、Catmull-Rom / Lanczos(高质量重采样)、TAA(时间性)、`textureGrad`/`textureLod`(手动 LOD)。

### 3.5 关键点

1. 硬件原生免费的只有 bilinear;trilinear/AF 是硬件扩展;bicubic/PCF/Lanczos 多是 shader 里多次 bilinear fetch 组合。
2. 过滤设置在 **sampler** 上,不在纹理本身。GL:`GL_TEXTURE_MIN/MAG_FILTER`、`MAX_ANISOTROPY`;Vulkan:`VkSamplerCreateInfo`。
3. mipmap 是缩小类算法的前提;放大用不到。

---

## 4. OpenGL / Vulkan 的内容交给谁处理

### 4.1 API 调用 → 谁执行渲染

GL/Vulkan 都只是接口规范,本身不画东西:

```
App 调 GL/Vulkan API → 用户态驱动(厂商 .so) → 翻译成 GPU 指令(command buffer)
  → 内核态 GPU 驱动(KMD) → GPU 执行,把结果写进 buffer
```

- **OpenGL ES**:驱动"厚",状态/内存/同步隐式替你做。
- **Vulkan**:驱动"薄",command buffer 录制、内存、同步(semaphore/fence/barrier)由 App 显式负责。
- Android 上很多设备 GL ES 通过 **ANGLE** 跑在 Vulkan 之上。

### 4.2 渲染结果 → 交给谁

无论 GL/Vulkan,**GPU 都把像素写进 GraphicBuffer**,经 BufferQueue 交给 SurfaceFlinger:

```
GPU 渲染完 → queueBuffer → SurfaceFlinger(作为 Layer)→ 合成(GPU 当纹理 / HWC)→ 屏幕(DPU)
```

绑定层:GL 走 **EGL**(`eglSwapBuffers`),Vulkan 走 **WSI / VkSwapchain**(`vkQueuePresentKHR`),底层都连 `ANativeWindow`/BufferQueue。

### 4.3 完整图景

```
┌─────────────── App 进程 ───────────────┐
│  OpenGL ES ──EGL──┐                     │
│                   ├─> GPU 驱动 ─> GPU   │  GPU 把像素写进
│  Vulkan ─WSI(swapchain)┘                │  GraphicBuffer
└────────────────────────┬────────────────┘
                         │ queueBuffer (BufferQueue)
                         ▼
              ┌──────── SurfaceFlinger ────────┐
              │  ① GPU 合成: buffer 当纹理画    │ ← 又用 GL/Vulkan(RenderEngine)
              │  ② HWC 硬件合成: 交给 DPU       │
              └───────────────┬─────────────────┘
                             ▼  显示控制器(DPU)→ 屏幕
```

### 4.4 关键澄清

1. GL/Vulkan 不"生成内容",是指挥 GPU 生成;真正干活的是 GPU + 厂商驱动。
2. 产物统一是 GraphicBuffer,所以 SF 能无差别处理 GL/Vulkan/视频解码的 buffer。
3. SF 自己合成也用 GL/Vulkan(RenderEngine),GPU 可能被用两次:App 渲染 + SF 合成。

---

## 5. 模糊(Blur)是采样算法的一种

模糊**不是纹理属性,而是采样时的算法**:每个像素 = 周围像素的加权平均(故意丢细节)。

```
清晰: 像素 = 自己那个 texel
模糊: 像素 = 周围 N×N texel 的加权平均
```

| 算法 | 做法 | 特点 |
|---|---|---|
| Box Blur | 周围 N×N 平均 | 最简单 |
| Gaussian Blur | 按高斯权重加权 | 自然;可分离成横竖两次 1D |
| Kawase | 多次小核 + 逐步扩大偏移 | 性能好,近似高斯 |
| Dual Kawase | 降采样+升采样 + Kawase | 大半径仍很快 |

**降采样优化**:先缩到 1/4 再模糊,大半径几乎免费。

---

## 6. 高斯模糊算法结构

### 6.1 数学

二维高斯:`G(x,y) = 1/(2πσ²)·e^(-(x²+y²)/(2σ²))`,σ 控制强度。离散成归一化卷积核(权重和=1),半径 `radius ≈ ⌈3σ⌉`(3σ 外权重近 0)。

### 6.2 可分离性(最重要优化)

`G(x,y) = G(x)·G(y)` → 一次 N×N 二维卷积 = **先横 1D + 再竖 1D**,复杂度 O(N²) → O(2N)。

```
原图 ──横向1D高斯──> 临时RT ──纵向1D高斯──> 输出
```

### 6.3 GPU 再加速

1. **linear sampling**:利用 bilinear 硬件,一次 fetch 顶两个 texel。
2. **downsample**:缩小再模糊,等效放大半径。
3. **对称权重**:只存一半。

### 6.4 与 Kawase 对比

| | 高斯 | Kawase |
|---|---|---|
| 原理 | 严格加权卷积 | 多次小核 + 扩大偏移近似 |
| 大半径成本 | tap 多,较贵 | 降采样迭代,可控 |
| 质量 | 最标准 | 视觉几乎一致 |

---

## 7. 可深入的源码位置(AOSP)

- `frameworks/native/services/surfaceflinger/` — SF 主体、Layer、合成、`chooseBlurAlgorithm()`
- `frameworks/native/libs/renderengine/skia/filters/` — RenderEngine、GPU 合成、各 BlurFilter
- `frameworks/native/libs/gui/` — `Surface`、`BufferQueue`
- `frameworks/base/libs/hwui/` — HWUI、RenderEffect、View 侧模糊

---

## 8. AOSP 16 模糊实现细节(已核对源码)

### 8.1 三种实现 + 默认选择

`frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp` 的 `chooseBlurAlgorithm()`:

```cpp
auto algorithm = base::GetProperty(PROPERTY_DEBUG_RENDERENGINE_BLUR_ALGORITHM, "");
if (algorithm == "gaussian")     → GAUSSIAN;
else if (algorithm == "kawase2") → KAWASE_DUAL_FILTER;
else if (algorithm == "kawase")  → KAWASE;
else {                            // 默认
    if (window_blur_kawase2 flag) → KAWASE_DUAL_FILTER;
    return KAWASE;                 // 否则默认 Kawase
}
```

- **默认是 Kawase,不是高斯**(注释:faster & looks close enough)。
- 强制切换:`adb shell setprop debug.renderengine.blur_algorithm gaussian`(或 `kawase` / `kawase2`),改完重启 SurfaceFlinger 生效。
- AOSP 16 新增 `kawase2`(Dual-Kawase),由 `window_blur_kawase2` aconfig flag 控制是否成为默认。

### 8.2 GaussianBlurFilter.cpp(瘦封装,委托 Skia)

```cpp
// kInputScale = 0.25f → 先降采样到 1/4
SkImageInfo scaledInfo = input->imageInfo().makeWH(
        ceil(blurRect.width()  * kInputScale),
        ceil(blurRect.height() * kInputScale));

// radius → sigma:乘 BLUR_SIGMA_SCALE = 0.57735f (= 1/√3)
paint.setImageFilter(SkImageFilters::Blur(
        blurRadius * kInputScale * BLUR_SIGMA_SCALE,
        blurRadius * kInputScale * BLUR_SIGMA_SCALE,
        SkTileMode::kClamp, nullptr));

// 用 bilinear 把原图画进缩小 surface
canvas->drawImageRect(input, ..., SkSamplingOptions{kLinear, kNone}, &paint, kFast);
```

| 源码 | 对应理论 |
|---|---|
| `kInputScale = 0.25f`(BlurFilter.h) | 降采样优化,缩到 1/4 |
| `BLUR_SIGMA_SCALE = 0.57735f` | radius→sigma 换算(对齐 `SkBlurMask::Blur` 高质量模式) |
| `SkFilterMode::kLinear` | 降采样用双线性 |
| `SkImageFilters::Blur` | 真正的可分离两-pass 高斯在 **Skia 内部**完成 |

> Android 的"高斯模糊" = 降采样 + 委托 Skia 的高斯 image filter;RenderEngine 只管缩放和参数换算。

### 8.3 KawaseBlurFilter.cpp(自写 AGSL shader)

```glsl
// 中心 + 4 个对角点,平均(×0.2 = /5)
half4 main(float2 xy) {
    half4 c = child.eval(xy);
    c += child.eval(xy + float2(+off, +off));
    c += child.eval(xy + float2(+off, -off));
    c += child.eval(xy + float2(-off, -off));
    c += child.eval(xy + float2(-off, +off));
    return half4(c.rgb * 0.2, 1.0);
}
```

迭代:`numberOfPasses = min(kMaxPasses, ceil(blurRadius/2))`,每 pass 把 offset 逐步放大,两个 surface **ping-pong**。即"固定 5 点小核 + 逐 pass 放大偏移 + 降采样"逼近高斯。

### 8.4 为什么默认 Kawase

| | GaussianBlurFilter | KawaseBlurFilter |
|---|---|---|
| 实现 | 委托 `SkImageFilters::Blur` | 自写 5-tap AGSL + 多 pass |
| 大半径成本 | sigma 大则 tap 多,较贵 | 多 pass + 降采样,可控 |
| 质量 | 数学标准、最平滑 | 近似,肉眼几乎一致 |
| AOSP 默认 | 否(需 setprop) | **是** |

两者共用基类 `BlurFilter` 的 `kInputScale=0.25` 降采样 和 `drawBlurRegion()`(用 `mMixEffect` 做 crossfade,避免小半径降采样瑕疵)。

---

## 9. View 模糊:两套完全不同的机制

Android 的 "View 模糊" 不是一回事,算法和发生进程都不同:

| | ① `View.setRenderEffect(blur)` | ② `Window.setBackgroundBlurRadius` / FLAG_BLUR_BEHIND |
|---|---|---|
| 模糊对象 | **View 自己(及子 View)的内容** | View/窗口**背后**的内容(毛玻璃) |
| 发生进程 | **App 进程**的 RenderThread(HWUI) | **SurfaceFlinger** 进程 |
| 实现 | Skia `SkImageFilters::Blur` | RenderEngine 的 `BlurFilter` |
| 算法 | **高斯模糊** | **Kawase**(默认)/ Gaussian / Dual-Kawase |

### 9.1 算法:View 自身模糊 = 高斯

`frameworks/base/libs/hwui/jni/RenderEffect.cpp` 的 `createBlurEffect`:

```cpp
sk_sp<SkImageFilter> blurFilter = SkImageFilters::Blur(
        Blur::convertRadiusToSigma(radiusX),
        Blur::convertRadiusToSigma(radiusY),
        static_cast<SkTileMode>(edgeTreatment),
        sk_ref_sp(inputImageFilter), nullptr);
```

radius→sigma(`hwui/utils/Blur.cpp`),和 SF 那套用同一常数 0.57735:

```cpp
static const float BLUR_SIGMA_SCALE = 0.57735f;   // 1/√3
float Blur::convertRadiusToSigma(float radius) {
    return radius > 0 ? BLUR_SIGMA_SCALE * radius + 0.5f : 0.0f;
}
```

> View 自身模糊 = Skia `SkImageFilters::Blur` 高斯(小 sigma 真高斯,大 sigma 退化成多次 box 近似,Ganesh GPU 后端执行)。**这里没有 Kawase**,Kawase 只在 SurfaceFlinger 那条路。`shader/BlurShader.cpp` 的 shader 版本也走同一个 `SkImageFilters::Blur`。

### 9.2 是"先在 CPU 糊好再传 GPU"吗?——不是

**模糊全程在 GPU 完成,不存在 CPU 预糊。**

**情况 ①(setRenderEffect,模糊 View 自己内容):**
```
1. View 绘制记录成 RenderNode(DisplayList,只是"画什么"指令)
2. RenderNode 挂一个高斯 SkImageFilter
3. RenderThread 把 RenderNode 渲染到离屏纹理
4. GPU 对离屏纹理跑高斯 → 模糊结果
5. 合成进最终窗口 buffer
```
输入是 View **自己**渲染的内容,不是它的背景。全程 GPU。

**情况 ②(窗口背景模糊,你说的"背景"更像这个):**
```
1. SurfaceFlinger 先把"这个窗口下面"的图层合成 → 背景纹理(GPU 里)
2. 对背景纹理在 GPU 上跑 Kawase
3. 模糊结果作为窗口背景,再叠窗口本身内容
```
"背景被模糊后再合成"是对的,但**模糊在 SurfaceFlinger 的 GPU 合成阶段**;App 只传一个 `backgroundBlurRadius` 参数,不在 App 里先糊。

**为何没有 CPU 预糊**:模糊是大量采样+加权平均,正是 GPU 强项;数据本就是 GPU 可读纹理,原地处理零拷贝;搬回 CPU 糊完再传回要两次拷贝。两条路都是 **GPU in → GPU blur → GPU out**。

### 9.3 一句话

1. View 自身模糊(`setRenderEffect`)用**高斯**(HWUI/Skia,App 进程);窗口背景模糊用 **Kawase**(SurfaceFlinger)。两者 radius→sigma 都用 0.57735。
2. **不是 CPU 先糊再传**:`setRenderEffect` 是 GPU 渲到离屏纹理后做高斯;背景模糊是 SF 在 GPU 合成时对下层背景纹理做 Kawase——模糊全在 GPU。

源码:`frameworks/base/libs/hwui/jni/RenderEffect.cpp`、`shader/BlurShader.cpp`、`utils/Blur.cpp`(View 侧);`frameworks/native/libs/renderengine/skia/filters/`(SF 侧)。

---

## 10. 总览速记

- **纹理** = 为 GPU 采样硬件组织好的图像;GPU 读不了 PNG/JPEG,要先解码上传。
- **SurfaceFlinger** 不造纹理,GPU 合成时把 GraphicBuffer 当纹理;能用 HWC 时连 GPU 都不用。
- **采样算法**:Nearest/Bilinear(硬件免费)→ Trilinear/各向异性(扩展)→ Bicubic/PCF/Lanczos/TAA(shader)。参数在 sampler 上。
- **GL/Vulkan** 只是 API,产物统一 GraphicBuffer,经 EGL/WSI → BufferQueue → SF。
- **模糊**是"采样一张纹理→输出另一张"的 shader 算法。AOSP 16 背景模糊默认 **Kawase**(可 setprop 切高斯/kawase2);高斯走 Skia `SkImageFilters::Blur`,Kawase 是自写 5-tap AGSL。
- **View 模糊两套**:`setRenderEffect`=高斯(HWUI,App 进程);窗口背景模糊=Kawase(SF)。都在 GPU 做,无 CPU 预糊。radius→sigma 常数 0.57735 两处通用。
