---
layout: post
title: "自定义Spring Cloud Gateway过滤器工厂"
date: 2020-7-11
categories: 微服务 分布式
tags: SpringCloud Gateway
--- 

自定义 Spring Cloud Gateway 过滤器工厂需要继承 AbstractGatewayFilterFactory 类，重写 apply 方法的逻辑。命名需要以 GatewayFilterFactory 结尾，比如 AccessAuthGatewayFilterFactory，那么在使用的时候 AccessAuth就是这个过滤器工厂的名称。

自定义过滤器工厂代码如下所示。

```java
@Slf4j
@Component
public class AccessAuthGatewayFilterFactory extends AbstractGatewayFilterFactory<AccessAuthGatewayFilterFactory.Config> {

    public static final String WARNING_MSG = "访问未授权";

    public AccessAuthGatewayFilterFactory(){
        super(AccessAuthGatewayFilterFactory.Config.class);
        log.info("Loaded GatewayFilterFactory [AccessAuth]");
    }

    @Override
	public List<String> shortcutFieldOrder() {
	return Arrays.asList("name");
    } 

    @Override
    public GatewayFilter apply(Config config) {

        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();
            ServerHttpResponse response = exchange.getResponse();
            if(config.name.equals("zhangsan")) {
                return chain.filter(exchange.mutate().request(request).build());
            } else {

                byte[] bits = WARNING_MSG.getBytes(StandardCharsets.UTF_8);
                DataBuffer buffer = response.bufferFactory().wrap(bits);
                //指定编码，否则在浏览器中会中文乱码
                response.getHeaders().add("Content-Type", "text/plain;charset=UTF-8");
                return response.writeWith(Mono.just(buffer));
            }
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

修改service-gateway服务application.yml配置文件的路由配置，如下：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: hello
        predicates:   #断言
        - Path=/hello
        uri: http://localhost:8080  #目标服务地址
        filters:
        - AccessAuth=lisi
```

启动服务，在浏览器访问在浏览器访问[http://localhost/hello?name=xiaoming](http://localhost/hello?name=xiaoming),返回如下：

```
访问未授权
```

若将service-gateway服务application.yml配置文件的路由配置修改，如下：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: hello
        predicates:   #断言
        - Path=/hello
        uri: http://localhost:8080  #目标服务地址
        filters:
        - AccessAuth=zhangsan
```

重新启动服务，从浏览器再次访问[http://localhost/hello?name=xiaoming](http://localhost/hello?name=xiaoming)，返回如下：

```
hello, xiaoming, 我是服务提供者:端口为：8080
```


本文代码git地址 [https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)