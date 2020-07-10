---
layout: post
title: "创建服务消费通过Fegin调用"
date: 2020-7-10
categories: 微服务 分布式
tags: SpringCloud Eureka
--- 

前面，我们讲述了如何通过RestTemplate+Ribbon去消费服务，现在我们讲述一下如何通过Feign去消费服务。
Feign是一个声明式的伪Http客户端，它使得写Http客户端变得更简单。使用Feign，只需要创建一个接口并注解，就可以实现调用。Feign默认集成了Ribbon，并和Eureka结合，默认实现了负载均衡的效果。

简而言之：

Feign 采用的是基于接口的注解

Feign 整合了ribbon，具有负载均衡的能力

整合了Hystrix，具有熔断的能力

首先在父模块下，创建一个名称为consumer-fegin，pom.xml文件配置如下：

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
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
</dependencies>
```

在resources目录下新增application.yml配置文件，配置如下：

```yaml
server:
  port: 8091

spring:
  application:
    name: consumer-fegin

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    instance-id: ${spring.cloud.client.ip-address}:${server.port}
```

创建启动类，加上`@EnableFeignClients`注解开启Feign的功能，如下：

```java
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients
public class ConsumerFeginApp {

    public static void main(String[] args) {
        SpringApplication.run(ConsumerFeginApp.class);
    }

}
```

定义一个feign接口，通过`@ FeignClient（"服务名"）`，来指定调用哪个服务，代码如下：

```java
@FeignClient("eureka-client-provider")
public interface HelloFeginClient {
    @GetMapping("/hello")
    String hello(@RequestParam(value = "name") String name);
}
```

接下来创建一个HelloController,代码如下：

```java
@RestController
public class HelloController {

    @Autowired
    private HelloFeginClient helloFeginClient;

    @GetMapping(value = "/hello")
    public String sayHi(@RequestParam String name) {
        return "通过fegin客户端调用结果为：" + helloFeginClient.hello(name);
    }
}
```

启动服务，通过浏览器访问[http://localhost:8091/hello?name=xiaoming](http://localhost:8091/hello?name=xiaoming)，返回结果如下：

```
通过fegin客户端调用结果为：hello, xiaoming, 我是服务提供者:端口为：8080
```

本文代码git地址 [https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)