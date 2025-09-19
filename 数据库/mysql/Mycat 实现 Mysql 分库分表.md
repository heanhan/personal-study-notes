## 实现Mycat 对 mysql 的分别的分库分表。

### 1、使用 mycat 的前提

```
1.1、mysql 至少两个，已经进行主从搭建。
1.2 jdk 环境已经安装。
```

### 2、安装 

```
2.1 官网下载 mycat 的安装包，
2.2 解压 即可看使用。
tar -zxvf Mycat-server-1.6.7.1-release-20200209222254-linux.tar.gz
解压后得到   mycat 的文件夹。
```

### 3、创建一个新的 mysql 用户

```
需要创建一个新的 mysql 的用户用来连接 mycat,以下就是创建用户的流程
//创建 mycat 的用户 mycat  和密码 mycat
create USER 'mycat'@'%'IDENTIFIED BY 'mycat'
//修改密码
ALTER USER 'mycat'@'%'IDENTIFIED WITH mysql_native_password BY '123456';
//刷新权限
FLUSH PRIVILEGES;

```

### 4、配置 mycat

4.1 配置一：server.xml
此处使用上边创建的新用户mycat ，可以管理的逻辑库 mycat_order ,对应的 schema.xml中的<schema name="mydatabase"

框出来的看一下：

第一行：name 值后面上是边创建 MYSQL 用户，第二行：是 mycat 用户的密码，第三行：是数据库

![在这里插入图片描述](https://img-blog.csdnimg.cn/202008152147292.png)

4.2 配置项二：schema.xml

这个文件主要修改连接其他数据库的俩个节点

使用规则是mod-long这个需要注意一下子

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200816115712264.png)

4.3 配置项三：rule.xml

这里是order_id使用mod-long规则

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200816120116885.png)

这个修改就是你有几个节点就写多少即可

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200816120316818.png)