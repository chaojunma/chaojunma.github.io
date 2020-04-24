---
layout: post
title: "SpringCloud Nacos Seata实现分布式事务"
date: 2020-4-3 
categories: 微服务 分布式
tags: Nacos Seata SpringCloudAlibaba
--- 


<div style="margin:30px 0px;">
   本文代码github地址 <a href="https://github.com/chaojunma/springcloud-seata">https://github.com/chaojunma/springcloud-seata</a>
</div>

**应用介绍**

+ springcloud-seata-common(公共包)
+ springcloud-seata-account(用户服务，用于账户支付)
+ springcloud-seata-storage(库存服务，用于库存管理)
+ springcloud-seata-order（订单服务，记录订单状态及调用用户服务和库存服务）


**目标介绍**

本文，我们将通过一个实战案例来具体介绍Seata的使用方式，我们将模拟一个简单的用户购买商品下单场景，创建4个子工程，分别是 springcloud-seata-order（下单服务）、springcloud-seata-storage（库存服务）、springcloud-seata-account（支付服务）和 springcloud-seata-common(公共包)，具体流程图如图：

<div style="width:770px;height:450px;margin:50px 0px">
   <img alt="seata-flow.png" src="/images/seata-flow.png" width="770" height="450"/>
</div>

**环境准备**

***Seata环境安装及配置***

在本次实战中，我们使用Nacos做为服务中心和配置中心，Nacos部署请自行查阅文档，这里不再赘述。
接下来我们需要部署Seata的Server端，下载地址为：https://github.com/seata/seata/releases ，建议选择最新版本下载，本文用到的版本为 v1.1.0 ，下载 seata-server-1.1.0.zip 解压后，打开 conf 文件夹，我们需对其中的一些配置做出修改。

````
registry {
  type = "nacos"

  nacos {
    serverAddr = "192.168.227.1"
    namespace = ""
    cluster = "default"
  }
}

config {
  type = "nacos"

  nacos {
    serverAddr = "192.168.227.1"
    namespace = ""
    group = "SEATA_GROUP"
  }
}

````
这里我们选择使用Nacos作为服务中心和配置中心，这里做出对应的配置，同时可以看到Seata的注册服务支持：file 、nacos 、eureka、redis、zk、consul、etcd3、sofa等方式，配置支持：file、nacos 、apollo、zk、consul、etcd3等方式。

接下来我们修改seata根目录下的config.txt文件如下：

