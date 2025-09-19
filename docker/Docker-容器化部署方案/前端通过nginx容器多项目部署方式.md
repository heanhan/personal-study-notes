

## 基于Docker 容器构建统一的nginx 镜像 部署多套前端的实现方案

### 1. 拉取 Nginx 镜像

首先，拉取官方的 Nginx 镜像【目前82 服务器上已经配置可以去的镜像源地址】：

```bash
docker pull nginx:1.24.0
```

### 2. 创建基础 Nginx 容器

创建一个基础 Nginx 容器，以便从容器中复制配置文件和 HTML 文件到本地目录：

```bash
docker run --name demo-nginx -p 18080:80 -d nginx:1.24.0
```

### 3. 复制配置文件和 HTML 文件到本地目录

将容器中的 Nginx 配置文件和 HTML 文件复制到本地目录：

```
在宿主机上执行 创建对应的前端路径地址，目前以当前82 服务器上的路径作为参考，我将前端部署在 /data/nginx/下 为例子
mkdir -p /data/nginx/{conf,html,log}

docker cp demo-nginx:/etc/nginx/nginx.conf /data/nginx/conf/
docker cp demo-nginx:/etc/nginx/conf.d/ /data/nginx/conf/
docker cp demo-nginx:/etc/nginx/mime.types /data/nginx/conf/
docker cp demo-nginx:/usr/share/nginx/html /data/nginx/html/
docker cp demo-nginx:/var/log/nginx /data/nginx/log/
```

### 4. 停止并删除基础容器

停止并删除基础容器【把第二步骤构建的容器删除，构建第二步骤只为了 得到第三步的一些路径和文件】：

```bash
docker stop demo-nginx
docker rm -f demo-nginx
```

### 5. 创建 Dockerfile

创建一个 Dockerfile 来构建自定义的 Nginx 镜像：

Dockerfile 文件路径位于  /data/nginx 下

```bash
#创建文件
touch Dockerfile 
# 添加文件操作的权限
chmod 755 Dockerfile
```

具体的 Dockerfile 内容如下 

```bash
FROM nginx:1.24.0
# 复制配置文件和 HTML 文件
COPY nginx.conf /etc/nginx/
COPY conf.d/ /etc/nginx/conf.d/
COPY html/ /usr/share/nginx/html/
COPY log/ /var/log/nginx/
```

### 6. 构建自定义 Nginx 镜像

在包含 Dockerfile 的目录中构建自定义 Nginx 镜像：

如 我在 82服务器上构建 nginx 的容器镜像名称为  nginx-common

```bash
docker build -t nginx-common .
```

### 7. 使用 Docker Compose 编排部署部署多个前端项目

创建一个 `docker-compose.yml` 文件，用于管理多个前端项目的部署：

```yml
#该项的值 取决于 docker-compose 版本，26版本后 都是 3.+ ，之前是2 
version: '3.8'

services:
  nginx:
    image: nginx-common
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - /data/nginx/conf:/etc/nginx
      - /data/nginx/html:/usr/share/nginx/html
      - /data/nginx/log:/var/log/nginx
    networks:
      - my-network
    restart: unless-stopped
    
 networks:
  my-network:
    driver: bridge
```

### 8. 启动 Nginx 服务

运行以下命令启动 Nginx 服务：

**注释**：因为同台服务器上 不同的项目组可能都有自己的 编排的 docker-compose.yml文件

因此启动docker-compose.yml  尽量指定自己的路径 指定编排文件，防止启动他人编排文件 

使用参数  -p  加上当前文件的名称 ，如当前 docker-compose.yml 位于  /data/nginx 下 因此

启动参数  -p nginx 即可。

```
docker-commpose -p nginx up -d
```

### 9. 部署多个前端项目

1、将每个前端项目的 HTML 文件放置在 `/data/nginx/html` 目录下的不同子目录中。例如：

```bash
/data/nginx/html/project1/index.html
/data/nginx/html/project2/index.html
```

2、配置 Nginx 配置文件

在 `/data/nginx/conf/conf.d/default.conf` 文件中，通过添加多个 `server` 块来配置不同的前端项目。

### 11. 重启 Nginx 服务

每次修改配置文件或更新 HTML 文件后，重启 Nginx 服务以应用更改：

```bash
docker-compose restart nginx
或者使用 
docker exec nginx nginx -s reload
```

通过以上步骤，你可以自由部署多个前端项目，并通过 Nginx 的配置文件来管理每个项目的访问路径和静态文件目录。