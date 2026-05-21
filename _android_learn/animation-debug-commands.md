---
layout: private_post
title: "动画调试常用命令"
date: 2026-05-21
categories: android-learn
tags: [animation, debugging, adb, build]
render_with_liquid: false
---

# 动画调试常用命令

## 编译环境配置

```bash
# 新打开终端时需执行
export BUILD_TARGET_PRODUCT=star   # 替换成自己的机型
source build/envsetup.sh
lunch XXX-userdebug

# S版本首次编译
make framework-minus-apex -j8
make services -j8
make miui-services -j8
make miui-framework -j8

# T/U版本首次编译
repo sync -j12 -d -c --no-tags --force-sync
make services -j8 && make framework-minus-apex -j8 && make miui-services -j8 && make miui-framework -j8
make idegen -j5 && source development/tools/idegen/idegen.sh
```

## 快速编译（ninja）

```bash
# 配置 ninja 环境
cp prebuilts/build-tools/linux-x86/bin/ninja out/host/linux-x86/bin/
cp prebuilts/build-tools/linux-x86/lib64/* out/host/linux-x86/lib64/
ln -sf out/combined-missi_phone_cn.ninja build.ninja

ninja framework-minus-apex -j8
ninja services -j8
ninja miui-framework -j8
ninja miui-services -j8

# S版本 quickbuild
quickbuild framework-minus-apex -j8  && sh pushframeworkmissi.sh
quickbuild services -j8 && sh pushservicesmissi.sh
quickbuild MiuiSystemUI -j12 && sh pushSystemUi.sh
quickbuild miui-services -j8 && sh pushmiservices.sh
quickbuild miui-framework -j8 && sh pushmiframework.sh

# T版本 ninja+push
ninja framework-minus-apex -j8 && sh t_push.sh
ninja services -j8 && sh t_push.sh
```

## 模块 Push 到手机

```bash
adb root && adb disable-verity && adb reboot
adb root && adb remount

# framework.jar
adb push $OUT/system/framework/framework.jar /system/framework/

# services.jar
adb push $OUT/system/framework/services.jar /system/framework

# miui-services.jar
adb push $OUT/system_ext/framework/miui-services.jar /system_ext/framework

# miui-framework.jar
adb push $OUT/system_ext/framework/miui-framework.jar /system_ext/framework

# 重启生效
adb shell stop && adb shell start
```

## 动画调试开关

```bash
# 查看所有可用开关
adb shell dumpsys window logging

# 打开动画相关 ProtoLog 开关
adb shell wm logging enable-text WM_DEBUG_APP_TRANSITIONS WM_DEBUG_APP_TRANSITIONS_ANIM
adb shell wm logging enable-text WM_DEBUG_REMOTE_ANIMATIONS WM_DEBUG_RECENTS_ANIMATIONS
adb shell wm logging enable-text WM_DEBUG_WINDOW_TRANSITIONS WM_DEBUG_SYNC_ENGINE
adb shell wm logging enable-text WM_DEBUG_BACK_PREVIEW

# 关闭开关
adb shell wm logging disable-text WM_DEBUG_FOCUS WM_DEBUG_FOCUS_LIGHT

# MIUI 自定义动态开关
adb shell wm miuilogging enable-text DEBUG_INPUT_METHOD
adb shell wm miuilogging disable-text DEBUG_INPUT_METHOD
adb shell wm miuilogging enable DEBUG_WALLPAPER
```

## 动画调试常用 ADB 命令

