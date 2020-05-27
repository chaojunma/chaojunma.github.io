---
layout: post
title: "SpringBoot基于RateLimiter+AOP动态的为不同接口限流"
date: 2020-5-22 
categories: 后端
tags: SpringBoot AOP 限流 
--- 


RateLimiter是guava提供的基于令牌桶算法的实现类，可以非常简单的完成限流特技，并且根据系统的实际情况来调整生成token的速率。以下为基于SpringBoot+AOP实现对不同接口的限流。

导入相关依赖包


```xml
<!--引入SpringBoot的Web模块-->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!--引入AOP依赖-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.0</version>
</dependency>
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>20.0</version>
</dependency>
```

自定义注解

```java
@Documented
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
	
	String value() default "";

	/**
	* 默认每秒放入桶中的token
	* @return
	*/
	double limit() default Double.MAX_VALUE;

	/**
	* 获取令牌的等待时间 默认1
	* @return
	*/
	int timeOut() default 1;

	/**
	* 超时时间单位 默认：秒
	* @return
	*/
	TimeUnit timeOutUnit() default TimeUnit.SECONDS;

}
```


封装定义返回结果

```java
@Data
@Builder
public class Result {

	// 状态码
	private Integer code; //

	// 描述
	private String message;

	// 返回数据
	private Object data;

}
```

AOP实现

```java
@Slf4j
@Aspect
@Component
public class RateLimitAspect {

	private RateLimiter rateLimiter;
	
	@Autowired
	public HttpServletResponse response;
	
	// 用来存放不同接口的RateLimiter(key为接口名称，value为RateLimiter)
	private ConcurrentHashMap<String, RateLimiter> map = new ConcurrentHashMap<>();
	
	// 限流提示语
	public static final String RATE_LIMIT_WARNING = "系统繁忙！";


	@Pointcut("@annotation(com.datahub.annotation.RateLimit)")
	public void pointcut() {
	}

	@Around("pointcut()")
	public Object around(ProceedingJoinPoint joinPoint) throws NoSuchMethodException {
		Object obj = null;
		// 获取目标方法
		Signature signature = joinPoint.getSignature();
		MethodSignature msig = (MethodSignature) signature;
		Method targetMethod = msig.getMethod();
		
		// 获取注解信息
		RateLimit rateLimit = targetMethod.getAnnotation(RateLimit.class);
		// 获取注解每秒加入桶中的token
		double limitNum = rateLimit.limit();
		String functionName = msig.getName(); // 注解所在方法名区分不同的限流策略
		
		// 获取rateLimiter
		if (map.containsKey(functionName)) {
			rateLimiter = map.get(functionName);
		} else {
			map.put(functionName, RateLimiter.create(limitNum));
			rateLimiter = map.get(functionName);
		}

		try {
			// 尝试消费
			if (rateLimiter.tryAcquire()) {
				// 执行方法
				obj = joinPoint.proceed();
			} else {
				writeLimitResult(response);
			}
		} catch (Throwable e) {
			log.error(e.getMessage(), e);
		}
		
		
		return obj;
	}

	
	/**
	 * 限流响应结果
	 * @param response
	 */
	public void writeLimitResult(HttpServletResponse response) {
		Result result = Result.builder().code(500).message(RATE_LIMIT_WARNING).build();
		response.setContentType("application/json;charset=UTF-8");
		try {
			response.getWriter().write(JSON.toJSONString(result));
		} catch (IOException e) {
			log.error(e.getMessage(), e);
		}
	}

}
```

Controller控制器

```java
@RestController
public class RateLimitController {

	@RateLimit(limit = 1)
	@GetMapping("/test/limit")
	public Result testLimit() {
		return Result.builder().code(200).message("success").build();
	}
	
}
```

启动SpringBoot项目，在浏览器中请求`http://localhost:8080/test/limit`接口

正常返回如下：

```json
{"code":200,"message":"success","data":null}
```

操作频繁返回如下：

```json
{"code":500,"message":"系统繁忙！"}
```

`此时说明限流起到作用了!!!`

