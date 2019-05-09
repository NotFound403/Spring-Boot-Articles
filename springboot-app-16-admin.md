---
title: Spring Boot 2实践系列(十六)：集成监控管理Web平台和客户端
comments: true
date: 2018-06-17 16:54:35
tags: [actuator,server,client,admin,spring-boot-admin]
categories: [Spring Boot 2实践系列]
---
　　Spring Boot 提供了 Actuator 组件来监控应用运行情况，而Actuator监控的端点返回的只是`json`格式的数据，可以结合 Admin 监控组件能来使用。

　　Admin组件包含 **服务端**和注册到服务端的**客户端**，服务端提供了 Web 视图页面包含了图表，可以更方便更直观的了解应用运行情况。

　　Spring Boot Admin 官方参考指南：http://codecentric.github.io/spring-boot-admin/2.0.0/  多看官网技术文档。
<!-- more -->
　　基于 Spring Boot 2.0.x 和 spring-boot-admin-server 2.0.x版本的整合和演示。

## 服务端 ##
新建一个独立的项目, 不包含业务处理, 只添加** Spring Boot Admin Server** 的依赖，作为一个统一的管理应用的平台; 或者在当前系统上添加 **Spring Boot Admin Server** 和 **Clien** 来集成 **Spring Boot Admin**。

Spring-boot-admin-server 已经包含了 spring-boot-starter-actuator依赖包。

1. 添加依赖
服务端添加**admin-server**依赖
``` xml
<!-- starter web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!-- 服务端：admin-server -->
<dependency>  
	<groupId>de.codecentric</groupId>  
	<artifactId>spring-boot-admin-server</artifactId>
    <version>2.0.0</version>
</dependency>  
<dependency>
	<groupId>de.codecentric</groupId>  
	<artifactId>spring-boot-admin-server-ui</artifactId>
    <version>2.0.0</version>
</dependency>
```
2. 添加开启管理服务注解`@EnableAdminServer`
``` java
@Configuration
@EnableAutoConfiguration
@EnableAdminServer
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```
3. 配置管理页面映射路径
访问路径：http://127.0.0.1/admin
``` 
spring.boot.admin.context-path=/admin
```

## 客户端 ##
spring-boot-admin-starter-client 已经包含了 spring-boot-starter-actuator 依赖。
客户端可以是额外的单独的业务项目，作用被监控和管理的客户端，注册到服务器。
1. 添加依赖
``` xml
<!-- 客户端：admin-client -->
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-client</artifactId>
    <version>2.0.0</version>
</dependency>
```
2. 设置应用管理监控属性
监控默认访问路径：http://127.0.0.1:8081/actuator
``` propertie
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
management.server.port=8081
management.server.address=127.0.0.1
```
3. 设置注册到管理服务器的地址
``` properties
spring.boot.admin.client.url=http://localhost/admin
```

## 访问管理Web ##
浏览器访问服务器的管理页面：http://localhost/admin#/applications
![](http://112.74.59.39:90/images/1529406162298.png)
![](http://112.74.59.39:90/images/1529406242507.png)
![](http://112.74.59.39:90/images/1529406282787.png)

## 邮件通知 ##
Spring Boot Admin 支持配置邮件来发送邮件通知。
1. 引入邮件依赖
``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```
2. 增加邮件服务器配置
```
spring.mail.host=smtp.qq.com
spring.mail.username=username@xx.com
spring.mail.password=password
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
spring.mail.properties.mail.smtp.starttls.required=true
# 发送给谁
spring.boot.admin.notify.mail.to=to_admin@xxx.com
# 是谁发送出去的
spring.boot.admin.notify.mail.from=from_user@xxx.com
```

[原码 -> Github](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-admin)

## Spring Security ##
可以整合 **spring-boot-starter-security**来对客户端注册和后台Web访问权限进行定制，后续再额外详解。

## Spring Cloud ##
可以整合到 Spring Cloud 项目中，添加**spring-cloud-starter-netflix-eureka-client**依赖，用于发现和管理服务,后续再额外详解。

