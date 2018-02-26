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
3. **Selector**：I/O多路复用的选择器，是实现Non-Blocking I/O、高并发的关键，只能用于网络Channel，所以暂时先放一边，等到网络Channel时会一起讲。

所以对于前两者，它们的关系大概如下图，但是注意写入或读取的Channel都可以有多个，不仅限于一个。
![](https://raw.githubusercontent.com/ouriris/ouriris.github.io/hexo/source/uploads/2018-02-25/relationship_buffer_channel.png)











