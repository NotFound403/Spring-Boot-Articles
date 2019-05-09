---
title: Spring Boot 2实践系列(六)：应用监控模块 Actuator 详解和集成
comments: true
date: 2018-05-25 17:02:30
tags: [actuator,endpoints]
categories: [Spring Boot 2实践系列]
---
　　Spring Boot提供了一些非常实用的附加功能组件，比如应用监控模块 `Actuator`。 **Actuator** 可以采集应用和系统环境的一些指标数据，通过端点(endpoint)对外提供这些数据，用户可根据这些数据来对应用进行监控和管理。可以选择使用**HTTP,JMX,SSH**来管理和监控。该组件会自动对应用审计，健康和收集相关指标信息。

　　**spring-boot-actuator** 提供了很多监控应用程序所需的神奇的运维特性，可以查看了解应用程序运行时的内部工作细节，可以查看`IoC`容器都注册了那些`Bean`、**Spring MVC**控制器的路径映射、请求跟踪、系统环境、配置属性、日志设置、程序信息、活动线程快照、堆存储信息，还可以通过端口来关闭应用。

　　`Actuator`组件非容易地使用，只需要添加依赖`spring-boot-starter-actuator`。　　

<!-- more -->
## 2.0.x Actuator ##
本篇内容基于 Spring Boot 2.0.x.Release 版本了，与 1.5.x.Release 相比在设置上有些改动。
Spring Boot 2.0.x官网: https://docs.spring.io/spring-boot/docs/2.0.2.RELEASE/reference/htmlsingle/

### 主要修改点 ###
1. 默认暴露的端点不同
HTTP访问监控端点，在 2.0.x 版本，默认暴露的端点只有两个，分别是`info,health`。如果是`JMX`方式访问, 则所有端点都是开放的。
【官方说明】For security purposes, all actuators other than /health and /info are disabled by default. The management.endpoints.web.exposure.include property can be used to enable the actuators.
【译】出于安全的目的，除了 /health 和 /info ,其它端点默认是关闭的, 可以使用 management.endpoints.web.exposure.include 属性来开启监控端点。
	开启所有
	```
	management.endpoints.web.exposure.include=*
	```
	开启指定端点
	```
	management.endpoints.web.exposure.include=beans,env,info,mappings
	```
	在开启所有的前提下排除端点
	```
	management.endpoints.web.exposure.exclude=env,beans
	```
2. 端点映射路径调整
之前在 endpoints 下的属性被移到了 management 下; 所有端点移到 base-path 路径下, 默认是 /actuator
	1.5.x 版本
	```
	endpoints.beans.enabled=true
	```
	2.0.x 版本
	```
	management.endpoint.beans.enabled=true
	```

## 添加依赖 ##
``` xml
<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-actuator</artifactId>
	</dependency>
</dependencies>
```

## 监控端点 ##
监控端点用于监控应用并进行交互。Spring Boot包含许多内置端点，并允许定制自己的端点。每个端点都可以**启用或禁用**，这控制着端点是否被创建。

| 端点                | 描述     			|  默认是否开启  |  
| -------------------| -------------------- |:------------:|
| actuator      	 | 所有端点列表       			|   yes    |
| health        	 | 报告应用程序的健康指标，这些值由HealthIndicator的实现类提供|   yes    |
| shutdown      	 | 关闭应用程序，要求endpoints.shutdown.enabled设置为true       |   no    |
| hystrix.stream	 | 若是微服务应用，使用 Spring Cloud 和 hystrix 熔断器，就会有此端点    |   yes    |
| env           	 | 获取系统环境属性和应用配置的属性      |   yes |
| beans         	 | 描述应用程序上下文里全部的Bean，以及它们的关系     |   yes    |
| heapdump      	 | 返回一个 GZip 压缩的 hprof 堆信息文件      |   yes    |
| trace         	 | 自动跟踪最近的100个请求-响应交换的基本信息|   yes		| 
| autoconfig    	 | 提供了一份自动配置报告，记录哪些自动配置条件通过了，哪些没通过|   yes    |
| auditevents   	 | 获取当前应用的审计事件信息      |  yes  | 
| loggers        	 | 获取应用的日志配置信息      |   yes    |
| dump   			 | 获取线程活动的快照|   yes    |
| mappings           | 描述全部的URI路径，以及它们和控制器（包含Actuator端点）的映射关系|   yes    |
| info               | 获取应用程序的定制信息，这些信息由info打头的属性提供      |   yes 	|
| metrics            | 报告各种应用程序指标信息，比如内存用量和HTTP请求计数     |   yes 	|
| configprops        | 显示使用@ConfigurationProperties注解的注入属性参数列表|   yes 	|

