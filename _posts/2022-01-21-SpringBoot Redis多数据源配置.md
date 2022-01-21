---
layout: post
title: "SpringBoot Redis多数据源配置"
date: 2022-01-21
categories: 微服务

tags:  SpringBoot Redis
--- 



此处提供了一个SpringBoot starter插件

<div style="margin:30px 0px;">
   gitee地址 <a href="https://gitee.com/xmingtx/spring-boot-starter-dynamic-redis">https://gitee.com/xmingtx/spring-boot-starter-dynamic-redis</a>
</div>

**客户端集成**

1. 在pom.xml中添加如下依赖:

```xml
<dependency>
    <groupId>com.mk</groupId>
    <artifactId>spring-boot-starter-dynamic-redis</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

2.在yml配置文件中添加如下配置:

```yaml
dms:
  dynamic:
    redis:
      enabled: true
      connection:
        # 第一个Redis连接
        demo1Redis:
          host: 127.0.0.1
          port: 6379
          database: 1
          timeout: 2000
          jedis:
            pool:
              max-wait: 3000
              max-active: 100
              max-idle: 20
              min-idle: 0
              timeout: 3000
        # 第二个Redis连接
        demo2Redis:
          host: 127.0.0.1
          port: 6379
          database: 2
          timeout: 2000
          jedis:
            pool:
              max-wait: 3000
              max-active: 100
              max-idle: 20
              min-idle: 0
              timeout: 3000
```
3.在启动类上面添加`@EnableDynamicRedis`注解，如下:

```java
@EnableDynamicRedis
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class);
    }
}
```

4.创建Redis配置类，如下:

```java
@Configuration
public class RedisConfig {

    @Resource
    private DynamicRedisProvider dynamicRedisProvider;

    @Bean(name = "demo1Redis")
    public RedisTemplate demo1Redis() {
         return new StringRedisTemplate(dynamicRedisProvider.loadRedis().get("demo1Redis"));
    }

    @Bean(name = "demo2Redis")
    public RedisTemplate demo2Redis() {
        return new StringRedisTemplate(dynamicRedisProvider.loadRedis().get("demo2Redis"));
    }
}
```

5.测试

```java
@RestController
public class TestController {
    @Resource(name = "demo1Redis")
    private StringRedisTemplate demo1Template;

    @Resource(name = "demo2Redis")
    private StringRedisTemplate demo2Template;

    @GetMapping("/testRedis")
    public String testRedis(){
        demo1Template.opsForValue().set("testKey", "我存放到Redis下db为1的库");
        demo2Template.opsForValue().set("testKey", "我存放到Redis下db为2的库");
        return "success";
    }
}
```

6、在浏览器访问/testRedis接口，通过Redis客户端查看数据如下：

<div style="width:780px;height:450px;margin:50px auto;">
    <img alt="redis-starter1.png" src="/images/redis-starter1.png" width="780" height="450"/>
</div>

<div style="width:780px;height:450px;margin:50px auto;">
    <img alt="redis-starter2.png" src="/images/redis-starter2.png" width="780" height="450"/>
</div>