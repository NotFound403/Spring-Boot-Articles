---
title: Spring Boot 2实践系列(十一)：Ehcache集成详解和使用
comments: true
date: 2018-06-05 09:54:53
tags: [ehcache]
categories: [Spring Boot 2实践系列]
---
　　SpringBoot支持的缓存技术完全依赖于 Spring 对缓存技术的支持，了解 Spring 支持的缓存可以移步上一篇文章[Spring Boot实践系列(十)：数据缓存Cache](http://112.74.59.39/2018/05/31/springboot-app-10-cache/)

　　Spring 缓存技术支持 `Ehcache`，但要注意点的是 **Ehcache** 现在有两个版本，分别是**2.x**和**3.x**, **3.x**版本是`JSR-107`标准的实现，两者在配置和使用上存在较大的差异。

<!-- more -->
## EhCache 2.x ##
集成 Ehcache 2.x 非常简单，添加 EhCache 依赖, 创建配置文件**ehcache.xml**到项目根目录。
### 添加依赖 ###
``` java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>

<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

### ehcache.xml ###
``` xml
<?xml version="1.0" encoding="UTF-8"?>
<ehcache>
    <!--<diskStore path="java.io.tmpdir"/>
    <defaultCache
            maxEntriesLocalHeap="10000"
            eternal="false"
            timeToIdleSeconds="120"
            timeToLiveSeconds="120"
            maxEntriesLocalDisk="10000000"
            diskExpiryThreadIntervalSeconds="120"
            memoryStoreEvictionPolicy="LRU">
        <persistence strategy="localTempSwap"/>
    </defaultCache>-->
    <cache name="category" maxEntriesLocalHeap="1000"
           eternal="true" memoryStoreEvictionPolicy="FIFO"/>
</ehcache>
```

### 指定配置文件路径 ###
如果配置文件名是`ehcache.xml`并放在项目根目径下(resources/ehcache.xml)，可以省略指定，Spring 会自动找到该配置文件。

**application.properties**
``` 
#----------spring cache -------------------
spring.cache.type=ehcache
spring.cache.ehcache.config=classpath:ehcache.xml
```

### 业务代码 ###
业务代码与[Spring Boot实践系列(十)：数据缓存Cache](http://112.74.59.39/2018/05/31/springboot-app-10-cache/)中的**示例一**代码完全一致，使用第三方缓存技术，会自动注入对应的`cacheManager` **Bean**。 

[代码：https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-cache-ehcache2]

## EhCache 3.x ##
