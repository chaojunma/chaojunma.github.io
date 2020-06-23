---
layout: post
title: "Nginx配置负载均衡"
date: 2020-6-23
categories: 运维
tags: Nginx
--- 


### 实现效果

在浏览器访问地址 [http://192.168.192.10/weight](http://192.168.192.10/weight)，实现负载均衡效果，轮询访问8080服务器和8081服务器

### 部署tomcat

1、部署两台tomcat，一台端口为 8080，一台端口为8081

2、在8080服务器的webapps目录下创建weight目录，并在weight目录下创建index.html文件，内容如下：

```
<h1>port:8080</h1>
```

3、在8081服务器的webapps目录下创建weight目录，并在weight目录下创建index.html文件，内容如下：

```
<h1>port:8081</h1>
```


### 配置nginx配置文件

切换到/usr/local/nginx/conf目录下，通过`vim`命令编辑`nginx.conf`文件，如下：

<div style="width:780px;height:400px;margin:50px auto">
    <img alt="nginx-weight.png" src="/images/nginx-weight.png" width="780" height="400"/>
</div>

重新加载nginx配置

```
./nginx -s reload
```

在本地浏览器中访问[http://192.168.192.10/weight](http://192.168.192.10/weight) 轮询返回如下结果：

<div style="width:780px;height:250px;margin:50px auto">
    <img alt="nginx-w-8080.png" src="/images/nginx-w-8080.png" width="780" height="250"/>
</div>

<div style="width:780px;height:250px;margin:50px auto">
    <img alt="nginx-w-8081.png" src="/images/nginx-w-8081.png" width="780" height="250"/>
</div>

此时说明负载均衡生效了！！！

<span style="color:red">注意：在谷歌浏览器访问可能会看不到效果，建议在其他浏览器测试 </span>
