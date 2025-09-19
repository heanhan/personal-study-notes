### Docker Compose 安装-通用

Compose 是用于定义和运行多容器 Docker 应用程序的工具。通过 Compose，您可以使用 YML 文件来配置应用程序需要的所有服务。然后，使用一个命令，就可以从 YML 文件配置中创建并启动所有服务。

### 安装

从`github`拉取,想使用指定版本, 就把链接中的`v2.4.1`换成指定

(推荐) 国内服务器请使用下方命令,已增加代理,高速下载(https://ghproxy.com/)

```shell
# 运行以下命令以下载 Docker Compose 的当前稳定版本：
sudo curl -L "https://ghproxy.com/https://github.com/docker/compose/releases/download/v2.4.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

```

2、将可执行权限应用于二进制文件

```shell
# chmod
sudo chmod +x /usr/local/bin/docker-compose
```

3、创建 软链接

软链接，以路径的形式存在, 相对于Windows操作系统中的快捷方式, 可跨文件系统

```shell
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

4、测试安装成功

```
# 测试是否安装成功
docker-compose version	# 检查版本信息
```





### 安装后报错的处理

ips:

1. 如果出现类似这种错误: `-bash: /usr/local/bin/docker-compose: Text file busy`(或者中文: `文件忙`) 说明文件正在被占用,解决方法:

   ```
   # 找出正在使用该文件的进程
   fuser /usr/local/bin/docker-compose
   
   # 杀死该进程
   sudo kill -9 PID(上面显示的数字)
   
   # 运行其他命令
   docker-compose version
   
   
   附：  # 解决fuser命令不存在-fuser:command not found
   yum install -y psmisc
   ```

   