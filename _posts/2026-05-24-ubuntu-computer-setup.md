---
layout: post
title:  "Ubuntu 电脑配置"
date:   2026-05-24 00:00:00 +0800
categories: linux
tag: Ubuntu
---

# 1.安装截图工具 flameshot

1. sudo apt update
2. sudo apt install flameshot
3. flameshot gui #命令启动
4. #设置快捷键 ctrl+alt+A

自定义到flameshot快捷键
如何配置：需要前往 Ubuntu 设置 > 键盘 > 查看及自定义快捷键 > 自定义快捷键，添加一个指向命令 `flameshot gui` 的新快捷键，然后按下你想要的组合键（如 `Ctrl + Alt + A`）即可

# 2.MPV视频播放

1. sudo apt install mpv
2. mpv --version #确认安装的版本

mpv安装后记得将视频类应用的默认打开方式改为mpv
打开系统设置->详细信息->默认应用程序
逐帧播放一般使用键盘上的 , .

# 3.安装搜狗输入法
https://chat.deepseek.com/share/cj8whpplmpw9jg6gm0

# 4.安装永久破解版Idea
https://zhuanlan.zhihu.com/p/1932770777503077742
1.官网直接下载
https://www.jetbrains.com/idea/download/?section=linux

1. uname -m

如果输出结果包含 "arm"，则表示你的计算机是 ARM 架构。
如果输出结果包含 "x86" 或 "amd"，则表示你的计算机是 AMD 或者 x86/x64 架构。
2.安装
1. sudo tar -xzf idea-2025.3.3.tar.gz -C /opt   #解压
2. cd /opt/idea-IC-xxx/bin        #进入目录，请将 idea-IC-xxx 替换为你的实际文件夹名
3. ./idea.sh            #执行启动脚本

3.破解前须知
破解前，请务必删除多余的其他破解工具和文件包，确保只有本教程中唯一的工具（先执行激活工具的卸载脚本）。避免在激活时，遇到各种奇葩问题。如果你在操作中一直无法成功，请检查下面路径下文件，并删除多余的javaagent。
1. - windows: "%APPDATA%\JetBrains\IntelliJIdea2025.1\idea64.exe.vmoptions" 
2. - macOS: "~/Library/Application Support/JetBrains/IntelliJIdea2025.1/idea.vmoptions"
3. - Linux: "~/.config/JetBrains/IntelliJIdea2025.1/idea64.vmoptions"

# 4.下载破解工具
双击下面文件选择要激活的即可

1. chmod 755 jetbra-free-linux-amd64 

5.桌面添加idea快捷方式
https://chat.deepseek.com/share/6gxtwc4s5d21mkb04e

# 5.安装破解版typora
最后一个可以直接使用的破解版 https://github.com/wyf9661/typora-free
windows版本：
https://github.com/wyf9661/typora-free/blob/master/typora-setup-x64-0.11.18.exe

如果安装后无法使用：

https://xiaoniuhululu.com/2022-07-28_Typora_isExpired_deal

先打开typora的注册表看看，里面是什么情况

1. 打开注册表：按`Windows+R`打开运行窗口，输入` regedit`
2. 进入路径`计算机\HKEY_CURRENT_USER\SOFTWARE\Typora`

![](/images/ubuntu-config-typora.PNG)

linux版本：
安装指令

1. sudo dpkg -i typora

# 6.android studio设置导航后退键和前进键
打开设置：点击 File -> Settings （在 macOS 上是 Android Studio -> Preferences）。
进入菜单栏和工具栏设置：在设置界面中，导航到 Appearance & Behavior -> Menus and Toolbars。
选择工具栏位置：在右侧列表中找到 Main Toolbar。你可以选择将按钮放在 Left、Center 或 Right 分组下（通常选择 Left 以便于操作）。
添加按钮：
在你选择的分组上点击鼠标右键，选择 Add Action。
在弹出的对话框中，依次展开 Main Menu -> Navigate。
先选中 Back，点击 OK 添加"返回"按钮。
重复上述步骤，再次选择 Forward，添加"前进"按钮。
完成：点击 OK 或 Apply 保存设置。此时，前进和返回键就会出现在你指定的工具栏位置了。

# 7.andorid studio修改快捷键
File->Settings->keyMap

# 8.安装klogg
klogg官网：https://klogg.filimonov.dev/
