---
layout: post
title: "Logstash json filter插件"
date: 2020-9-1
categories: 后端 运维
tags: SpringBoot logstash Kibana Elasticsearch
--- 


通常情况，Logstash收集到的数据都会转成json格式，但是默认logstash只是对收集到的格式化数据转成json，如果收到的数据仅仅是一个字符串是不会转换成Json.

例如：
```
{
   "message":{\"age\":22,"\id\":1,\"username\":\"张三\"} 
}
```

message字段的内容是一个json字符串，不是格式化的Json格式，如果数据导入到Elasticsearch，message字段也是一个字符串，不是一个Json对象；json filter插件可以解决这种问题。下面为具体实现：


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

filter {
      json {
           source => "message"
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

    <!--从application.yml配置中加载spring.application.name属性-->
    <springProperty scope="context" name="springAppName" source="spring.application.name"/>

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
                        "message": "%message"
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
    User user = User.builder()
            .id(1)
            .username("张三")
            .age(22)
            .build();
    log.info(JSONObject.toJSONString(user));
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

接下来在浏览器调用test接口

然后进入发现页，选择刚刚的索引，如下所示:

<div style="width:780px;height:450px;margin:50px auto;">
    <img alt="logstash-json.png" src="/images/logstash-json.png" width="780" height="450"/>
</div>

此时说明日志信息已经存储到Elasticsearch当中，并且可以用过Kibaba实现展示与查询

### Logstash Filter其他配置

Logstash Filter另外还支持字段白名单、数据处理等等，如下：

```json
filter {
      # 使用json方式分解message字段
      json {
           source => "message"
      }

      # 过滤获取包含request_uri的字段
      prune {
           whitelist_names => [ "request_uri" ]
      }

      # 替换字符串中的某值为空
      mutate {
            gsub => [ "request_uri", "/plugs.html\?", "" ]
      }

      # 引入某字段,使用自动切分完成数组，并且移除元数据
      kv  {
            source => "request_uri"
            field_split => "&"
            #value_split => "="
            remove_field => [ "request_uri" ]
      }
}
```