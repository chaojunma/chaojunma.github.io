---
layout: post
title: "服务网关 Spring Cloud Gateway 熔断、限流、重试"
date: 2020-8-12
categories: 微服务 分布式
tags: SpringCloud Gateway Hyxtrix 限流
--- 



微服务系统中熔断限流环节，对保护系统的稳定性起到了很大的作用，作为网关，Spring Cloud Gateway也提供了很好的支持。先来理解下熔断限流概念：

- 熔断降级：在分布式系统中，网关作为流量的入口，大量请求进入网关，向后端远程系统或服务发起调用，后端服务不可避免的会产生调用失败（超时或者异常），失败时不能让请求堆积在网关上，需要快速失败并返回回去，这就需要在网关上做熔断、降级操作。
- 限流：网关上有大量请求，对指定服务进行限流，可以很大程度上提高服务的可用性与稳定性，限流的目的是通过对并发访问/请求进行限速，或对一个时间窗口内的请求进行限速来保护系统。一旦达到限制速率则可以拒绝服务、排队或等待、降级。

### 熔断

pom.xml依赖如下：

```xml
<properties>
	<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
	<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
	<java.version>1.8</java.version>
	<spring-cloud.version>Finchley.SR1</spring-cloud.version>
</properties>


<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>2.0.4.RELEASE</version>
</parent>

<dependencies>
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-gateway</artifactId>
	</dependency>
    <!-- 集成histrix熔断器 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-dependencies</artifactId>
			<version>${spring-cloud.version}</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>
```

application.yml配置如下：

```yaml
server: 
  port: 8888
  
spring:
  application:
    name: datahub-gateway
  cloud:
    config:
      name: ${spring.application.name}
    gateway:
      default-filters:
      - name: Hystrix
        args:
          name: fallbackcmd
          fallbackUri: forward:/defaultfallback
      routes:
      - id: service1_v1
        uri: http://172.22.144.131
        predicates:
        - Path=/test
      discovery:
        locator:
#          开启通过serviceId转发到具体的服务实例
          enabled: true
#          开启可以通过小写的serviceId进行基于服务路由转发
          lower-case-service-id: true
          
          
#histrix超时时间
hystrix:
  command:
    fallbackcmd:
      execution:
        isolation:
          strategy: SEMAPHORE
          semaphore:
            maxConcurrentRequests: 2000
          thread:
            timeoutInMilliseconds: 3000
    default:
      fallback:
        isolation:
          semaphore:
            maxConcurrentRequests: 2000
```

hystrix.command.fallbackcmd.execution.isolation.strategy 设置HystrixCommand.run()的隔离策略，有两种选项：

> THREAD —— 在固定大小线程池中，以单独线程执行，并发请求数受限于线程池大小。
> SEMAPHORE —— 在调用线程中执行，通过信号量来限制并发量。

hystrix.command.fallbackcmd.execution.isolation.semaphore.maxConcurrentRequests 设置最大请求数量， 默认的Hystrix会限制某个请求的最大并发量：默认10，如果超过了这个默认的并发值且开启了fallback，则丢弃剩下的请求直接进入fallback方法

<div style="width:300px;height:200px;margin:50px auto;">
    <img alt="gateway-hystrix.png" src="/images/gateway-hystrix.png" width="300" height="200"/>
</div>


构建defaultfallback处理器,如下：

```java
@RestController
public class HystrixController {

	@RequestMapping("/defaultfallback")
    public Map<String,Object> defaultfallback(){
        Map<String,Object> map = new HashMap<>();
        map.put("Code",5001);
        map.put("Message","服务异常");
        return map;
    }
    
}
```

先不构建下游服务，直接运行网关，返回结果如下：

```java
{
  "Code": 5001,
  "Message": "服务异常"
}
```

### 限流

限速在高并发场景中比较常用的手段之一，可以有效的保障服务的整体稳定性，Spring Cloud Gateway 提供了基于 Redis 的限流方案。所以我们首先需要添加对应的依赖包`spring-boot-starter-data-redis-reactive`

```xml
 <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

配置文件中需要添加 Redis 地址和限流的相关配置

```yaml
spring:
  application:
    name: datahub-gateway
  redis:
    url:  redis://10.32.15.78:6379
  cloud:
    gateway:
      default-filters:
      - name: Hystrix
        args:
          name: fallbackcmd
          fallbackUri: forward:/defaultfallback
      routes:
        - id: service_customer
          #下游服务地址
          uri: http://127.0.0.1:8083/
          order: 0
          #网关断言匹配
          predicates:
            - Path=/gateway/**
          filters:
            - StripPrefix=1
            #限流过滤器
            - name: RequestRateLimiter
              args:
                key-resolver: '#{@userKeyResolver}'
                # 每秒最大访问次数（放令牌桶的速率）
                redis-rate-limiter.replenishRate: 10
                # 令牌桶最大容量（令牌桶的大小）
                redis-rate-limiter.burstCapacity: 10
```

- filter 名称必须是 RequestRateLimiter
- redis-rate-limiter.replenishRate：允许用户每秒处理多少个请求
- redis-rate-limiter.burstCapacity：令牌桶的容量，允许在一秒钟内完成的最大请求数
- key-resolver：使用 SpEL 按名称引用 bean

项目中设置限流的策略，创建 Config 类,如下：

```java
public class Config {

    @Bean
    KeyResolver userKeyResolver() {
        return exchange -> Mono.just(exchange.getRequest().getQueryParams().getFirst("user"));
    }
}
```

根据请求参数中的 user 字段来限流，也可以设置根据请求 IP 地址来限流，设置如下:

```java
@Bean
public KeyResolver ipKeyResolver() {
    return exchange -> Mono.just(exchange.getRequest().getRemoteAddress().getHostName());
}
```

这样网关就可以根据不同策略来对请求进行限流了。

### 路由重试

RetryGatewayFilter 是 Spring Cloud Gateway 对请求重试提供的一个 GatewayFilter Factory

配置示例：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: retry_test
        uri: lb://spring-cloud-producer
        predicates:
        - Path=/retry
        filters:
        - name: Retry
          args:
            retries: 3
            statuses: BAD_GATEWAY
```

Retry GatewayFilter 通过这四个参数来控制重试机制： retries, statuses, methods, 和 series。

- retries：重试次数，默认值是 3 次
- statuses：HTTP 的状态返回码，取值请参考：org.springframework.http.HttpStatus
- methods：指定哪些方法的请求需要进行重试逻辑，默认值是 GET 方法，取值参考：org.springframework.http.HttpMethod
- series：一些列的状态码配置，取值参考：org.springframework.http.HttpStatus.Series。符合的某段状态码才会进行重试逻辑，默认值是 SERVER_ERROR，值是 5，也就是 5XX(5 开头的状态码)，共有5 个值。