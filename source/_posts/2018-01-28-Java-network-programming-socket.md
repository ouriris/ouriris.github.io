---
title: Java网络编程之Socket篇
date: 2018-01-28 18:26:04
tags:
- Java
- 进击的Java新人
---

Author: Jonas Yu

## 目录

 - Socket介绍
 - 基于Socket的Java网络编程
 - 分层网络协议

## Socket介绍
网络上两个程序通过一个双向的通讯链接实现数据交换，这个双向链路的一端称为一个Socket。Socket通常用来实现客户端和服务器端的通信,在Internet上的主机一般会运行多个服务软件，软件一旦需要跟互联网上其他计算机之间通信，就需要通过Socket来绑定到一个端口，这个端口则是计算机用来标识对应的Socket进行数据传递。一个Socket由一个IP地址和一个端口唯一确定。

## 基于Socket的Java网络编程

基于TCP通讯过程：
通常情况下，服务端的Socket会监听某个端口是否有连接请求，当服务端接收到客户端的连接请求时，会向客户端发送接收的消息，这样一个连接就建立了。客户端与服务端都可以通过Socket进行数据通信。

Java的网络编程的接口大多数位于 java.net 和 java.nio 这两个package里，掌握这两个package是Java程序员必备的基础技能。Java.net中提供了两个类Socket和ServerSocket，分别用来表示双向连接的客户端和服务端。

下面是使用Socket跟ServerSocket来实现简单的C/S程序代码：


```java
public class Server {
    public static void main(String args[]) throws IOException, InterruptedException {
        ServerSocket ss = new ServerSocket(8000);
        System.out.println("Accept start *** ");
        Socket conn = ss.accept();
        System.out.println("Accept in progress ");
        BufferedReader br = new BufferedReader(new InputStreamReader(conn.getInputStream()));
        BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(conn.getOutputStream()));
        String s = br.readLine();
        while (s != null) {
            System.out.println("Receive: " + s);
            Thread.sleep(5000);
            bw.write(s.toUpperCase() + "\n");
            bw.flush();
            System.out.println("Send: " + s.toUpperCase());
            s = br.readLine();
        }

        br.close();
        bw.close();
        conn.close();
    }
}
```
```java
public class Client {
    public static void main(String args[]) throws IOException, InterruptedException {
        Socket conn = new Socket("127.0.0.1", 8000);
        BufferedReader br = new BufferedReader(new InputStreamReader(conn.getInputStream()));
        BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(conn.getOutputStream()));
        System.out.println("conn server success !");
        Thread.sleep(2000);
        bw.write("hello\n");
        bw.flush();
        System.out.println("hello has been send.");
        String s = br.readLine();
        System.out.println(s);
        Thread.sleep(5000);
        bw.write("world\n");
        bw.flush();
        System.out.println("world has been send.");
        s = br.readLine();
        System.out.println(s);
        br.close();
        bw.close();
        conn.close();
    }
}
```


从上面代码可以看出，Socket通信步骤如下:

 1. 服务端通过指定端口号实例化ServerSockt对象，调用accept函数阻塞等待连接请求
 2. 客户端通过IP跟端口实例化Socket对象
 3. 客户端跟服务端分别得到一个Socket对象，根据Socket对象，得到对应的输出流跟输入流，通过这两个IO可以进行两个端之间的通信

## 分层网络协议

OSI模型
OSI模型是国际标准化组织ISO创立的。这是一个理论模型，并无实际产品完全符合OSI模型。制订OSI模型只是为了分析网络通讯方便而引进的一套理论。也为以后制订实用协议或产品打下基础。OSI模型共分七层：从上至下依次是

 - 应用层 指网络操作系统和具体的应用程序，对应WWW服务器、FTP服务器等应用软件
 - 表示层：数据语法的转换、数据的传送等
 - 会话层：建立起两端之间的会话关系，并负责数据的传送
 - 传输层：负责错误的检查与修复，以确保传送的质量，是TCP工作的地方。
 - 网络层：提供了编址方案,IP协议工作的地方(数据包）
 - 数据链路层：将由物理层传来的未经处理的位数据包装成数据帧
 - 物理层：对应网线、网卡、接口等物理设备(位)

TCP/IP四层模型
在实际开发中，我们不会使用这种模型，更常用的是TCP/IP协议族的四层模型。
这四层模型分别对应的是：
 

 - 应用层 最上面一层是应用层，负责处理特定的应用程序消息。例如Telnet，Ftp, Http等。

 - 传输层 主要为两台主机上的应用程序提供端到端的通信。在 TCP/IP 协议族中，有两种传输协议：TCP（传输控制协议）和UDP（用户数据报协议） 。TCP为两台主机提供高可靠性的数据通信。它所做的工作包括把应用程序交给它的数据分成合适的小块交给下面的网络层，确认接收到的分组，设置发送最后确认分组的超时时钟等。由于运输层提供了高可靠性的端到端的通信，因此应用层可以忽略所有这些细节。另一种传输协议，UDP则为应用层提供一种非常简单的服务。它只是把称作数据报的分组从一台主机发送到另一台主机，但并不保证该数据报能到达另一端。任何必需的可靠性必须由应用层来提供。这两种运输层协议分别在不同的应用程序中有不同的用途。

 - 网络层 处理分组在网络中的活动，例如分组的选路。在TCP/IP协议族中，网络层协议包括 IP 协议（网际协议） ，ICMP 协议（Internet互联网控制报文协议） ，以及 IGMP 协议（Internet组管理协议） 。

 - 链路层 也称作数据链路层，通常包括操作系统中的设备驱动程序和计算机中对应的网络接口卡。它们一起处理与电缆（或其他任何传输媒介）的物理接口细节。这一层大约相当于OSI模型中的物理层和链路层的总和。


OSI分层模型和TCP/IP分层模型的对应关系：
![](https://raw.githubusercontent.com/ouriris/ouriris.github.io/hexo/source/uploads/week8/osi_tcp_ip.jpg)
