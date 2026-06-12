---
layout: private_post
title:  "SurfaceFlinger 合成流程详解：含圆角小窗的混合合成"
date:   2026-05-29 00:00:00 +0800
categories: android
tag: SurfaceFlinger
description: 圆角 setCornerRadius() 只影响小窗自身，不通过 forceClientComposition 传播，壁纸与桌面仍保持 DEVICE 硬件合成的混合合成案例分析。
---

**场景**：屏幕上有 5 个 Layer，从低到高依次为：

| Z 序 | Layer | 合成方式 |
|------|-------|---------|
| ① | 壁纸 | DEVICE（HWC） |
| ② | 桌面 | DEVICE（HWC） |
| ③ | 小窗（厂商实现，`setCornerRadius()` 圆角） | CLIENT（GPU） |
| ④ | 状态栏 | DEVICE（HWC） |
| ⑤ | 导航栏 | DEVICE（HWC） |

**核心结论**：圆角不通过 `forceClientComposition` 传播，只影响小窗自身；
壁纸和桌面不被拉入 GPU 合成，保持 DEVICE。

---

## 与 BackgroundBlur 场景的本质区别

| 维度 | BackgroundBlur 场景 | 圆角场景 |
|------|---------------------|---------|
| 触发入口 | `findLayerRequestingBackgroundComposition()` | `hasRoundedCorners()` |
| 强制路径 | `updateCompositionState` → `forceClientComposition` 参数向下传播 | `isClientCompositionForced()` 单层判断 |
| 影响范围 | 模糊层及其以下所有层全部 CLIENT | 仅圆角层自身 CLIENT |
| 原因 | 模糊需要采样下方所有层的像素作为 blurInput | GPU 只需 clipRRect 裁剪这一层的内容 |

---

## 第一步：圆角属性如何产生影响

`setCornerRadius()` 调用后，`LayerSnapshot` 更新两个字段（`LayerSnapshot.cpp:519,531`）：

```cpp
// LayerSnapshot.cpp:519 — forceClientComposition 的设置条件
forceClientComposition = shadowSettings.length > 0   // 阴影
                      || stretchEffect.hasEffect()    // Overscroll 拉伸
                      || edgeExtensionEffect.hasEffect()
                      || borderSettings.strokeWidth > 0;
// ↑ 注意：圆角不在这里！LayerSnapshot 层面不强制 CLIENT

// LayerSnapshot.cpp:531 — 圆角让层变成非不透明
isOpaque = contentOpaque
        && !roundedCorner.hasRoundedCorners()   // ← 有圆角 → isOpaque = false
        && color.a == 1.f;
```

`LayerFE::hasRoundedCorners()`（`LayerFE.cpp:377`）：

```cpp
bool LayerFE::hasRoundedCorners() const {
    return mSnapshot->roundedCorner.hasRoundedCorners();
    // 即 roundedCorner.radius.x > 0 && radius.y > 0
}
```

---

## 第二步：OutputLayer 判断 forceClientComposition

`OutputLayer::updateCompositionState`（`OutputLayer.cpp:346`）：

```cpp
void OutputLayer::updateCompositionState(
        bool includeGeometry, bool forceClientComposition, ...) {

    if (includeGeometry) {
        state.forceClientComposition = false;  // 每帧先清零

        // 触发强制的条件（圆角不在其中）：
        if ((layerFEState->isSecure && !outputState.isSecure) ||
            (state.bufferTransform & ui::Transform::ROT_INVALID)) {
            state.forceClientComposition = true;
        }
    }

    // 本场景：
    //   没有 blur 层 → forceClientComposition 参数 = false
    //   layerFEState->forceClientComposition = false（圆角不在 LayerSnapshot 条件里）
    if (layerFEState->forceClientComposition
        || !profile.isDataspaceSupported(state.dataspace)
        || forceClientComposition) {
        state.forceClientComposition = true;
    }
    // 结论：state.forceClientComposition = false（圆角走另一条路）
}
```

---

## 第三步：写入 HWC 前的强制判断（核心）

`OutputLayer::isClientCompositionForced`（`OutputLayer.cpp:979`）：

```cpp
bool OutputLayer::isClientCompositionForced(bool isPeekingThrough) const {
    return getState().forceClientComposition          // = false（上面分析）
        || (!isPeekingThrough                         // = true（正常渲染路径）
            && getLayerFE().hasRoundedCorners());      // = true（小窗有圆角）
    // 最终返回 true ← 圆角在这里触发强制 CLIENT
}
```

