---
layout: post
title: "SpringBoot服务使用Nacos配置中心"
date: 2020-4-7 
categories: 微服务
tags: SpringBoot Nacos
--- 



**环境准备**

<div style="margin:30px 0px;">
   下载地址 <a href="https://github.com/alibaba/nacos/releases">https://github.com/alibaba/nacos/releases</a>
</div>

下载完成以后解压，进入到bin目录启动 

Linux下执行sh startup.sh -m standalone命令(standalone代表着单机模式运行，非集群模式)

Windows下双击startup.cmd文件启动

服务启动以后，通过浏览器输入http://localhost:8848/nacos/进行访问，登录名：nacos 密码：nacos

**具体实现**

1、在需要继承的SpringBoot服务中添加如下依赖

````
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    <version>${latest.version}</version>
</dependency>
````

注意：版本 0.2.x.RELEASE 对应的是 Spring Boot 2.x 版本，版本 0.1.x.RELEASE 对应的是 Spring Boot 1.x 版本。我的是spring Boot 2.x 以上的所以选择 0.2.1.RELEASE版本的

2.添加配置bootstrap.yml配置文件，内容如下

````
spring:
    application:
        name: api-i18n
    cloud:
        nacos:
            config:
                server-addr: 127.0.01:8848
                group: DEFAULT_GROUP
                prefix: ${spring.appliction.name} #可以不写，默认为${spring.appliction.name}
                file-extension: yaml
````

3、按照上面的配置规则在Nacos中添加相关配置，如下：

<div style="width:800px;height:480px;margin:50px auto">
    <img alt="nacos-config.png" src="/images/nacos-config.png" width="800" height="480"/>
</div>


启动服务正常，从nacos配置中心读取相关配置

**数据持久化**

1.安装数据库，版本要求：5.6.5+
2.初始化mysql数据库，数据库初始化文件：nacos-mysql.sql
3.修改conf/application.properties文件，增加支持mysql数据源配置(目前只支持mysql)，添加mysql数据源的url、用户名和密码

配置如下：

````
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
db.user=root
db.password=123456
````






