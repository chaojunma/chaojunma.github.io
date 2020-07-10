---
layout: post
title: "服务的注册于发现"
date: 2020-7-10
categories: 微服务 分布式
tags: SpringCloud Eureka
--- 

Spring Cloud Eureka是Spring Cloud Netflix项目下的服务治理(服务注册于发现)模块，如下图：

<div style="width:580px;height:464px;margin:50px 10px;">
    <img alt="eureka.png" src="/images/eureka.png" width="580" height="464"/>
</div>

**创建Eureka注册中心**

首先创建一个父Maven工程，在其pom.xml文件引入以下相关依赖

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.4.RELEASE</version>
    <relativePath/>
</parent>
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <java.version>1.8</java.version>
    <spring-cloud.version>Finchley.SR1</spring-cloud.version>
    <lombok.version>1.18.0</lombok.version>
    <skipTests>true</skipTests>
</properties>
<dependencies>
    <!-- 集成lombok 框架 -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>${lombok.version}</version>
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
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

然后创建一个model工程，名称为eureka-server,创建完后的工程，其pom.xml继承了父pom文件，并引入spring-cloud-starter-netflix-eureka-server的依赖,如下：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
```


接下来需要创建一个启动类，并添加@EnableEurekaServer注解，表示本服务是EurekaServer服务，并且开启此功能，代码如下：

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run( EurekaServerApplication.class);
    }
}
```

最后需要在resources目录下创建一个appication.yml配置文件，配置如下：

```yaml
server:
  port: 8761

spring:
  application:
    name: eurka-server
```

**注册中心配置**

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
    register-with-eureka: false #不会将自身服务注册到eureka
    fetch-registry: false
```

启动工程,打开浏览器访问：[http://localhost:8761](http://localhost:8761) ,界面如下：

<div style="width:780px;height:384px;margin:50px auto">
    <img alt="eureka1.png" src="/images/eureka1.png" width="780" height="384"/>
</div>
