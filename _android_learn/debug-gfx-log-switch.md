---
layout: private_post
title:  "跨进程公共 Log 开关实现（debug.gfx.log）"
date:   2026-06-02 00:00:00 +0800
categories: android
tag: SurfaceFlinger
description: 在 SystemServer / HWUI / SurfaceFlinger 三方读同一个 System Property（debug.gfx.log），一条 adb setprop 即时开关三方日志，无需重启进程，基于 base::CachedBoolProperty 与 SystemProperties 回调实现。
---

> 目标：在 **SystemServer / HWUI / SurfaceFlinger** 三方各自提供一个公共 log 开关，进程内所有类可访问，通过 `adb` 命令开关，**无需重启手机/进程即时生效**。

---

## 一、核心原理

跨进程**不能共享同一个 C++/Java 对象**，但 System Property 是内核级全局共享区，所有进程都能读同一个 key。因此「公共开关」的本质是：

> **三个进程读同一个 property key（`debug.gfx.log`）**，一条 `adb setprop` 同时影响三方；每个进程内部各做一个**进程级单例**供本进程所有类访问。

### 为什么不用重启即时生效

`setprop` 写的是内核维护的全局 property 共享区。每次 `set` 会 bump 一个全局 **serial 版本号**，读取方靠该 serial 判断「变了没」。只要进程还活着、且走「每次读 / 监听变化」的路径，新值立即可见。

| 进程 | 进程模型 | 读取机制 | 即时生效 |
|------|---------|---------|---------|
| SurfaceFlinger | 独立 native 进程 | `base::CachedBoolProperty` 单例 | ✅ 实时 |
| HWUI | `libhwui.so`，链接进每个硬件加速 App 进程 | `base::CachedBoolProperty` 单例 + 原子缓存/限流 | ✅ ≤250ms |
| SystemServer | 独立 java 进程 | `SystemProperties` + `addChangeCallback` | ✅ 回调刷新 |

> ⚠️ **避坑**：HWUI 现成的 `Properties::load()` 体系（static 成员）只在 RenderThread 特定时机刷新，界面静止不重绘时开关不更新——**不保证即时**。故 HWUI 这里采用独立 `CachedBoolProperty` 单例（方案 B），与 SF 一致。

### `CachedBoolProperty` 为何高效，以及它的多线程坑

`system/libbase/include/android-base/properties.h` 提供。它在每次 `Get()` 时只比对 property-area 的 serial，**值没变走缓存（近零开销）**，值一变才重新解析。

⚠️ **但它内部自带一把 `std::mutex`，每次 `Get()` 都要加锁**（见 `CachedParsedProperty::Get`）。底层 serial 比对是无锁的，真正被串行化的是这把 mutex。HWUI 高频调用场景下——例如 SurfaceFlinger 的 RenderEngine 线程，5 个线程 1s 内调用上千次——所有线程都排队抢这把锁，于是出现**锁等待（lock-wait）**。

> 注：`CachedProperty::Get` 自身被注释标为「完全线程不安全」，所以 `CachedParsedProperty` 才用一把进程级 mutex 把它串行化——这把锁是必需的，但放在每次调用的热路径上就成了瓶颈。

### HWUI 的解法：原子缓存 + 限流刷新

把**热路径改成无锁原子读**，把「读 property」这件相对重的事**限流**到至多每隔 250ms 由**一个**线程做一次：

- 热路径（绝大多数调用）= 一次 `std::atomic<bool>` 的 relaxed load，**无锁、无竞争**。
- 用 `compare_exchange_strong` 当闸门：每个间隔内只有抢到 CAS 的那个线程才会调 `sProp.Get()`，其余线程直接返回缓存值。因此那把 mutex 永远**无竞争**，且对 `CachedProperty` 的访问天然单线程。
- 代价：开关生效有最多 250ms 延迟（对调试日志开关无感）。`setprop` 后无需重启即生效不变。

> 备选「零延迟 + 热路径零开销」方案：起后台 watcher 线程阻塞在 `__system_property_wait` 上，属性一变即更新原子。但 libhwui 链进每个 App 进程，给每进程常驻一个线程只为一个 debug 开关太重，故未采用。

---

## 二、统一约定

| 项 | 值 |
|----|----|
| Property key | `debug.gfx.log`（三处一字不差） |
| 前缀 | `debug.`（userdebug/eng 默认可写，重启清空）；长期保留改 `persist.gfx.log`（需配 `property_contexts`） |
| 开 | `adb shell setprop debug.gfx.log 1` |
| 关 | `adb shell setprop debug.gfx.log 0` |
| 查 | `adb shell getprop debug.gfx.log` |

---

