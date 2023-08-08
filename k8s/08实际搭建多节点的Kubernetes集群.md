# 实际搭建多节点的Kubernetes集群

## 什么是 kubeadm

前面的几节课里我们使用的都是 minikube，它非常简单易用，不需要什么配置工作，就能够在单机环境里创建出一个功能完善的 Kubernetes 集群，给学习、开发、测试都带来了极大的便利。不过 minikube 还是太“迷你”了，方便的同时也隐藏了很多细节，离真正生产环境里的计算集群有一些差距，毕竟许多需求、任务只有在多节点的大集群里才能够遇到，相比起来，minikube 真的只能算是一个“玩具”。

那么，多节点的 Kubernetes 集群是怎么从无到有地创建出来的呢？Kubernetes 是很多模块构成的，而实现核心功能的组件像 apiserver、etcd、scheduler 等本质上都是可执行文件，所以也可以采用和其他系统差不多的方式，<font color='red'>使用 Shell 脚本或者 Ansible 等工具打包发布到服务器上。</font>不过 Kubernetes 里的这些组件的配置和相互关系实在是太复杂了，用 Shell、Ansible 来部署的难度很高，需要具有相当专业的运维管理知识才能配置、搭建好集群，而且即使这样，搭建的过程也非常麻烦。

为了简化 Kubernetes 的部署工作，让它能够更“接地气”，社区里就出现了一个专门用来在集群中安装 Kubernetes 的工具，名字就叫“kubeadm”，意思就是“Kubernetes 管理员”。

kubeadm，原理和 minikube 类似，也是用容器和镜像来封装 Kubernetes 的各种组件，但它的目标不是单机部署，而是要能够轻松地在集群环境里部署 Kubernetes，并且让这个集群接近甚至达到生产级质量。

而在保持这个高水准的同时，kubeadm 还具有了和 minikube 一样的易用性，只要很少的几条命令，如 init、join、upgrade、reset 就能够完成 Kubernetes 集群的管理维护工作，这让它不仅适用于集群管理员，也适用于开发、测试人员。

## 实验环境的架构是什么样的

在使用 kubeadm 搭建实验环境之前，我们先来看看集群的架构设计，也就是说要准备好集群所需的硬件设施。这里我画了一张系统架构图，图里一共有 3 台主机，当然它们都是使用虚拟机软件 VMWare 虚拟出来的，下面我来详细说明一下：

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202212/yyf5db64d398b4d5dyyd5e8e23ece53e.jpg)

所谓的多节点集群，要求服务器应该有两台或者更多，为了简化我们只取最小值，所以这个 Kubernetes 集群就只有两台主机，一台是 Master 节点，另一台是 Worker 节点。当然，在完全掌握了 kubeadm 的用法之后，你可以在这个集群里添加更多的节点。

- Master 节点需要运行 apiserver、etcd、scheduler、controller-manager 等组件，管理整个集群，所以对配置要求比较高，至少是 2 核 CPU、4GB 的内存。
- 而 Worker 节点没有管理工作，只运行业务应用，所以配置可以低一些，为了节省资源我给它分配了 1 核 CPU 和 1GB 的内存，可以说是低到不能再低了。

基于模拟生产环境的考虑，在 Kubernetes 集群之外还需要有一台起辅助作用的服务器。它的名字叫 Console，意思是控制台，我们要在上面安装命令行工具 kubectl，所有对 Kubernetes 集群的管理命令都是从这台主机发出去的。

这也比较符合实际情况，因为安全的原因，集群里的主机部署好之后应该尽量少直接登录上去操作。要提醒你的是，Console 这台主机只是逻辑上的概念，不一定要是独立，你在实际安装部署的时候完全可以复用之前 minikube 的虚拟机，或者直接使用 Master/Worker 节点作为控制台。这 3 台主机共同组成了我们的实验环境，所以在配置的时候要注意它们的网络选项，必须是在同一个网段。

## 安装前的准备工作

不过有了架构图里的这些主机之后，我们还不能立即开始使用 kubeadm 安装 Kubernetes，因为 Kubernetes 对系统有一些特殊要求，我们必须还要在 Master 和 Worker 节点上做一些准备。

这些工作的详细信息你都可以在 Kubernetes 的官网上找到，但它们分散在不同的文档里，比较凌乱，所以我把它们整合到了这里，包括改主机名、改 Docker 配置、改网络设置、改交换分区这四步。

### 一、节点主机名hostname不能重名

由于 Kubernetes 使用主机名来区分集群里的节点，所以每个节点的 hostname 必须不能重名。你需要修改“/etc/hostname”这个文件，把它改成容易辨识的名字，比如 Master 节点就叫 master，Worker 节点就叫 worker：

```bash
# 192.168.234.130
#修改为每台机器的hostname
hostnamectl set-hostname master
echo "127.0.0.1   $(hostname)" >> /etc/hosts
#查看hostname
hostname
#--------------------
# 192.168.234.131
#修改为每台机器的hostname
hostnamectl set-hostname worker
echo "127.0.0.1   $(hostname)" >> /etc/hosts
```

