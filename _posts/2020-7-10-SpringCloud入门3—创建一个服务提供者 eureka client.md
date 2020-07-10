---
layout: post
title: "创建一个服务提供者 Eureka Client"
date: 2020-7-10
categories: 微服务 分布式
tags: SpringCloud Eureka
--- 

当client向server注册时，它会提供一些元数据，例如服务地址和端口等等。Eureka server 从每个client实例接收心跳消息。 如果心跳超时，则通常将该实例从注册server中删除。

创建过程同server类似,在父模块下创建一个eureka-client-provider,引入以下相关依赖：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

然后创建一个启动类，通过注解@EnableEurekaClient 表明自己是一个eurekaclient，代码如下：

```java
@SpringBootApplication
@EnableEurekaClient
@RestController
public class ProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run( ProviderApplication.class, args );
    }

    @Value("${server.port}")
    String port;

    @GetMapping("/hello")
    public String hello(@RequestParam String name) {
        return "hello, " + name + ", 我是服务提供者:端口为：" + port;
    }
}
```

仅仅`@EnableEurekaClient`是不够的，还需要在配置文件中注明自己的服务注册中心的地址，application.yml配置文件如下：

```yaml
server:
  port: 8080

spring:
  application:
    name: eureka-client-provider

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    instance-id: ${spring.cloud.client.ip-address}:${server.port}
```

分别启动eureka-server和eureka-client-provider服务，打开浏览器访问http://localhost:8761，我们会发现注册中心多了一个服务，如下：

<div style="width:780px;height:384px;margin:50px auto">
    <img alt="eureka-client.png" src="/images/eureka-client.png" width="780" height="384"/>
</div>


在浏览器中访问[localhost:8080/hello?name=xiaoming](localhost:8080/hello?name=xiaoming)，返回如下结果：

```
hello, xiaoming, 我是服务提供者:端口为：8080
```

本文代码git地址[https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)