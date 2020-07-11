---
layout: post
title: "Spring Cloud Gateway初体验"
date: 2020-7-11
categories: 微服务 分布式
tags: SpringCloud Gateway
--- 

Spring Cloud Gateway是Spring Cloud官方推出的第二代网关框架，取代Zuul网关。网关在微服务系统中有着非常重要的作用，网关常见的功能有路由转发、权限校验、限流等等。

**构建gateway**

首先在父模块下，创建一个名称为service-gateway的子模块，pom.xml文件配置如下：

```xml
<dependencies>
	<dependency>
	    <groupId>org.springframework.cloud</groupId>
	    <artifactId>spring-cloud-starter-gateway</artifactId>
	</dependency>
</dependencies>
```

创建application.yml配置文件，配置如下

```yaml
server:
  port: 80
spring:
  application:
    name: service-gateway
```

创建启动类如下：

```java
@SpringBootApplication
public class GatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class);
    }
}
```

创建网关配置如下：

```java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator myRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
                .route(p -> p
                        .path("/baidu")
                        .filters(f -> f.addRequestHeader("Hello", "World"))
                        .uri("http://www.baidu.com"))
                .build();
    }
}
```

在浏览器访问[http://localhost/baidu](http://localhost/baidu)会转发到百度，如下：

<div style="width:780px;height:500px;margin:50px auto;">
    <img alt="baidu.png" src="/images/baidu.png" width="780" height="500"/>
</div>


本文代码git地址 [https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)