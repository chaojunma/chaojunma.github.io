---
layout: post
title: "使用Docker搭建Maven私服"
date: 2020-6-3
categories: 运维
tags: Docker Maven
--- 

### 环境搭建

（1）拉取私有仓库镜像

```
docker pull sonatype/nexus3
```

（2）根据镜像创建并执行docker容器

```
docker run -d -p 8081:8081 --name nexus -v /root/nexus-data:/var/nexus-data --restart=always sonatype/nexus3
```

- 上面命令是指使用nexus3镜像创建并启动一个容器，然后指定暴露8081端口到对应主机的8081端口。

- 将容器内部/var/nexus-data挂载到主机/root/nexus-data目录。

（3）查看日志

```
docker logs -f 容器名称
```
当看到以下信息时，说明私服容器启动完毕

```
-------------------------------------------------

Started Sonatype Nexus OSS 3.23.0-03

-------------------------------------------------
```

此时在浏览器输入：`http://ip:8081`即可看到以下页面：(ip为远程主机的ip地址)

<div style="width:800px;height:480px;margin:50px auto">
    <img alt="nexus.png" src="/images/nexus.png" width="800" height="480"/>
</div>

点击右上方的`Sign in`进行登录，初始账号密码为admin/admin123.

后来注意到在登陆时有提示信息 ，密码保存在 /nexus-data下的 admin.password  中，如下图所示：

<div style="width:800px;height:480px;margin:50px auto">
    <img alt="nexus-w.png" src="/images/nexus-w.png" width="800" height="480"/>
</div>

（4）进入容器查看密码

```
docker exec -it 容器名称 (或者容器ID) /bin/bash
```

<div style="width:800px;height:480px;margin:50px auto">
    <img alt="nexus-exec.png" src="/images/nexus-exec.png" width="800" height="480"/>
</div>

进入容器后，找到admin.password ，查看并复制密码。

注意密码只有这一部分：3694698d-5766-490f-b9b9-b72f217b15a5，紧跟后面的 bash-4.4$ 不是密码内容

（5）退出容器
<div style="width:800px;height:480px;margin:50px auto">
    <img alt="nexus-exit.png" src="/images/nexus-exit.png" width="800" height="480"/>
</div>

（6）登陆，输入密码：

<div style="width:450px;height:305px;margin:50px auto">
    <img alt="nexus-login.png" src="/images/nexus-login.png" width="450" height="305"/>
</div>

根据提示修改登录密码并完成登录
