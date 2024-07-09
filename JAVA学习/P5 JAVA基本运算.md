# P5 JAVA基本运算

# 1.算术运算符

## 1.1 隐式转换

把取值范围小的转换成取值范围大的数值。

不同类型的值对应的二进制的位数是不一样的。

![image-20240324142438799](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324142438799.png)

**注意三点：**

- 取值范围：byte < short < int < long < float < double
- 转化条件：数值类型不一样，需要进行运算，需要转换成一样才能进行转换。
- 转换规则：取值范围小的和取值范围大的进行运算，小的会提升成大的再进行运算；byte char short三种类型在进行运算时会先提升成int再进行运算。

![image-20240324132652063](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324132652063.png)

```
public class Test {
    public static void main(String[] args) {
    	//小转大，a+b是double型
        int a = 20;
        double b = 30.0;
        System.out.println(a + b);
        
        //byte类型转换，转换成int型
        byte c = 10;
        byte d = 20;
        System.out.println(c + d);
    }
}
```

## 1.2 显式转换（强制转换）

取值大的数值向取值范围小的进行转换。

```
public class Test {
    public static void main(String[] args) {
        int a = 20;
        byte e = (byte)a;
        System.out.println(e);
    }
}
输出:
20

```

int转byte，截取高位，保留低位。最高位是符号位

```
public class Test {
    public static void main(String[] args) {
        int a = 200;
        byte e = (byte)a;
        System.out.println(e);
    }
}
输出：
-56
```

![image-20240324142842687](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324142842687.png)

# 2.字符串的“+”运算符

当+操作中出现字符串时候，这个+就变成了字符串连接符，而不是算数运算符。会将前后的数据进行拼接，产生新的字符串。

```
public class Text {
    public static void main(String[] args) {
        System.out.println("123"+123);
    }
}

输出是：123123
```

连续进行+操作时候会自左向右执行

```
public class Test {
    public static void main(String[] args) {
        System.out.println(1 + 98 + "offer");
        System.out.println(1 + 98 + "offer" + 98 + 1);
    }
}
输出是：
99offer
99offer981
```

字符相加或者字符加整数时候，会去ASCII表找到相应值相加

```
public class Test {
    public static void main(String[] args) {
    	//'a'对应的ascii值是97，+1就是输出的98
        System.out.println(1+'a');
        //相当于字符串拼接
        System.out.println('a'+"abc");
    }
}
输出是：
98
aabc
```



# 3.运算符

## 3.1 短路逻辑运算符

&&：短路逻辑与，左侧为false无需判断右侧

||：短路逻辑或，左侧为true无需进行右侧运算

```
public class Test {
    public static void main(String[] args) {
        int a = 5;
        int b = 10;
        //左侧条件不满足，因此右侧b不进行自增
        boolean c = ++a < 5 && ++b < 10;
        System.out.println(" a = " + a + " b = " + b + " c = " + c);
        //左侧条件满足，所以右侧不需要判断
        boolean d = a > 5 || ++b < 10;
        System.out.println(" a = " + a + " b = " + b + " c = " + c);

    }
}
输出为：
 a = 6 b = 10 c = false
 a = 6 b = 10 c = false
```

## 3.2 普通逻辑运算符

&：逻辑与，两侧同为true，结果是true，两侧都要运算

|：逻辑或，两侧有一位true，结果是true，两侧都要运算

![image-20240324143059076](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324143059076.png)

## 3.3 左移运算符 右移运算符 无符号右移运算符

<<：左移，向左移动，低位补0，相当于乘以0操作

'>>'：右移，向右移动，高位补0或者1，正数补0，负数补1，相当于除以2操作

'>>>'：无符号右移，高位补0

-1的右移永远是-1





# 4.原码 反码 补码

计算机中的存储和计算都是以补码的形式进行的。



一个字节的最大值是127，最小值是-128

原码？

最高位是符号位，0为正，1位负。



反码？

反码是为了解决原码不能计算负数的问题出现的。

正数的反码不变，负数的反码是符号位不变，其他位取反。数值取反，0变1，1变0。



反码的弊端是：负数运算时，如果结果不跨0，没有问题，如果结果跨0，跟实际结果会有1的偏差。



补码？



补码的出现就是解决负数计算时候跨0的问题而出现的。

正数的补码不变，负数的补码是反码基础上加1.



# 5.switch语句新特性

```java
public class Test {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.println("请输入你要选择的服务：");
        int choose = sc.nextInt();
        switch (choose) {
            case 1:
                System.out.println("机票预订");
                break;
            case 2:
                System.out.println("机票打印");
                break;
            case 3:
                System.out.println("人工服务");
                break;
            default:
                System.out.println("没有该选项");
                break;
        }
        
        /*与上面表达意思一样，用->{语句}来进行替代，更加简洁，只有一行时候可以删除大括号。
        switch (choose) {
            case 1 -> {
                System.out.println("机票预订");
            }
            case 2 -> {
                System.out.println("机票打印");
            }
            case 3 -> {
                System.out.println("人工服务");
            }
            default -> {
                System.out.println("没有该选项");
            }
        }
        */
       /*
       switch (choose) {
            case 1 -> System.out.println("机票预订");
            case 2 -> System.out.println("机票打印");
            case 3 -> System.out.println("人工服务");
            default -> System.out.println("没有该选项");
        }
        */
    }
}
```

# 6.生成随机数



```
//导入生成随机数的包
import java.util.Random;

public class Test {
    public static void main(String[] args) {
    	//创建一个随机数对象
        Random r = new Random();
        //生成一个[0,100)的随机数，前开后闭，包头不包尾
        int target = r.nextInt(100);
        //生成[100,199)的随机数
        int target2 = r.nextInt(100) + 100;

    }
}
```

