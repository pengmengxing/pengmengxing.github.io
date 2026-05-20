---
layout: post
title:  "View 绘制深度分析：HWUI · RenderThread · SurfaceFlinger"
date:   2026-05-12 00:00:00 +0800
categories: android
tag: HWUI
---

> 基于 Android 16 AOSP 源码实测  
> 核心路径：`frameworks/base/libs/hwui/` · `frameworks/native/services/surfaceflinger/`

---

## 一、总体流水线概览

```mermaid
flowchart LR
    subgraph UI["UI 线程 (Main Thread)"]
        direction TB
        CHO["Choreographer\ndoFrame()"]
        PTV["ViewRootImpl\nperformTraversals()"]
        URDL["updateRootDisplayList()\n录制 DisplayList"]
        DFD["DrawFrameTask.drawFrame()\n投递任务 → 阻塞等待"]
    end

    subgraph RT["RenderThread (进程单例)"]
        direction TB
        SFS["syncFrameState()\n① 同步阶段"]
        UBK["unblockUiThread()\n② 解锁UI线程"]
        CTX["CanvasContext::draw()\n③ GPU渲染阶段"]
        SWAP["swapBuffers()\n④ 提交Buffer"]
    end

    subgraph SF["SurfaceFlinger 进程"]
        direction TB
        LATCH["commit()\nlatchBufferImpl()"]
        COMP["composite()\nCompositionEngine::present()"]
        HWC2["HWComposer / RenderEngine"]
        DISP["Display 输出"]
    end

    CHO -->|"TRAVERSAL回调"| PTV
    PTV --> URDL
    URDL -->|"postAndWait()"| DFD
    DFD -->|"投递 run() 任务"| SFS
    SFS -->|"prepareTextures 成功\n早期解锁"| UBK
    UBK -.->|"解除阻塞"| DFD
    SFS --> CTX
    CTX --> SWAP
    SWAP -->|"queueBuffer\nBLASTBufferQueue"| LATCH
    LATCH --> COMP
    COMP --> HWC2
    HWC2 --> DISP
```

---

## 二、UI 线程详解

### 2.1 Choreographer 触发入口

```mermaid
sequenceDiagram
    participant EVT as SF EventThread
    participant DER as DisplayEventReceiver
    participant CHO as Choreographer (UI线程)
    participant VRI as ViewRootImpl

    EVT->>DER: onVsync(frameTimeNanos, vsyncId)
    DER->>CHO: scheduleVsync() → dispatchVsync()
    Note over CHO: doFrame(frameTimeNanos)
    CHO->>CHO: doCallbacks(INPUT)
    CHO->>CHO: doCallbacks(ANIMATION)
    Note over CHO: ValueAnimator → animateValue()
    CHO->>VRI: doCallbacks(TRAVERSAL)
    Note over VRI: mChoreographerCallback\n→ performTraversals()
```

**关键源码**：`Choreographer.java`
- `VSYNC_SOURCE_APP`：App 侧 Vsync 源，与 SF 的 `VSYNC_SOURCE_SURFACE_FLINGER` 不同
- RenderThread 有独立的 `AChoreographer`，通过 `extendedFrameCallback` 接收 Vsync，用于驱动 RT 侧动画（`RenderThread.cpp:58`）

---

### 2.2 performTraversals 核心流程

