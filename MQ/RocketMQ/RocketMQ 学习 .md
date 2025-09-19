### 一、RocketMQ 的介绍

```
消息队列 是一种先进先出的数据结构。

应用的场景：应用解偶、异步削峰

RabbitMQ：小规模场景，吞吐量低，性能高，功能全面，erlang语言不好控制；
kafka：吞吐量大，会丢失数据，日志分析，大数据采集；
RocketMQ：兼具了上面两种的优点
```

### 二、概念模型

**Producer**：消息生产者，负责生产消息，一般由业务系统负责产生消息。

**Consumer**：消息消费者，负责消费消息，一般是后台系统负责异步消费。

**Push Consumer**：Consumer的一种，需要向Consumer对象注册监听。

**Pull Consumer**：Consumer的一种，需要主动请求Broker拉取消息。

**Producer Group**：生产者集合，一般用于发送一类消息。

**Consumer Group**：消费者集合，一般用于接收一类消息进行消费。

**Broker**：MQ消息服务（中转角色，用于消息存储于生产消息转发）。



### 三、消息

分为：普通消息、顺序消息、延时消息、事务消息、批量消息、消息过滤。

##### 3.1 普通消息

```
1、同步发送消息
生产者发出一条消息，会在收到返回的 ACK 之后才会发下一条消息，此类消息可靠性最高，但是消息的发送的效率低。

2、异步发送消息
生产者发出消息后无需等待 MQ 返回 ack ，直接发送下一条消息，此类消息可靠性可以得到保证，消息发送效率尚可。

3、单向消息
生产者仅负责发送消息，不等待，不处理MQ 的 ack ,此类方式的 MQ 也不返回 ACK，消息发送效率高，但消息可靠性差。
```

##### 3.2 顺序消息

```
严格按照消息的发送顺序进行消费。默认情况下，生产者会把消息一负载均衡的的轮询方式发送到不同的 queue 分区队列，而消费者消费时会从多个 queue上拉取消息。这种情况下发送的消息和消费的消息时不能保证顺序的，如果将消息金发送到同一个 queue中，消费时也只从这个 queue中拉取消息，就严格保证的消息的有序性。

1、有序分类： 分区有序，全局有序
有多个Queue参与，其仅可保证在该Queue分区队列上的消息顺序，称为分区有序。
在定义Producer时我们可以指定消息队列选择器，而这个选择器是我们自己实现了MessageQueueSelector接口定义的。在定义选择器的选择算法时，一般需要使用选择key。这个选择key可以是消息key也可以是其它数据。但无论谁做选择key，都不能重复，都是唯一的。

一般性的选择算法是，让选择key（或其hash值）与该Topic所包含的Queue的数量取模，其结果即为选择出的Queue的QueueId。

2、全局有序
当发送和消费参与的Queue只有一个时所保证的有序是整个Topic中消息的顺序，称为全局有序

在创建Topic时指定Queue的数量。有三种指定方式：

a. 在代码中创建Producer时，可以指定其自动创建的Topic的Queue数量
b. 在RocketMQ可视化控制台中手动创建Topic时指定Queue数量
c. 使用mqadmin命令手动创建Topic时指定Queue数量

```



##### 3.3 延时消息

```
当消息写入到Broker后，在指定的时长后才可被消费处理的消息，称为延时消息

延时等级
延时时长不支持随意时长的延迟，是通过特定的延迟等级来指定的，默认变量有1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h，分别对应1~18等级。例如等级为3，对应于10s
如果需要自己定义延时等级，需要在broker加载的配置文件中配置messageDelayLevel

messageDelayLevel = 1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h 1d

延时消息实现原理
修改消息

修改消息的Topic为SCHEDULE_TOPIC_XXXX
根据延时等级，在consumequeue目录中SCHEDULE_TOPIC_XXXX主题下创建出相应的queueId目录与consumequeue文件
修改消息索引单元内容。索引单元中的Message Tag HashCode部分原本存放的是消息的Tag的Hash值。现修改为消息的投递时间。投递时间是指该消息被重新修改为原Topic后再次被写入到commitlog中的时间。投递时间 = 消息存储时间 + 延时等级时间。消息存储时间指的是消息被发送到Broker时的时间戳
将消息索引写入到SCHEDULE_TOPIC_XXXX主题下相应的consumequeue中

投递延时消息
Broker内部有⼀个延迟消息服务类ScheuleMessageService，其会消费SCHEDULE_TOPIC_XXXX中的消息，即按照每条消息的投递时间，将延时消息投递到⽬标Topic中。不过，在投递之前会从commitlog中将原来写入的消息再次读出，并将其原来的延时等级设置为0，即原消息变为了一条不延迟的普通消息。然后再次将消息投递到目标Topic中。
将消息重新写入commitlog
延迟消息服务类ScheuleMessageService将延迟消息再次发送给了commitlog，并再次形成新的消息索引条目，分发到相应Queue

```

3.4 事务消息

