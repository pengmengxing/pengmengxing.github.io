---
layout: private_post
title: "View 动画与 Window 动画的本质区别"
---

## 前置知识：三大核心组件

| 组件 | 一句话定位 | 类比 |
|------|-----------|------|
| **Surface** | 图形缓冲区的管道，内部流动着 GraphicBuffer | 画纸 |
| **SurfaceControl** | 图层的遥控器，控制摆放方式（位置/缩放/透明度等） | 相框 |
| **SurfaceFlinger** | 唯一的合成器，所有画面必经此路上屏 | 摄影师（把所有相框拍成一张照片） |

**三者关系**：
```
App 通过 Surface        写像素（画什么）
App 通过 SurfaceControl  控制属性（怎么摆）
SurfaceFlinger           合成一切（拍合照上屏）
```

### 关键对象的生命周期

```
Window 创建 ──────────────────────────────────── Window 销毁
  │                                                │
  ├─ Surface（1个，不变）                            │
  │    └─ BufferQueue（1个，不变）                   │
  │         ├─ GraphicBuffer slot0 ──┐              │
  │         ├─ GraphicBuffer slot1   ├─ 循环复用     │
  │         └─ GraphicBuffer slot2 ──┘              │
  │              └─ 像素内容：每帧更新覆盖            │
  │                                                │
  └─ SurfaceControl（1个，不变，但属性可被 Transaction 修改）
```

> Surface 是管道，GraphicBuffer 是管道里流动的水，像素是水的颜色。
> View 动画每帧换新水，管道不变；Window 动画水不变，只是挪动管道的位置。

---

## 问题 1：View 的展开动画涉及 SurfaceControl 吗？

### 结论：不涉及

View 动画（无论是 `android.view.animation.Animation` 还是属性动画 `ObjectAnimator`）始终工作在 **View 树内部**，不会触及 SurfaceControl。

### 源码依据

**硬件加速路径** — 动画 matrix/alpha 设置到 RenderNode 上：

```java
// View.java line 26900
renderNode.setAnimationMatrix(transformToApply.getMatrix());
renderNode.setAlpha(alpha * getAlpha() * getTransitionAlpha());
```

**软件渲染路径** — 动画 matrix/alpha 直接应用到 Canvas 上：

```java
canvas.concat(transformToApply.getMatrix());
```

两条路径最终都是 **在同一个 Surface 的 GraphicBuffer 上重新绘制像素**，SurfaceControl 完全没有参与。

### 每帧流程

```
Choreographer VSync
  → ViewRootImpl.performTraversals()
    → measure → layout → draw
      → View.updateDisplayListIfDirty()
        → RenderNode 记录绘制指令（含动画 matrix/alpha）
      → ThreadedRenderer.draw()
        → RenderThread 执行指令 → 写入 GraphicBuffer
  → queueBuffer() 提交给 BufferQueue
  → SurfaceFlinger 拿到新 buffer → 合成上屏
```

---

## 问题 2：View 展开过程中每帧绘制的作用对象是谁？

### 结论：作用对象是 Surface（准确说是 Surface 内的 GraphicBuffer）

View 动画每帧做的事：

```
从 BufferQueue 借一个空闲的 GraphicBuffer
  → 往里画新像素（通过 Canvas 或 HWUI RenderThread）
  → 还回去给 SurfaceFlinger 合成
```

### 关键区分

| 对象 | 是否每帧创建 | 说明 |
|------|-------------|------|
| **Surface** | 否，Window 生命周期内始终同一个 | Surface 是管道，不会每帧重建 |
| **GraphicBuffer** | 否，首次 dequeue 时分配，之后循环复用 | 通常 2~3 个 slot 轮转 |
| **像素内容** | 是，每帧重新绘制 | 这才是真正每帧变化的东西 |

### 两条渲染路径

```
ViewRootImpl.draw()
  ├─ 硬件加速路径（主流）:
  │   View.updateDisplayListIfDirty()
  │     → RenderNode.beginRecording() 获取 RecordingCanvas
  │     → View.draw(canvas) 记录绘制指令到 DisplayList
  │     → RenderNode.endRecording()
  │   ThreadedRenderer.draw()
  │     → RenderThread 执行 DisplayList → 写入 Surface 的 GraphicBuffer
  │
  └─ 软件渲染路径:
      Surface.lockCanvas(dirty)       ← 获取 GraphicBuffer
      View.draw(canvas)               ← 直接写入 buffer 像素
      Surface.unlockCanvasAndPost()   ← 提交 buffer 到 BufferQueue
```

---

## 问题 3：对于 Window，动画的对象是针对 SurfaceControl 的吗？

### 结论：是的，Window 动画的对象就是 SurfaceControl

Window 动画采用 **"Leash（缰绳）"模式**，动画期间 App 不重绘，SurfaceFlinger 直接对图层做几何变换。

### 源码依据

**第一步：创建动画 Leash**（`SurfaceAnimator.java:430`）

```java
// 创建一个新的 SurfaceControl 作为 leash
SurfaceControl leash = createAnimationLeash();
t.setWindowCrop(leash, width, height);
t.setPosition(leash, x, y);
t.show(leash);
t.reparent(surface, leash);  // 窗口的真实 Surface 挂到 leash 下
```

**第二步：每帧操作 Leash 的 SurfaceControl 属性**（`WindowAnimationSpec.java:132`）

