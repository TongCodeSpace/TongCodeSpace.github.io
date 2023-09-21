---
title: ElasticSearch
layout: default
---
## ElasticSearch 是什么
**Elasticsearch**是一个基于 Lucene 库的搜索引擎。它提供了一个分布式、支持多租户的全文搜索引擎，具有Http Web接口和无模式 Json 文档。ElasticSearch 是用 Java 开发的，并在Apache许可证下作为开源软件发布。
## 建立搜索引擎
建立搜索引擎一般有三个过程：**爬取数据**，**进行分词**，**建立倒排索引**

> 爬取数据一般通过爬虫获取，所以不在 ElasticSeach 考虑的范围内
### 进行分词
分词：就是把内容拆分为很多个词语，ES是把text格式的字段按照分词器进行分词并保存为索引的

在 ElasticSearch 中是通过 **Text Analysis**（文本分析）将文本切分成词项的。文本分析的功能主要是通过 **Analyzer**（分析器）来实现的。

**Analyzer**由三部分组成：字符过滤器（**Character filters**）、分词器（**Tokenizer**）和词元过滤器（**Token filter**）。每一个Analyzer有且只能有一个tokenizer。

- Character filters：针对原始文本处理，例如去除html，阿拉伯数字转换等
- Tokenizer：按照规则将文本切分为单词
- Token Filter：将切分的单词进行加工，如单词小写、删除stopword、增加同义词等


1. 切分关键词，把关键的、核心的单词切出来。 （根据分词器的不同，单词切分的方式也不同，常用的中文分词器是词库分词器 LK）
2. 去除停用词。（去除一些无谓的词，类似于：的，了，啊等）
3. 对于英文单词，把所有字母转为小写（搜索时不区分大小写）

### 建立倒排索引 

#### 倒排索引是什么
也常被称为**反向索引**、**置入文件**或**反向文件**，是一种索引方法，被用来存储在全文索引下某个单词在一个文档或者一组文档中的存储位置的映射。它是文档检索系统中最常用的数据结构。

## ElasticSearch
在 ElasticSearch 中有三个重要的专有名词：**索引**、**类型**、**文档**

**索引**：在 ElasticSearch 中的索引和 MySQL 中的索引有很大区别，是用来存放数据的地方，可以理解为一个数据库

**类型**：类型是用来定义数据结构的，可以认为是关系型数据库中的一张表

**文档**：文档对应的就是数据库中的一条数据

![es对照.drawio.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/dataes%E5%AF%B9%E7%85%A7.drawio.png)

示例

> text 和 keyword 区别：首先 text 和 keyword 都是用来表示字符串的，但是 text 在存入 ES 时会进行分词，然后感觉分词内容进行反向索引，而 keyword 会直接根据内容建立反向索引

使用 ElasticSearch

使用
1. 怎么使用
	1. 使用了 restful 接口，直接调用接口即可进行操作
2. master 和 slaver