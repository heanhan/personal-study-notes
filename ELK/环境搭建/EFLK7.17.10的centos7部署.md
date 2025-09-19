### ELK 环境的安装

#### 一、Elasticsearch 7.17.20的安装

**启动的用户是：elasticsearch** 

**elasticsearch 的账号/密码：elastic/elastic@123456**

1、下载和解压文件

```shell
以我为例，我的软件文件统一放到了   /usr/local/temp/ 文件夹下
cd /usr/local/temp/
解压
tar -xvf elasticsearch-7.17.20-linux-x86_64.tar.gz
得到 elasticsearch-7.17.20 文件夹  然后将文件夹移动到  /opt 下
mv elasticsearch-7.17.20/ /opt


```

2、创建elasticsearch 的用户

```shell
useradd elastic
chown -R elastic:elastic /opt/elasticsearch-7.17.20/
su - elastic
```

3、单机的配置

创建数据和日志存放的地址

```shell
mkdir -p  /opt/elasticsearch-7.17.20/data/
mkdir -p /opt/elasticsearch-7.17.20/logs/

```

配置elasticsearch 

```shell
vim /opt/elasticsearch-7.17.20/config/elasticsearch.yml
```

修改的文件内容如下，已经注释

```yaml
# ======================== Elasticsearch Configuration =========================
# ---------------------------------- Cluster -----------------------------------
# Use a descriptive name for your cluster:
#   集群名称
cluster.name: my-elasticsearch
# ------------------------------------ Node ------------------------------------
# Use a descriptive name for the node:
#   集群节点名称
node.name: node-1
# Add custom attributes to the node:
#node.attr.rack: r1
# ----------------------------------- Paths ------------------------------------
# Path to directory where to store the data (separate multiple locations by comma):
path.data: /opt/elasticsearch-7.17.20/data
# Path to log files:
path.logs: /opt/elasticsearch-7.17.20/logs
# ---------------------------------- Network -----------------------------------
# Set the bind address to a specific IP (IPv4 or IPv6):
#  设置绑定的ip，设置为0.0.0.0可以让任何计算机节点访问
network.host: 0.0.0.0
# Set a custom port for HTTP:
#  默认端口
http.port: 9200
# For more information, consult the network module documentation.
# --------------------------------- Discovery ----------------------------------
# Pass an initial list of hosts to perform discovery when this node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#discovery.seed_hosts: ["host1", "host2"]
discovery.seed_hosts: []
# Bootstrap the cluster using an initial set of master-eligible nodes:
cluster.initial_master_nodes: ["node-1"]
# 跨域配置 便于后面使用es-head和kabana
http.cors.enabled: true
http.cors.allow-origin: "*"

```

```she
su root
vim /etc/sysctl.conf
```

\#最后添加以下配置：注意等号两边有空格

```shell
vm.max_map_count = 655360
```

\#【保存完毕后，从指定的文件加载系统参数（不指定即从/etc/sysctl.conf中加载）】
\#执行

```shell
sysctl -p
```

```
vim /etc/security/limits.conf
```

\#【末尾加上一下内容，首单词是用户名 这里使用 * 指所有】

```shell
*  soft  noproc 65535
*  hard  noproc 65535
*  soft  nofile 65535
*  hard  nofile 65535
```

修改elasticsearch 的conf下的 jvm.options的配置文件

```shell
vim ./config/jvm.options
#这里的4g不能超过最大内存的一半，需要给lucene留内存
-Xms4g
-Xmx4g
```

启动 elasticsearch 服务

```shell
注意：不能用root用户 启动（使用刚创建的es用户）
./bin/elasticsearch -d

# 查看日志
vim ./logs/elasticsearch.log
# 使用命令访问查看服务是否启动  
curl localhost:9200
或者配置文件已经配置可以远程访问，使用 ip:9200访问
```

安装成功后访问出现如下界面

```json
// 20220106215842
// http://192.168.1.74:9200/

{
  "name": "node-1",
  "cluster_name": "my-elasticsearch",
  "cluster_uuid": "M34qY90pRwK1xa9Zi5CrfA",
  "version": {
    "number": "7.17.20",
    "build_flavor": "default",
    "build_type": "tar",
    "build_hash": "757314695644ea9a1dc2fecd26d1a43856725e65",
    "build_date": "2020-06-14T19:35:50.234439Z",
    "build_snapshot": false,
    "lucene_version": "8.5.1",
    "minimum_wire_compatibility_version": "6.8.0",
    "minimum_index_compatibility_version": "6.0.0-beta1"
  },
  "tagline": "You Know, for Search"
}
```

设置elasticsearch的用户和密码

#####  elasticsearch.yml 中添加如下配置

```yaml
# 配置X-Pack
http.cors.allow-headers: Authorization
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
```

##### 重启 elasticsearch 服务,（略）

### 设置密码

```shell
./bin/elasticsearch-setup-passwords interactive
# 接下来一系列是输入对应用户的密码  elastic，apm_system，kibana，kibana_system，logstash_system，beats_system，remote_monitoring_user 这些用户的密码

如下设置这些账号：
Changed password for user [kibana_system]
Changed password for user [kibana]
Changed password for user [logstash_system]
Changed password for user [beats_system]
Changed password for user [remote_monitoring_user]
Changed password for user [elastic]

```





#### 二、安装 Kabana



注：

```shell
kibana重启
ps -ef|grep kibana
ps -ef|grep 5601
都找不到

 


尝试 使用 fuser -n tcp 5601
kill -9 端口
 ps -ef|grep node   或 netstat -anltp|grep 5601
启动即可 ./kibana
后台启动：nohup ../bin/kibana &
```



#### 三、FileBeat 的部署
