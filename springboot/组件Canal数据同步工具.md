### Cannal入门学习

**背景**

一个系统的重要性就是数据，数据是保存在数据库里，但是很多时候不单只要保存在数据库中，好要同步保存到Elasticsearch 、HBase、Rdis等等。

阿里地开源框架Canal就是很方便的同步数据的增量数据到其他的存储应用，所以这里总结以下。

#### 一、什么是Canal 

**官网介绍**

```
canal,翻译为水道/管道/沟渠。主要的用途是基于Mysql 数据库增量日志解析，提供增量数据订阅和消费
```

**canal组件的实现核心的思想**：Canal 的工作原理就是把自己伪装成Mysql slave ,模拟Mysql salve 的交互协议向Mysql Master 发送dump 协议，mysql  master 收到canal发送来的dump请求，开始推送binary log 给canal,然后canal 解析binary log 日志，在发送到储存的目的地，

#### 二、canal能做什么

*  数据镜像
* 数据库实时备份
* 索引构建和实时维护
* 业务cache(缓存)刷新
* 带业务逻辑的增量数据处理

#### 三、如何搭建canal

##### 3.1 首先有一个mysql的服务器

当前msyql的版本支持源端mysql版本：5.1.x , 5.5.x , 5.6.x , 5.7.x , 8.0.x

首先需要在mysql中创建一个用户，并授权

```mysql
-- 使用命令登录  mysql -uroot -p
-- 创建用户   用户名 canal 密码  Canal@123456
create user 'canal'@'%' identified by 'Canal@123456';
-- 授权 *.* 表示所有的库
grant SELECT, REPLICATION SLAVE, REPLICATION CLIENT on *.* to 'canal'@'%' identified by 'Canal@123456';


```

下一步在mysql 配置文件中 my.cnf 设置如下信息

```mysql
[mysqld]
# 打开binlog
log-bin=mysql-bin
# 选择ROW(行)模式
binlog-format=ROW
# 配置MySQL replaction需要定义，不要和canal的slaveId重复
server_id=1

```

改了配置文件之后，重启MySQL，使用命令查看是否打开binlog模式：log_bin的value 值on 才表示开启

```mysql
show variables like 'log_bin';
```

查看binlog 日志文件列表

```mysql
show binary logs;
```

查看当前正在写入的binlog文件

```mysql
show master starus;
```

以上mysql这边的设置就已经完成。

##### 3.2 安装canal

官网的安装地址：

```
https://github.com/alibaba/canal/releases
 例如：我下载的1.1.5（2022-01-05）
 选择 canal.deployer-1.1.5.tar.gz 
 
 解压文件夹的后的内容：
 bin   各种二进制文件
 conf   配置文件
 lib    依赖的jar包
 logs   日志
 
```

接着打开配置文件conf/example/instance.properties，配置信息如下：

```properties
## mysql serverId , v1.0.26+ will autoGen
## v1.0.26版本后会自动生成slaveId，所以可以不用配置
# canal.instance.mysql.slaveId=0

# 数据库地址
canal.instance.master.address=127.0.0.1:3306
# binlog日志名称
canal.instance.master.journal.name=mysql-bin.000001
# mysql主库链接时起始的binlog偏移量
canal.instance.master.position=154
# mysql主库链接时起始的binlog的时间戳
canal.instance.master.timestamp=
canal.instance.master.gtid=

# username/password
# 在MySQL服务器授权的账号密码
canal.instance.dbUsername=canal
canal.instance.dbPassword=Canal@123456
# 字符集
canal.instance.connectionCharset = UTF-8
# enable druid Decrypt database password
canal.instance.enableDruid=false

# table regex .*\\..*表示监听所有表 也可以写具体的表名，用，隔开
canal.instance.filter.regex=.*\\..*
# mysql 数据解析表的黑名单，多个表用，隔开
canal.instance.filter.black.regex=

```

启动canal  在bin路径下  start.sh



#### 四、java客户端的操作

首先引入maven的坐标

```xml
<dependency>
    <groupId>com.alibaba.otter</groupId>
    <artifactId>canal.client</artifactId>
    <version>1.1.5</version>
</dependency>

```

##### java的关键代码

在CannalClient类使用Spring Bean的生命周期函数afterPropertiesSet()：

