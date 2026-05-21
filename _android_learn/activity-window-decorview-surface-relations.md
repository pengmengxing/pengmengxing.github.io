---
layout: private_post
title: "Android 图形体系核心组件与关联全景分析"
---

## 第一章：Activity、Window、DecorView、View、Surface、SurfaceControl、Buffer 都是什么？

### 1.1 各组件定义

| 组件 | 一句话定义 | 类比 |
|------|-----------|------|
| **Activity** | 一个"页面"的管理者，持有 Window，处理生命周期 | 房间的主人 |
| **Window** (PhoneWindow) | 窗口的策略层，定义标题栏、背景、按键行为等装饰策略 | 房间的装修方案 |
| **DecorView** | 窗口的根 View（继承 FrameLayout），承载所有 UI 内容 | 房间的墙壁（所有家具挂在上面） |
| **View / ViewGroup** | 具体的 UI 控件，组成 View 树 | 房间里的家具 |
| **Surface** | 图形缓冲区管道，View 树绘制的目标 | 画纸 |
| **SurfaceControl** | 图层遥控器，控制摆放属性（位置/缩放/透明度） | 相框 |
| **Buffer** (GraphicBuffer) | Surface 内流动的图形缓冲区，承载实际像素 | 画纸上的颜料 |

### 1.2 创建链路：谁创建了谁

```
ActivityThread.performLaunchActivity()
  │
  ▼
① Activity.attach()
  │  创建 PhoneWindow
  │  mWindow = new PhoneWindow(this)
  │
  ▼
② Activity.onCreate() → setContentView(R.layout.xxx)
  │
  ▼
③ PhoneWindow.setContentView()
  │  → installDecor()
  │     → generateDecor()    创建 DecorView
  │     → generateLayout()   创建 mContentParent（内容区域）
  │  → LayoutInflater.inflate(layoutResId, mContentParent)
  │     把用户的 XML 布局加入 View 树
  │
  ▼
④ ActivityThread.handleResumeActivity()
  │  → WindowManager.addView(decorView, layoutParams)
  │
  ▼
⑤ WindowManagerGlobal.addView()
  │  → new ViewRootImpl()              创建 ViewRootImpl
  │  → root.setView(decorView, ...)    绑定 DecorView
  │
  ▼
⑥ ViewRootImpl.setView()
  │  → mWindowSession.addToDisplayAsUser()   Binder 调用 WMS
  │     → WMS 创建 WindowState
  │     → WMS 创建 SurfaceControl（在 SurfaceFlinger 中对应一个 Layer）
  │
  ▼
⑦ ViewRootImpl.relayoutWindow()
     → WMS 返回 SurfaceControl
     → Surface 从 SurfaceControl 获取 BufferQueue 连接
     → 此后 View 树绘制到 Surface 的 GraphicBuffer 中
```

### 1.3 持有关系：谁持有谁

```
Activity
  └── mWindow: PhoneWindow
        ├── mDecor: DecorView (FrameLayout)
        │     └── mContentParent: ViewGroup
        │           └── 用户的 View 树（Button、TextView...）
        │
        └── (通过 WindowManager 关联)
              └── ViewRootImpl
                    ├── mView: DecorView（同上面那个）
                    ├── mSurface: Surface          ← 绘制目标
                    ├── mSurfaceControl: SurfaceControl  ← 图层句柄
                    └── mChoreographer: Choreographer     ← VSync 调度
```

**关键点**：
- Surface 和 SurfaceControl 初始为空壳，通过 `relayoutWindow()` Binder 调用 WMS 后才真正关联到 SurfaceFlinger 的 Layer

### 1.4 各层职责边界

