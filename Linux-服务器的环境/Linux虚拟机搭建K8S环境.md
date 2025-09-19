 

#### 文章目录

*   [一、环境准备](#_1)
*   [二、系统初始化](#_7)
*   [三、部署master](#master_67)
*   [四、添加node节点](#node_89)
*   [五、部署网络](#_116)
*   [六、部署dashboard](#dashboard_186)
*   [七、登录dashboard面板](#dashboard_256)
*   [八、常见问题](#_272)

一、环境准备
------

首先在[vmware](https://so.csdn.net/so/search?q=vmware&spm=1001.2101.3001.7020)上新建4台相同配置的虚拟机，除了IP和主机名外，其余配置相同。由于是搭建K8S初始环境，没有跑其他服务，因此配置不高，1核CPU，2G内存，20G磁盘。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/12e39241a1ed42c7932ddbd500fd422c.png)

二、系统初始化
-------

下列操作在所有主机执行

```bash
# 关闭swap
swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab

# 配置系统参数
echo "net.bridge.bridge-nf-call-iptables=1" >> /etc/sysctl.conf
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables=1" >> /etc/sysctl.conf
sysctl -p

# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 永久关闭selinux
sed -i 's/enforcing/disabled/g' /etc/selinux/config

# 配置hosts解析
echo "192.168.0.201 k8s-master" >> /etc/hosts
echo "192.168.0.211 k8s-node1" >> /etc/hosts
echo "192.168.0.212 k8s-node2" >> /etc/hosts
echo "192.168.0.213 k8s-node3" >> /etc/hosts

# 配置K8s源
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 安装docker-ce
yum install -y docker-ce
systemctl start docker
systemctl enable docker

#安装kubelet、kubeadm和kubectl
yum install -y kubelet-1.23.0 kubeadm-1.23.0 kubectl-1.23.0
#设置kubelet开机自启
systemctl enable kubelet

#查看docker版本信息
docker --version
```

确认[k8s](https://so.csdn.net/so/search?q=k8s&spm=1001.2101.3001.7020)相关软件安装完成。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/542b9d70daea41f8bb73aa32e4a06492.png)

确认docker版本

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/5e9640c899924901ba9ec99467d8605a.png)

三、部署master
----------

以下操作在master节点执行

```bash
# master初始化
kubeadm init --apiserver-advertise-address=192.168.0.201 --image-repository=registry.aliyuncs.com/google_containers --kubernetes-version v1.23.0 --service-cidr=10.10.10.0/24 --pod-network-cidr=10.20.20.0/24 --ignore-preflight-errors=all
```

此处需要注意的是，参数 `--apiserver-advertise-address` 表示master节点的IP，根据实际情况填写。需要等一段时间，该初始化命令才会执行完。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7f149928536749f2b0f7ab87536cc038.png)

```shell
父节点执行初始化 可能存在报错，报错信息如下：
The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp [::1]:10248: connect: connection refused.
报错原因：
docker的cgroup驱动程序默认设置为system。默认情况下Kubernetes cgroup为systemd，我们需要更改Docker cgroup驱动

解决方式：
1、
先执行命令： kubeadm reset -f
然后执行命令： iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X

2）重新更换docker驱动
vim /etc/docker/daemon.json
将下面的文本粘贴到 daemon.json里
{
  "registry-mirrors": ["https://82m9ar63.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
然后将容器分别重启
systemctl daemon-reload
systemctl restart docker
```



然后拷贝k8s认证文件

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

四、添加node节点
----------

以下操作在3个node节点主机操作

```bash
kubeadm join 192.168.0.201:6443 --token g777be.ntr54x4ws0b17xm3 --discovery-token-ca-cert-hash sha256:f4df1236793fa3f3c09c3678461445466fc302a2d88b094ad9e31795297ce5e3
```

添加node节点的命令，就是上一步master初始化后生成的命令。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7d93938521f048dcb7cece9ddd847c9d.png)

需要注意的是，这个token的有效期只有24小时，过期就不可用，以下是重新创建token的命令。

```bash
# token到期后，在master节点执行
kubeadm token create --print-join-command
```

检查node节点是否加入k8s集群，此步骤要在master节点操作

```bash
kubectl get nodes

备注:执行上面的命令可能会出错，如
The connection to the server localhost:8080 was refused - did you specify the right host or port?
原因：出现这个问题的原因是kubectl命令需要使用kubernetes-admin来运行，解决方法如下，将主节点中的【/etc/kubernetes/admin.conf】文件拷贝到从节点相同目录下，然后配置环境变量：
然后在主节点和几个从节点分别配置下环境变量
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
立即生效
source ~/.bash_profile
接着再运行kubectl命令就OK了


```

可以看出集群有1个master和3个node，节点状态是NotReady，这是正常的。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b21b218e060c4f61b83b86e4520d7505.png)

五、部署网络
------

以下操作在master节点执行

下载 calico.yaml 的yaml配置文件

```bash
wget https://docs.projectcalico.org/manifests/calico.yaml --no-check-certificate
```

修改里面定义Pod网络（CALICO\_IPV4POOL\_CIDR），与之前kubeadm init的 --pod-network-cidr指定的一样。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/44391e4899f940b6a5309d4b16c6188a.png)

修改国外镜像源，使镜像从国内镜像加速站点下载。

```bash
# 检查镜像源
cat calico.yaml | grep 'image:'
```

配置文件中默认是国外镜像源。  
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/dfeb3076d1ed4ff99fe001e39fe01e4f.png)

一键替换为国内镜像加速站点。

```bash
sed -i 's#docker.io/calico/cni:v3.25.0#swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/cni:v3.25.0#' calico.yaml
sed -i 's#docker.io/calico/node:v3.25.0#swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/node:v3.25.0#' calico.yaml
sed -i 's#docker.io/calico/kube-controllers:v3.25.0#swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/kube-controllers:v3.25.0#' calico.yaml
```

操作后检查确认。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6f1a3e7e876a4f82aa5e8ce113296210.png)

部署calico网络服务

```bash
kubectl apply -f calico.yaml
```

如图所示。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7bcacc3e24104ae9ae93f865429a0540.png)

检查服务创建的状态，由于要拉取镜像到本地，所以服务创建需要一段时间。

```bash
kubectl get pods -n kube-system
```

如图部分pod还在初始化。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/e5676fab624e42c68e3870a81f9c1984.png)

如果想查看 `calico-node-297xl` 这个pod的具体状态信息，执行下列命令。

```bash
kubectl describe pods calico-node-297xl -n kube-system
```

会显示相关日志信息。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/941e9e2513664d418a2179e9880c3027.png)

