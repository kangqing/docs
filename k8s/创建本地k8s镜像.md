## 没用上这篇文章，测试未通过，重点了解阿里云代码

为了解决kubeadm在初始化集群时，需要从谷歌官网下载image，而网络被墙的问题。

在[阿里云代码](https://codeup.aliyun.com/)或者github上创建7个仓库，名称分别为：kube-apiserver，kube-controller-manager，kube-scheduler，kube-proxy，pause，etcd，coredns。

以下以[阿里云代码](https://codeup.aliyun.com/)进行示范。

## 创建代码仓库

登录[阿里云代码](https://codeup.aliyun.com/)，创建一个空仓库`kube-apiserver`，可见等级可以设置为public。



```bash
# 创建了一个空代码仓库之后，在你本地执行如下命令，git链接请替换成自己的
git clone git@code.aliyun.com:272904483/kube-apiserver.git
cd kube-apiserver
touch README.md
git add README.md
git commit -m "add README"
git push -u origin master
```

## 创建镜像仓库

登录[阿里云容器镜像服务](https://cr.console.aliyun.com)

1. 创建命名空间
    创建一个命名空间，名称自定，我用的是`k8s-kubeadm-mirror`，设置为公有。

2. 创建镜像仓库
    创建一个镜像仓库，**地域**选择一个国内的，**命名空间**选择上面创建的`k8s-kubeadm-mirror`，**仓库名称**为`kube-apiserver`，**仓库类型**选择公开，如下图：

   ![image-20221213083125494](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202212/image-20221213083125494.png)

   点击“下一步”，开始关联镜像仓库与代码仓库。

   选择云code，也就是我们上面创建的阿里云代码源，选择好仓库，这里是

   ```
   272904483/kube-apiserver
   ```

   ，勾选

   代码变更时自动构建镜像

   和

   海外机器构建

   ，然后点击创建镜像仓库。

   ![image-20221213083150550](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202212/image-20221213083150550.png)

   

3、管理镜像仓库
 开始管理镜像仓库，进入镜像仓库管理页面。



![image-20221213083212936](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202212/image-20221213083212936.png)



可以看到创建完成后，默认就有一条自动构建规则，规则内容如下：

> 当代码仓库中tag为release-v1.2.3的代码触发构建时，会自动构建版本为1.2.3的镜像；当存在代码tag为1.2.3的自定义构建规则时，以自定义构建规则为准。

![image-20221213083230229](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202212/image-20221213083230229.png)

## 开始构建

在你本地电脑上，在第一步创建的git仓库中，新建Dockerfile文件，并执行相关操作即可。如下：



```bash
# 进入本地git代码库，就在master分支上工作即可。
cd kube-apiserver
# 新建文件Dockerfile
vim Dockerfile
# 写入如下内容，构建命令非常简单：从谷歌的镜像开始构建，然后加上维护人信息
From k8s.gcr.io/kube-apiserver:v1.12.3
MAINTAINER chenzheng <272904483@qq.com>
# 提交版本
git add Dockerfile
git commit -m "add Dockerfile"
# 一定要打上tag，并按照如下格式
git tag -a release-v1.12.3 -m "version 1.12.3"
# 将tag对应版本，push到仓库，就会开始自动构建过程
git push origin release-v1.12.3
```

然后进入`阿里云容器镜像服务`页面就可以看到如下信息：

![image-20221213083247117](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202212/image-20221213083247117.png)


 之后，针对`kube-apiserver`这个镜像，你如果需要其他的版本，修改完Dockerfile里面的版本信息后，重复上面的步骤即可自动构建。



## 使用镜像

在需要使用kubeadm进行k8s集群安装的机器上，拉取镜像，并重命名。



```csharp
# 拉取镜像
docker pull registry.cn-shenzhen.aliyuncs.com/k8s-kubeadm-mirror/kube-apiserver:1.12.3
# 重命名
docker tag registry.cn-shenzhen.aliyuncs.com/k8s-kubeadm-mirror/kube-apiserver:1.12.3 k8s.gcr.io/kube-apiserver:v1.12.3
# 查看镜像
[root@k8s-node2 ~]# docker image ls
REPOSITORY                                                            TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-apiserver                                             v1.12.3             a197e26bc609        12 minutes ago      194MB
registry.cn-shenzhen.aliyuncs.com/k8s-kubeadm-mirror/kube-apiserver   1.12.3              a197e26bc609        12 minutes ago      194MB
hello-world                                                           latest              4ab4c602aa5e        2 months ago        1.84kB
```

这样kubeadm就不会去官方拉取镜像，而使用本地镜像了。

