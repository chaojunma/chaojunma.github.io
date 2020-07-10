---
layout: post
title: "创建一个服务消费者 ribbon+restTemplate"
date: 2020-7-10
categories: 微服务 分布式
tags: SpringCloud Eureka
--- 

在前面，讲了服务的注册和发现。在微服务架构中，业务都会被拆分成一个独立的服务，服务与服务的通讯是基于http restful的。Spring cloud有两种服务调用方式，一种是ribbon+restTemplate，另一种是feign。现在我们首先讲解下基于ribbon+restTemplate。

首先我们在父模块下创建一个子模块，名称为consumer-ribbon，pom.xml依赖如下：

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
        <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
    </dependency>
</dependencies>
```


在resources目录下创建application.yml配置文件，配置如下：

```yaml
server:
  port: 8090

spring:
  application:
    name: consumer-ribbon

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    instance-id: ${spring.cloud.client.ip-address}:${server.port}
```

然后创建一个启动类，通过注解@EnableEurekaClient 表明自己是一个eurekaclient，代码如下：

```java
@EnableEurekaClient
@SpringBootApplication
public class ConsumerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConsumerApplication.class);
    }
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

创建HelloControler，代码如下：

```java
@RestController
public class HelloController {

    @Autowired
    private DiscoveryClient discoveryClient;

    @Autowired
    private RestTemplate restTemplate;

    @GetMapping(value = "/hello")
    public String hello(@RequestParam String name) {
        List<ServiceInstance> instances = discoveryClient.getInstances("eureka-client-provider");
        ServiceInstance instance = instances.get(0);
        return restTemplate.getForObject(instance.getUri() + "/hello?name=" + name, String.class);
    }
}
```

启动服务，eureka注册中心又多了一个服务，如下：

<div style="width:780px;height:384px;margin:50px auto">
    <img alt="ribbon.png" src="/images/ribbon.png" width="780" height="384"/>
</div>

在浏览器访问[http://localhost:8090/hello?name=xiaoming](http://localhost:8090/hello?name=xiaoming)，返回结果如下：

```
hello, xiaoming, 我是服务提供者:端口为：8080
```

说明通过消费者调用服务提供者的服务调用成功了