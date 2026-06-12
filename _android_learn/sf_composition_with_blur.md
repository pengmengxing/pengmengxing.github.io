---
layout: private_post
title:  "SurfaceFlinger 合成流程详解：含 BackgroundBlur 的混合合成"
date:   2026-05-30 00:00:00 +0800
categories: android
tag: SurfaceFlinger
description: backgroundBlurRadius 触发窗口级模糊，模糊层及其下方层强制 GPU(CLIENT)合成，上方层保持 DEVICE 的混合合成路径分析。
---

**场景**：屏幕上有 5 个 Layer，从低到高依次为：

| Z 序 | Layer | 特性 |
|------|-------|------|
| ① | 壁纸 | 普通 Buffer |
| ② | 桌面 | 普通 Buffer |
| ③ | 音量条 | `backgroundBlurRadius > 0`（窗口级模糊） |
| ④ | 状态栏 | 普通 Buffer，HWC 有空闲 plane |
| ⑤ | 导航栏 | 普通 Buffer，HWC 有空闲 plane |

**结论**：①②③ 强制 CLIENT（GPU 合成），④⑤ 为 DEVICE（HWC 直接叠加）。

---

## 一、合成类型分配：forceClientComposition 传播

### 1.1 找到「模糊层」

```cpp
// Output.cpp:974
compositionengine::OutputLayer* Output::findLayerRequestingBackgroundComposition() const {
    compositionengine::OutputLayer* layerRequestingBgComposition = nullptr;
    for (auto* layer : getOutputLayersOrderedByZ()) {          // 从底到顶遍历
        const auto* compState = layer->getLayerFE().getCompositionState();

        if (compState->sidebandStream != nullptr) return nullptr;  // 有 sideband → 禁用模糊
        if (compState->hasProtectedContent && ...) return nullptr;  // 受保护内容 → 禁用模糊

        if (compState->isOpaque) continue;                         // 不透明层跳过

        if (compState->backgroundBlurRadius > 0 || compState->blurRegions.size() > 0) {
            layerRequestingBgComposition = layer;                  // 记录最顶部的模糊层
        }
    }
    return layerRequestingBgComposition;  // 返回 ③ 音量条
}
```

### 1.2 以模糊层为分界线传播 forceClientComposition

```cpp
// Output.cpp:851
void Output::updateCompositionState(const CompositionRefreshArgs& refreshArgs) {
    // 找到模糊层（音量条），flag 初始化为 true
    mLayerRequestingBackgroundBlur = findLayerRequestingBackgroundComposition();
    bool forceClientComposition = (mLayerRequestingBackgroundBlur != nullptr);  // = true

    for (auto* layer : getOutputLayersOrderedByZ()) {  // 从底(①)到顶(⑤)
        layer->updateCompositionState(
            refreshArgs.updatingGeometryThisFrame,
            refreshArgs.devOptForceClientComposition || forceClientComposition,  // 传入强制标志
            ...);

        if (mLayerRequestingBackgroundBlur == layer) {
            forceClientComposition = false;   // ③处理完后关闭强制标志
        }
    }
}
```

**传播结果**：

```
Layer       forceClientComposition    最终合成方式
① 壁纸      true  ────────────────→  CLIENT（GPU 强制）
② 桌面      true  ────────────────→  CLIENT（GPU 强制）
③ 音量条    true → 处理后置 false  →  CLIENT（GPU 强制，同时是模糊层）
④ 状态栏    false ───────────────→   DEVICE（HWC 决定，有空闲 plane）
⑤ 导航栏    false ───────────────→   DEVICE（HWC 决定，有空闲 plane）
```

---

## 二、chooseCompositionStrategy：向 HWC 做 validate

### 2.1 入口：prepareFrame

```cpp
// Output.cpp:1144
void Output::prepareFrame() {
    std::optional<android::HWComposer::DeviceRequestedChanges> changes;

    bool success = chooseCompositionStrategy(&changes);  // 向 HWC 做 validate
    resetCompositionStrategy();

    if (success) {
        applyCompositionStrategy(changes);               // 应用 HWC 返回的结果
    }
    finishPrepareFrame();                                // 通知 RenderSurface 准备帧
}
```

### 2.2 chooseCompositionStrategy：发送 validate 请求

