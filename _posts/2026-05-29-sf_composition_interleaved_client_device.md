---
layout: post
title:  "SurfaceFlinger 合成流程详解：CLIENT/DEVICE 交叉层的 z 序矛盾处理"
date:   2026-05-29 00:00:00 +0800
categories: android
tag: SurfaceFlinger
description: 当 DEVICE 层被两个 CLIENT 层在 z 序上夹住时，HWC 只有一个 clientTarget，分析 SurfaceFlinger 如何强制中间层降级为 CLIENT 合成。
---

**场景**：屏幕上有 5 个 Layer，从低到高依次为：

| Z 序 | Layer | 初始请求 | 最终合成方式 |
|------|-------|---------|------------|
| ① | 壁纸（`setCornerRadius()` 圆角） | CLIENT | CLIENT |
| ② | 桌面 | DEVICE | **CLIENT（HWC validate 强制变更）** |
| ③ | 小窗（`setCornerRadius()` 圆角） | CLIENT | CLIENT |
| ④ | 状态栏 | DEVICE | DEVICE |
| ⑤ | 导航栏 | DEVICE | DEVICE |

**核心问题**：DEVICE 层（桌面②）被两个 CLIENT 层（壁纸①、小窗③）夹住，
HWC 只有一个 clientTarget，无法在其中插入 DEVICE 层，因此强制桌面变为 CLIENT。

---

## 与前两个场景的本质区别

| 场景 | CLIENT 触发机制 | 影响范围 |
|------|----------------|---------|
| BackgroundBlur | `forceClientComposition` 向下传播 | 模糊层及以下所有层 |
| 单层圆角小窗 | `isClientCompositionForced()` 单层判断 | 仅圆角层自身 |
| **本场景** | SF 强制 + **HWC validate z 序矛盾** | 圆角层自身 + 被夹住的 DEVICE 层 |

---

## 第一步：SF 向 HWC 声明各层合成类型

`OutputLayer::writeCompositionTypeToHWC`（`OutputLayer.cpp:884`）：

```cpp
void OutputLayer::writeCompositionTypeToHWC(
        HWC2::Layer* hwcLayer,
        Composition requestedCompositionType,
        bool isPeekingThrough, bool skipLayer) {

    if (isClientCompositionForced(isPeekingThrough)) {
        requestedCompositionType = Composition::CLIENT;
    }
    hwcLayer->setCompositionType(requestedCompositionType);
}
```

`isClientCompositionForced`（`OutputLayer.cpp:979`）：

```cpp
bool OutputLayer::isClientCompositionForced(bool isPeekingThrough) const {
    return getState().forceClientComposition
        || (!isPeekingThrough && getLayerFE().hasRoundedCorners());
}
```

各层声明结果：

```
① 壁纸   hasRoundedCorners()=true  → setCompositionType(CLIENT)  ← SF 强制，HWC 不可改变
② 桌面   hasRoundedCorners()=false → setCompositionType(DEVICE)  ← SF 请求，HWC 可以改变
③ 小窗   hasRoundedCorners()=true  → setCompositionType(CLIENT)  ← SF 强制，HWC 不可改变
④ 状态栏 hasRoundedCorners()=false → setCompositionType(DEVICE)
⑤ 导航栏 hasRoundedCorners()=false → setCompositionType(DEVICE)
```

---

## 第二步：z 序矛盾——HWC 无法表达单一 clientTarget 被夹住的情况

`HWComposer::getDeviceCompositionChanges`（`HWComposer.cpp:511`）：

```cpp
hwcDisplay->validate(expectedPresentTime, frameInterval, &numTypes, &numRequests);
```

HWC 收到的层信息：

```
z=1  壁纸   compositionType=CLIENT   （SF 强制）
z=2  桌面   compositionType=DEVICE
z=3  小窗   compositionType=CLIENT   （SF 强制）
z=4  状态栏 compositionType=DEVICE
z=5  导航栏 compositionType=DEVICE
```

**HWC 的约束**：clientTarget 是单一 2D 缓冲区，在最终合成时只占一个 z 位置。
若 clientTarget 包含 ① 和 ③，DEVICE 层 ② 无法插入它们之间：

```
HWC 能表达的 z 序：
  [壁纸+桌面+小窗 → clientTarget(z=X)] + 状态栏(DEVICE) + 导航栏(DEVICE)
  ↑ 三层内容只能合并进同一个 clientTarget，无法在其中插入 DEVICE 桌面

无法表达：
  壁纸(CLIENT,z=1) → 桌面(DEVICE,z=2) → 小窗(CLIENT,z=3)
  因为不存在两个独立 clientTarget
```

HWC validate 返回变更请求：

```cpp
hwcDisplay->getChangedCompositionTypes(&changedTypes);
// changedTypes = { 桌面的 hwcLayer → Composition::CLIENT }
//                  ↑ HWC 要求将桌面从 DEVICE 改为 CLIENT
```

---

## 第三步：SF 验证类型变更合法性

`OutputLayer::detectDisallowedCompositionTypeChange`（`OutputLayer.cpp:951`）：

