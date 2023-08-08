## Pod 核心概念

为了解决这样多应用联合运行的问题，同时还要不破坏容器的隔离，就需要在容器外面再建立一个“收纳舱”，让多个容器既保持相对独立，又能够小范围共享网络、存储等资源，而且永远是“绑在一起”的状态。所以，Pod 的概念也就呼之欲出了，容器正是“豆荚”里那些小小的“豌豆”，你可以在 Pod 的 YAML 里看到，“spec.containers”字段其实是一个数组，里面允许定义多个容器。

### 为什么 Pod 是 Kubernetes 的核心对象

因为 Pod 是对容器的“打包”，里面的容器是一个整体，总是能够一起调度、一起运行，绝不会出现分离的情况，而且 Pod 属于 Kubernetes，可以在不触碰下层容器的情况下任意定制修改。所以有了 Pod 这个抽象概念，Kubernetes 在集群级别上管理应用就会“得心应手”了。Kubernetes 让 Pod 去编排处理容器，然后把 Pod 作为应用调度部署的最小单位，Pod 也因此成为了 Kubernetes 世界里的“原子”（当然这个“原子”内部是有结构的，不是铁板一块），基于 Pod 就可以构建出更多更复杂的业务形态了。

下图中展示的是以Pod为中心扩展出的Kubernetes 里的一些重要 API 对象，比如配置信息 ConfigMap、离线作业 Job、多实例部署 Deployment 等等，它们都分别对应到现实中的各种实际运维需求。图中你也应该能够看出来，所有的 Kubernetes 资源都直接或者间接地依附在 Pod 之上，所有的 Kubernetes 功能都必须通过 Pod 来实现，所以 Pod 理所当然地成为了 Kubernetes 的核心对象。

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202207/b5a7003788cb6f2b1c5c4f6873a8b5cf.jpg)

### 如何使用 YAML 描述 Pod

既然 Pod 这么重要，那么我们就很有必要来详细了解一下 Pod，理解了 Pod 概念，我们的 Kubernetes 学习之旅就成功了一半。还记得吧，我们始终可以用命令 `kubectl explain` 来查看任意字段的详细说明，所以接下来我就只简要说说写 YAML 时 Pod 里的一些常用字段。

因为 Pod 也是 API 对象，所以它也必然具有 `apiVersion、kind、metadata、spec `这四个基本组成部分。“apiVersion”和“kind”这两个字段很简单，对于 Pod 来说分别是固定的值 v1 和 Pod，而一般来说，“metadata”里应该有 name 和 labels 这两个字段。

我们在使用 Docker 创建容器的时候，可以不给容器起名字，但在 Kubernetes 里，Pod 必须要有一个名字，这也是 Kubernetes 里所有资源对象的一个约定。在课程里，我通常会为 Pod 名字统一加上 pod 后缀，这样可以和其他类型的资源区分开。

name 只是一个基本的标识，信息有限，所以 labels 字段就派上了用处。它可以添加任意数量的 Key-Value，给 Pod“贴”上归类的标签，结合 name 就更方便识别和管理了。

比如说，我们可以根据运行环境，使用标签 env=dev/test/prod，或者根据所在的数据中心，使用标签 region: north/south，还可以根据应用在系统中的层次，使用 tier=front/middle/back ……如此种种，只需要发挥你的想象力。下面这段 YAML 代码就描述了一个简单的 Pod，名字是“busy-pod”，再附加上一些标签：

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: busy-pod
  labels:
    owner: chrono
    env: demo
    region: north
    tier: back
