---
layout: post
title: "Nginx配置反向代理实例2"
date: 2020-6-23
categories: 运维
tags: Nginx
--- 


### 实现效果

使用 nginx 反向代理，根据访问的路径跳转到不同端口的服务中，nginx 监听端口为 9001

访问 http://192.168.192.10:9001/v1/  直接跳转到 127.0.0.1:8080

访问 http:// 192.168.192.10:9001/v2/ 直接跳转到 127.0.0.1:8081

### 部署tomcat

1、部署两台tomcat，一台端口为 8080，一台端口为8081

2、在8080服务器的webapps目录下创建v1目录，并在v1目录下创建index.html文件，内容如下：

```
<h1>port:8080</h1>
```

3、在8081服务器的webapps目录下创建v2目录，并在v2目录下创建index.html文件，内容如下：

```
<h1>port:8081</h1>
```


### 配置nginx配置文件

切换到/usr/local/nginx/conf目录下，通过`vim`命令编辑`nginx.conf`文件，如下：

<div style="width:780px;height:400px;margin:50px auto">
    <img alt="proxy2.png" src="/images/proxy2.png" width="780" height="400"/>
</div>

重新加载nginx配置

```
./nginx -s reload
```

### 开放端口

在 windows 系统中访问 linux 中 nginx，默认不能访问的，因为防火墙问题

查看开放的端口号

```
firewall-cmd --list-all
```

设置开放的端口号

```
firewall-cmd --add-port=9001/tcp --permanent
firewall-cmd --add-port=8081/tcp --permanent  
```

重启防火墙

```
firewall-cmd --reload
```


在本地浏览器中访问[http://192.168.192.10:9001/v1/index.html](http://192.168.192.10:9001/v1/index.html) 如下：

<div style="width:780px;height:450px;margin:50px auto">
    <img alt="tomcat8080.png" src="/images/tomcat8080.png" width="780" height="450"/>
</div>

在本地浏览器中访问[http://192.168.192.10:9001/v2/index.html](http://192.168.192.10:9001/v2/index.html) 如下：

<div style="width:780px;height:450px;margin:50px auto">
    <img alt="tomcat8081.png" src="/images/tomcat8081.png" width="780" height="450"/>
</div>