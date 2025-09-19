## RocketMQ 最新的 4.7.1 安装前准备:

1. 建议使用64位操作系统，Linux / Unix / Mac;本次使用的是CentOS7。
2. 64位JDK 1.8+;RocketMQ4.3.x之后都是使用jdk1.8+，所以你得保证你得安装环境必须装有jdk1.8+
3. Maven 3.2.x；Maven工具其实不一定需要使用的，但是也建议大家在安装环境下配置好Maven3.3.9+
4. 4g+内存用于Broker服务器, RocketMQ定位是分布式消息中间件，属重量级应用，所以它是比较吃内存的。但是如果你只是想在本地尝试一下搭建流程，或者你是想在自己的虚拟机上搭建一个对性能没什么要求的测试服务器，那么你可以只为这个虚拟机分配1.5g + 的内存。当然，低内存就代表着低性能，而且RocketMQ在低内存下运行是相对容易出错的，具体的内存分配，看你个人情况而定。
5. 本次安装的是单机版本

## 一、下载RocketMQ到安装环境下:

开始正式安装RocketMQ了。首先我们需要下载RocketMQ的安装包：本次使用的 4.7 版本，然后将安装包上传到Linux服务器。

## 二、完成RocketMQ的安装和配置

**2.1 将安装文件解压到安装目录下,这里将它安装到 /opt/ 下**

```
/opt/ 是我个人安装软件习惯存放的地方，个人自定义
unzip rocketmq-all-4.7.1-bin-release.zip
注:
1、如果没有unzip命令，可以使用yum命令安装:   yum install unzip -y;

2、解压完之后，可以看到 /opt 目录下多了个文件夹 -> rocketmq-all-4.7.1-bin-release，由于文件名太长，决定使用 ln命令给这个目录映射一个简短的新名字:
ln -s rocketmq-all-4.7.1-bin-release rocketmq
这样以后配置文件目录时只需要配置rocketmq即可替代rocketmq-all-4.3.1-bin-release
【第2操作可以忽略，不重要 只是介绍 ln 命令的使用方式 ，也可使用 mv 更改名称，这里只是使用 ln命令而已,本次我是用 mv rocketmq-all-4.7.1-bin-release rocketmq 】

```

**2.2 创建文件存储目录**   这个为后面自定义的 （2.3 修改配置文件） 配置文件设置，非必要更改的项目，

```
mkdir -p /opt/rocketmq/store
mkdir -p /opt/rocketmq/store/commitlog
mkdir -p /opt/rocketmq/store/consumequeue
```

**2.3 修改配置文件**

```
vim /opt/rocketmq/conf/2m-noslave/broker-a.properties
打开后是 rocket 自带的配置内容，可以删掉，使用下面的配置文件内容
这里建议将默认配置删除，我这里贴出一份本人的配置文件[配置参考网络]

## 配置broker的配置文件##
## 集群名称
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样 
brokerName=broker-a
#0 表示 Master，>0 表示 Slave
brokerId=0
#nameServer地址，如果mq部署在局域网上，则用局域网ip，如果是在阿里云上，配置成阿里云外网ip
namesrvAddr=192.168.20.128:9876
 
## broker ip地址
brokerIP1=192.168.20.128
 
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
 
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
 
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
 
#Broker 对外服务的监听端口
listenPort=10911
 
#删除文件时间点，默认凌晨 4点
deleteWhen=04
 
#文件保留时间，默认 48 小时
fileReservedTime=120 
 
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824 
 
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
 
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
##设置的磁盘最大利用率。默认是75
diskMaxUsedSpaceRatio=75
#存储路径
storePathRootDir=/opt/rocketmq/store
#commitLog 存储路径
storePathCommitLog=/opt/rocketmq/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/opt/rocketmq/store/consumequeue
#消息索引存储路径
storePathIndex=/opt/rocketmq/store/index
#checkpoint 文件存储路径
storeCheckpoint=/opt/rocketmq/store/index/checkpoint
#abort 文件存储路径
abortFile=/opt/rocketmq/store/index/abort
#限制的消息大小 建议 65536
maxMessageSize=10240
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
 
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=ASYNC_MASTER
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量 128
#sendMessageThreadPoolNums=64
#拉消息线程池数量 128
#pullMessageThreadPoolNums=64


```

