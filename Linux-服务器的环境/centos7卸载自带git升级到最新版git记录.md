### centos7卸载自带git升级到最新版git记录

##### 1、查看自带的git的版本

```shell
[root@localhost ~]# git --version
git version 1.8.3.1
```

##### 2、安装依赖

```shell
[root@localhost ~]# yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel asciidoc
 
[root@localhost ~]# yum install  gcc perl-ExtUtils-MakeMaker
```

3、卸载git 

```shell
[root@localhost ~]# yum remove git
```

4、下载、编译、安装最新的git版本

```sh
请查看 当前文件下的安装的教程 
```

