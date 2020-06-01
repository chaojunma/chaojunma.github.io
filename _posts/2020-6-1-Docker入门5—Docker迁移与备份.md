---
layout: post
title: "Docker入门5—Docker迁移与备份"
date: 2020-6-1
categories: 运维
tags: Docker 
--- 

### 容器保存为镜像
我们可以通过以下命令将容器保存为镜像
```
docker commit mynginx mynginx_i
```
### 镜像备份
我们可以通过以下命令将镜像保存为tar 文件
```
docker  save -o mynginx.tar mynginx_i
```
### 镜像恢复与迁移
首先我们先删除掉mynginx_img镜像 然后执行此命令进行恢复
```
docker load -i mynginx.tar
```

-i 输入的文件

执行后再次查看镜像，可以看到镜像已经恢复