`OutputLayer::writeCompositionTypeToHWC`（`OutputLayer.cpp:884`）：

```cpp
void OutputLayer::writeCompositionTypeToHWC(
        HWC2::Layer* hwcLayer,
        Composition requestedCompositionType,   // 初始 = DEVICE（HWC 分配的）
        bool isPeekingThrough, bool skipLayer) {

    if (isClientCompositionForced(isPeekingThrough)) {   // = true
        requestedCompositionType = Composition::CLIENT;  // 覆盖为 CLIENT
    }

    if (outputDependentState.hwc->hwcCompositionType != requestedCompositionType) {
        outputDependentState.hwc->hwcCompositionType = requestedCompositionType;
        hwcLayer->setCompositionType(requestedCompositionType);
        // ↑ 通过 HIDL/AIDL 告知 HWC：这一层必须是 CLIENT
    }
}
```

**关键**：`state.forceClientComposition=false` 但 `isClientCompositionForced()=true`，
圆角走的是独立检测路径，**不通过 `updateCompositionState` 传播，只影响自身**。

---

## 第四步：DEVICE 层的 Buffer 如何送给 HWC

壁纸/桌面/状态栏/导航栏走 `writeOutputIndependentPerFrameStateToHWC`（`OutputLayer.cpp:722`）：

```cpp
void OutputLayer::writeOutputIndependentPerFrameStateToHWC(
        HWC2::Layer* hwcLayer,
        const LayerFECompositionState& outputIndependentState,
        Composition compositionType, bool skipLayer) {

    hwcLayer->setColorTransform(outputIndependentState.colorTransform);
    hwcLayer->setSurfaceDamage(surfaceDamage);

    switch (compositionType) {
        case Composition::DEVICE:
        case Composition::CURSOR:
        case Composition::DISPLAY_DECORATION:
            writeBufferStateToHWC(hwcLayer, outputIndependentState, skipLayer);
            // ↑ DEVICE 层在这里把 GraphicBuffer 句柄交给 HWC
            break;
        case Composition::CLIENT:
            // Ignored ← CLIENT 层不在这里提交 Buffer
            //            GPU 合成后走 setClientTarget 单独提交
            break;
    }
}
```

`writeBufferStateToHWC`（`OutputLayer.cpp:835`）：

```cpp
void OutputLayer::writeBufferStateToHWC(HWC2::Layer* hwcLayer, ...) {
    // 从缓存获取 slot（避免重复传输同一 Buffer）
    hwcSlotAndBuffer = state.hwc->hwcBufferCache.getHwcSlotAndBuffer(
            outputIndependentState.buffer);   // App 的 GraphicBuffer
    hwcFence = outputIndependentState.acquireFence;  // App GPU 写完的 fence

    hwcLayer->setBuffer(hwcSlotAndBuffer.slot,
                        hwcSlotAndBuffer.buffer,  // GraphicBuffer handle
                        hwcFence);
    // HWC 等 acquireFence 信号后，通过 DMA 直接读像素，不经过 SF 的 GPU
}
```

---

## 第五步：generateClientCompositionRequests 筛选 CLIENT 层

`Output.cpp:1493`，只有 `requiresClientComposition()=true` 的层才进入 GPU 合成：

```cpp
std::vector<LayerFE::LayerSettings> Output::generateClientCompositionRequests(...) {
    for (auto* layer : getOutputLayersOrderedByZ()) {   // 底 → 顶

        const bool clientComposition = layer->requiresClientComposition();
        // requiresClientComposition() = (hwcCompositionType == CLIENT)
        // ① 壁纸   hwcCompositionType=DEVICE → false  跳过
        // ② 桌面   hwcCompositionType=DEVICE → false  跳过
        // ③ 小窗   hwcCompositionType=CLIENT → true   ← 进入 GPU 合成
        // ④ 状态栏 hwcCompositionType=DEVICE → false  跳过
        // ⑤ 导航栏 hwcCompositionType=DEVICE → false  跳过

        if (clientComposition || clearClientComposition) {
            auto clientCompositionSettings =
                    layerFE.prepareClientComposition(targetSettings);
            clientCompositionLayers.push_back(std::move(*clientCompositionSettings));
        }
    }
    // 返回只含 ③小窗 的列表，传入 RenderEngine
    return clientCompositionLayers;
}
```

---

## 第六步：LayerFE 把圆角参数打包给 RenderEngine

