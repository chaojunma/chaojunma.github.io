---
layout: post
title: "MybatisPlus配置双数据库驱动"
date: 2022-01-19
categories: 微服务 数据库

tags: Mybatis Mysql Postgresql
--- 



最近项目中需要用到2种数据库驱动连接数据库，下面我们基于MybatisPlus实现一下

**具体实现**

1、在pom.xml中添加如下依赖:

```xml
<properties>
    <java.version>1.8</java.version>
    <lombok.version>1.18.2</lombok.version>
    <mybatis-plus.version>3.2.0</mybatis-plus.version>
    <druid.version>1.1.9</druid.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- mysql-->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <scope>runtime</scope>
    </dependency>
    <!-- postgrepsql-->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-boot-starter</artifactId>
        <version>${mybatis-plus.version}</version>
    </dependency>
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus</artifactId>
        <version>${mybatis-plus.version}</version>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>${lombok.version}</version>
    </dependency>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid-spring-boot-starter</artifactId>
        <version>${druid.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
</dependencies>
```

2、在yml配置文件中添加如下配置:

```yaml
server:
  port: 8080

spring:
  application:
    name: xxxx
  datasource:
    druid:
      # mysql数据源配置
      db1:
        driver-class-name: com.mysql.jdbc.Driver
        url: jdbc:mysql://127.0.0.1:3306/db1?useUnicode=true&characterEncoding=UTF-8&useSSL=false
        username: ${username}
        password: ${password}
        initial-size: 5
        min-idle: 5
        max-active: 50
      # postgresql 数据源配置
      db2:
        driver-class-name: org.postgresql.Driver
        url: jdbc:postgresql://127.0.0.1:5432/db2?useUnicode=true&characterEncoding=utf-8
        username: ${username}
        password: ${password}
        initial-size: 5
        min-idle: 5
        max-active: 50

# mybatis-plus配置
mybatis-plus:
  type-aliases-package: com.dms.gateway.api.entity
  mapper-locations: classpath:/mapper/*Mapper.xml
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

3、新建DataSourceEnum枚举类，如下：

```java
public enum DataSourceEnum {

    DB1("db1"),
    DB2("db2");

    private String value;

    DataSourceEnum(String value){this.value=value;}

    public String getValue() {
        return value;
    }
}
```



4、新建DataSourceContextHolder类，如下：

```java
public class DataSourceContextHolder {

    // 默认数据源
    public static final String DEFAULT_DS = DataSourceEnum.DB1.getValue();

    private static final ThreadLocal<String> contextHolder = new InheritableThreadLocal<>();

    /**
     *  设置数据源
     * @param db
     */
    public static void setDataSource(String db){
        contextHolder.set(db);
    }

    /**
     * 取得当前数据源
     * @return
     */
    public static String getDataSource(){
        return contextHolder.get();
    }

