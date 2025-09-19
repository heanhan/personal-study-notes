前言：
===

nginx作为当今火爆的、高性能的http及反向代理服务，不管前端还是后端，都需要全面去了解，学习，实操。一句话：搞懂nginx如何使用以及工作逻辑对于程序员来说是必不可少的！

我们看看本文的大纲 先了解一下本文都讲了哪些东西，大纲如下：

1.  nginx介绍
2.  nginx安装
3.  nginx目录一览
4.  nginx.conf文件解读
5.  location路由匹配规则
6.  反向代理
7.  负载均衡
8.  动静分离
9.  跨域
10.  缓存
11.  黑白名单
12.  nginx限流
13.  https配置
14.  压缩
15.  其他一些常用指令与说明
16.  重试策略
17.  最后总结

**一些说明：**

1.  系统： centos7
2.  本文使用nginx版本：nginx/1.24.0
3.  关于nginx如何安装（本文不再赘述），参考之前我的一篇文章：[Centos7中安装nginx](https://juejin.cn/post/7293176193322729510 "https://juejin.cn/post/7293176193322729510")
4.  在学习nginx前，最好需要知道或者了解事件驱动思想以及几种常见多路复用I/O模型和Reactor模式，这样你才能从底层 更深刻的理解nginx的架构设计

1、nginx 介绍
==========

为了有一个全面的认知，接下来我们先来看看nginx的架构以及一些特点。

1.1、nginx 特点
------------

1.  处理响应请求快（异步非阻塞I/O，零拷贝，mmap，缓存机制）
2.  扩展性好（模块化设计）
3.  内存消耗低（异步非阻塞，多阶段处理）
4.  具有很高的可靠性（无数次的生产验证，很多头部公司都在用）
5.  热部署
6.  高并发连接（事件驱动模型，多进程机制）
7.  自由的BSD许可协议（可以自己修改代码后发布，包容性极强）

1.2、nginx 架构
------------

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2db0016998d447f1bf0c657454180238~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1986&h=1182&s=483033&e=png&b=f4f4f4) 从上边这张图，我们可以一览nginx的架构设计，首先我们可以直观得出`nginx的几大特点：`

1.  **事件驱动&异步非阻塞：**
    
    > 本质来说，**事件驱动是一种思想（事实上它不仅仅局限于编程）** ，事件驱动思想是实现 **异步非阻塞特性** 的一个重要手段。对于web服务器来说，造成性能拉胯不支持高并发的常见原因就是由于使用了传统的I/O模型造成在`内核没有可读/可写事件（或者说没有数据可供用户进程读写）时`，**用户线程** `一直在等待`（其他事情啥也干不了就是干等等待内核上的数据可读/可写），这样的话其实是一个线程（ps:线程在Linux系统也是进程）对应一个请求，请求是无限的，而线程是有限的从而也就形成了并发瓶颈。而大佬们为了解决此类问题，运用了事件驱动思想来对传统I/O模型做个改造，即在客户端发起请求后，用户线程`不再阻塞等待内核数据就绪`，而是`立即返回`（可以去执行其他业务逻辑或者继续处理其他请求）。当内核的I/O操作完成后，`内核系统`会向用户线程`发送一个事件通知`，用户线程才来处理这个读/写操作，之后拿到数据再做些其他业务后响应给客户端，从而完成一次客户端请求的处理。事件驱动的I/O模型中，程序不必阻塞等待I/O操作的完成，也无需为每个请求创建一个线程，从而提高了系统的并发处理能力和响应速度。`事件驱动型的I/O模型通常也被被称为I/O多路复用`，即这种模型可以在一个线程中，处理多个连接（复用就是指多个连接复用一个线程，多路也即所谓的 多个连接），通过这种方式避免了线程间切换的开销，同时也使得用户线程不再被阻塞，提高了系统的性能和可靠性。nginx支持事件驱动是因为他利用了操作系统提供的I/O多路复用接口，如Linux系统中，常用的I/O多路复用接口有select/poll，epoll。这些接口可以监视多个文件描述符的状态变化，当文件描述符可读或可写时，就会向用户线程发送一个事件通知。用户线程通过事件处理机制（读取/写入数据）来处理这个事件，之后进行对应的业务逻辑完了进行响应。**简单一句话概括：** `事件驱动机制就是指当有读/写/连接事件就绪时 再去做读/写/接受连接这些事情，而不是一直在那里傻傻的等，也正应了他的名词： 【事件驱动！】，基于事件驱动思想设计的多路复用I/O（如select/poll，epoll），相对于传统I/O模型，达到了异步非阻塞的效果！`
    > 
    > 既然提到了select/poll,epoll 那么我们就简单说一下（注意我这里是简单描述，后续有时间会对相关知识点从源码层面做个系统的整理和图解）：
    > 
    > **select：** 将已连接的 Socket 都放到一个文件描述符集合，然后用户态调用 select 函数将文件描述符集合拷贝到内核里，让内核来检查是否有网络事件产生，检查的方式很粗暴，就是通过遍历文件描述符集合的方式，当检查到有事件产生后，将此 Socket 标记为可读或可写， 接着再把整个文件描述符集合拷贝回用户态里，然后用户态还需要再通过遍历的方法找到可读或可写的 Socket，然后再对其处理。
    > 
    > **poll：** poll函数的话其实和select大差不差，唯一区别可能就是socket列表的结构有所不同，不再受FD\_SETSIZE的限制。这里就不多说了。
    > 
    > **epoll：** epoll在前边两者的基础上做了很大的优化，select/poll都需要遍历整个socket列表，当检测到传入的socket可读/可写时，则copy socket列表给用户空间，用户态仍然需要遍历（因为内核copy给用户态的是整个socket列表） ，而epoll则是通过红黑树结构将需要监控的socket插入到进去，然后当有socket可读时会通过回调机制来将其添加到可读列表中，然后内核将可读列表copy给用户态即可(据说此处使用了mmap这里我们不去验证探究，后续写相关文章时在深究吧)，整个过程少了无效的遍历以及不用copy整个socket集合。
    
2.  **多进程机制：**
    
    > 另外可以得知nginx有两种类型的进程，一种是Master主进程，一种是Worker工作进程。主进程主要负责3项工作：`加载配置`、`启动工作进程`及`非停升级`。另外work进程是主进程启动后，fork而来的。假设 Nginx fork了多个(具体在于你的配置)Worker进程，并且在Master进程中通过 socket 套接字监听（listen）80端口。然后每个worker进程都可以去 accept 这个监听的 socket。 当一个连接进来后，所有Worker进程，都会收到消息，但是只有一个Worker进程可以 accept 这个连接，其它的则 accept 失败，Nginx 保证只有一个Worker去accept的方式就是加锁（accept\_mutex）。有了锁之后，在同一时刻，就只会有一个Worker进程去 accpet 连接，在 Worker 进程拿到 Http 请求后，就开始按照worker进程内的预置模块去处理该 Http 请求，最后返回响应结果并断开连接。其实如果熟悉reactor模型你会发现，nginx的设计有reactor的影子，只不过reactor的主reactor是会负责accept的，而nginx的主进程（对应主reactor） 是不会去accept的，而是交给了worker进程来处理。
    > 
    > worker进程除了accept连接之外，还会执行：网络读写、存储读写、内容传输、以及请求分发等等。而其代码的模块化设计，也使得我们可以根据需要对功能模块 进行适当的选择和修改，编译成符合特定需要/业务的服务器
    
3.  **proxy cache（服务端缓存）：**
    
    > proxy cache主要实现 nginx 服务器对客户端数据请求的快速响应。nginx 服务器在接收到被代理服务器的响应数据之后，一方面将数据传递给客户端，另一方面根据proxy cache的配置将这些数据缓存到本地硬盘上。当客户端再次访问相同的数据时，nginx服务器直接从硬盘检索到相应的数据返回给用户，从而减少与被代理服务器交互的时间。在缓存数据时，运用了零拷贝以及mmap技术，使得数据copy性能大幅提升。
    
4.  **反向代理：**
    
    > nginx的强大之处其中一个就是他的反向代理，通过反向代理，可以隐藏真正的服务，增加其安全性，同时便于统一管理处理请求，另外可以很容易的做个负载均衡，更好的面对高并发的场景。
    

1.3、nginx模块
-----------

> nginx服务器由n多个模块组成，每个模块就是一个功能，某个模块只负责自身的功能，所以说对于 **`“高内聚，低耦合“`** 的编程规则，在`nginx`身上可谓`体现的淋漓尽致`。

**nginx模块示意图如下：** ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9abd5af68a4f4581b34e14af6810e3a7~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1306&h=928&s=154272&e=png&b=f8f6f2)

*   **核心模块** ：是nginx 服务器正常运行必不可少的模块，提供错误日志记录、配置文件解析、事件驱动 机制、进程管理等核心功能
*   **标准HTTP模块** ：提供 HTTP 协议解析相关的功能，如：端口配置、网页编码设置、HTTP 响应头设 置等
*   **可选HTTP模块** ：主要用于扩展标准的 HTTP 功能，让nginx能处理一些特殊的服务，如：Flash 多 媒体传输、解析 GeoIP 请求、SSL 支持等
*   **邮件服务模块** ：主要用于支持 nginx 的邮件服务，包括对 POP3 协议、IMAP 协议和 SMTP 协议的支持
*   **第三方模块** ：是为了扩展 Nginx 服务器应用，完成开发者自定义功能，如：Json 支持、Lua 支持等

1.4、nginx常见应用场景
---------------

nginx常用场景挺多的，比如：

*   反向代理
*   负载均衡
*   缓存
*   限流
*   黑/白名单
*   静态资源服务
*   动静分离
*   防盗链
*   跨域
*   高可用
*   .......

其中我认为 **最最** 基础的也是应用最多的就是 **反向代理**，这里我们画个图简单看下什么是反向代理 （ps：其他的那些使用场景，我们先不做展开，放到下边一个个哔哔。）

所谓反向代理，其实很好理解就是代理的服务端（与之对应的正向代理一般代理的是客户端），**nginx反向代理如下示意：** ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/12feb26340c74998bd7917f1a21d83fa~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1906&h=1212&s=132766&e=png&b=fdfdfd)

* * *

**好了介绍了这么多，想必到这里应该对nginx有个大体的了解了吧，接下来我们安装并一个一个的分析介绍nginx的知识点。**

2、nginx安装
=========

关于nginx的安装在这里不再赘述，参考我之前的一篇文章：[Centos7中安装nginx](https://juejin.cn/post/7293176193322729510 "https://juejin.cn/post/7293176193322729510")

3、nginx目录一览
===========

我们使用 tree /usr/local/nginx/ -L 2 命令查看一下nginx的目录，对其结构有个初步的认识：

```shell
[root@localhost /]# tree /opt/nginx/ -L 2
/usr/local/nginx/
├── conf                        #存放一系列配置文件的目录
│   ├── fastcgi.conf           #fastcgi程序相关配置文件
│   ├── fastcgi.conf.default   #fastcgi程序相关配置文件备份
│   ├── fastcgi_params         #fastcgi程序参数文件
│   ├── fastcgi_params.default #fastcgi程序参数文件备份
│   ├── koi-utf           #编码映射文件
│   ├── koi-win           #编码映射文件
│   ├── mime.types        #媒体类型控制文件
│   ├── mime.types.default#媒体类型控制文件备份
│   ├── nginx.conf        #主配置文件
│   ├── nginx.conf.default#主配置文件备份
│   ├── scgi_params      #scgi程序相关配置文件
│   ├── scgi_params.default #scgi程序相关配置文件备份
│   ├── uwsgi_params       #uwsgi程序相关配置文件
│   ├── uwsgi_params.default#uwsgi程序相关配置文件备份
│   └── win-utf          #编码映射文件
├── html                 #存放网页文档
│   ├── 50x.html         #错误页码显示网页文件
│   └── index.html       #网页的首页文件
├── sbin                #存放启动程序
│   ├── nginx           #nginx启动程序

3 directories, 18 files
```

从输出可以看到nginx分的很清晰，有配置目录，html目录，log目录，启动程序目录。

*   **关于目录的一点小说明：**
    
    上边的仅仅是nginx的主目录，事实上，生效的主配置文件一定是/usr/local/nginx/conf.conf ？这不一定，而是取决于你启动nginx时候有没有指定nginx.conf，实际使用中我发现我机器上有好几个地方都存在nginx.conf文件，使用 locate nginx.conf看一下 如下图所示： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/26191e37310e42e4ae8e166927959eaf~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=904&h=186&s=29866&e=png&b=010101) 那如何确定nginx当前生效的是哪个nginx.conf呢，很简单使用nginx -T命令即可查看当前生效的nginx.conf，如下：可以看到我当前生效的是 /etc/nginx/nginx.conf这个文件（我是使用的 systemctl start nginx.service命令启动的，未指定用哪个文件启动，所以可以看出默认使用的是 /etc/nginx/nginx.conf这个配置文件） ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7cd376f7eec1444298216905fb5d2669~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1116&h=196&s=26459&e=png&b=010101) 另外还有一个就是nginx的日志，我发现我的nginx日志就不是在 /usr/local/nginx/logs/这个目录下，而是放到了/var/log/nginx/ 这个目录下了（ps：log文件的存放和我的nginx.conf文件中的 access\_log配置有关系）。如下演示： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fa46cd957f224031bb05950196630777~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3316&h=1624&s=335118&e=png&b=000000)

好了在了解了nginx整体的目录结构后，就来看看 **nginx.conf** 这个文件这个文件是nginx的核心配置，**想玩转nginx，读懂这个配置文件是必不可少的一项基本功！**

4、nginx.conf文件 解读
=================

