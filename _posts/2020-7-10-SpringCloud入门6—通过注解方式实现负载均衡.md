---
layout: post
title: "通过注解方式实现负载均衡"
date: 2020-7-10
categories: 微服务 分布式
tags: SpringCloud Eureka
--- 

除了上面可以手动实现负载均衡模式，另外也提供了注解模式实现了负载均衡，在RestTemplate配置上添加`@LoadBalanced`注解，如下：

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}
```

```java
@GetMapping(value = "/hello")
    public String hello(@RequestParam String name) {
        return restTemplate.getForObject("http://eureka-client-provider//hello?name=" + name, String.class);
    }
```

启动消费者服务，在浏览器访问http://localhost:8090/hello?name=xiaoming，返回结果轮序交替，如下：

```
hello, xiaoming, 我是服务提供者:端口为：8080
```
```
hello, xiaoming, 我是服务提供者:端口为：8081
```

说明默认的负载均衡机制采用的是轮询模式

本文代码git地址 [https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)