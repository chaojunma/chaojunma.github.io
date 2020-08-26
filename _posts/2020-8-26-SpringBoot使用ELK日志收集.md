---
layout: post
title: "SpringBoot使用ELK日志收集"
date: 2020-8-26
categories: 后端 运维
tags: SpringBoot ELK Elasticsearch logstash Kibana
--- 

### Logstash配置

1、首先在 Logstach_HOME 目录中创建一个配置文件，名为 logstash.conf（名字任意）

在logstash.conf配置文件中添加如下配置：

```json
input {
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 9600
    codec => json_lines
  }
}

output {
    elasticsearch {
        hosts => "192.168.192.11:9200"
        index => "syslog-%{+YYYY.MM.dd}"
    }
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
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>5.3</version>
</dependency>
```

2、在resources目录下创建logback.xml文件，配置如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/base.xml" />

    <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>192.168.192.11:9600</destination>
        <!-- 日志输出编码 -->
        <encoder charset="UTF-8"
                 class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp>
                    <timeZone>UTC</timeZone>
                </timestamp>
                <pattern>
                    <pattern>
                        {
                        "logLevel": "%level",
                        "serviceName": "${springAppName:-}",
                        "pid": "${PID:-}",
                        "thread": "%thread",
                        "class": "%logger{40}",
                        "rest": "%message"
                        }
                    </pattern>
                </pattern>
            </providers>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="LOGSTASH" />
        <appender-ref ref="CONSOLE" />
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

4、测试

分别启动Elasticsearch、Kibana、Logstash

打开kibana管理页面，添加刚刚创建的索引，如图所示：

<div style="width:780px;height:450px;margin:50px auto;">
    <img alt="kibana-index.png" src="/images/kibana-index.png" width="780" height="450"/>
</div>

<div style="width:780px;height:450px;margin:50px auto;">
    <img alt="kibana-index2.png" src="/images/kibana-index2.png" width="780" height="450"/>
</div>

接下来在浏览器多次调用`test`接口

然后进入发现页，选择刚刚的索引，如下所示:

<div style="width:780px;height:450px;margin:50px auto;">
    <img alt="kibana-find.png" src="/images/find.png" width="780" height="450"/>
</div>

此时说明日志信息已经存储到Elasticsearch当中，并且可以用过Kibaba实现展示与查询