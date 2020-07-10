---
layout: post
title: "路由网关Zuul"
date: 2020-7-10
categories: 微服务 分布式
tags: SpringCloud Zuul
--- 


**Zuul简介**

Zuul的主要功能是路由转发和过滤器。路由功能是微服务的一部分，比如／api/user转发到到user服务，/api/shop转发到到shop服务。zuul默认和Ribbon结合实现了负载均衡的功能。

**准备工作**

首先在父模块下，创建一个名称为service-zuul，pom.xml文件配置如下：

```xml
<parent>
    <artifactId>springcloud-lesson</artifactId>
    <groupId>com.mk</groupId>
    <version>1.0-SNAPSHOT</version>
</parent>
<modelVersion>4.0.0</modelVersion>

<artifactId>service-zuul</artifactId>

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
    </dependency>
</dependencies>
```

创建启动类ZuulApplication，并添加`@EnableZuulProxy`注解开启zuul网关的功能，如下：

```java
@SpringBootApplication
@EnableZuulProxy
@EnableEurekaClient
public class ZuulApplication {

    public static void main(String[] args) {
        SpringApplication.run(ZuulApplication.class);
    }
}
```

新建配置文件application.yml，并添加以下的配置：

```yaml
server:
  port: 8769
spring:
  application:
    name: service-zuul

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    instance-id: ${spring.cloud.client.ip-address}:${server.port}

zuul:
  routes:
    api-a:
      path: /api-a/**
      serviceId: consumer-ribbon
    api-b:
      path: /api-b/**
      serviceId: consumer-fegin
```

首先指定服务注册中心的地址为http://localhost:8761/eureka/，服务的端口为8769，服务名为service-zuul；以/api-a/ 开头的请求都转发给consumer-ribbon服务；以/api-b/开头的请求都转发给consumer-feign服务；


启动服务;打开浏览器访问：[http://localhost:8769/api-a/hello?name=xiaoming](http://localhost:8769/api-a/hello?name=xiaoming);浏览器显示：

```
hello, xiaoming, 我是服务提供者:端口为：8080
```

启动服务;打开浏览器访问：[http://localhost:8769/api-b/hello?name=xiaoming](http://localhost:8769/api-b/hello?name=xiaoming);浏览器显示：

```
通过fegin客户端调用结果为：hello, xiaoming, 我是服务提供者:端口为：8080
```


这说明zuul的路由转发功能生效了

本文代码git地址 [https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)