---
layout: post
title: "Zuul网关过滤器"
date: 2020-7-10
categories: 微服务 分布式
tags: SpringCloud Zuul
--- 

**过滤器**

zuul不仅只是路由，并且还能过滤，做一些安全验证。继续改造工程；

创建MyFilter过滤器,如下：

```java
@Slf4j
@Component
public class MyFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        log.info("method:{}, url:{}", request.getMethod(), request.getRequestURL().toString());
        Object accessToken = request.getParameter("token");
        if(accessToken == null) {
            log.warn("token is empty");
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
            try {
                ctx.getResponse().getWriter().write("token is empty");
            }catch (Exception e){}

            return null;
        }
        log.info("ok");
        return null;
    }
}
```

filterType：返回一个字符串代表过滤器的类型，在zuul中定义了四种不同生命周期的过滤器类型，具体如下：
- pre：路由之前
- routing：路由之时
- post： 路由之后
- error：发送错误调用
- filterOrder：过滤的顺序
- shouldFilter：这里可以写逻辑判断，是否要过滤，本文true,永远过滤。
- run：过滤器的具体逻辑。可用很复杂，包括查sql，nosql去判断该请求到底有没有权限访问。


这时访问：[http://localhost:8769/api-a/hello?name=xiaoming](http://localhost:8769/api-a/hello?name=xiaoming) ；网页显示：

```
token is empty
```

访问[http://localhost:8769/api-a/hello?name=xiaoming&token=123456](http://localhost:8769/api-a/hello?name=xiaoming&token=123456) ；网页显示：

```
hello, xiaoming, 我是服务提供者:端口为：8080
```


本文代码git地址 [https://gitee.com/xmingtx/springcloud-lesson.git](https://gitee.com/xmingtx/springcloud-lesson.git)