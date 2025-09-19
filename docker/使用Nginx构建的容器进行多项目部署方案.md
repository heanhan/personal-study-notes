## 基于Docker容器构建Nginx 容器部署多套前端方案

### 基于192.168.8.82 服务器上的路径

```
/home/data/v1.6/nginx/        
[root@localhost nginx]# tree  -L 3
.
├── conf								# Nginx 配置目录
│   ├── conf.d					# 存放各项目配置文件的目录
│   │   ├── local-one.conf  # 项目一的配置
│   │   └── local-two.conf  # 项目二的配置
│   ├── mime.types
│   └── nginx.conf			# 主配置文件
├── docker-compose.yml	#docker-compose.yml
├── html								# 静态文件根目录
│   ├── one							# 项目一的静态文件
│   │   ├── index11.html
│   │   └── index.html
│   └── two							# 项目二的静态文件
│       ├── index22.html
│       └── index.html
└── logs
    ├── access.log
    ├── error.log

```