### 二、安装docker Engine

虽然 Kubernetes 目前支持多种容器运行时，但 Docker 还是最方便最易用的一种，所以我们仍然继续使用 Docker 作为 Kubernetes 的底层支持，使用 apt 安装 Docker Engine

安装完成后需要你再对 Docker 的配置做一点修改，在“/etc/docker/daemon.json”里把 cgroup 的驱动程序改成 systemd ，然后重启 Docker 的守护进程，具体的操作我列在了下面：

安装完成后需要你再对 Docker 的配置做一点修改，在“/etc/docker/daemon.json”里把 cgroup 的驱动程序改成 systemd ，然后重启 Docker 的守护进程，具体的操作我列在了下面：

```bash
# 安装docker
# 1. 卸载旧版本
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
# 2.安装依赖包
#安装所需资源包
yum install -y yum-utils \
device-mapper-persistent-data \
lvm2
#设置docker下载地址
sudo yum-config-manager \
--add-repo \
http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
    
#安装docker
sudo yum -y install docker-ce docker-ce-cli containerd.io

# 3. 启动docker
sudo systemctl start docker
docker -v

# 4.修改阿里云镜像加速

cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "registry-mirrors": ["https://7d7gzqhj.mirror.aliyuncs.com"]
}
EOF

# 重启，设置开机自启动
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker

```

### 三、关闭防火墙等

为了让 Kubernetes 能够检查、转发网络流量，你需要修改 iptables 的配置，启用“br_netfilter”模块：

```bash

#关闭防火墙
systemctl stop firewalld
systemctl disable firewalld
 
#关闭selinux
sed -i 's/enforcing/disabled/' /etc/selinux/config
setenforce 0
 
#关闭swag提升k8s性能
# swapoff -a  # 临时关闭
sed -ri 's/.*swap.*/#&/' /etc/fstab  # 永久关闭
 
#修改/etc/sysctl.conf配置 存在则修改，不存在则追加配置
#修改
sed -i "s#^net.ipv4.ip_forward.*#net.ipv4.ip_forward=1#g"  /etc/sysctl.conf
sed -i "s#^net.bridge.bridge-nf-call-ip6tables.*#net.bridge.bridge-nf-call-ip6tables=1#g"  /etc/sysctl.conf
sed -i "s#^net.bridge.bridge-nf-call-iptables.*#net.bridge.bridge-nf-call-iptables=1#g"  /etc/sysctl.conf
sed -i "s#^net.ipv6.conf.all.disable_ipv6.*#net.ipv6.conf.all.disable_ipv6=1#g"  /etc/sysctl.conf
sed -i "s#^net.ipv6.conf.default.disable_ipv6.*#net.ipv6.conf.default.disable_ipv6=1#g"  /etc/sysctl.conf
sed -i "s#^net.ipv6.conf.lo.disable_ipv6.*#net.ipv6.conf.lo.disable_ipv6=1#g"  /etc/sysctl.conf
sed -i "s#^net.ipv6.conf.all.forwarding.*#net.ipv6.conf.all.forwarding=1#g"  /etc/sysctl.conf
#追加
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.all.disable_ipv6 = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6 = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.lo.disable_ipv6 = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.all.forwarding = 1"  >> /etc/sysctl.conf
# 执行命令以应用
sysctl -p
 
```

完成之后，最好记得重启一下系统，然后给虚拟机拍个快照做备份，避免后续的操作失误导致重复劳动。



## 安装 kubeadm

好，现在我们就要安装 kubeadm 了，在 Master 节点和 Worker 节点上都要做这一步。kubeadm 可以直接从 Google 自己的软件仓库下载安装，但国内的网络不稳定，很难下载成功，需要改用其他的软件源，这里我选择了国内的某云厂商：

```bash
# 配置K8S的yum源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 卸载旧版本
yum remove -y kubelet kubeadm kubectl
 
 
# 查看可以安装的版本
yum list kubelet --showduplicates | sort -r
 
# 安装kubelet、kubeadm、kubectl 指定版本

# 安装kubelet、kubeadm、kubectl
yum install -y kubelet-1.23.3 kubeadm-1.23.3 kubectl-1.23.3
 
# 开机启动kubelet
systemctl enable kubelet && systemctl start kubelet

##注意，如果此时查看kubelet的状态，他会无限重启，等待接收集群命令，和初始化。这个是正常的。

```

### 准备所需镜像

