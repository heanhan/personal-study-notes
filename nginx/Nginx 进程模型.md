### 1、Nginx 进程模型

#### 介绍：Nginx 

Nginx 是 master-worker 进程模型.

master 进程：主进程，做监视 worker 进程

worker 进程：工作进程，做响应的工作

可以通过配置文件进行配置 worker 进程数量，默认配置为 1，

下图是从百度截图  Master-worker 进程的工作流程示意图

![image-20201211153557947](/Users/zhaojh0912/Library/Application Support/typora-user-images/image-20201211153557947.png)

### 2、worker 进程工作原理

#### 2.1 Worker 进程抢占机制

如图所示，假设我们配置文件中，配置了三个 worker 进程，那么启动 nginx 时，master 会 fork 出三个worker 进程，如果现在客户端发起一个请求，那么三个 worker 进程会进行争锁（accept_mutex 锁）谁抢到是谁的。

![image-20201211155616965](/Users/zhaojh0912/Library/Application Support/typora-user-images/image-20201211155616965.png)

#### 2.2 异步阻塞特性

Nginx 是异步非阻塞，在 Linux 下，Nginx 使用 epol 的 I/O 多路复用的模型，每个 worker 可以同时处理多个 client 请求。

![image-20201211155834143](/Users/zhaojh0912/Library/Application Support/typora-user-images/image-20201211155834143.png)

默认conf 文件

```
events{
	#默认使用 epoll 模型
	use epoll
	#每个 worker 允许连接的最大数
	worker_connections 1024
}
```

### 3 配置

#### 3.1nginx.conf 配置结构

![image-20201211160132938](/Users/zhaojh0912/Library/Application Support/typora-user-images/image-20201211160132938.png)

#### 3.2 nginx.conf核心配置