```
┌─ Activity ─────────────────────────────────────────────────┐
│  职责: 生命周期管理、用户交互入口                              │
│                                                            │
│  ┌─ PhoneWindow ────────────────────────────────────────┐  │
│  │  职责: 窗口装饰策略（标题栏、背景、按键处理）              │  │
│  │                                                      │  │
│  │  ┌─ DecorView (FrameLayout) ──────────────────────┐  │  │
│  │  │  职责: 根 View，承载 StatusBar 占位 + 内容区域    │  │  │
│  │  │                                                │  │  │
│  │  │  ┌─ ContentParent ──────────────────────────┐  │  │  │
│  │  │  │  职责: 放置用户 setContentView 的布局      │  │  │  │
│  │  │  │                                          │  │  │  │
│  │  │  │   Button / TextView / RecyclerView ...   │  │  │  │
│  │  │  └──────────────────────────────────────────┘  │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
         │
         │ WindowManager.addView(decorView)
         ▼
┌─ ViewRootImpl ──────────────────────────────────────────────┐
│  职责: View 树与系统之间的桥梁                                │
│  - 驱动 measure / layout / draw                             │
│  - 管理 Surface 和 SurfaceControl                           │
│  - 接收 Input 事件分发给 View 树                             │
│  - 通过 IWindowSession (Binder) 与 WMS 通信                 │
│                                                             │
│  mSurface ──→ GraphicBuffer ──→ 像素写入（View 树绘制的目标）  │
│  mSurfaceControl ──→ Layer 属性（WMS/动画系统操作的目标）      │
└─────────────────────────────────────────────────────────────┘
         │ Binder                        │ Binder
         ▼                               ▼
┌─ WMS (SystemServer) ──┐    ┌─ SurfaceFlinger ──────────────┐
│  WindowState           │    │  Layer                        │
│  - 窗口策略管理         │    │  - 元数据: position/matrix/... │
│  - 动画调度             │    │  - buffer: GraphicBuffer      │
│  - SurfaceControl 创建  │    │  - 合成 → 上屏                │
└────────────────────────┘    └──────────────────────────────┘
```

### 1.5 数据流：一帧画面如何从 View 到屏幕

```
                        App 进程
    ┌──────────────────────────────────────────┐
    │                                          │
    │  ① View.invalidate()                     │
    │     → ViewRootImpl.scheduleTraversals()  │
    │     → Choreographer 等待 VSync            │
    │                                          │
    │  ② VSync 到来                             │
    │     → performMeasure()  测量              │
    │     → performLayout()   布局              │
    │     → performDraw()     绘制              │
    │                                          │
    │  ③ View 树绘制到 Surface                  │
    │     → Surface.dequeueBuffer()            │
    │       获取空闲 GraphicBuffer              │
    │     → RenderThread 执行 DisplayList       │
    │       将像素写入 GraphicBuffer             │
    │     → Surface.queueBuffer()              │
    │       提交 buffer 到 BufferQueue          │
    │                                          │
    └──────────────┬───────────────────────────┘
                   │ BufferQueue（共享内存）
                   ▼
    ┌──────────────────────────────────────────┐
    │          SurfaceFlinger 进程              │
    │                                          │
    │  ④ onFrameAvailable() 收到新 buffer       │
    │                                          │
    │  ⑤ VSync 到来 → 开始合成                  │
    │     → commitTransactions()               │
    │       应用 SurfaceControl 的元数据变更     │
    │     → latchBuffer()                      │
    │       锁定各 Layer 最新 buffer             │
    │     → composite()                        │
    │       按 Z 序合成所有 Layer                │
    │                                          │
    │  ⑥ 通过 HWComposer 送显                  │
    └──────────────┬───────────────────────────┘
                   │
                   ▼
              ┌──────────┐
              │  屏幕显示  │
              └──────────┘
```

### 1.6 常见误区澄清

| 误区 | 实际情况 |
|------|---------|
| Activity 就是界面 | Activity 是管理者，界面是 DecorView + View 树 |
| Window 是一个可见的东西 | Window (PhoneWindow) 是策略对象，不可见。可见的是 DecorView |
| 每个 View 有自己的 Surface | 错。一个 ViewRootImpl 一个 Surface，整棵 View 树共享同一个 Surface |
| Surface 每帧重新创建 | 错。Surface 在 Window 生命周期内不变，变的是内部 GraphicBuffer 的像素内容 |
| SurfaceControl 用于绘制 | 错。SurfaceControl 只控制图层元数据（怎么摆），绘制通过 Surface 完成 |
| View 动画操作 SurfaceControl | 错。View 动画操作 RenderNode/Canvas，重绘到 Surface 的 buffer |
| Window 动画重绘内容 | 错。Window 动画只操作 SurfaceControl 属性，内容（buffer）不变 |

### 1.7 关系总览图

