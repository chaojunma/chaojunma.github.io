---
layout: post
title: "Nginx配置反向代理实例1"
date: 2020-6-22
categories: 运维
tags: Nginx
--- 


### 实现效果

打开浏览器，在浏览器地址栏输入地址 www.nginx.com，跳转到 liunx 系统 tomcat 主页
面中

### 部署tomcat

1、将本地的apache-tomcat-7.0.70.tar.gz文件上传到linux的/usr/local/src目录下

2、解压文件

```
tar -zxvf apache-tomcat-7.0.70.tar.gz
```

解压之后，会在当前目录下生成apache-tomcat-7.0.70文件夹

3、启动tomcat

进入到apache-tomcat-7.0.70下的bin目录下

```
cd apache-tomcat-7.0.70/bin/
```

启动tomcat服务器，在当前目录下执行以下命令：

```
./startup.sh
```

4、开放8080端口

```
firewall-cmd --add-port=8080/tcp --permanent
firewall-cmd --reload
```

5、在本地浏览器中访问[http://192.168.192.10:8080](http://192.168.192.10:8080) 如下：

<div style="width:780px;height:450px;margin:50px auto">
    <img alt="tomcat.png" src="/images/tomcat.png" width="780" height="450"/>
</div>

### 配置nginx配置文件

切换到/usr/local/nginx/conf目录下，通过`vim`命令编辑`nginx.conf`文件，如下：

<div style="width:780px;height:300px;margin:50px auto">
    <img alt="proxy.png" src="/images/proxy.png" width="780" height="300"/>
</div>

重新加载nginx配置

```
./nginx -s reload
```

### 配置本地host

```
192.168.192.10 www.nginx.com
```

在本地浏览器访问[www.nginx.com](www.nginx.com)如下：

<div style="width:780px;height:450px;margin:50px auto">
    <img alt="nginx-tomcat.png" src="/images/nginx-tomcat.png" width="780" height="450"/>
</div>