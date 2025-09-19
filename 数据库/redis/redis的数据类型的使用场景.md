### Redis的五种数据类型的使用场景

#### 1、String

字符串String 是redis最简单的数据结构，可以存储字符串、整数、或者浮点数。最常见的场景就是对象的缓存，例如缓存用户信息，key是"userInfo"+#{用户ID}，value是用户信息对象的JSON字符串。

##### 案例：

key：userInfo123

value：{"gender":1,"nickname":"java程序鱼","userId":123}

```ruby
127.0.0.1:6379> set name hzy # 设置
OK
127.0.0.1:6379> get name # 获取
"hzy"
127.0.0.1:6379> exists name  # 判断是否存在
(integer) 1
127.0.0.1:6379> del name # 删除
(integer) 1
127.0.0.1:6379> get key
(nil)
```

##### Redis的 String批量操作

可以批量对多个字符串进行读写，节省网络耗时的开销。

```ruby
127.0.0.1:6379> mset name1 xiaoming name2 xiaohong # 批量设置
OK
127.0.0.1:6379> mget name1 name2 # 批量获取
1) "xiaoming"
2) "xiaohong"
```

##### Redis String的计数操作

```ruby
127.0.0.1:6379> incr likenum # 自增1
(integer) 1
127.0.0.1:6379> get likenum
"1"
127.0.0.1:6379> decr likenum # 减1
(integer) 0
127.0.0.1:6379> get number
"0"
```

##### Redis的过期操作

```ruby
127.0.0.1:6379> expire name 60 # 设置超时时间
(integer) 1
127.0.0.1:6379> setex name 60 value # 等价于 setex + expire
OK
127.0.0.1:6379> ttl key # 查看数据还有多久过期
(integer) 11
```

字符串是由多个字节组成，每个字节又是由 8 个 bit 组成.

##### Redis String使用场景

- 缓存：像我们平时开发，经常会把一个对象转成json字符串，然后放到redis里缓存
- 计数器：像博客文章的阅读量、评论数、点赞数等等
- 分布式系统生成自增长ID

### 2、List

Redis的列相当于java语言的LinkedList。

LinkedList优点：插入性能很高，不管是从尾部还是中间插入。

LInkedList缺点：随机性能差，例如LinkedList.get(10), 这个操作，性能就很差，因为他需要遍历这个链表，从头开始遍历这个链表，直到找到index=10的这个元素。

##### Redis List数据类型如何实现队列。

右边进左边出。

```ruby
127.0.0.1:6379> rpush apps qq # 将元素插入到列表的尾部(最右边)
(integer) 1
127.0.0.1:6379> rpush apps wechat taobao # 将多个元素插入到列表的尾部(最右边)
(integer) 3
127.0.0.1:6379> lpop apps # 移除并返回列表的第一个元素(最左边)
"qq"
127.0.0.1:6379> lrange apps 0 1 # 返回列表中指定区间内的元素，0表示第一个，1表示第二个，-1表示最后一个
1) "wechat"
2) "taobao"
127.0.0.1:6379> lrange apps 0 -1 # -1表示倒数第一
1) "wechat"
2) "taobao"
```

注意： **当列表弹出了最后一个元素之后，该数据结构自动被删除，内存被回收**

##### Redis List 数据类型如何实现栈？

先进先出，右边进右边出

```ruby
127.0.0.1:6379> rpush apps qq wechat taobao
(integer) 3
127.0.0.1:6379> rpop apps # 移除列表的最后一个元素，返回值为移除的元素
"taobao"
```

##### Redis List 使用场景

- 异步队列
- 任务轮询（RPOPLPUSH）
- 文章列表（lrange key 0 9）

#### 3.Hash

Redis的Hash结构相当于Java语言的HashMap，内部实现结构上与JDK1.7的HashMap一致，底层通过数据+链表实现。

Redis Hash 常用命令