首先我们要知道`nginx.conf文件是由一个一个的指令块组成的`，nginx用{}标识一个指令块，指令块中再设置具体的指令(注意 指令必须以 ; 号结尾)，指令块有`全局块`，`events块`，`http块`，`server块`和`location块 以及 upstream块`。精简后的结构如下：

```vbscript
全局模块
event模块
http模块
    upstream模块
    
    server模块
        location块
        location块
        ....
    server模块
        location块
        location块
        ...
    ....    

```



**各模块的功能作用如下描述：**

1.  **全局模块：** 配置影响nginx全局的指令，比如运行nginx的用户名，nginx进程pid存放路径，日志存放路径，配置文件引入，worker进程数等。
2.  **events块：** 配置影响nginx服务器或与用户的网络连接。比如每个进程的最大连接数，选取哪种事件驱动模型（select/poll epoll或者是其他等等nginx支持的）来处理连接请求，是否允许同时接受多个网路连接，开启多个网络连接序列化等。
3.  **http块：** 可以嵌套多个server，配置代理，缓存，日志格式定义等绝大多数功能和第三方模块的配置。如文件引入，mime-type定义，日志自定义，是否使用sendfile传输文件，连接超时时间，单连接请求数等。
4.  **server块：** 配置虚拟主机的相关参数比如域名端口等等，一个http中可以有多个server。
5.  **location块：** 配置url路由规则
6.  **upstream块：** 配置上游服务器的地址以及负载均衡策略和重试策略等等

**下面看下nginx.conf长啥样并对一些指令做个解释：**

```shell
# 注意：有些指令是可以在不同指令块使用的（需要时可以去官网看看对应指令的作用域）。我这里只是演示
# 这里我以/usr/local/nginx/conf/nginx.conf文件为例

[root@localhost /usr/local/nginx]# cat /usr/local/nginx/conf/nginx.conf

#user  nobody; # 指定Nginx Worker进程运行用户以及用户组，默认由nobody账号运行
worker_processes  1;  # 指定工作进程的个数，默认是1个。具体可以根据服务器cpu数量进行设置， 比如cpu有4个，可以设置为4。如果不知道cpu的数量，可以设置为auto。 nginx会自动判断服务器的cpu个数，并设置相应的进程数
#error_log  logs/error.log;  # 用来定义全局错误日志文件输出路径，这个设置也可以放入http块，server块，日志输出级别有debug、info、notice、warn、error、crit可供选择，其中，debug输出日志最为最详细，而crit输出日志最少。
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info; # 指定error日志位置和日志级别
#pid        logs/nginx.pid;  # 用来指定进程pid的存储文件位置

events {
    accept_mutex on;   # 设置网路连接序列化，防止惊群现象发生，默认为on
    
    # Nginx支持的工作模式有select、poll、kqueue、epoll、rtsig和/dev/poll，其中select和poll都是标准的工作模式，kqueue和epoll是高效的工作模式，不同的是epoll用在Linux平台上，而kqueue用在BSD系统中，对于Linux系统，epoll工作模式是首选
    use epoll;
    
    # 用于定义Nginx每个工作进程的最大连接数，默认是1024。最大客户端连接数由worker_processes和worker_connections决定，即Max_client=worker_processes*worker_connections在作为反向代理时，max_clients变为：max_clients = worker_processes *worker_connections/4。进程的最大连接数受Linux系统进程的最大打开文件数限制，在执行操作系统命令“ulimit -n 65536”后worker_connections的设置才能生效
    worker_connections  1024; 
}

# 对HTTP服务器相关属性的配置如下
http {
    include       mime.types; # 引入文件类型映射文件 
    default_type  application/octet-stream; # 如果没有找到指定的文件类型映射 使用默认配置 
    # 设置日志打印格式
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';
    # 
    #access_log  logs/access.log  main; # 设置日志输出路径以及 日志级别
    sendfile        on; # 开启零拷贝 省去了内核到用户态的两次copy故在文件传输时性能会有很大提升
    #tcp_nopush     on; # 数据包会累计到一定大小之后才会发送，减小了额外开销，提高网络效率
    keepalive_timeout  65; # 设置nginx服务器与客户端会话的超时时间。超过这个时间之后服务器会关闭该连接，客户端再次发起请求，则需要再次进行三次握手。
    #gzip  on; # 开启压缩功能，减少文件传输大小，节省带宽。
    sendfile_max_chunk 100k; #每个进程每次调用传输数量不能大于设定的值，默认为0，即不设上限。
    
    # 配置你的上游服务（即被nginx代理的后端服务）的ip和端口/域名
    upstream backend_server { 
        server 172.30.128.65:8080;
        server 172.30.128.65:8081 backup; #备机
    }

    server {
        listen       80; #nginx服务器监听的端口
        server_name  localhost; #监听的地址 nginx服务器域名/ip 多个使用英文逗号分割
        #access_log  logs/host.access.log  main; # 设置日志输出路径以及 级别，会覆盖http指令块的access_log配置
        
        # location用于定义请求匹配规则。 以下是实际使用中常见的3中配置（即分为：首页，静态，动态三种）
       
        # 第一种：直接匹配网站根目录，通过域名访问网站首页比较频繁，使用这个会加速处理，一般这个规则配成网站首页，假设此时我们的网站首页文件就是： usr/local/nginx/html/index.html
        location = / {  
            root   html; # 静态资源文件的根目录 比如我的是 /usr/local/nginx/html/
            index  index.html index.htm; # 静态资源文件名称 比如：网站首页html文件
        }
        # 第二种：静态资源匹配（静态文件修改少访问频繁，可以直接放到nginx或者统一放到文件服务器，减少后端服务的压力），假设把静态文件我们这里放到了 usr/local/nginx/webroot/static/目录下
        location ^~ /static/ {
            alias /webroot/static/; #访问 ip:80/static/xxx.jpg后，将会去获取/url/local/nginx/webroot/static/xxx.jpg 文件并响应
        }
        # 第二种的另外一种方式：拦截所有 后缀名是gif,jpg,jpeg,png,css.js,ico这些 类静态的的请求，让他们都去直接访问静态文件目录即可
        location ~* \.(gif|jpg|jpeg|png|css|js|ico)$ {
            root /webroot/static/;
        }
        # 第三种：用来拦截非首页、非静态资源的动态数据请求，并转发到后端应用服务器 
        location / {
            proxy_pass http://backend_server; #请求转向 upstream是backend_server 指令块所定义的服务器列表
            deny 192.168.3.29; #拒绝的ip （黑名单）
            allow 192.168.5.10; #允许的ip（白名单）
        }
        
        # 定义错误返回的页面，凡是状态码是 500 502 503 504 总之50开头的都会返回这个 根目录下html文件夹下的50x.html文件内容
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
        
    }
    # 其余的server配置 ,如果有需要的话
    #server {
        ......
    #    location / {
               ....
    #    }
    #}
    
    # include /etc/nginx/conf.d/*.conf;  # 一般我们实际使用中有很多配置，通常的做法并不是将其直接写到nginx.conf文件，
    # 而是写到新文件 然后使用include指令 将其引入到nginx.conf即可，这样使得主配置nginx.conf文件更加清晰。
    
}

```



以上就是nginx.conf文件的配置了，主要讲了一些指令的含义，当然实际的指令有很多，我在配置文件并没有全部写出来，准备放到后边章节详细阐述这些东西，比如：**location匹配规则，反向代理，动静分离，负载均衡策略，重试策略，压缩，https,限流，缓存，跨域这些** 我们都没细说，这些东西比较多比较细不可能把使用规则和细节都写到上边的配置文件中，所以我们下边一一解释说明关于这些东西的配置和使用方式。（另外值的注意的是： 因为有些指令是可以在不同作用域使用的，如果在多个作用域都有相同指令的使用，那么nginx将会遵循就近原则或者我愿称之为 **内层配置优先**。 eg: 你在 http配了日志级别，也在某个server中配了日志级别，那么这个server将使用他自己配置的已不使用外层的http日志配置）

5、location 路由匹配规则
=================

**什么是location? :** nginx根据用户请求的URI来匹配对应的location模块，匹配到哪个location，请求将被哪个location块中的配置项所处理。

location配置语法：`location [修饰符] pattern {…}`

**常见匹配规则如下：**

| 修饰符 | 作用 |
| --- | --- |
| 空 | 无修饰符的前缀匹配，匹配前缀是 你配置的（比如说你配的是 /aaa） 的url |
| \= | 精确匹配 |
| ~ | 正则表达式模式匹配，区分大小写 |
| ~\* | 正则表达式模式匹配，不区分大小写 |
| ^~ | ^~类型的前缀匹配，类似于无修饰符前缀匹配，不同的是，如果匹配到了，那么就停止后续匹配 |
| / | 通用匹配，任何请求都会匹配到（只要你域名对，所有请求通吃！） |

5.1、前缀匹配（无修饰符）
--------------

首先我提前创建了prefix\_match.html文件，之后改一下nginx.conf文件（给前缀是 /prefixmatch 的请求返回 /etc/nginx/locatest/prefix\_match.html 这个文件） ，如下： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7ddbcfcc11c34bbd96c9fea044bf73d4~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=622&h=378&s=32366&e=png&b=010101)

然后在宿主机hosts中配置域名 172.30.128.65 [www.locatest.com](https://link.juejin.cn?target=http%3A%2F%2Fwww.locatest.com "http://www.locatest.com") 映射后，观察到nginx服务器返回内容如下： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8ec24b7ceb4046fda34c95041393ad95~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2420&h=2014&s=291249&e=png&b=010101)

shell

 代码解读

复制代码

`curl http://www.locatest.com/prefixmatch     ✅ 301 curl http://www.locatest.com/prefixmatch?    ✅ 301 curl http://www.locatest.com/PREFIXMATCH     ❌ 404 curl http://www.locatest.com/prefixmatch/    ✅ 200 curl http://www.locatest.com/prefixmatchmmm  ❌ 404 curl http://www.locatest.com/prefixmatch/mmm ❌ 404 curl http://www.locatest.com/aaa/prefixmatch/❌ 404`

可以看到 `域名/prefixmatch` 和`域名/prefixmatch?` 返回了301 ，原因在于prefixmatch映射的 /etc/nginx/locatest/ 是个目录，而不是个文件所以nginx提示我们301，这个我们不用管没关系，总之我们知道：`域名/prefixmatch`，`域名/prefixmatch?` 和`域名/prefixmatch/` 这三个url通过我们配置的 **无修饰符前缀匹配规则** 都能匹配上就行了。

ps：_为了方便，我们下边的几个location规则演示不再跳转静态文件了，而是直接return一句话。_

5.2、精确匹配（ = ）
-------------

为了演示精确匹配，我们再给nginx.conf文件增加一个location配置，如下标红处： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/79c3aaedaedc46e2bdb4e1d21918ebb1~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=730&h=498&s=36209&e=png&b=000000)

实际效果如下： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4b20c9d644e64ee18f03901f9adf2263~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1342&h=1142&s=139125&e=png&b=010101)

ruby

`http://www.locatest.com/exactmatch      ✅ 200 http://www.locatest.com/exactmatch？    ✅ 200 http://www.locatest.com/exactmatch/     ❌ 404 http://www.locatest.com/exactmatchmmmm  ❌ 404 http://www.locatest.com/EXACTMATCH      ❌ 404`

可以看出来精确匹配就是精确匹配，差一个字也不行！

5.3、前缀匹配（ ^~ ）
--------------

我们上边说了不带任何修饰符的前缀匹配（5.1小节），这里我们看下 修饰符是 ^~的 前缀匹配和不带修饰符的前缀匹配有啥区别，先在ngnx.conf文件增加个location并配好如下： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/02a5131e4e7e41b68aa60fad6ca3a950~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=682&h=390&s=38232&e=png&b=010101) curl效果如下： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/901e9f889df34413bd261db7f31807a9~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1446&h=1168&s=171046&e=png&b=010101)



`curl http://www.locatest.com/exactprefixmatch     ✅ 200 curl http://www.locatest.com/exactprefixmatch/    ✅ 200 curl http://www.locatest.com/exactprefixmatch?    ✅ 200 curl http://www.locatest.com/exactprefixmatchmmm  ✅ 200 curl http://www.locatest.com/exactprefixmatch/mmm ✅ 200 curl http://www.locatest.com/aaa/exactprefixmatch ❌ 404 curl http://www.locatest.com/EXACTPREFIXMATCH     ❌ 404`

可以看到带修饰符(`^~`)的前缀匹配 像：`域名/exactprefixmatchmmm` 和`域名/exactprefixmatch/mmm` 是可以匹配上的，而不带修饰符的前缀匹配这两个类型的url是匹配不上的直接返回了404 ，其他的和不带修饰符的前缀匹配似乎都差不多。

5.4、正则匹配（~ 区分大小写）
-----------------

ps：正则表达式的匹配，需要你对正则语法比较熟悉，熟悉语法后写匹配规则也就得心应手了。

添加个location并配置，如下：（ ^表示开头，$表示结尾） ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e8f1374b3dbf4875bf0959c5b41bef3e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=752&h=360&s=36101&e=png&b=000000) 实际效果如下： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c6e2f9bfc3c4d159cca6c18d5e27d1f~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1422&h=2088&s=254957&e=png&b=010101)

ruby

 代码解读

复制代码

`curl http://www.locatest.com/regexmatch      ✅ 200 curl http://www.locatest.com/regexmatch/     ❌ 404 curl http://www.locatest.com/regexmatch?     ✅ 200 curl http://www.locatest.com/regexmatchmmm   ❌ 404 curl http://www.locatest.com/regexmatch/mmm  ❌ 404 curl http://www.locatest.com/REGEXMATCH      ❌ 404 curl http://www.locatest.com/aaa/regexmatch  ❌ 404 curl http://www.locatest.com/bbbregexmatch   ❌ 404`