```
分布式事务：一次操作由若干分支操作组成，这些分支操作分属不同应用，分布在不同服务器上。分布式事务需要保证这些分支操作要么全部成功，要么全部失败，分布式事务于普通事务一样，就是为了保证操作结果的一致性

事务消息：RocketMQ提供了类似X/Open XA的分布式事务功能，通过事务消息能达到分布式事务的最终一致。XA是一种分布式事务解决方案，一种分布式事务处理模式

半事务消息：暂不能投递的消息，发送方已经成功地将消息发送到了Broker，但是Broker未收到最终确认指令，此时该消息被标记成“暂不能投递”状态，即不能被消费者看到。处于该种状态下的消息即半事务消息

本地事务状态：Producer 回调操作执行的结果为本地事务状态，其会发送给TC，而TC会再发送给TM。TM会根据TC发送来的本地事务状态来决定全局事务确认指令

消息回查：重新查询本地事务的执行状态
关于消息回查，有三个常见的属性设置。它们都在broker加载的配置文件中设置
transactionTimeout=20，指定TM在20秒内应将最终确认状态发送给TC，否则引发消息回查。默认为60秒
transactionCheckMax=5，指定最多回查5次，超过后将丢弃消息并记录错误日志。默认15次。
transactionCheckInterval=10，指定设置的多次消息回查的时间间隔为10秒。默认为60秒。

XA协议
XA（Unix Transaction）是一种分布式事务解决方案，一种分布式事务处理模式，是基于XA协议的。XA协议由Tuxedo（Transaction for Unix has been Extended for Distributed Operation，分布式操作扩展之后的Unix事务系统）首先提出的，并交给X/Open组织，作为资源管理器与事务管理器的接口标准。
XA模式中有三个重要组件：TC、TM、RM。

TC(Transaction Coordinator)：事务协调者。维护全局和分支事务的状态，驱动全局事务提交或回滚。RocketMQ中Broker充当着TC

TM(Transaction Manager)：事务管理器。定义全局事务的范围：开始全局事务、提交或回滚全局事务。它实际是全局事务的发起者。
RocketMQ中事务消息的Producer充当着TM

RM(Resource Manager)：资源管理器。管理分支事务处理的资源，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。RocketMQ中事务消息的Producer及Broker均是RM
XA模式是一个典型的2PC，其执行原理如下：

	1. TM向TC发起指令，开启一个全局事务。
	2. 根据业务要求，各个RM会逐个向TC注册分支事务，然后TC会逐个向RM发出预执行指令。
	3. 各个RM在接收到指令后会在进行本地事务预执行。
	4. RM将预执行结果Report给TC。当然，这个结果可能是成功，也可能是失败。
	5. TC在接收到各个RM的Report后会将汇总结果上报给TM，根据汇总结果TM会向TC发出确认指 令。 若所有结果都是成功响应，则向TC发送			Global Commit指令。 只要有结果是失败响应，则向TC发送Global Rollback指令。
	6. TC在接收到指令后再次向RM发送确认指令。

注意：

事务消息不支持延时消息
对于事务消息要做好幂等性检查，因为事务消息可能不止一次被消费（因为存在回滚后再提交的 情况）
```



##### 3.5 批量消息

```
1. 发送限制
生产者进行消息发送时可以一次发送多条消息，这样可以提升发送效率，需注意以下几点：
批量发送的消息必须具有相同的Topic
批量发送的消息必须具有相同的刷盘策略
批量发送的消息不能是延时消息与事务消息

2. 批量发送大小
默认情况下，一批发送的消息总大小不能超过4MB字节。如果想超出该值，有两种解决方案：

方案一：将批量消息进行拆分，拆分为若干不大于4M的消息集合分多次批量发送
方案二：在Producer端与Broker端修改属性
Producer端需要在发送之前设置Producer的maxMessageSize属性
Broker端需要修改其加载的配置文件中的maxMessageSize属性
```



##### 3.6 消息过滤

```
消息者在进行消息订阅时，除了可以指定要订阅消息的Topic外，还可以对指定Topic中的消息根据指定条件进行过滤，即可以订阅比Topic更加细粒度的消息类型。
对于指定Topic消息的过滤有两种过滤方式：Tag过滤与SQL过滤

1. Tag过滤
通过消费者端指定要订阅消息的Tag，如果订阅多个Tag的消息，Tag间使用或运算符(||)连接

2. SQL过滤
SQL过滤是一种通过特定表达式对事先埋入到消息中的用户属性进行筛选过滤的方式。通过SQL过滤，可以实现对消息的复杂过滤。不过，只有使用PUSH模式的消费者才能使用SQL过滤。
SQL过滤表达式中支持多种常量类型与运算符。
支持的常量类型：

数值：比如：123，3.1415
字符：必须用单引号包裹起来，比如：‘abc’
布尔：TRUE 或 FALSE
NULL：特殊的常量，表示空

支持的运算符有：

数值比较：>，>=，<，<=，BETWEEN，=
字符比较：=，<>，IN
逻辑运算 ：AND，OR，NOT
NULL判断：IS NULL 或者 IS NOT NULL
```

默认情况下Broker没有开启消息的SQL过滤功能，需要在Broker加载的配置文件中添加如下属性，以开启该功能

```
enablePropertyFilter 1 = true

修改broker配置文件后，需要重启broker
```



### 四、结合 springboot 使用 RocketMQ  

技术背景：基于 JDK 8  springboot +RcoketMQ4.7+mysql +druid ，完成对消息的生产、消费以及防丢失。

1.环境的搭建

pom 坐标的引入

```xml
<dependencies>
        <!--   web容器去掉內嵌的tomcat使用jetty     -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-tomcat</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <!-- 引入jetty容器 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jetty</artifactId>
            <scope>provided</scope>
        </dependency>

        <!-- 條用工具類的jar 包       -->
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>common-utils</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>

        <!--RocketMQ的坐标 是基于 rocketMq 4.6.0 版本-->
        <dependency>
            <groupId>org.apache.rocketmq</groupId>
            <artifactId>rocketmq-spring-boot-starter</artifactId>
            <version>2.1.0</version>
        </dependency>

    </dependencies>
```

