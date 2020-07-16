---
layout: post
title: "Spring Cloud Gateway 实现Token校验"
date: 2020-7-16
categories: 微服务 分布式
tags: SpringCloud Gateway JWT
--- 


在我看来，在某些场景下，网关就像是一个公共方法，把项目中的都要用到的一些功能提出来，抽象成一个服务。比如，我们可以在业务网关上做日志收集、Token校验等等，当然这么理解很狭隘，因为网关的能力远不止如此，但是不妨碍我们更好地理解它。下面的例子演示了，如何在网关校验Token，并提取用户信息放到Header中传给下游业务系统。

## 1. 生成Token

用户登录成功以后，生成token，此后的所有请求都带着token。网关负责校验token，并将用户信息放入请求Header，以便下游系统可以方便的获取用户信息。

<div style="width:500px;height:300px;margin:50px auto;">
    <img alt="gateway-jwt.png" src="/images/gateway-jwt.png" width="500" height="300"/>
</div>


为了方便演示，本例中涉及三个工程

公共项目：commons-jwt

认证服务：auth-service

网关服务：gateway-service

### 1.1. Token生成与校验工具类

因为生成token在认证服务中，token校验在网关服务中，因此，我把这一部分写在了公共项目commons-jwt中

**pom.xml**

```xml
<dependencies>
    <dependency>
        <groupId>com.auth0</groupId>
        <artifactId>java-jwt</artifactId>
        <version>3.10.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-lang3</artifactId>
        <version>3.9</version>
    </dependency>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.66</version>
    </dependency>
</dependencies>
```

**JWTUtil.java**

```java
public class JWTUtil {


    private static final String USER_INFO_KEY = "username";

    /**
     * 生成Token
     * @param issuser    签发者
     * @param username   用户标识(唯一)
     * @param secretKey  签名算法以及密匙
     * @param tokenExpireTime 过期时间
     * @return
     */
    public static String generateToken(String issuser, String username, String secretKey, long tokenExpireTime) {
        Algorithm algorithm = Algorithm.HMAC256(secretKey);

        Date now = new Date();

        Date expireTime = new Date(now.getTime() + tokenExpireTime);

        String token = JWT.create()
                .withIssuer(issuser)
                .withIssuedAt(now)
                .withExpiresAt(expireTime)
                .withClaim(USER_INFO_KEY, username)
                .sign(algorithm);

        return token;
    }

    /**
     * 校验Token
     * @param issuser   签发者
     * @param token     访问秘钥
     * @param secretKey 签名算法以及密匙
     * @return
     */
    public static void verifyToken(String issuser, String token, String secretKey) {
        try {
            Algorithm algorithm = Algorithm.HMAC256(secretKey);
            JWTVerifier jwtVerifier = JWT.require(algorithm).withIssuer(issuser).build();
            jwtVerifier.verify(token);
        } catch (JWTDecodeException jwtDecodeException) {
            throw new TokenAuthenticationException(ResponseCodeEnum.TOKEN_INVALID.getCode(), ResponseCodeEnum.TOKEN_INVALID.getMessage());
        } catch (SignatureVerificationException signatureVerificationException) {
            throw new TokenAuthenticationException(ResponseCodeEnum.TOKEN_SIGNATURE_INVALID.getCode(), ResponseCodeEnum.TOKEN_SIGNATURE_INVALID.getMessage());
        } catch (TokenExpiredException tokenExpiredException) {
            throw new TokenAuthenticationException(ResponseCodeEnum.TOKEN_EXPIRED.getCode(), ResponseCodeEnum.TOKEN_INVALID.getMessage());
        } catch (Exception ex) {
            throw new TokenAuthenticationException(ResponseCodeEnum.UNKNOWN_ERROR.getCode(), ResponseCodeEnum.UNKNOWN_ERROR.getMessage());
        }
    }

    /**
     * 从Token中提取用户信息
     * @param token
     * @return
     */
    public static String getUserInfo(String token) {
        DecodedJWT decodedJWT = JWT.decode(token);
        String username = decodedJWT.getClaim(USER_INFO_KEY).asString();
        return username;
    }


}

```

**ResponseCodeEnum.java**

