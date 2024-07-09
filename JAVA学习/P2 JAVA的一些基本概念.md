# P2 JAVA的一些基本概念

注意：

1）安装软件时注意目录中不要有**空格和中文字符**

2）保存环境变量

设置->系统->关于->高级系统设置->环境变量->系统变量->Path->编辑->新建->添加javac.exe和java.exe所在目录->保存

但是对于java可以按照以下配置，方便使用java的jdk其他的exe（单个编译器下的路径貌似不可用）

![image-20240323102710361](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240323102710361.png)



# 1.JAVA常见名词及基础概念 

## 1.1 JDK、JVM和JRE是什么

**JDK(JAVA Development kit)：**Java开发工具包，需要写代码运行，安装JDK即可。

JDK的bin目录下存放了各种工具命令，常用的是javac（编译的命令）和java（运行的命令）

**JRE（Java Runtime Environment）** :JAVA运行环境，包含JVM、核心类库和（java.exe）运行工具。假如代码已经编译好，只需要安装JRE即可直接运行，无需安装JDK，内存更小。

**JVM（java virtual machine)**：JAVA虚拟机，是代码真正运行的地方。



## 1.2  JDK、JVM和JRE关系是什么

JDK包含JRE，JRE包含JVM。

JDK比JRE多了一些开发工具。

JRE比JVM多了核心类库和运行工具。

![image-20240323110834025](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240323110834025.png)





## 1.3 JAVA SE ME EE是什么

JAVA SE是java语言标准版

![image-20240323103824388](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240323103824388.png)

JAVA ME是JAVA小型版，已经很少使用，被android和ios替代

![image-20240323103921515](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240323103921515.png)



JAVA EE是java企业版

![image-20240323103959786](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240323103959786.png)

# 1.4 JAVA能做什么

![image-20240323104218755](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240323104218755.png)

### 1.4.1 桌面应用开发 IDEA

### 1.4.2 企业级应用开发  SPRINGCLOUD

### 1.4.3 移动应用开发 android

### 1.4.4 科学计算 MATLAB

### 1.4.5 大数据开发 hadoop

### 1.4.6 游戏开发 我的世界

 ![image-20240323104436595](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240323104436595.png)

## 1.5 JAVA版本更新

java5.0是一个大版本更新，java8.0是常用的版本

![image-20240323103527164](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240323103527164.png)

## 1.6 JAVA为什么这么火

### 1.6.1 java主要特性

![image-20240323104757604](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240323104757604.png)



1）面向对象

三大特征：封装，继承，多态。

2）安全性

问题较少，稳定

3）多线程

效率高，例如12306多人可以同时抢票

4）跨平台

可在多平台windows,mac和linux使用

![image-20240323105219086](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240323105219086.png)

5）开源

会有安装包和代码的提供

6）简单易学

api多

### 1.6.2 JAVA为什么可以跨平台

JAVA跨平台的原理：虚拟机，JAVA不是运行在系统中的，而是运行在虚拟机中的，例如window系统可以安装虚拟机玩手机游戏。

![image-20240323105718872](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240323105718872.png)

![image-20240323105555864](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240323105555864.png)



# 4.常用的IDE和记事本

IDE就是集成开发环境：把代码编写，编译执行以及测试多种功能综合成一起的开发环境。



1）Android Studio  

2）IDEA

3）Visual Studio

4）Eclipse



记事本

1）notepad++

2）sublime



# 5.编译和运行

什么是编译？

java文件是不能够直接被window和linux直接识别的，编译就是将java文件转换成系统可以识别的二进制文件。

```
javac helloworld.java   	//编译生成helloworld.class文件

java helloworld				//运行编译后的class文件
```

