---
layout: post
title: "logstash的drop过滤器插件"
date: 2020-9-4
categories: 后端 运维
tags: logstash  
--- 

logstash在filter段对日志进行解析的时候, 可以直接筛选出我们想要的日志内容, 如果日志内容里不包括某些字段或者不匹配某种规则, 我们可以把整条日志直接扔掉, 如下配置：

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
      
      #如果message字段不是以'{'开头且以'}'结尾，则丢弃
      if ([message] !~ "^{.*}$") {
         drop {}
      }

      json {
           source => "message"             #将message字段进行解析
           skip_on_invalid_json => true    #无效的json则跳过
           remove_field => ["message"]     #删除message字段
      }


}

output {
    # 标准输出
    stdout { codec => rubydebug }
}

```

补充:

logstash可以使用条件判断来控制filter的执行。官方说明见Accessing Event Data and Fields in the Configuration。支持的运算符包括：

```
相等: ==, !=, <, >, <=, >=
正则: =~(匹配正则), !~(不匹配正则)
包含: in(包含), not in(不包含)
布尔操作: and(与), or(或), nand(非与), xor(非或)
一元运算: !(取反), ()(复合表达式), !()(对复合表达式结果取反)
```