```java
public void apply(Transaction t, SurfaceControl leash, long currentPlayTime) {
    mAnimation.getTransformation(currentPlayTime, tmp.transformation);
    t.setMatrix(leash, tmp.transformation.getMatrix(), tmp.floats);  // 变换
    t.setAlpha(leash, tmp.transformation.getAlpha());                // 透明度
    t.setWindowCrop(leash, clipRect);                                // 裁剪
    t.setCornerRadius(leash, mWindowCornerRadius);                   // 圆角
}
```

**第三步：Transaction.apply() 提交到 SurfaceFlinger**

### Leash 模式图解

```
动画前:                          动画中:
                                 ┌─ Leash (SurfaceControl) ──────┐
┌─ Window Surface ──┐            │  setMatrix/setAlpha/setCrop    │
│  App 绘制的内容    │   ──→     │  ┌─ Window Surface ──────────┐ │
└───────────────────┘            │  │  App 内容不变，不重绘      │ │
                                 │  └───────────────────────────┘ │
                                 └────────────────────────────────┘
                                   ↑ 每帧只改 Leash 的属性
```

### 两种路径都走 SurfaceControl

| 路径 | 入口 | 最终操作 |
|------|------|---------|
| 传统路径 | `WindowStateAnimator.applyAnimationLocked()` → `SurfaceAnimator` → `SurfaceAnimationRunner` | `Transaction.setMatrix/setAlpha(leash)` |
| Shell Transitions（现代路径） | `Transition.java` 构建 `TransitionInfo` → Shell 进程接管 | Shell 直接操作 Leash 的 `Transaction` |

---

## 全景对比

| 维度 | View 动画 | Window 动画 |
|------|----------|-------------|
| **操作对象** | Surface 内的 GraphicBuffer | SurfaceControl (Leash) |
| **改变的是** | 像素内容（每帧新图） | 图层元数据（matrix/alpha/crop） |
| **Surface** | 不变，始终同一个 | 不变，始终同一个 |
| **SurfaceControl 属性** | 不变 | 每帧在变 |
| **是否重绘** | 是，每帧重绘 dirty 区域 | 否，内容不变 |
| **执行进程** | App 进程 | SystemServer (WMS) / Shell 进程 |
| **执行线程** | UI Thread + RenderThread | Binder + SF 主线程 |
| **性能开销** | 较高（CPU measure/layout + GPU draw） | 很低（仅 Binder 传几个数值） |
| **Winscope 可见** | 不可见 | 可见 |
| **典型场景** | 按钮缩放、列表展开、RecyclerView 动画 | Activity 打开/关闭、窗口最小化 |

---

## 为什么 Winscope 只能看到 Window 动画

```
SurfaceFlinger 每个 Layer 包含两部分：

┌─────────────────────────────┐
│  元数据（SurfaceControl 属性） │ ← Winscope 记录这个
│  position / matrix / alpha   │
│  crop / z-order / corner...  │
├─────────────────────────────┤
│  buffer 内容（像素数据）       │ ← Winscope 不记录这个
│  GraphicBuffer               │
└─────────────────────────────┘
```

| 动画类型 | 元数据 | buffer | Winscope 看到的 |
|---------|--------|--------|----------------|
| View 动画 | 不变 | 每帧新 buffer | "静止"——属性没变化可记录 |
| Window 动画 | 每帧在变 | 不变 | "运动"——属性变化被完整记录 |

> **Winscope 是图层属性的录像机，不是像素的录像机。**

---

## 一图总结

```
                        ┌──────────────────────────┐
                        │      SurfaceFlinger       │
                        │   (所有画面必经此路上屏)     │
                        └─────┬──────────┬──────────┘
                              │          │
                    合成新 buffer    合成变换后的图层
                              │          │
               ┌──────────────┘          └──────────────┐
               │                                        │
     ┌─────────▼──────────┐               ┌─────────────▼────────────┐
     │    View 动画        │               │     Window 动画           │
     │                    │               │                          │
     │ 作用对象: Surface   │               │ 作用对象: SurfaceControl  │
     │ (GraphicBuffer)    │               │ (Leash)                  │
     │                    │               │                          │
     │ 每帧: 新像素        │               │ 每帧: 新属性              │
     │ App 重绘 ✓         │               │ App 重绘 ✗               │
     │ Winscope 可见 ✗    │               │ Winscope 可见 ✓           │
     │                    │               │                          │
     │ 开销: 高            │               │ 开销: 低                  │
     │ (CPU+GPU 每帧画)    │               │ (仅 Binder 传数值)        │
     └────────────────────┘               └──────────────────────────┘
```

---

## 源码路径索引

| 组件 | 文件路径 |
|------|---------|
| View 绘制入口 | `frameworks/base/core/java/android/view/ViewRootImpl.java` — `performDraw()`, `draw()` |
| View 动画应用 | `frameworks/base/core/java/android/view/View.java` — `applyLegacyAnimation()` (line 26623) |
| RenderNode 动画矩阵 | `frameworks/base/core/java/android/view/View.java` — line 26900 |
| Surface 锁定/提交 | `frameworks/base/core/java/android/view/Surface.java` — `lockCanvas()`, `unlockCanvasAndPost()` |
| SurfaceAnimator | `frameworks/base/services/core/java/com/android/server/wm/SurfaceAnimator.java` |
| Window 动画规格 | `frameworks/base/services/core/java/com/android/server/wm/WindowAnimationSpec.java` — `apply()` (line 132) |
| 动画执行器 | `frameworks/base/services/core/java/com/android/server/wm/SurfaceAnimationRunner.java` |
| Shell Transition | `frameworks/base/services/core/java/com/android/server/wm/Transition.java` |