```mermaid
flowchart TD
    PT["ViewRootImpl.performTraversals()"]

    PT --> CHK{"mFirst 或尺寸/配置变化?"}
    CHK -->|是| WMS["relayout() → WMS\n重新申请 Surface 尺寸"]
    CHK -->|否| PM["performMeasure()\nMeasureSpec 向下传递"]
    WMS --> PM

    PM --> M["View.measure() → onMeasure()\nMeasureSpec.makeMeasureSpec()\n递归所有子 View 确定宽高"]

    PM --> PL["performLayout()\nleft/top/right/bottom 确定"]
    PL --> L["View.layout() → onLayout()\nAbsoluteLayout/LinearLayout...\n递归所有子 View 确定位置"]

    PL --> PD["performDraw()"]
    PD --> D["draw(fullRedrawNeeded)"]
    D --> HR{"mAttachInfo.mThreadedRenderer != null?"}
    HR -->|"硬件加速路径"| URDL["updateRootDisplayList()\n构建 RootRenderNode"]
    HR -->|"软件绘制回退"| SW["Surface.lockCanvas()\ndrawSoftware()"]

    URDL --> VD["View.draw(canvas)\n→ View.onDraw()\n→ RecordingCanvas 录制"]
    VD --> DL["DisplayList\n(SkiaDisplayList)"]
    DL --> DFT["ThreadedRenderer.draw()\n→ DrawFrameTask.drawFrame()"]
```

**关键细节**：
- `updateRootDisplayList()` 在 UI 线程执行，只"录制"指令，不执行 GPU 操作
- `RecordingCanvas` → `SkiaRecordingCanvas` 将 `onDraw()` 中的 Canvas 调用序列化为 `SkiaDisplayList`（`pipeline/skia/SkiaRecordingCanvas.cpp`）
- 每个 `View` 对应一个 `RenderNode`，`RenderNode` 持有 `DisplayList`

---

### 2.3 UI 线程的同步阻塞点

```mermaid
sequenceDiagram
    participant UI as UI线程
    participant DFT as DrawFrameTask
    participant RT as RenderThread

    UI->>DFT: drawFrame()
    DFT->>DFT: mSyncQueued = systemTime()
    DFT->>RT: queue().post([this]{ run(); })
    DFT->>DFT: mSignal.wait(mLock) ← 阻塞
    Note over RT: syncFrameState() 执行
    alt prepareTextures == true (纹理上传成功)
        RT->>UI: unblockUiThread() ← 早期解锁
        Note over UI: UI线程继续处理下一帧
        Note over RT: 异步执行 draw()
    else prepareTextures == false (纹理缓存满)
        Note over RT: draw() 执行完毕后才解锁
        RT->>UI: unblockUiThread() ← 延迟解锁
        Note over UI: 此帧 UI 线程被阻塞到绘制完成
    end
```

> **源码位置**：`DrawFrameTask.cpp:82-127`  
> `syncFrameState()` 返回 `info.prepareTextures`，即 `CacheManager` 纹理缓存是否充足（默认 24 MB）。缓存耗尽时触发 UI 线程超时等待，是 Jank 的常见原因之一。

---

## 三、RenderThread 详解

### 3.1 RenderThread 初始化

```mermaid
graph LR
    subgraph RT_INIT["RenderThread::initThreadLocals()"]
        CH["AChoreographer_create()\n独立 Vsync 订阅"]
        EGL["EglManager::initialize()\n创建 EGLContext(全局唯一)"]
        RS["RenderState 初始化\nOpenGL状态机"]
        VK["VulkanManager::getInstance()\nVulkan上下文(可选)"]
        CM["CacheManager 初始化\nGPU纹理缓存 24MB默认"]
    end

    CH --> EGL --> RS --> VK --> CM
```

**单例保证**（`RenderThread.cpp:158-166`）：
```cpp
RenderThread& RenderThread::getInstance() {
    [[clang::no_destroy]] static sp<RenderThread> sInstance = []() {
        sp<RenderThread> thread = sp<RenderThread>::make();
        thread->start("RenderThread");
        return thread;
    }();
    return *sInstance;
}
```
- 进程内唯一，所有 Window 共享同一个 `EGLContext`
- 不同 Window 切换靠切换 `EGLSurface`（`CanvasContext::makeCurrent()`）

---

### 3.2 syncFrameState — 同步阶段详解

