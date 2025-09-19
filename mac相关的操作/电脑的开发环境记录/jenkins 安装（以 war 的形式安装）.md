### jenkins 的安装（jenkins.war 形式）



准备

```
下载地址：
wget http://mirrors.jenkins.io/war-stable/latest/jenkins.war
```

2、安装启动

```
1.1	执行 命令
nohup java -jar jenkins.war & 
后台便启动 Jenkins 服务。因为 jenkins.war 内置 Jetty 服务器，所以无需丢到 Tomcat 等等容器下，可以直接进行启动。
执行完成后，执行如下命令，控制台查看打印日志
tail -f nohup.out 
查看下启动日志。如果看到如下日志，说明启动成功：

Please use the following password to proceed to installation:

88528fe8848e4110b5ae8ce5643e18f6   （我的机器生成的密码）

This may also be found at: /root/.jenkins/secrets/initialAdminPassword
```

