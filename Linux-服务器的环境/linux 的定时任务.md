# linux 的定时任务

**我们用crontab -e进入当前用户的工作表编辑，是常见的vim界面。每行是一条命令**



检查是否安装了crontab

rpm -qa | grep crontab

出现例如：

crontabs-1.11-6.20121102git.el7.noarch

即已经安装

如果没有安装,执行：

yum  install vixie-cron  crontabs

ps:

vixie-cron 是 cron 的主程序；

crontabs 是用来安装、卸装、或列举用来驱动 cron 守护进程的表格的程序。

启动和配置服务：

systemctl start crond     //启动服务

systemctl stop crond    //关闭服务

systemctl restart crond  //重启服务

systemctl reload crond   //重新载入配置

systemctl status crond    //查看crontab服务状态



