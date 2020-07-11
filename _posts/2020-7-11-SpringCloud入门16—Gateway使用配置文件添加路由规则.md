---
layout: post
title: "Gateway使用配置文件添加路由规则"
date: 2020-7-11
categories: 微服务 分布式
tags: SpringCloud Gateway
--- 

前面我们利用注解的方式添加路由规则，这一节我们学习一下如何在配置文件中添加路由规则

改造一下service-gateway服务，首先注释之前创建的网关配置类，并在application.yml配置文件中添加如下配置:

```yaml
server:
  port: 80
spring:
  application:
    name: service-gateway
  cloud:
    gateway:
      routes:
      - id: baidu
        predicates:   #断言
        - Path=/baidu  #匹配规则
        uri: http://www.baidu.com  #目标服务地址
```


在浏览器访问[http://localhost/baidu](http://localhost/baidu)会转发到百度，和之前通过注解方式配置路由实现的效果一样

<div style="width:780px;height:500px;margin:50px auto;">
    <img alt="baidu.png" src="/images/baidu.png" width="780" height="500"/>
</div>



本文代码git地址 [https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)