```mermaid
flowchart TD
    SFS["DrawFrameTask::syncFrameState(TreeInfo& info)"]

    SFS --> TL["TimeLord::vsyncReceived()\n校准帧时间节拍"]
    TL --> MC["CanvasContext::makeCurrent()\n切换到本Window的EGLSurface"]
    MC --> LAY["DeferredLayerUpdater::apply()\n更新离屏Layer纹理"]
    LAY --> PT

    subgraph prepareTree["prepareTree 遍历 RenderNode 树"]
        PT["CanvasContext::prepareTree()"]
        PT2["mAnimationContext->startFrame()\n启动RT动画帧"]
        RN["RenderNode::prepareTree()\n遍历每个RenderNode"]
        REMA["runRemainingAnimations()\n收尾RT动画"]

        subgraph RN_impl["prepareTreeImpl() 每个RenderNode"]
            PSP["pushStagingPropertiesChanges()\n同步 UI线程写入的属性\n(位移/缩放/透明度等)"]
            PSP --> ANIM["AnimatorManager::animate()\nRenderThread侧动画推进\n(PropertyValuesAnimatorSet)"]
            ANIM --> PDSL["pushStagingDisplayListChanges()\n提交新录制的 DisplayList"]
            PDSL --> CHILD["递归处理子 RenderNode"]
            CHILD --> PLU["pushLayerUpdate()\n标记需更新的离屏Layer"]
        end

        PT --> PT2 --> RN --> RN_impl
        RN_impl --> REMA
    end

    REMA --> PTEX["return info.prepareTextures\n(纹理缓存是否充足)"]
```

**关键概念 — Staging 双缓冲**：
- `RenderNode` 维护两套属性：**Staging**（UI线程写）和 **Active**（RT线程读）
- `pushStagingPropertiesChanges()` 是同步阶段把 Staging → Active 的时刻
- 这是 UI 线程和 RenderThread 唯一的数据交换窗口

---

### 3.3 CanvasContext::draw — GPU 渲染阶段

```mermaid
flowchart TD
    D["CanvasContext::draw()"]

    D --> DA["DamageAccumulator::finish()\n计算本帧脏区 dirty rect"]
    DA --> CHK{dirty 为空\n且 skipEmptyFrames?}
    CHK -->|是| SKIP["跳过本帧\ngrContext->flushAndSubmit()"]
    CHK -->|否| GF["getFrame()\ndequeueBuffer from ANativeWindow\n获取可写 GraphicBuffer"]

    GF --> CDR["computeDirtyRect()\n计算 window 脏区"]
    CDR --> PIPE["IRenderPipeline::draw()\nSkiaOpenGLPipeline 或\nSkiaVulkanPipeline"]

    subgraph SKIA_DRAW["Skia Pipeline 渲染（RenderThread）"]
        SKS["SkSurface::makeFromBackendRenderTarget()\n包装 GraphicBuffer 为 Skia 渲染目标"]
        SKS --> SKC["SkCanvas 绘制\n遍历 RenderNode 树"]
        SKC --> RND["RenderNodeDrawable::draw()\n每个 RenderNode 的 SkiaDisplayList"]
        RND --> SKI["Skia 绘图命令\n→ GPU 驱动命令队列"]
        SKI --> FLUSH["grContext->flush()\n提交 GPU 命令(异步)"]
    end

    PIPE --> SKIA_DRAW

    SKIA_DRAW --> WOF["waitOnFences()\n等待上帧 releaseFence 信号"]
    WOF --> FTL["native_window_set_frame_timeline_info()\n传递 vsyncId 给 SurfaceFlinger\n用于 FrameTimeline Jank 分析"]
    FTL --> SWP["IRenderPipeline::swapBuffers()\nEglManager::swapBuffers()\n→ eglSwapBuffers()"]
    SWP --> QB["底层 queueBuffer()\n→ BLASTBufferQueue\n→ SF 收到新帧信号"]
```

---

### 3.4 Pipeline 双路径

