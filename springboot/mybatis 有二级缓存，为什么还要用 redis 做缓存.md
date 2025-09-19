Mybatis 有二级缓存为什么还要用 Redis 做缓存。

1、Mybatis 一级缓存作用域是 session ,session commit 之后缓存就失效了。

2、mybatis 二级缓存的作用域是 namespace 为单位的（也就是一个 Mapper.xml 的文件中。）不同的 namespace 下的操作互不影响。

3、所有对数据表的改变操作都会做出刷新缓存。但是一般不要使用二级缓存。例如在 UserMapper.xml 中有大多数针对 User 表操作，但是在另一个 namespace 中还有一个针对 User 表操作的，这会导致 user 在两个命民空间下的数据不一致。

4、如果在 UserMapper.xml 中做了缓存的操作在另个 XXXMapper.xml 中缓存仍然有效，如果针对 user 的单表查询，使用缓存的结果可能会不正确，读到了脏数据。

5、Redis 很好的解决了这个问题，而且比一二级缓存好处很多，Redis 可以搭建在其他的服务器上，缓存容量可以扩展，Redis 可以灵活的使用唉需要缓存色数据上，比如一些热点数据统计点赞。