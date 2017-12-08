---
title: 进击的Java新人Week.3学习笔记(位操作)
date: 2017-11-23 20:10:16
tags: Java
---
Author: Oscar

### 目录
+ 基础题目
+ 计算二进制数中1的个数
+ 异或
+ 巧解找数字问题

# 基础题目
当执行乘法，除法和模运算时，有出现关于2的整数次幂的，往往可以使用位操作来解决。例如，求n mod 16，比较好的写法是
```java
int t = n & 0xf; // not n % 16
```


判断一个数的奇偶性
```java
// 奇数返回 true
boolean isOdd(int n) {
    return n & 1 == 1;
}
```
对整数n乘(除)以2的整数次幂，可以使用左(右)移来实现，而不是直接乘或者除。有一道笔试题，不准使用乘法，实现 n*7 ，答案是
```java
n << 3 − n;
// 当然，也可以使用 n << 2 + n << 1 + n，这种办法要差一些。
```
# 计算二进制数中1的个数
一个整数，要求去掉它的二进制表示法的最后一次出现的1，例如， 110，去掉最后一个1就是100。
```java
int expellOne(int a) {
    //a的最后一个1的位置，对应于a-1是0
     return a & (a − 1);
}
```

统计整数转成二进制后总共有多少位是1。例如110，统计的结果就是2。
```java
int calcOnes(int a) {
    int count = 0;
    while(a != 0) {
        a = expellOne(a);
        count++;
    }
    return count;
}
```
# 异或
异或是按位操作，相同为0，不同为1。异或有三个特点：
1. 0^0=0,0^1=1  0异或任何数＝任何数
2. 1^0=1,1^1=0  1异或任何数－任何数取反 
3. 任何数异或自己＝把自己置0 <br>

因此，在汇编语言中，经常使用“xor ax, ax”这样的语句来清零寄存器。异或遵循结合律和交换律，从而有“a xor b xor a = a xor a xor b = (a xor a) xor b = 0 xor b = b”。

不使用第三个变量，交换整型变量 a 和 b 的值。可以使用异或来解决。代码是:
```java
a = a ˆ b, b = a ˆ b, a = a ˆ b;
```

# 巧解找数字问题
一个整型数组里除了一个数字之外，其他的数字都出现了两次。请找出这个数字。

我们可以使用异或来解决这个问题，把数组里的所有元素全部进行异或操作。由于异或操作的交换律和结合律我们知道，所有出现了两次的数字，都与自己先结合进行运算，那么结果就是0，最后，剩下的那个数字就是要找的数字。
```java
public class Demo03_05_02 {
    public static void main(String[] args) {
        int [] nums = {1,3,5,6,8,9,5,6,1,3};
        int result = 0;
        for(int num : nums){
            result^=num;
            System.out.println("result:"+result);
        }
    }
}
```
如果数组中只出现了一次的数字有两个，而其他数字都出现了两次呢？

思路是，如果能够把原数组分为两个子数组。在每个子数组中，包含一个只出现一次的数字，而其他数字都出现两次。如果能够这样拆分原数组，按照上一题的办法就可以分别求出这两个只出现一次的数字了。

还是从头到尾依次异或数组中的每一个数字，那么最终得到的结果就是两个只出现一次的数字的异或结果。因为其他数字都出现了两次，在异或的过程中全部抵消掉了。由于这两个数字不一样，那么这个异或结果肯定不为0，也就是说在这个结果数字的二进制表示中至 少就有一位为1。我们在结果数字中找到第一个为1的位的位置，记为第 N 位。这两个数字不 同，意味着为1的那个位是相异的。现在我们以第 N 位是不是 1 为标准把原数组中的数字分成两个子数组，第一个子数组中每个数字的第 N 位都为1，而第二个子数组的每个数字的第 N 位都为0。
```java
public class Demo03_05_03 {

    public static void main(String[] args) {
        int[] nums = {2, 3, 6, 4, 5, 3, 4, 6};
        findNums(nums, nums.length);
    }

    public static int findNums(int[] data, int length) {
        int num1 = 0;
        int num2 = 0;

        if (length < 2)
            return -1;

        int ansXor = 0;
        for (int i = 0; i < length; i++)
            ansXor ^= data[i];               //异或

        int pos = findFirstOne(ansXor);
        for (int i = 0; i < length; i++) {
            if (testBit(data[i], pos))
                num1 ^= data[i];
            else
                num2 ^= data[i];
        }
        System.out.println("the two numbers are " + num1 + " and " + num2);
        return 0;
    }

    public static int findFirstOne(int value)  //取二进制中首个为1的位置
    {
        int pos = 0;
        while ((value & 1) != 1) {
            value = value >> 1; //向右移一位
            pos++;
        }
        return pos;
    }

    public static boolean testBit(int value, int pos) //测试某位置是否为1
    {
        return ((value >> pos) & 1) != 0;
    }
}
```

1-1000中少了两个数，找出没有出现的两个数，数组是无序的。
```java
public class Demo03_05_04 {

    public static void main(String[] args) {
        int[] num = {1, 3, 5, 7, 2, 6, 9, 0, 0, 10};
        int num1 = 0;
        int num2 = 0;
        findLostTwo(num, num.length, num1, num2);
    }

    public static void findLostTwo(int num[], int length, int num1, int num2) {
        int temp = 0;
        int i;
        //返回的temp就是缺失的两个数的异或
        for (i = 0; i < 10; i++) {
            temp ^= (i + 1);
            temp ^= num[i];
        }
        int first_not_zero = 1;
        while ((temp & 1) == 0) {
            first_not_zero++;
            temp = temp >> 1;
        }
        for (i = 0; i < 10; i++) {
            if (testBit(i + 1, first_not_zero - 1))
                num1 ^= (i + 1);
            else
                num2 ^= (i + 1);
            if (testBit(num[i], first_not_zero - 1))
                num1 ^= num[i];
            else
                num2 ^= num[i];
        }
        System.out.println("num1:" + num1 + ",num2:" + num2);
    }

    public static boolean testBit(int value, int pos) //测试某位置是否为1
    {
        return ((value >> pos) & 1) != 0;
    }

}
```