```

“metadata”一般写上 name 和 labels 就足够了，而“spec”字段由于需要管理、维护 Pod 这个 Kubernetes 的基本调度单元，里面有非常多的关键信息，今天我介绍最重要的“containers”，其他的 hostname、restartPolicy 等字段你可以课后自己查阅文档学习。

“containers”是一个数组，里面的每一个元素又是一个 container 对象，也就是容器。和 Pod 一样，container 对象也必须要有一个 name 表示名字，然后当然还要有一个 image 字段来说明它使用的镜像，这两个字段是必须要有的，否则 Kubernetes 会报告数据验证错误。container 对象的其他字段基本上都可以和“入门篇”学过的 Docker、容器技术对应，理解起来难度不大，我就随便列举几个：

- ports：列出容器对外暴露的端口，和 Docker 的 -p 参数有点像。
- imagePullPolicy：指定镜像的拉取策略，可以是 Always/Never/IfNotPresent，一般默认是 IfNotPresent，也就是说只有本地不存在才会远程拉取镜像，可以减少网络消耗。
- env：定义 Pod 的环境变量，和 Dockerfile 里的 ENV 指令有点类似，但它是运行时指定的，更加灵活可配置。
- command：定义容器启动时要执行的命令，相当于 Dockerfile 里的 ENTRYPOINT 指令。
- args：它是 command 运行时的参数，相当于 Dockerfile 里的 CMD 指令，这两个命令和 Docker 的含义不同，要特别注意。

现在我们就来编写“busy-pod”的 spec 部分，添加 env、command、args 等字段：

```yaml

spec:
  containers:
  - image: busybox:latest
    name: busy
    imagePullPolicy: IfNotPresent
    env:
      - name: os
        value: "ubuntu"
      - name: debug
        value: "on"
    command:
      - /bin/echo
    args:
      - "$(os), $(debug)"