```mermaid
graph TB
    CC["CanvasContext"]

    CC --> |"debug.hwui.renderer=skiagl\n(默认 OpenGL)"| OGL["SkiaOpenGLPipeline\n+ EglManager\n+ GrDirectContexts::MakeGL()"]
    CC --> |"debug.hwui.renderer=skiaVk\n(Vulkan)"| VKP["SkiaVulkanPipeline\n+ VulkanManager\n+ GrDirectContexts::MakeVulkan()"]

    OGL --> |"swapBuffers"| EGL["EglManager::swapBuffers()\n→ eglSwapBuffers(display,surface)"]
    VKP --> |"swapBuffers"| VKS["VulkanSurface::present()\n→ vkQueuePresentKHR()"]

    EGL --> BQ["BLASTBufferQueue\nqueueBuffer()"]
    VKS --> BQ
```

---

## 四、SurfaceFlinger 侧详解（Android 16）

> Android 16 SF 主循环采用 `commit()` + `composite()` 两阶段，取代旧的 `onMessageInvalidate/onMessageRefresh`

### 4.1 SF 主线程调度

```mermaid
sequenceDiagram
    participant HWC as HWComposer(硬件)
    participant SCH as SF Scheduler
    participant SF as SurfaceFlinger主线程
    participant CE as CompositionEngine
    participant HWC2 as HWComposer HAL

    HWC->>SCH: onComposerHalVsync(timestamp)
    SCH->>SCH: VsyncDispatch 分发
    SCH->>SF: scheduleFrame() → commit()
    Note over SF: SurfaceFlinger::commit()
    SF->>SF: updateLayerSnapshots(vsyncId)\n处理 Transaction 队列
    SF->>SF: latchBufferImpl()\n从 BLASTBufferQueue 取最新 Buffer
    SF->>SF: chooseRefreshRateForContent()\n动态刷新率选择
    SF->>CE: composite() → CompositionEngine::present()
    CE->>CE: Output::prepareFrameAsync()\n逐 Layer 决策合成类型
    CE->>HWC2: validateDisplay() + presentDisplay()
    HWC2-->>SF: presentFence
    SF->>SF: onCompositionPresented()\nFrameTimeline 记录实际帧时间
    SF-->>RT: releaseFence → BLASTBufferQueue\nBuffer 可复用
```

---

### 4.2 commit() 阶段 — latchBuffer

```mermaid
flowchart TD
    CMT["SurfaceFlinger::commit()"]

    CMT --> ULS["updateLayerSnapshots(vsyncId)\n处理所有待提交 Transaction\n更新 LayerSnapshotBuilder"]

    ULS --> LB["遍历 mLayerLifecycleManager\n→ layer->latchBufferImpl()"]

    subgraph LATCH["latchBufferImpl() 每个 Layer"]
        HRF["hasReadyFrame()?\n检查 BLASTBufferQueue 是否有新 Buffer"]
        HRF -->|有| ACQ["BufferQueueConsumer::acquireBuffer()\nQUEUED → ACQUIRED\n获取 acquireFence"]
        ACQ --> TEX["更新 Layer 纹理\nmExternalTexture = GraphicBuffer"]
        TEX --> MARK["标记 contentDirty = true"]
    end

    LB --> LATCH

    LATCH --> ULH["updateLayerHistory()\n记录帧率历史\n→ 动态刷新率调整"]
    ULH --> MC["mustComposite 标志位\n→ 决定是否触发 composite()"]
```

---

### 4.3 composite() 阶段 — 合成决策

```mermaid
flowchart TD
    CMP["SurfaceFlinger::composite()"]

    CMP --> PFA["Output::prepareFrameAsync()\n构建 CompositionRefreshArgs"]

    PFA --> UCS["OutputLayer::updateCompositionState()\n每个 Layer 决策合成方式"]

    subgraph DECISION["合成类型决策"]
        D1{"Layer 条件检查"}
        D1 -->|"无变换\n无混合\n无圆角\n HWC支持"| DEV["CompositionType::DEVICE\nHWC 硬件 Overlay\n零 GPU 开销"]
        D1 -->|"有模糊/旋转\n有复杂特效\nHWC不支持"| CLI["CompositionType::CLIENT\nGPU RenderEngine 合成\nSkia离屏渲染"]
        D1 -->|"纯色背景Layer"| SOL["CompositionType::SOLID_COLOR\nHWC 直接填充"]
    end

    UCS --> DECISION

    DECISION --> VAL["HWComposer::validateDisplay()\nHW告知最终决策\n(可能推翻软件预判)"]

    VAL -->|"CLIENT类型Layer"| RE["RenderEngine::drawLayers()\nSkia离屏渲染到 FrameBuffer"]
    VAL -->|"DEVICE类型Layer"| HWC3["直接交由 HWC 硬件合成"]

    RE --> HWC3
    HWC3 --> PD["HWComposer::presentDisplay()\nAidlComposerHal::presentDisplay()"]
    PD --> PF["presentFence 回传\n→ FrameTimeline::onPresent()"]
    PD --> RF["releaseFence 逐 Layer 回传\n→ BLASTBufferQueue::releaseBuffer()\n→ App 可 dequeueBuffer 复用"]
```

