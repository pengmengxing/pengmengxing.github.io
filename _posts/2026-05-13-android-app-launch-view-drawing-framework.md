---
layout: post
title:  "Android 应用启动与 View 绘制 — 三进程图形框架总览"
date:   2026-05-13 00:00:00 +0800
categories: android
tag: 图形框架
---

> 基于 Android 16 AOSP（`J:\aosp16`）源码分析

---

## 一、三进程全景架构

```mermaid
graph TB
    subgraph APP["App 进程"]
        AT["ActivityThread\nmain()"]
        ACT["Activity\nonCreate / onResume"]
        VRI["ViewRootImpl\nperformTraversals()"]
        CHO["Choreographer\ndoFrame()"]
        HR["HardwareRenderer\n(UI线程)"]
        RT["RenderThread\n进程单例"]
        BBQ_P["BLASTBufferQueue\n生产者端"]
    end

    subgraph SS["SystemServer 进程"]
        ATMS["ActivityTaskManagerService\n(ATMS)"]
        WMS["WindowManagerService\n(WMS)"]
        WS["WindowState\n单窗口状态"]
        SCC["SurfaceComposerClient\n向SF申请Layer"]
    end

    subgraph SF["SurfaceFlinger 进程"]
        EVT["EventThread\nVsync分发"]
        SFL["SurfaceFlinger\n合成主循环"]
        LAYER["Layer\nBufferQueue消费端"]
        CE["CompositionEngine\nOutput::present()"]
        HWC["HWComposer / GPU\n硬件/软件合成"]
        DISP["Display Driver\n屏幕输出"]
    end

    AT -->|"Binder\nattachApplication"| ATMS
    ATMS -->|"scheduleLaunchActivity"| ACT
    ACT -->|"addView"| VRI
    VRI -->|"Binder\nIWindowSession"| WMS
    WMS --> WS
    WMS --> SCC
    SCC -->|"Binder\nISurfaceComposer"| SF

    CHO -->|"TRAVERSAL回调"| VRI
    VRI --> HR
    HR -->|"syncAndDrawFrame"| RT
    RT -->|"eglSwapBuffers"| BBQ_P

    BBQ_P -->|"queueBuffer\n共享内存"| LAYER
    EVT -->|"Vsync信号"| SFL
    EVT -->|"Vsync信号"| CHO
    SFL --> CE
    CE --> HWC
    HWC --> DISP
    LAYER --> CE
```

---

## 二、应用启动流程

### 2.1 完整启动时序

```mermaid
sequenceDiagram
    participant User as 用户/Launcher
    participant ATMS as ATMS (SystemServer)
    participant Zygote as Zygote进程
    participant App as App进程
    participant WMS as WMS (SystemServer)
    participant SF as SurfaceFlinger

    User->>ATMS: startActivity() [Binder]
    ATMS->>ATMS: ActivityStarter.execute()
    ATMS->>Zygote: 进程不存在 → Process.start() [Socket]
    Zygote->>App: forkAndSpecialize()
    App->>App: ActivityThread.main()
    App->>App: Looper.prepareMainLooper()
    App->>ATMS: attachApplication() [Binder]
    ATMS->>App: scheduleLaunchActivity() [Binder]
    App->>App: handleLaunchActivity()
    App->>App: Activity.attach() 绑定Window/Context
    App->>App: Activity.onCreate() setContentView()
    App->>App: Activity.onResume()
    App->>WMS: addWindow() [Binder IWindowSession]
    WMS->>WMS: 创建 WindowState
    WMS->>SF: createSurface() [Binder ISurfaceComposer]
    SF->>SF: 创建 Layer + BLASTBufferQueue
    SF-->>WMS: 返回 SurfaceControl
    WMS-->>App: 返回 SurfaceControl
    App->>App: ViewRootImpl 持有 Surface
    App->>App: 请求第一帧绘制 (scheduleTraversals)
```

### 2.2 阶段说明

