## 安装docker-centos7.9 基于  aliyun

1、运行以下命令，下载docker-ce的yum源。

```
sudo wget -O /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

2、运行以下命令，安装Docker。

```
sudo yum -y install docker-ce
```

3、查看`docker`版本信息。

```
docker -v
```

4、启动Docker守护进程并设置开机自启动。

- 执行以下命令，启动Docker服务，并设置开机自启动。

  ```
  sudo systemctl start docker
  sudo systemctl enable docker
  ```

  执行以下命令，查看Docker是否启动。

  ```
  sudo systemctl status docker
  ```

  ## Docker基本使用

  您可以通过如下命令管理Docker守护进程。

```shell
sudo systemctl start docker     #运行Docker守护进程
sudo systemctl stop docker      #停止Docker守护进程
sudo systemctl restart docker   #重启Docker守护进程
sudo systemctl enable docker    #设置Docker开机自启动
sudo systemctl status docker    #查看Docker的运行状态
```

**管理镜像**

本文以阿里云仓库的Apache镜像为例，介绍如何使用Docker管理镜像。

- 拉取镜像。

```
sudo docker pull registry.cn-hangzhou.aliyuncs.com/lxepoo/apache-php5
```

  修改标签。如果镜像名称较长，您可以修改镜像标签以便记忆区分。

```
sudo docker tag registry.cn-hangzhou.aliyuncs.com/lxepoo/apache-php5:latest aliweb:v1
```

 查看已有镜像。

```
sudo docker images
```

强制删除镜像。

```
sudo docker rmi -f registry.cn-hangzhou.aliyuncs.com/lxepoo/apache-php5
```

启动一个容器。

您可以通过守护模式 

```
sudo docker run -d --name <容器名> <镜像ID>
```

交互模式启动一个容器。

```
# 使用交互模式启动一个新容器。
sudo docker run -it <镜像ID> /bin/bash
```

查看容器ID。

```
sudo docker ps -a
```

启动停止状态的容器。

```
sudo docker start <容器ID>
```

在一个运行中的容器中执行命令。

```
sudo docker exec -it <容器ID> /bin/bash
```

如果要退出容器，可以执行`exit`命令。

## **使用Docker制作镜像**

本步骤指导如何通过Dockerfile定制制作一个简单的Nginx镜像。

1. 执行以下命令，拉取镜像。本示例以拉取阿里云仓库的Apache镜像为例。

```
sudo docker pull registry.cn-hangzhou.aliyuncs.com/lxepoo/apache-php5
```

2、修改镜像名称标签，便于记忆。

```
sudo docker tag registry.cn-hangzhou.aliyuncs.com/lxepoo/apache-php5:latest aliweb:v1
```

3、执行以下命令，新建并编辑Dockerfile文件。

执行以下命令，新建并编辑Dockerfile文件。

```
vim Dockerfile
```

按`i`进入编辑模式，并添加以下内容，改造原镜像。

```
#声明基础镜像来源。
FROM aliweb:v1
#声明镜像拥有者。
MAINTAINER DTSTACK
#RUN后面接容器运行前需要执行的命令，由于Dockerfile文件不能超过127行，因此当命令较多时建议写到脚本中执行。
RUN mkdir /dtstact
#开机启动命令，此处最后一个命令需要是可在前台持续执行的命令，否则容器后台运行时会因为命令执行完而退出。
ENTRYPOINT ping www.aliyun.com
```

按`Esc`键，输入`:wq`并按`Enter`键，保存并退出Dockerfile文件

4、执行以下命令，基于基础镜像nginx构建新镜像。

命令格式为`docker build -t <镜像名称>:<镜像版本> .`，**命令末尾的**`**.**`**表示Dockerfile文件的路径，不能忽略。**以构建新镜像aliweb:v2为例，则命令为：

```
sudo docker build -t aliweb:v2 .
```

5、执行以下命令，查看新镜像是否构建成功。

```
sudo docker images 
```

## **安装并使用docker-compose**

docker-compose是Docker官方提供的用于定义和运行多个Docker容器的开源容器编排工具，可以使用YAML文件来配置应用程序需要的所有服务，然后使用docker-compose运行命令解析YAML文件配置，创建并启动配置文件中的所有Docker服务，具有运维成本低、部署效率高等优势。

**重要**

仅Python 3及以上版本支持docker-compose，并请确保已安装pip。

### **安装docker-compose**

1、运行以下命令，安装setuptools。

```
sudo pip3 install -U pip setuptools
```

2、运行以下命令，安装docker-compose。

```
sudo pip3 install docker-compose
```

3、运行以下命令，验证docker-compose是否安装成功。

```
docker-compose --version
```

如果回显返回docker-compose版本信息，表示docker-compose已安装成功。

### 使用docker-compose部署应用

下文以部署WordPress为例，介绍如何使用docker-compose部署应用。

创建并编辑docker-compose.yaml文件。

1. 运行以下命令，创建docker-compose.yaml文件

```
sudo vim docker-compose.yaml
```

（可选）配置镜像加速服务。

由于运营商网络原因，会导致您拉取Docker Hub镜像变慢，甚至下载失败。您可以使用阿里云容器镜像服务ACR提供的官方镜像加速器，加速官方镜像的下载。具体操作，请参见

按下`i`键，进入编辑模式，新增以下内容。

本示例以安装WordPress为例。

```
version: '3.1'             # 版本信息

services:

  wordpress:               # 服务名称         
    image: wordpress       # 镜像名称
    restart: always        # docker启动，当前容器必启动
    ports:
      - 80:80              # 映射端口
    environment:           # 编写环境
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: 123456
      WORDPRESS_DB_NAME: wordpress
    volumes:               # 映射数据卷
      - wordpress:/var/www/html

  db:                      # 服务名称    
    image: mysql:5.7       # 镜像名称
    restart: always        # docker启动，当前容器必启动
    ports:
       - 3306:3306         # 映射端口
    environment:           # 环境变量
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: 123456
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:               # 卷挂载路径
      - db:/var/lib/mysql

volumes:
  wordpress:
  db:
```

1. 按下`Esc`键，退出编辑模式，然后输入`:wq`保存并退出。

执行以下命令，启动应用。

```
sudo env "PATH=$PATH" docker-compose up -d
```

1. 在浏览器中输入`http://云服务器ECS实例的公网IP`，即可进入WordPress配置页面，您可以根据界面提示配置相关参数后，访问WordPress。