---

### 4.4 RenderEngine — SF 侧 GPU 合成

```mermaid
flowchart TB
    subgraph RE["RenderEngine (SF进程内 GPU 合成)"]
        SRE["SkiaRenderEngine\n(Android 12+ 统一使用 Skia)"]
        SRE --> FBO["离屏 FramebufferObject\n所有 CLIENT Layer 合成到此"]
        FBO --> SKL["SkiaLayerEngine::drawLayersInternal()\n按 Z-order 叠加绘制"]
        SKL --> SKIFX["Skia 特效：\n模糊(makeWithFilter)\n圆角(clipRRect)\n色彩变换(ColorFilter)"]
        SKIFX --> SUBMIT["grContext->flush()\n提交到 GPU"]
        SUBMIT --> OUT["输出到 HWC FrameBuffer\n与 DEVICE Layer 最终合成"]
    end

    NOTE["注：RenderEngine 与 App RenderThread\n使用不同 EGLContext，互相独立"]
    OUT --> NOTE
```

> **RenderEngine 与 HWUI 的区别**：
> - **HWUI RenderThread**：在 App 进程内，渲染单个 App 的 View 树，输出到 `GraphicBuffer`
> - **SF RenderEngine**：在 SF 进程内，将多个 App 的 `GraphicBuffer`（作为纹理）合成到最终 FrameBuffer

---

## 五、完整跨进程时序图

```mermaid
sequenceDiagram
    participant HW as 硬件 Vsync
    participant SF_SCH as SF Scheduler
    participant APP_CHO as App Choreographer
    participant UI as UI线程
    participant RT as RenderThread
    participant BBQ as BLASTBufferQueue
    participant SF as SurfaceFlinger主线程
    participant GPU_A as App GPU
    participant GPU_SF as SF GPU(RenderEngine)
    participant DISP as Display

    HW->>SF_SCH: Vsync N
    SF_SCH->>APP_CHO: onVsync(frameTimeNanos)
    SF_SCH->>SF: scheduleFrame()

    APP_CHO->>UI: doFrame() → TRAVERSAL
    UI->>UI: performMeasure / performLayout
    UI->>UI: updateRootDisplayList() 录制DisplayList
    UI->>RT: DrawFrameTask.drawFrame() 投递+阻塞
    RT->>RT: syncFrameState() 同步属性/DisplayList
    RT->>UI: unblockUiThread() 早期解锁
    Note over UI: UI线程继续处理
    RT->>RT: CanvasContext::draw()
    RT->>BBQ: dequeueBuffer() 获取GraphicBuffer
    RT->>GPU_A: Skia绘制命令 → 提交GPU
    GPU_A-->>RT: 渲染完成(acquireFence 信号ready)
    RT->>BBQ: queueBuffer(acquireFence)

    BBQ->>SF: onFrameAvailable() 通知
    SF->>SF: commit() → latchBufferImpl()
    SF->>BBQ: acquireBuffer() 取Buffer
    SF->>SF: composite()
    SF->>GPU_SF: RenderEngine drawLayers() (CLIENT层)
    GPU_SF-->>SF: 合成完成
    SF->>DISP: HWComposer::presentDisplay()
    DISP-->>SF: presentFence(帧扫描完成)
    SF->>BBQ: releaseBuffer(releaseFence)
    BBQ->>RT: 通知 Buffer 可复用
```

