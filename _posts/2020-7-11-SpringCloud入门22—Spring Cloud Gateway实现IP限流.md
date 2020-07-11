---
layout: post
title: "Spring Cloud Gateway实现IP限流"
date: 2020-7-11
categories: 微服务 分布式
tags: SpringCloud Gateway 限流
--- 


在高并发的系统中，往往需要在系统中做限流，一方面是为了防止大量的请求使服务器过载，导致服务不可用，另一方面是为了防止网络攻击。这节课详细探讨在 Spring Cloud Gateway 中如何实现限流

常见的限流算法

- 滑动时间窗口算法  
- 漏桶算法 
- 令牌桶算法


从某种意义上讲，令牌桶算法是对漏桶算法的一种改进，桶算法能够限制请求调用的速率，而令牌桶算法能够在限制调用的平均速率的同时还允许一定程度的突发调用。在令牌桶算法中，存在一个桶，用来存放固定数量的令牌。算法中存在一种机制，以一定的速率往桶中放令牌。每次请求调用需要先获取令牌，只有拿到令牌，才有机会继续执行，否则选择选择等待可用的令牌、或者直接拒绝。放令牌这个动作是持续不断的进行，如果桶中令牌数达到上限，就丢弃令牌，所以就存在这种情况，桶中一直有大量的可用令牌，这时进来的请求就可以直接拿到令牌执行，比如设置qps为100，那么限流器初始化完成一秒后，桶中就已经有100个令牌了，这时服务还没完全启动好，等启动完成对外提供服务时，该限流器可以抵挡瞬时的100个请求。所以，只有桶中没有令牌时，请求才会进行等待，最后相当于以一定的速率执行。

**网关IP限流实现**

改造service-gateway服务，在pom.xml中添加以下依赖

```xml
<dependency>
    <groupId>com.github.vladimir-bukhtoyarov</groupId>
    <artifactId>bucket4j-core</artifactId>
    <version>4.0.0</version>
</dependency>
```

新建获取IP地址工具类，代码如下：

```java
@Slf4j
public class IpUtil {

    public static String getIpAddr(ServerHttpRequest request) {

        HttpHeaders headers = request.getHeaders();
        String ipAddress = headers.getFirst("X-Forwarded-For");
        if (ipAddress == null || ipAddress.length() == 0 || "unknown".equalsIgnoreCase(ipAddress)) {
            ipAddress = headers.getFirst("Proxy-Client-IP");
        }
        if (ipAddress == null || ipAddress.length() == 0 || "unknown".equalsIgnoreCase(ipAddress)) {
            ipAddress = headers.getFirst("WL-Proxy-Client-IP");
        }
        if (ipAddress == null || ipAddress.length() == 0 || "unknown".equalsIgnoreCase(ipAddress)) {
            ipAddress = request.getRemoteAddress().getAddress().getHostAddress();
            if (ipAddress.equals("127.0.0.1") || ipAddress.equals("0:0:0:0:0:0:0:1")) {
                // 根据网卡取本机配置的IP
                try {
                    InetAddress inet = InetAddress.getLocalHost();
                    ipAddress = inet.getHostAddress();
                } catch (UnknownHostException e) {
                    log.error("根据网卡获取本机配置的IP异常", e);
                }

            }
        }

        // 对于通过多个代理的情况，第一个IP为客户端真实IP，多个IP按照','分割
        if (ipAddress != null && ipAddress.indexOf(",") > 0) {
            ipAddress = ipAddress.split(",")[0];
        }

        return ipAddress;
    }
}
```


然后创建一个全局的过滤器，代码如下：

```java
@Slf4j
@Component
public class IpLimitFilter implements GlobalFilter, Ordered {


    public static final String WARNING_MSG = "请求过于频繁，请稍后再试";

    public static final Map<String, Bucket> LOCAL_CACHE = new ConcurrentHashMap<>();

    // 令牌桶初始容量
    public static final long capacity = 2;

    // 补充桶的时间间隔，即2秒补充一次
    public static final long seconds = 2;

    // 每次补充token的个数
    public static final long refillTokens = 1;



    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {

        ServerHttpRequest request = exchange.getRequest();
        ServerHttpResponse response = exchange.getResponse();

        String ip = IpUtil.getIpAddr(request);

        log.info("访问IP为:{}", ip);

        Bucket bucket = LOCAL_CACHE.computeIfAbsent(ip, k -> createNewBucket(ip));

        log.info("IP:{} ,令牌通可用的Token数量:{} ", ip, bucket.getAvailableTokens());

        if (bucket.tryConsume(1)) {
            // 直接放行
            return chain.filter(exchange);
        } else {

            byte[] bits = WARNING_MSG.getBytes(StandardCharsets.UTF_8);
            DataBuffer buffer = response.bufferFactory().wrap(bits);
            //指定编码，否则在浏览器中会中文乱码
            response.getHeaders().add("Content-Type", "text/plain;charset=UTF-8");
            return response.writeWith(Mono.just(buffer));
        }

    }

    @Override
    public int getOrder() {
        return 1;
    }


    private Bucket createNewBucket(String ip) {
        Duration refillDuration = Duration.ofSeconds(seconds);
        Refill refill = Refill.of(refillTokens, refillDuration);
        Bandwidth limit = Bandwidth.classic(capacity, refill);
        return Bucket4j.builder().addLimit(limit).build();
    }
}
```

启动每个服务，在浏览器访问[http://localhost/hello?name=xiaoming&token=123](http://localhost/hello?name=xiaoming&token=123)，请求频繁，返回如下：

```
请求过于频繁，请稍后再试
```

控制台信息如下：

<div style="width:780px;height:300px;margin:50px auto;">
    <img alt="console.png" src="/images/console.png" width="780" height="300"/>
</div>

说明IP限流此时生效了

过一段时间再次访问，返回信息如下：

```
hello, xiaoming, 我是服务提供者:端口为：8080
```

本文代码git地址 [https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)