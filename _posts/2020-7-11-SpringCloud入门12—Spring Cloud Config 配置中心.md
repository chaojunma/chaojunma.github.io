---
layout: post
title: "Spring Cloud Config 配置中心"
date: 2020-7-11
categories: 微服务 分布式
tags: SpringCloud
--- 

**Spring Cloud Config简介**

在分布式系统中，由于服务数量巨多，为了方便服务配置文件统一管理，实时更新，所以需要分布式配置中心组件。在Spring Cloud中，有分布式配置中心组件spring cloud config ，它支持配置服务放在配置服务的内存中（即本地），也支持放在远程Git仓库中。在spring cloud config 组件中，分两个角色，一是config server，二是config client。

**构建Config Server**

首先在父模块下，创建一个名称为config-server，pom.xml文件配置如下：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!-- spring cloud config 服务端包 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

创建bootstrap.yml 文件，配置如下：

```yaml
server:
  port: 8888
spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/xmingtx/springcloud-config.git #配置文件所在仓库
          username: xmingtx #github登录账号
          password: 123456a #github登录密码
          default-label: master #配置文件分支
          search-paths: config  #配置文件所在根目录
```

在程序的入口Application类加上`@EnableConfigServer`注解开启配置服务器的功能，代码如下：

```java
@SpringBootApplication
@EnableConfigServer
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

启动服务，接下来测试一下。

Spring Cloud Config 有它的一套访问规则，我们通过这套规则在浏览器上直接访问就可以。

```
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties
```

{application} 就是应用名称，对应到配置文件上来，就是配置文件的名称部分，例如我上面创建的配置文件。
{profile} 就是配置文件的版本，我们的项目有开发版本、测试环境版本、生产环境版本，对应到配置文件上来就是以 application-{profile}.yml 加以区分，例如application-dev.yml、application-sit.yml、application-prod.yml。
{label} 表示 git 分支，默认是 master 分支，如果项目是以分支做区分也是可以的，那就可以通过不同的 label 来控制访问不同的配置文件了。

通过访问以上地址，如果可以正常返回数据，则说明配置中心服务端一切正常。如下：

<div style="width:780px;height:264px;margin:50px 0px;">
    <img alt="config.png" src="/images/config.png" width="780" height="264"/>
</div>

本文代码git地址 [https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)