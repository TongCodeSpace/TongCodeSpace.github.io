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

### 阻塞型 IO 模型
![阻塞IO.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/data%E6%88%AA%E5%B1%8F2023-10-19%2016.19.41.png)

阻塞 IO 的进程从调用 recvfrom 到收到内核返回的成功只指示这一段时间都是阻塞的

### 非阻塞 IO 模型

![非阻塞IO.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/data20231019163403.png)
非阻塞 IO 和阻塞 IO 最大的不同点是在于等待数据的过程，拷贝的过程并无不同

在等待数据时，非阻塞 IO 会不停地轮询内核，查看是否准备好，直到数据准备好，这也是非常消耗 CPU 的



### 多路复用 IO 模型
### 信号驱动 IO 模型
### 异步 IO 模型

**注意**：


$$阻塞型 IO \bigcup 非阻塞型 IO = 所有 IO 类型$$


1.  阻塞型 IO 和非阻塞型 IO 已经是数学意义上的全集了，以上 IO 模型并不是并列关系
2. IO 模型不是孤立的，是可以叠加的，例如：非阻 IO 也可以是多路复用 IO ，只要符合当前 IO 模型特征那么他就可以说是某种 IO 模型
3. 不要混淆非阻塞型 IO 和多路复用 IO，两者❎不存在必然关系
## 多种 IO 模型对比
### BIO

### NIO

### AIO