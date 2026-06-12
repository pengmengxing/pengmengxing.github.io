---
layout: private_post
title:  "图形渲染笔记:纹理、GPU 采样与 SurfaceFlinger 合成"
date:   2026-06-12 00:00:00 +0800
categories: android
tag: SurfaceFlinger
description: 从"纹理是什么"出发,串起 GPU 采样算法、OpenGL/Vulkan 渲染链路,以及 Android 背景模糊的实现,并落到 SurfaceFlinger 合成。
---

> 主题:从"纹理是什么"出发,串起 GPU 采样算法、OpenGL/Vulkan 渲染链路,以及 Android 背景模糊的实现。
> 场景:AOSP / Android 图形栈。

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

原因:
1. **PNG/JPEG 是压缩编码**,GPU 采样单元是固定功能硬件,只认 RGBA8、ETC2/ASTC 等格式,不会解码 PNG/JPEG。必须先在 CPU(或专用解码硬件)解码成裸像素。
2. **内存布局不对**,线性排列对 GPU 缓存不友好。上传成纹理时驱动会重排成 tiled 布局,提高采样相邻像素的缓存命中率。

典型流程:

```
PNG 文件 → CPU 解码 → 裸像素(线性) → glTexImage2D/上传 → 驱动重排 → 纹理(显存) → GPU 采样
```

---

## 2. 纹理与 SurfaceFlinger 的关系(纠正常见误解)

**纹理不是 SurfaceFlinger "创建"出来给 GPU 的。** 准确图景:

1. **每个 App 自己画自己的窗口**:通过 BufferQueue 申请一块 **GraphicBuffer**(底层 gralloc 分配的图形内存,常对应 `AHardwareBuffer` / dma-buf),用 GPU(GL/Vulkan)把内容画进去,再 `queueBuffer` 提交。

2. **SurfaceFlinger 是合成器(compositor)**,负责把屏幕上所有窗口(状态栏、App、导航栏……)的 buffer 合成成一帧。两条路径:
   - **GPU 合成(Client Composition)**:把每个 Layer 的 GraphicBuffer **当作纹理绑定**,用 GPU 一层层画到一起。GraphicBuffer 从分配起就是 GPU 可读格式,可零拷贝直接当纹理采样,省掉"解码+重排上传"。
   - **硬件合成(HWC / Hardware Composer)**:显示控制器(DPU)直接处理各图层,**完全绕过 GPU**,更省电。

> 精确说法:SurfaceFlinger 不"造纹理",而是在 GPU 合成时把各 App 已画好的 GraphicBuffer **当作纹理来采样**。

---

## 3. GPU 采样(纹理过滤)算法

核心问题:纹理被放大、缩小、旋转、透视后,一个屏幕像素的采样坐标常落在纹理"格子之间"。采样算法决定"取哪些 texel、怎么混合"。

### 3.1 基础过滤(同一层纹理内)

- **最近邻(Nearest / Point)**:取最近的 1 个 texel。最快;放大有马赛克块,像素风游戏故意用。
- **双线性(Bilinear)**:取周围 4 个 texel 按距离加权平均(先水平后垂直)。放大平滑,最常用;硬件几乎免费。

### 3.2 缩小时的多级渐远(Mipmap)

缩小时一个屏幕像素覆盖很多 texel,只取少数会走样/闪烁。预生成逐级减半的小图链(mipmap),按需选层级。

- **三线性(Trilinear)**:在两个相邻 mip 层各做一次 bilinear(共 8 texel),再在层间插值,消除 mip 层突变接缝。成本约 bilinear 的 2 倍。
- **各向异性过滤(Anisotropic, AF)**:斜视平面(地面、跑道)上,像素在纹理的覆盖区是狭长的;trilinear 会因选过高 mip 层而过度模糊。AF 沿狭长方向多次采样(2x/4x/8x/16x)再平均。最贵但效果最明显。

### 3.3 质量与采样数对比

| 算法 | 采样 texel 数 | 用途 | 成本 |
|---|---|---|---|
| Nearest | 1 | 像素风、查表纹理 | 最低 |
| Bilinear | 4(1 层) | 放大、通用 | 低 |
| Trilinear | 8(2 层) | 缩小、消除 mip 接缝 | 中 |
| Anisotropic Nx | 最多 N×8 | 斜视平面 | 高 |