```
Activity ──1:N──→ Window (主窗口 + Dialog + PopupWindow + Toast ...)
                    │
                  1:1
                    ▼
               ViewRootImpl ──1:1──→ Surface ──1:1──→ SurfaceControl ──1:1──→ Layer
                    │                   │
                  1:1                 1:1
                    ▼                   ▼
                DecorView          BufferQueue
                    │                   │
                  1:N                 1:N（通常 2~3 个）
                    ▼                   ▼
              View 树（N 个 View）   GraphicBuffer
              全部画到同一个 Surface
```

---

## 第二章：一个 Activity 可以添加多个 Window 吗？

### 2.1 结论：可以。一个 Activity 可以拥有多个 Window。

### 2.2 源码依据

`WindowManagerGlobal.java` 内部用 ArrayList 管理所有 Window：

```java
// WindowManagerGlobal.java line 173-187
private final ArrayList<View> mViews = new ArrayList<>();           // 所有窗口的根 View
private final ArrayList<ViewRootImpl> mRoots = new ArrayList<>();   // 所有 ViewRootImpl
private final ArrayList<WindowManager.LayoutParams> mParams = new ArrayList<>(); // 所有窗口参数
```

每次调用 `WindowManager.addView()` 都会往这三个列表中添加一项，每项对应一个独立的 Window。

### 2.3 实际场景

```
Activity
├── 主窗口 (TYPE_APPLICATION)          ← Activity 自身
│   └── ViewRootImpl #1 → Surface #1 → SurfaceControl #1
│
├── Dialog (TYPE_APPLICATION_SUB_PANEL) ← new AlertDialog().show()
│   └── ViewRootImpl #2 → Surface #2 → SurfaceControl #2
│
├── PopupWindow (TYPE_APPLICATION_PANEL) ← popupWindow.showAtLocation()
│   └── ViewRootImpl #3 → Surface #3 → SurfaceControl #3
│
└── Toast (TYPE_TOAST)                  ← Toast.makeText().show()
    └── ViewRootImpl #4 → Surface #4 → SurfaceControl #4
```

**每个 Window 独立拥有**：
- 自己的 PhoneWindow（Dialog 场景）或直接 addView（PopupWindow 场景）
- 自己的 DecorView / 根 View
- 自己的 ViewRootImpl
- 自己的 Surface 和 SurfaceControl
- 在 SurfaceFlinger 中对应独立的 Layer

### 2.4 各 Window 的创建入口

| 类型 | 创建方式 | 源码关键行 |
|------|---------|-----------|
| **Activity 主窗口** | `Activity.attach()` 创建 PhoneWindow，resume 时 `wm.addView(mDecor)` | Activity.java:9436 |
| **Dialog** | 构造函数创建独立 PhoneWindow，`show()` 时 `mWindowManager.addView(mDecor)` | Dialog.java:227, 375 |
| **PopupWindow** | 不创建 PhoneWindow，直接 `mWindowManager.addView(decorView, p)` | PopupWindow.java:1593 |
| **Toast** | ToastPresenter 创建 View，`windowManager.addView(mView, mParams)` | ToastPresenter.java:370 |

### 2.5 主窗口与子窗口的区别

| 维度 | 主窗口 | 子窗口（Dialog/PopupWindow） |
|------|--------|---------------------------|
| 创建时机 | Activity.attach() | 运行时动态创建 |
| Window 类型 | TYPE_APPLICATION | TYPE_APPLICATION_SUB_PANEL 等 |
| 是否有 PhoneWindow | 是 | Dialog 有，PopupWindow 没有 |
| 生命周期 | 跟随 Activity | 独立管理（show/dismiss） |
| 在 WMS 中 | 独立的 WindowState | 独立的 WindowState，但有 parentWindow 关联 |
| 在 SurfaceFlinger 中 | 独立 Layer | 独立 Layer |

---

## 第三章：Android 桌面布局中，所有图标都属于一个 Surface 吗？

### 3.1 结论：是的。桌面上所有图标共享同一个 Surface。

### 3.2 原因

Launcher 是一个**单 Activity 应用**，所有图标都是这个 Activity View 树中的普通 View（`BubbleTextView`），它们画到同一个 Surface 的 GraphicBuffer 上。

### 3.3 Launcher 的 View 层级结构

源码位置：`packages/apps/Launcher3/`