`LayerFE::prepareClientCompositionInternal`（`LayerFE.cpp:121`）：

```cpp
std::optional<LayerFE::LayerSettings>
LayerFE::prepareClientCompositionInternal(...) const {
    LayerFE::LayerSettings layerSettings;

    // 圆角参数写入 LayerSettings（从 LayerSnapshot 读取）
    const auto& roundedCornerState = mSnapshot->roundedCorner;
    layerSettings.geometry.roundedCornersRadius = roundedCornerState.radius;
    // ↑ 例如 vec2{20.f, 20.f}（小窗圆角半径）
    layerSettings.geometry.roundedCornersCrop = roundedCornerState.cropRect;
    // ↑ 圆角裁剪的矩形区域（窗口边界）

    // Buffer、dataspace、alpha 等其他属性...
    prepareBufferStateClientComposition(layerSettings, targetSettings);
    return layerSettings;
}
```

---

## 第七步：SkiaRenderEngine 执行圆角裁剪

**计算圆角 RRect**（`SkiaRenderEngine.cpp:885`）：

```cpp
const auto [bounds, roundRectClip] =
    getBoundsAndClip(
        layer.geometry.boundaries,           // 窗口内容边界
        layer.geometry.roundedCornersCrop,   // 圆角裁剪区域
        layer.geometry.roundedCornersRadius  // vec2{20, 20}
    );
```

`getBoundsAndClip`（`SkiaRenderEngine.cpp:178`）三条执行路径：

```cpp
static inline std::pair<SkRRect, SkRRect> getBoundsAndClip(
        const FloatRect& boundsRect,
        const FloatRect& cropRect,
        const vec2& cornerRadius) {

    if (cornerRadius.x > 0 && cornerRadius.y > 0) {

        // 路径1（最优）：bounds 和 crop 一致，直接生成圆角矩形，clip 为空
        if (bounds == crop || crop.isEmpty()) {
            return {SkRRect::MakeRectXY(bounds, cornerRadius.x, cornerRadius.y), clip};
        }

        if (crop.contains(bounds)) {
            const auto insetCrop = crop.makeInset(cornerRadius.x, cornerRadius.y);
            if (insetCrop.contains(bounds)) {
                // 路径2：crop 完全包含且内缩后仍包含，不需要圆角，clip 为空
                return {SkRRect::MakeRect(bounds), clip};
            }
            // 路径3：尝试合并为单次 RRect drawcall
            if (intersectionIsRoundRect(bounds, crop, insetCrop, cornerRadius, radii)) {
                SkRRect intersectionBounds;
                intersectionBounds.setRectRadii(bounds, radii);
                return {intersectionBounds, clip};
            }
        }

        // 路径4（通用）：用 clip 来裁剪圆角
        clip.setRectXY(crop, cornerRadius.x, cornerRadius.y);
    }
    return {SkRRect::MakeRect(bounds), clip};
}
```

**实际绘制**（`SkiaRenderEngine.cpp:1237`）：

```cpp
// 先应用圆角裁剪（如果需要）
if (!roundRectClip.isEmpty()) {
    canvas->clipRRect(roundRectClip, true);  // true = 抗锯齿裁剪
}

// 绘制内容
if (!bounds.isRect()) {
    paint.setAntiAlias(true);
    canvas->drawRRect(bounds, paint);   // 圆角边缘抗锯齿绘制
} else {
    canvas->drawRect(bounds.rect(), paint);
}
```

此时 clientTarget Buffer 的像素状态：

```
clientTarget Buffer（全屏大小）：
┌────────────────────────────────────┐
│  alpha=0   alpha=0   alpha=0       │  ← 小窗以外全透明（alpha=0）
│  alpha=0  ╭──────────────╮  alpha=0│  ← 圆角处：抗锯齿半透明过渡
│  alpha=0  │              │  alpha=0│
│  alpha=0  │  小窗内容     │  alpha=0│  ← 圆角矩形内：正常像素
│  alpha=0  │              │  alpha=0│
│  alpha=0  ╰──────────────╯  alpha=0│  ← 圆角处：抗锯齿半透明过渡
│  alpha=0   alpha=0   alpha=0       │
└────────────────────────────────────┘
```

---

## 第八步：HWC 5 个 plane 最终合成送显

