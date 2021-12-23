---
layout: post
title: "Spring高级之注解@PropertySource详解"
date: 2020-12-1
categories: 分布式
tags: SpringBoot
--- 

`@PropertySource`注解用于指定资源文件读取的位置，它不仅能读取properties文件，也能读取xml文件，并且通过YAML解析器，配合自定义PropertySourceFactory实现解析yaml文件



### 读取properties文件

在resources资源目录下存在datasource-config.properties，要加载此文件中的配置，需要用到`@PropertySource`注解，具体如下：

***datasource-config.properties文件***

```properties
druid.driverClassName=com.mysql.jdbc.Driver
druid.url=jdbc:mysql://127.0.0.1/db1?useUnicode=true&amp;characterEncoding=UTF-8
druid.username=root
druid.password=123456
```

***加载配置类***

```java
@Slf4j
@Setter
@Getter
@Configuration
@PropertySource(value = "classpath:datasource-config.properties")
@ConfigurationProperties(prefix = "druid")
public class SpringConfig {

    @Value("${druid.driverClassName}")
    private String driverClassName;

    @Value("${druid.url}")
    private String url;

    @Value("${druid.username}")
    private String username;

    @Value("${druid.password}")
    private String password;


    @Bean
    public void druidDataSource(){
        log.info("driverClassName:[{}], url:[{}], username:[{}], password:[{}]", driverClassName, url, username, password);
    }
}
```

### 读取xml文件

在resources资源目录下存在datasource-config.xml，要加载此文件中的配置，需要用到`@PropertySource`注解，具体如下：

***加载配置类***
```java
@Slf4j
@Setter
@Getter
@Configuration
@PropertySource(value = "classpath:datasource-config.xml")
@ConfigurationProperties(prefix = "druid")
public class SpringConfig {

    @Value("${druid.driverClassName}")
    private String driverClassName;

    @Value("${druid.url}")
    private String url;

    @Value("${druid.username}")
    private String username;

    @Value("${druid.password}")
    private String password;


    @Bean
    public void druidDataSource(){
        log.info("driverClassName:[{}], url:[{}], username:[{}], password:[{}]", driverClassName, url, username, password);
    }
}
```

***datasource-config.xml***

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
    <entry key="druid.driverClassName">com.mysql.jdbc.Driver</entry>
    <entry key="druid.url">jdbc:mysql://127.0.0.1/db1?useUnicode=true&amp;characterEncoding=UTF-8</entry>
    <entry key="druid.username">root</entry>
    <entry key="druid.password">5201314..a</entry>
</properties>
```

### 读取yaml文件

如果想通过`@PropertySorce`注解加载yaml文件,需要配合自定义PropertySourceFactory实现。


***添加依赖***

```xml
<dependency>
    <groupId>org.yaml</groupId>
    <artifactId>snakeyaml</artifactId>
    <version>1.23</version>
</dependency>
```


***自定义PropertySourceFactory***

```java
import org.springframework.beans.factory.config.YamlPropertiesFactoryBean;
import org.springframework.core.env.PropertiesPropertySource;
import org.springframework.core.env.PropertySource;
import org.springframework.core.io.support.EncodedResource;
import org.springframework.core.io.support.PropertySourceFactory;
import java.io.IOException;
import java.util.Properties;

public class YAMLPropertySourceFactory implements PropertySourceFactory {

    @Override
    public PropertySource<?> createPropertySource(String name, EncodedResource encodedResource) throws IOException {
        //创建一个YAML解析工厂。
        YamlPropertiesFactoryBean factory = new YamlPropertiesFactoryBean();
        //设置资源。
        factory.setResources(encodedResource.getResource());

        //获取解析后的Properties对象
        Properties properties = factory.getObject();
        //返回。此时不能像默认工厂那样返回ResourcePropertySource对象 ，要返回他的父类PropertiesPropertySource对象。
        return name != null ? new PropertiesPropertySource(name, properties) :
                new PropertiesPropertySource(encodedResource.getResource().getFilename(),properties);
    }
}
```

***datasource-config.yaml***

```yaml
druid:
  driverClassName: com.mysql.jdbc.Driver
  url: jdbc:mysql://127.0.0.1/db1?useUnicode=true&amp;characterEncoding=UTF-8
  username: root
  password: 123456
```


***加载配置类***
```java
@Slf4j
@Setter
@Getter
@Configuration
@PropertySource(value = "classpath:datasource-config.yml", factory = YAMLPropertySourceFactory.class)
@ConfigurationProperties(prefix = "druid")
public class SpringConfig {

    @Value("${druid.driverClassName}")
    private String driverClassName;

    @Value("${druid.url}")
    private String url;

    @Value("${druid.username}")
    private String username;

    @Value("${druid.password}")
    private String password;


    @Bean
    public void druidDataSource(){
        log.info("driverClassName:[{}], url:[{}], username:[{}], password:[{}]", driverClassName, url, username, password);
    }
}
```