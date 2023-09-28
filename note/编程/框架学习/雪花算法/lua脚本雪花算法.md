---
layout: post
tags:
  - 分布式
  - ID
  - 编程
title: 雪花算法lua脚本
author: tong
---
## 动态分配-Redis

通过 redis 去获取 workId 和 datacenterId

![redis雪花算法.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/dataredis%E9%9B%AA%E8%8A%B1%E7%AE%97%E6%B3%95.png)

具体 Lua 脚本逻辑如下：

![image.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/data20230928113629.png)


1. 第一个服务节点在获取时，Redis 可能是没有 snowflake_work_id_key 这个 Hash 的，应该先判断 Hash 是否存在，不存在初始化 Hash，dataCenterId、workerId 初始化为 0
2. 如果 Hash 已存在，判断 dataCenterId、workerId 是否等于最大值 31，满足条件初始化 dataCenterId、workerId 设置为 0 返回
3. dataCenterId 和 workerId 的排列组合一共是 1024，在进行分配时，先分配 workerId
4. 判断 workerId 是否 != 31，条件成立对 workerId 自增，并返回；如果 workerId = 31，自增 dataCenterId 并将 workerId 设置为 0

dataCenterId、workerId 是一直向下推进的，总体形成一个环状。通过 **Lua 脚本的原子性**，保证 1024 节点下的雪花算法生成不重复。如果标识位等于 1024，则从头开始继续循环推进

![redis存储标识位.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/dataredis%E5%AD%98%E5%82%A8%E6%A0%87%E8%AF%86%E4%BD%8D.png)


---

[back](雪花算法重复主键问题分析.md)

[home](../../../../index.md)