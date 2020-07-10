---
layout: post
title: "fegin使用断路器 Hystrix"
date: 2020-7-10
categories: 微服务 分布式
tags: SpringCloud Hystrix
--- 


Feign是自带断路器的，在D版本的Spring Cloud之后，它没有默认打开。需要在配置文件中配置打开它，在application.yml配置文件添加以下配置：

```yaml
feign:
  hystrix:
    enabled: true
```

基于consumer-feign工程进行改造，只需要在HelloFeginClient接口的FeignClient注解中加上fallback的指定类就行了：

```java
@FeignClient(value = "eureka-client-provider", fallback = HelloFeginFallback.class)
public interface HelloFeginClient {

    @GetMapping("/hello")
    String hello(@RequestParam(value = "name") String name);
}
```

HelloFeginFallback需要实现HelloFeginClient 接口，并注入到Ioc容器中，代码如下：

```java
@Component
public class HelloFeginFallback implements HelloFeginClient {

    @Override
    public String hello(String name) {
        return "hello," + name + ",sorry,error!";
    }
}
```

启动四consumer-feign工程，浏览器打开[http://localhost:8091/hello?name=xiaoming](http://localhost:8091/hello?name=xiaoming),注意此时eureka-client-provider工程没有启动，网页显示：

```
通过fegin客户端调用结果为：hello,xiaoming,sorry,error!
```

启动eureka-client-provider服务，再次访问，浏览器显示：

```
通过fegin客户端调用结果为：hello, xiaoming, 我是服务提供者:端口为：8080
```

这证明断路器起到作用了。


本文代码git地址 [https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)