如果想检查服务跑在哪个主机节点上，应执行下列命令。

```bash
kubectl get svc,pods -o wide -n kube-system
```

\-n 参数后面跟的是 kube-system 这个命名空间。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/905c4ee3280448f2bcdff936a4614c78.png)

六、部署dashboard
-------------

以下命令在master节点执行

下载dashboard的部署yaml文件，并修改名称

```bash
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.4.0/aio/deploy/recommended.yaml
mv recommended.yaml dashboard.yaml
```

修改配置文件，设置端口号和类型。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7d85c11c8de34a8ba07f1b80891a0242.png)

检查配置文件的镜像源。

```bash
grep "image:" dashboard.yaml
```

如图。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a48d49f1302d4ac8a1c6a89651bd9577.png)

执行下列命令。

```bash
sed -i 's#kubernetesui/dashboard:v2.4.0#swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/kubernetesui/dashboard:v2.7.0#' dashboard.yaml
sed -i 's#kubernetesui/metrics-scraper:v1.0.7#swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/kubernetesui/metrics-scraper:v1.0.8#' dashboard.yaml
```

更换为国内的镜像加速站点。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/5482205bee144e74902954aaf885a86b.png)

部署dashboard

```bash
kubectl apply -f dashboard.yaml
```

如图。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/e2d06ab44e884ccaa638e52335b15fd4.png)

检查服务状态。

```bash
kubectl get pods -n kubernetes-dashboard
```

如图，dashboard 服务部署完成。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b1c21035a8104cbda30b8901ad602715.png)

创建并绑定集群管理员角色。

```bash
# 创建用户
kubectl create serviceaccount dashboard-admin -n kube-system
# 用户授权
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
# 获取用户Token
kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
```

如图，用户创建成功。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/1194f6da38074d3c96b14ca45983b4f8.png)

注意保存好token，稍后登录dashboard时会输入该token

七、登录dashboard面板
---------------

打开一个浏览器，输入下列的IP+端口。

```bash
https://192.168.0.201:30001/
```

如图。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/bf43fb2b18bd4b0da7b37c2ff5da64f6.png)

至此，k8s集群环境搭建完成。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6413f5e3df3a4d9b9eb1424145c21e32.png)

八、常见问题
------

报错一：在node节点查看pod时，无法连接服务器。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/ec7268dfeab641aba8b8df5e9857fa1d.png)  
检查自己的 /etc/kubernetes/kubelet.conf 文件路径，写入 /etc/profile 文件并加载生效

```bash
ls /etc/kubernetes/kubelet.conf
echo "export KUBECONFIG=/etc/kubernetes/kubelet.conf" >> /etc/profile
source /etc/profile
```

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/79c3be5df1264b7e92205b39096ddfe0.png)  
接着检查pod

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/1658aad00aa34452810536804d0abfc7.png)

本文转自 <https://blog.csdn.net/oldboy1999/article/details/141714073>，如有侵权，请联系删除。