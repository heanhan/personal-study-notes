## 一、Zookeeper介绍

​	ZooKeeper 是 Apache 软件基金会的一个开源项目，主要基于 Java 语言实现。
Apache ZooKeeper 是一个开源的分布式应用程序协调服务，提供可靠的数据管理通知、数据同步、命名服务、分布式配置服务、分布式协调等服务。

### **关键特性**

- **分布式协调**：ZooKeeper提供了一个简单的架构，可以解决分布式应用中遇到的一些最复杂的问题，例如总线死锁、互斥锁、同步和组多播通信。
- 高性能：ZooKeeper旨在存储小数据量，可以完成大量的读取以及少量的写入。数据保存在内存中，因此 ZooKeeper 可以快速提供响应。
- **高可用性**：ZooKeeper使用复制来提高可用性和存储数据的稳健性。它的所有写操作都会复制到配置的所有服务器。
- **一致性**：客户端将看到相同的服务端视图，因为更新都是有序并且一致的。即使在发生更改时，所有的写操作都会有全局有序。
- **事务性**：一旦更新成功，更改将持久化，直到新的更新覆盖它。

### 应用场景

- **配置管理**：ZooKeeper可以为分布式应用提供统一的配置存储服务，当配置信息更新时，可以实时推送给所有相关的应用实例，这样就可以确保所有实例的配置信息都是一致的。
- **集群管理**：ZooKeeper是分布式环境中的一种集群管理解决方案。它可以记录和监控节点信息，比如节点状态（在线、离线、空闲等），一旦某个节点发生异常，它可以进行及时通知。
- **命名服务**：ZooKeeper提供了类似于文件系统的树状名称空间，可以实现分布式中的命名服务，每一次命名服务的改变都会被 ZooKeeper用事务的方式记录下来，从而保证命名服务的一致性。
- 选举系统：在许多分布式算法中，如 Paxos 和 Raft 等，都需要进行一种选举策略以确保在群体中能选出一个高效可靠的领导者，ZooKeeper提供了这样的机制。
- **分布式队列**：ZooKeeper可以用于实现分布式队列，队列中的元素按照 FIFO（First-In-First-Out）的顺序进行出队。这在分布式环境中的任务分配上有很大的用处。
- **分布式协调/同步（分布式锁）**：在分布式系统中，往往需要协调或同步各个节点的操作。ZooKeeper 的锁和条件变量可以使得分布式应用之间能够进行精细的同步控制。

### 安装说明

```bash
# 下载和解压
# wget https://dlcdn.apache.org/zookeeper/zookeeper-3.9.2/apache-zookeeper-3.9.2-bin.tar.gz
# tar xzvf apache-zookeeper-3.9.2-bin.tar.gz
# mv apache-zookeeper-3.9.2-bin /opt/apache
# cd /opt/apache/apache-zookeeper-3.9.2
# ls
bin  conf  docs  lib  LICENSE.txt  logs  NOTICE.txt  README.md  README_packaging.md

# 配置用户环境变量
[root@Ali zookeeper]# vim ~/.bash_profile
PATH=$PATH:/opt/apache/apache-zookeeper-3.9.2/bin/
export PATH

```

### 配置说明

#### 单机模式

在 单机模式 (Standalone Mode) 下，只有一个 ZooKeeper 服务器负责处理客户端的所有请求。但是，如果该 ZooKeeper 服务器失败了，那么整个 ZooKeeper 服务就不可用。因此，这种模式通常用于开发和测试环境。

使用 单机模式 需要一个 zoo.cfg 配置文件，可以直接在conf目录下拷贝和修改 zoo_sample.cfg

```bash
# cp conf/zoo_sample.cfg conf/zoo.cfg
# vim conf/zoo.cfg
tickTime=2000
dataDir=/opt/apache/data/zookeeper/data
dataLogDir=/opt/apache/log/zookeeper/log
clientPort=2181

tickTime: 基本时间单位（毫秒）。服务器之间维持心跳的时间间隔为1个 tickTime。
dataDir: 存储内存中数据库快照的位置。
dataLogDir : 记录写事件的事务日志，默认写到 dataDir 目录下。
clientPort: 客户端连接端口号
```

#### 集群模式

在 **集群模式 (Quorum Mode)** 下，ZooKeeper 运行在一个服务器集群中，集群中的任何一台机器都可以处理客户端的请求。如果某个服务器故障，其余的服务器仍然可以提供服务，这就实现了高可用。而且，这种模式也能保证强一致性。因为只有当过半的服务器同意进行某个操作，才能进行该操作。因此，这种模式主要用在生产环境。

**集群模式** 至少需要三台服务器，强烈建议您使用奇数台服务器。如果只有两台服务器，那么如果其中一台出现故障，则没有足够的机器来形成多数仲裁。两台服务器本质上不如一台服务器稳定，因为有两个单点故障。

```bash
伪集群模式 (Pseudo-distributed Mode) ：指在一台机器上启动多个 ZooKeeper 服务（每个服务使用不同的配置和端口），模拟 ZooKeeper 的集群模式。
这种模式中，每个 ZooKeeper 实例都以为自己是在不同的机器上运行。通常，每个实例都会有自己的配置文件，指定自己的客户端通讯端口、服务器的通信端口、数据和日志的存储位置等等，这些都需要区别于其他实例。并且，每个实例的配置文件还需要列出所有的 ZooKeeper 服务实例，包括他自己，来模拟集群中服务器间的通讯。
```

如下以 `伪集群模式` 的部署3个实例

