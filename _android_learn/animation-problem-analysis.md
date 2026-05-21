---
layout: private_post
title: 'Android 动画问题排查思路'
date: 2026-05-21
categories: android-learn
tags: [animation, debugging, wm]
render_with_liquid: false
---

# 动画的一些问题思路

# **1.activity可能是透明的**

对于一个WindowContainer来说，可以通过fillsParent来判断是否是透明的activity。

路径：frameworks/base/services/core/java/com/android/server/wm/WindowContainer.java

```java
   /**
     * Returns true if this container is opaque and fills all the space made available by its parent
     * container.
     *
     * NOTE: It is possible for this container to occupy more space than the parent has (or less),
     * this is just a signal from the client to window manager stating its intent, but not what it
     * actually does.
     */
    boolean fillsParent() {
        return false;
}
```



# **2.桌面点击app无缩放动画**

## 2.1 android T以及之前版本

**开启wm中remoteanimation相关日志**

```

adb shell wm logging enable-text WM_DEBUG_REMOTE_ANIMATIONS WM_DEBUG_APP_TRANSITIONS WM_DEBUG_APP_TRANSITIONS_ANIM
```

 

**过滤关键字LauncherAnimationRunner查看桌面是否收到我们的回调**

```
adb logcat -c && adb logcat | grep -iE "windowmanager:|applyanimation:|LauncherAnimationRunner: "
```

 

**如果打印出来，说明桌面已经接受到了我们的回调，但是还是没有动画，就说明桌面存在问题**

```
07-19 16:22:57.454  2891  5102 E LauncherAnimationRunner: onAnimationStart
07-19 16:22:57.455  2891  5102 E LauncherAnimationRunner: onAnimationStart:   mode=1   taskId=2   isTranslucent=false
07-19 16:22:57.455  2891  5102 E LauncherAnimationRunner: onAnimationStart:   mode=0   taskId=98   isTranslucent=true
```

## 2.2 android T之后版本

打开shelltransition相关开关

```
adb shell wm logging enable-text WM_DEBUG_APP_TRANSITIONS WM_DEBUG_APP_TRANSITIONS_ANIM WM_DEBUG_WINDOW_TRANSITIONS WM_DEBUG_SYNC_ENGINE
```

过滤关键字    

```
adb logcat -c && adb logcat | grep -iE "windowmanager:|windowmanagershell:|BLASTSyncEngine:|ShellRecents"
```



| 作用                           | 关键字                            | 类名                                   |
| ------------------------------ | --------------------------------- | -------------------------------------- |
| 查看桌面拿到动画的target信息   | LauncherAnimationRunner:          | LauncherAnimationRunner.java           |
| 查看recents动画的一些信息      | ShellRecents                      | ShellProtoLogGroup.java（shell侧）     |
| recents动画的一些信息          | RecentsAnimationListenerImpl      | RecentsAnimationListenerImpl.java      |
| 点击桌面图标以及动画的一些信息 | QuickstepAppTransitionManagerImpl | QuickstepAppTransitionManagerImpl.java |

# 3.哪些是recentsanimation和remoteanimation

## 3.1 remoteanimation

常见的remoteanimation操作

**1.** **从桌面点击一个****app**

**2.全面屏手势：从侧边返回**

**3.经典导航键：返回键（回桌面才是）、home键、最近任务键**

**4.最近任务进入****app**



## 3.2 recentsanimation

**全面屏上滑（目前知道的只有这个）**

1）上滑回桌面

2）上滑进最近任务

3）上滑



# 4.闪黑问题

- 可以通过抓winscope来看，如果winscope抓不到可以连续dumpsys surfaceflinger信息
- adb shell dumpsys SurfaceFlinger	#查看屏幕上显示的图层
- adb shell "while true; do dumpsys SurfaceFlinger; sleep 0.016;done" > dump_SF.log  //连续dumpsys SurfaceFlinger

# 5. **T上change编译时存在依赖（miui和framework）**

https://gerrit.pt.mioffice.cn/c/miui/frameworks/base/+/2285600

http://gerrit.bsp.xiaomi.srv/c/platform/frameworks/base/+/47653

添加统一的Topic，然后再触发Reply中的Presubmit-Ready。这个跟S版本上的depend change号是一样的



# 6. **锁屏解锁闪黑问题**

**SystemUI使用setWallpaperAsTarget接口确保解锁不因为壁纸消失而闪黑，有一个前提条件是需要SystemUI在过渡动画开始后调用setWallpaperAsTarget(false)。**

GOOD TO GO表示过渡动画开始，因此要想不闪黑，需要在执行过渡动画开始后systemui再进行设置setWallpaperAsTarget(false)。如果在GOOD TO GO之前设置或者先调用该函数，就会出现闪黑。

adb shell "while true; do dumpsys window | grep -Ei "mWallpaperTarget"; sleep 0.016;done"

 

**闪黑**的mWallpaperTarget=null，系统不显示壁纸，会导致存在几帧是黑色的。

**锁屏**的mWallpaperTarget=Window{39bbe37 u0 **NotificationShade**}

**桌面**的 mWallpaperTarget=Window{ae093ea u0 com.miui.home/com.miui.home.launcher.Launcher}

**非桌面和锁屏**的mWallpaperTarget=null



**可以打开动画的开关**

```
adb shell dumpsys window logging
adb shell wm logging enable-text WM_DEBUG_APP_TRANSITIONS WM_DEBUG_APP_TRANSITIONS_ANIM WM_DEBUG_REMOTE_ANIMATIONS WM_DEBUG_RECENTS_ANIMATIONS
adb logcat -c && adb logcat | grep -iE "good to go|setWallPaperAsTarget"
```



过滤关键字：good to go和setWallPaperAsTarget





# 7.壁纸相关问题处理

打开壁纸的动态log开关：

___________________________________________________________________________

**adb shell wm miuilogging enable DEBUG_WALLPAPER**

___________________________________________________________________________

查看WindowManager: Wallpaper visibility: (false|true)来确认解锁过程中壁纸是否有消失过。



# 8.Android中AIDL oneway用法

