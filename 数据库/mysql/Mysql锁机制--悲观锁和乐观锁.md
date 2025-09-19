# Mysql锁机制--悲观锁和乐观锁

#### 1. 悲观锁简介

　　悲观锁（Pessimistic Concurrency Control，缩写PCC），它指的是对数据被外界修改持保守态度，因此，在整个数据处理过程中， 将数据处于锁定状态。悲观锁的实现，往往依靠数据库提供的锁机制，也只有数据库层提供的锁机制才能真正保证数据访问的排他性，否则，即使在本系统中实现了加锁机制，也无法保证外部系统不会修改数据。

#### 2、悲观锁举例

商品表t_goods中有字段status标识商品的状态，0标识未售出，1标识已售出。当用户生产订单时，该商品的status必须为0则认为是有效订单。

```mysql
create table t_goods (id int(11) primary key auto_increment, name varchar(255), status int(1) default 0);
insert into t_goods (name) value ('手机');
select * from t_goods;

create table t_orders (id int(11) primary key auto_increment, goods_id int(11));
```

假如不添加锁，则顺序为：

```mysql
# 1.查询商品
select * from t_goods where id  = 1 and status  = 0;
# 2.生产订单
insert into t_orders (goods_id) value (1);
# 3.更新状态
update t_goods set status = 1 where id = 1;

select * from t_goods;
```

这种情况在单线程或只有一个用户使用时没有问题，但是在多线程或并发场景下是不安全的。若在更新前，有其他订单提前将状态更新为1，则商品被重复下单，造成数据不一致的情况。

使用悲观锁实现，则顺序为：

```mysql
#必须关闭Mysql数据库的自动提交属性
#因为Mysql默认使用autocommit模式，即执行更新操作后，Mysql会立刻提交结果

show variables like '%autocommit%';
#临时设置
set autocommit  = 0;

# 1.开启事务（begin; begin work; start transaction;）
begin;
# 2.查询商品
select * from t_goods where id  = 1 and status  = 0 for update;
# 3.生产订单
insert into t_orders (goods_id) value (1);
# 4.更新状态
update t_goods set status = 1 where id = 1;
# 5.提交事务（commit; commit work;都可以）
commit;

select * from t_goods;
```

使用select * from tbale_name for update实现了悲观锁，即对id=1的数据进行加锁，其他事务对该数据的操作会进行阻塞，当前事务提交后，才允许其他事务对其进行操作。

​		注意：当开始一个事务，执行select * from tbale_name for update，在另外事务中select ...for update时会阻塞，但是使用select * from tbale_name 则不会阻塞。

　　select ... for update详解：Mysql的InnoDB引擎默认使用Row-Level Lock（行级锁），因此在查询中要指定主键，否则，Mysql会执行Table-Level Lock（表级锁）。

```mysql
#分别在两个窗口执行
# 1. 指定主键，并且主键存在，row lock
select * from t_goods where id = 1 for update;
# 2. 指定主键，主键不存在，no lock
select * from t_goods where id = 999 for update;
# 3. 无主键，table lock
select * from t_goods where name = '电脑' for update;
# 4. 主键不明确，table lock
select * from t_goods where id > 10 for update;
select * from t_goods where id != 1 for update;
# 5. 指定索引，并且索引存在，row lock
select * from t_goods where status = 1 for update;
# 6. 指定索引，索引不存在，no lock
select * from t_goods where status = 999 for update;
```

#### 3. 乐观锁简介

　　乐观锁（Optimistic Locking，简写PL），相对悲观锁而言，乐观锁认为数据一般情况下不会造成冲突，所以在数据进行提交更新的时候，才会对数据的冲突与否进行检测，如果发现冲突了，则让返回错误信息，让用户处理。

#### 4. 乐观锁举例

　　使用数据版本（Version）记录机制实现。这是乐观锁最常用的一种实现方式。即为数据增加一个版本标识，即增加一个数字类型的 “version” 字段。当读取数据时，将version字段的值一同读出，数据每更新一次，对version值加一。当提交更新时，判断数据库表对应记录的当前version值与取出来的version值进行比对，如果数据库表当前version值与取出来的version值相等，则执行更新，否则不更新。

![image-20201209141904768](/Users/zhaojh0912/Library/Application Support/typora-user-images/image-20201209141904768.png)

