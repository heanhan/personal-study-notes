### 更新centos7 的系统时间

```
使用  date -R 查看时间和区时  
 是标识  为东八区  +0800
安装工具：sudo yum install -y  ntpdate

同步系统时间与网络时间：sudo ntpdate cn.pool.ntp.org

再次查看 时间  date -R 


```

# vmware安装完centos7之后 时区不对

1、查看修改时间

timedatectl（查看时间状态）

timedatectl set-local-rtc 1 (将时钟调整为与本地时钟一致, 0 为设置为 UTC 时间)

timedatectl set-timezone Asia/Shanghai （设置系统时区为上海）

2、校准时间（通过阿里云时间服务器校准时间）

yum -y install ntp

ntpdate ntp1.aliyun.com
