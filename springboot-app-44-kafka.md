---
title: Spring Boot 2实践系列(四十四)：集成 Kafka 消息中间件
comments: true
date: 2019-05-09 08:51:00
tags: [kafka,消息]
categories: [Spring Boot 2实践系列]
---

　　spring-kafka 为支持 Apache Kafka 提供了自动配置。Spring Boot 集成 Kafka 的配置由 `spring.kafka.*` 属性控制。
<!-- more -->

## Kafka 服务

Kafka 服务安装与运行参考 [Kafka系列(一)：Kafka 介绍和安装运行、发布订阅](http://112.74.59.39:90/2019/05/08/kafka-01-install/)。

## 集成 Kafka

### 添加依赖

**pom.xml 导入 spring-kafka 包**

``` xml
<!--Kafka-->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

### 添加配置

在 **application.properties** 配置文件中添加连接 kafka 服务器的配置。

``` properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=myGroup
```





