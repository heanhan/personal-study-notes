## Docker安装中间件

1、安装RabbitMQ【单节点】

```
# 在线拉取镜像
docker pull rabbitmq:3-management

# 使用docker images查看是否已经成功拉取
```

启动MQ容器
执行docker run命令来运行MQ容器

-e参数设置环境变量: 配置登录RabbitMQ的管理平台用户和密码
hostname: 设置主机名(集群部署的时候需要用到)
-p参数设置端口映射: 15672是RabbitMQ的管理平台的端口, 5672是将来做消息通信的端口(服务之间发送消息和接收消息)

```
# 启动一个RabbitMQ容器
docker run \
  -e RABBITMQ_DEFAULT_USER=root \
  -e RABBITMQ_DEFAULT_PASS=123456 \
  --name mq \
  --hostname mq1 \
  -p 15672:15672 \
  -p 5672:5672 \
  -d \
  rabbitmq:3-management
  
  # 查看启动的容器
  docker ps
  
  # 停止或重新启动mq容器
  docker start/stop mq

```

**容器启动成功之后在浏览器中输入虚拟机ip:15672访问RabbitMQ的管理平台**