可以看到~修饰的正则是区分大小写的。接下来我们看下 不区分大小写的匹配。

5.5、正则匹配（~\* 不区分大小写）
--------------------

改下location 在修饰符~后加个 \* ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d698156eb04349b4abe409bda34c32e1~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=984&h=354&s=37607&e=png&b=010101) 看下实际效果： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/552f152769094533a332b4362ed18395~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1382&h=1872&s=235117&e=png&b=010101) 可以看到这次 curl [www.locatest.com/REGEXMATCH](https://link.juejin.cn?target=http%3A%2F%2Fwww.locatest.com%2FREGEXMATCH "http://www.locatest.com/REGEXMATCH") 是可以匹配上的，说明 ~\* 确实是不区分大小写的。

5.6、通用匹配（ / ）
-------------

通用匹配使用一个 / 表示，可以匹配所有请求，一般nginx配置文件最后都会有一个通用匹配规则，当其他匹配规则均失效时，请求会被路由给通用匹配规则处理，如果没有配置通用匹配，并且其他所有匹配规则均失效时，nginx会返回404错误。 ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a014bb34510b4db1a141a4b3d9fb3cf0~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1016&h=382&s=41404&e=png&b=000000) 通用匹配实际效果： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3068466ae35c4400b3915b260eb2ec75~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1034&h=710&s=123910&e=png&b=020202) 可以看到通用匹配很好理解，只要你域名写对了，那么所有的url都会被匹配上，来者不拒的感觉。

5.7、关于location 匹配优先级
--------------------

上边我们说了6种location匹配规则，那么如果存在多个到底走哪个location呢？这就的说说location的匹配优先级了。先来看下nginx官网和stackoverflow上的资料如下： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aadcbf83cfa24e118bfd7f996b51c3dd~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2972&h=1698&s=626226&e=png&b=fefefe) ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/146fa555a03f42c7905ad4af477597c0~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2226&h=1800&s=471724&e=png&b=fcfcfc) 综上资料我们对**location匹配优先级的总结如下：**

1.  优先走`精确匹配`，精确匹配命中时，直接走对应的location，停止之后的匹配动作。
2.  `无修饰符类型的前缀匹配`和 `^~ 类型的前缀匹配`命中时，收集命中的匹配，对比出最长的那一条并存起来(最长指的是与请求url匹配度最高的那个location)。
3.  `如果`步骤2中最长的那一条匹配`是^~类型的前缀匹配`，直接走此条匹配对应的location并`停止`后续匹配动作；如果步骤2`最长的那一条匹配`不是^~类型的前缀匹配（也就`是无修饰符的前缀匹配`），则`继续往下`匹配
4.  按location的声明顺序，执行正则匹配，当找到第一个命中的正则location时，停止后续匹配。
5.  都没匹配到，走通用匹配（ / ）（如果有配置的话），如果没配置通用匹配的话，上边也都没匹配上，到这里就是404了。

**如果非要给修饰符排个序的话就是酱样子：** `=` > `^~` > `正则` > `无修饰符的前缀匹配` > `/`

_ok关于location就到这里，location是一个很重要的点，学好这个才知道nginx到底是咋匹配url的。_

6、反向代理
======

反向代理示意图我们上边说过，这里再次粘一下： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/12feb26340c74998bd7917f1a21d83fa~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1906&h=1212&s=132766&e=png&b=fdfdfd)

接下来我们开始用一个小demo来演示反向代理的使用

6.1、服务准备
--------

首先将我本地的一个服务打成 胖jar： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b061b43645f469c86b008c7b8a341da~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3394&h=2096&s=633913&e=png&b=2c2c2c) 然后使用java -jar方式启动服务，且指定端口为8081：

perl

 代码解读

复制代码

`java -jar /Users/hzz/myself_project/xzll/study-admin/study-admin-service/target/study-admin-service.jar --server.port=8081`

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/88bd8441e4b0444f9966a97482248319~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3536&h=1816&s=837365&e=png&b=010101) 最后使用postman测试接口是否正常（注意此时还没被nginx代理，而是直接调的服务）： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d1179942c8e84befb0eed6e7b9238e88~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3238&h=1396&s=224332&e=png&b=fcfcfc) 启动服务并验证接口无误后，接下来我们修改nginx配置文件。让nginx反向代理我们的服务。

6.2、修改nginx.conf文件
------------------

要让 **`nginx 代理`** 我们的 **`服务`** 很简单，简单描述一下就是 **`两步：`**

1.  **通过upstream指令块来定义我们的上游服务（即被代理的服务）**
2.  **通过location指令块中的 proxy\_pass指令，指定该location要路由到哪个upstream**

配置好1和2后，如果来了请求后 会通过url路由到对应的location, 然后nginx会将请求打到upstream定义的服务地址中去，下边我们看看：

使用 vi /etc/nginx/nginx.conf命令修改nginx.conf文件，如下： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f39d0590e33a4fde83367db2665481d5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1160&h=940&s=87778&e=png&b=000000) （注意上边的 proxy\_pass [http://mybackendserver/](https://link.juejin.cn?target=http%3A%2F%2Fmybackendserver%2F "http://mybackendserver/") **后边这个斜线加和不加区别挺大的**，`加的话不会拼接/backend` , `而不加的话会拼接 /backend` ,这一点我们在15.5.3小节会讲到，这里留意下就好）

6.3、测试反向代理
----------

修改完后我们执行 nginx -s reload 命令重新加载nginx配置，然后再potsman中调用一下，如下：

> ps：在第一次调用中出现了一个错误: failed (13: Permission denied) while connecting to upstream： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b20fc1bdc0fd41678904ae1859ad6255~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3562&h=160&s=78420&e=png&b=010101) 解决办法很简单在虚拟机执行命令：setenforce 0 来关闭`seLinux的限制`即可，或者参考：[stackoverflow上的解决方案](https://link.juejin.cn?target=https%3A%2F%2Fstackoverflow.com%2Fquestions%2F23948527%2F13-permission-denied-while-connecting-to-upstreamnginx "https://stackoverflow.com/questions/23948527/13-permission-denied-while-connecting-to-upstreamnginx")

解决后再次调用发现可以了：（注意 [www.proxytest.com](https://link.juejin.cn?target=http%3A%2F%2Fwww.proxytest.com "http://www.proxytest.com") 是我随便写的域名没做dns解析，我是在请求发出方 宿主机 配了host，所以才能请求通） ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fcf2fba55362411cb87886220be08b31~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3220&h=1416&s=227407&e=png&b=fdfdfd)同时也可使用命令 tail -n +1 -f access.log 观察到nginx日志输出如下： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6da779ef690a4df0b18e56f914e30ff6~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3134&h=328&s=82297&e=png&b=010101)

6.4、反向代理流程与原理
-------------

对于上边演示的反向代理案例的流程与原理，我们来个示意图如下：（这个图比较重要） ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6cd983baf59944abb6510a3b717a8490~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2612&h=1450&s=259427&e=png&b=ffffff)

接下来我们演示下负载均衡。

7、负载均衡
======

说到负载均衡很多人应该并不陌生，总而言之负载均衡就是：避免高并发高流量时请求都聚集到某一个服务或者某几个服务上，而是让其均匀分配（或者能者多劳），从而减少高并发带来的系统压力，从而让服务更稳定。对于nginx来说，负载均衡就是从 `upstream` 模块定义的后端服务器列表中按照配置的负载策略选取一台服务器接受用户的请求。

7.1、准备3个不同端口的springboot服务
-------------------------

想要演示负载均衡，我们首先得多搞几个服务，搞一个服务是没法儿演示的。所以我启动了3个不同端口（8081，8082，8083）的springboot服务，如下：

perl

 代码解读

复制代码

`java -jar /Users/hzz/myself_project/xzll/study-admin/study-admin-service/target/study-admin-service.jar --server.port=8081 java -jar /Users/hzz/myself_project/xzll/study-admin/study-admin-service/target/study-admin-service.jar --server.port=8082 java -jar /Users/hzz/myself_project/xzll/study-admin/study-admin-service/target/study-admin-service.jar --server.port=8083`

使用 command+D 对iterm2进行分屏 ，最左侧是8081端口，中间是8082，右侧是8083： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9ff1fcf675f74538ab25a0808abc8d5d~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3550&h=1950&s=1320742&e=png&b=232323) 接下来我们说一下负载策略再开始。

7.2、nginx常用的负载策略:
-----------------

| 负载策略 | 描述 | 特点 |
| --- | --- | --- |
| 轮询 | 默认方式 | 1\. 每个请求会按时间顺序逐一分配到不同的后端服务器  
2\. 在轮询中，如果服务器down掉了，会自动剔除该服务器  
3\. 缺省配置就是轮询策略  
4\. 此策略适合服务器配置相当，无状态且短平快的服务使用 |
| weight | 权重方式 | 1\. 在轮询策略的基础上指定轮询的几率  
2\. 权重越高分配到的请求越多  
3\. 此策略可以与least\_conn和ip\_hash结合使用  
4\. 此策略比较适合服务器的硬件配置差别比较大的情况 |
| ip\_hash | 依据ip的hash值来分配 | 1\. 在nginx版本1.3.1之前，不能在ip\_hash中使用权重（weight）  
2\. ip\_hash不能与backup同时使用  
3\. 此策略适合有状态服务，比如session  
4\. 当有服务器需要剔除，必须手动down掉 |
| least\_conn | 最少连接方式 | 1\. 此负载均衡策略适合请求处理时间长短不一造成服务器过载的情况 |
| fair（第三方） | 响应时间方式 | 1\. 根据后端服务器的响应时间来分配请求，响应时间短的优先分配  
2\. Nginx本身不支持fair，如果需要这种调度算法，则必须安装upstream\_fair模块 |
| url\_hash（第三方） | 依据URL分配方式 | 1\. 按访问的URL的哈希结果来分配请求，使每个URL定向到一台后端服务器  
2\. Nginx本身不支持url\_hash，如果需要这种调度算法，则必须安装Nginx的hash软件包 |

### 7.2.1、轮询

轮询策略是默认的，所以只需要如下这样修改配置文件就可以了： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ce021eaa72194ccea51dc6f699c55e95~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1186&h=938&s=102633&e=png&b=000000) 重启nginx后观察一下： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e2166f81edf34ec5a8b4e932753a4e75~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3542&h=1922&s=980779&e=png&b=2b2b2b) 以上图片可以看到，是按upstream中的先后顺序来进行轮询的。

### 7.2.2、weight

weight指令用于指定轮询机率，weight的默认值为1，weight的数值与访问比率成正比。 接下来我们指定8082端口的服务的weight=2，如下： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3243101b8ce348ce901e5bb72e4e16c8~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1182&h=950&s=99825&e=png&b=010101) 看下权重策略下的请求结果： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d5281831071b4af89abbcab0872bbca3~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3554&h=1958&s=1160694&e=png&b=222222) 可以看到在一轮轮询中，8081命中1次，8082由于配置了 weight=2所以命中了2次，8083命中了1次。即配置了weight=2的8082服务，命中几率是8081或者8083的两倍

### 7.2.3、ip\_hash

设定ip哈希很简单，就是在你的upstream中 指定 `ip_hash;`即可，如下： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/65caa05e4c074534b40d5e9d980c871a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1156&h=920&s=89812&e=png&b=010101) 重启nginx后看下效果： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ea9f2b9e30cf4bba9f57b33ecbc87976~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3542&h=1948&s=576034&e=png&b=282828) 可以看到，由于我的访问ip总是固定的宿主机的172.30.128.64 根据hash算法我的ip被匹配给了8083端口的服务，所以只要我不换ip 不管我请求多少次，请求都是被 转发到了8083的服务上了。

### 7.2.4、least\_conn

同ip\_hash一样，设定最小连接数策略也很简单，就是在你的upstream中 指定 `least_conn;`即可，如下： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/66c66d671b044574aa52abe89e4f1133~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1116&h=960&s=110038&e=png&b=000000) 由于我这里最小连接数看不出啥效果，所以就不演示截图了，知道怎么配置最小连接数即可。关于第三方的负载策略。不做过多说明了，可以看看：[nginx官方文档](https://link.juejin.cn?target=https%3A%2F%2Fdocs.nginx.com%2Fnginx%2Fadmin-guide%2Fload-balancer%2Fhttp-load-balancer%2F "https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/") 获取其他的网上资料。

8、动静分离
======

在说动静分离前，我们要知道为何要做动静分离以及他能解决啥问题，首先，我们常见的web系统中会有大量的静态资源文件比如掘金主页面刷新后的f12如下： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8215059d01a645e7a6ea31a4abec3314~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1924&h=1714&s=739313&e=png&b=fefefe) 可以看到有很多静态资源，如果将这些资源都搞到后端服务的话，将会提高后端服务的压力且占用带宽增加了系统负载（要知道，静态资源的访问频率其实蛮高的）所以为了避免该类问题我们可以把不常修改的静态资源文件放到nginx的静态资源目录中去，这样在访问静态资源时直接读取nginx服务器本地文件目录之后返回，这样就大大减少了后端服务的压力同时也加快了静态资源的访问速度，何为静，何为动呢？：

1.  **静：** 将不常修改且访问频繁的静态文件，放到nginx本地静态目录（当然也可以搞个静态资源服务器专门存放所有静态文件）
2.  **动：** 将变动频繁/实时性较高的比如后端接口，实时转发到对应的后台服务

