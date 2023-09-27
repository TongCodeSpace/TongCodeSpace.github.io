---
layout: default
tags:
  - 分布式
  - ID
  - 编程
title: 雪花算法lua脚本
---
雪花算法
具体 Lua 脚本逻辑如下：

1. 第一个服务节点在获取时，Redis 可能是没有 snowflake_work_id_key 这个 Hash 的，应该先判断 Hash 是否存在，不存在初始化 Hash，dataCenterId、workerId 初始化为 0
2. 如果 Hash 已存在，判断 dataCenterId、workerId 是否等于最大值 31，满足条件初始化 dataCenterId、workerId 设置为 0 返回
3. dataCenterId 和 workerId 的排列组合一共是 1024，在进行分配时，先分配 workerId
4. 判断 workerId 是否 != 31，条件成立对 workerId 自增，并返回；如果 workerId = 31，自增 dataCenterId 并将 workerId 设置为 0