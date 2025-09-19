### 使用 cronolog做日志的处理

#### 一、背景 安装软件

安装软件有两种方式：

```text
方式1: 通过 yum方式安装
yum -y install cronolog

默认的安装位置：   
方式2 自定下载：
1、 下载(一般网站访问失败概率挺大)

	wget http://cronolog.org/download/cronolog-1.6.2.tar.gz

2、 文件进行解压

 	tar zxvf cronolog-1.6.2.tar.gz
 
3、将文件解压放到指定的文件路径后，

	例如：我的放入 /opt/cronolog/
	
4、编译安装文件（推荐使用）
	
	./configure --prefix=/opt/cronolog/
  make
  make install
  
  注： linux 下将文件安装到指定的位置：如果我的第一种方式不行（我另一台服务器上述命令失效），试试另一种方式
  【
  1、另一种
  ./configure
  make
  make install DESTDIR=/opt/cronolog/
  2、另一种（上述的方法都不行 试试这种）
  make prefix=/opt/cronolog/
  make install prefix=/opt/cronolog/
  3、以上都不行，那就逐个拷贝。

】

5、校验是否安装成功
	which cronolog
	记住 这个路径，后续服务启动脚本需要依赖。我的 /usr/local/sbin/cronolog
	
```

#### 二、使用场景

##### 1、（springboot ） 使用nohup 守护进程启动服务

介绍一下nohup：no hang up 不挂断的意思。

在缺省的情况下该作业的所有输出都会被重定向的一个名为nohup.out的文件中。
讲微服务的日志进行切分,如 按照每天的切分日志，启动命令脚本如下。

```

nohup java -jar 你的服务jar包.jar --config application-prod.yml(这个服务的启动配置文件，没有可以删掉) | /usr/sbin/cronolog ./log/自定义日志名前缀-%Y-%m-%d.out >> /dev/null 2>&1 & 
```

##### 2、tomcat 部署的服务进行日志切分

修改要分隔的Tomcat的日志下bin/catalina.sh文件

修改要分隔的Tomcat的日志下bin/catalina.sh文件，要修改的在290行。修改前先拷贝一份。

1. cp catalina.sh catalina.sh.bak

2. vim catalina.sh -c 417  或者  vim catalina.sh之后:417    (进入到catalina.sh的第417行。)

  ```sh
  修改为：
    shift
    # touch "$CATALINA_OUT"
    if [ "$1" = "-security" ] ; then
      if [ $have_tty -eq 1 ]; then
        echo "Using Security Manager"
      fi
      shift
      eval $_NOHUP "\"$_RUNJAVA\"" "\"$LOGGING_CONFIG\"" $LOGGING_MANAGER $JAVA_OPTS $CATALINA_OPTS \
        -classpath "\"$CLASSPATH\"" \
        -Djava.security.manager \
        -Djava.security.policy=="\"$CATALINA_BASE/conf/catalina.policy\"" \
        -Dcatalina.base="\"$CATALINA_BASE\"" \
        -Dcatalina.home="\"$CATALINA_HOME\"" \
        -Djava.io.tmpdir="\"$CATALINA_TMPDIR\"" \
        org.apache.catalina.startup.Bootstrap "$@" start 2>&1\
        | /usr/local/sbin/cronolog "$CATALINA_BASE"/logs/catalina.%Y-%m-%d.out >> /dev/null &
   
    else
      eval $_NOHUP "\"$_RUNJAVA\"" "\"$LOGGING_CONFIG\"" $LOGGING_MANAGER $JAVA_OPTS $CATALINA_OPTS \
        -classpath "\"$CLASSPATH\"" \
        -Dcatalina.base="\"$CATALINA_BASE\"" \
        -Dcatalina.home="\"$CATALINA_HOME\"" \
        -Djava.io.tmpdir="\"$CATALINA_TMPDIR\"" \
        org.apache.catalina.startup.Bootstrap "$@" start 2>&1\
        | /usr/local/sbin/cronolog "$CATALINA_BASE"/logs/catalina.%Y-%m-%d.out >> /dev/null &
   
    fi
  ```

   

### 最后：使用shell 利用crontab 自动清理日志

linux 下运行程序有时会产生大量的记录日志，以便排除隐藏很深的问题，但时间一长就会占用很多的磁盘空间，每天手动清除也比较麻烦，因此一个定时执行的脚本很有必要

创建删除文件 的shell的脚本。

```sh
find 对应目录 -mtime +天数 -name "文件名" -exec rm -rf {} \ ;
例子：
	find /opt/soft/log/ -mtime +30 -name "*.log" -exec rm -rf {} \ ;
	说明：
	将 /opt/soft/log/  目录下的30天前带有  .log 文件删除。具体参数如下：
		 find : linux 下查命令 用户查找指定条件文件。
		 /opt/soft/log/ : 想要进行清理的任意目录
		 -mtime 标准语法写法
		 +30 查找30天前的文件，这里数字代表天数。
		 "*.log" : 希望查找的数据类型，  * 标识查找所有的文件
		 -exec ： 固定写法
		 rm -rf : 强制删除。
		 {} \ ; :固定写法  一对大括号+ 空格+\+;
```

 分配运行权限

```sh
chmod +X  自定删除文件的脚本.sh 
```

使用crontab 加入到系统执行计划

```shell
crontab -e
在计划任务内输入一下内容：

10 0 * * * /opt/soft/log/del-15-days-ago-logs.sh >/dev/null 2>&1
     (这里设置的是每天凌晨0点10分执行del-15-days-ago-logs.sh文件进行数据清理任务,根据自己需求灵活变化清理时间和脚本防止的地址)
```



