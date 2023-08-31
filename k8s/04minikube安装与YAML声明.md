```shell
brew install minikube

brew install kubernetes-cli
# 1.24+ 版本的一个重大变化是从 kubelet 中移除了 dockershim，因此将 container runtime 从 docker 切换至 containerd
minikube start --driver=docker --container-runtime=containerd --image-mirror-country=cn

```

### 番外篇：什么是 containerd ？？？

面对 Kubernetes“咄咄逼人”的架势，Docker 是看在眼里痛在心里，虽然有苦心经营了多年的社区和用户群，但公司的体量太小，实在是没有足够的实力与大公司相抗衡。不过 Docker 也没有“坐以待毙”，而是采取了“断臂求生”的策略，推动自身的重构，把原本单体架构的 Docker Engine 拆分成了多个模块，其中的 Docker daemon 部分就捐献给了 CNCF，形成了 containerd。containerd 作为 CNCF 的托管项目，自然是要符合 CRI 标准的。但 Docker 出于自己诸多原因的考虑，它只是在 Docker Engine 里调用了 containerd，外部的接口仍然保持不变，也就是说还不与 CRI 兼容。由于 Docker 的“固执己见”，这时 Kubernetes 里就出现了两种调用链：

- 第一种是用 CRI 接口调用 dockershim，然后 dockershim 调用 Docker，Docker 再走 containerd 去操作容器。
- 第二种是用 CRI 接口直接调用 containerd 去操作容器。



![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202207/a8abfe5a55d0fa8b383867cc6062089b.png)

显然，由于都是用 containerd 来管理容器，所以这两种调用链的最终效果是完全一样的，但是第二种方式省去了 dockershim 和 Docker Engine 两个环节，更加简洁明了，损耗更少，性能也会提升一些。



### 切换docker到containerd

只需要将docker命令换成`crictl`

```bash
minikube ssh
# 内部容器运行时全部使用containerd
# 因此查看容器运行
crictl ps
# 查看镜像
crictl images
```

容器运行时启动失败，修改配置文件minikube ssh里的root用户密码也是root

配置文件是/etc/containerd/config.toml修改如下

```bash
version = 2
root = "/var/lib/containerd"                                     
state = "/run/containerd"
oom_score = 0      
imports = ["/etc/containerd/containerd.conf.d/02-containerd.conf"]

[grpc]
  address = "/run/containerd/containerd.sock"
  uid = 0
  gid = 0
  max_recv_message_size = 16777216
  max_send_message_size = 16777216

[debug]
  address = ""
  uid = 0
  gid = 0
  level = ""

[metrics]
  address = ""
  grpc_histogram = false

[cgroup]
  path = ""

[plugins]
  [plugins."io.containerd.monitor.v1.cgroups"]
    no_prometheus = false
  [plugins."io.containerd.grpc.v1.cri"]
    stream_server_address = ""
    stream_server_port = "10010"
    enable_selinux = false
sandbox_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.7"
    stats_collect_period = 10
    enable_tls_streaming = false
    max_container_log_line_size = 16384
restrict_oom_score_adj = false

    [plugins."io.containerd.grpc.v1.cri".containerd]
      discard_unpacked_layers = true
      snapshotter = "overlayfs"
      [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
        runtime_type = "io.containerd.runc.v2"
      [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime]
        runtime_type = ""
        runtime_engine = ""
        runtime_root = ""
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
SystemdCgroup = false

    [plugins."io.containerd.grpc.v1.cri".cni]
      bin_dir = "/opt/cni/bin"
conf_dir = "/etc/cni/net.mk"
      conf_template = ""
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://7d7gzqhj.mirror.aliyuncs.com"]
  [plugins."io.containerd.service.v1.diff-service"]
    default = ["walking"]
  [plugins."io.containerd.gc.v1.scheduler"]
    pause_threshold = 0.02
    deletion_threshold = 0
    mutation_threshold = 100
    schedule_delay = "0s"
    startup_delay = "100ms"
```

之后root用户重启containerd，之后的命令需要root权限才能启动containerd做之后的操作

