---
layout: post
title: "Docker 之 Jenkins自动化部署"
date: 2020-6-4
categories: 运维
tags: Docker Jenkins
--- 


### 环境准备

（1）拉取私有仓库镜像
```
docker pull jenkins/jenkins
```

（2）创建挂载目录

```
mkdir -p /var/jenkins_home
```

(3) 修改目录权限

```
chown -R 1000 /var/jenkins_home
```

如果不执行这一步启动容器直接报错： /var/jenkins_home/copy_reference_file.log: Permission denied

这是因为jenkins容器里面的用户是jenkins，而你的主机用户不是jenkins，就算是root也不行一样报错，

这个时候我们就需要在主机上面给主机地址赋予权限，赋予它可以访问jenkins容器的权限，jenkins内部用的是uid 1000 的user

（4）创建并启动容器

```
docker run -d --name=jenkins -p 8080:8080 -v /var/jenkins_home:/var/jenkins_home jenkins/jenkins
```

（5）查看日志

```
docker logs -f 容器名称(或者容器ID)
```

容器启动完成在浏览器访问 [http://192.168.192.10:8080]（http://192.168.192.10:8080） 如下图所示：

<div style="width:780px;height:410px;margin:50px auto">
    <img alt="jenkins.png" src="/images/jenkins.png" width="780" height="410"/>
</div>

页面提示需要在`/var/jenkins_home/secrets/initialAdminPassword`目录下获取解锁密码

首先我们需要先通过以下命令进入容器
```
docker exec -it 容器名称(或者容器ID) /bin/bash
```
然后通过以下命令查看解锁密码

```
cat /var/jenkins_home/secrets/initialAdminPassword
```

然后将看到的一些列字符串解锁密码复制到页面的下面，继续就可以了，然后就来到如下页面

<div style="width:780px;height:410px;margin:50px auto">
    <img alt="jenkins-rm.png" src="/images/jenkins-rm.png" width="780" height="410"/>
</div>

然后Jenkins就会自动安装一些插件，这个过程比较漫长，插件安装完成以后会到以下页面

<div style="width:780px;height:410px;margin:50px auto">
    <img alt="jenkins-u.png" src="/images/jenkins-u.png" width="780" height="410"/>
</div>

根据表单填写信息之后就结束了,会跳转到jenkins主页，如下：

<div style="width:780px;height:410px;margin:50px auto">
    <img alt="jenkins-home.png" src="/images/jenkins-home.png" width="780" height="410"/>
</div>