```java
public enum ResponseCodeEnum {

    SUCCESS(0, "成功"),
    FAIL(-1, "失败"),
    LOGIN_ERROR(1000, "用户名或密码错误"),
    UNKNOWN_ERROR(2000, "未知错误"),
    PARAMETER_ILLEGAL(2001, "参数不合法"),
    TOKEN_INVALID(2002, "无效的Token"),
    TOKEN_SIGNATURE_INVALID(2003, "无效的签名"),
    TOKEN_EXPIRED(2004, "token已过期"),
    TOKEN_MISSION(2005, "token缺失"),
    REFRESH_TOKEN_INVALID(2006, "刷新Token无效");

    private int code;

    private String message;

    ResponseCodeEnum(int code, String message) {
        this.code = code;
        this.message = message;
    }

    public int getCode() {
        return code;
    }

    public String getMessage() {
        return message;
    }
}

```

**ResponseResult.java**

```java
@Builder
@Data
public class ResponseResult {

    private int code = 0;

    private String msg;

    private Object data;
}
```

**MD5Util.java**

```java
@Slf4j
public class MD5Util {
    public static String getMD5Str(String str) {
        byte[] digest = null;
        try {
            MessageDigest md5 = MessageDigest.getInstance("md5");
            digest  = md5.digest(str.getBytes("utf-8"));
        } catch (NoSuchAlgorithmException e) {
            log.error(e.getMessage(), e);
        } catch (UnsupportedEncodingException e) {
            log.error(e.getMessage(), e);
        }
        //16是表示转换为16进制数
        String md5Str = new BigInteger(1, digest).toString(16);
        return md5Str;
    }
}
```

**TokenAuthenticationException**

```java
public class TokenAuthenticationException extends RuntimeException  {

    private int code;

    private String message;

    public TokenAuthenticationException(int code, String message) {
        super();
        this.code = code;
        this.message = message;
    }
}
```

### 1.2. 生成token

这一部分在auth-service中

**pom.xml**

```xml
<dependencies>
     <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter-web</artifactId>
     </dependency>
     <dependency>
         <groupId>org.apache.commons</groupId>
         <artifactId>commons-lang3</artifactId>
         <version>3.9</version>
     </dependency>
     <dependency>
         <groupId>commons-codec</groupId>
         <artifactId>commons-codec</artifactId>
         <version>1.14</version>
     </dependency>
     <dependency>
         <groupId>org.apache.commons</groupId>
         <artifactId>commons-pool2</artifactId>
         <version>2.8.0</version>
     </dependency>
     <dependency>
         <groupId>com.mk</groupId>
         <artifactId>commons-jwt</artifactId>
         <version>1.0-SNAPSHOT</version>
     </dependency>
    <!-- mybatis-plus依赖 -->
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-boot-starter</artifactId>
        <version>3.1.1</version>
    </dependency>
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus</artifactId>
        <version>3.1.1</version>
    </dependency>
    <!-- mysql组件 -->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>
 </dependencies>
```

**application.yml**

```yaml
server:
  port: 8080

spring:
  application:
    name: auth-service
  redis:
    url: redis://10.32.15.78:6379
  datasource:
    url: jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=UTF-8&useSSL=false
    username: root
    password: 123456
    druid:
      # 验证连接是否有效。此参数必须设置为非空字符串，下面三项设置成true才能生效
      validation-query: SELECT 1
      # 连接是否被空闲连接回收器(如果有)进行检验. 如果检测失败, 则连接将被从池中去除
      test-while-idle: true
      # 是否在从池中取出连接前进行检验, 如果检验失败, 则从池中去除连接并尝试取出另一个
      test-on-borrow: true
      # 是否在归还到池中前进行检验
      test-on-return: false
      # 连接在池中最小生存的时间，单位是毫秒
      min-evictable-idle-time-millis: 30000

#mybatis配置
mybatis-plus:
  type-aliases-package: com.mk.auth.entity
  mapper-locations: classpath:mapper/**/*.xml
  global-config:
    db-config:
      id-type: auto
      field-strategy: not_empty
      logic-delete-value: 1
      logic-not-delete-value: 0
  configuration:
    map-underscore-to-camel-case: true
    cache-enabled: false
    call-setters-on-nulls: true
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

**OauthController.java**

```java
@RestController
@RequestMapping("/oauth")
public class OauthController {