接下来我们将构造一个html页面，然后点击按钮后发送get请求到后端接口。流程如下： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8418a768c2784f908ef61074ffc7b8a4~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1146&h=708&s=95694&e=png&b=ffffff)

8.1、准备工作
--------

首先我们搞个html（请原谅我这粗糙的前端代码😂😂）,内容如下： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aa5b2655fc014eadbe37f5670d575413~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2318&h=1974&s=470447&e=png&b=1f1f1f) 之后使用

ruby

 代码解读

复制代码

`scp /Users/hzz/fsdownload/index_page.html root@172.30.128.65:/usr/local/nginx/test/static`

命令将index\_page.html 文件上传到虚拟机。

8.2、修改nginx.conf文件
------------------

首先我们配置俩location规则,一个（ /frontend ）是读取静态文件，一个（/backend）是转发到 我们配置的upstream服务中去。如下： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb0421cc4d1c47a788e8a06c597a9da8~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1368&h=1590&s=170684&e=png&b=000000)

8.3、演示
------

首先我们在浏览器输入：[www.proxytest.com/frontend/](https://link.juejin.cn?target=http%3A%2F%2Fwww.proxytest.com%2Ffrontend%2F "http://www.proxytest.com/frontend/") ，可以看到请求返回了一个html页面，其实就是我们刚才的 /usr/local/nginx/test/static/index\_page.html文件 ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b2684cbef96645298fc0cd904579265a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3548&h=1980&s=719150&e=png&b=ffffff) 接着我们输入要查询的人名，之后点击 "调用get接口" 按钮，如下： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9e17b5fbfdbc4b56a36fe271dca2beb8~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3516&h=1884&s=930330&e=png&b=ffffff) 返回数据： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/25b739593cea436da5597cd299861115~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3416&h=1686&s=609426&e=png&b=ffffff) 查看nginx访问日志可以看到两次请求的输出： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/085c8b4eb6a747ef8694a3fbc043f9b2~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3584&h=296&s=63836&e=png&b=000000)

ok到这里动静分离就演示完了，接下来我们看下跨域

9、跨域
====

9.1、为何会产生跨域？
------------

产生跨域问题的主要原因就在于同源策略，为了保证用户信息安全，防止恶意网站窃取数据，同源策略是必须的，该政策由 Netscape 公司于1995年引入浏览器。目前，所有浏览器都实行这个政策。同源策略主要是指三点相同即：**协议+域名+端口 相同的两个请求**，则可以被看做**是同源**的，但如果**其中任意一点存在不同**，则代表是**两个不同源的请求**，同源策略会限制不同源之间的资源交互从而减少数据安全问题。

9.2、跨域演示
--------

首先我在nginx.conf文件中加一个server配置也即将前后端配成不同的server 并且监听的端口以及域名名称都不一致，从而`造成`访问`前端服务和后端服务`时候 这俩服务`不是"同源"`， 如下： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/820db4f4dfe643a88f3c628f23cfa392~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1288&h=1590&s=159800&e=png&b=000000) 之后我修改index\_page中的后端地址： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e3165e5e643d404c90e6dbb402530a7d~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1898&h=1072&s=243627&e=png&b=1f1f1f) 重启nginx，并配置宿主机的hosts文件：

makefile

 代码解读

复制代码

`172.30.128.65 www.front.com 172.30.128.65:90  www.backend.com`

之后在浏览器中测试一下： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9c4432be0fc544d3a63fa4e6b312cd9c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3196&h=1596&s=303575&e=png&b=f7f7f7) 可以看到浏览器提示我们受同源规则影响我们不能跨域访问资源。造成的原因是我的两个域名解析出来的端口不一致 一个是80一个是90。不符合同源策略，所以必然会有跨域报错。

9.3、nginx解决跨域
-------------

首先想解决跨越就得避免不同源，而我们可不可以 把对后端的代理 放在前端的server中呢（也就是说让前后端统一使用一个端口，一个server\_name）？答案是可以的，因为server支持多个location配置呀（一个location处理前端，一个location转发后端），我们改下配置文件试一把如下： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8b7c974e3c1647a78ebb7217b7e838f1~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1426&h=1416&s=147711&e=png&b=000000) 之后重启nginx后在浏览器输入 [www.xxxadminsystem.com/page/](https://link.juejin.cn?target=http%3A%2F%2Fwww.xxxadminsystem.com%2Fpage%2F "http://www.xxxadminsystem.com/page/") ，效果如下： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1f62b80ed64d42ba96a189d6d84c3371~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3548&h=2052&s=933049&e=png&b=fefefe) 上边/page请求返回了html页面之后我们输入参数点击“调用get接口”查看到后端接口的调用如下： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2bf18e319e7249e1841e8e8e08089c05~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3570&h=2020&s=883770&e=png&b=ffffff) ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3e328b4af14142dfad71ad39cfc09d68~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2454&h=1430&s=454537&e=png&b=ffffff) 从上边可以看到，我上边设想的方式是可行的。

*   当然有些资料上有说使用 设置header的方式解决跨域，但是在实际测试中，设置header的方式始终没解决跨域，试了好久也没解决掉😂😂，有试过此方式解决的大佬帮忙看看我这是哪里配错了还是咋的在此提前感谢了。
    
    > ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ab1b5d522d2f45b1a3ce5d7353f4c9c2~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1680&h=1880&s=395837&e=png&b=010101)
    

10、缓存
=====

在开头我们就介绍过，nginx代理缓存可以在某些场景下有效的减少服务器压力，让请求快速响应，从而提升用户体验和服务性能，那么nginx缓存如何使用呢？在使用及演示前我们先来熟悉下相关的配置以及其含义，知道了这些才能更好的使用nginx缓存。

10.1、nginx缓存配置参数表格一览
--------------------

| 指令名称 | 作用解释 | 语法 | 默认配置 | 示例 | 作用域 |
| --- | --- | --- | --- | --- | --- |
| proxy\_cache | 设置是否开启对后端响应的缓存。 | proxy\_cache zone | off; | proxy\_cache off; | proxy\_cache mycache; # 规定开启nginx缓存并且缓存名称为: mycache | http, server, location |
| proxy\_cache\_valid | 配置什么状态码可以被缓存，以及缓存时长 | proxy\_cache\_valid \[code ...\] time; | 没有默认值 | proxy\_cache\_valid 200 304 2m; # 对于状态为200和304的缓存文件，缓存时间是2分钟 | http, server, location |
| proxy\_cache\_key | 设置缓存文件的 key | proxy\_cache\_key string; | proxy\_cache\_key schemeschemeschemeproxy\_host$request\_uri; | proxy\_cache\_key "hosthosthostrequest\_uri $cookie\_user"; # 使用host +请求的uri以及cookie拼接成缓存key | http, server, location |
| proxy\_cache\_path | 指定缓存存储的路径，文件名为cache key的md5值，然后多级目录的话，根据level参数来生成，key\_zone参数用来指定在共享内存中缓存数据的名称和内存大小，比如keys\_zone=mycache:100m，inactive用来指定缓存没有被访问后超时移除的时间，默认是10分钟，也可以自己指定比如inactive=2h ；max\_size 用来指定缓存的最大值，超过这个值则会自动移除最近最少使用（lru淘汰算法）的缓存 这个指令对应的参数很多，具体见官网：[proxy\_cache\_path](https://link.juejin.cn?target=https%3A%2F%2Fnginx.org%2Fen%2Fdocs%2Fhttp%2Fngx_http_proxy_module.html%3F_ga%3D2.13518455.1300709501.1700036543-1660479828.1698914648%23proxy_cache_path "https://nginx.org/en/docs/http/ngx_http_proxy_module.html?_ga=2.13518455.1300709501.1700036543-1660479828.1698914648#proxy_cache_path") | proxy\_cache\_path path \[levels=levels\] \[use\_temp\_path=on|off\] keys\_zone=name:size \[inactive=time\] \[max\_size=size\] \[min\_free=size\] \[manager\_files=number\] \[manager\_sleep=time\] \[manager\_threshold=time\] \[loader\_files=number\] \[loader\_sleep=time\] \[loader\_threshold=time\] \[purger=on|off\] \[purger\_files=number\] \[purger\_sleep=time\] \[purger\_threshold=time\]; | 无 | proxy\_cache\_path /data/nginx/cache levels=1:2 keys\_zone=mycache:128m inactive=3d max\_size=2g; # 设置缓存存放的目录为/data/nginx/cache，并设置缓存名称为mycache，大小为128m， 三天未被访问过的缓存将自动清除，磁盘中缓存的最大容量为2GB。 | http |
| proxy\_cache\_bypass | 定义不从缓存中获取响应数据的条件。如果字符串参数中至少有一个值不为空且不等于" 0 "，则不会从缓存中获取响应: | proxy\_cache\_bypass string ...; | 没有默认值 | proxy\_cache\_bypass cookienocachecookie\_nocache cookien​ocachearg\_nocache$arg\_comment; | http, server, location |
| proxy\_cache\_min\_uses | 指定某一个相同请求在几次请求之后才缓存响应内容 | proxy\_cache\_min\_uses number; | proxy\_cache\_min\_uses 1; | proxy\_cache\_min\_uses 3; 规定某一个请求在第3次之后才走nginx缓存 | http, server, location |
| proxy\_cache\_use\_stale | 指定后端服务器在返回什么状态码的情况下可以使用过期的缓存 | proxy\_cache\_use\_stale error timeout invalid\_header http\_500 http\_502 http\_503 ... |off ; | proxy\_cache\_use\_stale off; | proxy\_cache\_use\_stale error timeout http\_500 http\_502 http\_503 http\_504; # 规定服务在出现error timeout,以及502,503,504时可使用过期缓存 | http, server, location |
| proxy\_cache\_lock | 默认不开启，开启后若出现并发重复请求，nginx只让一个请求去后端读数据，其他的排队并尝试从缓存中读取; | proxy\_cache\_lock on |off; | proxy\_cache\_lock off; | proxy\_cache\_lock on; # 开启缓存锁 | http, server, location |
| proxy\_cache\_lock\_timeout | 等待缓存锁(proxy\_cache\_lock)超时之后将直接请求后端，且结果不会被缓存 | proxy\_cache\_lock\_timeout time; | proxy\_cache\_lock\_timeout 5s; | proxy\_cache\_lock\_timeout 6s; # 等待缓存锁超时（6ms）之后将直接请求后端，结果不会被缓存。 | http, server, location |
| proxy\_cache\_methods | 如果客户端请求方法在该指令中，则响应将被缓存。“GET”和“HEAD”方法总是被添加到列表中，尽管建议显式地指定它 | proxy\_cache\_methods GET|HEAD |POST ...; | proxy\_cache\_methods GET HEAD; | proxy\_cache\_methods GET HEAD PUT POST; # 规定 可缓存的方法有 ：get head put post | http, server, location |
| .............. | .......... | .......... | .......... | .......... | .......... |

事实上，**ngx\_http\_proxy\_module**模块中**代理缓存proxy\_cache相关的指令远不止这些**，在这里我不可能把所有都列出来，只列出上边那几个已经很占篇幅了，如果有需要请参考： [nginx官方文档](https://link.juejin.cn?target=https%3A%2F%2Fnginx.org%2Fen%2Fdocs%2Fhttp%2Fngx_http_proxy_module.html%3F_ga%3D2.13518455.1300709501.1700036543-1660479828.1698914648%23proxy_cache "https://nginx.org/en/docs/http/ngx_http_proxy_module.html?_ga=2.13518455.1300709501.1700036543-1660479828.1698914648#proxy_cache")， 在官方文档中，详细描述了ngx\_http\_proxy\_module模块（包含了proxy\_cache部分）的各种指令、作用以及使用方式，相信在遇到困难和疑惑时，官方文档永远是你最好的老师！如下： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/17766455a2f74e7ab69e249594eb5756~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2650&h=1946&s=473524&e=png&b=fbfbfb)

10.2、nginx缓存使用与效果演示
-------------------

**首先我们想要的效果是：** 将 **url+参数一样** 的请求的结果，缓存到nginx。

接下来我们修改下nginx.conf文件,如下：

```ini
http{
    ...
    # 指定缓存存放目录为/usr/local/nginx/test/nginx_cache_storage，并设置缓存名称为mycache，大小为64m， 1天未被访问过的缓存将自动清除，磁盘中缓存的最大容量为1gb
    proxy_cache_path /usr/local/nginx/test/nginx_cache_storage levels=1:2 keys_zone=mycache:64m inactive=1d max_size=1g;
    ...
    
    server{
        ...
        #  指定 username 参数中只要有字母 就不走nginx缓存  
        if ($arg_username ~ [a-z]) {
             set $cache_name "no cache";
        }
        
        location  /interface {
                   proxy_pass http://mybackendserver/;
                   # 使用名为 mycache 的缓存空间
                   proxy_cache mycache;
                   # 对于200 206 状态码的数据缓存2分钟
                   proxy_cache_valid 200 206 1m;
                   # 定义生成缓存键的规则（请求的url+参数作为缓存key）
                   proxy_cache_key $host$uri$is_args$args;
                   # 资源至少被重复访问2次后再加入缓存
                   proxy_cache_min_uses 3;
                   # 出现重复请求时，只让其中一个去后端读数据，其他的从缓存中读取
                   proxy_cache_lock on;
                   # 上面的锁 超时时间为4s，超过4s未获取数据，其他请求直接去后端
                   proxy_cache_lock_timeout 4s;
                   # 对于请求参数中有字母的 不走nginx缓存
                   proxy_no_cache $cache_name; # 判断该变量是否有值，如果有值则不进行缓存，没有值则进行缓存
                   # 在响应头中添加一个缓存是否命中的状态（便于调试）
                   add_header Cache-status $upstream_cache_status;    
        }
        ...
}

```



![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/74810a8e148442b3a8e4261fb735fb8f~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2258&h=1738&s=346084&e=png&b=000000)

ps: 在上边配置文件中除了缓存相关的配置，我们还加了一个参数：

bash

 代码解读

复制代码

`add_header Cache-status $upstream_cache_status;`

> 这个参数可以方便从响应头看到是否命中了nginx缓存，方便我们观察，其不同的值有不同的含义，upstream\_cache\_status的值集合如下：  
>   
> MISS：请求未命中缓存  
> HIT：请求命中缓存。  
> EXPIRED：请求命中缓存但缓存已过期。  
> STALE：请求命中了陈旧缓存。  
> REVALIDDATED：Nginx验证陈旧缓存依然有效。  
> UPDATING：命中的缓存内容陈旧，但正在更新缓存。  
> BYPASS：响应结果是从原始服务器获取的。  

nginx.conf文件配好后，在请求之前先看下数据库 “hzznb” 和 “张无忌” 都是存在的，如下： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9c965cc026144f94805a82c97c80663d~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2194&h=456&s=226716&e=png&b=2e2e2e)

接下来我们`分别查询 张无忌 和 hzznb` 来看看缓存命中情况：

查询usernam=`张无忌；` 第一次（未命中，因为此 url+参数 之前没请求过缓存中确实没有）： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/05c283c6753a4f20b03745d3dea240de~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3508&h=1618&s=483677&e=png&b=fdfdfd) 第二次也未命中就不截图了

