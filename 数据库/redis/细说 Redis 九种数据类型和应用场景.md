# 细说 Redis 九种数据类型和应用场景

**redis是（REmote DIctionary Service）作为[NoSQL数据库，以key-value的字典方式来存储数据，其中的value主要支持五种数据类型**。，常见的有五种，String(字符串)、Hash(哈希)、List(列表)、Set(集合)、Zset(有序集合)。

随着Redis的版本更新，后面有支持了四种数据类型：BitMap(2.2版本新增)、HyperLogLog(2.8新增)、GEO(3.2版本新增)、Stream(5.0新增)。

Redis采用的Key-Value的方式来存储数据，每个键值对都会有一个dicEntry，里面有只想key,value的指针，还有指向下一个键值对的next指针。

每一种数据类型对象都有各自的使用场景，

### 1、String(字符串)

String是最基本的key-value，key是唯一的标识，value是具体的值，value其实不仅仅是字符串，也可以是证书或者浮点数，value最多可以容纳的数据长度是512MB.

底层实现：是int 加上 SDS（简单动态数组）