| 阶段 | 核心操作 | 关键文件 |
|------|---------|---------|
| **进程孵化** | Launcher → ATMS → Zygote fork | `ActivityStarter.java` `ActivityTaskSupervisor.java` |
| **Application初始化** | ActivityThread.main() → attach() | `ActivityThread.java` |
| **Activity创建** | onCreate() → setContentView() → View树构建 | `Activity.java` |
| **Window注册** | ViewRootImpl.setView() → WMS.addWindow() | `ViewRootImpl.java` `WindowManagerService.java` |
| **Surface创建** | WMS → SF创建Layer → BLASTBufferQueue | `SurfaceComposerClient.cpp` `BLASTBufferQueue.cpp` |
| **首帧绘制** | scheduleTraversals → Vsync → performTraversals | `ViewRootImpl.java` |

---

## 三、View 绘制流程

### 3.1 Vsync 驱动链

```mermaid
graph LR
    HV["HWComposer\n硬件Vsync"] -->|"周期信号"| DS["SF Scheduler\nDispSync软件锁相"]
    DS -->|"INVALIDATE\nREFRESH"| SFMQ["SF MessageQueue\n合成循环"]
    DS -->|"onVsync()"| DER["DisplayEventReceiver\nApp进程"]
    DER --> CHO["Choreographer\ndoFrame(frameTimeNanos)"]
    CHO -->|"INPUT"| INP["输入事件处理"]
    CHO -->|"ANIMATION"| ANI["ValueAnimator\nObjectAnimator"]
    CHO -->|"TRAVERSAL"| TRV["ViewRootImpl\nperformTraversals()"]
```

### 3.2 performTraversals — 三大流程

```mermaid
flowchart TD
    PT["ViewRootImpl.performTraversals()"]

    PT --> PM["performMeasure()"]
    PM --> M["View.measure()"]
    M --> OM["onMeasure()\n递归确定宽高\nMeasureSpec传递"]

    PT --> PL["performLayout()"]
    PL --> L["View.layout()"]
    L --> OL["onLayout()\n递归确定位置\nleft/top/right/bottom"]

    PT --> PD["performDraw()"]
    PD --> D["draw()"]
    D --> URDL["updateRootDisplayList()\nUI线程"]
    URDL --> VD["View.draw()\n→ onDraw(canvas)"]
    VD --> RC["RecordingCanvas\n录制绘制指令"]
    RC --> DL["DisplayList\n(RenderNode)"]

    DL --> SADF["RenderProxy.syncAndDrawFrame()\n通知RenderThread"]
    SADF --> DFT["DrawFrameTask.run()\nRenderThread"]
    DFT --> CC["CanvasContext.draw()"]
    CC --> SKIA["Skia Pipeline\nOpenGL ES / Vulkan"]
    SKIA --> GPU["GPU渲染\n→ GraphicBuffer"]
```

### 3.3 UI线程与RenderThread分工

```mermaid
sequenceDiagram
    participant UI as UI线程 (Main)
    participant RT as RenderThread
    participant GPU as GPU
    participant SF as SurfaceFlinger

    Note over UI: Choreographer.doFrame()
    UI->>UI: performMeasure / performLayout
    UI->>UI: updateRootDisplayList() 录制DisplayList
    UI->>RT: syncAndDrawFrame() 同步点
    Note over UI,RT: 同步阶段(短暂阻塞): 传递属性/纹理
    RT-->>UI: 同步完成，UI线程解锁
    Note over UI: UI线程继续处理下一帧逻辑
    RT->>RT: CanvasContext.draw()
    RT->>GPU: OpenGL/Vulkan 提交命令
    GPU-->>RT: 渲染完成 (acquireFence)
    RT->>SF: eglSwapBuffers → queueBuffer
    SF->>SF: 下一个Vsync合成
    SF-->>RT: releaseFence (Buffer可复用)
```

---

## 四、图形框架流水线（Buffer生命周期）

### 4.1 BufferQueue 状态机

```mermaid
stateDiagram-v2
    [*] --> FREE : 初始状态
    FREE --> DEQUEUED : dequeueBuffer()\nApp申请Buffer
    DEQUEUED --> QUEUED : queueBuffer()\nGPU渲染完成，提交
    DEQUEUED --> FREE : cancelBuffer()\n取消（异常路径）
    QUEUED --> ACQUIRED : acquireBuffer()\nSF获取Buffer准备合成
    ACQUIRED --> FREE : releaseBuffer()\nSF合成完毕，释放
```

### 4.2 BLASTBufferQueue 跨进程流转

