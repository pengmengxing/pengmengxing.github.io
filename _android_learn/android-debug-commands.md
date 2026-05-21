---
layout: private_post
title: "Android 系统调试命令速查"
date: 2026-05-21
categories: android-learn
tags: [adb, debugging, build, surfaceflinger, wms]
render_with_liquid: false
---

# Android 系统调试命令速查

## 显示系统常用命令

```bash
# 一键抓取常用 dump
adb shell dumpsys window w > window.log
adb shell dumpsys activity a > activity.log
adb shell dumpsys SurfaceFlinger > sf.log
adb shell dumpsys display d > display.log
adb shell dumpsys activity containers > containers.log
adb shell dumpsys input > input.log

# 连续 dump SurfaceFlinger（分析闪烁/图层变化）
adb shell "while true; do dumpsys SurfaceFlinger; sleep 0.008;done > /data/dump.SF.log"
adb pull /data/dump.SF.log sf.txt

# 查看容器/最近任务/焦点
adb shell dumpsys activity containers
adb shell dumpsys activity recents
adb shell dumpsys activity a | grep mCurrentFocus

# 屏幕分辨率与密度
adb shell wm density
adb shell wm size

# Insets 信息
adb shell dumpsys window d                      # 所有 Display Insets
adb shell dumpsys activity | grep cutout -i

# Display 信息
adb shell dumpsys display d | grep mBaseDisplayInfo=DisplayInfo -A20

# 导航栏/状态栏高度（SystemUI overlay）
adb shell cmd overlay lookup --verbose com.android.systemui android:dimen/navigation_bar_height
adb shell cmd overlay lookup --verbose com.android.systemui android:dimen/status_bar_height_default

# WMShell Transition 日志
adb shell dumpsys activity service SystemUIService WMShell WM_SHELL_TRANSITION WM_SHELL_RECENTS_TRANSITION
```

## WMS（dumpsys window 拆解）

```bash
adb shell dumpsys window                  # 全量输出
adb shell dumpsys window lastanr          # 最近 ANR
adb shell dumpsys window policy           # 策略状态
adb shell dumpsys window animator         # Animator 状态
adb shell dumpsys window sessions         # Session 列表
adb shell dumpsys window displays         # Display 内容
adb shell dumpsys window tokens           # Token 列表
adb shell dumpsys window windows          # 窗口列表
adb shell dumpsys window trace            # Trace 状态
adb shell dumpsys window logging          # ProtoLog 开关状态
adb shell dumpsys window refresh          # 高刷黑名单
adb shell dumpsys window constants        # 常量
```

## AMS（dumpsys activity 拆解）

```bash
adb shell dumpsys activity                       # 全量输出
adb shell dumpsys activity settings              # 常量配置
adb shell dumpsys activity broadcasts            # 广播状态
adb shell dumpsys activity providers             # ContentProvider
adb shell dumpsys activity permissions           # URI 权限
adb shell dumpsys activity services              # Service 列表
adb shell dumpsys activity recents               # 最近任务
adb shell dumpsys activity lastanr               # 最近 ANR
adb shell dumpsys activity starter               # 最近启动信息
adb shell dumpsys activity containers            # 容器层级
adb shell dumpsys activity activities            # Activity 信息
adb shell dumpsys activity processes             # 进程信息
adb shell dumpsys activity users                 # 用户信息
adb shell dumpsys activity loopers               # Looper 耗时消息
```

## SurfaceFlinger

```bash
# 全量 dump
adb shell dumpsys SurfaceFlinger > sf.log

# 主要结构说明：
# Composition list           合成列表
# Layer Hierarchy            Layer 树形结构
# Display (active) HWC layers: 当前上屏 Layer
# GraphicBufferAllocator buffers: Buffer 内存占用

# 查看刷新率信息
adb shell dumpsys display
```

## ProtoLog（WM 动态日志）

