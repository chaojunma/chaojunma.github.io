---
layout: post
title: "手动创建负载均衡器"
date: 2020-7-10
categories: 微服务 分布式
tags: SpringCloud Eureka
--- 

负载均衡常用的方式有轮询模式、随机模式等等，下面我们来实现一下，在consumer-ribbon服务添加如下代码：

```java
@Component
public class LoadBalancer {

    AtomicInteger counter = new AtomicInteger(0);

    /**
     * 轮询模式
     * @param instances
     * @return
     */
    public ServiceInstance getInstanceByCycle(List<ServiceInstance> instances){
        int index = counter.incrementAndGet() % 2;
        return instances.get(index);
    }

    /**
     * 随机模式
     * @param instances
     * @return
     */
    public ServiceInstance getInstanceByRandom(List<ServiceInstance> instances){
        int index = new Random().nextInt(instances.size());
        return instances.get(index);
    }
}

```

启动两个eureka-client-provider服务，端口分别为8080和8081，Eureka注册中心存在两个实例，如下：

<div style="width:780px;height:384px;margin:50px auto">
    <img alt="ribbon1.png" src="/images/ribbon1.png" width="780" height="384"/>
</div>

修改消费者服务consumer-ribbon的HelloController，代码如下：

```java
@RestController
public class HelloController {

    @Autowired
    private DiscoveryClient discoveryClient;

    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    private LoadBalancer balancer;

    @GetMapping(value = "/hello")
    public String hello(@RequestParam String name) {
        List<ServiceInstance> instances = discoveryClient.getInstances("eureka-client-provider");
        ServiceInstance instance = balancer.getInstanceByCycle(instances);
        return restTemplate.getForObject(instance.getUri() + "/hello?name=" + name, String.class);
    }
}
```

启动消费者服务，在浏览器访问http://localhost:8090/hello?name=xiaoming，返回结果轮序交替，如下：

```
hello, xiaoming, 我是服务提供者:端口为：8080
```
```
hello, xiaoming, 我是服务提供者:端口为：8081
```

将消费者服务中改为随机模式调用，生产者服务则被随机调到。