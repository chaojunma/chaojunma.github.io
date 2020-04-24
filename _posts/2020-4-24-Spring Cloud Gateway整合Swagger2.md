---
layout: post
title: "Spring Cloud Gateway整合Swagger2"
date: 2020-4-24 
categories: 微服务
tags: SpringCloud WebFlux Swagger2 
--- 

本文主要展示一下如何使用Spring Cloud Gateway支持Swagger2

### maven配置
 
在pom.xml文件中添加如下依赖：

```
<properties>
    <swagger.version>3.0.0-SNAPSHOT</swagger.version>
</properties>


<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>${swagger.version}</version>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.plugin</groupId>
            <artifactId>spring-plugin-core</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-spring-webflux</artifactId>
    <version>${swagger.version}</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>${swagger.version}</version>
</dependency>
```

swagger.version目前是3.0.0-SNAPSHOT，因而没有发布到maven官方仓库里头，需要从jcenter-snapshots中拉取

```
<repositories>
    <repository>
        <id>jcenter-snapshots</id>
        <name>jcenter</name>
        <url>http://oss.jfrog.org/artifactory/oss-snapshot-local/</url>
    </repository>
</repositories>
```

### swagger配置

```
@Configuration
@EnableSwagger2WebFlux
public class SwaggerConfig {

    @Value("${swagger.enabled:false}")
    private boolean enabled;

    @Bean
    public Docket api() {
        return new Docket(DocumentationType.SWAGGER_2)
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.gouuse.datahub.gateway.controller"))
                .paths(PathSelectors.any())
                .build()
                .enable(enabled);
    }
}
```

由于支持了WebFlux，所以之前的@EnableSwagger2就移除掉了，变为@EnableSwagger2WebMvc以及@EnableSwagger2WebFlux，这里使用的是@EnableSwagger2WebFlux

### application.yml配置

```
#swagger开关
swagger:
  enabled: true
```
