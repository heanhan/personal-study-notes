使用mysql bin目录下的二进制 mysqlbinlog命令进行查看

现将binlog日志进行解析  找到存放binlog的日志的地方，然后使用下面的命令，将binlog日志转换成sql语句

 mysqlbinlog --no-defaults  --base64-output=decode-rows -v  binlog日志的文件   --result-file=mysql0003.sql