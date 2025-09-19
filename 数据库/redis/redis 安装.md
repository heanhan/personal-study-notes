```shell
一、安装gcc依赖

由于 redis 是用 C 语言开发，安装之前必先确认是否安装 gcc 环境（gcc -v），如果没有安装，执行以下命令进行安装

 [root@localhost local]# yum install -y gcc 

二、下载并解压安装包

[root@localhost local]# wget cdredis-5.0.3.tar.gz

[root@localhost local]# tar -zxvf redis-5.0.3.tar.gz

三、cd切换到redis解压目录下，执行编译
m
[root@localhost local]# cd redis-5.0.3

[root@localhost redis-5.0.3]# make

四、安装并指定安装目录

[root@localhost redis-5.0.3]# make install PREFIX=/usr/local/redis

 
五、启动服务

5.1前台启动

[root@localhost redis-5.0.3]# cd /usr/local/redis/bin/

[root@localhost bin]# ./redis-server

5.2后台启动

从 redis 的源码目录中复制 redis.conf 到 redis 的安装目录

[root@localhost bin]# cp /usr/local/redis/redis.conf /usr/local/redis/bin/

 
修改 redis.conf 文件，把 daemonize no 改为 daemonize yes

[root@localhost bin]# vi redis.conf

后台启动

[root@localhost bin]# ./redis-server redis.conf


六、设置开机启动

添加开机启动服务

[root@localhost bin]# vi /etc/systemd/system/redis.service

复制粘贴以下内容：

复制代码
[Unit]
Description=redis-server
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/redis/bin/redis-server /usr/local/redis/bin/redis.conf
PrivateTmp=true

[Install]
WantedBy=multi-user.target

复制代码
注意：ExecStart配置成自己的路径 


设置开机启动

[root@localhost bin]# systemctl daemon-reload

[root@localhost bin]# systemctl start redis.service

[root@localhost bin]# systemctl enable redis.service

创建 redis 命令软链接

[root@localhost ~]# ln -s /usr/local/redis/bin/redis-cli /usr/bin/redis

测试 redis


服务操作命令

systemctl start redis.service   #启动redis服务

systemctl stop redis.service   #停止redis服务

systemctl restart redis.service   #重新启动服务

systemctl status redis.service   #查看服务当前状态

systemctl enable redis.service   #设置开机自启动

systemctl disable redis.service   #停止开机自启动

-----------------------------------------------------------------------------------------

下载源码包
wget http://download.redis.io/releases/redis-5.0.0.tar.gz

解压安装包
tar -zxvf redis-5.0.0.tar.gz -C /usr/local

编译安装
cd /usr/local
mv redis-5.0.0 redis
cd redis
make &&  make install

注册服务
cp /usr/local/redis/utils/redis_init_script /etc/rc.d/init.d/redis
vim /etc/rc.d/init.d/redis  //修改脚本文件
#chkconfig: 2345 80 90  //第二行加入
chkconfig --add redis  //注册服务

修改配置文件
mkdir -p /etc/redis
cp /usr/local/redis/redis.conf /etc/redis/6379.conf
vim /etc/redis/6379.conf
#注释bind 127.0.0.1（用于远程连接），将“daemonize no”修改为“daemonize yes”
#bind 127.0.0.1
设置后台运行（守护进程）：daemonize修改为yes
设置aof持久化：appendonly修改为yes
设置连接密码：去掉#，requirepass 后面的字符串则为密码

设置开启启动
systemctl enable redis
systemctl start|stop|restart redis 



```

