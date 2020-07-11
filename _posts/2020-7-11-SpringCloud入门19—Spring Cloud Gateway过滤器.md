---
layout: post
title: "Spring Cloud Gateway过滤器"
date: 2020-7-11
categories: 微服务 分布式
tags: SpringCloud Gateway
--- 

在前面我们介绍了Gateway的Predict，Predict决定了请求由哪一个路由处理，在路由处理之前，需要经过“pre”类型的过滤器处理，处理返回响应之后，可以由“post”类型的过滤器处理。

**filter的作用和生命周期**

由filter工作流程点，可以知道filter有着非常重要的作用，在“pre”类型的过滤器可以做参数校验、权限校验、流量监控、日志输出、协议转换等，在“post”类型的过滤器中可以做响应内容、响应头的修改，日志的输出，流量监控等。首先需要弄清一点为什么需要网关这一层，这就不得不说下filter的作用了。

**作用**

当我们有很多个服务时，客户端请求各个服务的Api时，每个服务都需要做相同的事情，比如鉴权、限流、日志输出等。

对于这样重复的工作，有没有办法做的更好，答案是肯定的。在微服务的上一层加一个全局的权限控制、限流、日志输出的Api Gatewat服务，然后再将请求转发到具体的业务服务层。这个Api Gateway服务就是起到一个服务边界的作用，外接的请求访问系统，必须先通过网关层。

**生命周期**

Spring Cloud Gateway同zuul类似，有“pre”和“post”两种方式的filter。客户端的请求先经过“pre”类型的filter，然后将请求转发到具体的业务服务，收到业务服务的响应之后，再经过“post”类型的filter处理，最后返回响应到客户端。

与zuul不同的是，filter除了分为“pre”和“post”两种方式的filter外，在Spring Cloud Gateway中，filter从作用范围可分为另外两种，一种是针对于单个路由的gateway filter，它在配置文件中的写法同predict类似；另外一种是针对于所有路由的global gateway filer。现在从作用范围划分的维度来讲解这两种filter。

**gateway filter**

Spring Cloud Gateway 中内置了很多过滤器工厂，直接采用配置的方式使用即可，同时也支持自定义 GatewayFilter Factory 来实现更复杂的业务需求。

接下来为大家介绍几个常用的过滤器工厂类。

**1. AddRequestHeader 过滤器工厂**

通过名称我们可以快速明白这个过滤器工厂的作用是添加请求头。

符合规则匹配成功的请求，将添加 X-Request-Foo：bar 请求头，将其传递到后端服务中，后方服务可以直接获取请求头信息。我们首先修改一下eureka-client-provider服务，代码如下所示。

```java
@GetMapping("/hello")
public String hello(HttpServletRequest request, @RequestParam String name) {
    System.out.println(request.getHeader("X-Request-Foo"));
    return "hello, " + name + ", 我是服务提供者:端口为：" + port;
}
```

然后在service-gateway服务的application.yml配置文件中添加以下路由配置:

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: hello
        predicates:   #断言
        - Path=/hello
        uri: http://localhost:8080  #转发到eureka-client-provider服务
        filters:
        - AddRequestHeader=X-Request-Foo, Bar
```

启动每个服务，在浏览器访问[http://localhost/hello?name=xiaoming](http://localhost/hello?name=xiaoming)，可以看到转发到eureka-client-provider服务的hello接口，并且将X-Request-Foo的header信息传递过去了，如下：


<div style="width:780px;height:500px;margin:50px auto;">
    <img alt="route-filter1.png" src="/images/route-filter1.png" width="780" height="500"/>
</div>


2. RemoveRequestHeader 过滤器工厂

RemoveRequestHeader 是移除请求头的过滤器工厂，可以在请求转发到后端服务之前进行 Header 的移除操作。

修改service-gateway服务的application.yml配置，如下：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: hello
        predicates:   #断言
        - Path=/hello
        uri: http://localhost:8080  #目标服务地址
        filters:
        - AddRequestHeader=X-Request-Foo, Bar
        - RemoveRequestHeader=X-Request-Foo
```

重新启动service-gateway服务，在浏览器访问[http://localhost/hello?name=xiaoming](http://localhost/hello?name=xiaoming)，可以看到转发到eureka-client-provider服务的hello接口，但是此时X-Request-Foo的header信息为空，如下：

<div style="width:780px;height:500px;margin:50px auto;">
    <img alt="route-filter2.png" src="/images/route-filter2.png" width="780" height="500"/>
</div>

说明RemoveRequestHeader过滤器工厂生效了

本文代码git地址 [https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)