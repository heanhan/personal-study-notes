### 升级linux【CentOs7.9】升级内核

1、将系统的源设置为阿里云的地址

首先，备份原始的 `CentOS-Base.repo` 文件，以防出现问题时可以恢复：

sudo mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup

然后新创建一个CentOS-Base.repo 文件，然后将下面的内容复制

```
[base]
name=CentOS-$releasever - Base
baseurl=https://mirrors.aliyun.com/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7
 
[updates]
name=CentOS-$releasever - Updates
baseurl=https://mirrors.aliyun.com/centos/$releasever/updates/$basearch/
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7
 
[extras]
name=CentOS-$releasever - Extras
baseurl=https://mirrors.aliyun.com/centos/$releasever/extras/$basearch/
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7
 
[centosplus]
name=CentOS-$releasever - Plus
baseurl=https://mirrors.aliyun.com/centos/$releasever/centosplus/$basearch/
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7
```

2. 保存并关闭文件。

3. 清除旧的YUM缓存，以便YUM可以使用新的[软件源](https://so.csdn.net/so/search?q=软件源&spm=1001.2101.3001.7020)：

```less
sudo yum clean all
```

4. 更新YUM[数据库](https://so.csdn.net/so/search?q=数据库&spm=1001.2101.3001.7020)，使其包含新的软件源信息：

```undefined
sudo yum makecache
```