第三次（未命中）： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ce285146c833461f9f405d9d12a12eb7~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3476&h=1486&s=438841&e=png&b=ffffff) 第四次（**命中**）： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/87dfb978af614a659333ba2bc990843d~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3476&h=1670&s=491789&e=png&b=ffffff)

查询usernam=`hzznb；` 第一次(未命中)： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3c576e4715344e6abc4560fc7812be03~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3508&h=1478&s=413353&e=png&b=ffffff) 第2,3次也都是miss 即未命中（截图略）

第四次：（仍然是未命中，说明我们在nginx.conf中配置的规则：”参数中带字母则不缓存“ 生效了！）： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/985388c367c14e8d9e1b53cd3093aede~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3506&h=1616&s=497156&e=png&b=fdfdfd)

11、黑白名单
=======

nginx黑白名单比较简单，allow后配置你的白名单，deny后配置你的黑名单，在实际使用中，我们一般都是建个黑名单和白名单的文件然后再nginx.copnf中incluld一下，这样保持主配置文件整洁，也好管理。下边我为了方便就直接在主配置写了。

11.1、语法作用域
----------

关于黑白名单的语法和作用，我们直接看下官网的示例： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/19d8b4060ade4e80bd57ec018630db22~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2664&h=2058&s=556935&e=png&b=fdfdfd) 可以看到ip 可以是ipv4 也可以是ipv6 也可以按照网段来配置，当然ip黑白配置可以在 http，server，location和limit\_except这几个域都可以区别只是作用粒度大小问题。当然nginx建议我们使用 ngx\_http\_geo\_module这个库，ngx\_http\_geo\_module库支持 按地区、国家进行屏蔽，并且提供了IP库，当需要配置的名单比较多或者根据地区国家屏蔽时这个库可以帮上大忙。

下面我们配置并演示一下：

11.2、黑白名单演示
-----------

**允许任何ip访问前端，然后禁止172.30.128.64访问后端**，nginx.conf文件如下： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aec3216ee5354b2a913c66bc0afe4f37~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2428&h=1832&s=356656&e=png&b=000000) 访问前端，走/page这个location（可以访问成功）： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fc8ae97280d4481cae7ee662a1c1e485~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3524&h=1808&s=809270&e=png&b=ffffff) 访问后端，走interface这个location（显示403被禁止了）： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/66ebb732a42249a8b1b91380970331c9~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3506&h=1760&s=806046&e=png&b=fefefe)

12、nginx限流
==========

Nginx主要有两种限流方式：按并发连接数限流(ngx\_http\_limit\_conn\_module)、按请求速率限流(ngx\_http\_limit\_req\_module 使用的令牌桶算法)。

关于 ngx\_http\_limit\_req\_module模块，里边有很多种限流指令，官网资料一览： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6fa7772d23e143939490bd8373a801ad~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2082&h=1782&s=339953&e=png&b=fdfdfd) 我们下面使用 ngx\_http\_limit\_req\_module 模块中的limit\_req\_zone和 limit\_req 这两个指令来达到限制单个IP的**请求速率** 的目的。

12.1、nginx限流配置解释
----------------

在 nginx.conf 中添加限流配置如下：

```ini
http{
    ...
    # 对请求速率限流
    limit_req_zone $binary_remote_addr zone=myRateLimit:10m rate=5r/s;
    
    server{
        location /interface{
            ...
            limit_req zone=myRateLimit burst=5  nodelay;
            limit_req_status 520;
            limit_req_log_level info;
        }
    }
}


```



![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8cfe0df0bd92490fb641ba50bb66c4a4~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2354&h=1860&s=358907&e=png&b=010101) 对上图标红的配置做个解释：

**$binary\_remote\_addr**：表示基于 remote\_addr(客户端IP) 来做限流 **zone=myRateLimit:10m**：表示使用myRateLimit来作为内存区域（存储访问信息）的名字，大小为10M，1M能存储16000 IP地址的访问信息，10M可以存储16W IP地址访问信息 **rate=5r/s**：表示相同ip每秒最多请求5次，nginx是精确到毫秒的，也就是说此配置代表每200毫秒处理一个请求，这意味着自上一个请求处理完后，若后续200毫秒内又有请求到达，将拒绝处理该请求（如果没配burst的话）  
**burst=5**：(英文 爆发 的意思)，意思是设置一个大小为5的缓冲队列，若同时有6个请求到达，Nginx 会处理第一个请求，剩余5个请求将放入队列，然后每隔200ms从队列中获取一个请求进行处理。若请求数大于6，将拒绝处理多余的请求，直接返回503  
**nodelay**：针对的是 burst 参数，burst=5 nodelay 这个配置表示被放到缓冲队列的这5个请求会立马处理，不再是每隔200ms取一个了。但是值得注意的是，即使这5个突发请求立马处理并结束，后续来了请求也不一定不会立马处理，因为虽然请求被处理了但是请求所占的坑并不会被立即释放，而是只能按 200ms 一个来释放，释放一个后 才将等待的请求 入队一个。  
**另外两个：** limit\_req\_status=520表示当被限流后，nginx的返回码，limit\_req\_log\_level info代表日志级别

**注意：** 如果不开启nodelay且开启了burst这个配置，那么将会严重影响用户体验（你想想假设burst队列长度为100的话每100ms处理一个,那队列最后那个请求得等10000ms=10s后才能被处理，那不超时才怪呢此时burst已经意义不大了）所以一般情况下 建议burst和nodelay结合使用，从而尽可能达到速率稳定，但突然流量也能正常处理的效果。

12.2、nginx限流（针对请求速率）
--------------------

为了突出burst和nodealy的作用，我们一步一步演示

### 12.2.1、限制每秒同一ip最多访问5次/1s

修改nginx.conf，把burst=5 nodelay注释掉，如下： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7f0d4083281f4bc68308cb03f75f053a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2150&h=1774&s=348261&e=png&b=010101) 上边的配置意味着每秒最多处理5次同样ip的请求，我们使用jmeter设置1个线程循环10次，间隔时间为100ms,效果如下（5成功5失败）： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/948f5582525a4e4ba7dfdb9b7ebacf73~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2834&h=1002&s=277682&e=png&b=3d4244) 如果我们将间隔时间改200的话，是都可以成功的，因为一秒最多5次精确到毫秒其实就是最多200ms一次,而200ms一次正好没超过我们配置的 5r/s 的速率： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2c0fe06a62b045108164463559e7db1d~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1614&h=380&s=77674&e=png&b=3d4244) 运行jemeter发现间隔200ms访问一次的请求都成功了： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cde886bec6754608af23eb37e5639329~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2856&h=936&s=279581&e=png&b=3d4244)

### 12.2.2、打开burst参数并设置成5

现在我们的速率不变还是最多5次一秒，但是设置burst=5代表缓冲队列的长度为5，nginx每隔200ms，从缓冲队列拿一个进行处理，配置如下： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4aadf8d462a2448ca290deef74241577~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1642&h=1680&s=322918&e=png&b=010101)

之后我们配置线程数量为15，每隔100ms掉一次，效果如下： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07d5fbe951ae48768073ae49e5b753ac~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2842&h=1196&s=389751&e=png&b=3d4244) 可以看到共计6个请求被处理，第一个是被nginx进程直接处理，之后往burst塞了5个（每隔200ms拿一个进行处理）剩下的都被返回了520状态码代表被拒绝了，我们找一个（1-10这个）被拒的看看状态码： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d167f2adfc7244ce87ccc75daf97abd2~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1382&h=790&s=100137&e=png&b=3a3f41)

### 12.2.3、打开nodelay

我们上边说过**打开nodlay的话**，代表放到burst队列的请求直接处理 ，**不再按速率 200ms/次 拿了**，配置如下： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d027c21be6784d1f89102cdd338c3de9~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1606&h=1744&s=321984&e=png&b=000000) 接下来我们还是配置15个线程，然后每个线程间隔100ms请求一次，看下效果： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/206e98db135f45799f7b85b9ef78d536~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2834&h=1160&s=334900&e=png&b=3d4244) 可以很明显的看到：开启nodelay后响应时间10几秒明显比不开启nodelay快很多，但是请求成功的还是6个，因为就像我们上边说的ngdelay虽然会即时处理，但是释放坑位是200ms释放一个 **`（也就是说即时开启了nodelay 但释放令牌的速度是不变的）`** ，所以nodelay参数本质上并没有提高访问速率，而仅仅是让处于burst队列的请求 `”被快速处理“` 罢了。

12.3、nginx限流（针对连接数量）
--------------------

针对连接数量的限流和速率不一样，即使你速率是1ms一次，只要你连接数量不超过设置的，那么也访问成功。如果连接数超过设置的值将会请求失败。值得注意的是他是 ngx\_http\_limit\_conn\_module模块中的，不要和 速率限流的 ngx\_http\_limit\_req\_module模块搞混了。

配置如下：

ini

 代码解读

复制代码

`http{     # 针对ip  对请求连接数限流     ...     limit_conn_zone $binary_remote_addr zone=myConnLimit:10m;      ...          server{        ...        limit_conn myConnLimit 12;     } }`    

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/96779ae313654b8eac3a78fe2fa3aa90~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1708&h=1852&s=323494&e=png&b=000000) 简单对以上标黄处说明一下，`limit_conn_zone $binary_remote_addr zone=myConnLimit:10m;` 代表的意思 是 基于连接数量限流，限流的对象是ip 名称是myConnLimit 存储空间大小10mb（即存放某ip的访问记录），limit\_conn myConnLimit 12;标识该ip最大支持12个连接超过则返回503（被限流后状态码默认是503，当然你也可以修改返回码 像上边的 针对请求速率限流 ，返回码就是 我修改的520）。

使用jmeter搞20个线程，0延迟，演示下效果： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/661dcb8c3fff497497a2beb325ea2ab0~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2844&h=1400&s=412439&e=png&b=3c4143) 可以看到由于我们配置的并发数是12，所以20个连接中有8个都被限了。这个理解起来似乎比速率限流（ngx\_http\_limit\_req\_module）简单些我们就不过多解释了。

13、https配置
==========

说到https大家应该并不陌生，我这里不啰嗦介绍了。一般我们安装的nginx模块都是不包含ssl模块的，所以需要手动安装下。安装完之后我们再说如何配置https。

13.1、https\_ssl模块安装
-------------------

首先我们使用 `nginx -V` (大写) 看下有没有安装https\_ssl模块：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7c1fb912b90f41e8aff840eac5654ad1~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3558&h=376&s=115535&e=png&b=010101) 可以看到我已经安装了,实际上我可以直接使用https\_ssl模块了但是为了文章完善性。我下边说一下https\_ssl模块的安装步骤：

> 注意：下边的https\_ssl模块是安装到我的旧版本1.23.0去了，而我当前生效运行的nginx是1.24.0 ，使用`ps -ef | grep nginx` 命令可看出当前运行的nginx： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a658014a50ff4be4bc116c7cf1d1c8d9~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1716&h=218&s=67057&e=png&b=010101) 下边的安装https\_ssl仅仅是为了内容全面，**而不是真正使用1.23.0版本的nginx或者1.23.0版本的https\_ssl模块**（本文使用的版本都是1.24.0 ，1.23.0是我之前的一个nginx版本）。

没有ssl模块情况下，首先我们`进入nginx解压目录`：我的是 /usr/local/nginx/nginx-1.23.0/ **总之就是找到你的nginx解压目录**) ，之后在该目录执行命令 `./configure --prefix=/usr/local/nginx --with-http_ssl_module` 来安装ssl模块，如下图： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fd6de3c1306e4e89aca67d583f51bc50~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1702&h=516&s=136899&e=png&b=010101) 安装好后，在/usr/local/nginx/nginx-1.23.0/ 目录中执行 `make` 命令，重新编译nginx,注意此处无需make install，make成功后，我们执行 `cp ./objs/nginx /usr/local/nginx/sbin/` 命令将编译后的文件覆盖到 /usr/local/nginx/sbin/，之后执行 `/usr/local/nginx/sbin/nginx -V` ，可看到ssl模块就安装成功了。操作记录图如下： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/63e8ba8d00a54a7fb129b4d006397444~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1600&h=1092&s=316783&e=png&b=010101)