```

这里我为 Pod 指定使用镜像 busybox:latest，拉取策略是 IfNotPresent ，然后定义了 os 和 debug 两个环境变量，启动命令是 /bin/echo，参数里输出刚才定义的环境变量。把这份 YAML 文件和 Docker 命令对比一下，你就可以看出，YAML 在 spec.containers 字段里用“声明式”把容器的运行状态描述得非常清晰准确，要比 docker run 那长长的命令行要整洁的多，对人、对机器都非常友好。

### 如何使用 kubectl 操作 Pod

有了描述 Pod 的 YAML 文件，现在我就介绍一下用来操作 Pod 的 kubectl 命令。kubectl apply、kubectl delete 它们可以使用 -f 参数指定 YAML 文件创建或者删除 Pod，例如：

```bash
kubectl apply -f busy-pod.yml
kubectl delete -f busy-pod.yml
# 由于我们在yaml中给pod定义了name,删除的时候也可以根据名字删除
kubectl delete -f busy-pod
```

和 Docker 不一样，Kubernetes 的 Pod 不会在前台运行，只能在后台（相当于默认使用了参数 -d），所以输出信息不能直接看到。我们可以用命令 kubectl logs，它会把 Pod 的标准输出流信息展示给我们看，在这里就会显示出预设的两个环境变量的值：

```bash
kubectl logs busy-pod
# 查看pod列表和运行状态
kubectl get po
# 查看pod的详细状态，在状态为“CrashLoopBackOff”时候排查错误可用
kubectl describe pod busy-pod
```

排查错误：

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202211/786bb31f3d6d69edd16ddfb540d9ef68.png)

通常需要关注的是末尾的“Events”部分，它显示的是 Pod 运行过程中的一些关键节点事件。对于这个 busy-pod，因为它只执行了一条 echo 命令就退出了，而 Kubernetes 默认会重启 Pod，所以就会进入一个反复停止 - 启动的循环错误状态。

### 交互和pod内部

kubectl 也提供与 docker 类似的 cp 和 exec 命令，kubectl cp 可以把本地文件拷贝进 Pod，kubectl exec 是进入 Pod 内部执行 Shell 命令，用法也差不多。比如我有一个“a.txt”文件，那么就可以使用 kubectl cp 拷贝进 Pod 的“/tmp”目录里：

```bash
echo 'aaa' > a.txt
kubectl cp a.txt ngx-pod:/tmp
# 不过 kubectl exec 的命令格式与 Docker 有一点小差异，需要在 Pod 后面加上 --，把 kubectl 的命令与 Shell 命令分隔开，你在用的时候需要小心一些：
kubectl exec -it ngx-pod -- sh
```

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202211/343756ee45533a056fdca97f9fe2dd6b.png)

虽然 Pod 是 Kubernetes 的核心概念，非常重要，但事实上在 Kubernetes 里通常并不会直接创建 Pod，因为它只是对容器做了简单的包装，比较脆弱，离复杂的业务需求还有些距离，需要 Job、CronJob、Deployment 等其他对象增添更多的功能才能投入生产使用。

Kubernetes 的核心对象 Pod，用来编排一个或多个容器，让这些容器共享网络、存储等资源，总是共同调度，从而紧密协同工作。因为 Pod 比容器更能够表示实际的应用，所以 Kubernetes 不会在容器层面来编排业务，而是把 Pod 作为在集群里调度运维的最小单位。

## 为什么不直接使用 Pod

Kubernetes 使用的是 RESTful API，把集群中的各种业务都抽象为 HTTP 资源对象，那么在这个层次之上，我们就可以使用面向对象的方式来考虑问题。如果你有一些编程方面的经验，就会知道面向对象编程（OOP），它把一切都视为高内聚的对象，强调对象之间互相通信来完成任务。

虽然面向对象的设计思想多用于软件开发，但它放到 Kubernetes 里却意外地合适。因为 Kubernetes 使用 YAML 来描述资源，把业务简化成了一个个的对象，内部有属性，外部有联系，也需要互相协作，只不过我们不需要编程，完全由 Kubernetes 自动处理（其实 Kubernetes 的 Go 语言内部实现就大量应用了面向对象）。

面向对象的设计有许多基本原则，其中有两条我认为比较恰当地描述了 Kubernetes 对象设计思路，一个是“单一职责”，另一个是“组合优于继承”。

- “单一职责”的意思是对象应该只专注于做好一件事情，不要贪大求全，保持足够小的粒度才更方便复用和管理。
- “组合优于继承”的意思是应该尽量让对象在运行时产生联系，保持松耦合，而不要用硬编码的方式固定对象的关系。

应用这两条原则，我们再来看 Kubernetes 的资源对象就会很清晰了。因为 Pod 已经是一个相对完善的对象，专门负责管理容器，那么我们就不应该再“画蛇添足”地盲目为它扩充功能，而是要保持它的独立性，容器之外的功能就需要定义其他的对象，把 Pod 作为它的一个成员“组合”进去。这样每种 Kubernetes 对象就可以只关注自己的业务领域，只做自己最擅长的事情，其他的工作交给其他对象来处理，既不“缺位”也不“越位”，既有分工又有协作，从而以最小成本实现最大收益。

### 为什么要有 Job/CronJob

现在我们来看看 Kubernetes 里的两种新对象：Job 和 CronJob，它们就组合了 Pod，实现了对离线业务的处理。

- “在线业务”类型的应用有很多，比如 Nginx、Node.js、MySQL、Redis 等等，一旦运行起来基本上不会停，也就是永远在线。
- 而“离线业务”类型的应用也并不少见，它们一般不直接服务于外部用户，只对内部用户有意义，比如日志分析、数据建模、视频转码等等，虽然计算量很大，但只会运行一段时间。“离线业务”的特点是必定会退出，不会无期限地运行下去，所以它的调度策略也就与“在线业务”存在很大的不同，需要考虑运行超时、状态检查、失败重试、获取计算结果等管理事项。

而这些业务特性与容器管理没有必然的联系，如果由 Pod 来实现就会承担不必要的义务，违反了“单一职责”，所以我们应该把这部分功能分离到另外一个对象上实现，让这个对象去控制 Pod 的运行，完成附加的工作。

“离线业务”也可以分为两种。

- 一种是“临时任务”，跑完就完事了，下次有需求了说一声再重新安排；
- 另一种是“定时任务”，可以按时按点周期运行，不需要过多干预。

对应到 Kubernetes 里，“临时任务”就是 API 对象 Job，“定时任务”就是 API 对象 CronJob，使用这两个对象你就能够在 Kubernetes 里调度管理任意的离线业务了。由于 Job 和 CronJob 都属于离线业务，所以它们也比较相似。我们先学习通常只会运行一次的 Job 对象以及如何操作。

### 如何使用 YAML 描述 Job

Job 的 YAML“文件头”部分还是那几个必备字段，我就不再重复解释了，简单说一下：

- apiVersion 不是 v1，而是 batch/v1。
- kind 是 Job，这个和对象的名字是一致的。
- metadata 里仍然要有 name 标记名字，也可以用 labels 添加任意的标签。

如果记不住这些也不要紧，你还可以使用命令 kubectl explain job 来看它的字段说明。不过想要生成 YAML 样板文件的话不能使用 kubectl run，因为 kubectl run 只能创建 Pod，要创建 Pod 以外的其他 API 对象，需要使用命令 kubectl create，再加上对象的类型名。比如用 busybox 创建一个“echo-job”，命令就是这样的：

```bash
kubectl create job echo-job --image=busybox --dry-run=client -o yaml >/tmp/echo-job.yaml
```

```yaml
# 上面的命令会在/tmp下生成一个模板
apiVersion: batch/v1
kind: Job
metadata:
  name: echo-job

spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - image: busybox
        name: echo-job
        imagePullPolicy: IfNotPresent
        command: ["/bin/echo"]
        args: ["hello", "world"]
```

你会注意到 Job 的描述与 Pod 很像，但又有些不一样，主要的区别就在“spec”字段里，多了一个 template 字段，然后又是一个“spec”，显得有点怪。

如果你理解了刚才说的面向对象设计思想，就会明白这种做法的道理。它其实就是在 Job 对象里应用了组合模式，template 字段定义了一个“应用模板”，里面嵌入了一个 Pod，这样 Job 就可以从这个模板来创建出 Pod。而这个 Pod 因为受 Job 的管理控制，不直接和 apiserver 打交道，也就没必要重复 apiVersion 等“头字段”，只需要定义好关键的 spec，描述清楚容器相关的信息就可以了，可以说是一个“无头”的 Pod 对象。

为了辅助你理解，我把 Job 对象重新组织了一下，用不同的颜色来区分字段，这样你就能够很容易看出来，其实这个“echo-job”里并没有太多额外的功能，只是把 Pod 做了个简单的包装：

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202211/9b780905a824d2103d4ayyc79267ae28.jpg)

总的来说，这里的 Pod 工作非常简单，在 containers 里写好名字和镜像，command 执行 /bin/echo，输出“hello world”。不过，因为 Job 业务的特殊性，所以我们还要在 spec 里多加一个字段 restartPolicy，确定 Pod 运行失败时的策略，OnFailure 是失败原地重启容器，而 Never 则是不重启容器，让 Job 去重新调度生成一个新的 Pod。

如何在 Kubernetes 里操作 Job现在让我们来创建 Job 对象，运行这个简单的离线作业，用的命令还是 kubectl apply：

```bash

kubectl apply -f job.yml
```

创建之后 Kubernetes 就会从 YAML 的模板定义中提取 Pod，在 Job 的控制下运行 Pod，你可以用 kubectl get job、kubectl get pod 来分别查看 Job 和 Pod 的状态：

![image-20221119100544790](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202211/image-20221119100544790.png)

可以看到，因为 Pod 被 Job 管理，它就不会反复重启报错了，而是会显示为 Completed 表示任务完成，而 Job 里也会列出运行成功的作业数量，这里只有一个作业，所以就是 1/1。

你还可以看到，Pod 被自动关联了一个名字，用的是 Job 的名字（echo-job）再加上一个随机字符串（h8fv7），这当然也是 Job 管理的“功劳”，免去了我们手工定义的麻烦，这样我们就可以使用命令 kubectl logs 来获取 Pod 的运行结果.



到这里，你可能会觉得，经过了 Job、Pod 对容器的两次封装，虽然从概念上很清晰，但好像并没有带来什么实际的好处，和直接跑容器也差不了多少。其实 Kubernetes 的这套 YAML 描述对象的框架提供了非常多的灵活性，可以在 Job 级别、Pod 级别添加任意的字段来定制业务，这种优势是简单的容器技术无法相比的。这里我列出几个控制离线作业的重要字段，其他更详细的信息可以参考 Job 文档：

- activeDeadlineSeconds，设置 Pod 运行的超时时间。
- backoffLimit，设置 Pod 的失败重试次数。
- completions，Job 完成需要运行多少个 Pod，默认是 1 个。
- parallelism，它与 completions 相关，表示允许并发运行的 Pod 数量，避免过多占用资源。

要注意这 4 个字段并不在 template 字段下，而是在 spec 字段下，所以它们是属于 Job 级别的，用来控制模板里的 Pod 对象。

下面我再创建一个 Job 对象，名字叫“sleep-job”，它随机睡眠一段时间再退出，模拟运行时间较长的作业（比如 MapReduce）。Job 的参数设置成 15 秒超时，最多重试 2 次，总共需要运行完 4 个 Pod，但同一时刻最多并发 2 个 Pod：

```yaml