```java
@Component
public class CannalClient implements InitializingBean {

    private final static int BATCH_SIZE = 1000;

    @Override
    public void afterPropertiesSet() throws Exception {
        // 创建链接
        CanalConnector connector = CanalConnectors.newSingleConnector(new InetSocketAddress("127.0.0.1", 11111), "example", "", "");
        try {
            //打开连接
            connector.connect();
            //订阅数据库表,全部表
            connector.subscribe(".*\\..*");
            //回滚到未进行ack的地方，下次fetch的时候，可以从最后一个没有ack的地方开始拿
            connector.rollback();
            while (true) {
                // 获取指定数量的数据
                Message message = connector.getWithoutAck(BATCH_SIZE);
                //获取批量ID
                long batchId = message.getId();
                //获取批量的数量
                int size = message.getEntries().size();
                //如果没有数据
                if (batchId == -1 || size == 0) {
                    try {
                        //线程休眠2秒
                        Thread.sleep(2000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                } else {
                    //如果有数据,处理数据
                    printEntry(message.getEntries());
                }
                //进行 batch id 的确认。确认之后，小于等于此 batchId 的 Message 都会被确认。
                connector.ack(batchId);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            connector.disconnect();
        }
    }

    /**
     * 打印canal server解析binlog获得的实体类信息
     */
    private static void printEntry(List<Entry> entrys) {
        for (Entry entry : entrys) {
            if (entry.getEntryType() == EntryType.TRANSACTIONBEGIN || entry.getEntryType() == EntryType.TRANSACTIONEND) {
                //开启/关闭事务的实体类型，跳过
                continue;
            }
            //RowChange对象，包含了一行数据变化的所有特征
            //比如isDdl 是否是ddl变更操作 sql 具体的ddl sql beforeColumns afterColumns 变更前后的数据字段等等
            RowChange rowChage;
            try {
                rowChage = RowChange.parseFrom(entry.getStoreValue());
            } catch (Exception e) {
                throw new RuntimeException("ERROR ## parser of eromanga-event has an error , data:" + entry.toString(), e);
            }
            //获取操作类型：insert/update/delete类型
            EventType eventType = rowChage.getEventType();
            //打印Header信息
            System.out.println(String.format("================》; binlog[%s:%s] , name[%s,%s] , eventType : %s",
                    entry.getHeader().getLogfileName(), entry.getHeader().getLogfileOffset(),
                    entry.getHeader().getSchemaName(), entry.getHeader().getTableName(),
                    eventType));
            //判断是否是DDL语句
            if (rowChage.getIsDdl()) {
                System.out.println("================》;isDdl: true,sql:" + rowChage.getSql());
            }
            //获取RowChange对象里的每一行数据，打印出来
            for (RowData rowData : rowChage.getRowDatasList()) {
                //如果是删除语句
                if (eventType == EventType.DELETE) {
                    printColumn(rowData.getBeforeColumnsList());
                    //如果是新增语句
                } else if (eventType == EventType.INSERT) {
                    printColumn(rowData.getAfterColumnsList());
                    //如果是更新的语句
                } else {
                    //变更前的数据
                    System.out.println("------->; before");
                    printColumn(rowData.getBeforeColumnsList());
                    //变更后的数据
                    System.out.println("------->; after");
                    printColumn(rowData.getAfterColumnsList());
                }
            }
        }
    }

    private static void printColumn(List<Column> columns) {
        for (Column column : columns) {
            System.out.println(column.getName() + " : " + column.getValue() + "    update=" + column.getUpdated());
        }
    }
}

```

以上就完成了Java客户端的代码。这里不做具体的处理，仅仅是打印，先有个直观的感受。

最后我们开始测试，首先启动MySQL、Canal Server，还有刚刚写的Spring Boot项目。然后创建表：

```mysql
CREATE TABLE `tb_commodity_info` (
  `id` varchar(32) NOT NULL,
  `commodity_name` varchar(512) DEFAULT NULL COMMENT '商品名称',
  `commodity_price` varchar(36) DEFAULT '0' COMMENT '商品价格',
  `number` int(10) DEFAULT '0' COMMENT '商品数量',
  `description` varchar(2048) DEFAULT '' COMMENT '商品描述',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品信息表';

```

此时idea的控制台会出现create的时间类型的日志

如果新增一条数据到表中

```mysql
INSERT INTO tb_commodity_info VALUES('3e71a81fd80711eaaed600163e046cc3','叉烧包','3.99',3,'又大又香的叉烧包，老人小孩都喜欢');
```

控制台依然会打出添加数据的信息

 

### 总结

canal的好处在于对业务的代码没有侵入，因为是基于监听binlog日志进行同步数据，实时性也能做到准实时，其实是很多企业一种比较常见的同步方案。

实际项目我们是**配置MQ模式，配合RocketMQ或者Kafka，canal会把数据发送到MQ的topic中，然后通过消息队列的消费者进行处理**。