```bash
# 查看所有可用开关
adb shell dumpsys window logging

# 常用开关
adb shell wm logging enable-text WM_DEBUG_ORIENTATION
adb shell wm logging enable-text WM_DEBUG_ADD_REMOVE
adb shell wm logging enable-text WM_DEBUG_STARTING_WINDOW
adb shell wm logging enable-text WM_DEBUG_RESIZE
adb shell wm logging enable-text WM_DEBUG_WINDOW_INSETS
adb shell wm logging enable-text WM_DEBUG_FOCUS_LIGHT WM_DEBUG_FOCUS
adb shell wm logging enable-text WM_DEBUG_CONFIGURATION
adb shell wm logging enable-text WM_DEBUG_WALLPAPER

# 动画相关开关（一次全开）
adb shell wm logging enable-text WM_DEBUG_APP_TRANSITIONS WM_DEBUG_APP_TRANSITIONS_ANIM WM_DEBUG_RECENTS_ANIMATIONS WM_DEBUG_REMOTE_ANIMATIONS WM_DEBUG_WINDOW_TRANSITIONS WM_DEBUG_SYNC_ENGINE

# 过滤 WM log
adb logcat -c && adb logcat | grep -iE "windowmanager:|windowmanagershell:"

# MIUI 动态 log
adb shell wm miuilogging enable-text DEBUG_INPUT_METHOD
adb shell wm miuilogging disable-text DEBUG_INPUT_METHOD
adb shell wm miuilogging enable DEBUG_WALLPAPER

# 开关名称说明
# WM_DEBUG_STARTING_WINDOW    StartingWindow
# WM_DEBUG_ADD_REMOVE         窗口添加和移除
# WM_DEBUG_CONFIGURATION      Configuration 变化
# WM_DEBUG_FOCUS              焦点
# WM_DEBUG_ORIENTATION        方向/转屏
# WM_DEBUG_STATES             窗口状态
# WM_DEBUG_WINDOW_TRANSITIONS 动画相关
```

## Logcat

```bash
# 清除并抓取全量 log
adb logcat -b all -c && adb logcat -b all > o2.log

# 按级别过滤
adb logcat -b all *:W          # Warning 及以上
adb logcat *:E                 # Error 级别

# 按 buffer 过滤
adb logcat -b events           # event log
adb logcat -v time             # system log（带时间）
adb logcat -b main             # main log
adb logcat -b crash            # crash log

# 崩溃排查
adb logcat -b crash
adb logcat | grep -iE "System.err"

# 生命周期
adb logcat -b events -b main -b system | grep -in "start u0\|wm_"

# WM 相关
adb logcat -b all -c && adb logcat -b all | grep -Ei "wm_|WindowManager:"

# 清除缓冲区
adb logcat -c
adb logcat -b all -c

# 完整 bugreport
adb bugreport
```

## SurfaceControl 动态日志（Android V+）

```bash
# 设置监听的方法名（逗号分隔多个）
adb shell setprop persist.wm.debug.sc.tx.log_match_call show,hide,setAlpha,apply,merge
adb shell setprop persist.wm.debug.sc.tx.log_match_call setPosition
adb shell setprop persist.wm.debug.sc.tx.log_match_name com.android.systemui
adb reboot

# 查看日志
adb logcat -s "SurfaceControlRegistry"
adb logcat -b all -c && adb logcat -b all | grep -iE "SurfaceControlRegistry:"

# 关闭
adb shell setprop persist.wm.debug.sc.tx.log_match_call ""
adb shell setprop persist.wm.debug.sc.tx.log_match_name ""
adb reboot
```

## Input

```bash
# 打开 Input 详细日志（级别 1~8）
adb shell dumpsys input debuglog 8

# dump input 状态（焦点窗口、触摸区域等）
adb shell dumpsys input

# 过滤
adb logcat -b all | egrep -iE "InputDispatcher"
adb logcat -b all | egrep -iE "input_focus"
```

## Trace / Winscope

```bash
# 在线 Perfetto：perfetto.pt.xiaomi.com

# Winscope 本地启动（在源码目录）
cd development/tools/winscope
npm install
npm run build:prod
npm run start
python src/adb/winscope_proxy.py   # 启动代理，把 token 粘贴到 Winscope 页面
```

## Feature Flag

```bash
adb shell device_config list | grep -in release_snapshot_aggressively

# 设置 flag 并重启生效
adb shell device_config put windowing_frontend com.android.window.flags.ensure_wallpaper_in_transitions true
adb reboot
```

## Klogg 高亮配置

常见异常关键字：

```
am-crash|FATAL EXCEPTION|NullPointerException|IndexOutOfBoundsException|IllegalStateException|OutOfMemoryError
```

Transition 关键阶段：

```
Creating Transition:|Start collecting|Requesting StartTransition:|Calling onTransitionReady|animated by|Finish Transition:|Transition animation finished
```

DrawState：

```
commitFinishDrawingLocked:|performShowLocked:|showSurfaceRobustly|mDrawState=
```

可见性：

```
setAppVisibility|setClientVisible:|commitVisibility:
```

焦点：

```
Focus (request|requested|receive|entering|changing)
```

Recents：

```
Recents transition|addRecentsFlagForTransition|notifyTransitionMerged|notifyRecentsAnimationFinished|recordMergedTransition
```

过滤器（Klogg）：

```
ShellRecents:|ShellTransitions:|RecentsTransitionHandler:|DefaultMixedHandler:|RemoteTransitionHandler:
BLASTSyncEngine:|Creating Transition:|Start collecting|Finish Transition:|Transition animation finished
KeyguardService:|KeyguardViewMediator:|KeyguardUnlock:|KeyguardTransition:|DisplayStateShaderController:
input_focus:|MIUIInput:|input_interaction:
```

