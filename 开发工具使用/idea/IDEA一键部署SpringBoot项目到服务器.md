1\. 安装Alibaba Cloud Toolkit插件
-----------------------------

![image-20220223091930632](https://img-blog.csdnimg.cn/img_convert/a80cf4f4d98332c55e1f09f814ae0d6e.png)

2\. 配置部署环境
----------

![image-20220223091431544](https://img-blog.csdnimg.cn/img_convert/af7c9101d5d9578196e9c7060714a68b.png)

![image-20220223092525004](https://img-blog.csdnimg.cn/img_convert/f2dd7455213a03f762b69ca1ae739b3c.png)

![image-20220223093036693](https://img-blog.csdnimg.cn/img_convert/94c9710df4f2233d2e7095463edd02f2.png)

### 2.1 为本次部署设置一个名字

### 2.2 选择被部署文件的生成方式

IDEA提供了三种方式：**Maven Build**，**Upload File**，**Gradle Build**，虽然我的SpringBoot项目使用的是Maven构建工具，但是我一般情况下选择**Upload File**的方式。因为我的项目是多模块项目，选择**Maven Build**方式的话IDEA并不知道需要上传的是哪个jar包（因为在每个模块下都会生成自己的jar包）。

使用**Upload File**特别需要注意的一点是，我们需要在自动部署之前先手动打个jar包，这样我们才能选择我们想上传的jar包，这一步并不意味着我们会上传刚刚手动打包的文件，只是告诉IDEA以后上传的文件的目录和名称而已。

手动打包的方式

![image-20220223093826474](https://img-blog.csdnimg.cn/img_convert/aa386c9f5fd4dca68d2c0014d9d40047.png)

然后选择你想上传的jar包即可，如下图

![image-20220223094027081](https://img-blog.csdnimg.cn/img_convert/690b74e85db57b5a12824e4cfff2c969.png)

### 2.3 选择目标服务器

#### 2.3.1 配置过了？直接选择

如果你之前配置过远程服务器的信息，直接选择即可，跳过配置的步骤；

![image-20220223102802103](https://img-blog.csdnimg.cn/img_convert/2d2002f39b42aa6effc4d661bf138cf5.png)

如果没有配置，那你需要先配置一下

#### 2.3.2 没配置过？那就配置服务器

![image-20220223102044544](https://img-blog.csdnimg.cn/img_convert/5f7abdaeb32a07a5beeaa818e100cab6.png)

点击左下角的**Manage Host**按钮，此时应该弹出如下界面，如果没有弹出，找到下图中的按钮点击即可

![image-20220223102138973](https://img-blog.csdnimg.cn/img_convert/53b798eb3642410a68af37b9d2d460f6.png)

![image-20220223102215420](https://img-blog.csdnimg.cn/img_convert/a6816e0e4a4f9bc1792f68b4e0eb49e8.png)

点击**Add Host**按钮，填写你的主机信息

![image-20220223102438592](https://img-blog.csdnimg.cn/img_convert/b6f6667b767162b11b9c7e45ffb98596.png)

其中，验证方式有两种

*   **Password**：就是通过密码校验你的身份
*   **Select a Private Key**：通过本地密钥文件验证你的身份

填写完之后，点击测试链接状况，查看是否链接成功，成功的话点击添加按钮；否则检查配置信息直到添加成功为止。

配置完服务器信息你就能选择你的主机了，如下图所示，选中它，然后点击**Select**即可

### 2.4 填写文件传输的目标目录（Target Directory）

就是说你想把jar放在服务器的哪个目录下

### 2.5 配置After deploy

从名字看出来，这是让我们设置deploy之后的动作，IDEA理解的deploy只是把你要上传的文件传到服务器上而已。

接下来点击**Select Command**按钮，选择你要运行的命令，如果你之前配置过，那就选择就好了；没配置过的话，点击下图中的按钮，填写你想执行的指令。这里的指令其实就是你在终端中运行的指令，比如执行一个脚本文件，或者执行一些linux内置的命令等等

![image-20220223103249032](https://img-blog.csdnimg.cn/img_convert/41d4c295484afb952d1e200f65d6a01b.png)

我个人的习惯是在部署的文件夹下配置启动脚本，`start.sh`和`stop.sh`

```bash
# start.sh
nohup java -jar zh-sensor-protocol.jar >/dev/null 2>&1 &
echo "服务启动成功"
```

```bash
# stop.sh
PID=$(ps -ef | grep zh-sensor-protocol.jar | grep -v grep | awk '{ print $2 }')
if [ -z "$PID" ]
then
echo Application is already stopped
else
echo kill -9 $PID
kill -9 $PID
fi
```

如此一来，我会在IDEA中配置如下命令

![image-20220223105040771](https://img-blog.csdnimg.cn/img_convert/062fee23f8efb9f17ccbf26d77decac1.png)

### 2.6 Before launch

这一步指的是在部署动作正式启动之前，你想执行什么操作。还记得之前我们选择的上传的文件吗，这一步是得到那个文件的关键了。

我们点击**+**按钮，选择**Run Maven Goal**选项

![image-20220223105404359](https://img-blog.csdnimg.cn/img_convert/6babc41c9df35a65cc05696210d92f55.png)

然后配置如下信息，因为我们是部署Spring Boot项目所以才选择的Maven选项，其他项目部署灵活选择即可。

![image-20220223110339773](https://img-blog.csdnimg.cn/img_convert/3e13ae4bee16e8c69540fa4c91ab75c3.png)

到此为止，我们就已经配置完了，接下来就行部署。

3\. 开始部署
--------

![image-20220223105911543](https://img-blog.csdnimg.cn/img_convert/9430e6b4d1007625c3db0df18204e8b5.png)

部署结果

![image-20220223110432296](https://img-blog.csdnimg.cn/img_convert/56849f64f7674d918e88c950ed0eb6de.png)  
大功告成！

* * *

完。

本文转自 <https://www.cnblogs.com/chanmufeng/p/15926928.html>，如有侵权，请联系删除。