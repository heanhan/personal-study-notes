

- Mysql8.0  数据库安装

  ```
  # 进入目录(自定义下载路径)
  cd /usr/local/temp
  
  # 下载安装包
  wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.20-linux-glibc2.12-x86_64.tar.xz
  
  # 解压
  tar -xvf mysql-8.0.20-linux-glibc2.12-x86_64.tar.xz
  
  # 拷贝到/opt
  mv /usr/local/temp/mysql-8.0.20-linux-glibc2.12-x86_64 /opt
  
  # 进入/opt
  cd /opt
  
  # 修改名称为mysql
  mv mysql-8.0.20-linux-glibc2.12-x86_64 mysql
  
  # 创建存放数据文件夹  使用 -p 保证可以级联创建目录
  mkdir -p /opt/mysql/data
  mkdir -p /opt/mysql/log/mariadb
  	  创建文件: cd  /opt/mysql/log/mariadb 
  					touch mariadb.log
  mkdir -p /opt/mysql/run/mariadb/
  		创建文件: cd  /opt/mysql/run/mariadb 
  					touch mariadb.pid
  
  # 初始化数据库(记录临时密码)
  cd /opt/mysql/
  
  # 创建用户及用户组 【mysql 需要非root用户启动】
  groupadd mysql
  useradd -g mysql mysql
  
  # 授权
  chown -R mysql.mysql /opt/mysql
  
  #初始化 数据库的密码
  ./bin/mysqld --user=mysql --lower-case-table-names=1 --basedir=/opt/mysql/ --datadir=/opt/mysql/data/ --initialize ;
  
  # 配置my.cnf
  vi /etc/my.cnf
  
  # 清空，使用下面内容
  
  ==================================/etc/my.cnf 配置开始===================================================
  
  // 文件内容开始
  
  [mysqld]
  basedir=/opt/mysql
  datadir=/opt/mysql/data
  character-set-server=utf8mb4
  lower-case-table-names=1
  default_authentication_plugin=mysql_native_password
  
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
  
  [mysql]
  default-character-set=utf8mb4
  
  [client]
  default-character-set=utf8mb4
  
  // 文件内容结束
  
  ==================================/etc/my.cnf 配置结束===================================================
  
  # 建立Mysql服务  【使用  见结尾 批注】
  cp -a ./support-files/mysql.server /etc/init.d/mysql
  chmod +x /etc/init.d/mysql
  chkconfig --add mysql
  
  # 检查服务是否生效
  chkconfig --list mysql
  
  # 启动、停止、重启
  service mysql start
  service mysql stop
  service mysql restart
  
  # 创建软连接
  ln -s /optl/mysql/bin/mysql /usr/bin 
  
  # 登录（使用临时密码）
  mysql -uroot -p
  
  # 修改密码
  ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'abcd@123456';
  
  # 退出，使用新密码登录
  #配置 mysql 环境变量才能使用 mysql命令
  export PATH=$PATH:/opt/mysql/bin
  
  quit
  mysql -uroot -p
  
  # 修改root权限，增加远程连接
  use mysql
  update user set host ='%' where user='root';
  alter user 'root'@'%' identified with mysql_native_password by 'abcd@123456';
  		注：如果这句报错，看下面的报错解决方案
  flush privileges;
  
  # 退出
  quit
  ```
  
  二、报错解决方案：

  ```
  mysql连数据库的时候报错:
  
  1251 client does not support authentication protocol requested by server;consider upgrading Mysql client
  
  ERROR 1396 (HY000): Operation ALTER USER failed for 'root'@'localhost'
  
  先登录mysql
  
  mysql -u root -p
  输入密码
  
  mysql> use mysql;
  mysql> select user,host from user;
  
  +------------------+-----------+
  | user             | host      |
  +------------------+-----------+
  | root             | %         |
  | admin            | localhost |
  | mysql.infoschema | localhost |
  | mysql.session    | localhost |
  | mysql.sys        | localhost |
  | zhangj           | localhost |
  +------------------+-----------+
  
  注意我的root，host是'%'
  
  你可能执行的是:
  ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'abcd@123456';
  改成:
  
  ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'abcd@123456';
  ```
  
  

**注：补充一个知识点**

```shell
chkconfig --list #列出所有的系统服务
chkconfig --add httpd #增加httpd服务（增加服务时，需要保证有对应的服务脚本在/etc/rc.d/init.d文件夹下）
chkconfig --del httpd #删除httpd服务
chkconfig --level 35 mysqld on #设定mysqld在等级3和5为开机运行服务，--level 35表示操作只在等级3和5执行，on表示启动，off表示关闭
chkconfig mysqld on #设定mysqld在各等级为on，“各等级”包括2、3、4、5等级
```