```ruby
127.0.0.1:6379> hmset userInfo name "hzy" age "24" sex "1"
OK
127.0.0.1:6379> hexists userInfo name # 相当于HashMap的containsKey()
(integer) 1
127.0.0.1:6379> hget userInfo name # 相当于HashMap的get()
"hzy"
127.0.0.1:6379> hget userInfo age
"24"
127.0.0.1:6379> hgetall userInfo # 数据量大时，谨慎使用！获取在哈希表中指定 key 的所有字段和值
1) "name"
2) "hzy"
3) "age"
4) "24"
5) "sex"
6) "1"
127.0.0.1:6379> hkeys userInfo # 数据量大时，谨慎使用！获取 key 列表
1) "name"
2) "age"
3) "sex"
127.0.0.1:6379> hvals userInfo # 获取 value 列表
1) "hzy"
2) "24"
3) "1"
127.0.0.1:6379> hset userInfo name "test" # 修改某个字段对应的值
127.0.0.1:6379> hget userInfo name
"test"
```

##### Redis Hash 使用场景

- 记录整个博客的访问人数（数据量大会考虑HyperLogLog，但是这个数据结构存在很小的误差，如果不能接受误差，可以考虑别的方案）
- 记录博客中某个博主的主页访问量、博主的姓名、联系方式、住址

#### 4.Set

Redis的set集合相当于Java的HashSet。Redis 中的 set 类型是一种无序集合，集合中的元素没有先后顺序。

补充：HashSet就是基于HashMap来实现的，HashSet，他其实就是说一个集合，里面的**元素是无序的**，他里面的**元素不能重复**的，HashMap的key是无顺序的，你插入进去的顺序，跟你迭代遍历的顺序是不一样的，而且HashMap的key是没有重复的，HashSet直接基于HashMap实现的。

##### Redis Set 常用命令

```ruby
127.0.0.1:6379> sadd apps wechat qq # 添加元素
(integer) 2
127.0.0.1:6379> sadd apps qq # 重复
(integer) 0
127.0.0.1:6379> smembers apps # 注意：查询顺序和插入的并不一致，因为 set 是无序的
1) "qq"
2) "wechat"
127.0.0.1:6379> scard apps # 获取长度
(integer) 2
127.0.0.1:6379> sismember apps qq # 谨慎使用！检查某个元素是否存在set中，只能接收单个元素
(integer) 1
127.0.0.1:6379> sadd apps2 wechat qq
(integer) 2
127.0.0.1:6379> sinterstore apps apps apps2 # 获取 apps 和 apps 的交集并存放在 apps3 中
(integer) 1
127.0.0.1:6379> smembers app3
1) "qq"
2) "wechat"
```

注意： **当集合中最后一个元素移除之后，数据结构自动删除，内存被回收**

##### Redis Set 使用场景

- 微博抽奖：如果数据量不是特别大的时候，可以使用spop（移除并返回集合中的一个随机元素）或srandmember（返回集合中一个或多个随机数）
- QQ标签：一个用户多个便签
- 共同关注（交集）
- 共同好友（交集）

#### 5、sorted set

sorted set 有序集合，sorted set 增加了一个权重参数 score，使得集合中的元素能够按 score 进行有序排列，还可以通过 score 的范围来获取元素的列表。使得它类似于Java的TreeSet和HashMap的结合体。

##### Sorted set 常用命令

```ruby
127.0.0.1:6379> zadd apps 3.0 qq # 添加元素到 sorted set 中,3.0是score的值
(integer) 1
127.0.0.1:6379> zadd apps 2.0 wechat 1.0 aliyun # 一次添加多个元素
(integer) 2
127.0.0.1:6379> zcard apps # 查看 sorted set 中的元素数量
(integer) 3
127.0.0.1:6379> zscore apps wechat # 查看某个 value 的权重
"2.0"
127.0.0.1:6379> zrange apps 0 -1 # 通过索引区间返回有序集合指定区间内的成员，0 -1 表示输出所有元素
1) "aliyun"
2) "wechat"
3) "qq"
127.0.0.1:6379> zrange apps 0 1 # 通过索引区间返回有序集合指定区间内的成员
1) "aliyun"
2) "wechat"
127.0.0.1:6379> zrevrange apps 0 1 # 相当于zrange的反转
1) "qq"
2) "wechat"
```

注意： **sorted set 中最后一个 value 被移除后，数据结构自动删除，内存被回收**

##### Sorted set 使用场景

- 排行榜
- 订单支付超时（下单时插入，member为订单号，score为订单超时时间戳，然后写个定时任务每隔一段时间执行zrange）