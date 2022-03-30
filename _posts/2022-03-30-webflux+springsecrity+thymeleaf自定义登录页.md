---
layout: post
title: "webflux+springsecrity+thymeleaf自定义登录页"
date: 2022-03-30
categories: 微服务

tags:  WebFlux springsecrity thymeleaf
--- 



最近在做网关相关的东西，Spring Cloud Gateway是用的WebFlux框架，和WebMvc框架有很大的区别，具体有什么区别大家可以自行百度。 网关侧有一些需要验证的路径，自己又不想写登录接口、验证等等，所以就整合了springsecrity 做简单的验证。springsecrity 本身自带登录页面，但是不太符合系统的风格，于是就想把默认的登录页面替换调。本来以为就是替换个页面的事情，以为很简单（如果是WebMvc框架确实也很简单），但是在过程中遇到了很多问题，特此记录一下。



**添加依赖**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
<dependency>
    <groupId>org.thymeleaf.extras</groupId>
    <artifactId>thymeleaf-extras-springsecurity5</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```



**application.yml配置文件**

```yaml
spring:
  thymeleaf:
    prefix: classpath:/templates/
    suffix: .html
  security:
    user:
      name: admin
      password: 123456
      roles: SUPERUSER
```

然后在resources目录下创建templates目录，用于存放页面，然后创建一个登陆页面login.html, 如下：

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.w3.org/1999/xhtml">
<head>
    <link rel="icon" type="image/x-icon" href="favicon.ico" />
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    <meta charset="UTF-8">
    <title>登录页面</title>
    <style>
        body {
            background-color: #F5F6F8;
        }

        .loginF {
            margin: 150px auto 0;
            width: 400px;
            height: 420px;
            background-color: #fff;
            border-radius: 5px;
            box-shadow: 10px 10px 20px 10px #E1E2E4
        }

        tr {
            color: #C6C6C6;
            font-weight: lighter;
            width: 400px;
            display: block;
            height: 40px;
            line-height: 40px;
        }


    </style>
</head>
<body>

<div style="">
    <form action="/login" method="post" class="loginF">
        <table style="display:block;margin: 0 auto;">
            <tr style="margin-top: 50px;">
                <td  style="text-align: center;color: #4381e6; font-size: 30px;width: 400px">
                    账号登录
                </td>
            </tr>
            <tr style="margin-top: 50px">
                <td  style="padding-left: 60px">
                    <input name="username" type="text" placeholder="用户名：" style="width: 280px;height: 30px;line-height: 30px;border:0px;border-bottom: solid 1px #C6C6C6"/>
                </td>
            </tr>
            <tr style="margin-top: 10px;">
                <td  style="padding-left: 60px">
                    <input name="password" type="password" placeholder="密码" style="width: 280px;height: 30px;line-height: 30px;border:0px;border-bottom: solid 1px #C6C6C6"/>
                </td>
            </tr>
            <tr>
                <td  style="text-align: center;width: 400px;padding-top: 20px;">
                    <span style="color:#FF3B30" th:if="${param.error}" th:text="${session?.SPRING_SECURITY_LAST_EXCEPTION?.message}"></span>
                </td>
            </tr>
            <tr style="margin-top: 30px;">
                <td  style="width: 400px;" align="center">
                    <input value="登录" type="submit" style="display:block;width: 300px;height: 40px;line-height: 40px;border-radius: 10px;color: #fff;background-image: linear-gradient(to left,#4381e6,#11C1E6);border: 0px;"/>
                </td>
            </tr>
        </table>
    </form>
</div>
</body>
</html>
```

**springsecrity配置类**

```java
@Slf4j
@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {


    ServerAuthenticationFailureHandler failureHandler = new FormAuthenticationFailureHandler("/login?error");

    ServerLogoutSuccessHandler logoutSuccessHandler = new FormServerLogoutSuccessHandler("/login?logout");

    @Bean
    public SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {

        http
                .formLogin() // 表单方式
                .loginPage("/login")
                .requiresAuthenticationMatcher(ServerWebExchangeMatchers.pathMatchers(HttpMethod.POST,"/login"))
                .authenticationFailureHandler(failureHandler)
                .and()
                .logout()
                // 注销操作的URL
                .requiresLogout(ServerWebExchangeMatchers.pathMatchers(HttpMethod.GET,"/logout"))
                .logoutSuccessHandler(logoutSuccessHandler)
                .and()
                .authorizeExchange()
                .pathMatchers("/index")
                .authenticated()
                .anyExchange()
                .permitAll()
                .and()
                .csrf()
                .disable()
                .cors();

        return http.build();
    }
}
```

**自定义handler类**

```java
public class FormAuthenticationFailureHandler implements ServerAuthenticationFailureHandler {

    private final URI location;

    private ServerRedirectStrategy redirectStrategy = new DefaultServerRedirectStrategy();

    public FormAuthenticationFailureHandler(String location) {
        Assert.notNull(location, "location cannot be null");
        this.location = URI.create(location);
    }

    @Override
    public Mono<Void> onAuthenticationFailure(WebFilterExchange exchange, AuthenticationException exception) {
        exchange.getExchange().getSession().block().getAttributes().put(WebAttributes.AUTHENTICATION_EXCEPTION, exception);
        return this.redirectStrategy.sendRedirect(exchange.getExchange(), this.location);
    }
}
```



```java
public class FormServerLogoutSuccessHandler implements ServerLogoutSuccessHandler {

    public static final String DEFAULT_LOGOUT_SUCCESS_URL = "/login?logout";

    private URI logoutSuccessUrl = URI.create(DEFAULT_LOGOUT_SUCCESS_URL);

    private ServerRedirectStrategy redirectStrategy = new DefaultServerRedirectStrategy();


    public FormServerLogoutSuccessHandler(String logoutSuccessUrl) {
        Assert.notNull(logoutSuccessUrl, "logoutSuccessUrl cannot be null");
        this.logoutSuccessUrl = URI.create(logoutSuccessUrl);
    }

    @Override
    public Mono<Void> onLogoutSuccess(WebFilterExchange exchange, Authentication authentication) {
        exchange.getExchange().getSession().block().invalidate();
        return this.redirectStrategy
                .sendRedirect(exchange.getExchange(), this.logoutSuccessUrl);
    }
}
```

**自定义跳转登录页和注销接口**



```java
@Controller
public class IndexController {

    @GetMapping("/index")
    public String index() {
        return "index";
    }

    @GetMapping("/login")
    public String toLogin() {
        return "login";
    }

    @GetMapping("/logout")
    public String logout() {
        return "login";
    }
}
```



启动服务，访问服务地址+/index， 会跳转到我们自定义的登录页面，输入错误的账号或密码，如下：

<div style="width:780px;height:400px;margin:50px auto;">
    <img alt="springsecrity-login.png" src="/images/springsecrity-login.png" width="780" height="400"/>
</div>

此时，替换登录页面完成，并且也可以展示错误信息！