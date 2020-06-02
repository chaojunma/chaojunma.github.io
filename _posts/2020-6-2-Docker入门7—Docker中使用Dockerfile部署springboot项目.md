---
layout: post
title: "Docker中使用Dockerfile部署springboot项目"
date: 2020-6-2
categories: 运维
tags: Docker 
--- 

### 将本地SpringBoot项目打包并上传到Linux服务器

将本地SpringBoot项目打包并上传到Linux服务器，首先我们将项目的jar包上传到某个目录下，例如我们将文件上传到`/usr/local/dockerapp`目录下。

### 准备Dockerfile文件

创建Dockerfile文件 `vi Dockerfile`

```
FROM java:8
VOLUME /tmp
ADD sso-server.jar /sso-server.jar
EXPOSE 8080
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/sso-server.jar"]
```

命令 | 作用  
-|-|
FROM java:8 | 依赖的基础镜像名称 |
VOLUME /tmp | 创建/tmp目录并持久化到Docker数据文件夹，因为Spring Boot使用的内嵌Tomcat容器默认使
用/tmp作为工作目录 |
ADD sso-server.jar /sso-server.jar | 将宿主机的文件复制到容器内，如果是一个压缩文件，将会在复制后自动解压 |
EXPOSE 80 | 并不是真正的发布端口，这个只是容器部署人员与建立image的人员之间的交流，即建立image的人
员告诉容器布署人员容器应该映射哪个端口给外界 |
ENTRYPOINT | 容器启动时运行的命令，相当于我们在命令行中输入java -jar xxxx.jar，为了缩短 Tomcat 的启动时间，添加java.security.egd的系统属性指向/dev/urandom作为 ENTRYPOINT |

### 构建镜像

把jar包和Dockerfile放在同一个目录下执行下面命令
```
docker build -t sso-server .
```
### 运行容器

```
docker run  -d --name 容器名称 -p 8080:8080 镜像名称
```
其中

-d 表示后台运行容器，这也就自然地解决的Spring Boot不支持后台运行应用程序的问题。

-p 8080:8080表示将容器内部的8080端口映射到宿主机器的8080端口，这样就可以通过宿主机器直接访问应用。

--name 给容器取一个容易记住的名字方便日后管理。

### 查看运行日志
```
docker logs -f --tail=100 容器名称 
```
tail后面是常看日志的条数