```bash
systemctl restart containerd
# 查看运行的容器
crictl ps
# 查看镜像
crictl images
```





如果您已经安装了kubectl，您现在可以使用它来访问闪亮的新集群：

```shell
minikube kubectl get po -A
```

拷贝

或者，minikube可以下载相应版本的kubectl，您应该能够像这样使用它：

```shell
minikube kubectl -- get po -A
```

拷贝

您还可以通过在shell配置中添加以下内容来让您的生活更轻松：

```shell
vim ~/.zshrc
# 起别名
alias kubectl="minikube kubectl --"
# 补全命令插件
source <(kubectl completion zsh)
```



* kubectl 通过命令 `kubectl completion zsh` 生成 Zsh 自动补全脚本。 在 shell 中导入（Sourcing）该自动补全脚本，将启动 kubectl 自动补全功能。

为了在所有的 shell 会话中实现此功能，请将下面内容加入到文件 `~/.zshrc` 中。 *

```zsh
source <(kubectl completion zsh)
```

**如果你为 kubectl 定义了别名，kubectl 自动补全将自动使用它。

重新加载 shell 后，kubectl 自动补全功能将立即生效。

如果你收到 `2: command not found: compdef` 这样的错误提示，那请将下面内容添加到 `~/.zshrc` 文件的开头：**

```zsh
autoload -Uz compinit
compinit
```

最初，一些服务，如存储提供程序，可能尚未处于运行状态。这是集群调出期间的正常情况，并将立即自行解决。为了进一步了解您的集群状态，minikube捆绑了Kubernetes仪表板，允许您轻松适应新环境：

```shell
minikube dashboard
```



## k8s架构图

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202207/344e0c6dc2141b12f99e61252110f6b7.png)

Kubernetes 采用了现今流行的“控制面 / 数据面”（Control Plane / Data Plane）架构，集群里的计算机被称为“节点”（Node），可以是实机也可以是虚机，少量的节点用作控制面来执行集群的管理维护工作，其他的大部分节点都被划归数据面，用来跑业务应用。

控制面的节点在 Kubernetes 里叫做 Master Node，一般简称为 Master，它是整个集群里最重要的部分，可以说是 Kubernetes 的大脑和心脏。

数据面的节点叫做 Worker Node，一般就简称为 Worker 或者 Node，相当于 Kubernetes 的手和脚，在 Master 的指挥下干活。Node 的数量非常多，构成了一个资源池，Kubernetes 就在这个池里分配资源，调度应用。

因为资源被“池化”了，所以管理也就变得比较简单，可以在集群中任意添加或者删除节点。在这张架构图里，我们还可以看到有一个 kubectl，它就是 Kubernetes 的客户端工具，用来操作 Kubernetes，但它位于集群之外，理论上不属于集群。



你可以使用命令 kubectl get node 来查看 Kubernetes 的节点状态：kubectl get node

![image-20221112211636381](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202207/image-20221112211636381.png)

可以看到当前的 minikube 集群里只有一个 Master，那 Node 怎么不见了？这是因为 Master 和 Node 的划分不是绝对的。当集群的规模较小，工作负载较少的时候，Master 也可以承担 Node 的工作，就像我们搭建的 minikube 环境，它就只有一个节点，这个节点既是 Master 又是 Node。



## 节点内部结构

Kubernetes 的节点内部也具有复杂的结构，是由很多的模块构成的，这些模块又可以分成组件（Component）和插件（Addon）两类。组件实现了 Kubernetes 的核心功能特性，没有这些组件 Kubernetes 就无法启动，而插件则是 Kubernetes 的一些附加功能，属于“锦上添花”，不安装也不会影响 Kubernetes 的正常运行。

### Master 里的组件有哪些Master 里有 4 个组件，分别是 apiserver、etcd、scheduler、controller-manager。

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202207/330e03a66f636657c0d8695397c508c6.jpg)

  1. apiserver 是 Master 节点——同时也是整个 Kubernetes 系统的唯一入口，它对外公开了一系列的 RESTful API，并且加上了验证、授权等功能，所有其他组件都只能和它直接通信，可以说是 Kubernetes 里的联络员。

