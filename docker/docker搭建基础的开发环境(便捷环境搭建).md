## 使用docker 快速搭建开发环境

#### 1、安装docker(建议两个核心)

```
yum install -y yum-utils device-mapper-persistent-data lvm2    //安装必要工具
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo //设置yum源
yum install -y docker-ce  //下载docker
systemctl start docker   //启动docker

```



#### 2、安装mysql

```
docker pull mysql //下载MySQL镜像
docker run --name mysql --restart=always -p 3306:3306 -e MYSQL_ROOT_PASSWORD=abcd@123456 -d mysql //启动MySQL

```

#### 3、安装Redis

```
docker pull redis //下载Redis镜像
docker run --name redis  --restart=always -p 6379:6379 -d redis --requirepass "密码" //启动Redis

```

#### 4、安装RabbitMQ 

```
docker pull rabbitmq:management //下载RabbitMQ镜像
docker run --name rabbit --restart=always -p 15672:15672 -p 5672:5672  -d  rabbitmq:management   //启动RabbitMQ,默认guest用户，密码也是guest。

```

