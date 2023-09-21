---
title: ElasticSearch
layout: default
---
搜索引擎的三个过程：爬取数据，进行分词，建立倒排索引

爬取数据不在考虑的范围内

进行分词

建立倒排索引 

倒排索引是什么

ElasticSearch
专有名词：索引、类型、文档

索引：在 ElasticSearch 中的索引和 MySQL 中的索引有很大区别，是用来存放数据的地方，可以理解为一个数据库

类型：类型是用来定义数据结构的，可以认为是关系型数据库中的一张表

文档：文档对应的就是数据库中的一条数据

![es对照.drawio.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/dataes%E5%AF%B9%E7%85%A7.drawio.png)

✅ Elasticsearch security features have been automatically configured!
✅ Authentication is enabled and cluster connections are encrypted.

ℹ️  Password for the elastic user (reset with `bin/elasticsearch-reset-password -u elastic`):
  E9RitrSnPqkaFT3ktMzx

ℹ️  HTTP CA certificate SHA-256 fingerprint:
  207d576852fd6520460b9fad102f154d100a0b5958086d4d2ff41cb9f4ae6464

ℹ️  Configure Kibana to use this cluster:
• Run Kibana and click the configuration link in the terminal when Kibana starts.
• Copy the following enrollment token and paste it into Kibana in your browser (valid for the next 30 minutes):
  eyJ2ZXIiOiI4LjEwLjEiLCJhZHIiOlsiMTAuMTAuMi4xMjA6OTIwMCJdLCJmZ3IiOiIyMDdkNTc2ODUyZmQ2NTIwNDYwYjlmYWQxMDJmMTU0ZDEwMGEwYjU5NTgwODZkNGQyZmY0MWNiOWY0YWU2NDY0Iiwia2V5IjoiOFdpa3RZb0JYRmVfYk9tdXhCdWY6SW5kS0R2M0xTUXlzakxKYm1oaXVxZyJ9

ℹ️  Configure other nodes to join this cluster:
• On this node:
  ⁃ Create an enrollment token with `bin/elasticsearch-create-enrollment-token -s node`.
  ⁃ Uncomment the transport.host setting at the end of config/elasticsearch.yml.
  ⁃ Restart Elasticsearch.
• On other nodes:
  ⁃ Start Elasticsearch with `bin/elasticsearch --enrollment-token <token>`, using the enrollment token that you generated.