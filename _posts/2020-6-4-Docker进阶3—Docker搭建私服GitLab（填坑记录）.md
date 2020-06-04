---
layout: post
title: "Docker搭建私服GitLab（填坑记录）"
date: 2020-6-4
categories: 运维
tags: Docker Git
--- 


作者在Docker笔记（持续更新）提到如何在Docker中安装GitLab，在此存在一个坑，作者觉得很有必要单开一篇文章单独将（希望让其他小伙伴能够通过标题搜到这篇文章，减少弯路）想必看到这篇文章的伙伴存在一个疑惑，为什么我在external_url设置ip+port却无法访问到GitLab，如果直接设置成ip地址在项目的checkout地址一栏，其git地址却不包含端口号，导致http的checkout地址不可用。

<div style="width:780px;height:450px;margin:50px auto">
    <img alt="docker-gitlab.webp" src="/images/docker-gitlab.webp" width="780" height="450"/>
</div>

问题的原因就出在external_url地址设置上。

GitLab默认的http访问端口号为80端口，如果想更改端口号，一般是通过docker run时设置端口映射，将80端口映射为其他端口。例如：
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
这里将GitLab的http端口改为8090，如果你这时修改external_url地址为http://ip:8090，那GitLab肯定访问不了，因为你已经将内部的端口号修改为8090端口了，而你通过docker run映射出来的端口号是80端口，所以不可能访问到。那该怎么办？

既然你已经将内部的端口号由80端口改为8090端口，这时候你就将容器停止并删除，但是不要将映射的配置文件删除(gitlab.rb文件)，docker在删除容器的时候不会将映射的文件删除。在此运行docker run命令，如下

```
docker run \
--detach \
--publish 8443:443 \
--publish 8090:8090 \
--name gitlab \
--restart unless-stopped \
-v /mnt/gitlab/etc:/etc/gitlab \
-v /mnt/gitlab/log:/var/log/gitlab \
-v /mnt/gitlab/data:/var/opt/gitlab \
gitlab/gitlab-ce
```
注意这里映射的端口为8090端口，根据自己设置的external_url端口号进行调整
接下来就能访问GitLab了，并且在checkout检出地址栏中，http地址端口号也正确了。