---

## 六、RenderThread 侧动画 vs UI 线程动画

```mermaid
graph TB
    subgraph UIThread["UI 线程动画"]
        VA["ValueAnimator / ObjectAnimator\n由 Choreographer ANIMATION 回调驱动\n在 UI线程计算属性值\n→ View.setXxx() → invalidate()"]
        VA --> URT["触发 performTraversals()\n→ 整体重绘"]
    end

    subgraph RTThread["RenderThread 动画（推荐路径）"]
        RTA["RenderNode Animator\n(ViewPropertyAnimator 硬件加速)\nPropertyValuesAnimatorSet\n目标：translationX/Y/alpha/scale 等 RenderProperty"]
        RTA --> SYNC["syncFrameState 同步阶段\nAnimatorManager::animate(info)\n在 RT 内直接修改 RenderNode 属性"]
        SYNC --> NOURI["无需 UI线程介入\n仅修改 Active 属性\n→ 下帧直接生效"]
    end

    note["RenderThread 动画优势：\n① 不占用 UI 线程时间\n② 即使 UI 线程 ANR 动画仍流畅\n③ 与绘制在同一线程零延迟"]
```

**关键源码路径**：
- `RenderNode.cpp:248` → `mAnimatorManager.animate(info)` 在 MODE_FULL 时执行
- `AnimationContext.cpp::runRemainingAnimations()` 处理 RT 动画收尾
- `RenderThread.cpp:73-96` → `frameCallback()` RT 侧有独立 Vsync 驱动

---

## 七、帧时序与 FrameTimeline

```mermaid
gantt
    title 单帧时间线（60Hz，16.67ms/帧）
    dateFormat x
    axisFormat %Lms

    section UI线程
    doFrame-INPUT+ANIMATION+TRAVERSAL    :ui1, 0, 4
    阻塞等待 syncFrameState               :crit, ui2, 4, 6

    section RenderThread
    syncFrameState-同步属性+prepareTree   :rt1, 4, 6
    早期解锁 UI 线程                       :milestone, rt2, 6, 0
    CanvasContext-draw-Skia录制           :rt3, 6, 9
    eglSwapBuffers-queueBuffer            :rt4, 9, 10

    section GPU App进程
    Skia GPU 渲染 GraphicBuffer           :gpu1, 9, 13

    section SurfaceFlinger
    latchBuffer                           :sf1, 10, 11
    HWC-RenderEngine合成                  :sf2, 11, 14
    presentDisplay                        :milestone, sf3, 14, 0

    section Display
    屏幕扫描输出 VsyncN+1                  :disp1, 16, 17
```

### FrameTimeline Jank 分类

| Jank 类型 | 根因 | 定位工具 |
|-----------|------|---------|
| `App Deadline Missed` | UI线程/RT超过 deadline | `FrameInfo` + Perfetto `RenderThread` track |
| `SF Deadline Missed` | SF commit/composite 超时 | Perfetto `SurfaceFlinger` track |
| `Buffer Stuffing` | App 生产过快，BLASTBufferQueue 堆积 | `latchBufferImpl` 中 `hasReadyFrame()` 计数 |
| `Prediction Error` | Vsync 预测偏差 | `FrameTimeline::FrameTimelineInfo` |
| `GPU 超时` | GPU 渲染超过 Vsync 周期 | `acquireFence` 等待时间 |

---

## 八、关键函数调用链速查

### App 侧（UI线程 → RenderThread）

