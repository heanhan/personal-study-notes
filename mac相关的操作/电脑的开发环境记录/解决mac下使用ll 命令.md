#### mac -bash: ll: command not found 解决方案

```sh

vim ~/.bashrc
#添加下面内容
alias ll='ls -l'
#保存退出后 执行命令生效
source ~/.bashrc
```