1. /env
基本上，任何能给Spring Boot应用程序提供属性的属性源都会列在/env的结果里，同时会显示具体的属性。一些敏感信息，如属性尾部包含**password,secret,key**的属性，在`/env`里,值都会被`*`替换。
该端点也可用于获取单个属性的值：http://localhost:8081/env/JAVA_HOME
2. /configprops
该端点会对环境属性注入带有 @ConfigurationProperties 注解的Bean的实例属性生成一个报告，说明是如何进行设置。
这个报告也能作为一个快速的参考指南。例如，若不清楚内置Tomcat的最大线程数，可以看配置属性报告里的`server.tomcat.max-threads`。
3. /metrics
该端点提供对应用运行时的度量信息。例如，内存情况、垃圾回收器、堆、类加载器、线程池、数据源、Tomcat会话、HTTP请求的度量数据。
4. /trace
在内存里维护了一个跟踪库，可以跟踪最近100条的请求响应信息。
``` json
[
    {
        "timestamp": "2018-05-28 18:19:56",
        "info": {
            "method": "POST",
            "path": "/ajaxLogin",
            "headers": {
                "request": {
                    "content-type": "application/json",
                    "cache-control": "no-cache",
                    "postman-token": "b4d260bc-b95b-4cf6-b839-6217ac1c4669",
                    "user-agent": "PostmanRuntime/7.1.1",
                    "accept": "*/*",
                    "host": "localhost:8080",
                    "cookie": "JSESSIONID=34c30a59-b8f1-41bd-9095-788770c0bfe7",
                    "accept-encoding": "gzip, deflate",
                    "content-length": "48",
                    "connection": "keep-alive"
                },
                "response": {
                    "X-Application-Context": "wd-qyd:dev",
                    "Set-Cookie": "JSESSIONID=bca276e7-ba28-4189-8c4c-bc210a131f9d; Path=/; HttpOnly",
                    "Content-Type": "application/json;charset=UTF-8",
                    "Transfer-Encoding": "chunked",
                    "Date": "Mon, 28 May 2018 10:19:56 GMT",
                    "status": "200"
                }
            },
            "timeTaken": "89"
        }
    }
]
```
5. /dump
当前线程快照，包含很多线程的特定信息，线程相关的阻塞和锁状态。还会有一个**跟踪栈**
6. /health
监控应用程序的健康情况。可以看到服务、db、diskSpace等信息。Spring Boot自带了一些健康检测的指标器并按需自动配置。

## http ##
### 访问端点 ###
访问端点路径是由: **http://域名:监控端口/监控路径/端点** 组成，例如:**http://localhost:8081/app/beans**
访问端点的路径里的**监控端口**，**监控路径**，**端点**都是可设置的。

### 监控设置 ###
可以对监控进行一些定制,在**Spring Boot**项目配置文件进行设置，如设置监控端口，监控路径等。
监控对象的状态有`up`和`down`两个，**up**表示正常，**down**反之。

1. 定制端点访问端口
默认监控的端口和是访问工程的端口是同一个，建议定制监控端口。
```
management.server.port=8081
```
2. 定制允许访问监控的IP
默认允许所有IP地址可以访问监控端点，建议设置为本地IP。
```
management.server.address=127.0.0.1
```
3. 定制端点访问路径
```
#默认:/actuator
management.server.servlet.context-path=/project
management.endpoints.web.base-path=/manage

#----访问--------------------------------
http://localhost:8081/project/manage
```
4. 访问端点
如果不设置监控访问路径和端点访问路径，则会有默认的监控访问路径。
```
#---------访问端点列表-----------------
默认：http://localhost:8081/actuator
定制：http://localhost:8081/project/manage
显示所有启用开放的端点列表，端点列表里包含了端点的引用(rel)和链接(href)
```