```bash
# 查看所需镜像
kubeadm config images list --kubernetes-version v1.23.3

#------------------显示所需镜像-------------------
k8s.gcr.io/kube-apiserver:v1.23.3
k8s.gcr.io/kube-controller-manager:v1.23.3
k8s.gcr.io/kube-scheduler:v1.23.3
k8s.gcr.io/kube-proxy:v1.23.3
k8s.gcr.io/pause:3.6
k8s.gcr.io/etcd:3.5.1-0
k8s.gcr.io/coredns/coredns:v1.8.6

#------------------------------------------------

# 每台机器均准备需要的镜像（这里从阿里云私有仓库拉取，没有镜像的需要自行搜索。。）
# 下载master节点需要的镜像【选做】
 
#1、(三台机器同时)创建一个.sh文件,同样是给每个机器都安装,vi一个image.sh，内容如下,
#!/bin/bash
images=(
	kube-apiserver:v1.23.3
    kube-proxy:v1.23.3
	kube-controller-manager:v1.23.3
	kube-scheduler:v1.23.3
	coredns:1.8.6
	etcd:3.5.1-0
    pause:3.6
)
for imageName in ${images[@]} ; do
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done
 
#2、开启执行权限:chmod 777 image.sh
 
#3、执行image.sh文件: ./image.sh

```



### 镜像准备好后初始化 Master 节点

<font color='red'>操作需要注意的地方,只在master执行</font>

1. 将apiserver-advertise-address=172.26.165.243这句后面的ip修改为自己的要成为master的节点的IP
2. 将pod-network-cidr=192.168.0.0/16后面修改为calico.yaml中的value值.(查找calico中的value值:cat calico.yaml |grep 192)

```bash
#1、初始化master节点(单节点操作,关闭xshell的输入到所有会话)
# -----------------------------------------
#***************注意只在master节点初始化-------
# ——---在worker节点指向 kubeadm join -------\
# ------------------------------------------
kubeadm init --apiserver-advertise-address=192.168.234.131 \
--image-repository registry.cn-hangzhou.aliyuncs.com/google_containers \
--kubernetes-version v1.23.3 \
--service-cidr=10.96.0.0/16 \
--pod-network-cidr=192.168.0.0/16
 
 # 端口占用解决
 netstat -ntlup|grep 10250
 kill -9 PID
 
#service网络和pod网络；docker service create 
#docker container --> ip brigde
#Pod ---> ip 地址，整个集群 Pod 是可以互通。255*255
#service ---> 
 
#2、配置 kubectl
# 意思是要在本地建立一个“.kube”目录，然后拷贝 kubectl 的配置文件，你只要原样拷贝粘贴就行。另外
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
 
#3、提前保存令牌
# 还有一个很重要的“kubeadm join”提示，其他节点要加入集群必须要用指令里的 token 和 ca 证书，所以这条命令务必拷贝后保存好：
sudo kubeadm join 192.168.234.130:6443 --token l4hsyr.iwcxze31rtynd4x0 \
        --discovery-token-ca-cert-hash sha256:817126fad602981baf2343d1651c4d876d94fa070eb138bd19767b3aeb3c93cb
 
#4、部署网络插件
#上传网络插件，并部署
# 如果没有calico.yaml,下载并执行
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# 如果已有calico.yaml直接执行
#kubectl apply -f calico.yaml
 
#网络好的时候，就没有下面的操作了
calico：
image: calico/cni:v3.14.0
image: calico/cni:v3.14.0
image: calico/pod2daemon-flexvol:v3.14.0
image: calico/node:v3.14.0
image: calico/kube-controllers:v3.14.0
 
 
 
 
#5、查看状态，等待就绪
watch kubectl get pod -n kube-system -o wide

# 查看pod是否就绪
kubectl get pods -A

```

## **worker节点加入集群** 

```bash
#1、使用刚才master打印的令牌命令加入
kubeadm join 172.26.248.150:6443 --token ktnvuj.tgldo613ejg5a3x4 \
    --discovery-token-ca-cert-hash sha256:f66c496cf7eb8aa06e1a7cdb9b6be5b013c613cdcf5d1bbd88a6ea19a2b454ec
#2、如果超过2小时忘记了令牌，可以这样做
kubeadm token create --print-join-command #打印新令牌
kubeadm token create --ttl 0 --print-join-command #创建个永不过期的令牌

# 新加入node节点后查看所有pod是否就绪:
kubectl get pod -A wide

# 在准备移除的 worker 节点上执行
kubeadm reset -f
# 2.在 master 节点上执行
kubectl get nodes -o wide
# 3.删除worker节点，在 master 节点上执行
kubectl delete node #{worker-name}
```

kubeadm join 它会连接 Master 节点，然后拉取镜像，安装网络插件，最后把节点加入集群。当然，这个过程中同样也会遇到拉取镜像的问题，你可以如法炮制，提前把镜像下载到 Worker 节点本地，这样安装过程中就不会再有障碍了。Worker 节点安装完毕后，执行 kubectl get node ，就会看到两个节点都是“Ready”状态：

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202212/f756ece9e81af80a7204243f15777026.png)



## 小结

好了，把 Master 节点和 Worker 节点都安装好，我们今天的任务就算是基本完成了。后面 Console 节点的部署工作更加简单，它只需要安装一个 kubectl，然后复制“config”文件就行，你可以直接在 Master 节点上用“scp”远程拷贝，例如：

```bash

scp `which kubectl` root@192.168.234.129:~/
scp ~/.kube/config root@192.168.234.129:~/.kube
```