    @Value("${secretKey:!@#$%^&*}")
    private String secretKey;

    @Value("${issuser:mark}")
    private String issuser;

    @Value("${tokenExpireTime:30}")
    private long tokenExpireTime;

    @Autowired
    private RedisTemplate redisTemplate;

    @Autowired
    private UserService userService;

    private static final String TOKEN_CACHE_PREFIX = "auth-service:";


    /**
     * 获取token
     * @param request
     * @param bindingResult
     * @return
     */
    @PostMapping("/getToken")
    public ResponseResult getToken(@RequestBody @Validated TokenRequest request, BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            return ResponseResult
                    .builder()
                    .code(ResponseCodeEnum.PARAMETER_ILLEGAL.getCode())
                    .msg(ResponseCodeEnum.PARAMETER_ILLEGAL.getMessage())
                    .build();
        }

        LambdaQueryWrapper<User> wrapper = new QueryWrapper<User>().lambda();
        wrapper.eq(User::getUsername, request.getUsername());
        wrapper.eq(User::getPassword, request.getPassword());
        User user = userService.getOne(wrapper);

        if (user != null) {

            //  生成Token
            String token = JWTUtil.generateToken(issuser, user.getUsername(), secretKey, tokenExpireTime * 1000);

            //  生成刷新Token
            String refreshToken = UUID.randomUUID().toString().replace("-", "");

            //  放入缓存
            String key = MD5Util.getMD5Str(user.getUsername());
            HashOperations<String, String, String> hashOperations = redisTemplate.opsForHash();
            hashOperations.put(TOKEN_CACHE_PREFIX + key, "token", token);
            hashOperations.put(TOKEN_CACHE_PREFIX + key, "refreshToken", refreshToken);
            redisTemplate.expire(TOKEN_CACHE_PREFIX + key, tokenExpireTime, TimeUnit.SECONDS);

            TokenResult data = TokenResult.builder()
                    .access_token(token)
                    .refresh_token(refreshToken)
                    .username(user.getUsername())
                    .expires_in(tokenExpireTime)
                    .build();

            return ResponseResult
                    .builder()
                    .code(ResponseCodeEnum.SUCCESS.getCode())
                    .msg(ResponseCodeEnum.SUCCESS.getMessage())
                    .data(data)
                    .build();
        }

        return ResponseResult
                .builder()
                .code(ResponseCodeEnum.LOGIN_ERROR.getCode())
                .msg(ResponseCodeEnum.LOGIN_ERROR.getMessage())
                .build();
    }

    /**
     * 删除token
     * @param username
     * @return
     */
    @GetMapping("/delToken")
    public ResponseResult deltoken(@RequestParam("username") String username) {
        HashOperations<String, String, String> hashOperations = redisTemplate.opsForHash();
        String key = MD5Util.getMD5Str(username);
        hashOperations.delete(TOKEN_CACHE_PREFIX + key);

        return ResponseResult
                .builder()
                .code(ResponseCodeEnum.SUCCESS.getCode())
                .msg(ResponseCodeEnum.SUCCESS.getMessage())
                .build();
    }

    /**
     * 刷新Token
     */
    @PostMapping("/refreshToken")
    public ResponseResult refreshToken(@RequestBody @Validated RefreshRequest request, BindingResult bindingResult) {
        String refreshToken = request.getRefreshToken();
        HashOperations<String, String, String> hashOperations = redisTemplate.opsForHash();
        String key = MD5Util.getMD5Str(request.getUsername());
        String originalRefreshToken = hashOperations.get(TOKEN_CACHE_PREFIX + key, "refreshToken");
        if (StringUtils.isBlank(originalRefreshToken) || !originalRefreshToken.equals(refreshToken)) {
            return ResponseResult
                    .builder()
                    .code(ResponseCodeEnum.REFRESH_TOKEN_INVALID.getCode())
                    .msg(ResponseCodeEnum.REFRESH_TOKEN_INVALID.getMessage())
                    .build();
        }

        //  生成新token
        String newToken = JWTUtil.generateToken(issuser, key, secretKey, tokenExpireTime * 1000);
        hashOperations.put(TOKEN_CACHE_PREFIX + key, "token", newToken);
        redisTemplate.expire(TOKEN_CACHE_PREFIX + key, tokenExpireTime, TimeUnit.SECONDS);

        return ResponseResult
                .builder()
                .code(ResponseCodeEnum.SUCCESS.getCode())
                .msg(ResponseCodeEnum.SUCCESS.getMessage())
                .data(newToken)
                .build();
    }
}

