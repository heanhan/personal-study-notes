### Mac下的minio的安装

使用 brew 命令安装

```
补充：
mac下使用brew安装应用，提前安装brew命令
1、brew又叫Homebrew，官网安装方式，终端命令运行以下命令
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

如果报错，MacOS系统使用Homebrew官方地址时，出现以下报错时，
curl: (35) LibreSSL SSL_connect: SSL_ERROR_SYSCALL in connection to raw.githubusercontent.com:443 

解决方案，使用国内源
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"
接下来输入 对应的需要选择国内的源，然后选择y继续执行

2、重启终端 或者 运行 配置文件
3、 使用brew -v 查看安装结果。
```

  安装 minio 文件服务器'

```
brew install minio/stable/minio
minio server /data  --  /Users/zhaojh0912/data/minio     -- 后面跟的是数据存储路径 指的是文件的存放Driver的路径盘地址
```