2. etcd 是一个高可用的分布式 Key-Value 数据库，用来持久化存储系统里的各种资源对象和状态，相当于 Kubernetes 里的配置管理员。注意它只与 apiserver 有直接联系，也就是说任何其他组件想要读写 etcd 里的数据都必须经过 apiserver。

3. scheduler 负责容器的编排工作，检查节点的资源状态，把 Pod 调度到最适合的节点上运行，相当于部署人员。因为节点状态和 Pod 信息都存储在 etcd 里，所以 scheduler 必须通过 apiserver 才能获得。

4. controller-manager 负责维护容器和节点等资源的状态，实现故障检测、服务迁移、应用伸缩等功能，相当于监控运维人员。同样地，它也必须通过 apiserver 获得存储在 etcd 里的信息，才能够实现对资源的各种操作。

   

   这 4 个组件也都被容器化了，运行在集群的 Pod 里，我们可以用 kubectl 来查看它们的状态，使用命令：

```bash
# 注意命令行里要用 -n kube-system 参数，表示检查“kube-system”名字空间里的 Pod，
kubectl get pod -n kube-system
```

![image-20221112212210061](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202207/image-20221112212210061.png)

Node 里的组件有哪些

Master 里的 apiserver、scheduler 等组件需要获取节点的各种信息才能够作出管理决策，那这些信息该怎么来呢？这就需要 Node 里的 3 个组件了，分别是 kubelet、kube-proxy、container-runtime。

1. kubelet 是 Node 的代理，负责管理 Node 相关的绝大部分操作，Node 上只有它能够与 apiserver 通信，实现状态报告、命令下发、启停容器等功能，相当于是 Node 上的一个“小
2. 管家”。kube-proxy 的作用有点特别，它是 Node 的网络代理，只负责管理容器的网络通信，简单来说就是为 Pod 转发 TCP/UDP 数据包，相当于是专职的“小邮差”。
3. 第三个组件 container-runtime 我们就比较熟悉了，它是容器和镜像的实际使用者，在 kubelet 的指挥下创建容器，管理 Pod 的生命周期，是真正干活的“苦力”。`1.24以后正式弃用docker改用containerd`

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202207/87bab507ce8381325e85570f3bc1d935.jpg)

我们一定要注意，因为 Kubernetes 的定位是容器编排平台，所以它没有限定 container-runtime 必须是 Docker，完全可以替换成任何符合标准的其他容器运行时，例如 containerd、CRI-O 等等，只不过在这里我们使用的是 Docker。

这 3 个组件中只有 kube-proxy 被容器化了，而 kubelet 因为必须要管理整个节点，容器化会限制它的能力，所以它必须在 container-runtime 之外运行。使用 minikube ssh 命令登录到节点后，可以用 docker ps 看到 kube-proxy：

```bash
minikube ssh
crictl ps | grep kube-proxy
# 而 kubelet 用 crictl ps 是找不到的，需要用操作系统的 ps 命令：
ps -ef | grep kubelet

```

![image-20221116214436028](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202207/image-20221116214436028.png)

现在，我们再把 Node 里的组件和 Master 里的组件放在一起来看，就能够明白 Kubernetes 的大致工作流程了：1. 每个 Node 上的 kubelet 会定期向 apiserver 上报节点状态，apiserver 再存到 etcd 里。

2. 每个 Node 上的 kube-proxy 实现了 TCP/UDP 反向代理，让容器对外提供稳定的服务。
3. scheduler 通过 apiserver 得到当前的节点状态，调度 Pod，然后 apiserver 下发命令给某个 Node 的 kubelet，kubelet 调用 container-runtime 启动容器。
4. controller-manager 也通过 apiserver 得到实时的节点状态，监控可能的异常情况，再使用相应的手段去调节恢复。

## minikube 也支持很多的插件，使用命令 minikube addons list 就可以查看插件列表：

插件中我个人认为比较重要的有两个：DNS 和 Dashboard。

