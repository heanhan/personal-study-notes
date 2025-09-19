 

在VUE安装依赖包时候，[npm](https://edu.csdn.net/cloud/sd_summit?utm_source=glcblog&spm=1001.2101.3001.7020) install任何都会出现对于[安装包](https://so.csdn.net/so/search?q=%E5%AE%89%E8%A3%85%E5%8C%85&spm=1001.2101.3001.7020)的400 Bad Request，更换镜像源也不起任何作用。

在网上找了很久，最后终于找到了解决方案：

控制台执行：

```bash
npm config get proxynpm config get https-proxy
```

 查看返回是否为null：

![](https://i-blog.csdnimg.cn/blog_migrate/8f2b26deaae099eb6aa463a21455212a.png)

我这里就有一个不是，所以导致了问题的出现。

直接执行这两个命令，把他们都改为空：

```bash
npm config set proxy null 
npm config set https-proxy null
```

再清理一下缓存：

```bash
 npm cache clean --force
```

 设置下载镜像为国内镜像：

```bash
npm config set registry https://registry.npmmirror.com
```

最后执行npm install即可成功安装所有依赖包。

本文转自 <https://blog.csdn.net/m0_54890506/article/details/135257913>，如有侵权，请联系删除。