---
layout: post
title: "Spring Cloud Gateway全局过滤器 GlobalFilter"
date: 2020-7-11
categories: 微服务 分布式
tags: SpringCloud Gateway
--- 

全局过滤器作用于所有的路由，不需要单独配置，我们可以用它来实现很多统一化处理的业务需求，比如权限认证、IP 限流等等。

单独定义只需要实现 GlobalFilter、Ordered 两个接口就可以了，具体代码如下所示。

```java
@Component
public class MyFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        ServerHttpResponse response = exchange.getResponse();

        String token = request.getQueryParams().getFirst("token");
        if(StringUtils.isEmpty(token)) {
            String msg = "token is empty";
            byte[] bits = msg.getBytes(StandardCharsets.UTF_8);
            DataBuffer buffer = response.bufferFactory().wrap(bits);
            //指定编码，否则在浏览器中会中文乱码
            response.getHeaders().add("Content-Type", "text/plain;charset=UTF-8");
            return response.writeWith(Mono.just(buffer));
        }

        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return 0;
    }
}

```

启动每个服务，在浏览器访问[http://localhost/hello?name=xiaoming](http://localhost/hello?name=xiaoming)，返回如下：

```
token is empty
```

此时说明全局过滤器生效了，请求被拦截

在浏览器访问[http://localhost/hello?name=xiaoming&token=123](http://localhost/hello?name=xiaoming&token=123)，返回如下：

```
hello, xiaoming, 我是服务提供者:端口为：8080
```

此时拦截验证通过，转发到目标服务


本文代码git地址 [https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)