````
transport.type=TCP
transport.server=NIO
transport.heartbeat=true
transport.enableClientBatchSendRequest=false
transport.threadFactory.bossThreadPrefix=NettyBoss
transport.threadFactory.workerThreadPrefix=NettyServerNIOWorker
transport.threadFactory.serverExecutorThreadPrefix=NettyServerBizHandler
transport.threadFactory.shareBossWorker=false
transport.threadFactory.clientSelectorThreadPrefix=NettyClientSelector
transport.threadFactory.clientSelectorThreadSize=1
transport.threadFactory.clientWorkerThreadPrefix=NettyClientWorkerThread
transport.threadFactory.bossThreadSize=1
transport.threadFactory.workerThreadSize=default
transport.shutdown.wait=3
service.vgroup_mapping.fsp_tx_group=default
service.default.grouplist=127.0.0.1:8091
service.enableDegrade=false
service.disableGlobalTransaction=false
client.rm.asyncCommitBufferLimit=10000
client.rm.lockRetryInternal=10
client.rm.lockRetryTimes=30
client.rm.reportRetryCount=5
client.rm.lockRetryPolicyBranchRollbackOnConflict=true
client.rm.tableMetaCheckEnable=false
client.rm.sqlParserType=druid
client.rm.reportSuccessEnable=false
client.tm.commitRetryCount=5
client.tm.rollbackRetryCount=5
store.mode=db
store.file.dir=file_store/data
store.file.maxBranchSessionSize=16384
store.file.maxGlobalSessionSize=512
store.file.fileWriteBufferCacheSize=16384
store.file.flushDiskMode=async
store.file.sessionReloadReadSize=100
store.db.datasource=dbcp
store.db.dbType=mysql
store.db.driverClassName=com.mysql.jdbc.Driver
store.db.url=jdbc:mysql://192.168.227.1:3306/seata-server?useUnicode=true
store.db.user=root
store.db.password=123456
store.db.minConn=1
store.db.maxConn=3
store.db.globalTable=global_table
store.db.branchTable=branch_table
store.db.queryLimit=100
store.db.lockTable=lock_table
server.recovery.committingRetryPeriod=1000
server.recovery.asynCommittingRetryPeriod=1000
server.recovery.rollbackingRetryPeriod=1000
server.recovery.timeoutRetryPeriod=1000
server.maxCommitRetryTimeout=-1
server.maxRollbackRetryTimeout=-1
server.rollbackRetryTimeoutUnlockEnable=false
client.undo.dataValidation=true
client.undo.logSerialization=jackson
server.undo.logSaveDays=7
server.undo.logDeletePeriod=86400000
client.undo.logTable=undo_log
client.log.exceptionRate=100
transport.serialization=seata
transport.compressor=none
metrics.enabled=false
metrics.registryType=compact
metrics.exporterList=prometheus
metrics.exporterPrometheusPort=9898
````
最主要的几个配置如下：
````
service.vgroup_mapping.fsp_tx_group=default #指定事务的分组
store.mode=db #改为数据库存储方式
store.db.url=jdbc:mysql://192.168.227.1:3306/seata-server?useUnicode=true #数据库连接地址
store.db.user=root #数据库用户名
store.db.password=123456 数据库密码
````
因为我们是使用的Nacos作为配置中心，所以这里需要通过Git Bash先执行脚本来初始化Nacos的相关配置，命令如下：
````
cd conf
sh nacos-config.sh 192.168.227.1 #Nacos服务地址
````
执行成功后可以打开Nacos的控制台，在配置列表中，可以看到初始化了很多 Group 为 SEATA_GROUP 的配置，如图：
<div style="width:770px;height:250px;margin:50px 0px">
   <img alt="seata-nacos.png" src="/images/seata-nacos.png" width="770" height="250"/>
</div>

初始化成功后，可以切换到bin目录下双击seata-server.bat（Windows）或执行sh seata-server.sh -p 8091 -m file（Linux下）启动seata-server服务

启动后在 Nacos 的服务列表下面可以看到一个名为 serverAddr 的服务

***数据库初始化***