```
HWC validate 收到的请求：
  ① 壁纸   compositionType=DEVICE  → setBuffer(壁纸 GraphicBuffer, acquireFence)
  ② 桌面   compositionType=DEVICE  → setBuffer(桌面 GraphicBuffer, acquireFence)
  ③ 小窗   compositionType=CLIENT  → 不 setBuffer，等 setClientTarget
  ④ 状态栏 compositionType=DEVICE  → setBuffer(状态栏 GraphicBuffer, acquireFence)
  ⑤ 导航栏 compositionType=DEVICE  → setBuffer(导航栏 GraphicBuffer, acquireFence)

GPU 合成完成后（HWComposer.cpp:499）：
  setClientTarget(clientTargetBuffer, acquireFence)
  ← 把含圆角小窗的 GPU 合成结果交给 HWC

HWC present → DRM AtomicCommit：
  plane 0 : ① 壁纸 Buffer     DMA 直读，零 GPU 开销
  plane 1 : ② 桌面 Buffer     DMA 直读，零 GPU 开销
  plane 2 : clientTarget      GPU 输出，小窗圆角外 alpha=0，壁纸/桌面透出
  plane 3 : ④ 状态栏 Buffer   DMA 直读，零 GPU 开销
  plane 4 : ⑤ 导航栏 Buffer   DMA 直读，零 GPU 开销
            ↓ Display Controller alpha 混合叠加
            屏幕最终画面
```

---

## 完整调用链

```
App 调用 SurfaceControl.Transaction.setCornerRadius(radius)
  └─ LayerSnapshot::roundedCorner.radius = radius
  └─ LayerSnapshot::isOpaque = false（有圆角→非不透明）

SF 合成循环（每帧）
  │
  ├─ updateCompositionState（Output.cpp:851）
  │   无 blur 层 → forceClientComposition 传播 = false
  │   小窗 state.forceClientComposition = false（圆角不触发此路径）
  │
  ├─ writeStateToHWC（OutputLayer.cpp:472）
  │   ├─ DEVICE 层（①②④⑤）
  │   │   writeBufferStateToHWC → hwcLayer->setBuffer(GraphicBuffer, fence)
  │   │
  │   └─ 小窗（③）
  │       isClientCompositionForced() = true（hasRoundedCorners=true）
  │       writeCompositionTypeToHWC → hwcLayer->setCompositionType(CLIENT)
  │
  ├─ chooseCompositionStrategy（Display.cpp:248）
  │   hwc.getDeviceCompositionChanges() → HWC validate（Binder 阻塞）
  │   HWC 确认：①②④⑤=DEVICE，③=CLIENT
  │   applyCompositionStrategy：usesClientComposition=true，usesDeviceComposition=true
  │
  ├─ composeSurfaces（Output.cpp:1337）
  │   generateClientCompositionRequests → 只取 ③小窗
  │   └─ LayerFE::prepareClientCompositionInternal
  │       roundedCornersRadius/Crop → LayerSettings
  │
  ├─ RenderEngine::drawLayersInternal（layers={③小窗}）
  │   getBoundsAndClip → SkRRect bounds + roundRectClip
  │   canvas->clipRRect(roundRectClip, antialias=true)
  │   canvas->drawRRect(bounds, paint)（或 drawRect）
  │   flush surface + flushGL
  │   → clientTarget Buffer（小窗圆角内有像素，圆角外 alpha=0）
  │
  ├─ HWComposer::setClientTarget(clientTargetBuffer, acquireFence)
  │
  └─ HWComposer::presentAndGetReleaseFences
      hwcDisplay->present()
      → DRM AtomicCommit
      → Display Controller 5 plane 叠加 → 屏幕
```

---

## 关键结论

| 问题 | 结论 |
|------|------|
| 圆角是否像 blur 一样传播 CLIENT？ | **否**。圆角只在 `isClientCompositionForced()` 里影响自身，不传播 |
| 壁纸/桌面为何保持 DEVICE？ | 没有 blur 层 → `forceClientComposition` 不传播 → 壁纸/桌面不被强制 |
| 圆角的 GPU 裁剪在哪执行？ | `SkiaRenderEngine::drawLayersInternal` 里的 `canvas->clipRRect()` |
| DEVICE 层的 Buffer 怎么到 HWC？ | `writeBufferStateToHWC → hwcLayer->setBuffer(GraphicBuffer, acquireFence)` |
| CLIENT 层的结果怎么到 HWC？ | GPU 合成完 → `HWComposer::setClientTarget(clientTargetBuffer)` |
| GPU 节省了什么？ | 4 个 DEVICE 层零 GPU 开销，GPU 只处理 1 个圆角小窗 |
