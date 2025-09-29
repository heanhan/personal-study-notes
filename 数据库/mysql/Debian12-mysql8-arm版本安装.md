## Debian12 操作系统上安装 mysql 8.0 【arm版本】

### 下载文件

```bash
wget https://downloads.mysql.com/archives/get/p/23/file/mysql-8.0.35-linux-glibc2.28-aarch64.tar.xz
```

2. ### 安装依赖

MySQL 二进制包需要一些依赖库。运行以下命令安装必要的依赖：

```bash
sudo apt update
sudo apt install -y libaio1 libnuma1
```

- libaio1：用于异步 I/O 操作。
- libnuma1：用于 NUMA 支持（视情况可能需要）。

3. 创建 MySQL 用户和组

   MySQL 通常以独立的非 root 用户运行。创建 MySQL 用户和组：

```bash
sudo groupadd mysql
sudo useradd -r -g mysql -s /bin/false mysql
```

#### 4. 解压和移动二进制包

1. 解压下载的二进制包：

   ```bash
   tar -xvf mysql-8.0.35-linux-glibc2.28-aarch64.tar.xz
   ```

2. 将解压后的目录移动到 /usr/local/ 并创建符号链接：

   ```bash
   sudo mv mysql-8.0.35-linux-glibc2.28-aarch64 /opt/mysql8/mysql
   ```

#### 5. 设置目录权限

确保 MySQL 目录的权限正确：

```bash
sudo chown -R mysql:mysql /opt/mysql8/mysql
sudo chmod -R 755 /opt/mysql8/mysql
```

#### 6.设置目录权限

MySQL 需要一个数据目录来存储数据库文件。创建并设置权限：

```bash
sudo mkdir /opt/mysql8/mysql/data
sudo chown -R mysql:mysql /opt/mysql8/mysql/data
```

#### 7. 初始化 MySQL

初始化 MySQL 数据目录并创建初始数据库：

```bash
sudo /opt/mysql8/mysql/bin/mysqld --initialize --user=mysql --basedir=/opt/mysql8/mysql --datadir=/opt/mysql8/mysql/data
```

