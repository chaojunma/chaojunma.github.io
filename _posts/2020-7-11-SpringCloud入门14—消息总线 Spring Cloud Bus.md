---
layout: post
title: "消息总线 Spring Cloud Bus"
date: 2020-7-11
categories: 微服务 分布式
tags: SpringCloud
--- 

Spring Cloud Bus 将分布式的节点用轻量的消息代理连接起来。它可以用于广播配置文件的更改或者服务之间的通讯，也可以用于监控。这节课讲述的是用Spring Cloud Bus实现通知微服务架构的配置文件的更改。
准备工作

本文还是基于上一篇文章来实现。按照官方文档，我们只需要在配置文件中配置
spring-cloud-starter-bus-amqp ；这就是说我们需要装rabbitmq。至于怎么使用 rabbitmq，请自行百度


**改造config-client**

在pom文件加上起步依赖spring-cloud-starter-bus-amqp，完整的配置文件如下：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- spring cloud config 客户端包 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
        <version>2.0.4.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-amqp</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

在配置文件bootstrap.yml中加上rabbitmq的配置，包括rabbitmq的地址、端口，用户名、密码。并需要加上spring.cloud.bus的三个配置，具体如下：

```yaml
server:
  port: 8881
spring:
  application:
    name: config-client
  cloud:
    config:
      uri: http://localhost:8888/
      label: master
      profile: dev
    bus:
      enabled: true
      trace:
        enabled: true
  rabbitmq:
    host: 192.168.192.10
    port: 5672
    username: guest
    password: guest
management:
  endpoints:
    web:
      exposure:
        include: bus-refresh
```

在Application启动类添加`@RefreshScope`注解 ，代码如下

```java
@RefreshScope
@RestController
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
    @Value("${user.name}")
    String name;
    @RequestMapping(value = "/hello")
    public String hi(){
        return "hello," + name;
    }
}
```

分别启动config-server和config-client服务，在浏览器中访问[http://localhost:8881/hello](http://localhost:8881/hello) 返回信息如下：

```
hello,zhangsan
```

将git配置中的user.name属性改为如下：

<div style="width:780px;height:350px;margin:50px auto;">
    <img alt="git-config1.png" src="/images/git-config1.png" width="780" height="350"/>
</div>

通过postman发送post请求调用[localhost:8881/actuator/bus-refresh](localhost:8881/actuator/bus-refresh)，如下：


<div style="width:780px;height:350px;margin:50px auto;">
    <img alt="postman.png" src="/images/postman.png" width="780" height="350"/>
</div>

配置刷新后在浏览器再次访问[http://localhost:8881/hello](http://localhost:8881/hello)，返回信息如下：

```
hello,lisi
```

此时说明刷新配置生效了。


本文代码git地址 [https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)