**2.4 修改日志配置:**

```

mkdir -p /opt/rocketmq/logs
cd /opt/rocketmq/conf && sed -i 's#${user.home}#/opt/rocketmq#g' *.xml

第二个命令可分为两部分: 首先cd /usr/local/rocketmq/conf 自 是进入rocketmq的config目录下，然后sed -i 's#${user.home}#/usr/local/rocketmq#g' *.xml 是为了将conf目录下所有的xml文件中的 ${user.home}内容修改为 /usr/local/rocketmq
```

修改前日志文件 ，可以看到RocketMQ的日志文件默认是打印到user.home目录下。我们执行sed 命令后，改变日志的输出路径:

可以看到RocketMQ的日志文件默认是打印到user.home目录下。我们执行sed 命令后，改变日志的输出路径:



**2.5 修改启动命令**

到了这一步其实已经可以启动了，但是如果你的安装环境内存不够的话，必须要修改了启动参数才能正常运行.

*2.5.1 修改broker启动参数*

```
vim /opt/rocketmq/bin/runbroker.sh

修改java-configuration :
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx512m -Xmn256"  修改堆内存的配置
```

*25.2 修改server的启动参数*

```
vim /opt/rocketmq/bin/runserver.sh

修改后：
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx512m -Xmn256m -XX:MetaspaceSize=64m -XX:MaxMetaspaceSize=128m"
上述配置的参数值仅仅是为了降低内存占用而设，其值没有其他参考价值
```

## 三、正式启动RocketMQ

**3.1 进入RocketMQ的bin目录下**

```
cd /opt/rocketmq/bin
```

3.2 启动nameserver

```
nohup sh mqnamesrv &

至于为什么要用nohup和&, nohup 命令用途是:不挂断地运行命令，& 放在命令后面表示设置此进程为后台进程。
这里可以用jps查看nameserver的启动情况，如果出现如下的界面，说明已经启动成功

注：在启动nameserver的过程中，有时会让当前命令行界面丢失，无法输入指令。这时候只能新开一个窗口来执行后面的操作
```

**3.3 启动broker**

```
nohup sh mqbroker -c /optl/rocketmq/conf/2m-noslave/broker-a.properties >/dev/null 2>&1 &

使用jps命令查看程序运行情况:
```

**3.4 查看RocketMQ的broker日志和namesrv日志:**

```
tail -f -n 500 /opt/rocketmq/logs/rocketmqlogs/broker.log

tail -f -n 500 /opt/rocketmq/logs/rocketmqlogs/namesrv.log
```

**3.5关闭nameserver和broker命令**

```
1、关闭**nameserver**服务
sh mqshutdown namesrv

2、关闭broker服务
sh mqshutdown broker

```

## 四、使用Tomcat部署RocketMQ的控制台

**4.1 首先下载RocketMQ的控制台扩展**

下载地址:https://github.com/apache/rocketmq-externals/tree/rocketmq-console-1.0.0

这里需要注意的是，我们这里下载的是[rocketmq-externals](https://github.com/apache/rocketmq-externals/tree/rocketmq-console-1.0.0)项目的1.0.0分支，这个分支亲测可以正常运行。



**4.2 配置console项目的application.properties文件**

![image-20201208151644811](/Users/zhaojh0912/Library/Application Support/typora-user-images/image-20201208151644811.png)

```
注：配置文件中第一个红色框 配置是配置ma服务器的地址
第二个红色框,一定要关闭，否则造成 mq无法连接
```

**4.3 打包项目**

```
使用 idea 的集成打包，也可以使用命令打包如：mvn package -Dmaven.test.skip
```

**4.4 运行项目**

将项目打包生成的jar包使用java -jar即可运行起来，如果jar包运行不起来，可以试试在项目根目录下运行 mvn spring-boot:run

运行完毕后，在浏览器上输入 localhost:8080即可启动RocketMQ的监控系统:

![image-20201208152025596](/Users/zhaojh0912/Library/Application Support/typora-user-images/image-20201208152025596.png)

好了，RocketMQ就此安装结束