```mermaid
sequenceDiagram
    participant RT as RenderThread (App)
    participant BBQ_P as BLASTBufferQueue-Producer
    participant BQC as BufferQueueCore (共享内存)
    participant BBQ_C as BLASTBufferQueue-Consumer
    participant SF as SurfaceFlinger Layer

    RT->>BBQ_P: eglSwapBuffers()
    BBQ_P->>BQC: queueBuffer(slot, acquireFence)
    Note over BQC: DEQUEUED → QUEUED
    BQC->>BBQ_C: onFrameAvailable() 回调
    BBQ_C->>SF: 通知有新帧
    SF->>BQC: acquireBuffer()
    Note over BQC: QUEUED → ACQUIRED
    SF->>SF: 等待 acquireFence 信号 (GPU写完)
    SF->>SF: HWC/GPU 合成此Layer
    SF->>BQC: releaseBuffer(releaseFence)
    Note over BQC: ACQUIRED → FREE
    BQC->>BBQ_P: onBufferReleased() 回调
    RT->>BQC: dequeueBuffer() 复用该Buffer
```

### 4.3 Fence 三种类型

```mermaid
graph LR
    GPU_W["GPU 写入 GraphicBuffer"] -->|"信号ready"| AF["acquireFence\nSF可读取此Buffer"]
    SF_R["SurfaceFlinger 读取合成完毕"] -->|"信号ready"| RF["releaseFence\nApp可dequeue复用"]
    DISP["Display 扫描出帧"] -->|"信号ready"| PF["presentFence\n计算实际帧呈现时间\nFrameTimeline使用"]
```

---

## 五、SurfaceFlinger 合成循环

```mermaid
flowchart TD
    VS["Vsync信号到来"] --> INV["SF::onMessageInvalidate()"]
    INV --> HT["handleTransaction()\n处理Layer属性变更Transaction"]
    HT --> RLS["rebuildLayerStacks()\n重建LayerSnapshot只读快照"]
    RLS --> REF["SF::onMessageRefresh()"]
    REF --> CEP["CompositionEngine::present()"]

    CEP --> PFA["Output::prepareFrameAsync()\n遍历每个Layer"]
    PFA --> DEC{合成策略决策}
    DEC -->|"简单Layer\n无变换/无混合"| DEVICE["DEVICE类型\nHWC硬件Overlay\n最低功耗"]
    DEC -->|"复杂Layer\n模糊/旋转/特效"| CLIENT["CLIENT类型\nGPU SkiaRenderEngine"]
    DEC -->|"纯色填充"| SOLID["SOLID_COLOR\nHWC直接填充"]

    DEVICE --> HWC2["HWComposer\nvalidateDisplay()"]
    CLIENT --> GPU2["SkiaRenderEngine\n离屏渲染到FrameBuffer"]
    SOLID --> HWC2

    GPU2 --> HWC2
    HWC2 --> PRES["presentDisplay()\n提交到Display驱动"]
    PRES --> PF2["presentFence回传\nFrameTimeline记录实际呈现时间"]
    PRES --> RF2["releaseFence回传给App\nBuffer可复用"]
```

---

## 六、关键类与文件索引

### App 进程

| 类 | 文件路径 | 核心职责 |
|---|---------|---------|
| `ActivityThread` | `base/core/java/android/app/ActivityThread.java` | 主线程Loop，分发Activity生命周期 |
| `ViewRootImpl` | `base/core/java/android/view/ViewRootImpl.java` | 窗口根，驱动measure/layout/draw |
| `Choreographer` | `base/core/java/android/view/Choreographer.java` | Vsync回调分发，帧时序协调 |
| `HardwareRenderer` | `base/core/java/android/graphics/HardwareRenderer.java` | UI线程侧硬件加速接口 |
| `RenderProxy` | `base/libs/hwui/renderthread/RenderProxy.cpp` | UI线程→RenderThread任务提交 |
| `DrawFrameTask` | `base/libs/hwui/renderthread/DrawFrameTask.cpp` | 单帧同步点，跨线程协调 |
| `CanvasContext` | `base/libs/hwui/renderthread/CanvasContext.cpp` | 渲染上下文，管理EGLSurface |
| `RenderNode` | `base/libs/hwui/RenderNode.cpp` | 场景图节点，持有DisplayList |

### SystemServer 进程