13.2、域名购买&解析&ssl证书申请与验证
-----------------------

要配置ssl最好是有个域名，所以我花一杯酱香拿铁的💰买了一个域名（在买域名前需要进行域名模板实名，具体操作去腾讯云官网看这里不啰嗦了），我买的域名是： **`hzznb-xzll.xyz`**

ps： 如果我没记错的话这是我的第一个域名，虽然他10块钱但是我很珍惜他😄😄，接下来可以在我的域名中看到已经成功了。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/98e304262b96425281fa2e86de4a9961~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3448&h=1066&s=347574&e=png&b=e4ebf9) 有域名了，下一步就需要配置DNS解析，让别人通过公网也访问到呀，所以点击上图的 **解析** 按钮后，来到下边的页面添加解析记录，如下：（主机记录www的结果是：[www.hzznb-xzll.xyz](https://link.juejin.cn?target=http%3A%2F%2Fwww.hzznb-xzll.xyz "http://www.hzznb-xzll.xyz") ，@的结果是 hzznb-xzll.xyz， \*的意思为泛解析，对应的结果是 xxx.hzznb-xzll.xyz）： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a0f2d211b5c4452da0831d377f1be81b~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3472&h=896&s=230996&e=png&b=fff9f9) 配置好解析后，我们开始申请一个免费的ssl证书并将其和我上边的域名绑定，如下： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a163ba2bd35488f82d3f4bf65b7c72e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3366&h=1708&s=454108&e=png&b=ffffff) 提交申请后，因为我们选择的手动DNS验证，所以接下来按照人家的提示手动配置（这个操作比较重要，不做这一步，证书验证肯定过不去，所以必须做并且做对）： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/54cade17f8ae4bac98ab76f024d36669~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3484&h=1754&s=812079&e=png&b=fef9f9) 之后我们在我的证书看到已经完成验证，此时就可以下载ssl证书，然后配置我们的nginx了： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/978f4c76529843e3bfb542e0b7121c4b~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3350&h=1228&s=266165&e=png&b=ffffff) 因为我们接下来要配置到nginx反向代理服务器，所以这里选择下载nginx类型的ssl证书： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/acb37e756e7d4cf8b3b8b2a485f38d64~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3122&h=1612&s=420025&e=png&b=323232)

13.3、上传并配置nginx以及演示
-------------------

下载到本地后是个zip我们解压之后会看到里边有4个文件分别是：  

hzznb-xzll.xyz\_bundle.crt 证书文件  
hzznb-xzll.xyz\_bundle.pem 证书文件（可忽略该文件）  
hzznb-xzll.xyz.key 私钥文件  
hzznb-xzll.xyz.csr CSR 文件 （CSR 文件是申请证书时由您上传或系统在线生成的，提供给 CA 机构。安装时可忽略该文件。）

之后我们仅需要把 hzznb-xzll.xyz.key 和 hzznb-xzll.xyz\_bundle.crt 这俩货上传到我新建的 certificate文件夹：/usr/local/nginx/certificate/ ，操作如下： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/651a45d41f244c04b7fe4d8ee7f68c6c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2230&h=538&s=118121&e=png&b=010101) 上传完成： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/29eb6206483c42b98eceb87ccb8ccbe3~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1138&h=194&s=39891&e=png&b=010101) ssl证书准备好后，我们需要配置一个https的server（如下配置：)  
下边的指令名称都有注释说明了各个指令是干啥的，我也就不啰嗦了：

```ini
# --------------------HTTPS 配置---------------------
    server {
        #SSL 默认访问端口号为 443
        listen 443 ssl; 
        #填写绑定证书的域名 
        server_name www.hzznb-xzll.xyz hzznb-xzll.xyz; 
        #请填写证书文件的相对路径或绝对路径
        ssl_certificate /usr/local/nginx/certificate/hzznb-xzll.xyz_bundle.crt; 
        #请填写私钥文件的相对路径或绝对路径
        ssl_certificate_key /usr/local/nginx/certificate/hzznb-xzll.xyz.key; 
        #停止通信时，加密会话的有效期，在该时间段内不需要重新交换密钥
        ssl_session_timeout 5m;
        #服务器支持的TLS版本
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; 
        #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; 
        #开启由服务器决定采用的密码套件
        ssl_prefer_server_ciphers on;
    }    

```



可以看到我们上传到 /usr/local/nginx/certificate/目录下的 .crt和.key文件被使用到了。

现在我们仅仅是配好一个https类型的server，光一个server没法访问也没意思，我们需要让给他配置上游服务以及路由规则，这里我们直接使用我们上边的 80端口那个server中的location配置，直接copy过来，**https server的配置** 如下： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1e5e659b5d504b73947c414db801d5a0~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1580&h=2200&s=481554&e=png&b=010101) 到这里，就可以通过https方式访问我们的页面和接口了。但是需要注意的是，由于我们的协议和域名换了，所以index\_page.html里边的接口地址也得变了，如下： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/469223d90de34cb8b81e12098fbecef3~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2058&h=1856&s=417514&e=png&b=1f1f1f) 之后我们将index\_page.html上传到nginx的 /usr/local/nginx/test/static/ 目录，并重启nginx（nginx -s reload） 之后在浏览器访问试试，

> 注意：我最开始在配置dns解析时记录值配的是公网ip,但是我发现好像不好使，总是连接失败： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b247a65f19ec4f4e8c2bfb6fcc4b2ea6~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2596&h=1678&s=253672&e=png&b=fff8f8) ，后来改成 局域网ip 172.30.128.65 发现可行了 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b89c7e7cd4d14d919bf2c81ecaee5df3~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2884&h=836&s=187153&e=png&b=ffffff)

修改云解析dns记录值为172.30.128.65后，访问效果如下： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/93b1857429404af3bc4fe460916175c2~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3522&h=1382&s=589204&e=png&b=ffffff) ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a793e9c760a4cf58245c4279c998d75~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3548&h=1706&s=836338&e=png&b=ffffff) 从上边可以看到，我们可以通过https方式访问前端页面和后台接口了。

13.4、http跳转https
----------------

一般情况下为了安全都不使用http方式访问页面or后台服务，但是有的人就是喜欢使用http访问怎么办？好说，我给http加个跳转，你访问 [xxx.com](https://link.juejin.cn?target=http%3A%2F%2Fxxx.com "http://xxx.com") 我给你跳转到[xxx.com](https://link.juejin.cn?target=https%3A%2F%2Fxxx.com "https://xxx.com"), 想达到此效果需要先修改下nginx.conf文件：

```perl
server_name www.hzznb-xzll.xyz hzznb-xzll.xyz;
# 重定向到目标地址
return 301 https://$server_name$request_uri;

```



![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e36c7f2dee244e5789942330d911a1d1~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1488&h=1646&s=304188&e=png&b=010101) 之后重启nginx看下演示效果：

首先请求 [www.hzznb-xzll.xyz/page/](https://link.juejin.cn?target=http%3A%2F%2Fwww.hzznb-xzll.xyz%2Fpage%2F "http://www.hzznb-xzll.xyz/page/") 会返回重定向地址，让你重新定向到 [www.hzznb-xzll.xyz/page/](https://link.juejin.cn?target=https%3A%2F%2Fwww.hzznb-xzll.xyz%2Fpage%2F "https://www.hzznb-xzll.xyz/page/") ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3ff97704505f48ecba42fcf55f1e4544~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3460&h=1616&s=1289722&e=png&b=fbfbfb) 请求（这一步浏览器会自动发起请求，无需手动点击或刷新啥的）定向后的目标地址： [www.hzznb-xzll.xyz/page/](https://link.juejin.cn?target=https%3A%2F%2Fwww.hzznb-xzll.xyz%2Fpage%2F "https://www.hzznb-xzll.xyz/page/") ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7dd23f4dc3634234b4385e17fd3e4bd0~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3486&h=1732&s=876098&e=png&b=fbfbfb) 可以看到http方式的请求被成功重定向到了https server(443端口对应的server)。

14、压缩
=====

压缩功能比较实用尤其是处理一些大文件时，而gzip 是规定的三种标准 HTTP 压缩格式之一。目前绝大多数的网站都在使用 gzip 传输 HTML 、CSS 、 JavaScript 等资源文件。需要知道的是，并不是每个浏览器都支持 gzip 压缩，如何知道客户端（浏览器）是否支持 压缩 呢？ 可以通过观察 某请求头中的 Accept-Encoding 来观察是否支持压缩，另外只有客户端支持也不顶事，服务端得返回gzip格式的文件呀，那么这件事nginx可以帮我们做，我们可以通过 Nginx 的配置来让服务端支持 gzip。服务端返回压缩文件后浏览器进行解压缩从而展示正常内容。

14.1、压缩前
--------

首先应该明确的是我当前是没开启压缩的： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/77321f75b8e04d72bb2740eb3e8b8dca~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2450&h=1244&s=169679&e=png&b=000000)

其次为了方便看出效果，我们先将之前那个index\_page.html文件加点图片，加点文字给他的文件大小弄大点，如下： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5453159a8ab14b3ab35bc55aa751a388~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2358&h=1770&s=385432&e=png&b=000000) 这里无需重启nginx，看下**没开启压缩** 时候的效果： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8f57509719384a728c1a61ab87600786~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3470&h=1664&s=2868882&e=png&b=fbf9f9) ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a16001d9150b4ad3a37db56f1a7e21e5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3520&h=1728&s=3000578&e=png&b=f7f5f5)

14.2、压缩后
--------

想要压缩就得配置nginx,我们修改nginx.conf文件，在http指令块添加如下内容：

```shell
http {
    # 开启/关闭 压缩机制
    gzip on;
    # 根据文件类型选择 是否开启压缩机制
    gzip_types text/plain application/javascript text/css application/xml text/javascript image/jpeg image/jpg image/gif image/png  application/json;
    # 设置压缩级别，一共9个级别  1-9   ，越高资源消耗越大 越耗时，但压缩效果越好，
    gzip_comp_level 9;
    # 设置是否携带Vary:Accept-Encoding 的响应头
    gzip_vary on;
    # 处理压缩请求的 缓冲区数量和大小
    gzip_buffers 32 64k;
    # 对于不支持压缩功能的客户端请求 不开启压缩机制
    gzip_disable "MSIE [1-6]\."; # 比如低版本的IE浏览器不支持压缩
    # 设置压缩功能所支持的HTTP最低版本
    gzip_http_version 1.1;
    # 设置触发压缩的最小阈值
    gzip_min_length 2k;
    # off/any/expired/no-cache/no-store/private/no_last_modified/no_etag/auth 根据不同配置对后端服务器的响应结果进行压缩
    gzip_proxied any;
} 

```



几个指令的作用在注释中写明了这里不再过多解释。接下来我们重启nginx，然后看下压缩前后的效果：

html文件压缩： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/77eb77074edc4c6ab170d2cf3915496f~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3478&h=1690&s=1190643&e=png&b=fefefe) 接口响应压缩： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/03f12df6d0a947cb8eb9a63b8032a8d6~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3512&h=1640&s=3327894&e=png&b=f9f7f7) 可以看到不管是html还是接口响应数据， 压缩后的体积变得非常小了，压缩的效果还是不错的，但是值得注意的是压缩后虽然体积变小了，但是响应的时间会变长，因为压缩/解压也需要时间呀！压缩功能似乎有点：**用时间换空间的感觉！**，当然压缩级别可以调的，你可以选择较低级别的压缩，这样既能实现压缩功能使得数据包体积降下来，同时压缩时间也会缩短是比较折中的一种方案（我在演示时为了效果，配置的压缩级别是9 ，一共9个级别， 9是最高级别的压缩等级）。

15、其他一些比较常用的指令与说明
=================

关于nginx的指令其实太多了，有些常用的指令不说一下的话，有时候遇见了不懂啥意思，所以这里说一下nginx几个比较常用的指令（上边nginx.conf文件解读 以及某些小节中已经说了很多指令了，这里也不管重不重复吧，说明几个我觉得有必要讲的几个）

15.1、rewrite
------------

rewrite指令是通过正则表达式来改变URI。可以同时存在一个或多个指令。需要按照顺序依次对URL进行匹配和处理，常用于重定向功能。

rewrite语法如下：

| 语法: | `rewrite 正则表达式 要替换的内容 [flag];` |
| --- | --- |
| 默认: | — |
| 作用域: | `server`, `location`, `if` |

其中flag有如下几个值：  

**last:**  本条规则匹配完成后，继续向下匹配新的location URI 规则。  
**break:**  本条规则匹配完成即终止，不再匹配后面的任何规则。  
**redirect:**  返回302临时重定向，浏览器地址会显示跳转新的URL地址。  
**permanent:**  返回301永久重定向。浏览器地址会显示跳转新的URL地址。

下边我们演示下四种重写的效果 首先修改nginx.conf文件：

```ini
  server {
      listen 80 default;
      charset utf-8;
      server_name www.hzznb-xzll.xyz hzznb-xzll.xyz;

      # 临时（redirect）重定向配置
      location /temp_redir {
          rewrite ^/(.*) https://www.baidu.com redirect;
      }
      # 永久重定向（permanent）配置
      location /forever_redir {

          rewrite ^/(.*) https://www.baidu.com permanent;
      }

      # rewrite last配置
      location /1 {
        rewrite /1/(.*) /2/$1 last;
      }
      location /2 {
        rewrite /2/(.*) /3/$1 last;
      }
      location /3 {
        alias  '/usr/local/nginx/test/static/';
        index location_last_test.html;
      }
    }

```



![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/55494f7625da405192cdd9b1abdf29ce~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1820&h=1228&s=257959&e=png&b=000000)