DNS 你应该比较熟悉吧，它在 Kubernetes 集群里实现了域名解析服务，能够让我们以域名而不是 IP 地址的方式来互相通信，是服务发现和负载均衡的基础。由于它对微服务、服务网格等架构至关重要，所以基本上是 Kubernetes 的必备插件。

Dashboard 就是仪表盘，为 Kubernetes 提供了一个图形化的操作界面，非常直观友好，虽然大多数 Kubernetes 工作都是使用命令行 kubectl，但有的时候在 Dashboard 上查看信息也是挺方便的。

## YAML 声明式与命令式

Docker 命令和 Dockerfile 就属于“命令式”，大多数编程语言也属于命令式，它的特点是交互性强，注重顺序和过程，你必须“告诉”计算机每步该做什么，所有的步骤都列清楚，这样程序才能够一步步走下去，最后完成任务，显得计算机有点“笨”。

“声明式”，在 Kubernetes 出现之前比较少见，它与“命令式”完全相反，不关心具体的过程，更注重结果。我们不需要“教”计算机该怎么做，只要告诉它一个目标状态，它自己就会想办法去完成任务，相比起来自动化、智能化程度更高。

### 什么是 API 对象

学到这里还不够，因为 YAML 语言只相当于“语法”，要与 Kubernetes 对话，我们还必须有足够的“词汇”来表示“语义”。那么应该声明 Kubernetes 里的哪些东西，才能够让 Kubernetes 明白我们的意思呢？

```bash
# 查看api对象
kubectl api-resources
# --v=9 它会显示出详细的命令执行过程，清楚地看到发出的 HTTP 请求 
kubectl get po --v=9
```

目前的 Kubernetes 1.23 版本有 50 多种 API 对象，全面地描述了集群的节点、应用、配置、服务、账号等等信息，apiserver 会把它们都存储在数据库 etcd 里，然后 kubelet、scheduler、controller-manager 等组件通过 apiserver 来操作它们，就在 API 对象这个抽象层次实现了对整个集群的管理。

### 如何描述 API 对象

现在我们就来看看如何以 YAML 语言，使用“声明式”在 Kubernetes 里描述并创建 API 对象。之前我们运行 Nginx 的命令你还记得吗？使用的是 kubectl run，和 Docker 一样是“命令式”的：

```bash
kubectl run ngx --image=nginx:alpine
```

我们来把它改写成“声明式”的 YAML，说清楚我们想要的 Nginx 应用是个什么样子，也就是“目标状态”，让 Kubernetes 自己去决定如何拉取镜像运行：

```bash

apiVersion: v1
kind: Pod
metadata:
  name: ngx-pod
  labels:
    env: demo
    owner: chrono

spec:
  containers:
  - image: nginx:alpine
    name: ngx
    ports:
    - containerPort: 80
```

因为 API 对象采用标准的 HTTP 协议，为了方便理解，我们可以借鉴一下 HTTP 的报文格式，把 API 对象的描述分成“header”和“body”两部分。

- “header”包含的是 API 对象的基本信息，有三个字段：apiVersion、kind、metadata。apiVersion 表示操作这种资源的 API 版本号，由于 Kubernetes 的迭代速度很快，不同的版本创建的对象会有差异，为了区分这些版本就需要使用 apiVersion 这个字段，比如 v1、v1alpha1、v1beta1 等等。
-  kind 表示资源对象的类型，这个应该很好理解，比如 Pod、Node、Job、Service 等等。
- metadata 这个字段顾名思义，表示的是资源的一些“元信息”，也就是用来标记对象，方便 Kubernetes 管理的一些信息。比如在这个 YAML 示例里就有两个“元信息”，一个是 name，给 Pod 起了个名字叫 ngx-pod，另一个是 labels，给 Pod“贴”上了一些便于查找的标签，分别是 env 和 owner。

apiVersion、kind、metadata 都被 kubectl 用于生成 HTTP 请求发给 apiserver，你可以用 --v=9 参数在请求的 URL 里看到它们，比如：https://192.168.49.2:8443/api/v1/namespaces/default/pods/ngx-pod

