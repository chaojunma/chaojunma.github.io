---
layout: post
title: "Nginx的常用的命令及配置"
date: 2020-6-22
categories: 运维
tags: Nginx
--- 


### Nginx常用命令

进入 nginx 目录中

```
cd /usr/local/nginx/sbin
```

1、查看 nginx 版本号

```
./nginx -v
```

2、启动 nginx

```
./nginx
```

3、查看 nginx进程

```
ps -ef|grep nginx
```

4、停止 nginx

```
./nginx -s stop
```


5、重新加载 nginx

```
./nginx -s reload
```

### Nginx的配置文件

1、nginx 配置文件位置

```
cd /usr/local/nginx/conf/nginx.conf
```

<div style="width:780px;height:180px;margin:50px auto">
    <img alt="nginx-conf.png" src="/images/nginx-conf.png" width="780" height="180"/>
</div>

2、配置文件中的内容

包含三部分内容

（1）全局块：配置服务器整体运行的配置指令

比如 worker_processes 1;处理并发数的配置

（2）events 块：影响 Nginx 服务器与用户的网络连接

比如 worker_connections 1024; 支持的最大连接数为 1024

（3）http 块

还包含两部分：http 全局块server 块

详细配置说明如下：

```shell

nginx.conf说明
#user  nobody;
worker_processes  1; #工作进程：数目。根据硬件调整，通常等于cpu数量或者2倍cpu数量。
 
#错误日志存放路径
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;
 
#pid        logs/nginx.pid; # nginx进程pid存放路径
 
 
events {
    worker_connections  1024; # 工作进程的最大连接数量
}
 
 
http {
    include       mime.types; #指定mime类型，由mime.type来定义
    default_type  application/octet-stream;
 
    # 日志格式设置
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';
 
    #access_log  logs/access.log  main; #用log_format指令设置日志格式后，需要用access_log来指定日志文件存放路径
					
    sendfile        on; #指定nginx是否调用sendfile函数来输出文件，对于普通应用，必须设置on。
			如果用来进行下载等应用磁盘io重负载应用，可设着off，以平衡磁盘与网络io处理速度，降低系统uptime。
    #tcp_nopush     on; #此选项允许或禁止使用socket的TCP_CORK的选项，此选项仅在sendfile的时候使用
 
    #keepalive_timeout  0;  #keepalive超时时间
    keepalive_timeout  65;
 
    #gzip  on; #开启gzip压缩服务
 
    #虚拟主机
    server {
        listen       80;  #配置监听端口号
        server_name  localhost; #配置访问域名，域名可以有多个，用空格隔开
 
        #charset koi8-r; #字符集设置
 
        #access_log  logs/host.access.log  main;
 
        location / {
            root   html;
            index  index.html index.htm;
        }
        #错误跳转页
        #error_page  404              /404.html; 
 
        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
 
        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}
 
        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ { #请求的url过滤，正则匹配，~为区分大小写，~*为不区分大小写。
        #    root           html; #根目录
        #    fastcgi_pass   127.0.0.1:9000; #请求转向定义的服务器列表
        #    fastcgi_index  index.php; # 如果请求的Fastcgi_index URI是以 / 结束的, 该指令设置的文件会被附加到URI的后面并保存在变量$fastcig_script_name中
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}
 
        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }
 
 
    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;
 
    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}
 
 
    # HTTPS server
    #
    #server {
    #    listen       443 ssl;  #监听端口
    #    server_name  localhost; #域名
 
    #    ssl_certificate      cert.pem; #证书位置
    #    ssl_certificate_key  cert.key; #私钥位置
 
    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m; 
 
    #    ssl_ciphers  HIGH:!aNULL:!MD5; #密码加密方式
    #    ssl_prefer_server_ciphers  on; # ssl_prefer_server_ciphers  on; #
 
 
    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}
 
}
```