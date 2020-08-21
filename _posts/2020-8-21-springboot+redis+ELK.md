---
layout: post
title: "SpringBoot+Redis+ELK"
date: 2020-8-21
categories: 后端 运维
tags: logstash SpringBoot Redis
--- 

### Logstash配置

1、首先在 Logstach_HOME 目录中创建一个配置文件，名为 logstash.conf（名字任意）

在logstash.conf配置文件中添加如下配置：

```json
input {
  redis {
    host => "127.0.0.1"
    port => 6379
    type => "redis-input"
    data_type => "list"
    key => "logstash:redis"
    threads => 5
    codec => "json"
  }
}

output {
    # 标准输出
    stdout { codec => rubydebug }
}
```

2、启动logstash

```
bin/logstash -f logstash.conf
```

### 整合SpringBoot项目

1、创建springboot项目，并在pom.xml中添加如下依赖：

```xml
<dependency>
    <groupId>com.cwbase</groupId>
    <artifactId>logback-redis-appender</artifactId>
    <version>1.1.5</version>
</dependency>
```

2、在resources目录下创建logback.xml文件，配置如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="Console" class="ch.qos.logback.core.ConsoleAppender">
        <Target>System.out</Target>
        <encoder>
            <pattern>[%d{HH:mm:ss}][%t][%p][%c]-%m%n</pattern>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>info</level>
        </filter>
    </appender>
    <appender name="LOGSTASH" class="com.cwbase.logback.RedisAppender">
        <source>admin-api</source>
        <type>dev</type>
        <host>172.22.150.39</host>
        <key>logstash:redis</key>
        <tags>dev</tags>
        <mdc>true</mdc>
        <location>true</location>
        <callerStackIndex>0</callerStackIndex>
    </appender>

    <root level="info">
        <appender-ref ref="Console"/>
        <appender-ref ref="LOGSTASH"/>
    </root>
</configuration>
```

3、在项目接口中添加log日志

```java
 @GetMapping("test")
    public void test(){
        logger.info("测试初始一些日志吧！");
}
```

4、在浏览器中调用test接口，logstash控制台显示如下：

<div style="width:780px;height:250px;margin:50px auto;">
    <img alt="logstash-redis.png" src="/images/logstash-redis.png" width="780" height="250"/>
</div>


以上收集的日志为所有日志，如何实现收集自定义日志呢？改造如下：

在pom.xml中添加如下依赖:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.60</version>
</dependency>
```

在application.yml中添加redis相关配置如下：

```yaml
spring:
  redis:
    url: redis://172.22.150.39:6379
```

接口改造如下：

```java
String logstashKey = "logstash:redis";

@Autowired
private StringRedisTemplate redisTemplate;

 @GetMapping("test")
    public void test(){
        Map<String, String> map = new HashMap();
        map.put("level", "INFO");
        map.put("message", "测试初始一些日志吧！");
        redisTemplate.opsForList().rightPush(logstashKey, JSONObject.toJSONString(map));
}
```

删除resources目录下logback.xml文件,重启服务，在浏览器中调用test接口，logstash控制台显示如下：

<div style="width:780px;height:140px;margin:50px auto;">
    <img alt="logstash-info.png" src="/images/logstash-info.png" width="780" height="140"/>
</div>