    /**
     * 清除上下文数据
     */
    public static void clear(){
        contextHolder.remove();
    }
}
```

5、新建MultipleDataSource类，如下：

```java
public class MultipleDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return DataSourceContextHolder.getDataSource();
    }
}
```

5、新建DataSource注解，如下：

```java
@Target({ElementType.METHOD,ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface DataSource {

    DataSourceEnum value() default DataSourceEnum.DB1;
}
```

6、新建面向类和方法级别的切面，如下：

```java
@Component
@Slf4j
@Aspect
@Order(-6)
public class DataSourceClassAspect {


    @Before("@within(dataSource)")
    public void doBefore(JoinPoint point, DataSource dataSource){
        log.info("切换到数据源[{}]", dataSource.value().getValue());
        DataSourceContextHolder.setDataSource(dataSource.value().getValue());
    }

    @After("@within(dataSource)")
    public void doAfter(JoinPoint point, DataSource dataSource){
        log.info("回收数据源[{}]", dataSource.value().getValue());
        DataSourceContextHolder.clear();
    }
}
```

```java
@Component
@Slf4j
@Aspect
@Order(-5)
public class DataSourceMethodAspect {


    @Before("@annotation(dataSource)")
    public void doBefore(JoinPoint point, DataSource dataSource){
        log.info("切换到数据源[{}]", dataSource.value().getValue());
        DataSourceContextHolder.setDataSource(dataSource.value().getValue());
    }

    @After("@annotation(dataSource)")
    public void doAfter(JoinPoint point, DataSource dataSource){
        log.info("回收数据源[{}]", dataSource.value().getValue());
        DataSourceContextHolder.clear();
    }
}
```

7、新建多数据源配置类，如下：

```java
@Configuration
@MapperScan("com.dms.gateway.api.mapper")
public class MyBatiesPlusConfig {

    /*
     * 分页插件，自动识别数据库类型
     * 多租户，请参考官网【插件扩展】
     */
    @Bean
    public PaginationInterceptor paginationInterceptor() {
        PaginationInterceptor paginationInterceptor = new PaginationInterceptor();
        return paginationInterceptor;
    }

    @Bean(name = "db1")
    @ConfigurationProperties(prefix = "spring.datasource.druid.db1" )
    public DataSource db1() {
        return DruidDataSourceBuilder.create().build();
    }

    @Bean(name = "db2")
    @ConfigurationProperties(prefix = "spring.datasource.druid.db2" )
    public DataSource db2() {
        return DruidDataSourceBuilder.create().build();
    }

    /**
     * 动态数据源配置
     * @return
     */
    @Bean
    @Primary
    public DataSource multipleDataSource(@Qualifier("db1") DataSource db1,
                                         @Qualifier("db2") DataSource db2) {
        MultipleDataSource multipleDataSource = new MultipleDataSource();
        Map< Object, Object > targetDataSources = new HashMap<>();
        targetDataSources.put(DataSourceEnum.DB1.getValue(), db1);
        targetDataSources.put(DataSourceEnum.DB2.getValue(), db2);
        //添加数据源
        multipleDataSource.setTargetDataSources(targetDataSources);
        //设置默认数据源
        multipleDataSource.setDefaultTargetDataSource(db1);
        return multipleDataSource;
    }

    @Bean("sqlSessionFactory")
    public SqlSessionFactory sqlSessionFactory() throws Exception {
        MybatisSqlSessionFactoryBean sqlSessionFactory = new MybatisSqlSessionFactoryBean();
        sqlSessionFactory.setDataSource(multipleDataSource(db1(),db2()));

        MybatisConfiguration configuration = new MybatisConfiguration();
        configuration.setJdbcTypeForNull(JdbcType.NULL);
        configuration.setMapUnderscoreToCamelCase(true);
        configuration.setCacheEnabled(false);
        sqlSessionFactory.setConfiguration(configuration);
        //添加分页功能
        sqlSessionFactory.setPlugins(paginationInterceptor());
        return sqlSessionFactory.getObject();
    }
}
```



接下来需要具体的业务逻辑，在service层的类或者方法上面添加`@DataSource`注解来指定该业务需要用到的数据源，如下：

```java
@Service
@DataSource(DataSourceEnum.DB2)
public class GatewayLogService {

    @Autowired
    private GatewayLogMapper mapper;

    @Override
    public Result<IPage<GatewayLog>> pageList(GatewayLogDTO gatewayLogDTO) {
        LambdaQueryWrapper<GatewayLog> wrapper = Wrappers.lambdaQuery();
        wrapper.like(StringUtils.isNotEmpty(gatewayLogDTO.getPath()), GatewayLog::getPath, gatewayLogDTO.getPath());
        wrapper.like(StringUtils.isNotEmpty(gatewayLogDTO.getSourceServer()), GatewayLog::getSourceServer, gatewayLogDTO.getSourceServer());
        wrapper.like(StringUtils.isNotEmpty(gatewayLogDTO.getTargetServer()), GatewayLog::getTargetServer, gatewayLogDTO.getTargetServer());
        wrapper.eq(StringUtils.isNotEmpty(gatewayLogDTO.getMethod()), GatewayLog::getMethod, gatewayLogDTO.getMethod());
        wrapper.like(StringUtils.isNotEmpty(gatewayLogDTO.getRequestBody()), GatewayLog::getRequestBody, gatewayLogDTO.getRequestBody());
        wrapper.like(StringUtils.isNotEmpty(gatewayLogDTO.getResponse()), GatewayLog::getResponse, gatewayLogDTO.getResponse());
        wrapper.orderByDesc(GatewayLog::getId);

        IPage<GatewayLog> page = new Page<>(gatewayLogDTO.getPageNo(), gatewayLogDTO.getPageSize());

        return Result.success(mapper.selectPage(page, wrapper));
    }

}
```

启动服务，调用相关的接口，我们会在控制台看到如下信息：

<div style="width:708px;height:108px;margin:50px auto;">
    <img alt="db2.png" src="/images/db2.png" width="708" height="108"/>
</div>

说明数据源切换成功！！！