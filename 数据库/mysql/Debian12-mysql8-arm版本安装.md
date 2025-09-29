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
   basedir=/usr/local/mysql
   datadir=/usr/local/mysql/data
   socket=/tmp/mysql.sock
   port=3306
   pid-file=/usr/local/mysql/data/mysqld.pid
   [client]
   socket=/tmp/mysql.sock
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
ALTER USER 'root'@'localhost' IDENTIFIED BY 'YourNewPassword';
FLUSH PRIVILEGES;
```

将 YourNewPassword 替换为你想要设置的密码。

#### 12. 验证安装

检查 MySQL 版本：

```bash
mysqladmin -u root -p version
```

