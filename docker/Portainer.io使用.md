## Portainer.io使用

### 1. 安装

首先，创建Portainer Server将用于存储其数据库的卷：

```bash
docker volume create portainer_data
```

然后，下载并安装Portainer Server容器：

```bash
docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
```

Portainer Server现已安装。您可以检查Portainer Server容器是否已通过运行`docker ps`启动：

```bash
docker ps

# 输出如下
CONTAINER ID   IMAGE                          COMMAND                  CREATED       STATUS      PORTS                                                                                  NAMES             
de5b28eb2fa9   portainer/portainer-ce:latest  "/portainer"             2 weeks ago   Up 9 days   0.0.0.0:8000->8000/tcp, :::8000->8000/tcp, 0.0.0.0:9443->9443/tcp, :::9443->9443/tcp   portainer

```

现在安装完成，您可以通过打开网页浏览器并前往：

```bash
https://localhost:9443
```

