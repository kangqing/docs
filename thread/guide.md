# MacOS idea 快捷键指南

python -m http.server 3000

## 查找
```bash
# 查找类
command + o

# try ... catch 等快捷键
# 先写被 try 的一行，然后
option + command + t

# 自动补全
option + enter

# 删除选中行
commmand + del

# 另起一行
shift + enter

# 查看最近操作的文件
command + E

# 进入光标的方法/变量的接口或定义处
command + B

# 行注释
command + /

# 块注释
command + option + /

# 复制当前行
command + D

# 快捷创建get set 方法
command + N

# 弹出首选项，系统配置
command + ,

# 格式化代码
command + option + L

# 重构代码成一个方法
command + option + M 

# 大小写切换
command + shift + U

# 关闭当前页
command + W

# 跳转到实现类，可以忽略接口
command + option + B


```

## Mac 使用记录

![Java之康庄大道](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202103/771217-20201229114846649-1476012694.jpg)

## 我的笔记本

```bash
# docs 目录下执行
python -m http.server 3000
# docsify
```



```markdown
<font color='red | orange | #ffcc00 | green | #33cccc | blue | ff00ff'>彩虹色</font>
```

## 0.端口查看

```bash
lsof -i tcp:8848
```



## 1. mysql

使用 homebrew 安装的，启动命令 `mysql.server start`

修改严格sql

对于已经创建的数据库例如数据库test

```mysql
use test;

set sql_mode = 'STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
```

对于其他默认的设置，也就是现在还未创建的数据库

```mysql
-- 查看sql 模式
select @@sql_mode;             
-- 修改默认模式
set @@global.sql_mode = 'STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
```





## 2. Nacos

直接下载安装的，在文稿里， 启动命令`sh startup.sh -m standalone`

关闭命令`sh shutdown.sh`

访问地址，127.0.0.1:8848/nacos

用户名密码都是 nacos

```bash
# docker 安装 nacos单机模式
docker pull nacos/nacos-server
# 创建挂载目录
sudo mkdir -p /opt/docker/nacos/logs
sudo mkdir -p /opt/docker/nacos/conf
# 创建配置文件
sudo vim /opt/docker/nacos/conf/custom.properties
# 内容自己去nacos中找application.properties，要是配置mysql自己配置
# 单体启动
# 需要把/opt/docker加到 docker 配置 -> 资源 -> 文件分享里
# 软连接 ln -s /opt/docker 一个快捷方式地址
docker  run \
--name nacos -d \
--privileged=true \
--restart=always \
-e MODE=standalone \
-e JVM_XMS=512m \
-e JVM_XMX=512m \
-e JVM_XMN=256m \
-p 8848:8848 \
-v /opt/docker/nacos/logs:/home/nacos/logs \
-v /opt/docker/nacos/conf/custom.properties:/home/nacos/init.d/custom.properties \
nacos/nacos-server


## 借鉴单机部署
docker run -d \
--name nacos \
-e MODE=standalone \
-e SPRING_DATASOURCE_PLATFORM=mysql \
-e MYSQL_SERVICE_HOST=数据库地址 \
-e MYSQL_SERVICE_PORT=3306 \
-e MYSQL_SERVICE_USER=数据库用户 \
-e MYSQL_SERVICE_PASSWORD=数据库密码 \
-e MYSQL_SERVICE_DB_NAME=nacos_config \ -- 其实就是你的库名称
-e NACOS_SERVER_IP=当前机器的IP \
-e NACOS_SERVER_PORT=端口 \
-e JVM_XMS=1g \
-e JVM_XMX=1g \
-e JVM_XMN=512m \
-p 18846:18846 \
-v /usr/nacos:/home/nacos/logs \
-v /etc/localtime:/etc/localtime:ro \
-v /etc/timezone:/etc/timezone:ro \
你生成后的镜像名称

## 借鉴集群部署
# 集群节点1
docker run -d \
--name 这个docker的名称 \
-e MODE=cluster \
-e NACOS_SERVERS="你的集群所有机器的IP和端口，逗号分隔" \
-e SPRING_DATASOURCE_PLATFORM=mysql \
-e MYSQL_SERVICE_HOST=数据库的地址 \
-e MYSQL_SERVICE_PORT=数据库端口 \
-e MYSQL_SERVICE_USER=数据库用户 \
-e MYSQL_SERVICE_PASSWORD=数据库密码 \
-e MYSQL_SERVICE_DB_NAME=nacos_config \ # 你那个库的名称，注意去掉这里的注释
-e JVM_XMS=1g \ # 机器内存不够的时候，减少一点内存，但太少就启动不了
-e JVM_XMX=1g \
-e JVM_XMN=512m \
-e NACOS_SERVER_IP=当前机器的IP \
-e NACOS_SERVER_PORT=当前机器nacos的访问端口 \
-p 18846:18846 \  # 映射地址
-v /usr/nacos-node:/home/nacos/logs \
-v /etc/localtime:/etc/localtime:ro \
-v /etc/timezone:/etc/timezone:ro \
你生成的镜像名称或ID

```



