# P7 JAVA内存分配

JAVA内存分为五个部分

![image-20240324211411365](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324211411365.png)

最重要的就是堆内存和栈内存。

![image-20240324211751602](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324211751602.png)



**栈：**方法运行时使用的内存，比如main方法运行，进入方法栈中执行。

**堆：**存储对象或者数组，new创建的，都存储在堆内存。

**方法区：**存储可以运行的class文件。

**本地方法栈：**JVM在使用操作系统功能时候使用，和我们开发无关

**寄存器：**给CPU使用，和我们开发无关。



# 1.栈内存

![image-20240324211940810](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324211940810.png)

会先在栈内存区申请一块内存，开辟a的空间，b的空间，计算a+b的值放到c的空间，输出到控制台。



# 2.数组的内存图

这里的栈内存中的arr记录的就是堆内存中的地址值。

## 2.1 数组指向单独的内存

![image-20240324212413797](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324212413797.png)



## 2.2 两个数组指向同一个空间的内存

修改的数组值是同一个空间的内存值。当两个数组指向同一个小空间时候，其中一个数组对小空间的值发生改变，那么其他数组再次访问的时候就是修改之后的结果了。

![image-20240324212752209](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240324212752209.png)