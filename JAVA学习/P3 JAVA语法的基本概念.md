# JAVA语法的基本概念

# 1.注释

是对代码进行解释 说明，分单行注释、多行注释和文件注释。

特点是不参与编译和运行，知识对代码的解释说明，也就是在class文件不存在的。



# 2.字面量

是告诉程序员在程序中的书写格式，直接是一个值而不是一个变量。

![image-20240324091500373](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324091500373.png)

特殊字面量的书写，制表符 \t和空类型null

```java
public class ValueDemo {
    public static void main(String[] args) {
        System.out.println("name" + "\t" + "age");
        System.out.println("tom " + "\t" + "12");
    }
}
```

# 3.变量

当某一个数据经常发生改变时候，我们可以用变量进行存储。当数据变化时候，只需要修改里面的值即可。

![image-20240324093251912](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324093251912.png)

**变量的注意事项**

- 只能存储一个值

- 变量名不允许重复定义

- 一条语句可以定义多个变量  int a = 20,b = 100;

- 变量在使用前必须要赋值

  定义时直接赋值  int a = 20;

  先定义后赋值      int a; a = 20;



# 4.计算机存储规则

**计算机中存储三种类型的数据：**

Text文本，Image图片，Sound声音。注意视频是由图片和声音组成的。

![image-20240324100315066](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324100315066.png)

## 二进制存储

在计算机中，任意数据都是以二进制的形式存储的。

![image-20240324094222657](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324094222657.png)

## 那为什么要用二进制存储数据呢？

最开始是打孔纸带，打孔为1，否则为0。只有两种状态就能表示数据，如果用十进制那就需要用十种状态表示数据，太麻烦了。



## Text文本

### ASCII码表

A-Z：65-90

a-z：97-122

‘0’-‘9’：48-57

![image-20240324094843727](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324094843727.png)

## 图像数据

### 像素

手机像素：1920*1080，就是表示宽有1920个点，高度有1080个点。



### 三原色

计算机采用的颜色是光学三原色，红绿蓝（RGB），可以写成10进制或者16进制的形式。

红色：(255,0,0)或者（FF0000)

绿色：(0,255,0)或者（00FF00)

蓝色：(0,0,255)或者（0000FF)

黑色：(000000)

白色：(FFFFFF)



## 声音数据

![image-20240324100152231](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324100152231.png)



# 5.数据类型

## 5.1 基本数据类型

共有八种数据类型，分成四类。基本数据类型里面存储的是个真实的数据，真实的值，数据存储在自己的空间中。

![image-20240324100645419](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324100645419.png)

long型可以在数值后面加一个大写L或者l,一般用大写表示。

float的取值范围比long要大，

https://blog.csdn.net/qq_39288456/article/details/104496479

https://blog.csdn.net/qq_42712280/article/details/84980598

```
long l = 99999L;
float f = 1213.2F;
double d = 1523.2356D;
```



## 5.2 引用数据类型

变量中存储的是地址值，或者说存储的**是其他空间里面的数据。**

通过new出来的变量的数据类型都是引用数据类型。

![image-20240324215230553](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324215230553.png)



## 5.3 区别

![image-20240324215405461](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324215405461.png)



# 6.标识符

标识符：给类、方法以及变量起的名字。

我们常遵循阿里巴巴规范。

## 6.1 命名硬性要求

- 由数字、字母、下划线_和美元符$组成。
- 不能以数字开头， double $ddd也是可以的。
- 不能是关键字
- 区分大小写



## 6.2 标识符软性要求

**见名知意**

小驼峰命名法：方法，变量名

大驼峰命名法：类名

![image-20240324110934522](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324110934522.png)



# 7.键盘录入

Scanner类，是java写好的一个类，可以接受键盘输入的数字。

![image-20240324111303955](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324111303955.png)

```
import java.util.Scanner;

public class ValueDemo {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.println("请输入第一个数字：");
        int a = sc.nextInt();
        System.out.println("请输入第二个数字：");
        int b = sc.nextInt();
        System.out.println("两数之和是 = " + (a+b));

    }
}
```

