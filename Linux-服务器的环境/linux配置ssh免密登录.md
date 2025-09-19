**linux配置ssh免密登录**

先检查下是否已安装ssh

```shell
systemctl status sshd
```

已安装的会正常显示ssh服务的状态，否则，执行下面命令安装

```
# 安装过程一路按y即可，或-y
yum install openssh-clients
yum install openssh-server
```

测试是否可用

```
#按提示确认连接后，输入当前用户密码即可，初始没有密码会提示创建密码
ssh localhost
```

设置无密码登录

```
#~ 代表的是用户的主文件夹，即 “/home/用户名” 这个目录，如你的用户名为 hadoop，则 ~ 就代表 “/home/hadoop/”
cd ~/.ssh/                     # 若没有该目录，请先执行一次ssh localhost
ssh-keygen -t rsa              # 会有提示，都按回车就可以
cat id_rsa.pub >> authorized_keys  # 加入授权
chmod 600 ./authorized_keys    # 修改文件权限
```

此时再用 ssh localhost 命令，无需输入密码就可以直接登陆了.

相关介绍：

~/.ssh/known_hosts	：记录ssh访问过计算机的公钥(public key)

id_rsa	：生成的私钥

id_rsa.pub	：生成的公钥

authorized_keys	：存放授权过得无秘登录服务器公钥



如果是两个服务器之间免密，则把A服务器的公钥复制到B服务器，在B服务器加入authorized_keys，则此时

A可以免密连接B服务器。

大致流程：

1.在A上生成公钥私钥。

2.将公钥拷贝给server B，加入authorized_keys

\3. A向 B发送一个连接请求。

4.B得到 A的信息后，在authorized_key中查找，如果有相应的用户名和IP，则随机生成一个字符串，并用A的公钥加密，发送给A。

5.A得到B发来的消息后，使用私钥进行解密，然后将解密后的字符串发送给B。B进行和生成的对比，如果一致，则允许免登录。



两种操作：

第一种:

把A的公钥文件复制到B的.ssh目录下，加入授权。

第二种：

执行ssh-copy-id命令，然后B有密码，则按照提示输入密码即可。

```
ssh-copy-id -i id_rsa.pub hadoop@192.168.0.141
```

