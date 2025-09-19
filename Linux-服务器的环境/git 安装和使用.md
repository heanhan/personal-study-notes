Linux 的 git 安装

#### 下载地址：

- ```
  wget命令下载：wget https://github.com/git/git/archive/refs/tags/v2.31.1.tar.gz
  ```

  

#### 解压安装：

```
# 安装依赖库
yum -y install curl-devel expat-devel gettext-devel openssl-devel zlib-devel perl-devel autoconf gcc-c++

# 进入根目录
cd git-2.31.1/

# 编译git源码
make prefix=/opt/git all

# 安装git至/opt/git 路径
make prefix=/opt/git install
```

##### 配置环境变量

```
# 编辑配置文件
vim /etc/profile

# git的配置成环境变量
export PATH=$PATH:/opt/git/bin

# 使配置文件生效
source /etc/profile

测试是否成功安装
# 查询git版本
git --version
```







### [git 配置公钥 mac版本]

配置用户名

```
git config --global user.name "zhaojianhua" 
```

配置邮箱名

```
git config --global user.email "1763124707@qq.com" 
```

创建ssh key 生产公钥

```
ssh-keygen -t rsa -C "1763124707@qq.com" 
```

检查电脑是否创建了公钥和私钥

```
cd ~/.ssh 
```

获取自己的公钥 

```
cat id_rsa.pub


ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCsreRPy2jQ+xEYHdOE7AXkTE0VXxqG478teLLTxdIQgQjtFkjvT7gbaJxJTsxpal6kknGYmoxIjU4lW/TFT4YfquyH8IqqHPShUMtmic2MlOYhljVmSoDtt9tn7pi83Gwz8X8HKn9I2JOGJI52lMvWMXAoqEezoTrPGFDyX8vlZ/1HpnwKQVL1ZREu1Fv7Crl/Vv8YpAtBNgjQesYoVZL13eJV563ltBbUWgOp2mgc48nBUvFiLoX/nrP/VFSZJgjb1J8btbI76081Tm7LCL7mouvCi6zwwukn0Bs17q8M6yPQuUJHIRrWpaocoEeAfO1EpQu/MqwkBcznzy7D3mOWrLGC4v6cA0sOvoTfWHzqYi83m8Vfu3tFYHCMTcj5VaBawnfVwwFv60thlGt8o2uhoPPp7oLqoSQU4xPyVuL+2ydFgjiJbNKEWAUs5+mHa/xAygieMxjnZFPsKrthz+IqOgCmEWjrr7VgU97HvYS0N0ooOIKOG85ttOaNOgGNvZs= 1763124707@qq.com
```

