---
layout: post
title: "ribbon使用断路器 Hystrix"
date: 2020-7-10
categories: 微服务 分布式
tags: SpringCloud Hystrix
--- 


在微服务架构中，根据业务来拆分成一个个的服务，服务与服务之间可以相互调用（RPC），在Spring Cloud可以用RestTemplate+Ribbon和Feign来调用。为了保证其高可用，单个服务通常会集群部署。由于网络原因或者自身的原因，服务并不能保证100%可用，如果单个服务出现问题，调用这个服务就会出现线程阻塞，此时若有大量的请求涌入，Servlet容器的线程资源会被消耗完毕，导致服务瘫痪。服务与服务之间的依赖性，故障会传播，会对整个微服务系统造成灾难性的严重后果，这就是服务故障的“雪崩”效应。

为了解决这个问题，业界提出了断路器模型。

<div style="width:580px;height:464px;margin:50px 10px;">
    <img alt="Hystrix.png" src="/images/Hystrix.png" width="580" height="464"/>
</div>


**在ribbon使用断路器**

在前面我们创建了一个consumer-ribbon服务，我们来改造一下，首先需要在pox.xml文件中加入hystrix的依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

在程序的启动类ConsumerApplication加`@EnableHystrix`注解开启Hystrix，如下：

```java
@EnableHystrix
@EnableEurekaClient
@SpringBootApplication
public class ConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConsumerApplication.class);
    }

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

创建HelloService类，代码如下：

```java
@Service
public class HelloService {

    @Autowired
    private RestTemplate restTemplate;

    @HystrixCommand(fallbackMethod = "helloFallback")
    public String callHello(String name) {
        return restTemplate.getForObject("http://eureka-client-provider/hello?name="+name,String.class);
    }

    public String helloFallback(String name) {
        return "hello," + name + ",sorry,error!";
    }
}
```

改造一下HelloController，代码如下：

```java
@RestController
public class HelloController {

    @Autowired
    private HelloService helloService;

    @GetMapping(value = "/hello")
    public String hello(@RequestParam String name) {
        return helloService.callHello(name);
    }
}
```

启动所有服务，我们在浏览器访问[http://localhost:8090/hello?name=xiaoming](http://localhost:8090/hello?name=xiaoming),返回以下结果:

```
hello, xiaoming, 我是服务提供者:端口为：8080
```

停止eureka-client-provider服务，再次访问，返回结果如下：

```
hello,xiaoming,sorry,error!
```

这证明断路器起到作用了。


本文代码git地址 [https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)