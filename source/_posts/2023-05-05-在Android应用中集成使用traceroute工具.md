---
layout:     post
title:      在Android应用中集成使用traceroute工具
subtitle:   又是些莫名其妙的版本兼容问题。
date:       2023-05-05
author:     YSY
header-style: text
catalog: true
cover: https://imgconvert.csdnimg.cn/b04cf6079ba04e7eb960c293d45fe951.png
tags:
    - Android
    - Linux
---

### 背景知识

traceroute是一个常用于Linux系统的网络工具，它可显示数据包在IP网络中所经过路由的IP地址，理想状态下可探测本机和目标地址之间的所有路由节点。

其他操作系统中也有类似的替代品，实现都大同小异。一般用法如下：

```bash
终端输入：
~ traceroute -I baidu.com
输出：
traceroute to baidu.com (39.156.66.10), 30 hops max, 60 byte packets
 1  9.102.191.130 (9.102.191.130)  0.638 ms  0.797 ms *
 2  * 9.102.250.222 (9.102.250.222)  0.745 ms  0.943 ms
 3  * * *
 4  10.200.46.253 (10.200.46.253)  1.332 ms  1.333 ms  1.332 ms
 5  * * *
 6  39.156.0.85 (39.156.0.85)  4.384 ms  4.184 ms  3.936 ms
 7  111.13.188.38 (111.13.188.38)  8.991 ms  9.029 ms  9.065 ms
 8  39.156.27.1 (39.156.27.1)  4.281 ms  4.366 ms  4.377 ms
 9  39.156.67.1 (39.156.67.1)  3.550 ms  3.561 ms  3.568 ms
10  * * *
11  * * *
12  * * *
13  * * *
14  39.156.66.10 (39.156.66.10)  3.973 ms  3.957 ms  4.015 ms
```

上面例子一共有14行输出结果，我们可称之为14跳，说明数据包途径了14个节点就到达了目标机器。每一跳会发送3个数据包，所以有3个对应的时间。

具体的实现原理可以直接参考[Wikipedia](https://zh.wikipedia.org/wiki/Traceroute)，主要是通过不断改变TTL值来发包实现的：

> 程序是利用增加存活时间（TTL）值来实现其功能的。每当数据包经过一个路由器，其存活时间就会减1。当其存活时间是0时，主机便取消数据包，并发送一个ICMP TTL数据包给原数据包的发出者。
> 程序发出的首3个数据包TTL值是1，之后3个是2，如此类推，它便得到一连串数据包路径。注意IP不保证每个数据包走的路径都一样。

### 集成到Android应用

Linux实现版本的源码在此：[Traceroute for Linux](https://sourceforge.net/projects/traceroute/files/traceroute/)，可以看到居然2023年还有一次更新。既然是Linux上的程序，有没有办法在Android上运行呢？或者直接集成到App的模块中？

因为Android系统本身没有预装traceroute命令工具（就算是在Linux上，大多发行版也是需要自己额外安装的），所以不能直接通过执行命令的方式来调用。**通过NDK编译traceroute源码到App中才是比较靠谱的办法。**

总的来说还是比较简单的，集成上述的Linux版本源码并添加相应的mk文件，就可以编译成库了。其实已经有开源网友实现了，GitHub上也有不少例子，这个[traceroute-for-android](https://github.com/wangjing53406/traceroute-for-android)较为完美，其中的library模块可以直接参考使用，甚至可以替换其中的traceroute源码为2023年最新版，也是没有问题的。

### 一些问题

#### 为什么同一跳会出现不同的IP地址

![在这里插入图片描述](https://imgconvert.csdnimg.cn/b04cf6079ba04e7eb960c293d45fe951.png)

在如图这个例子中，第4跳出现了一个不同的IP，很多人会比较疑惑。这是因为**网络环境是不断变化的，发包过程中会选择更好的路由**，可以参考[这个链接](https://www.baeldung.com/linux/traceroute-three-stars)中的解释：

> Line 8 shows that some probes take different paths at the same step: the first and third probes go through 96.112.146.26, while the second probe goes through 96.112.146.22. This is because network conditions are constantly changing, which affects the routing tables. Here, the router 96.112.146.22 was a better choice for a brief period of time, so the previous one chose it in the second probe.

#### 为什么要用“-I”参数

实际使用中我们会发现，很多主流的域名都无法成功trace到最终目标，最后几跳往往以星号结束，表示节点机器没有回应。这是为什么呢？

因为traceroute工具默认是发送UDP包来探测的，在当今这个复杂的互联网环境下，很多服务器都会因为安全机制拦截过滤掉UDP包，发送方得不到任何返回信息。所以在文章开头，你可以注意到我使用了“-I”参数，而不是默认无参。

在[Traceroute for Linux源码文档](https://traceroute.sourceforge.net/)中可以得知，此工具有多种发包方式，除了默认的UDP外，还可以用TCP、ICMP发包，后两者分别对应“-T”和“-I”参数，效果会比UDP好很多。

那么为什么我不使用更不容易被过滤的TCP发包呢？因为在非ROOT权限下，执行“-T”参数会有如下报错：

> You do not have enough privileges to use this traceroute method.
> socket: Operation not permitted

加sudo执行才不会报错。因为traceroute在使用TCP模式发包时会创建原始套接字，参考其源码：

```cpp
static int tcp_init (const sockaddr_any *dest,
			    unsigned int port_seq, size_t *packet_len_p) {
  ...
	/*  Create raw socket for tcp   */
	raw_sk = socket (af, SOCK_RAW, IPPROTO_TCP);
	if (raw_sk < 0)
		error_or_perm ("socket");
  ...
}
```

参考[自从学会原始套接字之后，我感觉掌握了整个世界](https://juejin.cn/post/6927974019421601806)，原始套接字必须有ROOT权限才能使用：

> 因为网络级IP数据包没有”端口“的概念，所以可以读取网络设备传入的所有数据包，这意味着什么？意味着安全性，使用了原始套接字的应用程序可以读取所有进入系统的网络数据包，也就是我们可以捕获其他应用程序的数据包，所以为了防止这种情况的发生，Linux要求所有访问原始套接字的程序都必须以root身份运行。

我们把traceroute编译到Android的App中，运行环境就在应用层，默认是没有ROOT权限的，所以“-T”参数自然也就用不了。

#### 低版本Android系统连“-I”也用不了

经过一些兼容性测试（覆盖了6.0及以上的所有大版本），我发现在Android 9.0及以下的系统中即便是“-I”参数也会执行失败，错误信息包含“socket bind”之类的，也就是说不同Android版本的socket函数库可能实现不同，才导致了低版本连ICMP发包都不行。

解决办法有两种：

1. 判断Android系统版本，在9.0及以下使用默认无参的traceroute，降级到UDP发包；10.0及以后使用“-I”参数。
2. 通过ping命令工具来模拟traceroute，因为ping工具是Android系统默认就预装了的，可以直接在Java层通过调用命令的方式执行，其次ping本身也有参数项来设置TTL值，且默认就用ICMP发包。为此我也做了一个简单的实现，可参考：[TraceRouteByPing](https://github.com/ysy950803/traceroute-for-android/blob/master/library/src/main/java/com/wandroid/traceroute/TraceRouteByPing.java)
