---
layout: post
title: "Docker搭建私服GitLab"
date: 2020-6-3
categories: 运维
tags: Docker Git
--- 


gitlab就相当于我们自己内网搭建的git服务，相当于公司内的github。

### 拉取镜像

```
docker pull gitlab/gitlab-ce
```

### 创建宿主机的数据目录

```
mkdir -p /mnt/gitlab/etc
mkdir -p /mnt/gitlab/log
mkdir -p /mnt/gitlab/data
```

### 创建并运行容器

```
docker run \
--detach \
--publish 8443:443 \
--publish 8090:80 \
--name gitlab \
--restart unless-stopped \
-v /mnt/gitlab/etc:/etc/gitlab \
-v /mnt/gitlab/log:/var/log/gitlab \
-v /mnt/gitlab/data:/var/opt/gitlab \
gitlab/gitlab-ce
```

按上面的方式，gitlab容器运行没问题，但在gitlab上创建项目的时候，生成项目的URL访问地址是按容器的hostname来生成的，也就是容器的id。作为gitlab服务器，我们需要一个固定的URL访问地址，于是需要配置gitlab.rb（宿主机路径：/mnt/gitlab/etc/gitlab.rb）。

```
vim /mnt/gitlab/etc/gitlab.rb
```

```
# 配置http协议所使用的访问地址,不加端口号默认为80
external_url 'http://192.168.192.10:8090'
```

修改/mnt/gitlab/data/gitlab-rails/etc/gitlab.yml

找到关键字 * ## Web server settings * 

将host的值改成映射的外部主机ip地址和端口，这里会显示在gitlab克隆地址

```
vi /mnt/gitlab/data/gitlab-rails/etc/gitlab.yml
```

```
host: 192.168.192.10
port: 8090
https: false
```

然后重启容器

```
docker restart 容器名称（或者容器ID）
```

查看容器启动日志

```
docker logs -f 容器名称（或者容器ID）
```

待容器启动完毕，就可以正常访问了