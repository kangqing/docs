# nginx 常用配置

Nginx 是一个高性能的 HTTP 和反向代理 web 服务器，同时也提供了 IMAP/POP3/SMTP 服务，其因丰富的功能集、稳定性、示例配置文件和低系统资源的消耗受到了开发者的欢迎。
```bash

# 侦听端口
server {
# 标准http协议
listen 80;
# 标准https协议
listen 443 ssl;
# For http2
listen 443 ssl http2;
# 使用ipv6监听80端口
listen [::]:80;
# 仅在使用ipv6时监听80端口
listen [::]:80 ipv6only=on;
}

#访问日志
server {
# Relative or full path to log file
access_log /path/to/file.log;
# Turn 'on' or 'off'  
access_log on;
}

# 域名
server {
# Listen to yourdomain.com
server_name yourdomain.com;
# Listen to multiple domains server_name yourdomain.com www.yourdomain.com;
# Listen to all domains
server_name *.yourdomain.com;
# Listen to all top-level domains
server_name yourdomain.*;
# Listen to unspecified Hostnames (Listens to IP address itself)
server_name "";
}

# 静态资产
server {
listen 80;
server_name yourdomain.com;
location / {
root /path/to/website;
}
}

# 重定向
server {
listen 80;
server_name www.yourdomain.com;
return 301 http://yourdomain.com$request_uri;
}
server {
listen 80;
server_name www.yourdomain.com;
location /redirect-url {
return 301 http://otherdomain.com;
}
}

# 反向代理
server {
listen 80;
server_name yourdomain.com;
location / {
proxy_pass http://0.0.0.0:3000;
# where 0.0.0.0:3000 is your application server (Ex: node.js) bound on 0.0.0.0 listening on port 3000
}
}

# 负载均衡
upstream node_js {
server 0.0.0.0:3000;
server 0.0.0.0:4000;
server 123.131.121.122;
}
server {
listen 80;
server_name yourdomain.com;
location / {
proxy_pass http://node_js;
}
}

# SSL 协议
server {
listen 443 ssl;
server_name yourdomain.com;
ssl on;
ssl_certificate /path/to/cert.pem;
ssl_certificate_key /path/to/privatekey.pem;
ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /path/to/fullchain.pem;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_session_timeout 1h;
ssl_session_cache shared:SSL:50m;
add_header Strict-Transport-Security max-age=15768000;
}

# HTTP到HTTPS的永久重定向
server 
{
listen 80;
server_name yourdomain.com;
return 301 https://$host$request_uri;
}
```
其实可以采用可视化的方式对 Nginx 进行配置，我在 GitHub 上发现了一款可以一键生成 Nginx 配置的神器，相当给力。

先来看看它都支持什么功能的配置：反向代理、HTTPS、HTTP/2、IPv6, 缓存、WordPress、CDN、Node.js 支持、 Python (Django) 服务器等等。

如果你想在线进行配置，只需要打开网站：https://nginxconfig.io/，按照自己的需求进行操作就行了。

选择你的场景，填写好参数，系统就会自动生成配置文件。

开源地址：github.com/digitalocean/nginxconfig.io

网站：digitalocean.com/community/tools/nginx

## 配置 https 报错

提示 Nginx 缺少 http_ssl_module 模块。如下：
nginx: [emerg] the "ssl" parameter requires ngx_http_ssl_module in /usr/local/nginx/conf/nginx.conf:42

```bash
# 进入Nginx源码目录
cd /root/FastDFS/nginx-1.12.2

# 附加--with-http_ssl_module
./configure --prefix=/usr/local/nginx --pid-path=/var/run/nginx/nginx.pid --lock-path=/var/lock/nginx.lock --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --with-http_gzip_static_module --http-client-body-temp-path=/var/temp/nginx/client --http-proxy-temp-path=/var/temp/nginx/proxy --http-fastcgi-temp-path=/var/temp/nginx/fastcgi --http-uwsgi-temp-path=/var/temp/nginx/uwsgi --http-scgi-temp-path=/var/temp/nginx/scgi --add-module=/root/FastDFS/fastdfs-nginx-module/src --with-http_ssl_module

# 运行make命令,一定不要make install ,否则会导致覆盖安装
make

# 备份
cp /youlocal/nginx/sbin/nginx /youlocal/nginx/sbin/nginx.bak

# 停止
nginx -s stop

# 新编译的Nginx覆盖原来的Nginx
cp ./objs/nginx /youlocal/nginx/sbin/

# 启动Nginx
nginx

# 再次查看已安装模块，确认是否成功
nginx -V

```

## 配置 https

```bash

#user  nobody;
worker_processes  1;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;
 
    server {
	    listen       443 ssl; # 添加https支持
        server_name  www.yunqing.xyz;

        #ssl配置
        ssl_certificate      /opt/software/nginx/html/ssl/7393205_www.yunqing.xyz.pem; # 配置证书
        ssl_certificate_key  /opt/software/nginx/html/ssl/7393205_www.yunqing.xyz.key; # 配置证书私钥
        ssl_protocols        TLSv1 TLSv1.1 TLSv1.2; # 配置SSL协议版本 # 配置SSL加密算法
        ssl_ciphers          HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on; # 优先采取服务器算法
        ssl_session_cache    shared:SSL:10m; # 配置共享会话缓存大小
        ssl_session_timeout  10m; # 配置会话超时时间
        


        location / {
            # 这里写自己hexo搭建的目录
            root   /opt/software/nginx/html/public; # public文件夹是hexo生成静态文件的默认文件夹
            index  index.html index.htm;
        }

        #error_page  404              /404.html;
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

    }

    server {
        listen 80;
        server_name www.yunqing.xyz;
        #将请求转成https
        return   301 https://$server_name$request_uri;
    }
    

}


```
## 防火墙
```bash

# 开启
service firewalld start

# 重启
service firewalld restart

# 关闭
service firewalld stop

# 查看防火墙规则
firewall-cmd --list-all

# 查询端口是否开放
firewall-cmd --query-port=8080/tcp

# 开放80端口
firewall-cmd --permanent --add-port=80/tcp

# 移除端口
firewall-cmd --permanent --remove-port=8080/tcp

#重启防火墙(修改配置后要重启防火墙)
firewall-cmd --reload

# 查看防火墙状态，是否是running
firewall-cmd --state
```