创建数据库seata-server，初始化脚本如下：
````
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for branch_table
-- ----------------------------
DROP TABLE IF EXISTS `branch_table`;
CREATE TABLE `branch_table`  (
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(128) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `transaction_id` bigint(20) NULL DEFAULT NULL,
  `resource_group_id` varchar(32) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `resource_id` varchar(256) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `branch_type` varchar(8) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `status` tinyint(4) NULL DEFAULT NULL,
  `client_id` varchar(64) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `application_data` varchar(2000) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `gmt_create` datetime(6) NULL DEFAULT NULL,
  `gmt_modified` datetime(6) NULL DEFAULT NULL,
  PRIMARY KEY (`branch_id`) USING BTREE,
  INDEX `idx_xid`(`xid`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Table structure for global_table
-- ----------------------------
DROP TABLE IF EXISTS `global_table`;
CREATE TABLE `global_table`  (
  `xid` varchar(128) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `transaction_id` bigint(20) NULL DEFAULT NULL,
  `status` tinyint(4) NOT NULL,
  `application_id` varchar(32) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `transaction_service_group` varchar(32) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `transaction_name` varchar(128) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `timeout` int(11) NULL DEFAULT NULL,
  `begin_time` bigint(20) NULL DEFAULT NULL,
  `application_data` varchar(2000) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `gmt_create` datetime(0) NULL DEFAULT NULL,
  `gmt_modified` datetime(0) NULL DEFAULT NULL,
  PRIMARY KEY (`xid`) USING BTREE,
  INDEX `idx_gmt_modified_status`(`gmt_modified`, `status`) USING BTREE,
  INDEX `idx_transaction_id`(`transaction_id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Table structure for lock_table
-- ----------------------------
DROP TABLE IF EXISTS `lock_table`;
CREATE TABLE `lock_table`  (
  `row_key` varchar(128) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `xid` varchar(96) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `transaction_id` bigint(20) NULL DEFAULT NULL,
  `branch_id` bigint(20) NOT NULL,
  `resource_id` varchar(256) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `table_name` varchar(32) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `pk` varchar(36) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `gmt_create` datetime(0) NULL DEFAULT NULL,
  `gmt_modified` datetime(0) NULL DEFAULT NULL,
  PRIMARY KEY (`row_key`) USING BTREE,
  INDEX `idx_branch_id`(`branch_id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

SET FOREIGN_KEY_CHECKS = 1;

````

创建数据库seata-account，初始化脚本如下：
````
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for account
-- ----------------------------
DROP TABLE IF EXISTS `account`;
CREATE TABLE `account`  (
  `id` bigint(11) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `user_id` bigint(11) NULL DEFAULT NULL COMMENT '用户id',
  `total` decimal(10, 0) NULL DEFAULT NULL COMMENT '总额度',
  `used` decimal(10, 0) NULL DEFAULT NULL COMMENT '已用余额',
  `residue` decimal(10, 0) NULL DEFAULT 0 COMMENT '剩余可用额度',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 2 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

SET FOREIGN_KEY_CHECKS = 1;
````
创建数据库seata-storage，初始化脚本如下：

````
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for storage
-- ----------------------------
DROP TABLE IF EXISTS `storage`;
CREATE TABLE `storage`  (
  `id` bigint(11) NOT NULL AUTO_INCREMENT,
  `product_id` bigint(11) NULL DEFAULT NULL COMMENT '产品id',
  `total` int(11) NULL DEFAULT NULL COMMENT '总库存',
  `used` int(11) NULL DEFAULT NULL COMMENT '已用库存',
  `residue` int(11) NULL DEFAULT NULL COMMENT '剩余库存',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 2 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

SET FOREIGN_KEY_CHECKS = 1;

````
创建数据库seata-order，初始化脚本如下：
````
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for orders
-- ----------------------------
DROP TABLE IF EXISTS `orders`;
CREATE TABLE `orders`  (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` int(11) NULL DEFAULT NULL COMMENT '用户id',
  `product_id` int(11) NULL DEFAULT NULL COMMENT '产品id',
  `count` int(11) NULL DEFAULT NULL COMMENT '数量',
  `money` int(11) NULL DEFAULT NULL COMMENT '金额',
  `status` int(1) NULL DEFAULT NULL COMMENT '订单状态：0：创建中；1：已完结',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 36 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

SET FOREIGN_KEY_CHECKS = 1;
````
接下来需要在数据库seata-account、seata-storage、seata-order中分别创建一张表，脚本如下：
````
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for undo_log
-- ----------------------------
DROP TABLE IF EXISTS `undo_log`;
CREATE TABLE `undo_log`  (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'increment id',
  `branch_id` bigint(20) NOT NULL COMMENT 'branch transaction id',
  `xid` varchar(100) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT 'global transaction id',
  `context` varchar(128) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT 'undo_log context,such as serialization',
  `rollback_info` longblob NOT NULL COMMENT 'rollback info',
  `log_status` int(11) NOT NULL COMMENT '0:normal status,1:defense status',
  `log_created` datetime(0) NOT NULL COMMENT 'create datetime',
  `log_modified` datetime(0) NOT NULL COMMENT 'modify datetime',
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE INDEX `ux_undo_log`(`xid`, `branch_id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 51 CHARACTER SET = utf8 COLLATE = utf8_general_ci COMMENT = 'AT transaction mode undo table' ROW_FORMAT = Dynamic;

SET FOREIGN_KEY_CHECKS = 1;

````

**代码实战**

由于本示例代码偏多，这里仅介绍核心代码和一些需要注意的代码，其余代码可以访问配套的代码仓库获取。

***父工程 springcloud-seata 依赖 pom.xml 文件如下***
````
<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- Spring Cloud Nacos Service Discovery -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
        <!-- Spring Cloud Nacos Config -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
        </dependency>
        <!-- Spring Cloud Seata -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-seata</artifactId>
            <version>2.1.0.RELEASE</version>
            <exclusions>
                <exclusion>
                    <artifactId>seata-all</artifactId>
                    <groupId>io.seata</groupId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>io.seata</groupId>
            <artifactId>seata-all</artifactId>
            <version>${seata.version}</version>
        </dependency>
        <!-- mybatis-plus依赖 -->
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
        <!-- mysql组件 -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <!-- 集成lombok 框架 -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>${spring-cloud-alibaba.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
````

***springcloud-seata-order订单服务数据源配置类如下***

````
@Configuration
public class DataSourceProxyConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DruidDataSource druidDataSource() {
        return new DruidDataSource();
    }

    @Primary
    @Bean
    public DataSourceProxy dataSource(DruidDataSource druidDataSource) {
        return new DataSourceProxy(druidDataSource);
    }
}
````
注意：这里使用的代理数据源，所以需要在启动类上面添加@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)注解，排除原有数据源

***springcloud-seata-order订单服务主要业务逻辑代码如下***
````
@Slf4j
@Service
public class OrderServiceImpl extends ServiceImpl<OrderMapper, Order> implements OrderService {

    @Autowired
    private OrderMapper mapper;

    @Autowired
    private AccountFeginClient accountFeginClient;

    @Autowired
    private StorageFeginClient storageFeginClient;

    @Override
    @GlobalTransactional
    public Result placeOrder(OrderVO orderVO) {

        Order order = Order.builder()
                .userId(orderVO.getUserId())
                .productId(orderVO.getProductId())
                .count(orderVO.getAmount())
                .money(orderVO.getPrice())
                .status(OrderStatus.INIT.getCode())
                .build();

        mapper.insert(order);

        log.info("保存订单{}", order.getId() != null ? "成功" : "失败");
        log.info("当前 XID: {}", RootContext.getXID());

        // 扣减库存
        log.info("开始扣减库存");
        Result storageResult = storageFeginClient.reduceStock(orderVO.getProductId(), orderVO.getAmount());
        log.info("扣减库存结果:{}", storageResult);

        // 扣减余额
        log.info("开始扣减余额");
        Result payResult = accountFeginClient.reduceBalance(orderVO.getUserId(), orderVO.getPrice());
        log.info("扣减余额结果:{}", payResult);

        order.setStatus(OrderStatus.SCUCCESS.getCode());
        Integer updateOrderRecord = mapper.updateById(order);
        log.info("更新订单:{} {}", order.getId(), updateOrderRecord > 0 ? "成功" : "失败");

        return Result.builder()
                .success(storageResult.isSuccess() && payResult.isSuccess())
                .build();
    }
}

````
这里采用feigin客户端调用支付和库存服务，并且需要方法上增加全局事务的注解@GlobalTransactional

***配置文件***

接下来我们需要在resources目录下创建registry.conf配置文件，和seata-server conf目录下registry.conf内容一致，如下：

````
registry {
  type = "nacos"

  nacos {
    serverAddr = "192.168.227.1"
    namespace = ""
    cluster = "default"
  }
}

config {
  type = "nacos"

  nacos {
    serverAddr = "192.168.227.1"
    namespace = ""
    group = "SEATA_GROUP"
  }
}
````
在 bootstrap.yml 中的配置如下：
````
spring:
  application:
    name: springcloud-seata-order
  cloud:
    nacos:
      # nacos config
      config:
        server-addr: 192.168.227.1  #Nacos配置中心地址
        namespace: public
        group: SEATA_GROUP
      # nacos discovery
      discovery:
        server-addr: 192.168.227.1  #Nacos注册中心地址
        namespace: public
        enabled: true
    alibaba:
      seata:
        tx-service-group: fsp_tx_group
````
spring.cloud.nacos.config.group ：这里的 Group 是 SEATA_GROUP ，也就是我们前面在使用 nacos-config.sh 生成 Nacos 的配置时生成的配置，它的 Group 是 SEATA_GROUP。

spring.cloud.alibaba.seata.tx-service-group ：这里是我们之前在修改 Seata Server 端配置文件 nacos-config.txt 时里面配置的 service.vgroup_mapping.${your-service-gruop}=default 中间的 ${your-service-gruop} 。这两处配置请务必一致，否则在启动工程后会一直报错 no available server to connect 

**注意：其他两个服务（支付服务和库存服务）配置如上**

**测试**
测试工具我们选择使用 PostMan ，启动三个服务，顺序无关。
使用 PostMan 发送测试请求，如图：

<div style="width:770px;height:200px;margin:50px 0px">
   <img alt="seata-postman.png" src="/images/seata-postman.png" width="770" height="200"/>
</div>

调用之前数据库数据如下：

<div style="width:908px;height:220px;margin:50px 0px">
   <img alt="seata-account1.png" src="/images/seata-account1.png" width="908" height="220"/>
</div>

<div style="width:908px;height:220px;margin:50px 0px">
   <img alt="seata-stock1.png" src="/images/seata-stock1.png" width="908" height="220"/>
</div>

运行结果如下：
<div style="width:908px;height:220px;margin:50px 0px">
   <img alt="seata-result1.png" src="/images/seata-result1.png" width="908" height="220"/>
</div>

<div style="width:908px;height:220px;margin:50px 0px">
   <img alt="seata-account2.png" src="/images/seata-account2.png" width="908" height="220"/>
</div>

<div style="width:908px;height:220px;margin:50px 0px">
   <img alt="seata-stock2.png" src="/images/seata-stock2.png" width="908" height="220"/>
</div>

操作成功以后，库存和账户余额都有相应的减少

下面我们来测试一下异常情况：
<div style="width:908px;height:220px;margin:50px 0px">
   <img alt="seata-postman2.png" src="/images/seata-postman2.png" width="908" height="220"/>
</div>

返回结果如下：
<div style="width:908px;height:220px;margin:50px 0px">
   <img alt="seata-result2.png" src="/images/seata-result2.png" width="908" height="220"/>
</div>

<div style="width:908px;height:220px;margin:50px 0px">
   <img alt="seata-account3.png" src="/images/seata-account3.png" width="908" height="220"/>
</div>

<div style="width:908px;height:220px;margin:50px 0px">
   <img alt="seata-stock3.png" src="/images/seata-stock3.png" width="908" height="220"/>
</div>

库存和余额并未较少，说明全局分布式事务已生效，控制台打印如下：

````
2020-04-03 17:29:20.677  INFO 19364 --- [nio-8080-exec-5] c.m.order.service.impl.OrderServiceImpl  : 保存订单成功
2020-04-03 17:29:20.677  INFO 19364 --- [nio-8080-exec-5] c.m.order.service.impl.OrderServiceImpl  : 当前 XID: 192.168.192.1:8091:2039608192
2020-04-03 17:29:20.677  INFO 19364 --- [nio-8080-exec-5] c.m.order.service.impl.OrderServiceImpl  : 开始扣减库存
2020-04-03 17:29:20.878  INFO 19364 --- [atch_RMROLE_1_8] i.s.core.rpc.netty.RmMessageListener     : onMessage:xid=192.168.192.1:8091:2039608192,branchId=2039608194,branchType=AT,resourceId=jdbc:mysql://192.168.227.1:3306/seata-order,applicationData=null
2020-04-03 17:29:20.880  INFO 19364 --- [atch_RMROLE_1_8] io.seata.rm.AbstractRMHandler            : Branch Rollbacking: 192.168.192.1:8091:2039608192 2039608194 jdbc:mysql://192.168.227.1:3306/seata-order
2020-04-03 17:29:21.016  INFO 19364 --- [atch_RMROLE_1_8] i.s.r.d.undo.AbstractUndoLogManager      : xid 192.168.192.1:8091:2039608192 branch 2039608194, undo_log deleted with GlobalFinished
2020-04-03 17:29:21.016  INFO 19364 --- [atch_RMROLE_1_8] io.seata.rm.AbstractRMHandler            : Branch Rollbacked result: PhaseTwo_Rollbacked
2020-04-03 17:29:21.421  INFO 19364 --- [nio-8080-exec-5] i.seata.tm.api.DefaultGlobalTransaction  : [192.168.192.1:8091:2039608192] rollback status: Rollbacked
2020-04-03 17:29:21.429 ERROR 19364 --- [nio-8080-exec-5] c.m.o.handler.GlobalExceptionHandler     : status 500 reading StorageFeginClient#reduceStock(Long,Integer); content:
{"timestamp":"2020-04-03T09:29:20.795+0000","status":500,"error":"Internal Server Error","message":"库存不足","path":"/storage/reduce"}
````

