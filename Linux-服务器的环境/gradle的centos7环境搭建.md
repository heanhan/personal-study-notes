### centos7服务器环境搭建Gradle7.5 

###### 1、下载 gradle

```
https://gradle.org/releases/
选择relese的稳定版本的 二进制的文件包下载  本次采用7.5

```

###### 2、解压安装

```
mkdir /opt/gradle
unzip -d /opt/env/gradle gradle-7.5-bin.zip
ls /opt/env/gradle/gradle-7.5
```

###### 3、配置环境变量

```
export PATH=$PATH:/opt/env/gradle/gradle-7.5/bin

```

###### 4、重启配置

```
source /etc/profile
```

###### 5、验证配置成功

```
gradle -v
```