- --initialize：生成一个随机初始 root 密码，记录在输出日志中（通常在 /usr/local/mysql/data/*.err 文件中）。
- 如果你希望不设置初始密码，可以使用 --initialize-insecure。

查看生成的随机 root 密码

```bash
sudo cat /opt/mysql8/mysql/data/*.err | grep password
```

记录这个密码，稍后需要使用。8. 配置 MySQL

1. 创建 MySQL 配置文件 /etc/my.cnf：

   ```bash
   sudo nano /etc/my.cnf
   ```

   添加以下基本配置：

   ```bash
   [mysqld]
   user=mysql
   basedir=/opt/mysql8/mysql
   datadir=/opt/mysql8/mysql/data
   socket=/opt/mysql8/mysql/tmp/mysql.sock
   port=3306
   pid-file=/opt/mysql8/mysql/data/mysqld.pid
   [client]
   socket=/opt/mysql8/mysql/tmp/mysql.sock
   ```

   保存并退出。

2. 设置环境变量以方便调用 MySQL 命令： 编辑 ~/.bashrc 或 /etc/profile：

   ```bash
   export PATH=$PATH:/opt/mysql8/mysql/bin
   ```

   应用更改：

   ```bash
   source ~/.bashrc
   ```

#### 9. 启动 MySQL

启动 MySQL 服务：

```bash
sudo /opt/mysql8/mysql/bin/mysqld_safe --user=mysql &
```

检查 MySQL 是否运行：

```bash
ps aux | grep mysqld
```

#### 10. 配置 systemd 服务（可选）

为了方便管理，可以为 MySQL 创建一个 systemd 服务：

1. 创建服务文件：

   ```bash
   sudo nano /etc/systemd/system/mysql.service
   ```

   添加以下内容：

   ```bash
   [Unit]
   Description=MySQL Community Server
   After=network.target
   
   [Service]
   User=mysql
   Group=mysql
   ExecStart=/opt/mysql8/mysql/bin/mysqld --defaults-file=/etc/my.cnf
   Restart=always
   LimitNOFILE=65535
   
   [Install]
   WantedBy=multi-user.target
   ```

2. 重新加载 systemd 并启用服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable mysql
sudo systemctl start mysql
```

#### 11. 登录并更改 root 密码

使用初始密码登录 MySQL：

```bash
mysql -u root -p
```

输入在上一步中记录的随机密码。

更改 root 密码：

```bash
ALTER USER 'root'@'localhost' IDENTIFIED BY 'abcd@123456';
FLUSH PRIVILEGES;
```

将 YourNewPassword 替换为你想要设置的密码。

开启远程连接

#### 12、开启 MySQL 8.0 远程连接

##### 1. 修改 MySQL 配置文件

MySQL 默认只监听本地连接（127.0.0.1）。要允许远程连接，需要修改 bind-address 设置。

1. 编辑 MySQL 配置文件 

   /etc/my.cnf

   （或你之前配置的路径，如 

   /usr/local/mysql/my.cnf

   ）：

   bash

   ```
   sudo nano /etc/my.cnf
   ```

2. 找到 

   [mysqld]

    部分，确保 

   bind-address

    设置为 

   0.0.0.0

   （监听所有 IP 地址）：

   ini

   ```
   [mysqld]
   bind-address = 0.0.0.0
   ```

   - 如果没有 bind-address 行，直接添加。

   - 确保配置文件中已包含：

     ini

     ```
     user=mysql
     basedir=/opt/mysql8/mysql
     datadir=/opt/mysql8/mysql/data
     socket=/opt/mysql8/mysql/tmp/mysql.sock
     port=3306
     pid-file=/opt/mysql8/mysql/data/mysqld.pid
     ```

3. 保存并退出。

##### 2. 重启 MySQL 服务

应用配置更改需要重启 MySQL。如果你是通过 systemd 管理 MySQL，运行：

bash

```
sudo systemctl restart mysql
```

##### 3. 配置 MySQL 用户权限

MySQL 8.0 默认限制 root 用户仅能从 localhost 登录。你需要为远程访问创建或修改用户权限。

1. 登录 MySQL：

   bash

   ```
   mysql -u root -p
   ```

   输入 root 密码。

2. 检查现有用户和主机：

   sql

   ```
   SELECT user, host FROM mysql.user;
   ```

   输出会显示所有用户及其允许连接的主机。例如，

   root@localhost

    表示 root 只能从本地登录。

3. 允许远程连接：

   - 选项 1：修改现有 root 用户

     （不推荐，仅用于测试环境）：

     sql

     ```sql
     ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'abcd@123456';
     GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'abcd@123456';
     FLUSH PRIVILEGES;
     ```

     - % 表示允许从任何 IP 地址连接。
     - 替换 YourRootPassword 为你的 root 密码。
     - mysql_native_password 是为了兼容某些客户端（如旧版 MySQL 客户端）。如果使用新版客户端，可以省略 IDENTIFIED WITH mysql_native_password。

     如果执行  GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'abcd@123456'; 报错  执行 

     ```sql
     SELECT user, host, plugin FROM mysql.user WHERE user = 'root';
     ```

     如果是 localhost 

     +------+------+-----------------------+
     | user | host | plugin                |
     +------+------+-----------------------+
     | root | localhost    | mysql_native_password |
     +------+------+-----------------------+

     需要更换执行语句也可以创建一个新的 

     ```sql
     CREATE USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'abcd@123456'; 
     GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'abcd@123456';
     FLUSH PRIVILEGES;
     ```

#### 13. 验证安装

检查 MySQL 版本：

```bash
mysqladmin -u root -p version
```

