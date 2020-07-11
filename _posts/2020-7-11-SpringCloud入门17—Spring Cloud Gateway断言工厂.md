---
layout: post
title: "Spring Cloud Gateway断言工厂"
date: 2020-7-11
categories: 微服务 分布式
tags: SpringCloud Gateway
--- 

通过前面的学习，大家已经对Spring Cloud Gateway的功能有一个初步的认识，网关作为一个系统的流量的入口，有着举足轻重的作用，通常的作用如下：

- 协议转换，路由转发
- 流量聚合，对流量进行监控，日志输出
= 作为整个系统的前端工程，对流量进行控制，有限流的作用
- 作为系统的前端边界，外部流量只能通过网关才能访问系统
- 可以在网关层做权限的判断
- 可以在网关层做缓存

<div style="width:300;height:500px;margin:50px auto;">
    <img alt="route.png" src="/images/route.png" width="300" height="500"/>
</div>

如上图所示，客户端向Spring Cloud Gateway发出请求。 如果Gateway Handler Mapping确定请求与路由匹配（这个时候就用到predicate），则将其发送到Gateway web handler处理。 Gateway web handler处理请求时会经过一系列的过滤器链。 过滤器链被虚线划分的原因是过滤器链可以在发送代理请求之前或之后执行过滤逻辑。 先执行所有“pre”过滤器逻辑，然后进行代理请求。 在发出代理请求之后，收到代理服务的响应之后执行“post”过滤器逻辑。这跟zuul的处理过程很类似。在执行所有“pre”过滤器逻辑时，往往进行了鉴权、限流、日志输出等功能，以及请求头的更改、协议的转换；转发之后收到响应之后，会执行所有“post”过滤器的逻辑，在这里可以响应数据进行了修改，比如响应头、协议的转换等。

在上面的处理过程中，有一个重要的点就是讲请求和路由进行匹配，这时候就需要用到predicate，它是决定了一个请求走哪一个路由。

Spring Cloud Gateway 内置了许多路由断言工厂，可以通过配置的方式直接使用，也可以组合使用多个路由断言工厂。接下来为大家介绍几个常用的路由断言工厂类。

**1）Path 路由断言工厂**

Path 路由断言工厂接收一个参数，根据 Path 定义好的规则来判断访问的 URI 是否匹配。

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: baidu
        predicates:   #断言
        - Path=/baidu
        uri: http://www.baidu.com  #目标服务地址
```
如果请求路径为 /baidu，则此路由将匹配

**2）Query 路由断言工厂**

Query 路由断言工厂接收两个参数，一个必需的参数和一个可选的正则表达式。

```yaml
spring:
  application:
    name: service-gateway
  cloud:
    gateway:
      routes:
      - id: query_route
        predicates:
        - Query=foo, ba.
        uri: http://c.biancheng.net
```
如果请求包含一个值与 ba 匹配的 foo 查询参数，则此路由将匹配。bar 和 baz 也会匹配，因为第二个参数是正则表达式。

在浏览器请求 [http://localhost?foo=baz](http://localhost?foo=baz) 会转发到 http://c.biancheng.net 目标服务

**3）Method 路由断言工厂**

Method 路由断言工厂接收一个参数，即要匹配的 HTTP 方法。

```yaml
spring:
  application:
    name: service-gateway
  cloud:
    gateway:
      routes:
      - id: method_route
        predicates:
        - Method=GET
        uri: http://jd.com
```

**4）Header 路由断言工厂**

Header 路由断言工厂接收两个参数，分别是请求头名称和正则表达式。

```yaml
spring:
  application:
    name: service-gateway
  cloud:
    gateway:
      routes:
      - id: header_route
        predicates:
        - Header=X-Request-Id, \d+
        uri: http://example.org
```

如果请求中带有请求头名为 x-request-id，其值与 \d+ 正则表达式匹配（值为一个或多个数字），则此路由匹配。


如果你想学习更多路由断言工厂的用法，可以参考官方文档进行学习。




本文代码git地址 [https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)