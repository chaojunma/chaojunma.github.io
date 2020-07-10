---
layout: post
title: "SpringCloud简介"
date: 2020-7-10
categories: 微服务 分布式
tags: SpringCloud 
--- 

## 什么是Spring Cloud

Spring Cloud是一系列框架的有序集合。它利用Spring Boot的开发便利性巧妙地简化了分布式系统基础设施的开发，如服务发现注册、配置中心、消息总线、负载均衡、断路器、数据监控等，都可以用Spring Boot的开发风格做到一键启动和部署。Spring Cloud并没有重复制造轮子，它只是将目前各家公司开发的比较成熟、经得起实际考验的服务框架组合起来，通过Spring Boot风格进行再封装屏蔽掉了复杂的配置和实现原理，最终给开发者留出了一套简单易懂、易部署和易维护的分布式系统开发工具包。

## Spring Cloud常用组件

**Eureka**

服务注册和发现组件

**Ribbon**

负载均衡组件

**Feign**

声明式远程调度组件。

**Hystrix**

熔断器组件。它通过控制服务的API接口的熔断来转移故障，防止微服务系统发生雪崩效应。另外Hystrix能够起到服务限流和服务降级的作用。使用Hystrix Dashboard组件监控单个服务的熔断状态，使用Hystrix Turbine组件可以监控多个服务的熔断器的状态。

**Zuul**

智能路由网关组件。能够起到智能路由和请求过滤的作用，内部服务API接口通过Zuul网关统一对外暴露，防止内部服务敏感信息对外暴露。也可以实现安全验证，权限控制。

**Spring Cloud Config**

服务配置中心，将所有的服务的配置文件放到本地仓库或者远程仓库，配置中心负责读取仓库的配置文件，其他服务向配置中心读取配置。Spring Cloud Config使得服务的配置统一管理，并可以在不人为重启服务的情况下进行配置文件的刷新。

**Spring Cloud Bus**

消息总线组件，常和Spring Cloud Config配合使用，用于动态刷新服务的配置。

**Spring Cloud Gateway**

和Zuul类似，主要负责服务的转发、鉴权、限流等等