```cpp
// Display.cpp:248
bool Display::chooseCompositionStrategy(
        std::optional<android::HWComposer::DeviceRequestedChanges>* outChanges) {

    auto& hwc = getCompositionEngine().getHwComposer();
    const bool requiresClientComposition = anyLayersRequireClientComposition(); // ①②③ = true

    // 调用 HWC validate（同步 Binder 跨进程，SF 主线程在此阻塞）
    hwc.getDeviceCompositionChanges(
        *halDisplayId,
        requiresClientComposition,       // 告知 HWC 本帧有 CLIENT 合成
        getState().earliestPresentTime,
        getState().expectedPresentTime,
        getState().frameInterval,
        outChanges);                     // HWC 返回各层合成类型变更

    return true;
}
```

### 2.3 HWC validate 的内部流程

```cpp
// HWComposer.cpp:511
status_t HWComposer::getDeviceCompositionChanges(...) {

    // 有 CLIENT 合成时不能跳过 validate（因为 clientTarget 还未准备好）
    const bool canSkipValidate = [&] {
        if (frameUsesClientComposition) return false;   // ← 本场景：不可跳过
        ...
    }();

    if (canSkipValidate) {
        hwcDisplay->presentOrValidate(...);  // 尝试合并 validate+present
    } else {
        hwcDisplay->validate(...);           // ← 本场景走这里，单独 validate
    }

    // 读取 HWC 的决定：哪些层改变了合成类型
    hwcDisplay->getChangedCompositionTypes(&changedTypes);
    // changedTypes 里可能包含：
    //   ④状态栏 → DEVICE（HWC 确认有 plane）
    //   ⑤导航栏 → DEVICE（HWC 确认有 plane）

    hwcDisplay->getClientTargetProperty(&clientTargetProperty); // 获取 clientTarget 格式要求
    hwcDisplay->acceptChanges();            // SF 接受 HWC 的决定

    return NO_ERROR;
}
```

### 2.4 applyCompositionStrategy：应用 HWC 结果

```cpp
// Display.cpp:289
void Display::applyCompositionStrategy(const std::optional<DeviceRequestedChanges>& changes) {
    if (changes) {
        applyChangedTypesToLayers(changes->changedTypes);          // 更新每层的合成类型
        applyDisplayRequests(changes->displayRequests);
        applyLayerRequestsToLayers(changes->layerRequests);
        applyClientTargetRequests(changes->clientTargetProperty);  // 设置 clientTarget 格式
    }

    auto& state = editState();
    state.usesClientComposition = anyLayersRequireClientComposition(); // = true（①②③）
    state.usesDeviceComposition = !allLayersRequireClientComposition();// = true（④⑤）
}
```

---

## 三、GPU 合成：composeSurfaces + drawLayersInternal

### 3.1 composeSurfaces 触发 RenderEngine

```cpp
// Output.cpp:1337（仅传入 CLIENT 层 ①②③）
std::optional<base::unique_fd> Output::composeSurfaces(...) {
    if (!outputState.usesClientComposition) return {};  // 无 CLIENT 层则跳过

    // 准备 clientTarget Buffer（DEVICE 层不需要这个 Buffer）
    auto clientCompositionLayers = generateClientCompositionRequests(...);
    // clientCompositionLayers = { 壁纸, 桌面, 音量条 }，不含状态栏/导航栏

    // 提交给 RenderEngine 线程异步执行
    renderEngine.drawLayers(clientCompositionDisplay, clientCompositionLayers, buffer, ...);
}
```

### 3.2 drawLayersInternal：执行 GPU 合成（含模糊）