apiVersion: batch/v1
kind: Job
metadata:
  name: sleep-job

spec:
  activeDeadlineSeconds: 15
  backoffLimit: 2
  completions: 4
  parallelism: 2

  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - image: busybox
        name: echo-job
        imagePullPolicy: IfNotPresent
        command:
          - sh
          - -c
          - sleep $(($RANDOM % 10 + 1)) && echo done
```

使用 kubectl apply 创建 Job 之后，我们可以用 kubectl get pod -w 来实时观察 Pod 的状态，看到 Pod 不断被排队、创建、运行的过程：

```bash

kubectl apply -f sleep-job.yml
kubectl get pod -w
```

![image-20221119101628125](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202211/image-20221119101628125.png)

等到 4 个 Pod 都运行完毕，我们再用 kubectl get 来看看 Job 和 Pod 的状态：

![image-20221119101737597](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202211/image-20221119101737597.png)

就会看到 Job 的完成数量如同我们预期的是 4，而 4 个 Pod 也都是完成状态。由于我只完成了三个就强制退出了所以我这里显示的是三个。显然，“声明式”的 Job 对象让离线业务的描述变得非常直观，简单的几个字段就可以很好地控制作业的并行度和完成数量，不需要我们去人工监控干预，Kubernetes 把这些都自动化实现了。

### 如何使用 YAML 描述 CronJob

学习了“临时任务”的 Job 对象之后，再学习“定时任务”的 CronJob 对象也就比较容易了，我就直接使用命令 kubectl create 来创建 CronJob 的样板。

要注意两点。第一，因为 CronJob 的名字有点长，所以 Kubernetes 提供了简写 cj，这个简写也可以使用命令 kubectl api-resources 看到；第二，CronJob 需要定时运行，所以我们在命令行里还需要指定参数 --schedule。

```bash

kubectl create cj echo-cj --image=busybox --schedule="" --dry-run=client -o yaml >/tmp/cj.yaml
```

然后我们编辑这个 YAML 样板，生成 CronJob 对象：

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: echo-cj

spec:
  schedule: '*/1 * * * *'
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - image: busybox
            name: echo-cj
            imagePullPolicy: IfNotPresent
            command: ["/bin/echo"]
            args: ["hello", "world"]
```

我们还是重点关注它的 spec 字段，你会发现它居然连续有三个 spec 嵌套层次：

- 第一个 spec 是 CronJob 自己的对象规格声明
- 第二个 spec 从属于“jobTemplate”，它定义了一个 Job 对象。
- 第三个 spec 从属于“template”，它定义了 Job 里运行的 Pod。

所以，CronJob 其实是又组合了 Job 而生成的新对象，我还是画了一张图，方便你理解它的“套娃”结构：

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202211/yy352c661ae37dd116dd12c61932b43c.jpg)

除了定义 Job 对象的“jobTemplate”字段之外，CronJob 还有一个新字段就是“schedule”，用来定义任务周期运行的规则。它使用的是标准的 Cron 语法，指定分钟、小时、天、月、周，和 Linux 上的 crontab 是一样的。像在这里我就指定每分钟运行一次，除了名字不同，CronJob 和 Job 的用法几乎是一样的，使用 kubectl apply 创建 CronJob，使用 kubectl get cj、kubectl get pod 来查看状态：

```bash

kubectl apply -f cronjob.yml
kubectl get cj
kubectl get pod
```

![img](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202211/b00fdd8541372fb7a4de00de5ac6342c.png)