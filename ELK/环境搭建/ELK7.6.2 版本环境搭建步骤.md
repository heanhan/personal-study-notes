

### centos7 搭建ELK 7.6.2 版本 kibana 

#### ELK简介

ELK是ElasticSerach、Logstash、Kibana三款产品名称的首字母集合，用于日志的搜集和搜索

##### Elasticsearch:

是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。设计用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。

##### Logstash:

用于管理日志和事件的工具，你可以用它去收集日志、转换日志、解析日志并将他们作为数据提供给其它模块调用，例如搜索、存储等。

##### Kibana:

是一个优秀的前端日志展示框架，它可以非常详细的将日志转化为各种图表，为用户提供强大的数据可视化支持。　　　

##### Filebeat:

类似于“agent”装在被监控端上（数据源），用来实时收集日志文件数据。

#### 一、安装准备

1、修改 设置文件打开数和进程数（要配置，避免 elasticsearch 因线程大小不足导致无法启动）

```
下载路径

wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.6.2-linux-x86_64.tar.gz
wget  https://artifacts.elastic.co/downloads/kibana/kibana-7.6.2-linux-x86_64.tar.gz
wget  https://artifacts.elastic.co/downloads/logstash/logstash-7.6.2.tar.gz
wget  https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.6.2-linux-x86_64.tar.gz

第一步：  vim /etc/security/limits.conf 中追加如下内容
*  soft  noproc 65535
*  hard  noproc 65535
*  soft  nofile 65535
*  hard  nofile 65535

第二步：  vim  /etc/sysctl.conf 追加max_map_count，避免elasticsearch 报错max virtual memory areas vm.max_map_count [65530] is too low。

max_map_count=65535

然后退出 
使用 sysctl -p 命令让文件生效

或者使用下面命令配置 sysctl.cnf 
 echo 'vm.max_map_count=655360' >> /etc/sysctl.conf ;sysctl -p   
 
 第三步：重启  让一二 步骤的配置生效
 sync;reboot
 


注：1、用root 账号修改，后面使用 为elasticsearch 创建的用户的才能启动不报错
   2、max_map_count文件包含限制一个进程可以拥有的VMA(虚拟内存区域)的数量。虚拟内存区域是一个连续的虚拟地址空间区域。在进程的生命周期中，每当程序尝试在内存中映射文件，链接到共享内存段，或者分配堆空间的时候，这些区域将被创建。调优这个值将限制进程可拥有VMA的数量。限制一个进程拥有VMA的总数可能导致应用程序出错，因为当进程达到了VMA上线但又只能释放少量的内存给其他的内核进程使用时，操作系统会抛出内存不足的错误。如果你的操作系统在NORMAL区域仅占用少量的内存，那么调低这个值可以帮助释放内存给内核用。
```

2、确保安装 jkd 1.8 以上。

#### 三、安装

##### 1、Elasticsearch 安装

```
#ElasticSerach要求以非root身份启动，所以我们要创建一个用户
groupadd elasticsearch
useradd elasticsearch -g elasticsearch

#创建文件夹
mkdir /opt/elk
#找到elasticsearch-7.6.2-linux-x86_64.tar.gz 位置进行解压
tar -xvf elasticsearch-7.6.2-linux-x86_64.tar.gz 
#将解压的文件放到
mv elasticsearch-7.6.2 /opt/elk/

//创建数据data存放的文件夹和 logs(如果没有创建)
mkdir -p /opt/elk/elasticsearch-7.6.2/data
# 创建 logs 文件夹并创建日志存放的文件 kibana.log
mkdir -p /opt/elk/elasticsearch-7.6.2/logs 
touch kibana.log
#添加授权的访问
chown -R elasticsearch.elasticsearch /opt/elk/elasticsearch-7.6.2/


修改配置  vim /opt/elk/elasticsearch-7.6.2/config/elasticsearch.yml
		修改如下基本几项
		#节点名称
		node.name: node-1
		#数据存放的路径
		path.data: /opt/elk/elasticsearch-7.6.2/data
		#日志存放的路径
		path.logs: /opt/elk/elasticsearch-7.6.2/logs
		bootstrap.system_call_filter: false
		bootstrap.memory_lock: false
		network.host: 0.0.0.0
		http.port: 9200
		
		cluster.initial_master_nodes: ["node-1"]
		
		
	启动服务 
	使用elasticsearch 用户 启动 ,root 用户无法启动 elasticsearch 服务
	su elasticsearch
	cd /opt/elk/elasticsearch-7.6.2/bin/
	./elasticsearch -d &  
	  
	  
	  注： 启动时 后面的& 加上 让 elasticsearch 以守护式启动
		

```

##### 2、logstash 安装

```
#解压 logstash 文件，并移动到自定义安装的路径  /opt/elk/
tar -zxvf logstash-7.6.2.tar.gz 
mv logstash-7.6.2 /opt/elk/

# 修改配置文件  将 config 中 复制一份 logstash-example.conf 为 logstash.conf 
cd /opt/elk/logstash-7.6.2/config
cp logstash-example.conf logstash.conf
vim logstash.conf
#配置自定义文件
vim /opt/elk/logstash-7.6.2/default.conf 
具体内容如下：

    # 监听5044端口作为输入
    input {
        beats {
            port => "5044"
        }
    }
    # 数据过滤
    filter {
        grok {
            match => { "message" => "%{COMBINEDAPACHELOG}" }
        }
        geoip {
            source => "clientip"
        }
    }
    # 输出配置为本机的9200端口，这是ElasticSerach服务的监听端口
    output {
        elasticsearch {
            hosts => ["127.0.0.1:9200"]
        }
    }
    
    
启动 logstash
nohup /opt/elk/logstash-7.6.2/bin/logstash -f /opt/elk/logstash-7.6.2/config/logstash.conf --config.reload.automatic &
```

##### 3、kibana 的安装

```
#安装 kibana 找到文件下载路径，解压文件  并移动到自定义安装的文件路径  /opt/elk/
  tar -zxvf kibana-7.6.2-linux-x86_64.tar.gz
  mv kibana-7.6.2-linux-x86_64 /opt/elk/
  
  #创建日志存放的文件夹 并创建文件 kibana.log
  mkdir  -p /opt/elk/kibana-7.6.2-linux-x86_64/logs/
  touch kibana.log

修改配置 支持中文及全网监听
  #server.host: "localhost" 改成 server.host: "0.0.0.0"
  vim /opt/elk/kibana-7.6.2-linux-x86_64/config/kibana.yml
  
  server.port: 5601
  server.host: "0.0.0.0"
  elasticsearch.hosts: ["http://192.168.20.128:9200"]
  i18n.locale: "zh-CN"
  
Kibana 启动
nohup /opt/elk/kibana-7.6.2-linux-x86_64/bin/kibana --allow-root > /opt/elk/kibana-7.6.2-linux-x86_64/logs/kibana.log &

```

##### 4、filebeat 的安装

```
暂时没搞明白原理，日后补上。
```