```cpp
// SkiaRenderEngine.cpp:888（节选核心逻辑）
void SkiaRenderEngine::drawLayersInternal(...) {

    // 发现有模糊层 → 切换到离屏缓冲区，以便拍照作为 blurInput
    if (mBlurFilter) {
        for (const auto& layer : layers) {          // layers = {壁纸, 桌面, 音量条}
            if (layerHasBlur(layer, ...)) {
                activeSurface = dstSurface->makeSurface(...); // 创建离屏缓冲区
                canvas = activeSurface->getCanvas();
                break;
            }
        }
    }

    // 按 Z 序从底到顶绘制 CLIENT 层
    for (const auto& layer : layers) {

        // ─── 渲染 ① 壁纸 ─────────────────────────────────────
        // DrawImage：壁纸像素写入 activeSurface（离屏缓冲区）

        // ─── 渲染 ② 桌面 ─────────────────────────────────────
        // DrawImage：桌面像素叠加写入 activeSurface

        //这里的模糊输入是一个SkImage
        sk_sp<SkImage> blurInput;
        // ─── 渲染 ③ 音量条 ───────────────────────────────────
        if (mBlurFilter && layerHasBlur(layer, ...)) {

            // 步骤1：把离屏缓冲区当前内容（壁纸+桌面）转为纹理
            blurInput = activeSurface->makeTemporaryImage();
            // ↑ blurInput 里只有 ①壁纸 + ②桌面，不含 ④状态栏/⑤导航栏

            // 步骤2：对纹理执行模糊
            // GaussianBlurFilter.cpp:45 或 KawaseBlurFilter.cpp:76
            auto blurredImage = mBlurFilter->generate(
                context,
                layer.backgroundBlurRadius,  // 模糊半径
                blurInput,                   // 输入：壁纸+桌面
                blurRect);                   // 模糊区域

            // 步骤3：切回 dstSurface（最终 clientTarget）
            canvas = dstCanvas;
            activeSurface = dstSurface;

            // 步骤4：把模糊结果画到音量条背后
            mBlurFilter->drawBlurRegion(canvas, effectRegion, ..., blurredImage, blurInput);
        }
        // DrawImage：音量条自身内容（图标/文字）叠加到模糊背景上
    }

    // flush + flushGL：提交 GPU 指令，等待执行完成
    // 输出：dstSurface = clientTarget Buffer（内容：壁纸+桌面+音量条模糊效果）
}
```

---

## 四、HWC present：三份输入合成送显

### 4.1 将 clientTarget 交给 HWC

```cpp
// HWComposer.cpp:499
status_t HWComposer::setClientTarget(
        HalDisplayId displayId, uint32_t slot,
        const sp<Fence>& acquireFence,      // GPU 渲染完成的 Fence
        const sp<GraphicBuffer>& target,    // clientTarget Buffer（含 ①②③）
        ui::Dataspace dataspace, float hdrSdrRatio) {

    hwcDisplay->setClientTarget(slot, target, acquireFence, dataspace, hdrSdrRatio);
    // → HIDL/AIDL 调用 → HWC HAL 接收 clientTarget
}
```

### 4.2 presentAndGetReleaseFences：触发 HWC 硬件合成

```cpp
// Display.cpp:405
compositionengine::Output::FrameFences Display::presentFrame() {
    auto fences = impl::Output::presentFrame();  // 获取 clientTargetAcquireFence

    auto& hwc = getCompositionEngine().getHwComposer();
    hwc.presentAndGetReleaseFences(*halDisplayIdOpt, getState().earliestPresentTime);
    // ↑ 触发 HWC 执行最终合成

    fences.presentFence = hwc.getPresentFence(*halDisplayIdOpt);  // 获取 presentFence

    // 收集每个 DEVICE 层的 releaseFence
    for (const auto* layer : getOutputLayersOrderedByZ()) {
        auto hwcLayer = layer->getHwcLayer();
        fences.layerFences.emplace(hwcLayer,
            hwc.getLayerReleaseFence(*halDisplayIdOpt, hwcLayer));
    }
    return fences;
}

// HWComposer.cpp:632
status_t HWComposer::presentAndGetReleaseFences(...) {
    hwcDisplay->present(&displayData.lastPresentFence);
    // ↑ HIDL/AIDL → HWC HAL → Display Controller 硬件合成：
    //
    //   plane 0: clientTarget Buffer（GPU 合成的 ①壁纸+②桌面+③音量条模糊）
    //   plane 1: ④状态栏 GraphicBuffer（App 直接写入，DMA 读取）
    //   plane 2: ⑤导航栏 GraphicBuffer（App 直接写入，DMA 读取）
    //   → DRM AtomicCommit → Display Controller 扫描输出到屏幕

    hwcDisplay->getReleaseFences(&releaseFences);  // 收集各 DEVICE 层的 releaseFence
}
```

### 4.3 释放 Fence 回调给 App

