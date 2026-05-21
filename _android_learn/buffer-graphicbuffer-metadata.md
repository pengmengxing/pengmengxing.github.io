---
layout: private_post
title: "Buffer、GraphicBuffer 与元数据的本质区别"
---

## 1. Buffer 和 GraphicBuffer 分别是什么

**Buffer 是泛称，GraphicBuffer 是具体实现。**

```
"buffer" （泛称，指图形缓冲区）
    │
    └── GraphicBuffer （具体的 C++ 类，定义在 GraphicBuffer.h）
            │
            ├── 描述信息:  width, height, stride, format, usage
            │
            └── handle (buffer_handle_t)
                  │
                  └── 指向 gralloc 分配的物理内存（通过 ION/dmabuf）
                        │
                        └── 这块内存就是实际存放像素的地方
```

### GraphicBuffer 核心字段

源码位置：`frameworks/native/libs/ui/include/ui/GraphicBuffer.h`

| 字段 | 类型 | 含义 |
|------|------|------|
| `width` | int32_t | 宽度（像素） |
| `height` | int32_t | 高度（像素） |
| `stride` | int32_t | 行跨度（字节对齐后的每行像素数） |
| `format` | PixelFormat | 像素格式（如 RGBA_8888） |
| `usage` | uint64_t | 用途标志（GPU 渲染 / CPU 读写 / 视频解码等） |
| `handle` | buffer_handle_t | **指向真正物理内存的句柄**（来自 gralloc HAL） |
| `mId` | uint64_t | 唯一标识（PID << 32 | 原子计数器） |
| `mOwner` | uint8_t | 所有权标记（ownNone / ownHandle / ownData） |

### 类比理解

- **GraphicBuffer** = 一张有固定规格的画纸（宽高、材质已定）
- **handle** = 画纸在仓库里的编号（指向真正的物理内存）
- **像素数据** = 画纸上的颜料（App 渲染写入的内容）

### 物理内存分配流程

```
GraphicBuffer 构造
  → GraphicBufferAllocator::allocate()
    → gralloc HAL（硬件抽象层）
      → ION / dmabuf 分配物理内存
        → 返回 buffer_handle_t（包含 fd 和元数据）
```

App 和 SurfaceFlinger 通过 `handle` 映射同一块物理内存，实现**零拷贝**共享。

---

## 2. GraphicBuffer 在 BufferQueue 中的生命周期

源码位置：`frameworks/native/libs/gui/include/gui/BufferSlot.h`

### BufferSlot 结构

每个 slot 持有一个 GraphicBuffer 和状态信息：

```cpp
struct BufferSlot {
    sp<GraphicBuffer> mGraphicBuffer;  // 实际的 buffer 对象
    BufferState mBufferState;          // 当前状态
    uint64_t mFrameNumber;             // 帧序号
    sp<Fence> mFence;                  // 同步栅栏
    bool mNeedsReallocation;           // 是否需要重新分配
};
```

### Buffer 状态机

```
FREE → DEQUEUED → QUEUED → ACQUIRED → FREE（循环复用）
(空闲)  (生产者持有)  (已提交)  (消费者持有)  (释放回来)

                         cancelBuffer()
FREE ← ← ← ← ← ← ← ← DEQUEUED
```

### BufferQueue 循环复用示意

```
BufferQueue（通常 2~3 个 slot，最大 64 个）：
┌──────────┬──────────┬──────────┐
│  slot 0  │  slot 1  │  slot 2  │
│ GraphicBuf│ GraphicBuf│ GraphicBuf│  ← 首次 dequeue 时分配，之后复用
│ ACQUIRED │ FREE     │ DEQUEUED │
└──────────┴──────────┴──────────┘
  ↑ SF 在读    ↑ 空闲待用   ↑ App 在写
```

### 完整的一帧 Buffer 生命周期

```
生产者（App）端：
  1. dequeueBuffer(slot) → 获取空闲 slot + releaseFence
  2. requestBuffer(slot) → 获取 GraphicBuffer 指针（首次）
  3. GraphicBufferMapper::lock() → 映射到 CPU 虚拟地址
  4. [渲染：写入像素数据]
  5. GraphicBufferMapper::unlock() → 解除映射
  6. queueBuffer(slot, QueueBufferInput{fence, timestamp, crop, ...})
     → 提交 buffer + 附带元信息

消费者（SurfaceFlinger）端：
  7. acquireBuffer(slot) → 获取已提交的 buffer + acquireFence
  8. [合成：读取像素，合成到帧]
  9. releaseBuffer(slot, releaseFence) → 释放 buffer
  10. Buffer → FREE（可被生产者再次 dequeue）
```

### queueBuffer 时附带的元信息

源码位置：`frameworks/native/libs/gui/include/gui/IGraphicBufferProducer.h`

