### linux之centos7搭建MongoDB4.4.x

> 本文档基于 mongdb 4.4.26版本搭建  本地虚拟机的ip 172.16.75.111  

#### 安装

```bash
1、解压文件  mongodb-linux-x86_64-rhel70-4.4.26.tgz  并将文件移动到指定的 opt文件夹下
# tar -xvf  mongodb-linux-x86_64-rhel70-4.4.26.tgz 
# mvn mongodb-linux-x86_64-rhel70-4.4.26 mongodb
#进入文件夹 mongodb中创建 data和log的文件夹
# mkdir data/ log/
对这两个文件夹授权  设置读写权限
chmod 777 data/ log/ 
```

#### 配置

```bash
使用mongo的配置文件形式进行设置mongo
在mongodb目录下新建配置文件mongodb.conf。（建议配置）

# vi /usr/local/mongodb/mongodb.conf

#配置文件中的目录和已创建的一一对应 ,文件如下
```

```shell
# 数据库数据存放目录
dbpath=/opt/mongodb/data
# 日志文件存放目录
logpath=/opt/mongodb/log/mongodb.log
# 日志追加方式
logappend=true
# 端口
port=27017
# 是否认证（首次创建超级管理员账户时，需要先设置为false。）
auth=false
# 以守护进程方式在后台运行
fork=true
# 远程连接要指定ip，否则无法连接；0.0.0.0代表不限制ip访问
bind_ip=0.0.0.0
```

*** **注意** ：上面配置文件中 auth 属性，在第一次进入mongo后设置管理账号后，在设置为ture ,重启后mongodb 操作需要认证。

##### 配置环境变量：

在/etc/profile 末尾添加以下内容并保存，最后使用 source /etc/profile命令重启系统配置。

```bash
export MONGODB_HOME=/opt/mongodb
export PATH=$PATH:$MONGODB_HOME/bin
```

#### **服务&自启动**

\# vi /etc/init.d/mongodb  

#注意前两行命令的目录需要根据实际情况对应。

```bash
#!/bin/bash
#chkconfig:  2345 81 96
#description: mongodb
start() {
/opt/mongodb/bin/mongod --config /opt/mongodb/mongodb.conf
}
stop() {
/opt/mongodb/bin/mongod --config /opt/mongodb/mongodb.conf --shutdown
}
case "$1" in
start)
start
;;
stop)
stop
;;
restart)
stop
start
;;
*)
echo
$"Usage: $0 {start|stop|restart}"
exit 1
esac
```

##### 添加自启动脚本执行权限　　

　　

```bash
# chmod 775 /etc/init.d/mongodb
# chkconfig --add mongodb
# chkconfig mongodb on

启动MongoDB，

# service mongodb start
```

浏览器访问：ip:port（根据实际IP调整），看到一下信息表示可以成功访问:

```
It looks like you are trying to access MongoDB over HTTP on the native driver port.
```

上述信息标识登录成功

##### 后续创建账号

MongoDB安装好后第一次进入是不需要密码的，也没有任何用户，通过shell命令 可直接进入
　　MongoDB 没有无敌用户root，只有能管理用户的角色：userAdminAnyDatabase

　　1、进入bin目录用 ./mongo ip:端口 进行命令行连接：

　　#./mongo 172.16.75.111:27017

2、创建用户admin：

```bash
db.createUser({user:"root",pwd:"abcd@123456",roles: [{role:"userAdminAnyDatabase", db:"admin"}]});
```

3、开始认证模式，重启MongoDB，使用超管admin进入：

```bash
db.auth("root","123456")
```

4、建新的数据库和用户：

```bash
#创建新数据库
use test
#建操作权限用户
db.createUser({user:"medknedit-test",pwd:"123456",roles: [{ "role": "readWrite", "db": "medknedit-test" },{ "role": "userAdmin", "db": "medknedit-test" }]});
```