```cpp
// Output.cpp:1636
void Output::presentFrameAndReleaseLayers(...) {
    auto frame = presentFrame();  // 执行 HWC present，得到 fences

    mRenderSurface->onPresentDisplayCompleted();

    for (auto* layer : getOutputLayersOrderedByZ()) {
        sp<Fence> releaseFence = Fence::NO_FENCE;

        // DEVICE 层（④⑤）：从 HWC 获取 releaseFence
        if (auto hwcLayer = layer->getHwcLayer()) {
            releaseFence = frame.layerFences[hwcLayer];
        }

        // CLIENT 层（①②③）：合并 clientTargetAcquireFence
        if (outputState.usesClientComposition) {
            releaseFence = Fence::merge("LayerRelease", releaseFence,
                                        frame.clientTargetAcquireFence);
        }

        // 通知 App：这个 Buffer 已可复用（通过 Binder 回调）
        layer->getLayerFE().setReleaseFence(releaseFence);
    }
}
```

---

## 五、完整流程总览

```
Vsync 到来
│
├─ SF 主线程：updateCompositionState（Output.cpp:851）
│   forceClientComposition=true 从底向上传播到 ③ 音量条
│   ① 壁纸  → CLIENT（强制）
│   ② 桌面  → CLIENT（强制）
│   ③ 音量条→ CLIENT（强制） ← 此后 force=false
│   ④ 状态栏→ 不强制
│   ⑤ 导航栏→ 不强制
│
├─ SF 主线程：prepareFrame → chooseCompositionStrategy（Display.cpp:248）
│   └─ HWC validate（Binder 阻塞，约 3~5ms）
│       HWC 返回：④状态栏=DEVICE，⑤导航栏=DEVICE
│   applyCompositionStrategy：usesClient=true，usesDevice=true
│
├─ SF 主线程：composeSurfaces（Output.cpp:1337）
│   只把 ①②③ 提交给 RenderEngine 线程
│
├─ RenderEngine 线程：drawLayersInternal（SkiaRenderEngine.cpp）
│   ┌────────────────────────────────────────────────┐
│   │  离屏缓冲区（activeSurface）                    │
│   │  ① 壁纸   DrawImage ──→ 写入离屏              │
│   │  ② 桌面   DrawImage ──→ 叠加写入离屏           │
│   │  ③ 音量条 BackgroundBlur                       │
│   │     makeTemporaryImage()  ← 离屏转纹理         │
│   │     KawaseBlur(blurInput) ← 模糊处理           │
│   │     drawBlurRegion()       ← 模糊结果画到背后  │
│   │     DrawImage()            ← 音量条自身内容    │
│   └──────────── flush → clientTarget Buffer ───────┘
│
├─ SF 主线程：setClientTarget（HWComposer.cpp:499）
│   把 clientTarget Buffer（含 ①②③ GPU 合成结果）交给 HWC
│
├─ SF 主线程：presentAndGetReleaseFences（HWComposer.cpp:632）
│   HWC Display Controller 硬件叠加：
│   ┌────────────────────────────────────────────────────┐
│   │  plane 0: clientTarget（①+②+③模糊，GPU 合成）    │
│   │  plane 1: ④ 状态栏 Buffer（DMA 直读，零 GPU）      │
│   │  plane 2: ⑤ 导航栏 Buffer（DMA 直读，零 GPU）      │
│   └─────────── DRM AtomicCommit ─────────── 屏幕 ──────┘
│   返回 presentFence + 各层 releaseFence
│
└─ SF 主线程：presentFrameAndReleaseLayers（Output.cpp:1636）
    向 App 发送 releaseFence（Binder 回调）
    ① 壁纸/② 桌面/③ 音量条：clientTargetAcquireFence
    ④ 状态栏/⑤ 导航栏：HWC layerReleaseFence
    App 收到 fence 信号后可复用对应 GraphicBuffer
```

---

## 六、关键结论

| 问题 | 结论 |
|------|------|
| 模糊输入包含状态栏/导航栏吗？ | **不包含**。blurInput 是 ①②绘制完的离屏快照，④⑤是 DEVICE 层，从未进入 RenderEngine |
| DEVICE 层的像素谁负责？ | App RenderThread 负责渲染到 GraphicBuffer，HWC 通过 DMA 直读，**不经过任何 GPU** |
| 为什么底层必须 CLIENT？ | 模糊需要采样背后像素；DEVICE 层像素 RenderEngine 无法访问，必须在 GPU 合成路径里 |
| validate 何时可跳过？ | `frameUsesClientComposition=false`（全 DEVICE 帧）且时间条件满足时，可跳过走 presentOrValidate |
| GPU 因 DEVICE 节省了什么？ | 状态栏/导航栏各 1 次 DrawImage + flush，约减少 0.1ms GPU 时间；更重要的是节省内存带宽 |
