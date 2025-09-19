docker 中安装 RabbitMQ 消息队列 步骤

安装步骤

```
1、先 搜索 rabbitmq 镜像

docker search rabbitmq 
2、拉取镜像
docker pull rabbitmq 
3、拉取完成之后，查看镜像信息，安装时需要该
docker images
4、安装镜像 通过 image id 定位镜像进行安装   下面的c05fdf32bdad 是通过docker images 查看得到
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 c05fdf32bdad    
5、查看正在运行的容器  取保惊醒制作成功并成功启动，该命令可以多使用几次，确保成功没有闪退。
docker ps
6、启动rabbitmq_management，下面的  rabbitmq为制作镜像时自定义的镜像的应用名称
docker exec -it rabbitmq rabbitmq-plugins enable rabbitmq_management
7、在网页中输入http://ip:15672/
8、登录默认用户密码为guest/gues

```

 