```
Launcher (Activity) ── 一个 Activity，一个主 Surface
  └── LauncherRootView
        └── DragLayer（拖拽坐标管理层）
              ├── Workspace（PagedView，可左右翻页）
              │     ├── CellLayout #0（第1页桌面）
              │     │     ├── BubbleTextView（图标：微信）     ← 普通 View
              │     │     ├── BubbleTextView（图标：支付宝）   ← 普通 View
              │     │     ├── BubbleTextView（图标：相机）     ← 普通 View
              │     │     └── ...
              │     ├── CellLayout #1（第2页桌面）
              │     │     └── ...
              │     └── CellLayout #2（第3页桌面）
              │           └── ...
              ├── Hotseat（底部固定栏）
              │     └── BubbleTextView × N
              ├── PageIndicator（页码指示器）
              └── SearchBar 等
```

**关键点**：`BubbleTextView` 继承自 `TextView`，是最普通的 View。所有图标都在同一棵 View 树中，共享 Launcher Activity 的那一个 Surface。

### 3.4 哪些情况会产生额外的 Surface？

| 场景 | 是否额外 Surface | 原因 |
|------|-----------------|------|
| 桌面上的 App 图标 | 否，共享主 Surface | 普通 View（BubbleTextView） |
| 长按弹出菜单 | 否，共享主 Surface | AbstractFloatingView 加入 DragLayer（同一 View 树） |
| 打开文件夹 | 否，共享主 Surface | Folder 继承 AbstractFloatingView（同一 View 树） |
| 桌面小部件（AppWidget） | **可能是** | 如果小部件内部使用了 SurfaceView 则会创建独立 Surface |
| 动态壁纸 | **是** | WallpaperService 有自己的 Surface |
| 手势导航动画 | **是** | FloatingSurfaceView 内嵌 SurfaceView |

### 3.5 SurfaceFlinger 视角：桌面的 Layer 结构

```
┌─ Layer: 壁纸 ──────────────────────────────────┐  z=0
│  WallpaperService 的 Surface                    │
│  (动态壁纸有独立 Surface，静态壁纸也有)            │
└─────────────────────────────────────────────────┘

┌─ Layer: Launcher 主窗口 ───────────────────────┐  z=1
│  Launcher Activity 的 Surface                   │
│                                                │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐                  │
│  │微信│ │支付宝│ │相机│ │设置│  ← 全部是像素     │
│  └────┘ └────┘ └────┘ └────┘    画在同一个      │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐    GraphicBuffer   │
│  │图标│ │图标│ │图标│ │图标│    里的             │
│  └────┘ └────┘ └────┘ └────┘                   │
│                                                │
│  SurfaceFlinger 看不到单个图标，                  │
│  它只看到一整张合成好的"桌面图"                    │
└────────────────────────────────────────────────┘

┌─ Layer: 状态栏 ────────────────────────────────┐  z=2
│  SystemUI StatusBar 的 Surface                  │
└─────────────────────────────────────────────────┘

┌─ Layer: 导航栏 ────────────────────────────────┐  z=3
│  SystemUI NavigationBar 的 Surface              │
└─────────────────────────────────────────────────┘
```

**SurfaceFlinger 的视角**：它看不到"微信图标"或"支付宝图标"这样的概念。它只看到 Launcher 的 Layer 提交了一个 GraphicBuffer，里面是一整张已经画好所有图标的位图。单个图标的绘制发生在 App 进程的 View 树 draw 阶段，对 SurfaceFlinger 完全透明。

### 3.6 特殊情况：SurfaceView 打破"一个 View 树一个 Surface"的规则

普通 View 共享所在 ViewRootImpl 的 Surface，但有两个例外：

| 特殊 View | 行为 | 典型用途 |
|-----------|------|---------|
| **SurfaceView** | 创建独立的 Surface，在主 Surface 上"挖洞" | 视频播放、相机预览、游戏 |
| **TextureView** | 创建独立的 SurfaceTexture，但最终合并到主 Surface 的 RenderNode 中 | 视频缩略图、动画 |

如果 Launcher 的某个小部件内部使用了 SurfaceView，那个小部件会有自己独立的 Surface 和 Layer，但桌面图标本身不会。

---

## 第四章：通过 WindowManager.addView 添加两个 View 会异常吗？

### 4.1 两个不同的 View 实例：不会异常