## 三、改动清单

| # | 文件 | 改动 | 接入示范点 |
|---|------|------|-----------|
| 1 | `frameworks/base/libs/hwui/GfxLogSwitch.h` | 新建 | HWUI 开关 |
| 2 | `frameworks/base/libs/hwui/renderthread/CanvasContext.cpp` | 改 | `CanvasContext::draw()` |
| 3 | `frameworks/native/services/surfaceflinger/GfxLogSwitch.h` | 新建 | SF 开关 |
| 4 | `frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp` | 改 | `setTransactionState()` |
| 5 | `frameworks/base/services/core/java/com/android/server/gfx/GfxLogSwitch.java` | 新建 | SystemServer 开关 |
| 6 | `frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java` | 改 | `relayoutWindow()` |

均为 header-only / 单文件，**无需改 Android.bp**：
- `libhwui` / `libsurfaceflinger` 已链接 `libbase`（`CachedBoolProperty` 所在库）。
- Java 文件落在 `services/core/java/...`，该模块以 glob `**/*.java` 收源，新文件自动纳入。

---

## 四、具体实现

### 1. HWUI 开关 — `frameworks/base/libs/hwui/GfxLogSwitch.h`（新建）

```cpp
#ifndef ANDROID_HWUI_GFX_LOG_SWITCH_H
#define ANDROID_HWUI_GFX_LOG_SWITCH_H

#include <android-base/properties.h>
#include <log/log.h>
#include <utils/Timers.h>

#include <atomic>

namespace android {
namespace uirenderer {

// Shared graphics log switch. Same property key is read by SurfaceFlinger and
// system_server so a single `setprop debug.gfx.log 1` toggles all of them.
#define PROPERTY_GFX_LOG "debug.gfx.log"

// Hot path is a single relaxed atomic load: lock-free and contention-free even
// when called thousands of times per second from many threads. The property is
// re-read at most once per 250 ms, and a compare-exchange ensures only ONE
// thread does that refresh per interval -- so CachedBoolProperty's internal
// mutex is always uncontended. setprop still takes effect without restart, with
// latency bounded by the refresh interval. Returns false until the property is set.
inline bool gfxLogEnabled() {
    static constexpr nsecs_t kRefreshIntervalNs = ms2ns(250);

    static std::atomic<bool> sCached{false};
    static std::atomic<nsecs_t> sNextRefreshNs{0};
    // Only touched by the single refresher selected by the CAS below, so its
    // (thread-unsafe) serial comparison runs single-threaded by construction.
    static base::CachedBoolProperty sProp(PROPERTY_GFX_LOG);

    const nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
    nsecs_t next = sNextRefreshNs.load(std::memory_order_relaxed);
    if (now >= next &&
        sNextRefreshNs.compare_exchange_strong(next, now + kRefreshIntervalNs,
                                               std::memory_order_relaxed)) {
        sCached.store(sProp.Get(/*default_value=*/false), std::memory_order_relaxed);
    }
    return sCached.load(std::memory_order_relaxed);
}

}  // namespace uirenderer
}  // namespace android

// The calling translation unit must define LOG_TAG (standard in hwui: "HWUI").
#define GFXLOG(...)                                  \
    do {                                             \
        if (android::uirenderer::gfxLogEnabled()) {  \
            ALOGD(__VA_ARGS__);                      \
        }                                            \
    } while (0)

#endif  // ANDROID_HWUI_GFX_LOG_SWITCH_H
```

### 2. HWUI 接入 — `CanvasContext.cpp`

```cpp
// include 区
#include "../GfxLogSwitch.h"
#include "../Properties.h"

// CanvasContext::draw() 开头
void CanvasContext::draw(bool solelyTextureViewUpdates) {
    GFXLOG("CanvasContext::draw context=%p solelyTextureViewUpdates=%d", this,
           solelyTextureViewUpdates);
#ifdef __ANDROID__
    ...
```

### 3. SF 开关 — `frameworks/native/services/surfaceflinger/GfxLogSwitch.h`（新建）

```cpp
#pragma once

#include <android-base/properties.h>
#include <log/log.h>

namespace android {

#define PROPERTY_GFX_LOG "debug.gfx.log"

// NOTE (hot path): a single call is cheap, but do not call repeatedly inside
// the composition loop. Read once at the top of the frame into a local bool.
inline bool gfxLogEnabled() {
    static base::CachedBoolProperty sProp(PROPERTY_GFX_LOG);
    return sProp.Get(/*default_value=*/false);
}

}  // namespace android

// The calling translation unit must define LOG_TAG (standard: "SurfaceFlinger").
#define GFXLOG(...)                       \
    do {                                  \
        if (android::gfxLogEnabled()) {   \
            ALOGD(__VA_ARGS__);           \
        }                                 \
    } while (0)
```

