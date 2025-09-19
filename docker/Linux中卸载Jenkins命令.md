# Linux中卸载Jenkins命令

1、停止服务并yum卸载

```

service jenkins stop
 
yum clean all
 
yum -y remove jenkins
```

2、rpm卸载

```
rpm -e jenkins
```

3、检查是否卸载成功

```
rpm -ql jenkins 
```

4、彻底删除残留文件：

```

find / -iname jenkins | xargs -n 1000 rm -rf
 
# find命令用来查找“/”下名称符合jenkins的文件
#   -name name, -iname name : 文件名称符合 name 的文件。iname 会忽略大小写
 
# xargs 命令 可以将管道或标准输入（stdin）数据转换成命令行参数，也能够从文件的输出中读取数据。
#   -n 选项 每次传递几个参数给其后面的命令执行
```