```bash
# 实例1
# cd /usr/local/zookeeper/
# vim data/myid
1
# vim conf/zoo.cfg 
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/usr/local/zookeeper/data
dataLogDir=/usr/local/zookeeper/trans_logs
clientPort=12181

server.1=127.0.0.1:12888:13888
server.2=127.0.0.1:22888:23888
server.3=127.0.0.1:32888:33888

# 实例2

# cd /srv/zookeeper2/
# vim data/myid
2
# vim conf/zoo.cfg 
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/srv/zookeeper2/data
dataLogDir=/srv/zookeeper2/trans_logs
clientPort=22181

server.1=127.0.0.1:12888:13888
server.2=127.0.0.1:22888:23888
server.3=127.0.0.1:32888:33888

# 实例3
# cd /srv/zookeeper3/
# vim data/myid
3
# vim conf/zoo.cfg 
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/srv/zookeeper3/data
dataLogDir=/srv/zookeeper3/trans_logs
clientPort=32181

server.1=127.0.0.1:12888:13888
server.2=127.0.0.1:22888:23888
server.3=127.0.0.1:32888:33888


data/myid：服务唯一ID，必须是ASCII码的一个数值。
initLimit：这是用于限制 ZooKeeper 服务器与集群中 Leader 服务器初次建立连接时的尝试次数，如果在 tickTime * initLimit 时间内无法建立连接，则该服务器退出选举过程。这个参数的值通常大于 3。
syncLimit：这是用于限制 ZooKeeper 服务器与集群中 Leader 服务器同步数据时的尝试次数，如果在 tickTime * syncLimit 时间内无法完成数据同步，则该服务器将停止与 Leader 的同步连接。这个参数的值通常大于 2。
server.X=A:B:C：包含4个参数，参数含义分别为

X ：每台 ZooKeeper 服务器的唯一编号必须和 myid 一致，这些服务器组成了 ZooKeeper 服务集群。
A ：该服务器的主机名
B ：ZooKeeper 服务器之间的数据通信端口号
C ：ZooKeeper 服务器之间的选举 Leader 的端口号

```

### 三、使用指南

### 服务端使用说明

```bash
# zkServer.sh version
/usr/bin/java
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/bin/../conf/zoo.cfg
Apache ZooKeeper, version 3.9.2 2024-02-12 20:59 UTC

# zkServer.sh status
/usr/bin/java
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Mode: standalone

# zkServer.sh start
/usr/bin/java
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED

# zkServer.sh stop
/usr/bin/java
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/bin/../conf/zoo.cfg
Stopping zookeeper ... STOPPED

```



### 客户端使用说明

```bash
# 连接服务端
[wengjianhong@Ali ~]$ zkCli.sh -server 127.0.0.1:12181

# 查看命令
[zk: 127.0.0.1:2181(CONNECTED) 0] help
ZooKeeper -server host:port -client-configuration properties-file cmd args
	addWatch [-m mode] path # optional mode is one of [PERSISTENT, PERSISTENT_RECURSIVE] - default is PERSISTENT_RECURSIVE
	addauth scheme auth
	close
	config [-c] [-w] [-s]
	connect host:port
	create [-s] [-e] [-c] [-t ttl] path [data] [acl]
	delete [-v version] path
	deleteall path [-b batch size]
	delquota [-n|-b|-N|-B] path
	get [-s] [-w] path
	getAcl [-s] path
	getAllChildrenNumber path
	getEphemerals path
	history
	listquota path
	ls [-s] [-w] [-R] path
	printwatches on|off
	quit
	reconfig [-s] [-v version] [[-file path] | [-members serverID=host:port1:port2;port3[,...]*]] | [-add serverId=host:port1:port2;port3[,...]]* [-remove serverId[,...]*]
	redo cmdno
	removewatches path [-c|-d|-a] [-l]
	set [-s] [-v version] path data
	setAcl [-s] [-v version] [-R] path acl
	setquota -n|-b|-N|-B val path
	stat [-w] path
	sync path
	version
	whoami
	

# 查看版本
[zk: 127.0.0.1:2181(CONNECTED) 2] version
ZooKeeper CLI version: 3.9.2-e454e8c7283100c7caec6dcae2bc82aaecb63023, built on 2024-02-12 20:59 UTC

# 创建节点
[zk: 127.0.0.1:2181(CONNECTED) 3] create /init_date "20240520"
Created /init_date

# 查看节点
[zk: 127.0.0.1:2181(CONNECTED) 4] get /init_date
20240520

# 删除节点
[zkshell: 2] delete /config/topics/test
[zkshell: 3] ls /config/topics/test
	Node does not exist: /config/topics/test

```





### Zookeeper 到底是 CP 还是 AP ?

1. 严格意义上讲，ZooKeeper 实现了 P 分区容错性、C 中的写入强一致性，丧失的是 C 中的读取一致性。ZooKeeper 并不保证读取的是最新数据。
2. 如果客户端刚好链接到一个刚好没执行事务成功的节点，也就是说没和 Leader 保持一致的 Follower 节点的话，是有可能读取到非最新数据的。
3. 如果要保证读取到最新数据，请使用 sync 回调处理。这个机制的原理：是先让 Follower 保持和 Leader 一致，然后再返回结果给客户端。
4. 关于 zookeeper 到底是 CP 还是 AP 的讨论，zk 的 AP 和  CP 是从不同的角度分析的： （1）从一个读写请求分析，保证了可用性（不用阻塞等待全部 follwer 同步完成），保证不了数据的一致性，所以是AP。 （2）但是从zk架构分析，zk在leader选举期间，会暂停对外提供服务（为啥会暂停，因为zk依赖leader来保证数据一致性)，所以丢失了可用性，保证了一致性，即 CP。 再细点话，这个 C 不是强一致性，而是最终一致性。即上面的写案例，数据最终会同步到一致，只是时间问题。 （3）综上，zk 广义上来说是 CP ，狭义上是AP。