### 4. SF 接入 — `SurfaceFlinger.cpp`

```cpp
// include 区
#include "SurfaceFlinger.h"

#include "GfxLogSwitch.h"

// setTransactionState() 内，permissions 计算之后
status_t SurfaceFlinger::setTransactionState(TransactionState&& transactionState) {
    SFTRACE_CALL();
    ...
    uint32_t permissions = LayerStatePermissions::getTransactionPermissions(originPid, originUid);
    GFXLOG("setTransactionState from pid=%d uid=%d composerStates=%zu displayStates=%zu flags=%#x",
           originPid, originUid, transactionState.mComposerStates.size(),
           transactionState.mDisplayStates.size(), transactionState.mFlags);
    ...
```

### 5. SystemServer 开关 — `frameworks/base/services/core/java/com/android/server/gfx/GfxLogSwitch.java`（新建）

```java
package com.android.server.gfx;

import android.os.SystemProperties;

/**
 * Shared graphics log switch for system_server. Reads the same key
 * {@code debug.gfx.log} as libhwui and SurfaceFlinger.
 * Usage: {@code if (GfxLogSwitch.enabled()) Slog.d(TAG, "...");}
 */
public final class GfxLogSwitch {
    private static final String KEY = "debug.gfx.log";

    private static volatile boolean sEnabled = read();

    static {
        // System property service notifies registered callbacks on any change;
        // refreshes the cache without polling.
        SystemProperties.addChangeCallback(() -> sEnabled = read());
    }

    private static boolean read() {
        return SystemProperties.getBoolean(KEY, false);
    }

    public static boolean enabled() {
        return sEnabled;
    }

    private GfxLogSwitch() {}
}
```

### 6. SystemServer 接入 — `WindowManagerService.java`

```java
// import 区
import com.android.server.gfx.GfxLogSwitch;

// relayoutWindow() 入口
public int relayoutWindow(Session session, IWindow client, LayoutParams attrs,
        int requestedWidth, int requestedHeight, int viewVisibility, int flags, int seq,
        int lastSyncSeqId, WindowRelayoutResult outRelayoutResult) {
    if (GfxLogSwitch.enabled()) {
        Slog.d(TAG, "relayoutWindow client=" + client + " req=" + requestedWidth + "x"
                + requestedHeight + " viewVisibility=" + viewVisibility + " flags=" + flags);
    }
    ...
```

---

## 五、验证

```bash
adb shell setprop debug.gfx.log 1
adb shell getprop debug.gfx.log     # 回读 1 = set 成功（user 版本注意 SELinux）

# 三条链路各自触发：
#   旋转屏幕/触摸滚动  → CanvasContext::draw   (tag: HWUI)
#   任意 App 提交事务  → setTransactionState   (tag: SurfaceFlinger)
#   窗口尺寸/可见性变化 → relayoutWindow         (tag: WindowManager)
adb logcat -s HWUI SurfaceFlinger WindowManager

adb shell setprop debug.gfx.log 0   # 三处日志同时停止
```

---

## 六、注意事项

1. **key 一字不差**：三处都是 `debug.gfx.log`，否则无法一条命令全控。
2. **代码改动需先编译刷入**：「不用重启」指镜像已刷、进程已跑之后纯切换开关值；改代码那次仍需重新编译并让进程加载新代码。
3. **user 版本 SELinux**：`debug.` 在 user 版本可能 `setprop` 被拒（`getprop` 回读不到值即是被拦），需单独配 `property_contexts` + sepolicy。
4. **热路径**：HWUI 的 `gfxLogEnabled()` 已改为「原子缓存 + 限流刷新」，多线程高频调用无锁无竞争，可放心直接调。SF 的 `GfxLogSwitch.h` 仍是 `CachedBoolProperty` 直读版（每次 `Get()` 加锁），单次调用很轻但勿在合成主循环逐 Layer 反复调——在帧开头读一次存局部 `bool` 复用；若 SF 侧也出现高频多线程调用，按 HWUI 同样的原子+限流方式改造即可。
   - HWUI 限流间隔为 250ms（`kRefreshIntervalNs`），即 `setprop` 后最迟 250ms 生效；需要更快可调小该常量。
5. **format 警告**：SF 那条 `%#x` 配 `mFlags`，若编译告警按整型宽度调整（如 `PRIx32`）。
6. **HWUI 进程模型**：HWUI 不是独立进程，是 `libhwui.so`，链接进每个硬件加速 App 进程；各进程各持一份缓存，但都读同一 property，行为一致。
```