## 3. Python默认2.7切换到3.9

```bash
# 下载地址 https://www.python.org/downloads/
# 下载后一路下一步安装
# 配置环境变量
vim ~/.bash_profile
# 添加如下代码
PATH="/Library/Frameworks/Python.framework/Versions/3.9/bin:${PATH}"
export PATH

alias python="/Library/Frameworks/Python.framework/Versions/3.9/bin/python3.9"
# 关闭终端重启终端
python --version
# 已经切换到3.9.4
# 想要切换回2.7版本，需要注释掉下面这行
alias python="/Library/Frameworks/Python.framework/Versions/3.9/bin/python3.9"
```

## 4.配置maven

```bash
# 配置支持 ll
vim ~/.bash_profile
# 添加如下
alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'

# 进入配置文件
vim ~/.bash_profile
# 追加maven环境变量
export M2_HOME=/Users/yunqing/Documents/apache-maven-3.6.3
# 设置立即生效
source ~/.bash_profile
# 关联mvn命令，mvn命令是maven执行文件，存在于$M2_HOME/bin下，关联该文件到用户bin目录下
sudo ln -s $M2_HOME/bin/mvn /usr/local/bin/mvn
# 验证
mvn -version


```

## 5.如何把mac上的文件传到linux上

```bash
# scp 命令 直接拖拽文件路径就出来了， 然后接远程linux上的地址
scp /Users/yunqing/Desktop/e65492dc.html root@115.159.217.53:/root/nginx-html/public/article
```

## 6. 如何修改linux权限

```bash
# 介绍： r = 4  w = 2  x = 1 分别代表 读、写、执行 的权限
# Linux下权限的属组有 拥有者 、群组 、其它组 三种。每个文件都可以针对这三个属组（粒度），设置不同的rwx(读写执行)权限
chmod 777 filename  # 相当于给这个文件的权限是 rwx-rwx-rwx 三个属组都是具有读写执行三种权限

```

## 7. brew命令

```bash
//更新软件：
brew upgrade [包名]

//安装软件
brew install ffmpeg

//清理所有包的旧版本 
brew cleanup 

//查看需要更新的包
brew outdated

//查看brew的版本
brew -v

//更新homebrew自己
brew update

//查看可清理的旧版本包，不执行实际操作
brew cleanup -n 

//锁定某个包
brew pin $FORMULA  

//取消锁定
brew unpin $FORMULA  

//卸载
brew uninstall [包名]

//例：卸载git
brew uninstall git

//查看包信息
brew info [包名]

//查看安装列表
brew list

//查询可用包
brew search [包名]

//安装固定的包
brew install [包名]@版本

# 安装软件
# 让文件夹目录格式化，安装之后 tree 命令执行 tree -h 输出路径和大小
brew install tree
# 返回json格式化
brew install jsonpp

# 查看正常启动的服务列表
brew services list

# 以下方式启动，命令行关闭之后不会退出服务
brew services start redis

```

## docker 

```bash
# 查看容器
docker ps -a
# 查看镜像
docker images
# 停止容器
docker stop ...
# 删除容器
docker stop ...
# 删除镜像
docker mvi ...

```

## github 下载慢的问题

https://www.ipaddress.com/

去上边的网址找到这两个域名的ip更改的hosts中

140.82.114.3 github.com
199.232.69.194 github.global.ssl.fastly.net

然后刷新DNS缓存(mac)
sudo killall -HUP mDNSResponder


## nginx

```
Docroot is: /opt/homebrew/var/www

The default port has been set in /opt/homebrew/etc/nginx/nginx.conf to 8080 so that
nginx can run without sudo.

nginx will load all files in /opt/homebrew/etc/nginx/servers/.

To restart nginx after an upgrade:
  brew services restart nginx
Or, if you don't want/need a background service you can just run:
  /opt/homebrew/opt/nginx/bin/nginx -g daemon off;
```

```
默认端口已在/opt/homebrew/etc/nginx/nginx.conf。配置为8080，以便nginx可以在没有sudo的情况下运行。

nginx将加载/opt/homebrew/etc/nginx/servers/中的所有文件。

要在升级后重新启动nginx，请执行以下操作：
#重启
brew services restart nginx
# stop start

或者，如果您不想要/不需要后台服务，您可以运行：

/opt/homebrew/opt/nginx/bin/nginx -g daemon off;

# 不重启刷新配置
nginx -s reload
```

## redis
```
redis-cli

127.0.0.1:6379> type myzset
zset
127.0.0.1:6379> object encoding myzset
"ziplist"
```