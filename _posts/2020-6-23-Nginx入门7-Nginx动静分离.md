---
layout: post
title: "Nginx动静分离"
date: 2020-6-23
categories: 运维
tags: Nginx
--- 


### 实现效果

在浏览器访问html静态页面及图片最终访问静态资源目录，访问jsp页面最终访问tomcat服务器

### 部署

1、创建静态资源目录/static

2、在/static目录下创建index.html,内容如下：

```
test!!!!!!!
```

3、在/static目录下上传一张图片1.jpg

4、在tomcat目录的ROOT目录下创建test.jsp文件，内容如下：

```html
<%@ page language="java" import="java.util.*" pageEncoding="utf-8"%>
<HTML>
    <HEAD>
        <TITLE>JSP Page</TITLE>
    </HEAD>
    <BODY>
        <%
            Random rand = new Random();
            out.println("<h1>随机数:<h1>");
            out.println(rand.nextInt(99)+100);
        %>
    </BODY>
 </HTML>

```




### 配置nginx配置文件

切换到/usr/local/nginx/conf目录下，通过`vim`命令编辑`nginx.conf`文件，如下：

<div style="width:780px;height:280px;margin:50px auto">
    <img alt="nginx-static.png" src="/images/nginx-static.png" width="780" height="280"/>
</div>



```shell
location  ~(.*)(\.jpg|\.png|\.gif|\.jepg|\.css|\.css|\.html) {
            root   /static/;
            index  index.html index.htm;
        }

location ~ \.jsp {
    proxy_pass http://192.168.192.10:8080;
}

```


重新加载nginx配置

```
./nginx -s reload
```

在本地浏览器中访问[http://192.168.192.10/index.html](http://192.168.192.10/index.html)  返回结果如下：

<div style="width:780px;height:250px;margin:50px auto">
    <img alt="nginx-static-index.png" src="/images/nginx-static-index.png" width="780" height="250"/>
</div>

在本地浏览器中访问[http://192.168.192.10/index.html](http://192.168.192.10/index.html)  返回结果如下：

<div style="width:780px;height:450px;margin:50px auto">
    <img alt="nginx-static-image.png" src="/images/nginx-static-image.png" width="780" height="450"/>
</div>


在本地浏览器中访问[http://192.168.192.10/index.html](http://192.168.192.10/index.html)  返回结果如下：

<div style="width:780px;height:250px;margin:50px auto">
    <img alt="nginx-static-jsp.png" src="/images/nginx-static-jsp.png" width="780" height="250"/>
</div>

