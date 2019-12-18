# 1. zipkin集成介绍

<!-- TOC -->

- [zipkin集成介绍](#zipkin集成介绍)
    - [相关介绍](#相关介绍)
        - [简介](#简介)
        - [为什么使用zipkin](#为什么使用zipkin)
        - [相关连接](#相关连接)
    - [集成说明](#集成说明)
        - [zipkin 服务集成方案说明](#zipkin-服务集成方案说明)
        - [zipkin 服务调用链记录说明](#zipkin-服务调用链记录说明)
        - [zipkin 数据持久化说明](#zipkin-数据持久化说明)
    - [集成步骤](#集成步骤)
        - [增加依赖](#增加依赖)
        - [zipkin server搭建](#zipkin-server搭建)
        - [服务配置](#服务配置)
    - [使用介绍](#使用介绍)
        - [服务调用时序信息](#服务调用时序信息)
        - [服务依赖关系图](#服务依赖关系图)

<!-- /TOC -->
---

## 1.1. 相关介绍

    一句话介绍: zipkin 是一个开源分布式追踪系统 

### 1.1.1. 简介

Zipkin是一款开源的分布式实时数据追踪系统（Distributed Tracking System），基于 Google Dapper的论文设计而来，由 Twitter 公司开发贡献。其主要功能是聚集来自各个**异构**系统的实时监控数据。分布式跟踪系统还有其他比较成熟的实现，例如：Naver的Pinpoint、Apache的HTrace、阿里的鹰眼Tracing、京东的Hydra、新浪的Watchman，美团点评的CAT，skywalking等。  

    各个异构服务通过向 zipkin 报告时序数据,zipkin 会根据调用关系通过 Zipkin UI 生成依赖关系图。

> 一般的，一个分布式服务跟踪系统主要由三部分构成：   
***数据收集 -> 数据存储 -> 数据展示***

### 1.1.2. 为什么使用zipkin

随着业务越来越复杂，系统也随之进行各种拆分，特别是随着微服务架构和容器技术的兴起，看似简单的一个应用，后台可能有几十个甚至几百个服务在支撑；一个前端的请求可能需要多次的服务调用最后才能完成；当请求变慢或者不可用时，我们无法得知是哪个后台服务引起的，这时就需要解决如何快速定位服务故障点，Zipkin分布式跟踪系统就能很好的解决这样的问题。

### 1.1.3. 相关连接
> [官方地址](https://zipkin.io/)  
> [代码库](https://github.com/openzipkin/zipkin)  
<!-- > [zipkin链路跟踪](https://www.jianshu.com/p/1ef5cd97ba2b)   -->

## 1.2. 集成说明

### 1.2.1. zipkin 服务集成方案说明

zipkin 与 spring cloud (2.0之后) 进行整合非常简单,spring 官方提供了 [spring sleuth](https://cloud.spring.io/spring-cloud-sleuth/reference/html/#introduction) 为我们进行服务调用链路追踪的信息暴露,另外提供了基于 zipkin 的快速构建包 spring-cloud-starter-zipkin 进行 zipkin 和 sleuth 的整合。

### 1.2.2. zipkin 服务调用链记录说明
    
zipkin 服务搭建好默认只会记录服务调用的时序信息,针对依赖信息并没有进行记录,需要记录的话要依赖 zipkin-dependencies 子模块实现。  

    [zipkin-dependencies](https://github.com/openzipkin/zipkin-dependencies) 基于 spark job来生成全局的调用链,感兴趣的同学可以自行研究。

> **zipkin-dependencies** 默认只会进行一天的依赖关系记录,现在解决方案是服务器增加了定时任务,每天进行对 zipkin-dependencies 容器进行重新构建。

### 1.2.3. zipkin 数据持久化说明

    zipkin 默认的数据持久化是内存,在重新启动 zipkin 服务后,数据会丢失。zipkin 可选则的数据持久化有 MySQL , ES 等,我们选择使用 ES 进行zipkin 持久化数据的存储。

## 1.3. 集成步骤

### 1.3.1. 增加依赖

``` xml

<!-- 服务链路追踪 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>

```

### 1.3.2. zipkin server搭建

我们的 zipkin 是基于docker进行搭建的。

``` yml
version: '3'

servicesbuzhou
  zipkin:
    image: openzipkin/zipkin:2.19
    hostname: zipkin
    ports:
      - 9411:9411
    restart: always
    environment:
      - RABBIT_ADDRESSES=rabbitmq:5672
      - RABBIT_USER=admin
      - RABBIT_PASSWORD=leading2018
      - STORAGE_TYPE=elasticsearch
      - ES_HOSTS=elasticsearch:9200
      - /etc/localtime:/etc/localtime
      - /etc/timezone:/etc/timezone
    networks:
      swarm_net: 
        aliases:
          - zipkin

  zipkin-dependencies:
    image: openzipkin/zipkin-dependencies:2.4.1
    hostname: zipkin-dependencies
    restart: always
    environment:
      - STORAGE_TYPE=elasticsearch
      - ES_HOSTS=elasticsearch:9200
      - ES_TIMEOUT=20000
      - ES_INDEX=zipkin
      - /etc/localtime:/etc/localtime
      - /etc/timezone:/etc/timezone
      - rm=true
    networks:
      swarm_net: 
        aliases:
          - zipkin-dependencies

networks:
  swarm_net:
    external:
      name: swarm_net

```

### 1.3.3. 服务配置

因为我们使用了官方推荐的 rabbitmq 进行服务间调用及注册心跳的信息传递的方式,所以我们无需在服务中进行 zipkin-server 地址的配置。只需要根据需求在服务各自的配置文件中配置响应的采样率就可以。

``` yml

  sleuth:
    sampler:
      # 链路追踪采样率
      probability: 1.0
```

## 1.4. 使用介绍

主要是两大功能模块 **服务调用时序** 和 **服务依赖关系** 信息检索查看

### 1.4.1. 服务调用时序信息

可通过多个查询条件进行服务调用时序信息的查询检索,一下是各个查询条件的含义:  

* serviceName:服务名  
* spanName:对应的接口名  
* maxDuration:最大持续时间(即查询总请求链路持续时间小于该值的结果)  
* minDuration:最大持续时间(即查询总请求链路持续时间大于该值的结果)  
* tags:zipkin 扩展字段,我们没有  
* remoteServiceName: 远程调用时服务名,我们没有  

同时支持返回结果条数限制,时间范围搜索及结果排序功能

### 1.4.2. 服务依赖关系图

可直观的通过动态图的形式看整体服务调用的情况,另外可直接查看每个服务调用即被调用时正常数和失败数。

> 该功能提供的均为基本查询功能,如果需要进行详细的服务调用情况分析,还是推荐直接自定义开发进行持久化数据消费分析。
