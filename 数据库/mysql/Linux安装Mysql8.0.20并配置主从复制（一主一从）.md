

#### 1. 主从复制解释

  将主数据库的增删改查等操作记录到二进制日志文件中，从库接收主库日志文件，根据最后一次更新的起始位置，同步复制到从数据库中，使得主从数据库保持一致。

#### 2. 主从复制的作用

- 高可用性：主数据库异常可切换到从数据库
- 负载均衡：实现读写分离
- 备份：进行日常备份

#### 3. Mysql主从复制过程

![image-20201209102256388](/Users/zhaojh0912/Library/Application Support/typora-user-images/image-20201209102256388.png)

**Binary log：主数据库的二进制日志；Relay log：从服务器的中继日志。**

复制过程：
  （1）主数据库在每次事务完成前，将该操作记录到binlog日志文件中；
  （2）从数据库中有一个I/O线程，负责连接主数据库服务，并读取binlog日志变化，如果发现有新的变动，则将变动写入到relay-log，否则进入休眠状态；
  （3）从数据库中的SQL Thread读取中继日志，并串行执行SQL事件，使得从数据库与主数据库始终保持一致。

注意事项：
  （1）涉及时间函数时，会出现数据不一致。原因是，复制过程的两次IO操作和网络、磁盘效率等问题势必导致时间戳不一致；
  （2）涉及系统函数时，会出现不一致。如：@@hostname，获取主机名称，主从数据库服务器名称不一致导致数据不一致；
  

#### 4. 一主一从配置

![image-20201209102613805](/Users/zhaojh0912/Library/Application Support/typora-user-images/image-20201209102613805.png)

*  服务器的配置信息

| 服务器 IP      | 角色   |
| -------------- | ------ |
| 192.168.20.128 | master |
| 192.168.20.129 | Slave  |
|                |        |

安装前准备

​	1、在 192.168.20.128 、192.168.20.129 上分别安装好 mysql 

​	2、开启远程连接。

​	3、关闭主从数据库服务器防火墙或开放3306端口

```
补充：centos7 开端口
firewall-cmd --zone=public --add-port=3306/tcp --permanent   # 开放5672端口

firewall-cmd --zone=public --remove-port=3306/tcp --permanent  #关闭5672端口

firewall-cmd --reload   # 配置立即生效

查看防火墙所有开放的端口

firewall-cmd --zone=public --list-ports

检查端口被哪个进程占用
netstat -lnpt |grep 3306

==============================================================================

Copy# 查看防火墙状态
systemctl status firewalld  或者  firewall-cmd --state

# 关闭防火墙（重启后失效）
systemctl stop firewalld

#停用防火墙
systemctl disabled firewalld
```

4、确保两个数据库之间可以相互连接通信

```
# 主数据库服务器测试从数据库
mysql -uroot -p -h192.168.20.129 -P3306

# 从数据库服务器测试主数据库
mysql -uroot -p -h192.168.20.128 -P3306
```



#### 5、配置 一主一从

（1）、修改 master(主)数据库的配置文件  /etc/my.cnf

```
# 主从复制-主机配置
# 主服务器唯一ID
server-id=1
# 启用二进制日志
log-bin=mysql-bin
# 设置不要复制的数据库(可设置多个)
binlog-ignore-db=sys
binlog-ignore-db=mysql
binlog-ignore-db=information_schema
binlog-ignore-db=performance_schema
# 设置需要复制的数据库(可设置多个)
binlog-do-db=test
# 设置logbin格式
binlog_format=STATEMENT

// 文件内容结束
```

(2)、修改 slave （从）数据库的配置文件  /etc/my.cnf

```
# 配置my.cnf
vi /etc/my.cnf

# 清空，使用下面内容
// 文件内容开始

[mysqld]
basedir=/usr/local/mysql-8.0.20
datadir=/usr/local/mysql-8.0.20/data
character-set-server=utf8
lower-case-table-names=1
default_authentication_plugin=mysql_native_password

# 主从复制-从机配置
# 从服务器唯一ID
server-id=2
# 启用中继日志
relay-log=mysql-relay

// 文件内容结束
```

（3）主数据库创建用户slave并授权

```
# 登录
mysql -uroot -p

# 创建用户
create user 'slave'@'%' identified with mysql_native_password by '123456';

# 授权
grant replication slave on *.* to 'slave'@'%';

# 刷新权限
flush privileges;

从数据库（即使用 192.168.20.129远程连接 192.168.20.128）验证slave用户是否可用
mysql -uslave -p -h192.168.29.128 -P3306
```

（4）主数据库查询服务ID及Master状态

```
# 登录
mysql -uroot -p

# 查询server_id是否可配置文件中一致
show variables like 'server_id';

# 若不一致，可设置临时ID（重启失效）
set global server_id = 1;

# 查询Master状态，并记录 File 和 Position 的值
show master status;

# 注意：执行完此步骤后退出主数据库，防止再次操作导致 File 和 Position 的值发生变化
```

（5）从数据库设置主数据库信息

```
# 登录
mysql -uroot -p

# 查询server_id是否可配置文件中一致
show variables like 'server_id';

# 若不一致，可设置临时ID（重启失效）
set global server_id = 2;

# 设置主数据库参数  
change master to master_host='192.168.20.128',master_port=3306,master_user='slave',master_password='password',master_log_file='mysql-bin.000002',master_log_pos=156;
# 注 mysql-bin.00000与156 值 是从主数据库中通过命令show master status;查询出来，要保证一样
# 开始同步
start slave;

# 若出现错误，则停止同步，重置后再次启动
stop slave;
reset slave;
start slave;

# 查询Slave状态
show slave status\G

# 查看是否配置成功
# 查看参数 Slave_IO_Running 和 Slave_SQL_Running 是否都为yes，则证明配置成功。若为no，则需要查看对应的 Last_IO_Error 或 Last_SQL_Error 的异常值。
```