```java
View viewA = new View(context);
View viewB = new View(context);
windowManager.addView(viewA, paramsA);  // OK
windowManager.addView(viewB, paramsB);  // OK，创建第二个 Window
```

完全合法，每次 `addView` 创建独立的 ViewRootImpl、Surface、SurfaceControl。

### 4.2 同一个 View 实例添加两次：抛异常

```java
View view = new View(context);
windowManager.addView(view, paramsA);  // OK
windowManager.addView(view, paramsB);  // ❌ IllegalStateException
```

源码依据（`WindowManagerGlobal.java:593-601`）：

```java
int index = findViewLocked(view, false);
if (index >= 0) {
    if (mDyingViews.contains(view)) {
        mRoots.get(index).doDie();
    } else {
        throw new IllegalStateException("View " + view
                + " has already been added to the window manager.");
    }
}
```

框架在 `mViews` 列表中检测到该 View 已存在，一个 View 实例不能同时属于两个 Window。

### 4.3 异常场景汇总

| 场景 | 是否异常 | 原因 |
|------|---------|------|
| 两个不同 View 实例各自 addView | 正常 | 各自创建独立 Window |
| 同一个 View 实例 addView 两次 | `IllegalStateException` | View 已被添加 |
| View 为 null | `IllegalArgumentException` | 空值检查 |
| 无 `SYSTEM_ALERT_WINDOW` 权限用 `TYPE_APPLICATION_OVERLAY` | WMS 拒绝 | 权限不足 |
| 子窗口类型但 token 找不到父窗口 | WMS 返回错误码 | 父窗口不存在 |

### 4.4 在同一个 Activity 中 addView 两次 = 一个 Activity 对应三个 Window

```java
// 在同一个 Activity 中
View viewA = new View(this);
View viewB = new View(this);
WindowManager wm = getWindowManager();
wm.addView(viewA, paramsA);  // Window #2
wm.addView(viewB, paramsB);  // Window #3
```

此时的完整结构：

```
Activity
├── 主窗口 (Activity 自带的 PhoneWindow)
│   └── ViewRootImpl #1 → Surface #1 → SurfaceControl #1 → Layer #1
│
├── Window #2 (addView viewA 产生)
│   └── ViewRootImpl #2 → Surface #2 → SurfaceControl #2 → Layer #2
│
└── Window #3 (addView viewB 产生)
    └── ViewRootImpl #3 → Surface #3 → SurfaceControl #3 → Layer #3
```

日常开发中一直在这样做：

```java
new AlertDialog.Builder(this).show();     // +1 Window
new PopupWindow().showAtLocation(...);    // +1 Window
Toast.makeText(this, "hi", ...).show();   // +1 Window
```

**Activity 和 Window 从来就不是 1:1，而是 1:N。**

---

## 源码路径索引

| 组件 | 文件路径 |
|------|---------|
| Activity | `frameworks/base/core/java/android/app/Activity.java` |
| Window | `frameworks/base/core/java/android/view/Window.java` |
| PhoneWindow | `frameworks/base/core/java/com/android/internal/policy/PhoneWindow.java` |
| DecorView | `frameworks/base/core/java/com/android/internal/policy/DecorView.java` |
| ViewRootImpl | `frameworks/base/core/java/android/view/ViewRootImpl.java` |
| WindowManagerGlobal | `frameworks/base/core/java/android/view/WindowManagerGlobal.java` |
| IWindowSession | `frameworks/base/core/java/android/view/IWindowSession.aidl` |
| Dialog | `frameworks/base/core/java/android/app/Dialog.java` |
| PopupWindow | `frameworks/base/core/java/android/widget/PopupWindow.java` |
| ToastPresenter | `frameworks/base/core/java/android/widget/ToastPresenter.java` |
| Window 类型定义 | `frameworks/base/core/java/android/view/WindowManager.java` |
| Launcher Activity | `packages/apps/Launcher3/src/com/android/launcher3/Launcher.java` |
| 桌面图标 View | `packages/apps/Launcher3/src/com/android/launcher3/BubbleTextView.java` |
| 桌面工作区 | `packages/apps/Launcher3/src/com/android/launcher3/Workspace.java` |
| 浮动视图基类 | `packages/apps/Launcher3/src/com/android/launcher3/AbstractFloatingView.java` |