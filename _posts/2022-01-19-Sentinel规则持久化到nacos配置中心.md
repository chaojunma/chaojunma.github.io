---
layout: post
title: "Sentinel规则持久化到nacos配置中心"
date: 2022-01-19
categories: 微服务

tags: Sentinel SpringBoot Nacos
--- 



**问题描述**

Sentinel Dashboard中添加的规则是存储在内存中的，只要项目一重启规则就丢失了

此处将规则持久化到nacos中，在nacos中添加规则，然后同步到dashboard中；

**具体实现**

1、在pom.xml中添加如下依赖:

```xml
<parent>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-parent</artifactId>
     <version>2.1.13.RELEASE</version>
</parent>

<dependencies>
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    </dependency>
    <!--sentinel配置持久化到nacos所需依赖-->
    <dependency>
        <groupId>com.alibaba.csp</groupId>
        <artifactId>sentinel-datasource-nacos</artifactId>
        <version>1.7.0</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2.2.1.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

2、在nacos配置中心添加限流和降级规则

- 2.1限流规则

```json
[
      {
          "resource": "/sentinel/testA",
          "limitApp": "default",
          "grade": 1,
          "count": 1,
          "strategy": 0,
          "controlBehavior": 0,
          "clusterMode": false
      }
]
```

```
注：

配置格式:选择 json 选项

配置内容:

resource: 资源名称

limitApp: 来源应用

grade: 阈值类型,0表示线程,1表示QPS

count: 单机阈值

strategy: 流控模式,0表示直接,1表示关联,2表示链路

controlBehavior: 流控效果,0表示快速失败,1表示Warm Up,2表示排队等待

clusterMode: 是否集群
```


<div style="width:780px;height:450px;margin:50px auto;">
    <img alt="sentinel-flow-config.png" src="/images/sentinel-flow-config.png" width="780" height="450"/>
</div>

- 2.2降级规则

```json
[
	{
          "resource": "/sentinel/testA",
          "grade": 0,
          "count": 1,
          "timeWindow": 2
      }
]
```

```
resource 资源名，即规则的作用对象

grade 熔断策略，支持慢调用比例/异常比例/异常数策略 慢调用比例

count 慢调用比例模式下为慢调用临界 RT（超出该值计为慢调用）；异常比例/异常数模式下为对应的阈值

timeWindow 熔断时长，单位为 s

minRequestAmount 熔断触发的最小请求数，请求数小于该值时即使异常比率超出阈值也不会熔断（1.7.0 引入） 5

statIntervalMs 统计时长（单位为 ms），如 60*1000 代表分钟级（1.8.0 引入） 1000 ms

slowRatioThreshold 慢调用比例阈值，仅慢调用比例模式有效（1.8.0 引入）
```

<div style="width:780px;height:450px;margin:50px auto;">
      <img alt="sentinel-degrade-config.png" src="/images/sentinel-degrade-config.png" width="780" height="450"/>
</div>

3、在yml配置文件中添加如下依赖:

```yaml
server:
  port: 8081

spring:
  application:
    name: sentinel-demo
  cloud:
    sentinel:
      transport:
        dashboard: 127.0.0.1:8080
        port: 8719
      datasource:
        # sentinel 熔断降级配置
        # ds1为数据源的名称，可以随便写
        ds1:
          nacos:
            server-addr: http://127.0.0.1:8848
            data-id: sentinel-degrade-config
            group-id: DEFAULT_GROUP
            data-type: json
            rule-type: degrade
        # sentinel 限流配置
        ds2:
          nacos:
            server-addr: http://127.0.0.1:8848
            data-id: sentinel-flow-config
            group-id: DEFAULT_GROUP
            data-type: json
            rule-type: flow
```

4、创建测试类如下：

```java
@RestController
@RequestMapping("/sentinel")
public class IndexController {

    @GetMapping("/testA")
    public String testA(){
        return "testA";
    }
}
```

启动服务，我们在浏览器快速访问`http://localhost:8081/sentinel/testA`, 会看到返回`Blocked by Sentinel (flow limiting)`信息，说明我们配置在nacos配置中心的限流规则生效了