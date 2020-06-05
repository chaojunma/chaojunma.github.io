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


### 全局工具配置

#### JDK配置

首先通过以下命令查找到jdk的安装路径

```
echo $JAVA_HOME
```

然后在jenkins全局工具配置中配置如下：

<div style="width:780px;height:410px;margin:50px auto">
    <img alt="jenkins-jdk.png" src="/images/jenkins-jdk.png" width="780" height="410"/>
</div>

#### Maven配置

<div style="width:780px;height:410px;margin:50px auto">
    <img alt="jenkins-maven.png" src="/images/jenkins-maven.png" width="780" height="410"/>
</div>

然后点击保存完成配置

### 安装Maven插件

接下来需要安装Maven插件，因为jenkins默认不支持maven项目的自动化部署，找到 “系统管理“ - “安装插件” ，点击 “可选插件”，找到如下maven插件的版本，插件名称 Maven Integration，如下：

<div style="width:780px;height:410px;margin:50px auto">
    <img alt="jenkins-plugin.png" src="/images/jenkins-plugin.png" width="780" height="410"/>
</div>

### jenkins部署SpringBoot项目

（1）发布一个任务，如下：

<div style="width:780px;height:410px;margin:50px auto">
    <img alt="jenkins-st.png" src="/images/jenkins-st.png" width="780" height="410"/>
</div>

（2）配置git地址及账号，如下：

<div style="width:780px;height:410px;margin:50px auto">
    <img alt="jenkins-git.png" src="/images/jenkins-git.png" width="780" height="410"/>
</div>

（3）配置maven打包命令，如下：

<div style="width:780px;height:410px;margin:50px auto">
    <img alt="jenkins-mvn.png" src="/images/jenkins-mvn.png" width="780" height="410"/>
</div>
（4）配置完成，点击立即构建即可，如下：

<div style="width:780px;height:410px;margin:50px auto">
    <img alt="jenkins-build.png" src="/images/jenkins-build.png" width="780" height="410"/>
</div>

第一次构建会比较慢，因为需要下载相关的依赖（jar包）,看到以下内容说明项目构建成功

<div style="width:780px;height:410px;margin:50px auto">
    <img alt="jenkins-bs.png" src="/images/jenkins-bs.png" width="780" height="410"/>
</div>

### jenkins通过执行shell脚本启动项目

在项目配置中配置shell脚本如下：

<div style="width:780px;height:410px;margin:50px auto">
    <img alt="jenkins-shell.png" src="/images/jenkins-shell.png" width="780" height="410"/>
</div>

```
#!/bin/bash
#服务名称
SERVER_NAME=springboot-thymeleaf
# 源jar路径,mvn打包完成之后，target目录下的jar包名称，也可选择成为war包，war包可移动到Tomcat的webapps目录下运行，这里使用jar包，用java -jar 命令执行  
JAR_NAME=springboot-thymeleaf
# 源jar路径  
#/usr/local/jenkins_home/workspace--->jenkins 工作目录
#demo 项目目录
#target 打包生成jar包的目录
JAR_PATH=/var/jenkins_home/workspace/springboot-thymeleaf/target/
# 打包完成之后，把jar包移动到运行jar包的目录--->work_daemon，work_daemon这个目录需要自己提前创建
JAR_WORK_PATH=/var/jenkins_home/workspace/springboot-thymeleaf/target/

echo "查询进程id-->$SERVER_NAME"
PID=`ps -ef | grep "$SERVER_NAME" | awk '{print $2}'`
echo "得到进程ID：$PID"
echo "结束进程"
for id in $PID
do
	kill -9 $id  
	echo "killed $id"  
done
echo "结束进程完成"

#复制jar包到执行目录
echo "复制jar包到执行目录:cp $JAR_PATH/$JAR_NAME.jar $JAR_WORK_PATH"
cp $JAR_PATH/$JAR_NAME.jar $JAR_WORK_PATH
echo "复制jar包完成"
cd $JAR_WORK_PATH
#修改文件权限
chmod 755 $JAR_NAME.jar



java -jar $JAR_NAME.jar
```

通过以上shell脚本，构建项目后，左侧进度条会一直加载，此时我们可以将shell脚本最后一行改为以下命令行：

```
BUILD_ID=dontKillMe nohup java -jar  $JAR_NAME.jar  &
```