```cpp
struct QueueBufferInput {
    int64_t timestamp;          // 时间戳
    bool isAutoTimestamp;       // 是否自动时间戳
    android_dataspace dataSpace; // 色彩空间
    Rect crop;                  // 裁剪区域
    int scalingMode;            // 缩放模式
    uint32_t transform;         // 变换标志
    sp<Fence> fence;            // acquire fence（生产者完成渲染的信号）
    Region surfaceDamage;       // 脏区域（只变化了哪些区域）
    HdrMetadata hdrMetadata;    // HDR 元数据
};
```

> 注意：这里的元信息是**跟随 buffer 一起提交**的，描述的是"这个 buffer 的属性"，和 SurfaceControl 的元数据是不同层面的东西。

---

## 3. 元数据（Metadata）是什么

**元数据 = SurfaceFlinger 中每个 Layer 的"显示属性"，描述一个图层"怎么摆放"，而不是"画了什么"。**

### 元数据的载体：layer_state_t

源码位置：`frameworks/native/libs/gui/include/gui/LayerState.h`（lines 224-901）

这是 `SurfaceControl.Transaction` 实际传输给 SurfaceFlinger 的结构体：

| 类别 | 具体字段 | 说明 |
|------|---------|------|
| **位置** | `x, y` | 图层在屏幕上的坐标 |
| **变换** | `matrix22_t matrix` (dsdx, dtdx, dtdy, dsdy) | 2x2 变换矩阵（缩放/旋转/倾斜） |
| **裁剪** | `crop`, `bufferCrop`, `destinationFrame` | 可见区域 |
| **层级** | `z` (z-order), `layerStack` | 前后叠放顺序、所属显示器 |
| **透明度** | `alpha`, `color (half4)` | 整层透明度和颜色 |
| **圆角** | `cornerRadii` | 四角圆角半径 |
| **模糊** | `backgroundBlurRadius`, `blurRegions` | 背景模糊效果 |
| **阴影** | `shadowRadius`, `boxShadowSettings` | 阴影效果 |
| **可见性** | `flags` (hidden / opaque / secure) | 是否隐藏、是否不透明 |
| **帧率** | `frameRate`, `frameRateCategory` | 期望帧率 |
| **输入** | `windowInfo` | 触摸区域配置 |
| **颜色** | `colorTransform (mat4)`, `dataspace` | 色彩变换矩阵、色彩空间 |

### 状态变更标志：what 字段

`layer_state_t` 中的 `uint64_t what` 是位掩码，标识哪些字段被修改了：

```cpp
ePositionChanged        // 位置变了
eAlphaChanged           // 透明度变了
eMatrixChanged          // 变换矩阵变了
eCropChanged            // 裁剪区域变了
eBufferChanged          // buffer 变了
eCornerRadiusChanged    // 圆角变了
eDataspaceChanged       // 色彩空间变了
// ... 等数十种标志
```

### 元数据 vs Buffer 内容的完整对比

```
一个 Layer 在 SurfaceFlinger 中：

┌─────────────────────────────────────┐
│          元数据（Metadata）           │ ← SurfaceControl.Transaction 控制
│                                     │
│  position=(100,200)                 │    "这个图层放在 (100,200)，
│  matrix=[0.8,0,0,0.8]              │     缩放到 0.8 倍，
│  alpha=0.5                          │     透明度 50%，
│  crop=(0,0,540,1200)               │     裁剪成这个矩形"
│  z=5, cornerRadius=20              │
│                                     │    → 告诉 SurfaceFlinger 怎么摆
├─────────────────────────────────────┤
│        buffer 内容（像素）            │ ← Surface.queueBuffer() 提交
│                                     │
│  GraphicBuffer → 物理内存 → 像素     │    "这张图上画了一个按钮、
│                                     │     一段文字、一张图片..."
│                                     │
│                                     │    → 告诉 SurfaceFlinger 画了什么
└─────────────────────────────────────┘
```

---

## 4. 为什么 SurfaceControl 不涉及 buffer 操作

### 原因一：职责不同，通道不同

```
App 进程：

  Surface ──→ BufferQueue ──→ SurfaceFlinger
  (写像素)    (共享内存，零拷贝)  (拿 buffer 合成)
              数据量: 数 MB 像素    传输方式: 共享内存

  SurfaceControl.Transaction ──→ SurfaceFlinger
  (改属性)         (Binder，传几个数值)
                   数据量: 几十字节     传输方式: Binder IPC
```

如果混在一起，每次改个位置就得带上整个 buffer，完全没必要。

### 原因二：更新频率不同，独立变化

| | buffer 更新 | 元数据更新 |
|---|---|---|
| **触发者** | App 主动渲染（draw） | WMS / Shell / App 调 Transaction |
| **频率** | 有内容变化才更新 | 有属性变化才更新 |

