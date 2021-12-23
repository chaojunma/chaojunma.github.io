---
layout: post
title: "Gateway + Nacos 实现动态路由"
date: 2021-12-23
categories: 分布式
tags: SpringCloud Gateway Nacos
--- 

今天我们通过Gateway + Nacos实现动态路由，整体的思路是将路由配置信息存储到Nacos配置中心，新增或者修改路由信息时动态刷新路由配置，使路由配置实时生效，不需要重启Gateway服务。


**动态路由配置**

```java
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.alibaba.nacos.api.NacosFactory;
import com.alibaba.nacos.api.config.ConfigService;
import com.alibaba.nacos.api.config.listener.Listener;
import com.alibaba.nacos.api.exception.NacosException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.cloud.gateway.event.RefreshRoutesEvent;
import org.springframework.cloud.gateway.route.RouteDefinition;
import org.springframework.cloud.gateway.route.RouteDefinitionWriter;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.context.ApplicationEventPublisherAware;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;
import javax.annotation.PostConstruct;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Executor;


@RefreshScope
@Component
public class DynamicRouteConfig implements ApplicationEventPublisherAware {

    private static Logger log= LoggerFactory.getLogger(DynamicRouteConfig.class);
    /**
     * nacos 配置dataId
     */
    @Value("${config.dynamic.route.dataId:gateway-routes}")
    private String dataId = "gateway-router";
    /**
     * nacos 配置group
     */
    @Value("${config.dynamic.route.group:DEFAULT_GROUP}")
    private String group = "DEFAULT_GROUP";

    /**
     * nacos 配置地址
     */
    @Value("${spring.cloud.nacos.config.server-addr}")
    private String serverAddr;

    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;

    private ApplicationEventPublisher applicationEventPublisher;

    private static final List<String> ROUTE_LIST = new ArrayList<>();

    @PostConstruct
    public void dynamicRouteByNacosListener() {
        try {
            ConfigService configService = NacosFactory.createConfigService(serverAddr);

// 程序首次启动, 并加载初始化路由配置
            String initConfigInfo = configService.getConfig(dataId, group, 5000);
            this.addAndPublishBatchRoute(initConfigInfo);

//添加监听路由变化
            configService.addListener(dataId, group, new Listener() {
                @Override
                public void receiveConfigInfo(String configInfo) {
                    clearRoute();
                    try {
                        List<RouteDefinition> gatewayRouteDefinitions = JSONObject.parseArray(configInfo, RouteDefinition.class);
                        for (RouteDefinition routeDefinition : gatewayRouteDefinitions) {
                            addRoute(routeDefinition);
                        }
                        publish();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }

                @Override
                public Executor getExecutor() {
                    return null;
                }
            });
        } catch (NacosException e) {
            e.printStackTrace();
        }
    }


    private void publish() {
        this.applicationEventPublisher.publishEvent(new RefreshRoutesEvent(this.routeDefinitionWriter));
    }

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.applicationEventPublisher = applicationEventPublisher;
    }

    /**
     * 批量 添加及发布 路由
     *
     * @param configInfo 配置文件字符串, 必须为json array格式
     */
    private void addAndPublishBatchRoute(String configInfo) {
        try {
            clearRoute();
            List<RouteDefinition> gatewayRouteDefinitions = JSONObject.parseArray(configInfo, RouteDefinition.class);
            for (RouteDefinition routeDefinition : gatewayRouteDefinitions) {
                addRoute(routeDefinition);
            }
            publish();
            log.info("Dynamic config gateway route finished. {}", JSON.toJSONString(gatewayRouteDefinitions));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 清除路由信息
     */
    private void clearRoute() {
        for (String id: ROUTE_LIST) {
            this.routeDefinitionWriter.delete(Mono.just(id)).subscribe();
        }
        ROUTE_LIST.clear();
    }

    /**
     * 添加路由
     *
     * @param definition
     */
    private void addRoute(RouteDefinition definition) {
        try {
            routeDefinitionWriter.save(Mono.just(definition)).subscribe();
            ROUTE_LIST.add(definition.getId());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

**Nacos配置中心路由配置**

```json
[
  {
    "id": "test",
    "predicates": [
      {
        "name": "Path",
        "args": {
          "_genkey_0": "/test1"
        }
      }
    ],
    "filters": [
      {
        "name": "StripPrefix",
        "args": {
          "_genkey_0": "1"
        }
      }
    ],
    "uri": "http://51ufo.cn",
    "order": 0
  },
  {
    "id": "es-log",
    "predicates": [
      {
        "name": "Path",
        "args": {
          "_genkey_0": "/es-log/**"
        }
      }
    ],
    "filters": [
      {
        "name": "RewritePath",
        "args": {
          "_genkey_0": "/es-log/(?<segment>.*)",
          "_genkey_1": "/$\\{segment}"
        }
      }
    ],
    "uri": "http://localhost:8080",
    "order": 0
  },
  {
    "id": "weight-route1",
    "predicates": [
      {
        "name": "Path",
        "args": {
          "_genkey_0": "/weight1"
        }
      },
      {
        "name": "Weight",
        "args": {
          "_genkey_0": "group1",
          "_genkey_1": "50"
        }
      }
    ],
    "filters": [
      {
        "name": "StripPrefix",
        "args": {
          "_genkey_0": "1"
        }
      }
    ],
    "uri": "http://www.baidu.com",
    "order": 0
  },
  {
    "id": "weight-route2",
    "predicates": [
      {
        "name": "Path",
        "args": {
          "_genkey_0": "/weight1"
        }
      },
      {
        "name": "Weight",
        "args": {
          "_genkey_0": "group1",
          "_genkey_1": "50"
        }
      }
    ],
    "filters": [
      {
        "name": "StripPrefix",
        "args": {
          "_genkey_0": "1"
        }
      }
    ],
    "uri": "http://51ufo.cn",
    "order": 0
  }
]
```