| 类 | 文件路径 | 核心职责 |
|---|---------|---------|
| `ActivityTaskManagerService` | `base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java` | Activity调度主体 |
| `ActivityStarter` | `base/services/core/java/com/android/server/wm/ActivityStarter.java` | 启动流程控制 |
| `WindowManagerService` | `base/services/core/java/com/android/server/wm/WindowManagerService.java` | 窗口管理主体 |
| `WindowState` | `base/services/core/java/com/android/server/wm/WindowState.java` | 单窗口服务端状态 |
| `RootWindowContainer` | `base/services/core/java/com/android/server/wm/RootWindowContainer.java` | Window容器树根节点 |

### SurfaceFlinger 进程

| 类 | 文件路径 | 核心职责 |
|---|---------|---------|
| `SurfaceFlinger` | `native/services/surfaceflinger/SurfaceFlinger.cpp` | 合成主循环、Transaction处理 |
| `Layer` | `native/services/surfaceflinger/Layer.cpp` | Layer属性、Buffer生命周期 |
| `LayerSnapshot` | `native/services/surfaceflinger/FrontEnd/LayerSnapshot.cpp` | 合成用只读快照 |
| `Output` | `native/services/surfaceflinger/CompositionEngine/Output.cpp` | 合成循环主体 |
| `HWComposer` | `native/services/surfaceflinger/DisplayHardware/HWComposer.cpp` | HWC接口总入口 |
| `EventThread` | `native/services/surfaceflinger/Scheduler/EventThread.cpp` | Vsync事件分发 |
| `FrameTimeline` | `native/services/surfaceflinger/FrameTimeline/FrameTimeline.cpp` | 帧时间线追踪，Jank分析 |

### libgui（跨进程共享）

| 类 | 文件路径 | 核心职责 |
|---|---------|---------|
| `BLASTBufferQueue` | `native/libs/gui/BLASTBufferQueue.cpp` | App→SF异步缓冲队列（主路径） |
| `BufferQueueCore` | `native/libs/gui/BufferQueueCore.cpp` | FREE/DEQUEUED/QUEUED/ACQUIRED状态机 |
| `SurfaceComposerClient` | `native/libs/gui/SurfaceComposerClient.cpp` | 创建Layer、提交Transaction |

> 路径前缀均为 `J:\aosp16\frameworks\`

---

## 七、核心概念速查

### 合成类型

| 类型 | 路径 | 适用场景 |
|------|------|---------|
| `DEVICE` | HWC 硬件 Overlay | 普通矩形Layer，无特殊混合 |
| `CLIENT` | GPU SkiaRenderEngine | 模糊、旋转、复杂特效 |
| `SOLID_COLOR` | HWC 纯色填充 | 背景色Layer |
| `CURSOR` | 硬件光标Plane | 系统鼠标光标 |

### 关键数据结构

| 对象 | 所在进程 | 含义 |
|------|---------|------|
| `ViewRootImpl` | App | 窗口根，驱动 measure/layout/draw |
| `Surface` (Java) | App | ANativeWindow 的 Java 封装 |
| `BLASTBufferQueue` | App/Native | Android 12+ 统一跨进程缓冲路径 |
| `SurfaceControl` | App + WMS | Layer客户端句柄，Transaction操作对象 |
| `WindowState` | SystemServer | WMS侧单窗口状态 |
| `Layer` | SurfaceFlinger | SF内部合成单元，持有BufferQueue消费端 |
| `LayerSnapshot` | SurfaceFlinger | 每帧合成前生成的只读快照 |
| `GraphicBuffer` | 跨进程共享 | GPU可直接读写的ION/DMA-BUF缓冲区 |
| `Fence` | 跨进程传递 | GPU同步信号（acquire/release/present） |

---

## 八、后续深入方向

```mermaid
graph LR
    A["今日：三进程框架总览"] --> B["应用启动细节\nActivityThread + Zygote fork"]
    A --> C["Window/Surface建立\nViewRootImpl.setView()\n→ WMS → SF createLayer"]
    A --> D["三大绘制流程\nperformTraversals逐行追踪"]
    A --> E["RenderThread渲染\nDrawFrameTask → CanvasContext\n→ Skia/EGL"]
    A --> F["SF合成循环\nonMessageRefresh全链路"]
    A --> G["Fence同步机制\nBLASTBufferQueue三Fence"]
    B --> H["进阶：冷启动优化\nJank定位"]
    C --> H
    D --> H
    E --> H
    F --> H
    G --> H
```