### 3.4 广义采样(shader 手写)

- **PCF(Percentage Closer Filtering)**:阴影贴图软阴影边缘。
- **双三次(Bicubic)**:16 texel 三次曲线插值,图像缩放/上采样,硬件不直接支持。
- **Catmull-Rom / Lanczos**:高质量重采样(TAA、超分、DLSS 类的空间滤波)。
- **时间性采样(Temporal / TAA)**:跨帧用历史样本 + 子像素抖动,等效提升采样率抗锯齿。
- **梯度/导数控制 LOD**:`textureGrad` / `textureLod`,手动给 ddx/ddy 控制选哪级 mip。

### 3.5 关键点

1. **硬件原生免费的只有 bilinear**;trilinear、AF 是硬件扩展;bicubic/PCF/Lanczos 多是 shader 里多次 bilinear fetch 组合。
2. **过滤设置在采样器(sampler)上,不在纹理本身**。OpenGL:`GL_TEXTURE_MIN/MAG_FILTER`、`GL_TEXTURE_MAX_ANISOTROPY`;Vulkan:`VkSamplerCreateInfo`。
3. **mipmap 是缩小类算法(trilinear/AF)的前提**;放大用不到 mipmap。
4. SurfaceFlinger 合成时图层多为 1:1 贴屏,一般 **bilinear 就够**;缩放/旋转/投射才需更高级过滤。相关代码:`frameworks/native/libs/renderengine/`。

---

## 4. OpenGL / Vulkan 的内容交给谁处理

分两层:**API 调用谁执行**,以及**渲染结果交给谁**。

### 4.1 API 调用 → 谁执行渲染

OpenGL ES 和 Vulkan 都只是**接口规范**,本身不画东西。调用链:

```
App 调用 GL/Vulkan API
  → 用户态驱动(GPU 厂商实现的 .so,如 Mali/Adreno)
  → 翻译成 GPU 指令(command buffer)
  → 内核态 GPU 驱动(KMD)提交给 GPU 硬件
  → GPU 执行,把结果写进一块 buffer
```

"翻译"厚薄不同:
- **OpenGL ES**:驱动"厚"。状态机、内存、指令编排、同步都隐式替你做。
- **Vulkan**:驱动"薄"。command buffer 录制、内存分配、同步(semaphore/fence/barrier)都由 App 显式负责,开销更低更可控,代码更复杂。

> Android 上很多设备的 OpenGL ES 通过 **ANGLE** 跑在 Vulkan 之上(GL 调用翻译成 Vulkan)。

### 4.2 渲染结果 → 交给谁

无论 GL 还是 Vulkan,**GPU 最终都把像素写进一块 GraphicBuffer**,再经 BufferQueue 交给 SurfaceFlinger:

```
GPU 渲染完 → queueBuffer 提交
  → SurfaceFlinger 拿到 buffer(作为一个 Layer)
  → 合成(GPU 合成当纹理 / 或 HWC 硬件合成)
  → 输出到屏幕(DPU 扫描)
```

中间的窗口系统绑定层:
- **OpenGL ES** 通过 **EGL**(`eglSwapBuffers`)提交;`ANativeWindow`(底层 `Surface`,连 BufferQueue)是桥梁。
- **Vulkan** 通过 **WSI 扩展**,即 `VkSwapchain`(`vkQueuePresentKHR`),底层一样连 `ANativeWindow` / BufferQueue。

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
              │  收集所有 Layer 的 buffer        │
              │  ① GPU 合成: 把 buffer 当纹理画  │ ← 又用 GL/Vulkan(RenderEngine)
              │  ② HWC 硬件合成: 交给显示控制器  │
              └───────────────┬─────────────────┘
                             ▼
                        显示控制器(DPU)→ 屏幕
