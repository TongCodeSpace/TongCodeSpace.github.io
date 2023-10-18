---
layout: post
author: tong
title: 初识 Netty
---
## Netty 是什么

"Netty is _an **asynchronous** **event-driven** network application framework_  
For rapid development of maintainable high performance protocol servers & clients."

netty 是一个异步的事件驱动的网络应用框架
用于快速开发可维护的高性能协议服务器和客户端

> 事件驱动 VS "请求/响应"模式
> 
> **"请求/响应"模式**：服务必须等待答复才能进入下一个任务
> 
> **事件驱动**：流程由事件运行，它旨在响应事件或针对事件执行一些操作。也被称为"异步"通信，着发件人和收件人不必互相等待才能完成下一个任务。系统并不依赖于那条消息

本质上是提供高性能网络IO通信的功能


- [ ] IO 模型
- [ ] Reactor 模型


> Reference：
> [netty 官网](https://netty.io/index.html)
> 

---
[back](../../编程相关文章汇总.md)

[home](../../../../index.md)