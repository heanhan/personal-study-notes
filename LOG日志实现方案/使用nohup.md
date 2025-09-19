### 使用 nohup 的命令

一般微服务的启动脚本内容分如下：

```sh
nohup java -jar service-name.jar --config application-prod.yml | /usr/sbin/cronolog ./log/service-name-%Y-%m-%d.out >> /dev/null 2>&1 & 
```

nohup命令会有一个重定向的日志输出设置：

*  /dev/null - 表示空设备文件
* 0  -stdin(标准输入)
* 1  -stdout(标准输出)
* 2 -stderr (标准错误输出)
* 2>&1 表示将标准输错误2 重定向到标准输出 &1 ,标准输出 &1再被重定向到 具体的文件中。
*  & 放在命令的结尾，表示后台运行，防止终端一直被某个进程占用，这样终端可以执行别的任务，配合 >file 2>&1 可以将log 保存到某个文件中，如果终端关闭则进程也关闭，将nohup 放到命令的开头表示不挂起，关闭终端或者退出某个账号进程也继续保持运行状态。