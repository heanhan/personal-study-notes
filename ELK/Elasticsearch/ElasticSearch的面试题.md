## 一、Elasticsearch的基础

### 什么是Elasticsearch?

Elasticsearch 是基于Lucene的Restful的分布式实时全文搜索引擎，每个字段都可被索引并可被搜索，可以快速存储、搜索、分析海量的数据。

全文检索是指对每个词简历一个索引，致命改词在文章中出现的次数和位置。当查询时根据事先建立的索引进行查询，并将查找的结果反馈给用户的检索方式，这个过程类似与通过字典中的检索字查找词的过程。

### Elasticsearch相关名词的解释

| Elasticsearch | Index        | type | document | fileds     |
| :-----------: | ------------ | ---- | -------- | ---------- |
|   **mysql**   | **database** | 无   | **row**  | **cloumn** |

**映射（mapping）：** 类比mysql中schema和数据库的设计(比如字段的类、长度...)，es中通过mapping定义那些字段是否可以分词操作，哪些字段可以被查询。

**分片（Shards）:** 类比mysql的水平分表。作用是容量的扩容，提高吞吐量。

**副本（replicas）:** 分片数据的副本，保障数据安全。

**分配（allocation）:** 将分片给某个捡点的过程（包括主分片和副本），有master节点完成。