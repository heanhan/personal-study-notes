#### 1. 安装前检查

```shell
#ContOS 7安装Docker系统为64位，内核版本为3.10+
lsb_release -a

uname -r

#更新yum源
yum -y update

#查看是否已经安装Docker
yum list installed | grep docker

#若存在Dcoker,则移除
yum remove docker*
```

#### 2. 安装Docker

```shell
#yum源安装
yum -y install docker

#启动、停止、重启Docker，并查询状态
service docker start
service docker stop
service docker restart
service docker status

#或者
systemctl start docker
systemctl stop docker
systemctl restart docker
systemctl status docker

#查看Docker系统信息
docker info

#查看Docker版本
docker version

#查看镜像
docker images

#删除镜像
docker rmi [IMAGE ID]/[REPOSITORY]

#列出容器
docker ps

#显示所有的容器，包括未运行的
docker ps -a

#列出最近创建的5个容器信息
docker ps -n 5

#删除容器
docker rm -f [CONTAINER ID]/[NAMES]

#查看日志
#-t 显示时间戳
#-f 跟踪显示日志
#--tail 末尾行数，默认全部
#--since 指定开始时间，如：30m "2020-05-22"
docker logs -t -f [CONTAINER ID]/[NAMES]
docker logs -t -f --tail=100 --since=30m [CONTAINER ID]/[NAMES]
docker logs -t -f --tail=100 --since="2020-05-22" [CONTAINER ID]/[NAMES]

#查看容器详细信息
docker inspect [CONTAINER ID]/[NAMES]
```

#### 3. 安装Tomcat

```
#搜索Tomcat
docker search tomcat

#拉取镜像，拉取最新版本
docker pull tomcat

#拉取镜像，并指定版本
docker pull tomcat:8.5.4

#查看镜像
docker images

#运行镜像并制定宿主机端口映射（临时启动）
docker run -p 8080:8080 tomcat:8.5.4
#运行镜像并制定宿主机端口映射（后台启动）
#-d : 后台运行
docker run -d -p 8080:8080 tomcat:8.5.4

#部署项目
#创建本地文件夹
mkdir /usr/local/tomcat
mkdir /usr/local/tomcat/webapps

#上传项目[helloworld.war]到此路径，并运行镜像
#--name : 容器名称
#--privileged=true ： 授权
#-p : 宿主机端口映射
#-v : 挂载宿主机目录
#-d : 后台启动
docker run --name=tomcat8.5.4 --privileged=true -p 8080:8080 -v /usr/local/tomcat/webapps/hellowrold.war:/usr/local/tomcat/webapps/hellowrold.war -d tomcat:8.5.4

#启动、停止容器
docker ps
docker start [CONTAINER ID]/[NAMES]
docker stop [CONTAINER ID]/[NAMES]
```

#### 4. 安装Mysql

```shell
#搜索Mysql
docker search mysql

#拉取镜像
docker pull mysql

#创建容器
#-e : 传递环境变量
#MYSQL_ROOT_PASSWORD : root用户密码
#--lower_case_table_names=1 ： 忽略表名大小写
docker run --name=mysql --privileged=true -p 3306:3306 -v /usr/local/src/mysql/data:/var/lib/mysql -v /usr/local/src/mysql/conf/my.cnf:/etc/mysql/my.cnf -e MYSQL_ROOT_PASSWORD=3edc#EDC -d mysql --lower_case_table_names=1

#删除/usr/local/src/mysql/conf/my.cnf
cd /usr/local/src/mysql/conf/
rm -rf my.cnf/

#创建文件my.cnf
touch my.cnf

#编辑文件
vi my.cnf

#插入内容并保存
[client]
default-character-set=utf8mb4

[mysql]
default-character-set=utf8mb4

[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
max_connections=10000
default-time_zone='+8:00'
character-set-client-handshake=FALSE
character_set_server=utf8mb4
collation-server=utf8mb4_unicode_ci
init_connect='SET NAMES utf8mb4 COLLATE utf8mb4_unicode_ci'
# Custom config should go here
!includedir /etc/mysql/conf.d/

#启动mysql容器
docker start mysql

#查询启动容器
docker ps

#修改root用户密码和权限
#交互式进入容器
docker exec -it mysql /bin/bash

#进入mysql,输入上边创建容器时指定的密码[3edc#EDC]
mysql -uroot -p

#修改root密码
alter user 'root'@'localhost' identified with mysql_native_password by 'new password';

#修改root权限
use mysql;
update user set host ='%' where user='root';
alter user 'root'@'%' identified with mysql_native_password by 'new password';
flush privileges;
quit

#退出交互
exit

#使用navicat远程登录

#备份数据库
docker exec -it mysql mysqldump -uroot -p[password] [dbname] > /tmp/[dbname].bak.sql

#还原数据库
docker exec -i mysql mysql -uroot -p[password] [dbname] < /tmp/[dbname].bak.sql
```

#### 5. 安装Redis

```shell
#查询镜像
docker search redis

#拉取镜像
docker pull redis

#运行镜像
#--requirepass "123456" ： 密码
#--appendonly=yes ： 开启持久化
docker run --name=redis-server --privileged=true -p 6379:6379 -v /usr/local/redis/data:/data -d redis --requirepass="123456" --appendonly=yes

#进入Redis客户端
docker exec -it redis-server redis-cli

#输入密码
auth 123456

#测试
set key1 helloworld
get key1
```

#### 6、安装RabbitMQ

```shell
# 拉取镜像
sudo docker pull rabbitmq:management
# 创建
docker run -d --name rabbit -p 5672:5672 -p 15672:15672 -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=123456 rabbitmq:management
    
# 说明
# -p 5672:5672 -p 15672:15672    端口映射，将宿主机中的端口映射进容器中，5672是AMPQ协议端口，15672是后台管理页面端口
# -e RABBITMQ_DEFAULT_USER=admin    设置后台管理登录账号
# -e RABBITMQ_DEFAULT_PASS=123456    设置后台管理登录账号的密码
```

