## Centos7使用yum安装Docker(使用阿里源)

#### 1、更新 yum

```bash
sudo yum update
```

#### 2、安装必要的工具

```bash
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

#### 3、添加软件源

```bash
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

#### 4、更新并安装Docker-CE

```bash
sudo yum makecache fast
sudo yum -y install docker-ce
```

#### 5、开启 Docker 服务

```bash
sudo service docker start
```

#### 6、让Docker 服务开机启动

```bash
sudo systemctl enable docker.service
```



