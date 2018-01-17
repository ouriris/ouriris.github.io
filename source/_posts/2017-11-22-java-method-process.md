---
title: Java--方法调用(进击的Java新人Week.3学习笔记)
date: 2017-11-22 19:47:09
tags:
- Java
- 进击的Java新人
---
Author: Oscar

### 目录
- 函数与方法的区别
- Java调用方法的过程

# 函数与方法的区别
> 函数与方法在不同的场景下，叫法不同。函数一般出现在面向过程，比如C语言，通过函数名就可以调用这个函数。而方法一般出现在面向对象，与对象相关联，通过对象调用里面的方法。

# Java调用方法的过程
> 当Java启动一个线程的时候，就会创建一个栈。每当线程执行一个方法的时候，就会压入一个栈帧。只有处于栈顶的栈帧才是最有效的，称为当前栈帧，即当前方法。**注意，当前方法没有返回或抛异常中断时，当前栈帧是不会出栈的**。

> 栈帧：主要由四部分组成，分别是局部变量表（Local Stack Frame）、操作数栈（Operand Stack）、动态链接（Dynamic Linking）以及返回地址（Reture Address）。
> + 局部变量表：保存函数的参数以及局部变量用
> + 操作数栈：存放一些操作后的结果<br>

**示例**<br>
### Java Code : <br>
```java
public class Main {
    public static void main(String args[]) {
        int a = 1;
        int b = 2;
        int t = a + b;
    }
}
```
### ByteCode
```byteCode
0: iconst_1  //把常量1 放到操作数栈顶
1: istore_1  //从操作数栈取出栈顶的值，并将其存入局部变量表里的第一个位置，可以理解为索引为1
2: iconst_2  //把常量2 放到操作数栈顶
3: istore_2  //从操作数栈取出栈顶的值，并将其存入局部变量表里的第二个位置，可以理解为索引为2
4: iload_1   //把局部变量表的第一个位置，及索引为1的值，放入操作数栈的栈顶
5: iload_2   //把局部变量表的第二个位置，及索引为2的值，放入操作数栈的栈顶
6: iadd      //把操作数栈的两个数取出来，求和，再把结果放入操作数栈的栈顶
7: istore_3  //从操作数栈取出栈顶的值，并将其存入局部变量表里的第三个位置，可以理解为索引为3
8: return
```
>栈帧组成结构

![](https://raw.githubusercontent.com/zhengdunhao/UploadFiles/master/wKiom1Uwf7GTNgVeAABCHdXGE6o641.jpg)



### 习题一
```java
public class Main {
    public static void main(String args[]) {
        int a = 1, b = 2;
        swap(a, b); 
        System.out.println("a is " + a + ", b is " + b); 
    }   

    public static void swap(int a, int b){ 
        int t = a;
        a = b;
        b = t;
    }   
}
```
#### 方法调用过程
1.当主线程启动的时候，就会创建一个栈。执行main函数，创建main帧，并压入栈。
![](https://raw.githubusercontent.com/zhengdunhao/UploadFiles/master/p1.png)

2.当主线程执行swap方法时，创建swap帧，并压入栈，**注意：同时把main帧的a=1,b=2的数值也拷贝一份，在swap帧也创建了变量a和b，因此，修改swap帧的变量，也不会影响main帧的变量。经过一系列操作，swap帧里的变量a=2，b=1，t=1。这是传值，适用于Java的基础数据类型，如int,float,double等等**

![](https://raw.githubusercontent.com/zhengdunhao/UploadFiles/master/p2.png)

3.当执行完swap方法时，swap帧就会出栈，swap帧里的变量a,b,t也会随着出栈而被回收。
![](https://raw.githubusercontent.com/zhengdunhao/UploadFiles/master/p1.png)


### 习题二
```java
public class Main {
    public static void main(String args[]) {
        int n = 5;
        int t = fact(n);
        System.out.println(t);
    }

    public static int fact(int n){
        if (n == 0)
            return 1;
        else
            return n * fact(n - 1);
    }
}
```
#### 方法调用过程
![](https://raw.githubusercontent.com/zhengdunhao/UploadFiles/master/20130108112638885.png)