```
Choreographer.doFrame()
  └─ ViewRootImpl.performTraversals()
       ├─ performMeasure() → View.onMeasure()
       ├─ performLayout() → View.onLayout()
       └─ performDraw()
            └─ draw()
                 └─ ThreadedRenderer.draw()                    [frameworks/base/core/java/android/graphics/HardwareRenderer.java]
                      └─ updateRootDisplayList()               [UI线程]
                           └─ View.draw() → onDraw()
                                └─ RecordingCanvas / SkiaRecordingCanvas
                                     └─ SkiaDisplayList 录制完成
                      └─ RenderProxy.syncAndDrawFrame()        [跨线程]
                           └─ DrawFrameTask.drawFrame()        [UI线程投递 + 阻塞]
                                └─ DrawFrameTask.run()         [RenderThread执行]
                                     ├─ syncFrameState()
                                     │    └─ CanvasContext.prepareTree()
                                     │         └─ RenderNode.prepareTree()
                                     │              └─ prepareTreeImpl()
                                     │                   ├─ pushStagingPropertiesChanges()  ← Staging→Active
                                     │                   ├─ AnimatorManager.animate()       ← RT动画
                                     │                   └─ pushStagingDisplayListChanges() ← DisplayList同步
                                     ├─ unblockUiThread()      ← 早期解锁
                                     └─ CanvasContext.draw()
                                          ├─ getFrame()          ← dequeueBuffer
                                          ├─ RenderPipeline.draw()
                                          │    └─ SkiaOpenGL/VulkanPipeline.draw()
                                          │         └─ Skia → GPU
                                          └─ swapBuffers()       ← eglSwapBuffers → queueBuffer
```

### SF 侧（commit + composite）

```
onComposerHalVsync()
  └─ Scheduler::scheduleFrame()
       └─ SurfaceFlinger::commit()                     [SurfaceFlinger.cpp:2691]
            ├─ updateLayerSnapshots()                  ← 处理 Transaction，更新 LayerSnapshotBuilder
            └─ latchBufferImpl()                       ← acquireBuffer from BLASTBufferQueue [2648]
       └─ SurfaceFlinger::composite()                  [SurfaceFlinger.cpp:2839]
            └─ CompositionEngine::present()
                 ├─ Output::prepareFrameAsync()         ← 每个 Layer 决策 DEVICE/CLIENT
                 ├─ HWComposer::validateDisplay()       ← HW 最终仲裁
                 ├─ RenderEngine::drawLayers()          ← CLIENT Layer GPU 合成
                 └─ HWComposer::presentDisplay()        ← AidlComposerHal → 扫描输出
       └─ SurfaceFlinger::onCompositionPresented()     [SurfaceFlinger.cpp:3263]
            └─ FrameTimeline::onPresent()              ← 记录实际帧时间
            └─ layer->releasePreviousBuffer()          ← releaseFence 回传 App
```

---

## 九、关键源文件索引

| 文件 | 路径 | 核心内容 |
|------|------|---------|
| `DrawFrameTask.cpp` | `base/libs/hwui/renderthread/` | UI线程↔RT同步点，`syncFrameState()`，早期解锁逻辑 |
| `CanvasContext.cpp` | `base/libs/hwui/renderthread/` | 帧渲染主体：`prepareTree()`，`draw()`，`swapBuffers()` |
| `RenderThread.cpp` | `base/libs/hwui/renderthread/` | 进程单例，独立 Choreographer，EGL/Vulkan 上下文初始化 |
| `RenderNode.cpp` | `base/libs/hwui/` | `prepareTreeImpl()`：属性同步，RT动画，DisplayList提交 |
| `SkiaOpenGLPipeline.cpp` | `base/libs/hwui/pipeline/skia/` | OpenGL路径 draw + swapBuffers |
| `SkiaVulkanPipeline.cpp` | `base/libs/hwui/pipeline/skia/` | Vulkan路径 draw + present |
| `SurfaceFlinger.cpp` | `native/services/surfaceflinger/` | `commit()`，`composite()`，`latchBufferImpl()` |
| `BLASTBufferQueue.cpp` | `native/libs/gui/` | 跨进程 Buffer 传递，acquireFence/releaseFence |
| `Output.cpp` | `native/services/surfaceflinger/CompositionEngine/` | 合成类型决策，`prepareFrameAsync()` |
| `ViewRootImpl.java` | `base/core/java/android/view/` | `performTraversals()`，`performDraw()`，`draw()` |