```bash
# 过滤动画相关 log
adb logcat -c && adb logcat | grep -iE "applyanimation:"
adb logcat -c && adb logcat | grep -iE "WindowManager:"
adb logcat -c && adb logcat | grep -iE "windowmanager:|windowmanagershell:|BLASTSyncEngine:|ShellRecents"

# 查看 opening/closing app
adb logcat -c && adb logcat | grep -iE "Now opening app"
adb logcat -c && adb logcat | grep -iE "Now closing app"
adb logcat -c && adb logcat | grep -iE "handleAppTransitionReady:"

# 查看桌面是否收到 remote animation 回调
adb logcat -c && adb logcat | grep -iE "windowmanager:|applyanimation:|LauncherAnimationRunner:"

# 查看 good to go 和 setWallPaperAsTarget
adb logcat -c && adb logcat | grep -iE "good to go|setWallPaperAsTarget"

# 连续 dump SurfaceFlinger（排查闪黑）
adb shell "while true; do dumpsys SurfaceFlinger; sleep 0.016;done" > dump_SF.log
adb shell "while true; do dumpsys window w; sleep 0.016;done" > dump_window.log
adb shell "while true; do dumpsys window | grep -Ei 'mWallpaperTarget'; sleep 0.016;done" > dump_wallpaper.log

# 设置动画速度（放慢5倍便于分析）
adb shell settings put global transition_animation_scale 5.0
adb shell settings put global window_animation_scale 5.0
adb shell settings put global animator_duration_scale 5.0

# 查看当前 activity
adb shell dumpsys activity a | grep mResumedActivity   # S版本
adb shell dumpsys window w | grep -iE "mResumeActivity"  # T版本

# 查看 WallpaperTarget
adb shell dumpsys window | grep -Ei "mWallpaperTarget"

# 查看 SurfaceFlinger 图层和 HWC layers
adb shell dumpsys SurfaceFlinger | grep -ie "hwc l" -A30 -B20

# 同时过滤 wm_ 和 windowmanager
adb logcat -b all | grep -v "prepareSurface:" | grep -iE "start u0|windowmanager:|Starting remote|LauncherAnimationRunner:|mdrawstate|wm_"
```

## SurfaceControl 动态日志（Android V+）

```bash
# 开启指定方法的日志
adb shell setprop persist.wm.debug.sc.tx.log_match_call setAlpha
adb shell setprop persist.wm.debug.sc.tx.log_match_call show,hide,apply,merge,setAlpha,release,reparent,setRelativeLayer
adb shell setprop persist.wm.debug.sc.tx.log_match_name com.android.systemui
adb reboot

# 查看日志
adb logcat -s "SurfaceControlRegistry"
adb logcat -b all -c && adb logcat -b all | grep -iE "SurfaceControlRegistry:"

# 关闭（清空属性值）
adb shell setprop persist.wm.debug.sc.tx.log_match_call ""
adb shell setprop persist.wm.debug.sc.tx.log_match_name ""
adb reboot
```

## WinScope 使用

```bash
# 导航到 Winscope 文件夹
cd development/tools/winscope

# 安装依赖
npm install

# 构建
npm run build:prod

# 启动
npm run start

# 启动代理（需在源码目录）
python src/adb/winscope_proxy.py
# 复制终端中出现的 token 到 Winscope 页面
```

也可访问 `perfetto.pt.xiaomi.com` 在线抓取分析。

## ROM 刷机步骤

```bash
# 进入 fastboot 模式刷机
adb reboot bootloader
sudo fastboot devices
chmod a+x flash_all.sh
sudo ./flash_all.sh

# 跳过小米账号
adb root && adb shell pm disable com.xiaomi.account

# 跳过开机引导
adb root
adb shell settings put secure user_setup_complete 1
adb shell settings put global device_provisioned 1
adb reboot
```

## Git 提交代码

```bash
git add .
git commit -s
repo upload . --no-verify

# cherry-pick 解冲突后继续
git add .
git cherry-pick --continue
repo upload . --no-verify

# 多笔合一
git log .                   # 找到第一笔之前的 commit id
git rebase -i <commit_id>   # 将后面的 pick 改为 s

git branch -vv
```

## 打印 log 示例

```java
// 打印堆栈（Java）
Slog.d("TAG", "CONTACT- openingWcs.size() = " + openingWcs.size()
        + "\n" + Log.getStackTraceString(new Throwable()));
```

```bash
# 录屏并同步抓日志（两个终端）
adb shell screenrecord --bugreport sdcard/record.mp4
adb logcat -b all -c && adb logcat -b all > bugreport.log
```
