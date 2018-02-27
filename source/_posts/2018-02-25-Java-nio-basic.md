---
title: Java NIO 基础白话文解释
date: 2018-02-25 18:26:04
tags:
- Java
- 进击的Java新人
---

Author: Yellow

# 关于

本文是以一种简单易懂的方式阐述Java NIO中一些基本但是可能让初学者费解的概念，参考了《进击的Java新人》和互联网上的一些资料，内容不仅限于Java，可能会牵涉到一些计算机的基础知识，比如I/O模型、操作系统知识等。

# 前言

当前这个时代，随着上网成本越来越低廉、终端性能越来越强大、应用种类也越来越丰富，互联网已触及我们生活中的每个角落。如今，如果一个年轻人从没点过一次美团外卖，或者从没玩过网络游戏或看过网络直播，那么他可能会被认为是刚从山洞里走出来的原始人。

而这一切便利的互联网服务的背后，是由一系列复杂的技术做支撑的，如何保证在海量用户访问的情况下服务还能稳定并且快速，这成了很多企业要着手解决的问题，毕竟谁都不想自己的产品三天两头崩溃打开一个页面要十几秒吧，如果真要那样，估计boss会提着菜刀在办公室等你了（参考J东）。

然而不管一个公司的业务是游戏，还是外卖，亦或是社交，其本质都是网络应用（包括客户端与服务器的交互、服务器与服务器的交互），所以一个公司如何想handle大量的数据、大量的用户与大量的交互，高性能网络编程必须得玩的溜。

