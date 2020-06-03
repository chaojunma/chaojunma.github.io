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

<div style="width:780px;height:450px;margin:50px auto">
    <img alt="nexus.png" src="/images/nexus.png" width="780" height="450"/>
</div>

点击右上方的`Sign in`进行登录，初始账号密码为admin/admin123.

后来注意到在登陆时有提示信息 ，密码保存在 /nexus-data下的 admin.password  中，如下图所示：

<div style="width:780px;height:450px;margin:50px auto">
    <img alt="nexus-w.png" src="/images/nexus-w.png" width="780" height="450"/>
</div>

（4）进入容器查看密码

```
docker exec -it 容器名称 (或者容器ID) /bin/bash
```

<div style="width:780px;height:170px;margin:50px auto">
    <img alt="nexus-exec.png" src="/images/nexus-exec.png" width="780" height="170"/>
</div>

进入容器后，找到admin.password ，查看并复制密码。

注意密码只有这一部分：3694698d-5766-490f-b9b9-b72f217b15a5，紧跟后面的 bash-4.4$ 不是密码内容

（5）退出容器
<div style="width:780px;height:280px;margin:50px auto">
    <img alt="nexus-exit.png" src="/images/nexus-exit.png" width="780" height="280"/>
</div>

（6）登陆，输入密码：

<div style="width:450px;height:305px;margin:50px auto">
    <img alt="nexus-login.png" src="/images/nexus-login.png" width="450" height="305"/>
</div>

根据提示修改登录密码并完成登录


### 创建私服仓库
创建仓库，点击`Create repository`,然后选择`maven2(hosted)`然后输入仓库名称`（nexus-release`）。在`version policy`中选择这个仓库的类型，这里选择`release`,在D`eployment policy`中选择`Allow redeploy`，（这个很重要），如下图所示

<div style="width:780px;height:450px;margin:50px auto">
    <img alt="nexus-r.png" src="/images/nexus-r.png" width="780" height="450"/>
</div>

### 配置本地Maven配置文件settings.xml

在本地Maven配置文件settings.xml中，添加如下内容：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
    <servers>
      <server>
        <id>releases</id>
        <username>admin</username>
        <password>admin123</password>
      </server>
    </servers>
</settings>
```

使用IDEA创建一个Maven项目：

pop.xml配置文件如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.mk</groupId>
    <artifactId>sso-sdk</artifactId>
    <version>1.0-RELEASE</version>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <servlet-api.version>3.1.0</servlet-api.version>
        <commons-lang.version>2.6</commons-lang.version>
        <lombok.version>1.18.0</lombok.version>
        <fastjson.version>1.2.60</fastjson.version>
        <mybatis-plus.version>3.1.1</mybatis-plus.version>
    </properties>


    <!--指定仓库地址-->
    <distributionManagement>
        <repository>
            <!--此名称要和maven/settings.xml中设置的ID一致-->
            <id>releases</id>
            <url>http://192.168.192.10:8081/repository/next-release/</url>
        </repository>
    </distributionManagement>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.4.RELEASE</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
        </dependency>
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>${servlet-api.version}</version>
        </dependency>
        <dependency>
            <groupId>commons-lang</groupId>
            <artifactId>commons-lang</artifactId>
            <version>${commons-lang.version}</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>${fastjson.version}</version>
        </dependency>
        <!-- mybatis-plus依赖 -->
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>${mybatis-plus.version}</version>
        </dependency>
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus</artifactId>
            <version>${mybatis-plus.version}</version>
        </dependency>
    </dependencies>
</project>
```
接下来通过执行`mvn deploy`命令发布到Maven私服，如下便说明发布成功

```
Uploading to releases: http://192.168.192.10:8081/repository/next-release/com/mk/sso-sdk/1.0-RELEASE/sso-sdk-1.0-RELEASE.jar
Uploaded to releases: http://192.168.192.10:8081/repository/next-release/com/mk/sso-sdk/1.0-RELEASE/sso-sdk-1.0-RELEASE.jar (14 kB at 1.7 kB/s)
Uploading to releases: http://192.168.192.10:8081/repository/next-release/com/mk/sso-sdk/1.0-RELEASE/sso-sdk-1.0-RELEASE.pom
Uploaded to releases: http://192.168.192.10:8081/repository/next-release/com/mk/sso-sdk/1.0-RELEASE/sso-sdk-1.0-RELEASE.pom (3.0 kB at 1.4 kB/s)
Downloading from releases: http://192.168.192.10:8081/repository/next-release/com/mk/sso-sdk/maven-metadata.xml
[DEBUG] Could not find metadata com.mk:sso-sdk/maven-metadata.xml in releases (http://192.168.192.10:8081/repository/next-release/)
[DEBUG] Writing tracking file D:\soft\work\maven_repository\com\mk\sso-sdk\resolver-status.properties
Uploading to releases: http://192.168.192.10:8081/repository/next-release/com/mk/sso-sdk/maven-metadata.xml
Uploaded to releases: http://192.168.192.10:8081/repository/next-release/com/mk/sso-sdk/maven-metadata.xml (305 B at 346 B/s)
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 19.669 s
[INFO] Finished at: 2020-06-03T12:26:55+08:00
[INFO] Final Memory: 25M/278M
[INFO] ------------------------------------------------------------------------
```

再到Maven私服中查看，jar包已被发布到仓库，如下：

<div style="width:780px;height:450px;margin:50px auto">
    <img alt="nexus-deploy.png" src="/images/nexus-deploy.png" width="780" height="450"/>
</div>


如其他项目需要依赖上面的jar包，可以在pom.xml配置文件中添加如下配置

```
<dependencies>
    <dependency>
      <groupId>com.mk</groupId>
      <artifactId>sso-sdk</artifactId>
      <version>1.0-RELEASE</version>
    </dependency>
</dependencies>

<repositories>
    <repository>
        <!--此名称要和maven/settings.xml中设置的ID一致-->
        <id>releases</id>
        <url>http://192.168.192.10:8081/repository/next-release/</url>
    </repository>
</repositories>
```