参考：
[android aidl oneway用法](https://unbroken.blog.csdn.net/article/details/73292012?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2~default~CTRLIST~Rate-1-73292012-blog-113652783.pc_relevant_multi_platform_featuressortv2dupreplace&depth_1-utm_sou)
[关于Binder （AIDL）的 oneway 机制](https://blog.csdn.net/Jason_Lee155/article/details/116637925?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2~default~TopNSimilar~default-1-116637925-blog-73292012.topnsimilarv1&depth_1-utm_source=distribute.pc_relevant)



# 9.android中的Matrix

参考：[Android Matrix矩阵](https://www.jianshu.com/p/c83f59613c18)





# 10.生命周期中的system_server进程和app 进程

**system_server进程：**am_create_activity，am_restart_activity，am_set_resumed_activity类似这种后面有一个activity的
**app进程：**am_on_create_called，am_on_start_called，am_on_resume_called类似这种后面有一个called



# 11.动画的资源文件

路径：frameworks/base/core/res/res/anim



# 12.android中的overlay机制

参考：[https://blog.csdn.net/qq_28837389/article/details/105637737](https://blog.csdn.net/qq_28837389/article/details/105637737)
Android Overlay是一种资源替换机制，它能在不重新打包apk的情况下，实现资源文件的替换
例如：packages/apps/MiuiSystemUI/packages/Overlays/systemui-devices-android/res/anim/lock_screen_behind_enter_scale.xml



# 13.app进程

frameworks/base/core/java/android/app/Activity.java
在frameworks/base/core/java/android/app路径下的都是app进程

# 14.一些app的layout文件路径

对应app中的layout的xml文件路径
例如接听来电：packages/apps/InCallUI/res/anim/incall_open_enter.xml



# 15.桌面相关日志过滤

```
adb logcat | grep RecentsModel
adb logcat -c && adb logcat | grep -iE "MIUIInput:|Launcher:|NavStubView:|8373"
```

**查看网址：**[https://opengrok.pt.xiaomi.com/opengrok2/xref/miui-separate-app-alpha/](https://opengrok.pt.xiaomi.com/opengrok2/xref/miui-separate-app-alpha/)
**关键字：**

| 搜索关键字                   | 解释                                                    |
| ---------------------------- | ------------------------------------------------------- |
| NavStubView:                 | 手势                                                    |
| Launcher:                    | 桌面                                                    |
| MIUIInput:                   | input                                                   |
| Launcher.itemIcon: folmeDown | 触摸（点击）                                            |
| LauncherAnimationRunner:     | 桌面接收到fw（remoteanimation）的onAnimationStart的回调 |



**相关路径：**core/java/android/view/MotionEvent.java

打印的log

```
//action的值便是相应的动作

//MIUIINPUT日志
11-17 23:24:21.517  1919  2047 D MIUIInput: [MotionEvent] publisher action=0x0, 2158599, channel '9d96f8b GestureStubHome (server)'
11-17 23:24:21.518  1919  2047 D MIUIInput: [MotionEvent] publisher action=0x0, 2158599, channel '[Gesture Monitor] miui-gesture (server)'

//桌面打印日志
11-17 23:24:21.519  8373  8373 D NavStubView: onTouchEvent: action=0, count=1
11-17 23:24:21.520  8373  8373 D NavStubView: onTouch ACTION_DOWN sendMessageDelayed MSG_CHECK_GESTURE_STUB_TOUCHABLE
11-17 23:24:21.579  8373  8373 D NavStubView: onTouchEvent: action=2, count=1

11-17 23:24:21.582  8373  8373 D NavStubView: onPointEvent: rawX=496.30002   rawY=2459.0   action=0   mIsInFsMode=false

//桌面打印日志，获取所有的runningtask
11-17 23:24:21.582  8373  8373 W RecentsModel: getRunningTaskContainHome
11-17 23:24:21.584  8373  8373 D RecentsModel: logTasks: type_all, [TaskInfo{userId=0 taskId=61 displayId=0 isRunning=true baseIntent=Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10200000 cmp=com.android.settings/.MainSettings } baseActivity=ComponentInfo{com.android.settings/com.android.settings.MainSettings} topActivity=ComponentInfo{com.android.settings/com.android.settings.MiuiSettings} origActivity=ComponentInfo{com.android.settings/com.android.settings.MainSettings} realActivity=ComponentInfo{com.android.settings/com.android.settings.MiuiSettings} numActivities=1 lastActiveTime=3614682 supportsSplitScreenMultiWindow=true supportsMultiWindow=true resizeMode=1 isResizeable=true minWidth=-1 minHeight=-1 defaultMinSize=200 token=WCT{android.window.IWindowContainerToken$Stub$Proxy@ea5a3a0} topActivityType=1 pictureInPictureParams=null shouldDockBigOverlays=false launchIntoPipHostTaskId=0 displayCutoutSafeInsets=Rect(0, 100 - 0, 0) topActivityInfo=ActivityInfo{2219b59 com.android.settings.MainSettings} launchCookies=[] positionInParent=Point(0, 0) parentTaskId=-1 isFocused=true isVisible=true isSleeping=false topActivityInSizeCompat=false topActivityInMiuiSizeCompat=false topActivityEligibleForLetterboxEducation= false locusId=null displayAreaFeatureId=1 cameraCompatControlState=hidden mIsCastMode=false}, TaskInfo{userId=0 taskId=60 displayId=0 isRunning=true baseIntent=Intent { act=android.intent.action.MAIN cat=[android.intent.category.HOME] flg=0x10800100 cmp=com.miui.home/.launcher.Launcher } baseActivity=ComponentInfo{com.miui.home/com.miui.home.launcher.Launcher} topActivity=ComponentInfo{com.miui.home/com.miui.home.launcher.Launcher} origActivity=null realActivity=ComponentInfo{com.miui.home/com.miui.home.launcher.Launcher} numActivities=1 lastActiveTime=3611990 supportsSplitScreenMultiWindow=true supportsMultiWindow=true resizeMode=2 isResizeable=true minWidth=-1 minHeight=-1 defaultMinSize=200 token=WCT{android.window.IWindowContainerToken$Stub$Proxy@57a731e} topActivityType=2 pictureInPictureParams=null shouldDockBigOverlays=false launchIntoPipHostTaskId=0 displayCutoutSafeInsets=Rect(0, 100 - 0, 0) topActivityInfo=ActivityInfo{b6a57ff com.miui.home.launcher.Launcher} launchCookies=[] positionInParent=Point(0, 0) parentTaskId=-1 isFocused=false isVisible=false isSleeping=false topActivityInSizeCompat=false topActivityInMiuiSizeCompat=false topActivityEligibleForLetterboxEducation= false locusId=null displayAreaFeatureId=1 cameraCompatControlState=hidden mIsCastMode=false}, TaskInfo{userId=0 taskId=64 displayId=0 isRunning=true baseIntent=Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10200000 cmp=com.xiaomi.gamecenter/.ui.MainTabActivity } baseActivity=ComponentInfo{com.xiaomi.gamecenter/com.xiaomi.gamecenter.ui.MainTabActivity} topActivity=ComponentInfo{com.xiaomi.gamecenter/com.xiaomi.gamecenter.ui.MainTabActivity} origActivity=null realActivity=ComponentInfo{com.xiaomi.gamecenter/com.xiaomi.gamecenter.ui.MainTabActivity} numActivities=1 lastActiveTime=3418331 supportsSplitScreenMultiWindow=false supportsMultiWindow=true resizeMode=2 isResizeable=true minWidth=-1 minHeight=-1 defaultMinSize=200 token=WCT{android.window.IWindowContainerToken$Stub$Proxy@15159cc} topActivityType=1 pictureInPictureParams=null shouldDockBigOverlays=false launchIntoPipHostTaskId=0 displayCutoutSafeInsets=Rect(0, 100 - 0, 0) topActivityInfo=ActivityInfo{8c79415 com.xiaomi.gamecenter.ui.MainTabActivity} launchCookies=[] positionInParent=Point(0, 0) parentTaskId=-1 isFocused=false isVisible=false isSleeping=false topActivityInSizeCompat=false topActivityInMiuiSizeCompat=false topActivityEligibleForLetterboxEducation= false locusId=null displayAreaFeatureId=1 cameraCompatControlState=hidden mIsCastMode=false}, TaskInfo{userId=0 taskId=68 displayId=0 isRunning=true baseIntent=Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10200000 cmp=com.miui.notes/.ui.NotesListActivity } baseActivity=ComponentInfo{com.miui.notes/com.miui.not

```



## 15.1 手势NavStubView值

| action         | value | 作用     |
| -------------- | ----- | -------- |
| ACTION_DOWN    | 0     | 手势按下 |
| ACTION_UP      | 1     | 手势起来 |
| ACTION_MOVE    | 2     | 手势移动 |
| ACTION_CANCEL  | 3     | 手势取消 |
| ACTION_OUTSIDE | 4     |          |



## 15.2 桌面点击app响应关键字

```
01-13 20:49:23.971 14022 14022 D Launcher.itemIcon: folmeDown
01-13 20:49:24.029 14022 14022 D Launcher.itemIcon: folmeUp
```







# 16.getClass()去查看调用的是哪一个类

可以使用getClass()去查看调用的是哪一个类，通常判断是走的父类还是某一个类，实际调试时候可能走了子类的逻辑没走父类的逻辑，可以查看class是哪一个。





# 17.handleAppTransitionReady中获取transit值

transit、openingAppsForAnimation以及closingAppsForAnimation初值在这里面获取

```JAVA
 void handleAppTransitionReady() {
 			......
                        final boolean excludeLauncherFromAnimation =
                    		mDisplayContent.mOpeningApps.stream().anyMatch(
                            		(app) -> app.isAnimating(PARENTS, ANIMATION_TYPE_RECENTS))
                    		|| mDisplayContent.mClosingApps.stream().anyMatch(
                            		(app) -> app.isAnimating(PARENTS, ANIMATION_TYPE_RECENTS));
           				 final ArraySet<ActivityRecord> openingAppsForAnimation = getAppsForAnimation(
                  								 mDisplayContent.mOpeningApps, excludeLauncherFromAnimation);
            			final ArraySet<ActivityRecord> closingAppsForAnimation = getAppsForAnimation(
                    							mDisplayContent.mClosingApps, excludeLauncherFromAnimation);

                    @TransitionOldType final int transit = getTransitCompatType(
                                mDisplayContent.mAppTransition, openingAppsForAnimation, closingAppsForAnimation,
                                mDisplayContent.mChangingContainers,
                                mWallpaperControllerLocked.getWallpaperTarget(), getOldWallpaper(),
                                mDisplayContent.mSkipAppTransitionAnimation);
     
     				......
                        
                     applyAnimations(openingAppsForAnimation, closingAppsForAnimation, transit, animLp,
                                     voiceInteraction);
                     handleClosingApps();
                     handleOpeningApps();
                     handleChangingApps(transit);
     
     				......
 }

                
                
```





# 18.SurfaceFlinger中surface的创建

WindowManagerService.relayoutWindow.createSurfaceControl调用到WindowStateAnimator.createSurfaceLocked中去创建，并设置 **mDrawState=DRAW_PENDING**。

**路径：WindowStateAnimator.java**



```java
WindowSurfaceController createSurfaceLocked() {
        final WindowState w = mWin;

        if (mSurfaceController != null) {
            return mSurfaceController;
        }

        w.setHasSurface(false);

        ProtoLog.i(WM_DEBUG_ANIM, "createSurface %s: mDrawState=DRAW_PENDING", this);

        resetDrawState();

        mService.makeWindowFreezingScreenIfNeededLocked(w);

        int flags = SurfaceControl.HIDDEN;
        final WindowManager.LayoutParams attrs = w.mAttrs;

        if (w.isSecureLocked()) {
            flags |= SurfaceControl.SECURE;
        }

        if ((mWin.mAttrs.privateFlags & PRIVATE_FLAG_IS_ROUNDED_CORNERS_OVERLAY) != 0
                || WindowStateAnimatorStub.get().isAllowedDisableScreenshot(mWin)) {
            flags |= SurfaceControl.SKIP_SCREENSHOT;
        }

        if (DEBUG_VISIBILITY) {
            Slog.v(TAG, "Creating surface in session "
                    + mSession.mSurfaceSession + " window " + this
                    + " format=" + attrs.format + " flags=" + flags);
        }

        // Set up surface control with initial size.
        try {

            // This can be removed once we move all Buffer Layers to use BLAST.
            final boolean isHwAccelerated = (attrs.flags & FLAG_HARDWARE_ACCELERATED) != 0;
            final int format = isHwAccelerated ? PixelFormat.TRANSLUCENT : attrs.format;

            mSurfaceController = new WindowSurfaceController(attrs.getTitle().toString(), format,
                    flags, this, attrs.type);
            mSurfaceController.setColorSpaceAgnostic((attrs.privateFlags
                    & WindowManager.LayoutParams.PRIVATE_FLAG_COLOR_SPACE_AGNOSTIC) != 0);

            w.setHasSurface(true);
            // The surface instance is changed. Make sure the input info can be applied to the
            // new surface, e.g. relaunch activity.
            w.mInputWindowHandle.forceChange();

            ProtoLog.i(WM_SHOW_SURFACE_ALLOC,
                        "  CREATE SURFACE %s IN SESSION %s: pid=%d format=%d flags=0x%x / %s",
                        mSurfaceController, mSession.mSurfaceSession, mSession.mPid, attrs.format,
                        flags, this);
        } catch (OutOfResourcesException e) {
            Slog.w(TAG, "OutOfResourcesException creating surface");
            mService.mRoot.reclaimSomeSurfaceMemory(this, "create", true);
            mDrawState = NO_SURFACE;
            return null;
        } catch (Exception e) {
            Slog.e(TAG, "Exception creating surface (parent dead?)", e);
            mDrawState = NO_SURFACE;
            return null;
        }


        if (DEBUG) Slog.v(TAG, "Created surface " + this);
        return mSurfaceController;
    }
```



# 19.android系统中的Bounds

| bounds       | 解释                                                         |
| ------------ | ------------------------------------------------------------ |
| SourceBounds | Layer在没有transformation和crop下的，在自身坐标空间的原始bounds |
| Bounds       | SourceBounds经过crop后的在自身坐标控件的bounds               |
| ScreenBounds | 在屏幕坐标空间的bounds（实际显示的）                         |

具体可参考：[Winscope中layer状态解读](https://wiki.n.miui.com/pages/viewpage.action?pageId=674170999)



# 20.在android源码中添加一个资源文件

通常是在添加到下面的res路径下：

**路径：**frameworks/base/core/res/res/drawable/XXXXX.xml

并在symbols中添加进去：

**路径：**frameworks/base/core/res/res/values/symbols.xml

```java
  <java-symbol type="dimen" name="status_bar_icon_size" />
  <java-symbol type="drawable" name="scrubber_progress_horizontal_holo_dark" />
  <java-symbol type="string" name="chooseUsbActivity" />
```

这样我们使用R.drawable.XXXXX可以识别到



# 21.全面屏侧边手势返回

**打开开关**

```
adb shell wm logging enable-text WM_DEBUG_BACK_PREVIEW
```

**过滤关键字**

```
adb logcat -c && adb logcat | grep -iE "CoreBackPreview"
```



# 22.configChanged

打开config的Protolog开关

```
adb shell wm logging enable-text WM_DEBUG_CONFIGURATION
```

可以从ActivityRecord.shouldRelaunchLocked入手



# 23.开启隐藏的api权限

有些app新添加的api是hide的，如果普通app想要访问，可能会没有权限，通过反射无法调用，这个时候偶需要开启访问隐藏api的权限

```
adb shell settings put global hidden_api_policy 1
```



# 24.Configuration日志打印

路径：services/core/java/com/android/server/wm/DisplayContent.java

打开相关开关

```
//WM_DEBUG_CONFIGURATION是Config相关信息
//WM_DEBUG_ORIENTATION是转屏相关信息，通常冻屏的逻辑也可以从此处查看
adb shell wm logging enable-text WM_DEBUG_CONFIGURATION WM_DEBUG_ORIENTATION
```

冻屏问题会导致无动画，走startFreezingDisplay逻辑

可以打印config信息对比，是哪些config发生变化

```java
    Configuration updateOrientation(WindowContainer<?> freezeDisplayWindow, boolean forceUpdate) {
        if (!mDisplayReady) {
            return null;
        }

        Configuration config = null;
        if (updateOrientation(forceUpdate)) {
            // If we changed the orientation but mOrientationChangeComplete is already true,
            // we used seamless rotation, and we don't need to freeze the screen.
            if (freezeDisplayWindow != null && !mWmService.mRoot.mOrientationChangeComplete) {
                final ActivityRecord activity = freezeDisplayWindow.asActivityRecord();
                if (activity != null && activity.mayFreezeScreenLocked()) {
                    activity.startFreezingScreen();
                }
            }
            config = new Configuration();
            computeScreenConfiguration(config);
        } else if (!(mTransitionController.isCollecting(this)
                // If waiting for a remote rotation, don't prematurely update configuration.
                || mDisplayRotation.isWaitingForRemoteRotation())) {
            // No obvious action we need to take, but if our current state mismatches the
            // activity manager's, update it, disregarding font scale, which should remain set
            // to the value of the previous configuration.
            // Here we're calling Configuration#unset() instead of setToDefaults() because we
            // need to keep override configs clear of non-empty values (e.g. fontSize).
            final Configuration currentConfig = getRequestedOverrideConfiguration();
			//重置mTmpConfiguration
            mTmpConfiguration.unset();
            //通过currentConfig更新mTmpConfiguration
            mTmpConfiguration.updateFrom(currentConfig);
            Slog.e("NOANIMATION"," DC " + " updateOrientation AAAAAAA "
                    + " mTmpConfiguration = " + mTmpConfiguration
                    + " currentConfig = " + currentConfig
                    + " diff = " + Configuration.configurationDiffToString(currentConfig.diff(mTmpConfiguration)));
            computeScreenConfiguration(mTmpConfiguration);
            Slog.e("NOANIMATION"," DC " + " updateOrientation 222 "
                    + " mTmpConfiguration = " + mTmpConfiguration
                    + " currentConfig = " + currentConfig
                    + " diff = " + Configuration.configurationDiffToString(currentConfig.diff(mTmpConfiguration)));
            //这里如果currentConfig和mTmpConfiguration对比不同，则会触发转屏的逻辑，会发生冻屏
            if (currentConfig.diff(mTmpConfiguration) != 0) {
                mWaitingForConfig = true;
                setLayoutNeeded();
                mDisplayRotation.prepareNormalRotationAnimation();
                config = new Configuration(mTmpConfiguration);
            }
        }

        return config;
    }
```



# 25.input window和焦点窗口更新

参考：https://tiny-isthmus-3be.notion.site/Input-window-72f96872d1f4479fa637b47c2f42e770



# 26.Surfaceflinger中可见区域

Region VisibleRegion (this=0 count=1)便是可见区域，count为1表示不可见

下面代码中表示有一个可见区域

```
+ Layer (com.android.settings/com.android.settings.MainSettings#743) uid=1000
  Region TransparentRegion (this=0 count=0)
  Region VisibleRegion (this=0 count=1)
    [-261,   0,  30, 2400]
  Region SurfaceDamageRegion (this=0 count=0)
      layerStack=   0, z=        0, pos=(-262.472,0), size=(   0,   0), crop=[  0,   0,  -1,  -1], cornerRadius=0.000000, isProtected=0, isTrustedOverlay=0, isOpaque=1, invalidate=0, dataspace=Default, defaultPixelFormat=RGBA_8888, backgroundBlurRadius=0, color=(-1.000,-1.000,-1.000,1.000), flags=0x00000102, tr=[0.00, 0.00][0.00, 0.00]
      parent=a53db94 com.android.settings/com.android.settings.MainSettings#742
      zOrderRelativeOf=none
      activeBuffer=[1080x2400:268437760,RGBA_8888], tr=[0.00, 0.00][0.00, 0.00] queued-frames=0 metadata={ownerPID:24935, ownerUID:1000, windowType:1, dequeueTime:63009855546118, 9:4bytes}, cornerRadiusCrop=[0.00, 0.00, 0.00, 0.00],  shadowRadius=0.000,
```



# 27.Android 的 Configuration

https://xiaomi.f.mioffice.cn/docx/doxk454GRfHAPUZpSINFTxKYAIc



# 28.窗口容器管理-ConfigurationContainer

https://xiaomi.f.mioffice.cn/docs/dock4ToVc6WLZmhKn4Xuaez9M9c#



# 29.HWC layer

dispFrame可以简单的理解为要显示的区域，sourceCrop可以简单的理解为是资源。

参考：https://wiki.n.miui.com/display/~fuyao8/SourceCrop+dispFrame



# 30.解锁时所加载的资源文件

路径：core/res/res/anim/lock_screen_behind_enter_subtle.xml

系统也可能会重新overlay该文件



# 31.Ubuntu增加和删除swap分区

参考：https://www.cnblogs.com/hit-tengjun/p/12044530.html

查看swap分区的位置以及使用情况：cat /proc/swaps    或者   swapon -s



# 32.查看bugreport当前那个页面在前台

搜索：mCurrentFocus



# 33.修改bash文件，source使生效

	可以在.bashrc文件中写一些默认操作，例如起别名  alias dw='adb shell dumpsys window w'
	gedit ~/.bashrc
	source ~/.bashrc



# 34.Android获取当前进程和线程ID常用方法

**参考**：https://blog.csdn.net/u012438830/article/details/88764891?spm=1001.2014.3001.5506

```
线程和进程的获取常用方式： android.os.Process
获取当前进程ID：android.os.Process.myPid()；
获取当前进程的用户ID:android.os.Process.myUid();
获取当前线程ID（1）： Thread.currentThread().getId()；
获取当前线程ID（2）： android.os.Process.myTid()；
获取应用主线程ID：Looper.getMainLooper().getThread().getId())；
```

获取进程名字和id、获取当前方法所在线程id和name、获取创建的handler的looper的名字和id

```
 Slog.e("WHICHTHERAD"," MNBC#createMiuiNarBarInternal 处理消息线程 "
                + " 当前的进程号 = " + android.os.Process.myPid()
                + " 当前的进程名称 = " + android.os.Process.myProcessName()
                + " threadName = "+ Thread.currentThread().getName()
                + " threadId = " + Thread.currentThread().getId());
                
 Slog.e("WHICHTHERAD"," MNBC#setUpFakeNavigationBar post消息线程 "
                + " 当前的进程号 = " + android.os.Process.myPid()
                + " 当前的进程名称 = " + android.os.Process.myProcessName()
                + " handler所在线程name = "+ handler.getLooper().getThread().getName()
                + " handler所在线程Id = " + handler.getLooper().getThread().getId());
```



# 35.如何知道Context是谁的Context

可以直接通过mContext.getPackageName()直接获取，通过名字可以获取到是应用的，systemui的或者services的等等

# 36.获取Activity的Window或者DecorView

对于每一个Activity来说，它都持有一个Window，因此可以通过getWindow进行获取，同理可以进一步获取DecorView

```
@Override
public void onWindowFocusChanged(boolean hasFocus) {
            super.onWindowFocusChanged(hasFocus);
            Rect rect = new Rect();
            getWindow().getDecorView().getWindowVisibleDisplayFrame(rect);
            top = rect.top;
}
```

# 37.运行CTS

1)进入CTS_U/android-cts/tools

2)执行cts-tradefed：./cts-tradefed

3)跑CTS命令



# 38.Android U自定义过渡动画  overridePendingTransition

实现：

## 38.1 xml文件  windowAnimationStyle

设置了windowAnimationStyle后，ActivityOptions中的mAnimationType值是ANIM_FROM_STYLE，不走ANIM_CUSTOM的逻辑

```
    <style name="myact">
        <item name="android:windowEnterAnimation">@anim/activity_open</item>
        <item name="android:windowExitAnimation">@anim/activity_close</item>
    </style>

    <style name="Theme.Mainww" parent="Theme.MaterialComponents.DayNight.DarkActionBar">
        <item name="android:windowAnimationStyle">@style/myact</item>
    </style>
```

## 38.2 overridePendingTransition

设置了overridePendingTransition之后，ActivityOptions中的mAnimationType值是ANIM_CUSTOM

```
    private void SwitchToApp7() {
        Intent intent = new Intent(this,MainActivity3.class);
        startActivity(intent);
        overridePendingTransition(R.anim.activity_open,R.anim.activity_close);
//        finish();
    }
```

走自定义动画的逻辑去加载动画资源

路径：libs/WindowManager/Shell/src/com/android/wm/shell/transition/DefaultTransitionHandler.java

函数：loadAnimation

```
} else if (overrideType == ANIM_CUSTOM
                && (!isTask || options.getOverrideTaskTransition())) {
            Slog.e("TRANSITIONANIMMM"," TAH#loadAnimation 走了谷歌自定义的动画 "
                    + " overrideType = " + overrideType
                    + " !isTask = " + !isTask
                    + " options.getOverrideTaskTransition() = " + options.getOverrideTaskTransition());
            a = mTransitionAnimation.loadAnimationRes(options.getPackageName(), enter
                    ? options.getEnterResId() : options.getExitResId());
            // MIUI ADD: WMS_MiuiAnimationEffect
            a = DefaultTransitionStub.getInstance().updateAnimationIfNeed(mContext, a, info, change, false);
            // END WMS_MiuiAnimationEffect
        }
```



# 39.Android U桌面相关日志

| 作用                           | 关键字                            | 类名                                   |
| ------------------------------ | --------------------------------- | -------------------------------------- |
| 查看桌面拿到动画的target信息   | LauncherAnimationRunner:          | LauncherAnimationRunner.java           |
| 查看recents动画的一些信息      | ShellRecents                      | ShellProtoLogGroup.java（shell侧）     |
| recents动画的一些信息          | RecentsAnimationListenerImpl      | RecentsAnimationListenerImpl.java      |
| 点击桌面图标以及动画的一些信息 | QuickstepAppTransitionManagerImpl | QuickstepAppTransitionManagerImpl.java |



# 40.Android U shell侧recentsanimation日志

**关键字：**ShellRecents

**shell侧protolog信息，路径：**

libs/WindowManager/Shell/src/com/android/wm/shell/protolog/ShellProtoLogGroup.java



**默认是开启的：**

```
WM_SHELL_RECENTS_TRANSITION(Consts.ENABLE_DEBUG, Consts.ENABLE_LOG_TO_PROTO_DEBUG, true,
        "ShellRecents"),
```



# 41.设置过渡动画和窗口动画

参考：[Dialog、Activity和Fragment设置切换动画](https://codeantenna.com/a/ZCTQcHgDWN)



注意：

设置activity过渡动画可以通过设置windowanimation设置，主要是通过activity获取到它的window，然后设置lp的windowanimations即可



# 42.使用git进行push

```
git push --no-thin ssh://pengmengxing@gerrit.pt.mioffice.cn:29418/platform/frameworks/base HEAD:refs/for/master-v-qcom
git push --no-thin ssh://pengmengxing@gerrit.pt.mioffice.cn:29418/miui/frameworks/base HEAD:refs/for/master-v
git push --no-thin ssh://pengmengxing@gerrit.pt.mioffice.cn:29418/miui/frameworks/interface  HEAD:refs/for/master-v
git push --no-thin ssh://pengmengxing@gerrit.pt.mioffice.cn:29418/platform/packages/apps/MiuiSystemUI  HEAD:refs/for/master-v

git push --no-thin ssh://pengmengxing@gerrit.pt.mioffice.cn:29418/device/xiaomi/peridot HEAD:refs/for/release-u-qcom-15.0.13

git push --no-thin ssh://pengmengxing@gerrit.pt.mioffice.cn:29418/platform/frameworks/base HEAD:refs/for/Feature-18133
git push --no-thin ssh://pengmengxing@gerrit.pt.mioffice.cn:29418/miui/frameworks/base HEAD:refs/for/Feature-18133
git push --no-thin ssh://pengmengxing@gerrit.pt.mioffice.cn:29418/miui/frameworks/interface  HEAD:refs/for/Feature-18133
git push --no-thin ssh://pengmengxing@gerrit.pt.mioffice.cn:29418/platform/packages/apps/MiuiSystemUI  HEAD:refs/for/Feature-18133


git checkout -b baranim21 miui/Feature-18133

make services miui-services framework-minus-apex miui-framework MiuiSystemUI -j2 && sh u_push.sh

```

使用git push 目录下device/xiaomi/common的文件system/miui_common.mk

```
git push --no-thin ssh://pengmengxing@gerrit.pt.mioffice.cn:29418/device/xiaomi/common  HEAD:refs/for/master-u
```

如果报错ERROR: commit 83ad80d: missing Change-Id in message footer就是缺乏change  id，这个时候需要创建一个change id再进行提交、按照报错的提示提交即可。

```
remote: Processing changes: refs: 1, done    
remote: ERROR: commit 83ad80d: missing Change-Id in message footer
remote: 
remote: Hint: to automatically insert a Change-Id, install the hook:
remote:   gitdir=$(git rev-parse --git-dir); scp -p -P 29418 pengmengxing@git.mioffice.cn:hooks/commit-msg ${gitdir}/hooks/
remote: (for OpenSSH >= 9.0 you need to add the flag '-O' to the scp command)
remote: or, for http(s):
remote:   f="$(git rev-parse --git-dir)/hooks/commit-msg"; curl -o "$f" https://gerrit.pt.mioffice.cn/tools/hooks/commit-msg ; chmod +x "$f"
remote: and then amend the commit:
remote:   git commit --amend --no-edit
remote: Finally, push your changes again

```

可按照下面步骤进行操作

```
第一步：创建change id
gitdir=$(git rev-parse --git-dir); scp -p -P 29418 pengmengxing@git.mioffice.cn:hooks/commit-msg ${gitdir}/hooks/
第二步：修改提交
git commit --amend
第三部：提交
git push --no-thin ssh://pengmengxing@gerrit.pt.mioffice.cn:29418/device/xiaomi/common  HEAD:refs/for/master-u
```



# 43.通过反射获取当前正在执行方法的名字

参考：https://www.volcengine.com/theme/6083160-R-7-1

```
        String methodName = new Object(){}.getClass().getEnclosingMethod().getName();
        Log.e("pmx"," 当前Activity是 " + this
                + " 当前的方法名 = " + methodName);
```



# 45.adb根据进程号查看进程的名字

```
adb shell ps | grep 当前进程号
```



# 46.让startingwindow延迟一会显示

```java
private class AddStartingWindow implements Runnable {

        @Override
        public void run() {
            // Can be accessed without holding the global lock
            final StartingData startingData;
            synchronized (mWmService.mGlobalLock) {
                // There can only be one adding request, silly caller!

                if (mStartingData == null) {
                    // Animation has been canceled... do nothing.
                    ProtoLog.v(WM_DEBUG_STARTING_WINDOW,
                            "startingData was nulled out before handling"
                                    + " mAddStartingWindow: %s", ActivityRecord.this);
                    return;
                }
                //在AddStartingWindow（实现了Runnable的run方法）中直接添加一个线程延迟500ms
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                startingData = mStartingData;
            }

        }
    }
```



# 47.获取当前进程名和pid号

```
Slog.e("SETTINGERROR"," Activity#startActivity "
        + " this = " + this
        + " 当前的进程号 = " + Process.myPid()
        + " 当前的进程名称 = " + Process.myProcessName());
```

# 48.查看Bugreport一些信息

## 48.1 查看dump window的时间

在bugreport中搜索：dump time

## 48.2确定手机ROM版本

在bugreport或者anr文件搜索：

miui version

# 49.遍历AnimationSet打印单个Animation的信息

```
        for (Animation animation : mAnimations) {
            if (animation.hasExtension()) {
                Slog.e("ZHUTIERROR"," AnimationSet$hasExtension "
                        + " animation = " + animation.getClass());
                return true;
            }
        }
        
        
        或者
        
        if(a != null) {
                if (a instanceof AnimationSet) {
                    for (Animation animation : ((AnimationSet) a).getAnimations()) {
                        Slog.d("NOANIMATION"," DTH#startAnimation "
                        + " animation = " + animation.getClass() + " duration = " + animation.getDuration());
                    }
                } else {
                        Slog.d("NOANIMATION"," DTH#startAnimation "
                        + " a = " + a.getClass() + " duration = " + a.getDuration());
                }
        }
```



# 50.在native层打印堆栈信息和log

1）添加链接库

在要加入堆栈的cpp文件中找到离它最近的Android.bp文件，在文件中的shared_libs添加"libutilscallstack",



```
shared_libs: [
    "libutils",
    "libutilscallstack",
]
```

2）在加堆栈的文件中加入头文件

```
#include <utils/CallStack.h>
```



3）在需要打印堆栈的地方 android::CallStack stack("XXXXX");

```
sp<Surface> SurfaceControl::generateSurfaceLocked()
{
    uint32_t ignore;
    auto flags = mCreateFlags & (ISurfaceComposerClient::eCursorWindow |
                                 ISurfaceComposerClient::eOpaque);
    ALOGE("Print bbq-wrapper");
    android::CallStack stack("TagXXXTag");
    mBbqChild = mClient->createSurface(String8("bbq-wrapper"), 0, 0, mFormat,
                                       flags, mHandle, {}, &ignore);
    mBbq = sp<BLASTBufferQueue>::make("bbq-adapter", mBbqChild, mWidth, mHeight, mFormat);

    // This surface is always consumed by SurfaceFlinger, so the
    // producerControlledByApp value doesn't matter; using false.
    mSurfaceData = mBbq->getSurface(true);

    return mSurfaceData;
}

```



4）编译SurfaceFlinger并push，这里因为修改了bp文件，因此需要使用make进行编译

```
make libgui -j8 && make libsurfaceflinger -j10 && sh pushSurfaceFlinger.sh


adb root && adb remount
adb push  $OUT/system/lib/libsurfaceflinger.so  system/lib
adb push  $OUT/system/lib64/libsurfaceflinger.so  system/lib64

adb push $OUT/system/lib/libgui.so system/lib/
adb push $OUT/system/lib64/libgui.so system/lib64/
adb reboot
```





# 51.app侧根据context获取WindowManager

```
    private WindowManager getWindowManager(Context context) {
        return (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
    }
```



# 52.抓Trace方法

可以直接使用Perfetto->Record new Trace根据自己的需求生成一个自己的脚本文件



# 53.grep查看某个文件夹下所有文件的某个字符串

[linux 用 grep 查找单个或多个字符串（关键字）](https://cloud.tencent.com/developer/article/1793323)

grep -iE "mdrawstate=" . -rn



过滤某一个文件中的某个字符串

grep -iE "windowmanagershell" windowmanager.java -n

# 54.Token是什么？

android中的Token或者appToken这些是一个Binder对象



# 55.notepad或者sublime正则化搜索

前面是一部分，后面.*进行匹配

```
01-31 20:55:.*windowmanager
```





# 56.获取Property系统属性

https://blog.csdn.net/zsyf33078/article/details/132008812



# 57.新添加api时候需要生成current.txt文件更新api，不能自己更改

例如添加一个hide的方法，或者增加了一个属性，或者在dimens.xml里加了一个值

打包报错

```
FAILED: out/soong/.intermediates/miui/frameworks/base/miui-api-stubs-docs/android_common/metalava/check_current_api.timestamp
```

在frameworks/base下

在根目录下执行make api-stubs-docs-non-updatable-update-current-api -j8

```
make api-stubs-docs-non-updatable-update-current-api -j8
```

或者 make/ninja api-stubs-docs-check-current-api -j12

在miui/frameworks/base下

根目录下执行m miui-api-stubs-docs-update-current-api， 可以更新/api/current.txt

```
m miui-api-stubs-docs-update-current-api
```



# 58.dumpsys SurfaceFlinger可以看Surface相关信息

层级、parent等等

例如StatusBar#108的parent是StatusBar#101的parent是WindowToken{1e15c17 type=2000 android.os.BinderProxy#100的parent是Leaf:15:15#15

```
+ Layer (WindowToken{1e15c17 type=2000 android.os.BinderProxy#100  screenFlags = 0  privateLayer = 0) uid=1000
  Region TransparentRegion (this=0 count=0)
  Region VisibleRegion (this=0 count=0)
  Region SurfaceDamageRegion (this=0 count=0)
      layerStack=   0, z=        0, pos=(0,0), size=(   0,   0), crop=[  0,   0,  -1,  -1], cornerRadius=(0.000000,0.000000), isProtected=0, isTrustedOverlay=0, isOpaque=0, invalidate=1, dataspace=BT709 sRGB Full range, defaultPixelFormat=Unknown/None, backgroundBlurRadius=0, color=(-1.000,-1.000,-1.000,1.000), flags=0x00000000, tr=[0.00, 0.00][0.00, 0.00]
      parent=Leaf:15:15#15  screenFlags = 0  privateLayer = 0
      zOrderRelativeOf=none
      activeBuffer=[   0x   0:   0,Unknown/None], tr=[0.00, 0.00][0.00, 0.00] queued-frames=0 metadata={9:4bytes}, cornerRadiusCrop=[0.00, 0.00, 0.00, 0.00],  shadowRadius=0.000, 


+ Layer (Surface(name=f3eb1ed StatusBar#101)/#1687  screenFlags = 0  privateLayer = 0) uid=1000
  Region TransparentRegion (this=0 count=0)
  Region VisibleRegion (this=0 count=0)
  Region SurfaceDamageRegion (this=0 count=0)
      layerStack=   0, z=        0, pos=(0,0), size=(   0,   0), crop=[  0,   0,   0,   0], cornerRadius=(0.000000,0.000000), isProtected=0, isTrustedOverlay=0, isOpaque=0, invalidate=1, dataspace=BT709 sRGB Full range, defaultPixelFormat=Unknown/None, backgroundBlurRadius=0, color=(-1.000,-1.000,-1.000,1.000), flags=0x00000000, tr=[0.00, 0.00][0.00, 0.00]
      parent=WindowToken{1e15c17 type=2000 android.os.BinderProxy#100  screenFlags = 0  privateLayer = 0
      zOrderRelativeOf=none
      activeBuffer=[   0x   0:   0,Unknown/None], tr=[0.00, 0.00][0.00, 0.00] queued-frames=0 metadata={9:4bytes}, cornerRadiusCrop=[0.00, 0.00, 0.00, 0.00],  shadowRadius=0.000, 
      
      
+ Layer (f3eb1ed StatusBar#101  screenFlags = 0  privateLayer = 0) uid=1000
  Region TransparentRegion (this=0 count=0)
  Region VisibleRegion (this=0 count=0)
  Region SurfaceDamageRegion (this=0 count=0)
      layerStack=   0, z=        0, pos=(0,0), size=(   0,   0), crop=[  0,   0,  -1,  -1], cornerRadius=(0.000000,0.000000), isProtected=0, isTrustedOverlay=0, isOpaque=0, invalidate=1, dataspace=BT709 sRGB Full range, defaultPixelFormat=Unknown/None, backgroundBlurRadius=0, color=(-1.000,-1.000,-1.000,1.000), flags=0x00000000, tr=[0.00, 0.00][0.00, 0.00]
      parent=Surface(name=f3eb1ed StatusBar#101)/#1687  screenFlags = 0  privateLayer = 0
      zOrderRelativeOf=none
      activeBuffer=[   0x   0:   0,Unknown/None], tr=[0.00, 0.00][0.00, 0.00] queued-frames=0 metadata={9:4bytes}, cornerRadiusCrop=[0.00, 0.00, 0.00, 0.00],  shadowRadius=0.000, 

+ Layer (StatusBar#108  screenFlags = 2  privateLayer = 0) uid=1000
  Region TransparentRegion (this=0 count=0)
  Region VisibleRegion (this=0 count=1)
    [  0,   0, 1200, 116]
  Region SurfaceDamageRegion (this=0 count=0)
      layerStack=   0, z=        0, pos=(0,0), size=(   0,   0), crop=[  0,   0,  -1,  -1], cornerRadius=(0.000000,0.000000), isProtected=0, isTrustedOverlay=0, isOpaque=0, invalidate=0, dataspace=BT709 sRGB Full range, defaultPixelFormat=RGBA_8888, backgroundBlurRadius=0, color=(-1.000,-1.000,-1.000,1.000), flags=0x00000100, tr=[0.00, 0.00][0.00, 0.00]
      parent=f3eb1ed StatusBar#101  screenFlags = 0  privateLayer = 0
      zOrderRelativeOf=none
      activeBuffer=[1200x 116:268438272,RGBA_8888], tr=[0.00, 0.00][0.00, 0.00] queued-frames=0 metadata={ownerUID:1000, ownerPID:7839, dequeueTime:335365526061649, windowType:2000, 9:4bytes}, cornerRadiusCrop=[0.00, 0.00, 0.00, 0.00],  shadowRadius=0.000, 

* Layer 0xb400007d94a68000 (StatusBar#108)
      isSecure=false geomUsesSourceCrop=true geomBufferUsesDisplayInverseTransform=false geomLayerTransform (ROT_0) (IDENTITY)

      geomBufferSize=[0 0 1200 116] geomContentCrop=[0 0 1200 116] geomCrop=[0 0 -1 -1] geomBufferTransform=0 
        Region transparentRegionHint (this=0xb400007d94986888, count=1)
    [  0,   0,   0,   0]
      geomLayerBounds=[0.000000 0.000000 1200.000000 116.000000]       shadowRadius=0.000000 
      blend=PREMULTIPLIED (2) alpha=1.000000 backgroundBlurRadius=0 selfBlurRadius=0 composition type=DEVICE (2) 
      buffer: buffer=0xb400007d94958b00 
      sideband stream=0x0 
      color=[0.000000 0.000000 0.000000] 
      isOpaque=false hasProtectedContent=false isColorspaceAgnostic=true dataspace=V0_SRGB (142671872) hdr metadata types=0 dimming enabled=true colorTransform=[[1.000,0.000,0.000,0.000][0.000,1.000,0.000,0.000][0.000,0.000,1.000,0.000][0.000,0.000,0.000,1.000]] caching hint=Enabled 

Display 4630947082089526659 (active) HWC layers:
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 Layer name
           Z |  Window Type |  Layer Class | Comp Type |  Transform |   Disp Frame (LTRB) |          Source Crop (LTRB) |     Frame Rate (Explicit) (Seamlessness) [Focused]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 Wallpaper BBQ wrapper#59
  rel      0 |            0 |            0 |     DEVICE |          0 |    0    0 1200 2670 |    0.0    0.0 1200.0 2670.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 com.miui.home/com.miui.home.launcher.Launcher#1678
  rel      0 |            1 |            0 |     DEVICE |          0 |    0    0 1200 2670 |    0.0    0.0 1200.0 2670.0 |                                              [*]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 StatusBar#108
  rel      0 |         2000 |            0 |     DEVICE |          0 |    0    0 1200  116 |    0.0    0.0 1200.0  116.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 NotificationShade#1694
  rel      0 |         2040 |            0 |     DEVICE |          0 |    0    0 1200 2670 |    0.0    0.0 1200.0 2670.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 NavigationBar0#104
  rel      0 |         2019 |            0 |     DEVICE |          0 |    0 2622 1200 2670 |    0.0    0.0 1200.0   48.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------

```





# 59.在系统目录下保存文件或者图片

注意：在data或者system目录下可以保存文件，其他目录下可能存在权限问题

adb shell setenforce 0		#临时关闭selinux，如果data或者system目录不生效，可以使用该命令后重启后试试

```java
//首先在data目录下创建一个CTSPPP文件夹
String file_dir = "/data/CTSPPP";

//随机生成一个0-999的数字
Random r = new Random();
int num = r.nextInt(1000);
//获取一个snapshot的Bitmap对象，这里也可以是自己创建一个Bitmap
Bitmap screenShot = mInstrumentation.getUiAutomation().takeScreenshot();

//创建一个文件名
File newFile = new File(file_dir,"第" + num + "个.jpg");

try {
// 将Bitmap保存到文件
    FileOutputStream out = new FileOutputStream(newFile);
    screenShot.compress(Bitmap.CompressFormat.JPEG,100,out);
    out.flush();
    out.close();
} catch (FileNotFoundException e) {
    e.printStackTrace();
} catch (IOException e) {
    throw new RuntimeException(e);
}

```





保存截图到data目录

```
Bitmap bmp = buffer.asBitmap();
saveBitmap(bmp,"statusbar.JPEG");
 
 void saveBitmap(Bitmap bitmap, String bitName){
            String fileName;
            File file;
            fileName = "/data/"+bitName;
            file = new File(fileName);
            if(file.exists()){
                file.delete();
            }
            FileOutputStream out;
            try{
                out = new FileOutputStream(file);
                if(bitmap.compress(Bitmap.CompressFormat.JPEG, 90, out)) {
                    out.flush();
                    out.close();
                }
            } catch (FileNotFoundException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            }

        }
```



# 60.获取屏幕的高度

## 60.1 获取屏幕的真实高度（包含修饰）

```java
//获取的是包含修饰（导航栏和状态栏）的实际高度，真实屏幕的尺寸
DisplayMetrics dm = new DisplayMetrics();
mDisplay.getRealMetrics(dm);
int height = dm.heightPixels;
```

##  60.2 获取屏幕的高度，不包含修饰（导航栏和状态栏）

```
        mDisplay.getHeight()
```





# 61.CTS问题处理

CTS官方说明 ：

https://source.android.com/docs/compatibility/cts/command-console-v2?hl=zh-cn

https://blog.csdn.net/wed110/article/details/100030685

## 61.1 进入CTS模式

前置条件参考：

https://blog.csdn.net/ChaoLi_Chen/article/details/127689196

设置好前置条件后，进入CTS模式

在android-cts/tools 执行  ./cts-tradefed

##  61.2 CTS测试

- 整个模块测试

格式：run cts  -m  模块名称

```
run cts  -m  CtsWindowManagerDeviceTestCases		  #WMS测试
run cts -m  CtsSystemUiTestCases										#SystemUI CTS测试
```



- 测试某一个模块的类

格式：run cts  -m  模块名称 -t 测试模块的某一个类

```
run cts  -m  CtsWindowManagerDeviceTestCases -t  android.server.wm.ActivityTransitionTests
```



- 测试某一个模块的类的某一个测试用例



格式：run cts  -m  模块名称 -t 测试模块的某一个类#的某一个测试用例

```
run cts  -m  CtsWindowManagerDeviceTestCases -t  android.server.wm.ActivityTransitionTests#testAnimationBackgroundColorIsUsedDuringActivityTransition
```



跑某一个模块但是不跑模块中某一个类

格式：run cts --include-filter "模块名" --exclude-filter "模块名 类名"

```
run cts --include-filter "CtsWindowManagerDeviceTestCases" --exclude-filter "CtsWindowManagerDeviceTestCases android.server.wm.AnrTests"
```



跑某一个模块的某一个类，但是不跑类中的某一个测试用例

格式：run cts --include-filter "模块名 类名" --exclude-filter "模块名 类名#测试用例名"

```
run cts --include-filter "CtsWindowManagerDeviceTestCases android.server.wm.AnrTests" --exclude-filter "CtsWindowManagerDeviceTestCases android.server.wm.AnrTests#slowUiThreadWithKeyEventTriggersAnr"
```



单个测试某些测试用例，单个生成结果，注意这里是要单个生成，与上面的--include-filter不太一样。主要跑某些fail的用例，可以写成sh的脚本进行测试，这里的

singleCommand表示执行完单条指令后退出（相当于exit或者ctrl+c退出CTS进入终端）。

路径：android-cts-14_r2-linux_x86-arm/android-cts/tools

格式：./cts-tradefed run singleCommand cts -m 模块名 -t 类名

```
./cts-tradefed run singleCommand cts -m CtsWindowManagerDeviceTestCases -t android.server.wm.AssistantStackTests
./cts-tradefed run singleCommand cts -m CtsWindowManagerDeviceTestCases -t android.server.wm.BackGestureInvokedTest
./cts-tradefed run singleCommand cts -m CtsWindowManagerDeviceTestCases -t android.server.wm.BackNavigationLegacyGestureTest
./cts-tradefed run singleCommand cts -m CtsWindowManagerDeviceTestCases -t android.server.wm.BackNavigationLegacyTest
./cts-tradefed run singleCommand cts -m CtsWindowManagerDeviceTestCases -t android.server.wm.BackNavigationTests
```



运行以前失败的所有失败的用例

仅适用于 Android 9 及更高版本。重新尝试运行在以前的会话中失败或未执行的所有测试。例如，run retry --retry -s 或 run retry --retry -- shard-count（包含 TF 分片）可以用l r（注意：在CTS终端窗口下   ./cts-tradefed）获取session id

格式：./cts-tradefed run retry --retry  session-id

```
./cts-tradefed run singleCommand retry --retry  2
./cts-tradefed run singleCommand retry --retry  34
./cts-tradefed run singleCommand retry --retry  38
./cts-tradefed run singleCommand retry --retry  40
./cts-tradefed run singleCommand retry --retry  43
./cts-tradefed run singleCommand retry --retry  73
./cts-tradefed run singleCommand retry --retry  89
./cts-tradefed run singleCommand retry --retry  93
./cts-tradefed run singleCommand retry --retry  24
./cts-tradefed run singleCommand retry --retry  35
./cts-tradefed run singleCommand retry --retry  39
```



正则表达式匹配 导出测试Case和失败

```
var=test_result.xml
echo ${var##*/}

#提取处TestCase的名字
pre=$(grep -o -P '(?<=TestCase name=").*(?=">)' ${var##*/})

echo $pre

#提取处Test fail的名字
grep -o -P '(?<=Test result="fail" name=").*(?=">)' ${var##*/} > result.java

```





## 61.3 CTS结果

测试跑完之后会在result生成一个目录，名字表示跑CTS的时间，test_result_failures_suite.html表示跑的结果详单，test_result.html表示测试的结果简略内容

目录：android-cts-14_r2-linux_x86-arm/android-cts/results/2024.04.06_10.58.45

在cts页面输入l r可以查看所有的cts结果

```
cts-tf > l r
Session  Pass  Fail  Modules Complete  Result Directory       Test Plan  Device serial(s)  Build ID         Product   
0        51    1375  0 of 1            2024.04.09_12.40.37    cts  
1        18    5     1 of 1            2024.04.09_20.28.50    cts
2        0     26    1 of 1            2024.04.09_20.33.41    cts  
3        24    0     1 of 1            2024.04.09_20.42.04    cts 
4        64    0     1 of 1            2024.04.09_21.00.45    cts   
5        11    0     1 of 1            2024.04.10_10.33.18    cts     

```







## 61.4 问题

1.谷歌浏览器打不开相应的html文件

```
cd /opt/google/chrome		//进入谷歌浏览器安装目录
./google-chrome --allow-file-access-from-files
```



# 62.获取是全面屏手势、经典导航键的方式

```java
//需要导入Settings库
import android.provider.Settings;

int navBarMode = -1;
if (mController.mAtm.mContext != null) {
    navBarMode = Settings.Secure.getInt(mController.mAtm.mContext.getContentResolver(),
                                        Settings.Secure.NAVIGATION_MODE,-1);
}


/**
* Navigation bar mode.
*  0 = 3 button
*  1 = 2 button
*  2 = fully gestural
* @hide
*/
@Readable
public static final String NAVIGATION_MODE =
            "navigation_mode";

```



# 63.sublime选中多行，修改文本

https://blog.csdn.net/sinat_34719507/article/details/53891927

选中文本，同时点击ctrl+shift+L，键盘左键或者右键操作到指定位置，统一添加或者删除即可





# 64.Android读写AIDL的Binder对象

Parcel对象的in,dest

写入

```
dest.writeStrongBinder(mCallback != null ?
                mUpdateCallback.asBinder() : null);
```

读取

```
mCallback = ICallback.Stub.asInterface(in.readStrongBinder());
```



# 65.android.anim线程和android.anim.if线程

参考：

[Android Systrace 基础知识(4) - SystemServer 解读](https://zhuanlan.zhihu.com/p/142792393)

[Android P——LockFreeAnimation](https://zhuanlan.zhihu.com/p/44864987)



https://blog.csdn.net/learnframework/article/details/138437493

两个线程都是在systemserver进程下，SurfaceAnimationThread主要执行的是窗口动画，用于分担AnimationThread的一部分动画工作，减少由于锁导致的窗口动画卡顿问题。





android.anim线程是AnimationThread动画线程

路径：

```java
/**
 * Thread for handling all legacy window animations, or anything that's directly impacting
 * animations like starting windows or traversals.
 *用于处理所有旧窗口动画或任何直接影响动画（例如启动窗口或遍历）的线程。
 */    

private AnimationThread() {
        super("android.anim", THREAD_PRIORITY_DISPLAY, false /*allowIo*/);
    }

    private static void ensureThreadLocked() {
        if (sInstance == null) {
            sInstance = new AnimationThread();
            sInstance.start();
            sInstance.getLooper().setTraceTag(Trace.TRACE_TAG_WINDOW_MANAGER);
            sHandler = makeSharedHandler(sInstance.getLooper());
        }
    }
    
        public static AnimationThread get() {
        synchronized (AnimationThread.class) {
            ensureThreadLocked();
            return sInstance;
        }
    }

    public static Handler getHandler() {
        synchronized (AnimationThread.class) {
            ensureThreadLocked();
            return sHandler;
        }
    }
```

android.anim.if是SurfaceAnimationThread窗口动画线程

路径：services/core/java/com/android/server/wm/SurfaceAnimationThread.java

```java
/**
 * Thread for running {@link SurfaceAnimationRunner} that does not hold the window manager lock.
 *用于运行不持有windowmanagerservice锁的SurfaceAnimationRunner线程
 */    

private SurfaceAnimationThread() {
        super("android.anim.lf", THREAD_PRIORITY_DISPLAY, false /*allowIo*/);
    }

    private static void ensureThreadLocked() {
        if (sInstance == null) {
            sInstance = new SurfaceAnimationThread();
            sInstance.start();
            sInstance.getLooper().setTraceTag(Trace.TRACE_TAG_WINDOW_MANAGER);
            sHandler = makeSharedHandler(sInstance.getLooper());
        }
    }

    public static SurfaceAnimationThread get() {
        synchronized (SurfaceAnimationThread.class) {
            ensureThreadLocked();
            return sInstance;
        }
    }

    public static Handler getHandler() {
        synchronized (SurfaceAnimationThread.class) {
            ensureThreadLocked();
            return sHandler;
        }
    }

```



# 66.自己添加的log使用Protolog开关打开的方式

例如:ProtoLogGroup.WM_DEBUG_WINDOW_TRANSITIONS.isLogToLogcat()

```
        if (ProtoLogGroup.WM_DEBUG_WINDOW_TRANSITIONS.isLogToLogcat()) {
            Slog.d(TAG,"Aborting Transition: " + this
                    + " mSyncId = " + mSyncId, new Throwable());
        }
```





# 67.Binder挂掉抛出异常DeadObjectException

Binder挂掉会打印下面信息

**Binder transaction failure**

对应代码中的位置

路径：frameworks/native/libs/binder/IPCThreadState.cpp

```
    ALOGE_IF(ee.command != BR_OK, "Binder transaction failure: %d/%d/%d",
             ee.id, ee.command, ee.param);
```

DeadObjectException继承于RemoteException

DeadObjectException.java

```java
/**
 * The object you are calling has died, because its hosting process
 * no longer exists.
 */
public class DeadObjectException extends RemoteException {
    public DeadObjectException() {
        super();
    }

    public DeadObjectException(String message) {
        super(message);
    }
    
    ......
}
```



RemoteException.java

```java
/**
 * Parent exception for all Binder remote-invocation errors
 *
 * Note: not all exceptions from binder services will be subclasses of this.
 *   For instance, RuntimeException and several subclasses of it may be
 *   thrown as well as OutOfMemoryException.
 *
 * One common subclass is {@link DeadObjectException}.
 */
public class RemoteException extends AndroidException {
    public RemoteException() {
        super();
    }

    public RemoteException(String message) {
        super(message);
    }
    
    ......
}
```



# 68. SurfaceFlinger 支持的最大 Layer 数量为 4096

 **SurfaceFlinger 支持的最大 Layer 数量为 4096 个**, 超过会返回内存不足的错误, 当然我们很难达到这个数值.

路径：// frameworks/native/services/surfaceflinger/SurfaceFlinger.h

```
static const size_t MAX_LAYERS = 4096; 
```



路径: frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp

```java
status_t SurfaceFlinger::addClientLayer(LayerCreationArgs& args, const sp<IBinder>& handle,
                                        const sp<Layer>& layer, const wp<Layer>& parent,
                                        uint32_t* outTransformHint) {
    if (mNumLayers >= MAX_LAYERS) {
        ALOGE("AddClientLayer failed, mNumLayers (%zu) >= MAX_LAYERS (%zu)", mNumLayers.load(),
              MAX_LAYERS);
        static_cast<void>(mScheduler->schedule([=, this] {
            ALOGE("Dumping layer keeping > 20 children alive:");
            bool leakingParentLayerFound = false;
            mDrawingState.traverse([&](Layer* layer) {
                if (leakingParentLayerFound) {
                    return;
                }
                if (layer->getChildrenCount() > 20) {
                    leakingParentLayerFound = true;
                    sp<Layer> parent = sp<Layer>::fromExisting(layer);
                    while (parent) {
                        ALOGE("Parent Layer: %s%s", parent->getName().c_str(),
                              (parent->isHandleAlive() ? "handleAlive" : ""));
                        parent = parent->getParent();
                    }
                    // Sample up to 100 layers
                    ALOGE("Dumping random sampling of child layers total(%zu): ",
                          layer->getChildrenCount());
                    int sampleSize = (layer->getChildrenCount() / 100) + 1;
                    layer->traverseChildren([&](Layer* layer) {
                        if (rand() % sampleSize == 0) {
                            ALOGE("Child Layer: %s%s", layer->getName().c_str(),
                                  (layer->isHandleAlive() ? "handleAlive" : ""));
                        }
                    });
                }
            });

            int numLayers = 0;
            mDrawingState.traverse([&](Layer* layer) { numLayers++; });

            ALOGE("Dumping random sampling of on-screen layers total(%u):", numLayers);
            mDrawingState.traverse([&](Layer* layer) {
                // Aim to dump about 200 layers to avoid totally trashing
                // logcat. On the other hand, if there really are 4096 layers
                // something has gone totally wrong its probably the most
                // useful information in logcat.
                if (rand() % 20 == 13) {
                    ALOGE("Layer: %s%s", layer->getName().c_str(),
                          (layer->isHandleAlive() ? "handleAlive" : ""));
                    std::this_thread::sleep_for(std::chrono::milliseconds(5));
                }
            });
            ALOGE("Dumping random sampling of off-screen layers total(%zu): ",
                  mOffscreenLayers.size());
            for (Layer* offscreenLayer : mOffscreenLayers) {
                if (rand() % 20 == 13) {
                    ALOGE("Offscreen-layer: %s%s", offscreenLayer->getName().c_str(),
                          (offscreenLayer->isHandleAlive() ? "handleAlive" : ""));
                    std::this_thread::sleep_for(std::chrono::milliseconds(5));
                }
            }
        }));
        return NO_MEMORY;
    }
```



# 69.过滤某一个文件下的某个关键字

格式：grep -inE "关键字" 绝对路径名

grep -inE "windowmanager:" /home/mi/下载/dddffff.java



//过滤当前文件夹下的dddffff.java文件

grep -inE "windowmanager:" dddffff.java







# 70.查看分支创建时间

git reflog show --date=iso **分支名**



# 71.V上动态开关SurfaceControl相关方法

```
V 上Google加了SurfaceControl增强日志，可以动态打开，很方便。
// set
adb shell setprop persist.wm.debug.sc.tx.log_match_call setAlpha
adb shell setprop persist.wm.debug.sc.tx.log_match_call setCrop
adb shell setprop persist.wm.debug.sc.tx.log_match_call setMatrix
adb shell setprop persist.wm.debug.sc.tx.log_match_call show
adb shell setprop persist.wm.debug.sc.tx.log_match_name com.android.systemui
adb reboot
adb logcat -s "SurfaceControlRegistry"
adb logcat -s -c && adb logcat -s "SurfaceControlRegistry"
// unset
adb shell setprop persist.wm.debug.sc.tx.log_match_call \"\"
adb shell setprop persist.wm.debug.sc.tx.log_match_name \"\"
adb reboot
adb logcat -s "SurfaceControlRegistry"
adb logcat -b all -c && adb logcat -b all | grep -iE "SurfaceControlRegistry:"
adb logcat -b all -c && adb logcat -b all | grep -iE "SurfaceControlRegistry: setPosition"


//打开多个属性
adb root
adb shell setprop persist.wm.debug.sc.tx.log_match_call show,hide,reparent,setMatrix,apply,merge,setCrop,setLayer,setPosition,construct,setAlpha,release
adb shell setprop persist.wm.debug.sc.tx.log_match_call show,hide,apply,merge
adb shell setprop persist.wm.debug.sc.tx.log_match_call setAlpha
adb shell setprop persist.wm.debug.sc.tx.log_match_call setAlpha,reparent,show,hide,apply,merge
adb reboot
```



# 72.git cherry-pick --continue

cherrypick存在冲突，解决冲突后

```
git add .
git cherry-pick --continue
repo upload . --no-veify
```



# 73.adb shell dumpsys input

dumpsys input可以打印Input Dispatcher State的状态信息。

**例如：**

FocusedDisplayId：焦点所在的DisplayId

FocusedApplications：焦点应用

FocusedWindows：焦点窗口

Windows:自上而下层级越来越低，Input Dispatcher也是自上而下去判断哪一个窗口可以处理事件。



不能响应事件关键信息：

1）inputConfig设置有NOT_TOUCHABLE或者NOT_VISIBLE的Flag

2） alpha=0

3） frame的范围太小

4）touchableRegion的范围没有在触摸区域



例如在桌面dump input信息

自上而下的窗口分别是

```
//可以看到RoundCornerBottom是NOT_TOUCHABLE，alpha=1， touchableRegion=[0,2909][1440,3200]，由于设置有NOT_TOUCHABLE的flag，因此不会响应事件
0: name=f08eb00 RoundCornerBottom  inputConfig=NOT_FOCUSABLE | NOT_TOUCHABLE | TRUSTED_OVERLAY | SLIPPERY, miuiFlags=0x0, alpha=1, frame=[0,2909][1440,3200], globalScale=1, applicationInfo.name=, applicationInfo.token=<null>, touchableRegion=[0,2909][1440,3200]
//同上
1: name=efabf64 RoundCornerTop inputConfig=NOT_FOCUSABLE | NOT_TOUCHABLE | TRUSTED_OVERLAY | SLIPPERY, miuiFlags=0x0, alpha=1, frame=[0,0]				  	[1440,291], globalScale=1, applicationInfo.name=, applicationInfo.token=<null>, touchableRegion=[0,0][1440,291]
//Frame大小为0
2: name=[Gesture Monitor] miui-gesture inputConfig=NOT_FOCUSABLE | TRUSTED_OVERLAY | SPY, miuiFlags=0x0, alpha=1, frame=[0,0][0,0], globalScale=1, 	  		 applicationInfo.name=[Gesture Monitor] miui-gesture, applicationInfo.token=<null>, touchableRegion=[-14399,-31999][14400,32000]
//设置有NOT_VISIBLE的flag，因此不会响应事件
3: name=Embedded{SplitImeControllerManager} inputConfig=NOT_VISIBLE | NOT_FOCUSABLE | PREVENT_SPLITTING, miuiFlags=0x0, alpha=1, frame=[0,0][367,147], globalScale=0, applicationInfo.name=, applicationInfo.token=<null>, touchableRegion=[0,0][367,147]
4: name=f80fed0 GestureStubRight
5: name=2738672 GestureStubLeft
6: name=a1f98da NavigationBar0
7: name=718e48e GestureStubHome
//可以获取touchableRegion=[0,0][1440,147]的事件
8: name=bcf3206 StatusBar inputConfig=NOT_FOCUSABLE | TRUSTED_OVERLAY, miuiFlags=0x0, alpha=1, frame=[0,0][1440,147], globalScale=1, applicationInfo.name=, applicationInfo.token=<null>, touchableRegion=[0,0][1440,147]
9: name=recents_animation_input_consumer
10: name=880ea5 ActivityRecordInputSink com.android.soundrecorder/.SoundRecorder
11: name=d07062c ActivityRecordInputSink com.android.soundrecorder/.RecordPreviewActivity
12: name=3a76b2d LauncherOverlayWindow:com.miui.personalassistant
//可以获取全屏的触摸事件
13: name=4510a7b com.miui.home/com.miui.home.launcher.Launcher  inputConfig=DUPLICATE_TOUCH_TO_WALLPAPER, miuiFlags=0x0, alpha=1, frame=[0,0][1440,3200], globalScale=1, applicationInfo.name=ActivityRecord{f8e4049 u0 com.miui.home/.launcher.Launcher t2}
14: name=75b0650 ActivityRecordInputSink com.miui.home/.launcher.Launcher,
15: name=62947e0 com.miui.miwallpaper.wallpaperservice.ImageWallpaper



```





完整的Input Dispatcher State信息

```
Input Dispatcher State:
  DispatchEnabled: true
  DispatchFrozen: false
  InputFilterEnabled: false
  FocusedDisplayId: 0
  FocusedApplications:
    displayId=0, name='ActivityRecord{f8e4049 u0 com.miui.home/.launcher.Launcher t2}', dispatchingTimeout=5000ms
  FocusedWindows:
    displayId=0, name='4510a7b com.miui.home/com.miui.home.launcher.Launcher'
  FocusRequests:
    displayId=0, name='4510a7b com.miui.home/com.miui.home.launcher.Launcher' result='OK'
  Pointer Capture Requested: false
  Current Window with Pointer Capture: None
  TouchStates: <no displays touched>
  Display: 0
    logicalSize=1080x2400
        transform (ROT_0) (SCALE )
            0.7500  -0.0000  0.0000
            -0.0000  0.7500  0.0000
            0.0000  0.0000  1.0000
    Windows:
      0: name=f08eb00 RoundCornerBottom, id=103, displayId=0, inputConfig=NOT_FOCUSABLE | NOT_TOUCHABLE | TRUSTED_OVERLAY | SLIPPERY, miuiFlags=0x0, alpha=1, frame=[0,2909][1440,3200], globalScale=1, applicationInfo.name=, applicationInfo.token=<null>, touchableRegion=[0,2909][1440,3200], ownerPid=5028, ownerUid=1000, dispatchingTimeout=5000ms, token=0xb400007ba51ffd10, touchOcclusionMode=BLOCK_UNTRUSTED, needMiuiEmbeddedEventMapping=0, miuiEmbeddedHotRegion=<empty>, miuiEmbeddedMidRegion=<empty>
    transform (ROT_0) (SCALE TRANSLATE)
        0.7500  -0.0000  0.0000
        -0.0000  0.7500  -2182.0000
        0.0000  0.0000  1.0000
      1: name=efabf64 RoundCornerTop, id=102, displayId=0, inputConfig=NOT_FOCUSABLE | NOT_TOUCHABLE | TRUSTED_OVERLAY | SLIPPERY, miuiFlags=0x0, alpha=1, frame=[0,0][1440,291], globalScale=1, applicationInfo.name=, applicationInfo.token=<null>, touchableRegion=[0,0][1440,291], ownerPid=5028, ownerUid=1000, dispatchingTimeout=5000ms, token=0xb400007ba52158d0, touchOcclusionMode=BLOCK_UNTRUSTED, needMiuiEmbeddedEventMapping=0, miuiEmbeddedHotRegion=<empty>, miuiEmbeddedMidRegion=<empty>
    transform (ROT_0) (SCALE )
        0.7500  -0.0000  0.0000
        -0.0000  0.7500  0.0000
        0.0000  0.0000  1.0000
      2: name=[Gesture Monitor] miui-gesture, id=51, displayId=0, inputConfig=NOT_FOCUSABLE | TRUSTED_OVERLAY | SPY, miuiFlags=0x0, alpha=1, frame=[0,0][0,0], globalScale=1, applicationInfo.name=[Gesture Monitor] miui-gesture, applicationInfo.token=<null>, touchableRegion=[-14399,-31999][14400,32000], ownerPid=2354, ownerUid=1000, dispatchingTimeout=5000ms, token=0xb400007ba51a70d0, touchOcclusionMode=BLOCK_UNTRUSTED, needMiuiEmbeddedEventMapping=0, miuiEmbeddedHotRegion=<empty>, miuiEmbeddedMidRegion=<empty>
    transform (ROT_0) (SCALE )
        0.7500  -0.0000  0.0000
        -0.0000  0.7500  0.0000
        0.0000  0.0000  1.0000
      3: name=Embedded{SplitImeControllerManager}, id=92, displayId=0, inputConfig=NOT_VISIBLE | NOT_FOCUSABLE | PREVENT_SPLITTING, miuiFlags=0x0, alpha=1, frame=[0,0][367,147], globalScale=0, applicationInfo.name=, applicationInfo.token=<null>, touchableRegion=[0,0][367,147], ownerPid=2354, ownerUid=1000, dispatchingTimeout=5000ms, token=0xb400007ba51d2210, touchOcclusionMode=BLOCK_UNTRUSTED, needMiuiEmbeddedEventMapping=0, miuiEmbeddedHotRegion=<empty>, miuiEmbeddedMidRegion=<empty>
    transform (ROT_0) (SCALE )
        0.7500  -0.0000  0.0000
        -0.0000  0.7500  0.0000
        0.0000  0.0000  1.0000
      4: name=f80fed0 GestureStubRight, id=80, displayId=0, inputConfig=NOT_FOCUSABLE | PREVENT_SPLITTING | TRUSTED_OVERLAY, miuiFlags=0x0, alpha=1, frame=[1357,1189][1440,3109], globalScale=1, applicationInfo.name=, applicationInfo.token=<null>, touchableRegion=<empty>, ownerPid=4569, ownerUid=10142, dispatchingTimeout=5000ms, token=0xb400007ba51d3610, touchOcclusionMode=BLOCK_UNTRUSTED, needMiuiEmbeddedEventMapping=0, miuiEmbeddedHotRegion=<empty>, miuiEmbeddedMidRegion=<empty>
    transform (ROT_0) (SCALE TRANSLATE)
        0.7500  -0.0000  -1017.9999
        -0.0000  0.7500  -891.9999
        0.0000  0.0000  1.0000
      5: name=2738672 GestureStubLeft, id=75, displayId=0, inputConfig=NOT_FOCUSABLE | PREVENT_SPLITTING | TRUSTED_OVERLAY, miuiFlags=0x0, alpha=1, frame=[0,1189][83,3109], globalScale=1, applicationInfo.name=, applicationInfo.token=<null>, touchableRegion=<empty>, ownerPid=4569, ownerUid=10142, dispatchingTimeout=5000ms, token=0xb400007ba51c8610, touchOcclusionMode=BLOCK_UNTRUSTED, needMiuiEmbeddedEventMapping=0, miuiEmbeddedHotRegion=<empty>, miuiEmbeddedMidRegion=<empty>
    transform (ROT_0) (SCALE TRANSLATE)
        0.7500  -0.0000  0.0000
        -0.0000  0.7500  -891.9999
        0.0000  0.0000  1.0000
      6: name=a1f98da NavigationBar0, id=115, displayId=0, inputConfig=NOT_FOCUSABLE | TRUSTED_OVERLAY | WATCH_OUTSIDE_TOUCH, miuiFlags=0x0, alpha=1, frame=[0,3144][1440,3200], globalScale=1, applicationInfo.name=, applicationInfo.token=<null>, touchableRegion=<empty>, ownerPid=5028, ownerUid=1000, dispatchingTimeout=5000ms, token=0xb400007ba50f7050, touchOcclusionMode=BLOCK_UNTRUSTED, needMiuiEmbeddedEventMapping=0, miuiEmbeddedHotRegion=<empty>, miuiEmbeddedMidRegion=<empty>
    transform (ROT_0) (SCALE TRANSLATE)
        0.7500  -0.0000  0.0000
        -0.0000  0.7500  -2357.9998
        0.0000  0.0000  1.0000
      7: name=718e48e GestureStubHome, id=79, displayId=0, inputConfig=NOT_FOCUSABLE | PREVENT_SPLITTING | TRUSTED_OVERLAY, miuiFlags=0x0, alpha=1, frame=[0,3109][1440,3200], globalScale=1, applicationInfo.name=, applicationInfo.token=<null>, touchableRegion=[0,3109][1440,3200], ownerPid=4569, ownerUid=10142, dispatchingTimeout=5000ms, token=0xb400007ba51d2fd0, touchOcclusionMode=BLOCK_UNTRUSTED, needMiuiEmbeddedEventMapping=0, miuiEmbeddedHotRegion=<empty>, miuiEmbeddedMidRegion=<empty>
    transform (ROT_0) (SCALE TRANSLATE)
        0.7500  -0.0000  0.0000
        -0.0000  0.7500  -2332.0000
        0.0000  0.0000  1.0000
      8: name=bcf3206 StatusBar, id=119, displayId=0, inputConfig=NOT_FOCUSABLE | TRUSTED_OVERLAY, miuiFlags=0x0, alpha=1, frame=[0,0][1440,147], globalScale=1, applicationInfo.name=, applicationInfo.token=<null>, touchableRegion=[0,0][1440,147], ownerPid=5028, ownerUid=1000, dispatchingTimeout=5000ms, token=0xb400007ba50ed190, touchOcclusionMode=BLOCK_UNTRUSTED, needMiuiEmbeddedEventMapping=0, miuiEmbeddedHotRegion=<empty>, miuiEmbeddedMidRegion=<empty>
    transform (ROT_0) (SCALE )
        0.7500  -0.0000  0.0000
        -0.0000  0.7500  0.0000
        0.0000  0.0000  1.0000
      9: name=recents_animation_input_consumer, id=78, displayId=0, inputConfig=NOT_VISIBLE | TRUSTED_OVERLAY, miuiFlags=0x0, alpha=1, frame=[0,0][1440,3200], globalScale=1, applicationInfo.name=recents_animation_input_consumer, applicationInfo.token=0xb400007af5184f30, touchableRegion=[0,0][1440,3200], ownerPid=2354, ownerUid=1000, dispatchingTimeout=5000ms, token=0xb400007ba51c4fd0, touchOcclusionMode=BLOCK_UNTRUSTED, needMiuiEmbeddedEventMapping=0, miuiEmbeddedHotRegion=<empty>, miuiEmbeddedMidRegion=<empty>
    transform (ROT_0) (SCALE )
        0.7500  -0.0000  0.0000
        -0.0000  0.7500  0.0000
        0.0000  0.0000  1.0000
      10: name=880ea5 ActivityRecordInputSink com.android.soundrecorder/.SoundRecorder, id=158, displayId=0, inputConfig=NO_INPUT_CHANNEL | NOT_VISIBLE | NOT_FOCUSABLE | NOT_TOUCHABLE, miuiFlags=0x0, alpha=0, frame=[217,59][217,59], globalScale=0, applicationInfo.name=, applicationInfo.token=<null>, touchableRegion=[217,59][273,115], ownerPid=2354, ownerUid=1000, dispatchingTimeout=0ms, token=0x0, touchOcclusionMode=BLOCK_UNTRUSTED, needMiuiEmbeddedEventMapping=0, miuiEmbeddedHotRegion=<empty>, miuiEmbeddedMidRegion=<empty>
    transform (ROT_0) (SCALE TRANSLATE)
        19.3306  -0.0000  -4202.4419
        -0.0000  19.3306  -1135.3190
        0.0000  0.0000  1.0000
      11: name=d07062c ActivityRecordInputSink com.android.soundrecorder/.RecordPreviewActivity, id=144, displayId=0, inputConfig=NO_INPUT_CHANNEL | NOT_VISIBLE | NOT_FOCUSABLE | NOT_TOUCHABLE, miuiFlags=0x0, alpha=1, frame=[-359,0][-359,0], globalScale=0, applicationInfo.name=, applicationInfo.token=<null>, touchableRegion=[-14399,-31999][14400,32000], ownerPid=2354, ownerUid=1000, dispatchingTimeout=0ms, token=0x0, touchOcclusionMode=BLOCK_UNTRUSTED, needMiuiEmbeddedEventMapping=0, miuiEmbeddedHotRegion=<empty>, miuiEmbeddedMidRegion=<empty>
    transform (ROT_0) (SCALE TRANSLATE)
        0.7500  -0.0000  269.8145
        -0.0000  0.7500  0.0000
        0.0000  0.0000  1.0000
      12: name=3a76b2d LauncherOverlayWindow:com.miui.personalassistant, id=218, displayId=0, inputConfig=NOT_VISIBLE | NOT_FOCUSABLE | NOT_TOUCHABLE | WATCH_OUTSIDE_TOUCH, miuiFlags=0x0, alpha=0, frame=[0,0][1440,3200], globalScale=1, applicationInfo.name=ActivityRecord{f8e4049 u0 com.miui.home/.launcher.Launcher t2}, applicationInfo.token=0xb400007af51bc1f0, touchableRegion=[0,0][1440,3200], ownerPid=7080, ownerUid=10135, dispatchingTimeout=5000ms, token=0xb400007ba523fad0, touchOcclusionMode=BLOCK_UNTRUSTED, needMiuiEmbeddedEventMapping=0, miuiEmbeddedHotRegion=<empty>, miuiEmbeddedMidRegion=<empty>, canOccludePresentation
    transform (ROT_0) (SCALE )
        0.7500  -0.0000  0.0000
        -0.0000  0.7500  0.0000
        0.0000  0.0000  1.0000
      13: name=4510a7b com.miui.home/com.miui.home.launcher.Launcher, id=219, displayId=0, inputConfig=DUPLICATE_TOUCH_TO_WALLPAPER, miuiFlags=0x0, alpha=1, frame=[0,0][1440,3200], globalScale=1, applicationInfo.name=ActivityRecord{f8e4049 u0 com.miui.home/.launcher.Launcher t2}, applicationInfo.token=0xb400007af51bc1f0, touchableRegion=[0,0][1440,3200], ownerPid=4569, ownerUid=10142, dispatchingTimeout=5000ms, token=0xb400007ba52030d0, touchOcclusionMode=BLOCK_UNTRUSTED, needMiuiEmbeddedEventMapping=0, miuiEmbeddedHotRegion=<empty>, miuiEmbeddedMidRegion=<empty>
    transform (ROT_0) (SCALE )
        0.7500  -0.0000  0.0000
        -0.0000  0.7500  0.0000
        0.0000  0.0000  1.0000
      14: name=75b0650 ActivityRecordInputSink com.miui.home/.launcher.Launcher, id=64, displayId=0, inputConfig=NO_INPUT_CHANNEL | NOT_FOCUSABLE, miuiFlags=0x0, alpha=1, frame=[0,0][0,0], globalScale=0, applicationInfo.name=, applicationInfo.token=<null>, touchableRegion=[-14399,-31999][14400,32000], ownerPid=2354, ownerUid=1000, dispatchingTimeout=0ms, token=0x0, touchOcclusionMode=BLOCK_UNTRUSTED, needMiuiEmbeddedEventMapping=0, miuiEmbeddedHotRegion=<empty>, miuiEmbeddedMidRegion=<empty>
    transform (ROT_0) (SCALE )
        0.7500  -0.0000  0.0000
        -0.0000  0.7500  0.0000
        0.0000  0.0000  1.0000
      15: name=62947e0 com.miui.miwallpaper.wallpaperservice.ImageWallpaper, id=57, displayId=0, inputConfig=NOT_FOCUSABLE | NOT_TOUCHABLE | PREVENT_SPLITTING | IS_WALLPAPER, miuiFlags=0x0, alpha=1, frame=[0,0][0,0], globalScale=1, applicationInfo.name=, applicationInfo.token=<null>, touchableRegion=<empty>, ownerPid=4612, ownerUid=10190, dispatchingTimeout=5000ms, token=0xb400007ba50c7f10, touchOcclusionMode=BLOCK_UNTRUSTED, needMiuiEmbeddedEventMapping=0, miuiEmbeddedHotRegion=<empty>, miuiEmbeddedMidRegion=<empty>
    transform (ROT_0) (SCALE )
        0.7500  -0.0000  0.0000
        -0.0000  0.7500  0.0000
        0.0000  0.0000  1.0000

```



# 75.android使用setprop自定义开关，开启以及关闭

参考：[使用Log.isLoggable方法](https://www.cnblogs.com/Peter-Chen/p/5042460.html)

方法：

1）通过Log.isLoggable设置TAG名字，以及输出

```java
public static boolean isEnableXXXDebug() {
        return Log.isLoggable("XXX",Log.DEBUG);
}


//TAG是指要设置的名字，level表示设置的等级
* @param tag The tag to check.
* @param level The level to check.
public static native boolean isLoggable(@Nullable String tag, @Level int level);
```

2.手动设置

```
adb shell setprop log.tag.XXX DEBUG  #可以看DEBUG及以上等级
adb shell setprop log.tag.XXX ERROR  #可以看ERROR及以上等级

adb shell setprop log.tag.XXX false  #关闭

adb shell stop && adb shell start #如果没有生效，可以重启下手机
```



3.调用

```java
if (isEnableXXXDebug()) {
    Slog.d(TAG," ...... ");
}
```

# 76.多个change cp到一笔

先cp，解决冲突

git log .查看log，找到第一笔之前的commit id

git rebase -i commit id

将后面的标识修改成s，挤压到第一笔



# 77.如何根据ROM包的manifest拉取相应的代码

1）从rom包中下载相应的manifests文件

2）将下载的相应的manifest-xxx-xxx.xml文件放到.repo/manifests目录下

```
sudo cp /media/pengmengxing/pmx/下载/qcom-manifest-system-OS2.0.250210.1.VNICNXM.PRE-TEST-15.0-c073fdbc36.xml /.repo/manifests
```

3）软连接指向.repo目录下的 manifest.xml文件

```
pengmengxing@mi:/media/pengmengxing/mySSD1/N2_O3_O8_N8_VPPE_0205/.repo$ ln -snf manifests/qcom-manifest-system-OS2.0.250210.1.VNICNXM.PRE-TEST-15.0-c073fdbc36.xml manifest.xml
```

4）拉取代码

```
repo sync -j12 -d -c --no-tags --force-sync
```

5）编译代码

这样就可以保证Rom包与相应的代码完全匹配了，编译代码生成的jar包就能完全对应了。



6）查看链接指向

```
ls -l	#ll
```



# 78.录屏并保存相关log（非抓系统bugreport版）

打开两个终端抓取

```
//录屏
adb shell screenrecord --bugreport sdcard/record.mp4
//抓bugreport
adb logcat -b all -c && adb logcat -b all > bugreport.log #或者adb  bugreport
```