## DUMP 脚本

```bash
#!/bin/bash
adb shell dumpsys window > window.log
adb shell dumpsys activity a > activity.log
adb shell dumpsys SurfaceFlinger > sf.log
adb shell dumpsys display d > display.log
adb shell dumpsys activity containers > containers.log
adb shell dumpsys input > input.log
adb shell dumpsys activity services > services.log
adb shell wm logging enable-text WM_DEBUG_FOCUS_LIGHT WM_DEBUG_FOCUS WM_DEBUG_ORIENTATION WM_DEBUG_DRAW WM_DEBUG_APP_TRANSITIONS WM_DEBUG_WINDOW_TRANSITIONS WM_DEBUG_SYNC_ENGINE WM_DEBUG_ANIM WM_DEBUG_STATES WM_DEBUG_ADD_REMOVE
adb shell dumpsys activity service SystemUIService WMShell WM_SHELL_TRANSITION WM_SHELL_RECENTS_TRANSITION
adb shell setprop persist.wm.debug.sc.tx.log_match_call show,hide,apply,merge,setAlpha,release,reparent,setRelativeLayer
```

## 编译系统

### 源码下载

```bash
repo sync -j20 -c -d --no-tags
repo sync -j4 -c -d --no-tags --no-clone-bundle
# 拉取完整历史（depth=1 时使用）
git fetch miui --unshallow
```

### 编译准备

```bash
source build/envsetup.sh

# V&W 系列
lunch missi-feature_phone_qcom_cn_only64-userdebug    # N2/N3
lunch missi-feature_foldable_qcom_cn_only64-userdebug # N18
lunch missi-feature_flip_qcom_cn_only64-userdebug     # O8
lunch missi-feature_phone_mtk_cn-userdebug            # M12

# T&U 系列
lunch missi_phone_cn-userdebug
lunch missi_pad_cn-userdebug
```

### 模块编译与 Push

```bash
adb root; adb remount

# framework.jar
make framework-minus-apex -j20
adb push $OUT/system/framework/framework.jar /system/framework/

# services.jar
make services -j20
adb push $OUT/system/framework/services.jar /system/framework

# miui-services.jar
make miui-services -j8
adb push $OUT/system_ext/framework/miui-services.jar /system_ext/framework

# miui-framework.jar
make miui-framework -j8
adb push $OUT/system_ext/framework/miui-framework.jar /system_ext/framework

# SurfaceFlinger
make libsurfaceflinger -j80
adb push $OUT/system_ext/lib64/libsurfaceflinger.so /system_ext/lib64/

# MiuiSystemUI
make MiuiSystemUI -j16
adb push out/target/product/missi/system_ext/priv-app/MiuiSystemUI/MiuiSystemUI.apk /system_ext/priv-app/MiuiSystemUI

# 资源文件
make framework-res framework-ext-res -j16
adb push $OUT/system/framework/framework-res.apk /system/framework
adb push $OUT/system_ext/framework/framework-ext-res/framework-ext-res.apk /system_ext/framework/framework-ext-res

# 重启
adb shell stop && adb shell start
```

### 快速编译（ninja）

```bash
cp prebuilts/build-tools/linux-x86/bin/ninja out/host/linux-x86/bin/
cp prebuilts/build-tools/linux-x86/lib64/* out/host/linux-x86/lib64/
ln -sf out/combined-missi.ninja build.ninja

ninja framework-minus-apex -j8
ninja services -j8
ninja miui-framework -j8
ninja miui-services -j8
```

### 系统刷机

```bash
adb devices               # 查看连接设备
adb reboot bootloader     # 进入 bootloader
./flash_all.sh            # 刷机

adb reboot fastboot
adb flash super super.img

adb reboot recovery
```

### idegen.jar

```bash
mmm development/tools/idegen -j16
./development/tools/idegen/idegen.sh
cp development/tools/idegen/idegen.i* .
```

## 日志分析

```bash
# ANR 信息
adb shell dumpsys activity lastanr
adb shell dumpsys activity lastanr-traces

# 内存
adb shell dumpsys meminfo
adb shell dumpsys meminfo --package <packagename>
adb shell cat /proc/meminfo

# Hprof 内存泄漏分析
adb shell am dumpheap com.xxx.xxx /data/local/tmp/oom_heap.hprof
hprof-conv oom_heap.hprof conv_oom_heap.hprof

# Binder
adb shell cat /sys/kernel/debug/binder/transaction_log
```

## 安全权限

```bash
adb shell setenforce 0   # 临时关闭 SELinux
adb shell setenforce 1   # 恢复
make selinux_policy -j4
```

## 系统属性

