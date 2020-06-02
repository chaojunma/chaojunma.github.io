---
layout: post
title: "Docker私有仓库"
date: 2020-6-2
categories: 运维
tags: Docker 
--- 

### 私有仓库搭建与配置
（1）拉取私有仓库镜像（此步省略）
```
docker pull registry
```
（2）启动私有仓库容器
```
docker run -di --name=registry -p 5000:5000 registry
```
（3）打开浏览器 输入地址[http://192.168.192.10:5000//v2/_catalog][1]看到`{"repositories":[]}` 表示私有仓库搭建成功并且内容为空

（4）修改daemon.json
```
vi /etc/docker/daemon.json
```
添加以下内容，保存退出。

```
{"insecure-registries":["192.168.192.10:5000"]}
```
此步用于让 docker信任私有仓库地址

（5）重启docker 服务
```
systemctl restart docker
```
### 镜像上传至私有仓库
（1）标记此镜像为私有仓库的镜像
```
docker tag jdk1.8 192.168.192.10:5000/jdk1.8
```
（2）再次启动私服容器
```
docker start registry
```
（3）上传标记的镜像
```
docker push 192.168.192.10:5000/jdk1.8
```
（4）下载私有仓库的镜像，使用如下命令：
```
docker pull 192.168.192.10:5000/镜像名:版本号
```