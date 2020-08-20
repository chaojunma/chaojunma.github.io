---
layout: post
title: "logstash安装及简单实用"
date: 2020-8-20
categories: 后端 运维
tags: logstash
--- 

### Logstash安装

1、下载Logstash

Logstash. 国内直接从官网 [https://www.elastic.co](https://www.elastic.co) 下载比较困难，需要一些技术手段。这里提供一个国内的镜像下载地址列表 [https://mirrors.huaweicloud.com/logstash/](https://mirrors.huaweicloud.com/logstash/)，方便网友下载。

本人是从本地上传到服务器上的，使用的是5.4.2

2、解压Logstash

进入上传目录解压Logstash

```
tar -zxvf logstash-5.4.2.tar.gz
```

3、测试Logstash

进入logstash-5.4.2目录

```
cd /usr/local/logstash-5.4.2
```

4、启动Logstash并将信息简单输出到控制台

```
bin/logstash -e 'input { stdin { } } output { stdout {} }'
```

启动 Logstash 后，再键入 Hello World，结果如下：

<div style="width:780px;height:230px;margin:50px auto;">
    <img alt="logstash.png" src="/images/logstash.png" width="780" height="230"/>
</div>

### Logstash与SpringBoot项目整合

1、在项目pom.xml中添加如下依赖：

```xml
 <dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>5.3</version>
</dependency>
```

2、在resources目录下创建logback-spring.xml文件，内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/base.xml" />

    <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>192.168.192.10:9600</destination>
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

3、创建logstash配置文件

在 Logstach_HOME 目录中创建一个配置文件，名为 logstash.conf（名字任意）

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
    # 标准输出
    stdout { codec => rubydebug }
}
```

4、启动logstash

```
bin/logstash -f logstash.conf
```

5、在项目接口中添加log日志

```java
 @GetMapping("test")
    public void test(){
        logger.info("测试初始一些日志吧！");
}
```

6、在浏览器中调用test接口，logstash控制台显示如下：

<div style="width:780px;height:240px;margin:50px auto;">
    <img alt="logstash-log.png" src="/images/logstash-log.png" width="780" height="240"/>
</div>

