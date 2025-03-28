---
layout: post
author: tong
title: IO 模型
tags:
  - 编程
  - IO
---
## I/O 流是什么

I/O 流全拼 Input/Output stream，也就是我们常说的输入输出流。而数据传输过程类似于水流，因此称为 IO 流。输入就是从其他外部设备将数据加载到计算机内存中，输出就是从计算机内部输出到外部存储设备中（数据库、硬盘等）

## IO 模型

首先我们要了解，一个输入操作一般有两个步骤：
1. 等待数据准备好
2. 从内核态拷贝到用户态

在《Linux 内核第一卷》这部书中有介绍 5 种 IO 模型，包括阻塞型 IO，非阻塞型 IO，多路复用 IO，信号驱动 IO，异步 IO。

### 1. 阻塞型 IO 模型

![阻塞IO.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/data%E6%88%AA%E5%B1%8F2023-10-19%2016.19.41.png)

阻塞 IO 的进程从调用 recvfrom 到收到内核返回的成功只指示这一段时间都是阻塞的

### 2. 非阻塞 IO 模型

![非阻塞IO.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/data20231019163403.png)
非阻塞 IO 和阻塞 IO 最大的不同点是在于等待数据的过程，拷贝的过程并无不同

在等待数据时，非阻塞 IO 会不停地轮询内核，查看是否准备好，直到数据准备好，这也是非常消耗 CPU 的，但是性能很高，因为线程不会停滞于此，可以去做一些其他的事情 

> [!note]
> **阻塞 IO 还是非阻塞 IO 区别主要体现在数据准备阶段，同步 IO 和异步 IO 的区别主要体现在数据拷贝阶段**


### 3. 同步 IO VS 异步 IO

其实上面提到的两种 IO 模型都属于同步 IO 模型，**同步**和**异步**主要看请求发起方对消息结果的获取方式，是**主动获取**还是**被动通知**

### 4. 多路复用 IO 模型

![异步IO.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/data%E6%88%AA%E5%B1%8F2023-10-20%2017.29.49.png)

复用 IO 会阻塞于 select ，等待数据报准备好，然后调用 recvfrom 将数据拷贝到用户态，从图上来看和阻塞 IO 模型没什么区别，但是使用 select 可以等待多个数据报准备好

### 5. 信号驱动 IO 模型

![信号驱动 IO.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/data%E6%88%AA%E5%B1%8F2023-10-20%2017.36.40.png)

信号驱动 IO 是通过系统直接调用 sigaction 安装一个信号处理程序，并立刻返回，进程继续执行，知道数据报准备好，发送 SIGIO 信号到该线程，然后调用 revfrom 将数据拷贝到用户态（非阻塞同步 IO）

### 6. 异步 IO 模型

![异步IO.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/data%E6%88%AA%E5%B1%8F2023-10-20%2017.38.00.png)

进程在进行了系统调用后立即返回，并且由操作系统准备好数据并将数据拷贝到用户态，拷贝完成后通过一个信号通知应用进程。异步 IO 模型是一种很好的方案，能让用户进程在 2 个阶段中都不会被阻塞，但是很多操作系统底层仍然未支持

### 7. 总结

![IO 对比.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/data%E6%88%AA%E5%B1%8F2023-10-20%2017.38.07.png)

**注意**：

$$阻塞型 IO \bigcup 非阻塞型 IO = 所有 IO 类型$$

$$同步 IO \bigcup 异步 IO = 所有 IO 类型$$

1.  阻塞型 IO 和非阻塞型 IO 已经是数学意义上的全集了，所以 IO 模型并不是并列关系
2. IO 模型不是孤立的，是可以叠加的，例如：非阻 IO 也可以是多路复用 IO ，只要符合当前 IO 模型特征那么他就可以说是某种 IO 模型
3. 不要混淆非阻塞型 IO 和多路复用 IO，两者❎不存在必然关系，准确来说多路复用 IO 属于同步非阻塞 IO
## Java 中的多种 IO 模型

我们常说的 **BIO** ，**NIO**，**AIO** 实际上就是以上部分 IO 模型在 Java 中的具体实现

BIO：阻塞IO

NIO：同步非阻塞 IO

AIO：异步 IO

虽然理论上来说，AIO 是性能最好的，但是目前由于各种限制也无法发挥出其性能，之前 Netty 也尝试过使用 AIO，但是由于性能提升并不明显也还是放弃了。所以目前来说 AIO 的应用还不够广泛


---
[back](初识Netty.md)


[home](../../../../index.md)