**last配置:** 可以看到我们定义 访问 [hzznb-xzll.xyz/1/](https://link.juejin.cn?target=http%3A%2F%2Fhzznb-xzll.xyz%2F1%2F "http://hzznb-xzll.xyz/1/") 的请求被替换为 [hzznb-xzll.xyz/2/](https://link.juejin.cn?target=http%3A%2F%2Fhzznb-xzll.xyz%2F2%2F "http://hzznb-xzll.xyz/2/") 之后再被替换为 [hzznb-xzll.xyz/3/](https://link.juejin.cn?target=http%3A%2F%2Fhzznb-xzll.xyz%2F3%2F "http://hzznb-xzll.xyz/3/") 最后找到/usr/local/nginx/test/static/location\_last\_test.html 这个文件并返回。  
**redirect配置：** 当访问 [hzznb-xzll.xyz/temp\_redir/](https://link.juejin.cn?target=http%3A%2F%2Fhzznb-xzll.xyz%2Ftemp_redir%2F "http://hzznb-xzll.xyz/temp_redir/") 这个请求会临时（302）重定向到百度页面  
**permanent配置：** 当访问 [hzznb-xzll.xyz/forever\_red…](https://link.juejin.cn?target=http%3A%2F%2Fhzznb-xzll.xyz%2Fforever_redir%2F "http://hzznb-xzll.xyz/forever_redir/") 这个请求会永久（301）重定向到百度页面

效果如下： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/27939667cad34a9583d79f8d2f37eff0~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3254&h=1778&s=1105567&e=png&b=fcfcfc)

15.2、if
-------

该指令用于条件判断，并且根据条件判断结果来选择不同的配置，其作用于为：server/location 块。这个指令比较简单，因为编程中if语句都是非常高频使用的，但是里边怎么写 就得说说nginx的全局变量了，因为我们很多时候，都是在对 比如：url 参数 ip 域名等等做比对或者判断（一般都使用正则的方式）,而这些都在nginx全局变量中可以拿到，比如下边这个if判断就用到了全局变量：

```bash
# 指定 username 参数中只要有字母 就不走nginx缓存
if ($arg_username ~ [a-z]) {
   set $cache_name "no cache";
}

```

正则表达式就不多说了，具体使用时查一下，我们下边列一下nginx的全局变量：

15.3、nginx全局变量
--------------

| 变量 | 解释 |
| --- | --- |
| $time\_local | 本地时间 |
| $time\_iso8601 | ISO 8601 时间格式 |
| $arg\_name | 请求中的的参数名，即“?”后面的arg\_name=arg\_value形式的arg\_name |
| $args | 与$query\_string相同 等于URL当中的参数(GET请求时)，如a=1&b=2 |
| $document\_uri | 与uri相同这个变量指当前的请求URI，不包括任何参数(见uri相同 这个变量指当前的请求URI，不包括任何参数(见uri相同这个变量指当前的请求URI，不包括任何参数(见args) |
| $request\_uri | 包含请求参数的原始URI，不包含主机名，如：/aaa/bbb.html?a=1&b=2 |
| $is\_args | 如果URL包含参数则为？，否则为空字符串 |
| $query\_string | 与$args相同 等于URL当中的参数(GET请求时)，如a=1&b=2 |
| $uri | 当前请求的URI,不包含任何参数 |
| $remote\_addr | 获取客户端ip |
| $binary\_remote\_addr | 客户端ip（二进制) |
| $remote\_port | 客户端port |
| $remote\_user | 用于基本验证的用户名。 |
| $host | 请求主机头字段，否则为服务器名称，如:[www.hzznb-xzll.xyz](https://link.juejin.cn?target=https%3A%2F%2Fwww.hzznb-xzll.xyz "https://www.hzznb-xzll.xyz") |
| $proxy\_host | proxy\_pass 指令设置的后端服务器的域名（或者IP地址） |
| $proxy\_port | proxy\_pass 指令设置的后端服务器的监听端口。 |
| $http\_host | 是host和host 和 host和server\_port 两个变量的结合 |
| $request | 用户请求信息，如：GET ?a=1&b=2 HTTP/1.1 |
| $request\_time | 请求所用时间，单位毫秒 |
| $request\_method | 请求的方法 比如 get post put delete update 等 |
| $request\_filename | 当前请求的文件的路径名，由root或alias和URI request组合而成,如：/aaa/bbb.html |
| $status | 请求的响应状态码,如：200 |
| $body\_bytes\_sent | 响应时送出的body字节数数量。即使连接中断，这个数据也是精确的,如：40，传输给客户端的字节数，响应头不计算在内；这个变量和Apache的mod\_log\_config模块中的“%B”参数保持兼容 |
| $content\_length | 等于请求行的“Content\_Length”的值 |
| $content\_type | 等于请求行的“Content\_Type”的值 |
| $http\_referer | 引用地址 |
| $http\_user\_agent | 客户端agent信息,如：Mozilla/5.0 (Macintosh; Intel Mac OS X 10\_15\_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.0.0 Safari/537.36 这个可以用来区分手机端还是pc端 |
| $document\_root | 针对当前请求的根路径设置值 |
| $hostname | 主机名称 |
| $http\_cookie | 客户端cookie信息 |
| $cookie\_COOKIE | cookie COOKIE变量的值 |
| $limit\_rate | 这个变量可以限制连接速率，0表示不限速 |
| $request\_body | 记录POST过来的数据信息 |
| $request\_body\_file | 客户端请求主体信息的临时文件名 |
| $scheme | HTTP方法（如http，https） |
| $request\_completion | 如果请求结束，设置为OK. 当请求未结束或如果该请求不是请求链串的最后一个时，为空(Empty)，如：OK |
| $server\_protocol | 请求使用的协议，通常是HTTP/1.0或HTTP/1.1，如：HTTP/1.1 |
| $server\_addr | 服务器IP地址，在完成一次系统调用后可以确定这个值 |
| $server\_name | 响应请求的服务器名称 |
| $server\_port | 请求到达服务器的端口号，如：80 |
| $connection | 连接序列号 |
| $connection\_requests | 当前通过连接发出的请求数 |
| $nginx\_version | nginx版本 |
| $pid | 工作进程的PID |
| $pipe | 如果请求来自管道通信，值为“p”，否则为“.” |
| $proxy\_protocol\_addr | 获取代理访问服务器的客户端地址，如果是直接访问，该值为空字符串 |
| $realpath\_root | 对应于当前请求的根目录或别名值的绝对路径名，所有符号连接都解析为真实路径。 |

可以看到全局变量比较多，但是没关系在使用时再去详查就好了。

15.4、auto\_index
----------------

一般用于快速搭建静态资源网站，比如我要给自己搞个书籍网站里边放些好书，以便需要时查看阅读，首先建一个book文件夹并往里放几本书： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/528eb4768338439dbbb70db277f4e070~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1574&h=276&s=103591&e=png&b=010101)

之后我们配置nginx， 使得访问 [hzznb-xzll.xyz/book/](https://link.juejin.cn?target=http%3A%2F%2Fhzznb-xzll.xyz%2Fbook%2F "http://hzznb-xzll.xyz/book/") 时,返回 /usr/local/nginx/test/book/目录下的书籍，配置如下：

vbnet

 代码解读

复制代码

      `location /book/ {           root /usr/local/nginx/test;           autoindex on; # 打开 autoindex，，可选参数有 on | off           autoindex_format html; # 以html的方式进行格式化，可选参数有 html | json | xml           autoindex_exact_size on; # 修改为off，会以KB、MB、GB显示文件大小，默认为on以bytes显示出⽂件的确切⼤⼩           autoindex_localtime off; # 显示⽂件时间 GMT格式       }`

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b44a714dcb3c43c78b3e95c5fe80b135~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1616&h=828&s=120547&e=png&b=010101)

重启nginx并在浏览器输入 [hzznb-xzll.xyz/book/](https://link.juejin.cn?target=http%3A%2F%2Fhzznb-xzll.xyz%2Fbook%2F "http://hzznb-xzll.xyz/book/") （注意book后边的斜线 / 不能去掉，否则404了，具体原因我们下边马上会说），看下效果： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8c6f9d133db64414ac7191663fcba115~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3530&h=1352&s=435859&e=png&b=ffffff)

15.5、root&alias
---------------

root和alias这俩货一般都是用于指定静态资源目录，但是还是有挺大区别的，虽然这是个小知识点但是如果你不清楚规则，很容易走弯路，所以这里阐明并演示这俩的区别。

### 15.5.1、root

说先说root：

| 语法: | `root path;` |
| --- | --- |
| 默认值: | `root html;` |
| 作用域: | `http`, `server`, `location`, `if in location` |

为请求设置根目录。例如，使用以下配置：

bash

 代码解读

复制代码

`location /static/ {     root /usr/local/nginx/test; }`

此时，当你请求 [www.hzznb-xzll.xyz/static/imag…](https://link.juejin.cn?target=http%3A%2F%2Fwww.hzznb-xzll.xyz%2Fstatic%2Fimage2.jpg "http://www.hzznb-xzll.xyz/static/image2.jpg") 时，/usr/local/nginx/test/static/image2.jpg 文件将被作为响应内容响应给客户端,也就是说 :  
**root指令会 将 /static/ 拼接到 /usr/local/nginx/test 后边  
即完整目录路径为： /usr/local/nginx/test/static/** 演示效果如下： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ea48800fdb14a909134a7b58a4df770~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3450&h=1912&s=1090431&e=png&b=fcfcfc)

### 15.5.2、alias

**alias** 中文意思别名，这个和root最大区别就是 **不会进行拼接**，下边我们改下nginx.conf文件来演示下：

bash

 代码解读

复制代码

`location /static { # 注意一般 alias的 url都不带后边的/      alias /usr/local/nginx/test/; # 使用alias时  这里的目录最后边一定要加/ 否则就404 }`

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/249189ce28f541e3838660b1b7fa0304~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1840&h=1052&s=176466&e=png&b=010101) 上边配置的意思就是当前你访问 [www.hzznb-xzll.xyz/static/imag…](https://link.juejin.cn?target=http%3A%2F%2Fwww.hzznb-xzll.xyz%2Fstatic%2Fimage2.jpg "http://www.hzznb-xzll.xyz/static/image2.jpg") 时，nginx会去alias配置的路径：/usr/local/nginx/test/目录下找 image2.jpg 这个文件从而返回。比如我现在的/usr/local/nginx/test/目录下没有image2.jpg 文件则返回404，如下： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f1ec27672ea1411eaacd9f29ddb11753~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3466&h=1592&s=681624&e=png&b=fafafa) 之后我们copy image2.jpg文件到 /usr/local/nginx/test/ 目录，然后看下效果： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f8edf38a338f46c18d475ccf878ac5b7~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3488&h=1642&s=1045818&e=png&b=fcfcfc) 可以看到成功返回了。

### 15.5.3、proxy\_pass 中的斜线与root和 alias的相似之处

在我们上边的负载均衡以及动静锋利等等演示中，可以看到我们的proxy\_pass的配置基本上都是这么配的：

arduino

 代码解读

复制代码

`proxy_pass http://mybackendserver/`

这里有个东西和root与alias的规则很像，所以我们在这里也提一下，就是 [http://mybackendserver/](https://link.juejin.cn?target=http%3A%2F%2Fmybackendserver%2F "http://mybackendserver/") 后边这个斜线 /，如果你不写 / 则会将location的url拼接到路径后边，如果你写了则不会，下边我们演示下这样更直观些。修改nginx.conf如下： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a7582c00a5314e2888ac4271628d0a3a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1818&h=1176&s=283414&e=png&b=010101) interface 这个location不会拼接 interface到 [http://mybackendserver/](https://link.juejin.cn?target=http%3A%2F%2Fmybackendserver%2F "http://mybackendserver/") 后边，如下： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ff6e676d27d403ca377fbe60999ab3c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3540&h=1626&s=694443&e=png&b=fafafa)

interface2 这个location **会拼接** interface2到 [http://mybackendserver/](https://link.juejin.cn?target=http%3A%2F%2Fmybackendserver%2F "http://mybackendserver/") 后边（从而导致接口404），如下： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/211632b276484ed2a2521dea765f3521~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3544&h=1556&s=416346&e=png&b=fbfbfb)

**看出来了吗，proxy\_pass 值后边加斜线和不加斜线 ，区别是很大的。不加斜线 有点像root（会拼接），加了斜线 有点像alias （不会进行 拼接）。要牢记这个事情。**

15.6、upstream 中常用的几个指令
----------------------

在upstream中，有些指令也是比较常用的所以我们这里也列一下：

| 参数 | 描述 |
| --- | --- |
| server | 反向服务地址 |
| weight | 权重 |
| fail\_timeout | 与max\_fails结合使用。 |
| max\_fails | 设置在fail\_timeout参数设置的时间内最大失败次数，如果在这个时间内，所有针对该服务器的请求都失败了，那么认为该服务器会被认为是停机了。 |
| max\_conns | 允许最大连接数 |
| fail\_time | 服务器会被认为停机的时间长度,默认为10s |
| backup | 标记该服务器为备用服务器，当主服务器停止时，请求会被发送到它这里。 |
| down | 标记服务器永久停机了 |
| slow\_start | 当节点恢复，不立即加入 |

16、重试策略
=======

16.1、服务不可用重试
------------

关于重试策略我们这里也说一下，重试是在发生错误时的一种不可缺少的手段，这样当某一个或者某几个服务宕机时（因为我们现在大多都是多实例部署），如果有正常服务，那么将请求 重试到正常服务的机器上去。

下边我们先修改下nginx.conf文件：

ini

 代码解读

复制代码

    `upstream mybackendserver {         # 60秒内 如果请求8081端口这个应用失败         # 3次，则认为该应用宕机 时间到后再有请求进来继续尝试连接宕机应用且仅尝试 1 次，如果还是失败，         # 则继续等待 60 秒...以此循环，直到恢复         server 172.30.128.64:8081 fail_timeout=60s max_fails=3;                   server 172.30.128.64:8082;         server 172.30.128.64:8083;     }`

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6183c53597d14a5bb8d8a35e46e9c5e7~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3466&h=1324&s=802033&e=png&b=020202)

16.2、错误重试
---------

错误重试是你可以配置哪些状态下 才会执行重试，比如如下这个配置：

bash

 代码解读

复制代码

 `# 指定哪些错误状态才执行 重试 比如下边的 error 超时，500,502,503 504 proxy_next_upstream error timeout http_500 http_502 http_503 http_504;`

接下来我们使用idea启动8082端口，然后打个断点让8082这个服务超时（验证下超时重试），效果如下： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0c5a45d2a9924e258417ca3ebff16556~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3184&h=1570&s=641853&e=png&b=fbfbfb) 可以看到8082因为超时，重试给8081 然后8081也不可用重试给8083 最终8083返回数据。

16.3、关于backup
-------------

Nginx 支持设置备用节点，当所有线上节点都异常时会启用备用节点，同时备用节点也会影响到失败 重试的逻辑。

我们可以通过 backup 指令来定义备用服务器，backup有如下特征：

1.  正常情况下，请求不会转到到 backup 服务器，包括失败重试的场景
2.  当所有正常节点全部不可用时，backup 服务器生效，开始处理请求
3.  一旦有正常节点恢复，就使用已经恢复的正常节点
4.  backup 服务器生效期间，不会存在所有正常节点一次性恢复的逻辑
5.  如果全部 backup 服务器也异常，则会将所有节点一次性恢复，加入存活列表
6.  如果全部节点（包括 backup）都异常了，则 Nginx 返回 502 错误

接着我们修改下nginx.conf文件演示下backup的作用：

ini

 代码解读

复制代码

`upstream mybackendserver {     server 172.30.128.64:8081 fail_timeout=60s max_fails=3; # 60秒内 如果请求某一个应用失败3次，则认为该应用宕机 时间到后再有请求进来继续尝试连接宕机应用且仅尝试 1 次，如果还是失败，则继续等待 60 秒...以此循环，直到恢复     server 172.30.128.64:8082;     server 172.30.128.64:8083 backup; # 设置8083位备机 }`

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f7717f7d6f1475cbbf666f5243dbe98~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3142&h=472&s=100351&e=png&b=000000)

之后我们启动8081 8082 8083 三个服务，然后先停掉8081 再停掉8082 看看效果：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/55b3edda22b947018964194635c76eec~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3262&h=1764&s=767075&e=png&b=020202) 可以看到即使8081 不可用也只是去8082重试而不会到备机重试，如果8081 8082都不可用则请求重试到备机：8083

17、最后
=====

为了方便粘贴以及观察，这里贴出我机器上完整的 nginx.conf文件，如下：

17.1、贴出完整nginx.conf文件
---------------------

```shell
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # log_format  debug  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';
    #
    log_format  debug  ' $remote_user [$time_local]  $http_x_Forwarded_for $remote_addr  $request '  
                      '$http_x_forwarded_for '  
                      '$upstream_addr '  
                      'ups_resp_time: $upstream_response_time '  
                      'request_time: $request_time'; 

    access_log  /var/log/nginx/access.log  debug;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    upstream mybackendserver {
        server 172.30.128.64:8081 fail_timeout=60s max_fails=3; # 60秒内 如果请求某一个应用失败3次，则认为该应用宕机 时间到后再有请求进来继续尝试连接宕机应用且仅尝试 1 次，如果还是失败，则继续等待 60 秒...以此循环，直到恢复
        server 172.30.128.64:8082;
        server 172.30.128.64:8083 backup; # 设置8083位备机
    }

     # 开启/关闭 压缩机制
    gzip on;
    # 根据文件类型选择 是否开启压缩机制
    gzip_types text/plain application/javascript text/css application/xml text/javascript image/jpeg image/jpg image/gif image/png  application/json;
    # 设置压缩级别，越高资源消耗越大越耗时，但压缩效果越好
    gzip_comp_level 9;
    # 设置是否携带Vary:Accept-Encoding 的响应头
    gzip_vary on;
    # 处理压缩请求的 缓冲区数量和大小
    gzip_buffers 32 64k;
    # 对于不支持压缩功能的客户端请求 不开启压缩机制
    gzip_disable "MSIE [1-6]\."; # 比如低版本的IE浏览器不支持压缩
    # 设置压缩功能所支持的HTTP最低版本
    gzip_http_version 1.1;
    # 设置触发压缩的最小阈值
    gzip_min_length 2k;
    # off/any/expired/no-cache/no-store/private/no_last_modified/no_etag/auth 根据不同配置对后端服务器的响应结果进行压缩
    gzip_proxied any;

    # 指定缓存存放目录为/usr/local/nginx/test/nginx_cache_storage，并设置缓存名称为mycache，大小为64m， 1天未被访问过的缓存将自动清除，磁盘中缓存的最大容量为1gb
    proxy_cache_path /usr/local/nginx/test/nginx_cache_storage levels=1:2 keys_zone=mycache:64m inactive=1d max_size=1g;
    # 对请求速率限流
    #limit_req_zone $binary_remote_addr zone=myRateLimit:10m rate=5r/s;
    # 对请求连接数限流
    limit_conn_zone $binary_remote_addr zone=myConnLimit:10m; 
    
    # --------------------HTTP 演示 配置---------------------
    server {
      listen 80 default;
      charset utf-8;
      server_name www.hzznb-xzll.xyz hzznb-xzll.xyz;

      #location /static/ {
      #    root /usr/local/nginx/test;  # /usr/local/nginx/test/static/xxx.jpg
      #}

      location /static { # 注意一般 alias的 url都不带后边的/ 
          alias /usr/local/nginx/test/; # 使用alias时  这里的目录最后边一定要加/ 否则就404
      }
      # 测试autoindex效果
      location /book/ {
          root /usr/local/nginx/test;
          autoindex on; # 打开 autoindex，，可选参数有 on | off
          autoindex_format html; # 以html的方式进行格式化，可选参数有 html | json | xml
          autoindex_exact_size on; # 修改为off，会以KB、MB、GB显示文件大小，默认为on以bytes显示出⽂件的确切⼤⼩
          autoindex_localtime off; # 显示⽂件时间 GMT格式
      }
      
      # 临时重定向
      location /temp_redir {
          rewrite ^/(.*) https://www.baidu.com redirect;
      }
      # 永久重定向
      location /forever_redir {
          
          rewrite ^/(.*) https://www.baidu.com permanent;
      }
      # rewrite last规则测试
      location /1 {
        rewrite /1/(.*) /2/$1 last;
      }
      location /2 {
        rewrite /2/(.*) /3/$1 last;
      }
      location /3 {
        alias  '/usr/local/nginx/test/static/';
        index location_last_test.html; 
      }
    }

    # --------------------HTTP配置---------------------
    server {
      listen 80;
      charset utf-8;
      #server_name www.xxxadminsystem.com;
      server_name www.hzznbc-xzll.xyz hzznbc-xzll.xyz;
      # 重定向，会显示跳转的地址server_name,如果访问的地址没有匹配会默认使用第一个，即www.likeong.icu
      return 301 https://$server_name$request_uri;

        # # 指定 username 参数中只要有字母 就不走nginx缓存  
        # if ($arg_username ~ [a-z]) {
        #     set $cache_name "no cache";
        # }
        # # 前端页面资源
        # location  /page {
        #   alias  '/usr/local/nginx/test/static/';
        #   index index_page.html; 

        #   allow all;
        # }
        # # 后端服务
        # location  /interface {
        #   proxy_pass http://mybackendserver/;
        #   # 使用名为 mycache 的缓存空间
        #   proxy_cache mycache;
        #   # 对于200 206 状态码的数据缓存2分钟
        #   proxy_cache_valid 200 206 1m;
        #   # 定义生成缓存键的规则（请求的url+参数作为缓存key）
        #   proxy_cache_key $host$uri$is_args$args;
        #   # 资源至少被重复访问2次后再加入缓存
        #   proxy_cache_min_uses 3;
        #   # 出现重复请求时，只让其中一个去后端读数据，其他的从缓存中读取
        #   proxy_cache_lock on;
        #   # 上面的锁 超时时间为4s，超过4s未获取数据，其他请求直接去后端
        #   proxy_cache_lock_timeout 4s;
        #   # 对于请求参数中有字母的 不走nginx缓存
        #   proxy_no_cache $cache_name; # 判断该变量是否有值，如果有值则不进行缓存，没有值则进行缓存
        #   # 在响应头中添加一个缓存是否命中的状态（便于调试）
        #   add_header Cache-status $upstream_cache_status;
          
        #   limit_conn myConnLimit 12;

        # limit_req zone=myRateLimit burst=5  nodelay;
        # limit_req_status 520;
        # limit_req_log_level info;
        #}  
    }

    # --------------------HTTPS 配置---------------------
    server {
        #SSL 默认访问端口号为 443
        listen 443 ssl; 
        #填写绑定证书的域名 
        server_name www.hzznb-xzll.xyz hzznb-xzll.xyz; 
        #请填写证书文件的相对路径或绝对路径
        ssl_certificate /usr/local/nginx/certificate/hzznb-xzll.xyz_bundle.crt; 
        #请填写私钥文件的相对路径或绝对路径
        ssl_certificate_key /usr/local/nginx/certificate/hzznb-xzll.xyz.key; 
        #停止通信时，加密会话的有效期，在该时间段内不需要重新交换密钥
        ssl_session_timeout 5m;
        #服务器支持的TLS版本
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; 
        #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; 
        #开启由服务器决定采用的密码套件
        ssl_prefer_server_ciphers on;
        
        # 指定 username 参数中只要有字母 就不走nginx缓存  
        if ($arg_username ~ [a-z]) {
            set $cache_name "no cache";
        }
        # 前端页面资源
        location  /page {
          alias  '/usr/local/nginx/test/static/';
          index index_page.html; 

          allow all;
        }
        # 后端服务
        location  /interface {
          proxy_pass http://mybackendserver/;

          # 指定哪些错误状态才执行 重试
          proxy_next_upstream error timeout http_500 http_502 http_503 http_504 http_404;


          # 使用名为 mycache 的缓存空间
          proxy_cache mycache;
          # 对于200 206 状态码的数据缓存2分钟
          proxy_cache_valid 200 206 1m;
          # 定义生成缓存键的规则（请求的url+参数作为缓存key）
          proxy_cache_key $host$uri$is_args$args;
          # 资源至少被重复访问2次后再加入缓存
          proxy_cache_min_uses 3;
          # 出现重复请求时，只让其中一个去后端读数据，其他的从缓存中读取
          proxy_cache_lock on;
          # 上面的锁 超时时间为4s，超过4s未获取数据，其他请求直接去后端
          proxy_cache_lock_timeout 4s;
          # 对于请求参数中有字母的 不走nginx缓存
          proxy_no_cache $cache_name; # 判断该变量是否有值，如果有值则不进行缓存，没有值则进行缓存
          # 在响应头中添加一个缓存是否命中的状态（便于调试）
          add_header Cache-status $upstream_cache_status;
          
          limit_conn myConnLimit 12;

        # limit_req zone=myRateLimit burst=5  nodelay;
        # limit_req_status 520;
        # limit_req_log_level info;
        } 

        # 后端服务
        location  /interface2 {
          proxy_pass http://mybackendserver;
          # 使用名为 mycache 的缓存空间
          proxy_cache mycache;
          # 对于200 206 状态码的数据缓存2分钟
          proxy_cache_valid 200 206 1m;
          # 定义生成缓存键的规则（请求的url+参数作为缓存key）
          proxy_cache_key $host$uri$is_args$args;
          # 资源至少被重复访问2次后再加入缓存
          proxy_cache_min_uses 3;
          # 出现重复请求时，只让其中一个去后端读数据，其他的从缓存中读取
          proxy_cache_lock on;
          # 上面的锁 超时时间为4s，超过4s未获取数据，其他请求直接去后端
          proxy_cache_lock_timeout 4s;
          # 对于请求参数中有字母的 不走nginx缓存
          proxy_no_cache $cache_name; # 判断该变量是否有值，如果有值则不进行缓存，没有值则进行缓存
          # 在响应头中添加一个缓存是否命中的状态（便于调试）
          add_header Cache-status $upstream_cache_status;
          
          limit_conn myConnLimit 12;

        # limit_req zone=myRateLimit burst=5  nodelay;
        # limit_req_status 520;
        # limit_req_log_level info;
        } 

    }

    # include /etc/nginx/conf.d/*.conf;
}


```

17.2、个人絮叨
---------

这篇文章写了前后有20天，中间遇到很多的问题，都一一解决了，到这里我感到收获颇多，因为我之前对nginx说实话了解的并不多，这篇文章也基本补齐了对nginx的认识和使用。另外值的一说的是，nginx可不止本文这些东西，功能海了去了，更多的请查阅nginx官网，里边每一个模块，每一个指令以及作用都说的清清楚楚，官网戳这里：[nginx官网文档](https://link.juejin.cn?target=https%3A%2F%2Fnginx.org%2Fen%2Fdocs%2F "https://nginx.org/en/docs/") 。

* * *

如果本文对你有帮助或启发请帮忙点个赞👍 👍 ，谢谢😋😋~~

本文转自 <https://juejin.cn/post/7306041273822527514>，如有侵权，请联系删除。