```cpp
void OutputLayer::detectDisallowedCompositionTypeChange(
        Composition from, Composition to) const {
    bool result = false;
    switch (from) {
        case Composition::CLIENT:
            result = false;
            // CLIENT → 任何类型 均不合法
            // 壁纸、小窗是 CLIENT，HWC 改变它们会触发 ALOGE
            // → 圆角效果被保护，不会被 HWC 撤销
            break;

        case Composition::DEVICE:
        case Composition::SOLID_COLOR:
            result = (to == Composition::CLIENT);
            // DEVICE → CLIENT 合法 ✓（桌面 DEVICE→CLIENT 被允许）
            // DEVICE → 其他类型 不合法
            break;

        case Composition::CURSOR:
        case Composition::SIDEBAND:
        case Composition::DISPLAY_DECORATION:
        case Composition::REFRESH_RATE_INDICATOR:
            result = (to == Composition::CLIENT || to == Composition::DEVICE);
            break;
    }
    if (!result) {
        ALOGE("[%s] Invalid device requested composition type change: %s → %s",
              getLayerFE().getDebugName(), ...);
    }
}
```

`Display::applyChangedTypesToLayers`（`Display.cpp:320`）：

```cpp
void Display::applyChangedTypesToLayers(const ChangedTypes& changedTypes) {
    for (auto* layer : getOutputLayersOrderedByZ()) {
        auto hwcLayer = layer->getHwcLayer();
        if (auto it = changedTypes.find(hwcLayer); it != changedTypes.end()) {
            layer->applyDeviceCompositionTypeChange(
                static_cast<Composition>(it->second));
            // 桌面：hwcCompositionType = CLIENT（正式变更）
        }
    }
}
```

`applyCompositionStrategy`（`Display.cpp:289`）：

```cpp
state.usesClientComposition = anyLayersRequireClientComposition();
// ①壁纸(CLIENT) + ②桌面(CLIENT) + ③小窗(CLIENT) → true

state.usesDeviceComposition = !allLayersRequireClientComposition();
// ④状态栏(DEVICE) + ⑤导航栏(DEVICE) → true
```

---

## 第四步：generateClientCompositionRequests 收集三个 CLIENT 层

`Output.cpp:1493`：

```cpp
for (auto* layer : getOutputLayersOrderedByZ()) {
    const bool clientComposition = layer->requiresClientComposition();
    // requiresClientComposition() = (hwcCompositionType == CLIENT)

    // ① 壁纸   hwcCompositionType=CLIENT → true   ← 加入，带圆角参数
    // ② 桌面   hwcCompositionType=CLIENT → true   ← 加入，无圆角（变更后的 CLIENT）
    // ③ 小窗   hwcCompositionType=CLIENT → true   ← 加入，带圆角参数
    // ④ 状态栏 hwcCompositionType=DEVICE → false  → 跳过
    // ⑤ 导航栏 hwcCompositionType=DEVICE → false  → 跳过

    if (clientComposition) {
        auto settings = layerFE.prepareClientComposition(targetSettings);
        clientCompositionLayers.push_back(std::move(*settings));
    }
}
// clientCompositionLayers = { 壁纸(roundedCorners), 桌面(无圆角), 小窗(roundedCorners) }
```

---

## 第五步：LayerFE 打包各层参数

`LayerFE::prepareClientCompositionInternal`（`LayerFE.cpp:121`）：

```cpp
// ① 壁纸
layerSettings.geometry.roundedCornersRadius = roundedCornerState.radius;  // vec2{r, r}
layerSettings.geometry.roundedCornersCrop   = roundedCornerState.cropRect;

// ② 桌面（无 setCornerRadius，radius = vec2{0, 0}）
layerSettings.geometry.roundedCornersRadius = vec2{0.f, 0.f};
layerSettings.geometry.roundedCornersCrop   = FloatRect{};

// ③ 小窗
layerSettings.geometry.roundedCornersRadius = roundedCornerState.radius;  // vec2{r, r}
layerSettings.geometry.roundedCornersCrop   = roundedCornerState.cropRect;
```

---

## 第六步：SkiaRenderEngine 分层绘制

`SkiaRenderEngine::drawLayersInternal`（`SkiaRenderEngine.cpp:885`）：

```cpp
for (const auto& layer : layers) {  // layers = {壁纸, 桌面, 小窗}，按 z 序底→顶

    // ─── ① 壁纸 ─────────────────────────────────────────────
    const auto [bounds, roundRectClip] =
        getBoundsAndClip(boundaries, roundedCornersCrop,
                         roundedCornersRadius);   // radius > 0，生成圆角 SkRRect

    if (!roundRectClip.isEmpty())
        canvas->clipRRect(roundRectClip, true);   // 圆角抗锯齿裁剪
    canvas->drawRRect(bounds, paint);             // 绘制壁纸，圆角外透明

    // ─── ② 桌面 ─────────────────────────────────────────────
    const auto [bounds2, roundRectClip2] =
        getBoundsAndClip(boundaries, cropRect, vec2{0.f, 0.f}); // radius=0
    // roundRectClip2 为空，无裁剪
    canvas->drawRect(bounds2.rect(), paint);      // 普通矩形绘制，覆盖壁纸

    // ─── ③ 小窗 ─────────────────────────────────────────────
    const auto [bounds3, roundRectClip3] =
        getBoundsAndClip(boundaries, roundedCornersCrop,
                         roundedCornersRadius);   // radius > 0

    if (!roundRectClip3.isEmpty())
        canvas->clipRRect(roundRectClip3, true);  // 圆角抗锯齿裁剪
    canvas->drawRRect(bounds3, paint);            // 小窗浮在桌面上，圆角外透明
}

// flush surface + flushGL → 输出 clientTarget Buffer
```