两者是**独立变化**的：
- 静态页面可以被 Window 动画移动（只改元数据，不改 buffer）
- 固定位置的视频可以每帧刷新（只改 buffer，不改元数据）
- 两者也可以同时变化（App 边滚动边被系统动画缩小）

### 原因三：性能差距巨大

```
假设 Window 动画走 buffer（重绘方案）：
  每帧: App 重绘整个窗口 → CPU measure/layout → GPU 渲染
        → 跨进程传 buffer → SurfaceFlinger 合成
  开销: CPU + GPU + 内存带宽，数 MB 数据

假设 Window 动画走 SurfaceControl（实际方案）：
  每帧: Transaction.setMatrix(float x4) → Binder 传几个 float
        → SurfaceFlinger 直接对已有 buffer 做矩阵变换 → 合成
  开销: 仅几十字节的 Binder 调用
```

**SurfaceFlinger 自带 GPU 能力**，可以对已有 buffer 做平移/缩放/旋转/透明度变换，不需要 App 重新渲染像素。

### 原因四：解耦带来的架构灵活性

```
分离后可以自由组合：

场景 1: View 动画
  → 只更新 buffer（每帧新像素），元数据不变

场景 2: Window 动画
  → 只更新元数据（位置/缩放/透明度），buffer 不变

场景 3: 视频播放 + 画中画拖动
  → buffer 和元数据同时独立更新，互不干扰

场景 4: 静态壁纸
  → buffer 和元数据都不变，SurfaceFlinger 直接复用上一帧
```

---

## 5. 三层概念的完整关系图

```
┌─────────────────────────────────────────────────────────┐
│                        App 进程                          │
│                                                         │
│   Surface                    SurfaceControl              │
│   ├─ lockCanvas()            ├─ Transaction {            │
│   ├─ queueBuffer()          │    setPosition()          │
│   └─ 操作: GraphicBuffer     │    setMatrix()            │
│       (写像素)               │    setAlpha()             │
│                              │    setCrop()              │
│                              │    apply() ──────────┐    │
│                              └───────────────────── │ ── │
└──────────┬──────────────────────────────────────── │ ───┘
           │                                         │
           │ IGraphicBufferProducer                  │ ISurfaceComposer
           │ (共享内存，零拷贝)                        │ setTransactionState()
           │ 数据量: 数 MB                            │ (Binder，几十字节)
           ▼                                         ▼
┌─────────────────────────────────────────────────────────┐
│                   SurfaceFlinger 进程                     │
│                                                         │
│   Layer                                                 │
│   ├── buffer 内容: GraphicBuffer → 像素数据              │
│   │   (来自 Surface.queueBuffer)                        │
│   │                                                     │
│   └── 元数据: position, matrix, alpha, crop, z, ...     │
│       (来自 SurfaceControl.Transaction)                  │
│                                                         │
│   合成: 用元数据决定怎么摆放 buffer → 输出到屏幕           │
└─────────────────────────────────────────────────────────┘
```

---

## 总结

| 概念 | 是什么 | 谁控制 | 传输方式 |
|------|--------|--------|---------|
| **Buffer** | 泛称，指图形缓冲区 | — | — |
| **GraphicBuffer** | Buffer 的具体实现，持有物理内存句柄和像素数据 | Surface (dequeue/queue) | 共享内存（零拷贝） |
| **元数据** | Layer 的显示属性（位置/变换/透明度/裁剪/层级等） | SurfaceControl.Transaction | Binder IPC（几十字节） |

> **SurfaceControl 不操作 buffer，是因为"怎么摆"和"画什么"本质上是两件独立的事，分开管理才能各自达到最优性能。** 这种分离让 Window 动画只需传几个数值就能驱动流畅的 60fps 动画，而不需要 App 每帧重绘整个窗口。

---

## 源码路径索引

| 组件 | 文件路径 |
|------|---------|
| GraphicBuffer | `frameworks/native/libs/ui/include/ui/GraphicBuffer.h` |
| GraphicBufferAllocator | `frameworks/native/libs/ui/include/ui/GraphicBufferAllocator.h` |
| GraphicBufferMapper | `frameworks/native/libs/ui/include/ui/GraphicBufferMapper.h` |
| BufferSlot | `frameworks/native/libs/gui/include/gui/BufferSlot.h` |
| BufferQueue | `frameworks/native/libs/gui/BufferQueue.cpp` |
| BufferQueueProducer | `frameworks/native/libs/gui/BufferQueueProducer.cpp` |
| IGraphicBufferProducer | `frameworks/native/libs/gui/include/gui/IGraphicBufferProducer.h` |
| layer_state_t | `frameworks/native/libs/gui/include/gui/LayerState.h` |
| Layer DrawingState | `frameworks/native/services/surfaceflinger/Layer.h` |
| SurfaceFlinger setTransactionState | `frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp` |