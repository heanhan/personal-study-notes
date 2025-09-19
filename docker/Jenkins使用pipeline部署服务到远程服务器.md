**阅读目录**

*   [Docker搭建Jenkins服务](#_label0)
*   [Pipeline脚本部署服务到远程服务器](#_label1)

* * *

写这篇文章是对之前[搭建Jenkins](https://www.cnblogs.com/cfzy/p/14945675.html)做的修改和完善，让jenkins更好的为我们服务

[回到顶部](#_labelTop)

Docker搭建Jenkins服务
-----------------

[![复制代码](//assets.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

使用过程中遇到的问题：  
　　为方便部署，打算将jenkins用到的jdk11、maven3.5.4、gradle6.4工具下载下来（方便以后部署使用），然后挂载到jenkins容器内部  
　　在使用maven打包服务过程中，发现每次构建都要重新下载maven依赖，耗时间耗内存，将jenkins容器内的maven仓库做持久化存储  
　　docker部署jenkins一般是使用jenkins/jenkins:lts-jdk11稳定版镜像，如出现插件下载失败，版本太老，可将容器内jenkins.war替换成最新版  
　　容器时间和宿主机时间同步  
　　docker启动容器限制内存策略。如果不限制会把内存都吃掉。  
　　jenkins构建项目的数据持久化存储  
  
用到的工具安装包和脚本在文章最后，有需要的可以去下载，放到脚本里配置的路径直接运行脚本就可以了  
  
最终的docker搭建jenkins脚本就变成：

[?](#)

<table border="0" cellpadding="0" cellspacing="0"><tbody><tr><td class="gutter"><div class="line number1 index0 alt2">1</div><div class="line number2 index1 alt1">2</div><div class="line number3 index2 alt2">3</div><div class="line number4 index3 alt1">4</div><div class="line number5 index4 alt2">5</div><div class="line number6 index5 alt1">6</div><div class="line number7 index6 alt2">7</div><div class="line number8 index7 alt1">8</div><div class="line number9 index8 alt2">9</div><div class="line number10 index9 alt1">10</div><div class="line number11 index10 alt2">11</div><div class="line number12 index11 alt1">12</div><div class="line number13 index12 alt2">13</div></td><td class="code"><div class="container"><div class="line number1 index0 alt2"><code class="csharp plain">docker run --name jenkins -p 8081:8080 -p 50000:50000 \</code></div><div class="line number2 index1 alt1"><code class="csharp plain">-u root --privileged=</code><code class="csharp keyword">true</code> <code class="csharp plain">-m 1G --memory-swap=3G \</code></div><div class="line number3 index2 alt2"><code class="csharp plain">-v /home/docker/server/jenkins/data:/</code><code class="csharp keyword">var</code><code class="csharp plain">/jenkins_home \</code></div><div class="line number4 index3 alt1"><code class="csharp plain">-v /</code><code class="csharp keyword">var</code><code class="csharp plain">/run/docker.sock:/</code><code class="csharp keyword">var</code><code class="csharp plain">/run/docker.sock \</code></div><div class="line number5 index4 alt2"><code class="csharp plain">-v /usr/bin/docker:/usr/bin/docker \</code></div><div class="line number6 index5 alt1"><code class="csharp plain">-v /home/docker/server/jenkins/m2:/root/.m2 \</code></div><div class="line number7 index6 alt2"><code class="csharp plain">-v /home/docker/server/jenkins/gradle:/root/.gradle \</code></div><div class="line number8 index7 alt1"><code class="csharp plain">-v /home/docker/server/jenkins/pack/apache-maven-3.5.4:/usr/local/maven \</code></div><div class="line number9 index8 alt2"><code class="csharp plain">-v /home/docker/server/jenkins/pack/gradle6.4:/usr/local/gradle \</code></div><div class="line number10 index9 alt1"><code class="csharp plain">-v /home/docker/server/jenkins/pack/jdk1.8.0_171:/usr/local/jdk \</code></div><div class="line number11 index10 alt2"><code class="csharp plain">-v /home/docker/server/jenkins/pack/jdk-11.0.9:/usr/local/jdk11 \</code></div><div class="line number12 index11 alt1"><code class="csharp plain">-v /etc/localtime:/etc/localtime \</code></div><div class="line number13 index12 alt2"><code class="csharp plain">-d jenkins/jenkins:lts-jdk11</code></div></div></td></tr></tbody></table>

　　然后访问jenkins，安装插件：

　　　　Locale　　　　　　　　汉化插件（如汉化不完全，下载此插件）  
　　　　Maven Integration 　构建maven项目插件  
　　　　Publish over SSH    使用ssh免密登录到目标服务器  
　　　　Deploy to container 用于部署war程序到tomcat中  
　　　　git parameter　　　　选择指定分支进行构建的功能  
　　　　pipeline　　　　　　　流水线脚本  
　　　　Pipeline Stage View Plugin　　构建过程图示

　　配置jdk、maven、gradle环境变量

　　配置应用服务器地址及账号：

　　　　系统管理——>系统配置——> Publish over SSH

　　　　![](https://img2022.cnblogs.com/blog/2347845/202208/2347845-20220809110700253-2042276921.png)

[![复制代码](//assets.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

[回到顶部](#_labelTop)

Pipeline脚本部署服务到远程服务器
--------------------

[![复制代码](//assets.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

新建流水线任务
进入任务配置——>参数配置——>git参数，选择代码分支
流水线语法——>选择sshPublisher: Send build artifacts over SSH  
    ![](https://img2022.cnblogs.com/blog/2347845/202208/2347845-20220809170920368-137165534.png)  
点击生成流水线脚本，复制到任务的流水线脚本中

　　![](https://img2022.cnblogs.com/blog/2347845/202208/2347845-20220809171126790-1596731655.png)

流水线脚本以及生成流水线语法的入口，有其他需求也可以用它生成

　　 ______![](https://img2022.cnblogs.com/blog/2347845/202208/2347845-20220809171510126-2012550617.png)______

[![复制代码](//assets.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

完整的流水线脚本如下：定义变量

[?](#)

<table border="0" cellpadding="0" cellspacing="0"><tbody><tr><td class="gutter"><div class="line number1 index0 alt2">1</div><div class="line number2 index1 alt1">2</div><div class="line number3 index2 alt2">3</div><div class="line number4 index3 alt1">4</div><div class="line number5 index4 alt2">5</div><div class="line number6 index5 alt1">6</div><div class="line number7 index6 alt2">7</div><div class="line number8 index7 alt1">8</div><div class="line number9 index8 alt2">9</div><div class="line number10 index9 alt1">10</div><div class="line number11 index10 alt2">11</div><div class="line number12 index11 alt1">12</div><div class="line number13 index12 alt2">13</div><div class="line number14 index13 alt1">14</div><div class="line number15 index14 alt2">15</div><div class="line number16 index15 alt1">16</div><div class="line number17 index16 alt2">17</div><div class="line number18 index17 alt1">18</div><div class="line number19 index18 alt2">19</div><div class="line number20 index19 alt1">20</div><div class="line number21 index20 alt2">21</div><div class="line number22 index21 alt1">22</div><div class="line number23 index22 alt2">23</div><div class="line number24 index23 alt1">24</div><div class="line number25 index24 alt2">25</div><div class="line number26 index25 alt1">26</div><div class="line number27 index26 alt2">27</div><div class="line number28 index27 alt1">28</div><div class="line number29 index28 alt2">29</div><div class="line number30 index29 alt1">30</div><div class="line number31 index30 alt2">31</div><div class="line number32 index31 alt1">32</div><div class="line number33 index32 alt2">33</div><div class="line number34 index33 alt1">34</div><div class="line number35 index34 alt2">35</div><div class="line number36 index35 alt1">36</div><div class="line number37 index36 alt2">37</div><div class="line number38 index37 alt1">38</div><div class="line number39 index38 alt2">39</div><div class="line number40 index39 alt1">40</div><div class="line number41 index40 alt2">41</div><div class="line number42 index41 alt1">42</div><div class="line number43 index42 alt2">43</div><div class="line number44 index43 alt1">44</div><div class="line number45 index44 alt2">45</div><div class="line number46 index45 alt1">46</div><div class="line number47 index46 alt2">47</div><div class="line number48 index47 alt1">48</div><div class="line number49 index48 alt2">49</div></td><td class="code"><div class="container"><div class="line number1 index0 alt2"><code class="python comments">#!/usr/bin/env groovy</code></div><div class="line number2 index1 alt1">&nbsp;</div><div class="line number3 index2 alt2"><code class="python keyword">def</code> <code class="python plain">desc_ip </code><code class="python keyword">=</code> <code class="python string">"192.168.29.22"</code></div><div class="line number4 index3 alt1"><code class="python keyword">def</code> <code class="python plain">desc_path </code><code class="python keyword">=</code> <code class="python string">"/server/temp"</code></div><div class="line number5 index4 alt2"><code class="python keyword">def</code> <code class="python plain">app_name </code><code class="python keyword">=</code> <code class="python string">"test"</code></div><div class="line number6 index5 alt1"><code class="python keyword">def</code> <code class="python plain">app_file </code><code class="python keyword">=</code> <code class="python string">"${app_name}-0.0.1-SNAPSHOT.jar"</code></div><div class="line number7 index6 alt2"><code class="python keyword">def</code> <code class="python plain">install_path </code><code class="python keyword">=</code> <code class="python string">"/server/${app_name}"</code></div><div class="line number8 index7 alt1"><code class="python keyword">def</code> <code class="python plain">target_path </code><code class="python keyword">=</code> <code class="python string">"target/"</code></div><div class="line number9 index8 alt2"><code class="python keyword">def</code> <code class="python plain">target_file </code><code class="python keyword">=</code> <code class="python string">"target/${app_file}"</code></div><div class="line number10 index9 alt1"><code class="python keyword">def</code> <code class="python plain">log_file </code><code class="python keyword">=</code> <code class="python string">"${app_name}.log"</code></div><div class="line number11 index10 alt2"><code class="python keyword">def</code> <code class="python plain">git_address </code><code class="python keyword">=</code> <code class="python string">"https://git.test.com/test/test.git"</code></div><div class="line number12 index11 alt1"><code class="python keyword">def</code> <code class="python plain">git_auth </code><code class="python keyword">=</code> <code class="python string">"fe896e04-3743-4931-83f8-32d716461388"</code></div><div class="line number13 index12 alt2"><code class="python keyword">def</code> <code class="python plain">JAVA_OPTS </code><code class="python keyword">=</code> <code class="python string">"-Xms128m -Xmx256m -Xmn64m -Dfile.encoding=UTF8 -Dspring.profiles.active=test"</code></div><div class="line number14 index13 alt1"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code>&nbsp;</div><div class="line number15 index14 alt2"><code class="python plain">pipeline {</code></div><div class="line number16 index15 alt1"><code class="python spaces">&nbsp;&nbsp;</code><code class="python plain">agent </code><code class="python functions">any</code></div><div class="line number17 index16 alt2">&nbsp;</div><div class="line number18 index17 alt1"><code class="python spaces">&nbsp;&nbsp;</code><code class="python plain">stages {</code></div><div class="line number19 index18 alt2"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python plain">stage(</code><code class="python string">'拉取代码'</code><code class="python plain">) {</code></div><div class="line number20 index19 alt1"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python plain">steps {</code></div><div class="line number21 index20 alt2"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python plain">checkout([$</code><code class="python keyword">class</code><code class="python plain">: </code><code class="python string">'GitSCM'</code><code class="python plain">, branches: [[name: </code><code class="python string">'${branch}'</code><code class="python plain">]], userRemoteConfigs: [[credentialsId: </code><code class="python string">"${git_auth}"</code><code class="python plain">, url: </code><code class="python string">"${git_address}"</code><code class="python plain">]]])</code></div><div class="line number22 index21 alt1"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python plain">}</code></div><div class="line number23 index22 alt2"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python plain">}</code></div><div class="line number24 index23 alt1"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code>&nbsp;</div><div class="line number25 index24 alt2"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python plain">stage(</code><code class="python string">'代码编译'</code><code class="python plain">) {</code></div><div class="line number26 index25 alt1"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python plain">steps {</code></div><div class="line number27 index26 alt2"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python plain">sh&nbsp; </code><code class="python comments">"""</code></div><div class="line number28 index27 alt1"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python comments">pwd</code></div><div class="line number29 index28 alt2"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python comments">JAVA_HOME=/usr/local/jdk</code></div><div class="line number30 index29 alt1"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python comments">PATH=$JAVA_HOME/bin:/usr/local/maven/bin:$PATH</code></div><div class="line number31 index30 alt2"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python comments">mvn clean package -Dmaven.test.skip=true</code></div><div class="line number32 index31 alt1"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python comments">"""</code></div><div class="line number33 index32 alt2"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python plain">}</code></div><div class="line number34 index33 alt1"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python plain">}</code></div><div class="line number35 index34 alt2"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code>&nbsp;</div><div class="line number36 index35 alt1"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python plain">stage(</code><code class="python string">'远程启动服务'</code><code class="python plain">) {</code></div><div class="line number37 index36 alt2"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python plain">steps {</code></div><div class="line number38 index37 alt1"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python plain">sshPublisher(publishers: [sshPublisherDesc(configName: </code><code class="python string">"${desc_ip}"</code><code class="python plain">, transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand:</code></div><div class="line number39 index38 alt2"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python comments">"""</code></div><div class="line number40 index39 alt1"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python comments">cd ${install_path}</code></div><div class="line number41 index40 alt2"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python comments">pwd</code></div><div class="line number42 index41 alt1"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python comments">ps -ef | grep test-0.0.1-SNAPSHOT.jar | grep -v grep |awk '{print \$2}' |xargs kill -9</code></div><div class="line number43 index42 alt2"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python comments">mv ${desc_path}/${app_file} ./</code></div><div class="line number44 index43 alt1"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python comments">java -jar ${JAVA_OPTS} ${app_file} &gt;${log_file} &amp;</code></div><div class="line number45 index44 alt2"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python comments">"""</code><code class="python plain">, execTimeout: </code><code class="python value">120000</code><code class="python plain">, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: </code><code class="python string">'[, ]+'</code><code class="python plain">, remoteDirectory: "${desc_path}</code><code class="python string">", remoteDirectorySDF: false, removePrefix: "</code><code class="python plain">${target_path}</code><code class="python string">", sourceFiles: "</code><code class="python plain">${target_file}")], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: true)])</code></div><div class="line number46 index45 alt1"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python plain">}</code></div><div class="line number47 index46 alt2"><code class="python spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="python plain">}</code></div><div class="line number48 index47 alt1"><code class="python spaces">&nbsp;&nbsp;</code><code class="python plain">}</code></div><div class="line number49 index48 alt2"><code class="python plain">}　</code></div></div></td></tr></tbody></table>

pipeline脚本需要注意：变量的使用需要加${}，如果是在pipeline语法中生成的语句，变量的使用需要加双引号"${}"

　　脚本写完后会提示哪里有错误需要怎么改，要注意特殊字符需要转义处理，如'{print \\$2}'需要转义

构建过程中遇到的错误：

[![复制代码](//assets.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

1、SSH出错：IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man\-in\-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ECDSA key sent by the remote host is
SHA256:uLW/iHik7jxKZ6IRgRV7pfWAKuBgZGxInXba1aSb8hA.
Please contact your system administrator.
Add correct host key in /home/docker/.ssh/known\_hosts to get rid of this message.
Offending ECDSA key in /home/docker/.ssh/known\_hosts:3
ECDSA host key for 192.168.29.22 has changed and you have requested strict checking.
Host key verification failed.  
  
原因：控制端保存的被控制端秘钥改变，导致SSH错误  
解决方案：需要删除控制端保存的秘钥，然后重新SSH登录  
　　　　cat ~/.ssh/know\_hosts  
　　　　删除文件中对应的主机和秘钥记录  
　　　　ssh 192.168.29.22     
　　　　输入密码就可以了  
  

2、publish over ssh传输文件数为0  
　　SSH: Connecting from host \[192.168.29.22\]  
　　SSH: Connecting with configuration \[192.168.29.22\] ...  
　　SSH: Disconnecting configuration \[192.168.29.22\] ...  
　　SSH: Transferred 0 file(s)  
　　Build step 'Send files or execute commands over SSH' changed build result to  SUCCESS  
　　Finished: SUCCESS  
  原因：源文件的位置没写对

   　　![](https://img2022.cnblogs.com/blog/2347845/202208/2347845-20220809150922474-1200263563.png)

  解决：可以在构建日志里看到jenkins运行的位置和jar包位置

 　　  ![](https://img2022.cnblogs.com/blog/2347845/202208/2347845-20220809172939406-76325414.png)

　　  ![](https://img2022.cnblogs.com/blog/2347845/202208/2347845-20220809172849184-89411793.png)

　　因为jenkins运行位置是/var/jenkins\_home/workspaces/test，所以源文件jar包的位置应该写target/test-0.0.1-SNAPSHOP.jar

[![复制代码](//assets.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

Jenkins安装包下载地址：

　　[链接：https://pan.baidu.com/s/1YjIozZxM4FL8\_vw-rCLplg](https://pan.baidu.com/s/1YjIozZxM4FL8_vw-rCLplg?pwd=wxfa)  
　　提取码：wxfa

原文链接：https://www.cnblogs.com/cfzy/p/16562925.html

本文转自 <https://www.cnblogs.com/cfzy/p/16562925.html>，如有侵权，请联系删除。