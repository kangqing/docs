# 安装 rabbitMQ

```bash
# 下载镜像、启动容器、进入容器、启动插件、退出容器
docker pull rabbitmq
docker run -d --hostname my-rabbit --name rabbit -p 15672:15672 -p 5672:5672 rabbitmq
docker exec -it rabbitmq /bin/bash
rabbitmq-plugins enable rabbitmq_management
exit
```
## 访问 localhost:15672

用户名密码都是 guest


