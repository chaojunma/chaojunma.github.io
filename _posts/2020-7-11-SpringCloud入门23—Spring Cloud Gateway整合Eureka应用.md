---
layout: post
title: "Spring Cloud Gateway整合Eureka应用"
date: 2020-7-11
categories: 微服务 分布式
tags: SpringCloud Gateway Eureka
--- 


前面我们学习了路由转发，但是目标服务配置的都是外部服务的形势，那么网关能够通过注册的Eureka注册中心，实现内部服务的转发吗？答案是肯定的，下面我们来具体实现一下

下面的示例启动两个服务：service-gataway 和 eureka-client-provider 都注册到注册中心 Eureka上，客户端请求后端服务【eureka-client-provider】,网关提供后端服务的统一入口。网关通过负载均衡将客户端请求转发到具体的后端服务。


序号 |	服务名称 |	端口号 |	备注
-|-|-|-|
1 |	gateway-server |	80	| 网关服务 |
2|	eureka-client-provider|	8080|	服务提供者|
3|	eureka-server|	8761|	注册中心|

**具体实现**

首先在service-gateway服务的pom.xml中添加以下依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

在工程的配置文件application.yml中，配置信息如下：

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    instance-id: ${spring.cloud.client.ip-address}:${server.port}
```

分别启动eureka-server  eureka-client-provider service-gateway 服务，在浏览器访问[http://localhost:8761/](http://localhost:8761/)，注册中心显示如下：

<div style="width:780px;height:500px;margin:50px auto;">
    <img alt="gateway-eureka.png" src="/images/gateway-eureka.png" width="780" height="500"/>
</div>


在浏览器访问[http://localhost/eureka-client-provider/hello?name=xiaoming&token=123](http://localhost/eureka-client-provider/hello?name=xiaoming&token=123)，返回如下：

```
hello, xiaoming, 我是服务提供者:端口为：8080
```

此时说明网关转发内部服务成功


本文代码git地址 [https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)