和 HTTP 协议一样，“header”里的 apiVersion、kind、metadata 这三个字段是任何对象都必须有的，而“body”部分则会与对象特定相关，每种对象会有不同的规格定义，在 YAML 里就表现为 spec 字段（即 specification），表示我们对对象的“期望状态”（desired status）。还是来看这个 Pod，它的 spec 里就是一个 containers 数组，里面的每个元素又是一个对象，指定了名字、镜像、端口等信息：

现在把这些字段综合起来，我们就能够看出，这份 YAML 文档完整地描述了一个类型是 Pod 的 API 对象，要求使用 v1 版本的 API 接口去管理，其他更具体的名称、标签、状态等细节都记录在了 metadata 和 spec 字段等里。

### 使用YAML创建删除对象

使用 kubectl apply、kubectl delete，再加上参数 -f，你就可以使用这个 YAML 文件，创建或者删除对象了：

Kubernetes 收到这份“声明式”的数据，再根据 HTTP 请求里的 POST/DELETE 等方法，就会自动操作这个资源对象，至于对象在哪个节点上、怎么创建、怎么删除完全不用我们操心。

```bash

kubectl apply -f ngx-pod.yml
kubectl delete -f ngx-pod.yml
```

### 如何编写 YAML

讲到这里，相信你对如何使用 YAML 与 Kubernetes 沟通应该大概了解了，不过疑问也会随之而来：这么多 API 对象，我们怎么知道该用什么 apiVersion、什么 kind？metadata、spec 里又该写哪些字段呢？还有，YAML 看起来简单，写起来却比较麻烦，缩进对齐很容易搞错，有没有什么简单的方法呢？这些问题最权威的答案无疑是 Kubernetes 的官方参考文档（https://kubernetes.io/docs/reference/kubernetes-api/），API 对象的所有字段都可以在里面找到。不过官方文档内容太多太细，查阅起来有些费劲，所以下面我就介绍几个简单实用的小技巧。

- 第一个技巧其实前面已经说过了，就是 kubectl api-resources 命令，它会显示出资源对象相应的 API 版本和类型，比如 Pod 的版本是“v1”，Ingress 的版本是“networking.k8s.io/v1”，照着它写绝对不会错。
- 第二个技巧，是命令 kubectl explain，它相当于是 Kubernetes 自带的 API 文档，会给出对象字段的详细说明，这样我们就不必去网上查找了。比如想要看 Pod 里的字段该怎么写，就可以这样：

```bash

kubectl explain pod
kubectl explain pod.metadata
kubectl explain pod.spec
kubectl explain pod.spec.containers
```

![image-20221117074009061](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202207/image-20221117074009061.png)

### 生成样例

使用前两个技巧编写 YAML 就基本上没有难度了。不过我们还可以让 kubectl 为我们“代劳”，生成一份“文档样板”，免去我们打字和对齐格式的工作。这第三个技巧就是 kubectl 的两个特殊参数 --dry-run=client 和 -o yaml，前者是空运行，后者是生成 YAML 格式，结合起来使用就会让 kubectl 不会有实际的创建动作，而只生成 YAML 文件。例如，想要生成一个 Pod 的 YAML 样板示例，可以在 kubectl run 后面加上这两个参数：

```bash

kubectl run ngx --image=nginx:alpine --dry-run=client -o yaml >/tmp/ngx.yaml
```

就会生成一个绝对正确的 YAML 文件：

![image-20221117075607923](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202207/image-20221117075607923.png)

接下来你要做的，就是查阅对象的说明文档，添加或者删除字段来定制这个 YAML 了。今后除了一些特殊情况，我们都不会再使用 kubectl run 这样的命令去直接创建 Pod，而是会编写 YAML，用“声明式”来描述对象，再用 kubectl apply 去发布 YAML 来创建对象。

```bash

kubectl apply -f ngx-pod.yml
kubectl delete -f ngx-pod.yml
```

![image-20221117075803580](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202207/image-20221117075803580.png)