对于大部分~~SSH~~Java程序员来说，拿SSH开发Web应用应该是家常便饭了，但如果把SSH拿掉，让他做一个网络应用（比如一个IM），他可能就懵逼了。个人认为，“框架”的确是一个好东西，特别是对于做项目、提高生产率来说；但是另一方面，如果一个程序员只是想着怎么去使用这些框架，而忽视了很多更核心更基本的东西，那对于他的个人发展来说，无异于故步自封。（见[Java程序员的堕落](http://www.vaikan.com/the-degradation-of-java-developers/ "Java程序员的堕落")）

另一方面，对于SSH中的核心Spring来说，其性能在不同语言的各种Web框架中也是几乎垫底的（见这个[Benchmark](https://www.techempower.com/benchmarks/#section=data-r15&hw=ph&test=json "Benchmark")），但是同样是基于Java的Netty却名列前茅，所以Java本身性能并不弱，弱的是Spring使用的堵塞式的I/O模型（但是最新的Spring 5似乎推出了能支持非堵塞I/O的Spring WebFlux，这个有带考察）。

所以，为了了解或者入门高性能网络编程，I/O模型的学习一定是重中之重，下面就会来谈一谈这方面的东西。（个人觉得，I/O这个东西一直都是拖累系统性能的吊车尾：内存拖累CPU、硬盘拖累内存、......，于是牛人们为了让系统不至于被拖累得那么惨，捣鼓出了各种玩意，比如Cache、LazyLoad等... 扯远了）

# JavaNIO之一：传统I/O与NIO的关系

## 诞生历史

从Java诞生开始，伴随着JDK1.0，Java程序员们就有了相应的I/O类可以使用，比如大家熟悉的老朋友`InputStream`、`OutputStream`等等，基本都集中于`java.io`这个package下面。那时候用户量小，数据量小，所以大家都用得很开心。那时候的代码大概长成这个样子：
```java
InputStream is = new FileInputStream(“a.txt”);
byte b = is.read();
```
到了大概两千年初，Java在其JDK1.4版本中包含了一个新的朋友：`java.nio`，提供了对非阻塞I/O的抽象（对下需要依赖OS的实现）。有人认为**nio** = Non-Blocking I/O，但是官方宣传**nio** = New I/O，然而是哪个并不重要，只要跟老朋友阻塞式Blocking I/O（BIO）区分开就行。

之后到了JDK1.7的时候，Java又引入了一个更新的朋友：AIO（具体的Class也是在nio package下），负责对异步I/O（Asynchronous I/O）进行抽象，然而很多OS的AIO的实现并不给力，效率不高，所以实际用处并不大。**我们下面的内容主要是基于Non-Blocking I/O**。

我们看到Java为我们带来了很多新朋友（Class、package），但是有一点要注意的是Java并没有创造任何I/O模型，I/O模型必须OS层面提供支持，并以syscall的形式暴露出来，Java做的工作实际上是对这些syscall的再一次封装。

## 有什么用处？
现代高性能服务器（Game Server、Web Server、IM Server…）、高性能通讯框架的基础，比如Netty、Nginx、Node.js、Redis、Dubbo等，虽然它们的编程语言并不相同，但是都是依赖于OS底层提供的能力实现的，最终的编程模型并不相差太多。业界经常说的C100K之类的问题，靠BIO基本是解决不了的，通常都是需要 Non-Blocking I/O（具体原因会在下面章节阐述）。所以理解这些新的I/O模型，是学习与从事高性能网络开发工作的前提。

# JavaNIO之二：三大组件（Buffer、Channel、Selector）

JavaNIO基本上是围绕这3个东西进行编程：
1. **Buffer**：缓冲区，也就是内存中一块用来暂存数据的固定大小的区域。可以想象成一个用于装水的玻璃缸；
2. **Channel**：通道，数据可以通过Channel写入Buffer或从Buffer读取到别的地方，主要分为文件Channel和网络Channel两大类。可以想象下我可以用一根水管一端接水龙头，另一端伸到玻璃缸里，那么水就能源源不断地流入玻璃缸了，这根水管是一个往Buffer写数据的Channel；我还可以用另一根水管把水从玻璃缸中抽出到其它地方，这根水管是一个从Buffer读数据的Channel。
3. **Selector**：I/O多路复用的选择器，与Channel配合使用，是实现Non-Blocking I/O、高并发的关键，只能用于网络Channel，所以暂时先放一边，等到网络Channel时会一起讲。

所以对于前两者，它们的关系大概如下图，但是注意写入或读取的Channel都可以有多个，不仅限于一个。
![](https://raw.githubusercontent.com/ouriris/ouriris.github.io/hexo/source/uploads/2018-02-25/relationship_buffer_channel.png)

# JavaNIO之三：Buffer
## 总体概述
学过计算机的同学应该不难理解什么是（一般意义上的）Buffer，其实JavaNIO中的Buffer类也是类似：一块尺寸在初始化之后就固定的内存区域，用来暂时存放数据。下面给出Buffer家族的类层次图：
![](https://raw.githubusercontent.com/ouriris/ouriris.github.io/hexo/source/uploads/2018-02-25/buffer_classes_hierarchy.png)

可以看到其实`Buffer`是一个抽象类，在其之下是对应所有包装类型的抽象Buffer类，其实就是属性中增加了一个对应的原始数据类型的数组（比如`DoubleBuffer`里面比`Buffer`多了一个`double[]`）。再往下就是具体的实现类，基本上每种数据类型都至少对应两种：HeapXXXBuffer（在Java Heap中分配）和DirectXXXBuffer（在本地内存中分配），比如`ByteBuffer`下面的`HeapByteBuffer`和`DirectByteBuffer`。

## Buffer的重要属性和操作
首先要说明的是，NIO中一些类的设计思路跟老的BIO类有些不一样，BIO虽然也可以自己new一个`byte[]`作为InputStream读取数据时的目的地，或OutputStream写数据时的目的地，但往往我们只需要把这个数组传进去read()或write()就行了，不需要知道读到或写到数组的什么地方，之后的工作Stream会帮我们做了。

然而NIO的使用方式相比起来却有点“**原始**”：管理Buffer读到哪、写到哪的工作不由Channel负责，而是回到Buffer自己身上，由程序员自己控制。最重要的属性（下标）有3个：
1. **position**：指向当前可被操作的位置（操作为读or写）
2. **limit**：指向可被操作的边界（如果是写操作，则表示空闲位置的边界；如果是读操作，则表示有效数据的边界）
3. **capacity**：Buffer的Size，Buffer被初始化之后就是一个固定值

它们的大小关系是：**position <= limit <= capacity**。只看文字有点抽象难理解，下面会结合图示说明。为了方便理解，你可以认为Buffer具有两种模式：读数据时处于**读模式**，写数据是处于**写模式**。然而实际上Buffer并没有这两种模式的设置，只是你可以逻辑上这样认为。

### 1.初始化
一个刚刚初始化好的空的Buffer（可以理解成做好准备可以写数据了！）长这样：
![](https://raw.githubusercontent.com/ouriris/ouriris.github.io/hexo/source/uploads/2018-02-25/buffer_op_1.png)

**mark**是什么可以先不管。**capacity**就是初始化时设定的Buffer的Size，这里是10。**position**指向的是当前可以被操作的位置，因为我们现在是**写模式**，所以它指向的是当前可以写的一个空位。**limit**则指向可写的边界，可以理解成写到哪就不允许继续往下写了，因为我们这是刚刚创建的空Buffer，默认下可以写到全部满位置，所以这时候**limit** = **capacity** = 10。

### 2.顺序写入几个字符
假设我们现在依次往Buffer里面写入5个字符 H、 e、 l、 l 、o：
```java
buffer.put((byte)'H').put((byte)'e').put((byte)'l').put((byte)'l').put((byte)'o');
```
则每写入一个字符，**position**都会向右移一个位置，写了5个字符后，它就指向5这个位置。现在它看起来像这样：
![](https://raw.githubusercontent.com/ouriris/ouriris.github.io/hexo/source/uploads/2018-02-25/buffer_op_2.png)

### 3.随机写入
假设我们要直接修改Buffer中的某个位置，可以这样做：
```java
buffer.put(0, (byte)'M');
```
这样做会把原先位置0的字符‘H’修改为‘M’，并且随机写入不会改变**position**的位置（它还是指向5）。如果这时候我们再顺序写入一个字符‘w’，这个字符是会写在位置5的，现在它看起来像这样：
```java
buffer.put((byte)'w');
```
![](https://raw.githubusercontent.com/ouriris/ouriris.github.io/hexo/source/uploads/2018-02-25/buffer_op_3.png)

### 4.翻转！变成读模式
**PS：flip操作经常会让刚接触NIO的人费解！所以下面内容请仔细阅读！**

假设现在我们要读出Buffer里的数据，直接调用`get()`是不行的，因为读操作也要依赖**position**、**limit**，而这时候**position**是指向一个空位置，所以读不出先前写进Buffer里的数据。于是这时候我们需要一个**翻转**操作：
```java
buffer.flip();
```
flip()执行后产生如下效果：
1. **当前limit**变成**先前position**的值；
2. **当前position**变成0。
![](https://raw.githubusercontent.com/ouriris/ouriris.github.io/hexo/source/uploads/2018-02-25/buffer_op_4.png)
可以简单理解成Buffer变成了**读模式**，**position**指向了当前可以被读的字符，**limit**则指向字符串的尾巴（也就是最多允许读到哪里为止）。之后就可以调用`get()`来读取数据了，每次读**position**都会往右移动一个位置，直到**limit**为止。

### 总结
上面罗列了Buffer的3个重要属性和flip操作，其中理解flip操作非常关键。可以想想，如果我们连续调用两次`flip()`，会产生什么效果？答案就是这个Buffer变成既不能读也不能写了，因为**position**、**limit**都指向0了。

## Heap Buffer与Direct Buffer
一般我们new出来的Buffer都是在Java Heap中分配的空间，在里面的空间可以被GC回收，但有时候我们为了性能考虑，比如不想GC干扰或增加GC压力，可以考虑在Java进程中的Native内存中分配Buffer，这时候的Buffer称为Direct Buffer，可以通过`allocateDirect(int capacity)`获得。但是分配在Native内存中的Buffer不能被GC回收，所以如果要使用，请参考这里：[Deallocating Direct Buffer Native Memory in Java for JOGL](https://stackoverflow.com/questions/3496508/deallocating-direct-buffer-native-memory-in-java-for-jogl/26777380#26777380 "Deallocating Direct Buffer Native Memory in Java for JOGL")

Heap Buffer与Direct Buffer在Java进程中的布局图如下所示：
![](https://raw.githubusercontent.com/ouriris/ouriris.github.io/hexo/source/uploads/2018-02-25/java_memory_layout.png)

# JavaNIO之四：三大组件之Channel
Channel可以想象成一个管子，一头连接着I/O数据来源/目的地（文件、网络等），一头连接着Buffer，数据可以在这两者之间传输。

Channel按照面向的设备的不同，可以分为两种：
1. FileChannel（面向磁盘文件）
2. SocketChannel、ServerSocketChannel（面向网络数据流）

SocketChannel、ServerSocketChannel将在之后跟着Selector一起讲，**这里先讲FileChannel**。另外从JDK1.7开始多了一些以**Asynchronous**打头从Channel类，这是属于AIO部分的东西，这里也不作阐述。

## 如何使用
可以通过FileChannel类的`open()`函数方便地创建一个与某个文件关联的FileChannel对象。常用的API大概有下面这些：
```java
public abstract int read (ByteBuffer dst, long position)
public abstract int write (ByteBuffer src, long position)
public abstract long size()
public abstract long position()
public abstract void position (long newPosition)
public abstract void truncate (long size)
public abstract void force (boolean metaData)
```





