---
layout: post
title: "Spring Cloud Gateway自定义路由断言工厂"
date: 2020-7-11
categories: 微服务 分布式
tags: SpringCloud Gateway
--- 

自定义路由断言工厂需要继承 AbstractRoutePredicateFactory 类，重写 apply 方法的逻辑和shortcutFieldOrder方法。

在 apply 方法中可以通过 exchange.getRequest() 拿到 ServerHttpRequest 对象，从而可以获取到请求的参数、请求方式、请求头等信息。

apply 方法的参数是自定义的配置类，在使用的时候配置参数，在 apply 方法中直接获取使用。

命名需要以 RoutePredicateFactory 结尾，比如 CheckAuthRoutePredicateFactory，那么在使用的时候 CheckAuth 就是这个路由断言工厂的名称。代码如下所示。

```java
@Slf4j
@Component
public class CheckAuthRoutePredicateFactory extends AbstractRoutePredicateFactory<CheckAuthRoutePredicateFactory.Config> {


    public CheckAuthRoutePredicateFactory() {
        super(Config.class);
        log.info("Loaded RoutePredicateFactory [CheckAuth]");
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return Arrays.asList("name");
    }


    @Override
    public Predicate<ServerWebExchange> apply(Config config) {

        return exchange -> {
            if (config.getName().equals("zhangsan")) {
                return true;
            }
            return false;
        };
    }


    public static class Config {

        private String name;

        public void setName(String name) {
            this.name = name;
        }

        public String getName() {
            return name;
        }
    }

}

```

在application.yml中添加路由配置，如下：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: customer_route
        predicates:
        - CheckAuth=zhangsan
        uri: http://51ufo.cn
```



本文代码git地址 [https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)