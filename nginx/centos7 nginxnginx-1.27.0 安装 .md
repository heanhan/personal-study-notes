### Centos7 ngonx1.27.1 安装步骤

下载路径

```
wget  https://nginx.org/download/nginx-1.27.0.tar.gz  
```

安装需要的支持系统环境

```
（1)安装gcc环境
	yum -y install gcc gcc-c++ autoconf automake make
 (2)安装PCRE库，用于解析正则表达式
  yum install -y pcre pcre-devel
 (3)zlib压缩和解压缩依赖，
  yum install -y zlib zlib-devel
 (4)SSL 安全的加密的套接字协议层，用于HTTP安全传输，也就是https
  yum install -y openssl openssl-devel
  
```

#### 一、安装

1、解压，需要注意，解压后得到的是源码，源码需要编译后才能安装.

```
#创建一个安装文件的位置  我习惯放到 /opt/nginx
mkdir -p /opt/nginx
#找到文件的 nginx-1.18.0.tar.gz 进行解压
 tar -zxvf nginx-1.18.0.tar.gz
```

2、编译之前，先创建nginx临时目录，如果不创建，在启动nginx的过程中会报错

```
mkdir -p /var/temp/nginx 

需要创建一个文件 在文件夹下 /var/run/nginx/
touch nginx.pid 
```

3、在nginx目录，输入如下命令进行配置，目的是为了创建Makefile文件

```
./configure \
--prefix=/opt/nginx \
--pid-path=/var/run/nginx/nginx.pid \
--lock-path=/var/lock/nginx.lock \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--with-http_gzip_static_module \
--http-client-body-temp-path=/var/temp/nginx/client \
--http-proxy-temp-path=/var/temp/nginx/proxy \
--http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
--http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
--http-scgi-temp-path=/var/temp/nginx/scgi
```

4、在 nginx 目录里进行编译安装

```
make && make install

注：查看安装的位置 
whereis nginx
```

5、启动 nginx

```
进入 nginx 的安装位置
cd /opt/nginx

#启动
 ./nginx
 
#停止
./nginx -s stop

#重新加载
./nginx -s reload





```

6、设置开机启动

6.1 、在目录/usr/lib/systemd/system下创建文件nginx.service，文件内容如下:

```shell
[Unit]
Description=nginx - high performance web server 
Documentation=http://nginx.org/en/docs/
After=network. target remote-fs.target nss -lookup. target

[Service] 
Type=forking
PIDFile=/var/run/nginx/nginx.pid
ExecStartPre=/opt/nginx/sbin/nginx -t -c /opt/nginx/conf/nginx.conf
ExecStart=/opt/nginx/sbin/nginx -c /opt/nginx/conf/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID 
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install] 
WantedBy=multi-user.target
```

6.2、经过创建了上述的脚本 ，重载系统的的自启服务

```shell
systemctl daemon-reload
```

6.3、设置开启自启动

```shell
systemctl enable nginx
```

6.4 、查看开机启动状态

```shell
systemctl is-enabled nginx
```

因此经过上述的设置的启动脚本，启动、停止、重启nginx的脚本可以使用系统命令

```shell
启动： systemctl start nginx
停止： systemctl stop nginx
重启： systemctl restart nginx
```







访问 http://ip   即可  (默认是 80 端口)

![image-20201210201125584](/Users/zhaojh0912/Library/Application Support/typora-user-images/image-20201210201125584.png)

错误问题的解决：

```
1、CentOS启动nginx出现nginx: [emerg] open() "/var/run/nginx/nginx.pid" failed (2: No such file or director）

原因：没有nginx文件夹，且其下没有nginx.pid文件

解决办法：创建文件
1.进入run下：cd /var/run
2.创建nginx文件夹：mkdir nginx
3.创建nginx.pid文件：touch nginx.pid
4.进入sbin文件夹：cd /usr/local/nginx/sbin/
5.启动nginx：./nginx
6.测试是否成功：打开浏览器，地址输入localhost,出现欢迎页面
```

```
nginx 的启动、关闭、重启

查看版本：./nginx -v

启动： ./nginx 

关闭：有两种方式（建议采用：./nginx -s quit）
./nginx -s stop
./nginx -s quit

重新加载 nginx 
./nginx -s reload
```

