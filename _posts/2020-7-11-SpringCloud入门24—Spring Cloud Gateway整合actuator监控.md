---
layout: post
title: "Spring Cloud Gateway整合actuator监控"
date: 2020-7-11
categories: 微服务 分布式
tags: SpringCloud Gateway Eureka
--- 

我们可以通过Spring Cloud Gateway整合actuator监控，查看网关的配置信息，下面我们来实现一下

首先在service-gateway服务pom.xml文件添加如下依赖：

```yaml
<!--actuator 监控-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

然后在服务的application.yml文件中添加如下配置：

```
#开放所有页面节点  默认只开启了health、info两个节点
management:
  endpoints:
    web:
      exposure:
        include:  '*'
```

重启service-gateway服务，在浏览器访问[http://localhost/actuator/gateway/routes](http://localhost/actuator/gateway/routes)，返回网关配置的路由信息，如下：

<div style="width:780px;height:180px;margin:50px auto;">
    <img alt="actuator.png" src="/images/actuator.png" width="780" height="180"/>
</div>


本文代码git地址 [https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)