clientTarget 像素状态示意：

```
clientTarget Buffer（全屏大小）：

╭──────────────────────────────────────────╮  ← 壁纸圆角（clipRRect）
│  壁纸像素（全屏底层）                      │
│  桌面像素（覆盖壁纸，普通 drawRect）        │
│         ╭──────────────╮                 │
│         │  小窗内容     │                 │  ← 小窗圆角（clipRRect）
│         ╰──────────────╯                 │
╰──────────────────────────────────────────╯  ← 壁纸圆角

注：壁纸圆角外、小窗圆角外的角落像素 alpha=0（透明）
```

---

## 第七步：HWC 最终合成

```
HWC::setClientTarget(clientTargetBuffer, acquireFence)
  ← GPU 合成结果（含壁纸圆角 + 桌面 + 小窗圆角）

HWC::presentAndGetReleaseFences（HWComposer.cpp:632）
  hwcDisplay->present() → DRM AtomicCommit

Display Controller 叠加：
  plane 0 : clientTarget Buffer  （GPU 合成：①壁纸+②桌面+③小窗）
  plane 1 : ④ 状态栏 Buffer       （DMA 直读，零 GPU）
  plane 2 : ⑤ 导航栏 Buffer       （DMA 直读，零 GPU）
            ↓
          屏幕最终画面
```

---

## 完整调用链

```
SF writeCompositionTypeToHWC
  ① 壁纸  isClientCompositionForced=true  → setCompositionType(CLIENT)  [不可被 HWC 改变]
  ② 桌面  isClientCompositionForced=false → setCompositionType(DEVICE)  [可被 HWC 改变]
  ③ 小窗  isClientCompositionForced=true  → setCompositionType(CLIENT)  [不可被 HWC 改变]
  ④⑤     → setCompositionType(DEVICE)
     │
     ▼
hwcDisplay->validate()
  CLIENT(z=1) → DEVICE(z=2) → CLIENT(z=3)：单一 clientTarget 无法表达此 z 序
  changedTypes = { 桌面 → CLIENT }
     │
     ▼
detectDisallowedCompositionTypeChange
  壁纸：from=CLIENT → result=false（不允许改变，保护圆角）
  桌面：from=DEVICE, to=CLIENT → result=true（允许，DEVICE→CLIENT 合法）
  小窗：from=CLIENT → result=false（不允许改变，保护圆角）
     │
     ▼
applyChangedTypesToLayers
  桌面 hwcCompositionType = CLIENT
  usesClientComposition=true（①②③），usesDeviceComposition=true（④⑤）
     │
     ▼
generateClientCompositionRequests → { ①壁纸, ②桌面, ③小窗 }
  LayerFE::prepareClientCompositionInternal
    壁纸：roundedCornersRadius = vec2{r,r}
    桌面：roundedCornersRadius = vec2{0,0}
    小窗：roundedCornersRadius = vec2{r,r}
     │
     ▼
SkiaRenderEngine::drawLayersInternal
  ① getBoundsAndClip → clipRRect → drawRRect  （壁纸圆角）
  ② getBoundsAndClip → 无 clip → drawRect      （桌面普通矩形）
  ③ getBoundsAndClip → clipRRect → drawRRect  （小窗圆角）
  flush + flushGL → clientTarget Buffer
     │
     ▼
HWC::setClientTarget + presentAndGetReleaseFences
  plane0: clientTarget  plane1: 状态栏  plane2: 导航栏
  DRM AtomicCommit → 屏幕
```

---

## 三个圆角场景对比总结

| 场景 | 被迫变 CLIENT 的原因 | 波及范围 | GPU 处理层数 |
|------|---------------------|---------|------------|
| 单层圆角小窗（DEVICE 壁纸/桌面）| `isClientCompositionForced()` 单层 | 仅小窗 | 1 层 |
| 圆角壁纸 + 圆角小窗（桌面 DEVICE）| SF 强制圆角层 + **HWC z 序矛盾** | 圆角层 + 被夹 DEVICE 层 | 3 层 |
| BackgroundBlur 场景 | `forceClientComposition` 向下传播 | 模糊层及以下所有层 | ≥ 模糊层以下全部 |

**关键规律**：CLIENT 层的位置决定了 DEVICE 层能否独立存在。
只要两个 CLIENT 层在 z 序上夹住了一个 DEVICE 层，HWC 必然将该 DEVICE 层改为 CLIENT，
因为单一 clientTarget 无法在 z 轴上被"插入"。