### 端点控制 ###
除了`shutdown`端点，其它所有端点默认都是启用的，可通过`endpoint.<id>.enabled`来设置端点的**启用**或**禁用**，有`true`和`false`可设置，`id`指的是端点的名称。如下示例：
```
#----全局控制所有端点开启/关闭，默认是true---------
endpoints.enabled=true

#----------只启用所需端点------------------------
endpoints.beans.enabled=false		// false:关闭所有端点
endpoints.info.enabled=true			// http://localhost:8081/app/info

#----------启用shutdown端点---------------------
endpoints.shutdown.enabled=true			//http只能是post请求

#----------访问：关闭应用------------------------
http://localhost:8081/app/shutdown
```

### 自定义端点 ###
1. 自定义端点名称
```
设置：endpoints.beans.id=appbeans
结果：
{
    "rel": "appbeans",
    "href": "http://localhost:8081/app/appbeans"
}
```
2. 自定义端点访问路径
``` 
设置：endpoints.beans.path=/bean
结果：
{
    "rel": "appbeans",
    "href": "http://localhost:8081/app/bean"
}
```
3. 更多自定义端点和健康检测可参考官方文档

## 权限设置 ##
除了用于查看端点列表的根端点，如默认的`/actuator`或定制的`/app`，其它端点的访问默认都是开启了权限控制的。
1. 端点访问权限
默认是`true`，即开启权限控制；设置为`false`则关闭权限控制，所有端点都可不受限的访问。
```
management.security.enabled=true
```

**备注：**因未使用过 Spring Security，对于使用 Shiro 来做权限控制的项目，在端点开始权限控制情况下如何访问端点官方没有说明或示例，待后续研究。

【参考】
官网指南:https://docs.spring.io/spring-boot/docs/1.5.13.RELEASE/reference/htmlsingle/#production-ready


## 其它配置 ##
其它常用的配置下面列了下，更多配置参考官方文档
```
# ----------------------------------------
# ACTUATOR PROPERTIES
# ----------------------------------------

# MANAGEMENT HTTP SERVER (ManagementServerProperties)
//允许访问监控的IP地址
management.server.address= # Network address to which the management endpoints should bind. Requires a custom management.server.port.
//访问监控的端口
management.server.port= # Management endpoint HTTP port (uses the same port as the application by default). Configure a different port to use management-specific SSL. 

# ENDPOINTS WEB CONFIGURATION (WebEndpointProperties)
//暴露的端点
management.endpoints.web.exposure.include=health,info # Endpoint IDs that should be included or '*' for all.
//排除的端点
management.endpoints.web.exposure.exclude= # Endpoint IDs that should be excluded.
//监控入口路径映射
management.endpoints.web.base-path=/actuator # Base path for Web endpoints. Relative to server.servlet.context-path or management.server.servlet.context-path if management.server.port is configured.
//在入口路径和端点之间的路径映射
management.endpoints.web.path-mapping= # Mapping between endpoint IDs and the path that should expose them.
```

## JMX ##
通过 **JMX** 对应用进行监控和管理。

## SSH ##
通过 **SSH** 或**TELNET**监控和管理我们的应用，Spring Boot借助` CraSH `来实现，只需添加`spring-boot-starter-remote-shell`依赖即可。

## 备注 ##
Spring Boot Actuator 给运维对应用监控提供了很好的支持,如果能结合运维监控平台就完美了，已有第三方提供了监控平台的组件，包含服务端和客户端(spring-boot-admin-server,spring-boot-admin-server-ui,spring-boot-admin-starter-client),具体整合使用见[Spring Boot实践系列(十六)：集成监控管理Web平台和客户端](http://112.74.59.39/2018/06/17/springboot-app-16-actuator-admin/)。

多看官方文档, 国内也有些不错的博文可以参考下：
[Spring boot 2.0 Actuator 的健康检查](https://www.jianshu.com/p/1aadc4c85f51)
[译 Spring Boot 2.0官方文档之 Actuator](https://blog.csdn.net/alinyua/article/details/80009435)