```bash
adb shell getprop ro.build.version.release     # Android 版本
adb shell getprop ro.build.version.sdk         # API 版本
adb shell getprop ro.product.model             # 设备型号
adb shell getprop ro.product.device            # 设备代号
adb shell getprop ro.build.type                # user/userdebug
adb shell getprop ro.debuggable                # 1=非 user 版本

adb shell settings put system screen_off_timeout 999999999  # 屏幕常亮
adb shell settings put global hidden_api_policy 1           # 调用 hide API
```

## 核心服务

### AMS

```bash
adb shell dumpsys activity a | grep mFocusedApp
adb shell dumpsys activity a | grep mResumedActivity
adb shell dumpsys activity top-resumed
adb shell dumpsys activity containers

adb shell am start -n com.example/.MainActivity
adb shell am force-stop com.miui.xx
adb shell am start -n com.android.settings/com.android.settings.MainSettings --display 1
```

### PMS

```bash
adb shell pm list package              # 所有应用
adb shell pm list package -s           # 系统应用
adb shell pm list package -3           # 三方应用
adb shell pm path 包名                  # APK 路径
adb pull <apk路径>

adb install -r -d <apk>

# 跳过开机引导
adb root
adb shell settings put secure user_setup_complete 1
adb shell settings put global device_provisioned 1
adb reboot
```

### 进程管理

```bash
adb shell dumpsys activity processes
adb shell dumpsys activity oom         # procState 和 adj
adb shell dumpsys activity exit-info   # 进程死亡记录
adb shell am force-stop <pkg>
adb shell kill -9 <pid>

# 启动前断点
adb shell am set-debug-app -w <pkg>
adb shell am start -n <pkg>/MainActivity -D
```

## BugReport

```bash
adb bugreport
grep -Eria "keyword" 文件名 | tee log
adb pull /data/user_de/0/com.android.shell/files/bugreports/
```

## Display

```bash
# 打开帧率显示
adb shell service call SurfaceFlinger 1034 i32 1
```

## 无线 Debug

```bash
adb root
adb shell "setprop persist.adb.tcp.port 5555"
adb tcpip 5555
adb connect <设备IP>
```

## Git

```bash
git add .
git commit -s
repo upload . --no-verify

git blame -L 520,540 WindowContainer.java
git stash && git stash pop
git branch -vv

# cherry-pick 解冲突
git add .
git cherry-pick --continue

# 多笔合一
git rebase -i HEAD~3
```

## 录屏截屏

```bash
adb shell screencap -p /sdcard/screen.png
adb shell screenrecord --time-limit 20 /sdcard/demo.mp4
adb shell screenrecord --bit-rate 6000000 /sdcard/demo.mp4 --bugreport
adb pull /sdcard/demo.mp4

adb root && adb pull /data/system_ce/0/snapshots
```

## Linux 工具

```bash
ncdu              # 目录内存占用
du -sh .          # 当前文件夹

sudo chown mi:mi adb.1000.log
sudo chmod 777 <file>
chmod a+x flash_all.sh

sudo apt install openjdk-11-jdk
sudo update-alternatives --config java
```

## 平行窗口

```bash
adb shell pm clear com.android.htmlviewer        # 清理云控 sp
adb shell cmd miui_embedding_window dump-rule <pkg>
adb shell cmd miui_embedding_window debug-bank

adb root; adb remount
adb push cloudFeature_embedded_rules_list.xml data/system/
adb shell cmd miui_embedding_window update-rule

adb shell cmd miui_embedding_window update-rule <pkg> fullRule::*        # 全屏
adb shell cmd miui_embedding_window update-rule <pkg> fullRule::null     # 取消全屏
adb shell cmd miui_embedding_window update-rule <pkg> sizecompatRule::*  # 内缩居中
adb shell cmd miui_embedding_window disable-system-override <pkg>        # 去除系统反向适配
```

## FixedOrientation

```bash
adb shell wm set-ignore-orientation-request true    # 开启横屏适配
adb shell wm set-ignore-orientation-request false   # 关闭
adb shell wm get-ignore-orientation-request

adb shell cmd miui_embedding_window set-fixedOri <pkg> disable::true  # 禁止进入 FO
adb shell cmd miui_embedding_window set-fixedOri <pkg> ratio::1.5     # 设置比例
adb shell cmd miui_embedding_window set-fixedOri <pkg> allPortrait::true  # 强制竖屏
adb shell cmd miui_embedding_window clear-fixedOri <pkg>              # 清除配置
```

## 大屏投屏

```bash
adb shell cmd miui_embedding_window_projection dump-rule <pkg>
adb shell cmd miui_embedding_window_projection update-rule
# mode: 0=Default, 1=ActivityEmbedding, 2=FixedOrientation, 3=FullScreen
adb shell cmd miui_embedding_window_projection set-appMode com.miui.carlink <pkg> <mode>
```