```

## 2. 校验Token

GatewayFilter和GlobalFilter都可以，这里用GlobalFilter

**pom.xml**

```xml
<dependencies>
    <dependency>
         <groupId>org.springframework.cloud</groupId>
         <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <dependency>
        <groupId>com.mk</groupId>
        <artifactId>commons-jwt</artifactId>
        <version>1.0-SNAPSHOT</version>
    </dependency>
</dependencies>
```

**application.yml**

```yaml
server:
  port: 80

spring:
  application:
    name: gateway-service
  redis:
    url: redis://10.32.15.78:6379
```

**AuthorizeFilter.java**

```java
@Component
public class AuthorizeFilter implements GlobalFilter, Ordered {

    @Value("${secretKey:!@#$%^&*}")
    private String secretKey;

    @Value("${issuser:mark}")
    private String issuser;

    @Autowired
    private RedisTemplate redisTemplate;

    private static final String TOKEN_CACHE_PREFIX = "auth-service:";


    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {

        ServerHttpRequest serverHttpRequest = exchange.getRequest();
        ServerHttpResponse serverHttpResponse = exchange.getResponse();

        String token = serverHttpRequest.getHeaders().getFirst("token");

        if (StringUtils.isBlank(token)) {
            serverHttpResponse.setStatusCode(HttpStatus.UNAUTHORIZED);
            return getVoidMono(serverHttpResponse, ResponseCodeEnum.TOKEN_MISSION);
        }

        // 检查是否可以解密
        String username = null;
        try {
            username = JWTUtil.getUserInfo(token);
        } catch (JWTDecodeException e) {
            return getVoidMono(serverHttpResponse, ResponseCodeEnum.TOKEN_INVALID);
        }

        if (StringUtils.isEmpty(username)) {
            return getVoidMono(serverHttpResponse, ResponseCodeEnum.TOKEN_INVALID);
        }

        // 检查Redis中是否有此Token
        String key = MD5Util.getMD5Str(username);
        if (!redisTemplate.opsForHash().hasKey(TOKEN_CACHE_PREFIX + key, "token")) {
            return getVoidMono(serverHttpResponse, ResponseCodeEnum.TOKEN_INVALID);
        }

        try {
            JWTUtil.verifyToken(issuser, token, secretKey);
        } catch (TokenAuthenticationException ex) {
            return getVoidMono(serverHttpResponse, ResponseCodeEnum.TOKEN_INVALID);
        } catch (Exception ex) {
            return getVoidMono(serverHttpResponse, ResponseCodeEnum.UNKNOWN_ERROR);
        }

        return chain.filter(exchange);
    }


    private Mono<Void> getVoidMono(ServerHttpResponse serverHttpResponse, ResponseCodeEnum responseCodeEnum) {
        serverHttpResponse.getHeaders().add("Content-Type", "application/json;charset=UTF-8");
        ResponseResult responseResult = ResponseResult.builder()
                .code(responseCodeEnum.getCode())
                .msg(responseCodeEnum.getMessage())
                .build();
        DataBuffer dataBuffer = serverHttpResponse.bufferFactory().wrap(JSON.toJSONString(responseResult).getBytes());
        return serverHttpResponse.writeWith(Flux.just(dataBuffer));
    }


    @Override
    public int getOrder() {
        return 0;
    }
}

```

### 网关配置

**GatewayConfig.java**

```java
@Configuration
public class GatewayConfig {
    @Bean
    public RouteLocator myRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
                .route(p -> p
                        .path("/baidu")
                        .uri("http://www.baidu.com"))
                .build();
    }
}
```

本文代码git地址 [https://github.com/chaojunma/springcloud-auth.git](https://github.com/chaojunma/springcloud-auth.git)