```

### 4.4 关键澄清

1. GL/Vulkan 不"生成内容"本身,是**指挥 GPU 生成**;真正干活的是 GPU 硬件 + 厂商驱动。
2. 产物统一是 **GraphicBuffer**,所以 SurfaceFlinger 能无差别处理 GL App、Vulkan App、视频解码输出的 buffer。
3. SurfaceFlinger 自己合成时也用 GL/Vulkan(`RenderEngine`),所以 GPU 在链上可能被用两次:App 渲染 + SF 合成(走 GPU 合成时)。

---

## 5. 模糊(Blur)是在纹理上做的吗

模糊**不是纹理本身的属性,而是采样时的一种主动算法**——本质是"多次采样再加权平均",只不过目的是故意丢失细节。

### 5.1 模糊 = 采样/着色算法

```
清晰: 像素 = 自己那个 texel
模糊: 像素 = 周围 N×N 个 texel 的加权平均(如高斯权重)
```

"模糊在哪做"= **在采样输入纹理、输出到目标 buffer 的那个 shader 里**。输入一张纹理,输出另一张(模糊后的)纹理/buffer。

### 5.2 常见模糊算法

| 算法 | 做法 | 特点 |
|---|---|---|
| Box Blur | 周围 N×N 平均 | 最简单,质量一般 |
| Gaussian Blur | 按高斯权重加权 | 自然,最常用;可分离成横竖两次 1D 省算力 |
| Kawase Blur | 多次小半径采样逐步扩大 | 移动端/SF 常用,性能好,接近高斯 |
| Dual Kawase | 降采样+升采样配合 Kawase | 大半径仍很快,Android 背景模糊用这类 |

**关键技巧——降采样(downsample)**:先把图缩小(如 1/4),在小图上模糊再放大。缩小本身 + bilinear 就带模糊,所以大半径模糊几乎免费,是实时模糊的核心优化。

### 5.3 Android / SurfaceFlinger 背景模糊在哪做

Android 12+ 的背景模糊(通知栏下拉、对话框后毛玻璃)**不在 App 纹理上做,而在合成阶段**:

```
1. SurfaceFlinger 先把"模糊层下面"的所有图层合成出来 → 一张背景纹理
2. 对这张背景纹理跑模糊 shader(RenderEngine 里,Kawase/Dual-Kawase)
3. 把模糊结果当作该窗口背景,再叠上窗口本身内容
```

代码:`frameworks/native/libs/renderengine/` 下的 **`BlurFilter` / `KawaseBlurFilter`**(新版还有 `GaussianBlurFilter`)。

> 背景模糊是 SurfaceFlinger 在 GPU 合成时,对"已合成好的下层结果(一张纹理)"做的后处理。

这也解释了为何**背景模糊必须走 GPU 合成**,不能纯 HWC:显示控制器只会简单叠加,不跑模糊 shader。开了背景模糊,这部分强制用 GPU。

### 5.4 "在纹理上做"对不对

- **App 内部模糊**(View 模糊、游戏景深/泛光):App 把场景渲染成离屏纹理(render target),再用模糊 shader 采样它输出到另一张纹理。**输入输出都是纹理,模糊发生在两者之间的 shader**。
- **窗口背景模糊**:输入"纹理"是 SurfaceFlinger 合成出的下层画面,处理者是 SF 的 RenderEngine。

---

## 6. 总览速记

- **纹理** = 为 GPU 采样硬件组织好的图像数据(显存里、特殊布局)。
- **GPU 读不了 PNG/JPEG**,必须先解码成裸像素再上传成纹理。
- **SurfaceFlinger** 不造纹理,在 GPU 合成时把各 App 提交的 GraphicBuffer 直接绑为纹理;能用 HWC 时连 GPU 都不用。
- **采样算法**:Nearest / Bilinear(硬件免费)→ Trilinear / 各向异性(硬件扩展)→ Bicubic / PCF / Lanczos / TAA(shader 手写)。过滤参数在 sampler 上。
- **GL/Vulkan** 只是 API,真正渲染靠 GPU+驱动;产物统一是 GraphicBuffer,经 EGL/WSI → BufferQueue → SurfaceFlinger。
- **模糊**是"采样一张纹理 → 输出另一张"的 shader 算法(box/高斯/Kawase);Android 窗口背景模糊由 SF 的 RenderEngine 在 GPU 合成阶段对下层画面做,典型 Kawase/Dual-Kawase + 降采样。

---

## 7. 可深入的源码位置(AOSP)

- `frameworks/native/services/surfaceflinger/` — SurfaceFlinger 主体、Layer、合成逻辑
- `frameworks/native/libs/renderengine/` — RenderEngine、GPU 合成、`BlurFilter` / `KawaseBlurFilter`
- `frameworks/native/libs/gui/` — `Surface`、`BufferQueue`(`Surface::queueBuffer` → `BufferQueueProducer`)
