---
title: Spring Boot 2实践系列(二十八)：Spring Session 集成配合Redis实现Session集群共享
comments: true
date: 2018-08-15 22:36:54
tags: [session,redis,HttpSession,session集群共享]
categories: [Spring Boot 2实践系列]
---
　　Web应用的 Session 默认是由服务器容器(如:Tomcat)管理，当部署服务器集群时，会出现在服务器A登录后，再次访问被负载均衡转发到服务器B时会被要求重新登录，这对访问同一个域名站点用户来说是非常不友好的体验，登录服务器的 Session 无法被集群中的其它服务器共用, 这就是集群环境下的 Session 共享问题。

　　解决集群环境下的 Session 问题的思路基本有三种：
　　1. **服务器 Session 复制**：让集群中每台服务器都有其它服务器的Session, 这是服务器Session复制机制。
　　2. **持久化 Session**：自定义 Session管理实现，接管容器对Session的管理，把 Session 持久化提供公共使用，每次请求根据 SessionId到 Session 缓存服务器取 Session 进行判断。
　　3. **在负载均衡服务器配置粘性 Session：**即把同一个 Session 的多个访问始终绑定到同一台服务器，不推荐使用, 当该服务器宕机机就会出现 Session 丢失，没有利点集群的优势。

　　目前行业广泛使用且推荐使用的是`持久化Session`，对Session 执行统一的存储和访问，实现 Session 在集群服务中的共享。
<!-- more -->
## Session 复制 ##
容器 Tomcat 自身提供了简单的 Session 复制策略(all-to-all)，会把 Session 复制到集群中的其它节点(可理解为广播的方式)，复制机制会占用网络带宽增加网络负载反应缓慢，官方建议是在小集群中可以使用，不建议用在较大集群的环境中。

关于 Tomcat 自带的 Session 复制的配置，可参考[官方文档：Tomcat 8.0 -> Clustering/Session Replication HOW-TO](http://tomcat.apache.org/tomcat-8.0-doc/cluster-howto.html)

## Session 持久化共享 ##
Session 持久化到数据库，当集群中的其它服务器接收负载均衡转发过来的请求时，应用服务器将创建的新 Session 存到数据库中，再次访问时从 Session 数据库中取出 Session，这样集群服务器统一访问 Session 数据库，实现了 Session 在集群服务器之间的共享。

Session 持久化操作是通过自定义**过滤器**接管容器对 Session 的管理, 实现有多种方式：
1. 使用中间插件：[tomcat-redis-session-manager](https://github.com/jcoleman/tomcat-redis-session-manager)，该插件官方只支持 Tomcat 6 和 Tomcat 7，不支持更新的 Tocmat 8 和 Tomcat 9；若要在 Tomcat 8 和 Tomcat 9中使用，可使用[tomcat-cluster-redis-session-manager](https://github.com/ran-jit/tomcat-cluster-redis-session-manager), 中间件的方式配置非常简单，参考**GitHub**上的项目说明。
2. 使用 Shiro 提供的 Session 管理：Shiro 提共了默认的 Session 管理器(SessionManager), 实现 Servlet 容器的 Session 管理(ServletContainerSessionManager), 关于配置和使用可参考 [Shiro官方文档 -> Session Management](http://shiro.apache.org/web.html#Web-SessionManagement) 或网上相关资源。
3. 使用 Spring Session 管理：Spring Session提供了用于管理用户 session 的API和实现，透明集成的 HttpSession,允许替换容器的 HttpSession 管理，可以非常方便的支持集群环境的 Session共享。Spring Session 也是本篇要重点描述的。

## Spring Session ##
Spring Session 支持将 Session 存储到 NoSQL 数据库(Redis, GemFire, Hazelcast, MongoDB)，还支持通过 JDBC 存储到关系数据库(MySQL,Oracle)。推荐使用 Redis 来缓存 Session，Redis 的**过期时间特性**正好满足 Session 过期失效的需求。

使用 Spring Session 持久化 Session 时，添加到 Session 中的实体类必须实现序列化接口，否则会报错，并且抛出的错误根本看不出具体原因，必须跟踪源码内部的错，才可知因无法序列化导致的错误。

Spring Session 官方：[Spring Session](https://spring.io/projects/spring-session#learn), 文档[Spring Session Doc](https://docs.spring.io/spring-session/docs/2.0.4.RELEASE/reference/html5/)

## Spring Boot - Spring Session ##
Spring Boot 框架为集成 Spring Session 数据存储提供了自动配置，支持`JDBC, Redis, Hazelcast, MongoDB`的自动配置，在使用上非常简单方便。
Spring Boot 集成 Spring Session Redis 文档：[Spring Boot -> Spring Session](https://docs.spring.io/spring-boot/docs/2.0.4.RELEASE/reference/htmlsingle/#boot-features-session)，[Spring Session - Spring Boot](https://docs.spring.io/spring-session/docs/2.0.4.RELEASE/reference/html5/guides/boot-redis.html)。

### 集成Spring Session Redis ###
Spring Boot 集成 Spring Session Redis 步骤如下：
1. 添加依赖:spring-session-data-redis
``` xml
<!-- redis -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<!-- spring-session-redis -->
<dependency>
	<groupId>org.springframework.session</groupId>
	<artifactId>spring-session-data-redis</artifactId>
</dependency>
```
2. 配置 session-redis, 在 application.properties 文件添加如下设置
更多属性值设置参考自动配置包下的Session属性配置类的源码`org.springframework.boot.autoconfigure.session.SessionProperties`。
``` 
spring.session.store-type=redis # Session store type.
// session过期时间，默认1800秒，即30分钟
server.servlet.session.timeout=1800 # Session timeout. If a duration suffix is not specified, seconds will be used.
// session更新模式，默认是保存时更新
spring.session.redis.flush-mode=on-save # Sessions flush mode.
// 保存session的名称空间，可理解为key值，默认是spring:session:sessions
spring.session.redis.namespace=spring:session # Namespace for keys used to store sessions.
```
3. 设置 Redis 联接，Redis 服务连接必须是可用的。
``` 
spring.redis.host=localhost # Redis server host.
spring.redis.password= # Login password of the redis server.
spring.redis.port=6379 # Redis server port.
spring.redis.database=0
```

以上三步就完成集成设置了，这也是 Spring Boot 自动配置的魅力所在：方便，快捷，高效。

### Spring Session 自动配置分析 ###
Spring Session 自动配置在 Servlet 容器初始化时，Spring Boot 的自动配置会创建一个名为`springSessionRepositoryFilter`的Spring Bean，该类继承自`OncePerRequestFilter`，实现了`Filter`。springSessionRepositoryFilter Bean 负责将 HttpSession 替换为 Spring Session 支持的自定义实现。

1. **SessionRepositoryFilter(springSessionRepositoryFilter)**：这个是启用Spring Session的核心过滤器，必须放在任何访问HttpSession或提交响应的Filter 之前，原码里`@Order`顺序是 **Integer.MIN_VALUE + 50**，负数最小值加50，在过滤器链中的排序是值越小越早执行，该类可以说是第一个执行以确保启用 Spring Session。该过滤器用于替换 HttpSession , 执行一系列 Spring Session 定义的 Session 操作。
2. **SpringHttpSessionConfiguration**：执行一些 Spring Session 的基础配置。该类是 Spring Boot 给 Spring Session 提供自动配置的核心配置类，在该类中注入了 **springSessionRepositoryFilter**。其它自定义的 session 实现继承此类，如：`RedisHttpSessionConfiguration`。
Spring Session Redis 配置类 `RedisSessionConfiguration` 注入了`SpringBootRedisHttpSessionConfiguration`，SpringBootRedisHttpSessionConfiguration继承自`RedisHttpSessionConfiguration`, RedisHttpSessionConfiguration 的父类是**SpringHttpSessionConfiguration**。![](http://112.74.59.39/images/1534736029099.png)
3. **SessionRepository**：管理 Session 存储的接口，包含`增,删,改,查`四个基本操作, 具体的存储操作类实现该接口并添加自定义操作, 如：RedisOperationsSessionRepository。 
![](http://112.74.59.39/images/1534752860783.png)

## HttpSessionListener ##
使用Spring Session 管理 Session后，在使用实现 HttpSessionListener 的自定义Session监听器上有些差异，容易被坑。

### Spring Session 管理 ###
Spring Session 接管了原本由容器管理的 HttpSession , 并将处理 Session 相关的资源(session 监听器)交由 Spring 容器管理。
使用Spring Session 管理Session，自定义的 HttpSessionListener 的实现类里可以直接注入 RedisTemplate, 类上的注解必须使用Spring 标注为Bean的注解(@Component, @Configuration)来将 Session 监听器注册为 Spring Bean 才会启效。
``` java
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpSessionEvent;
import javax.servlet.http.HttpSessionListener;

/**
 * @Name: CountOnlineUserListener
 * @Desc: 统计在线用户数
 **/

@Component
public class CountOnlineUserListener implements HttpSessionListener {

    private static final Logger logger = LogManager.getLogger(CountOnlineUserListener.class);

    private static final String TOTAL_ONLINE_USER="totalOnlineUser";

    @Autowired
    private RedisTemplate<String, Integer> redisTemplate;

    @Override
    public void sessionCreated(HttpSessionEvent event) {
        logger.info("=============Session监听:创建 Session===============");
        Integer integer = redisTemplate.opsForValue().get(TOTAL_ONLINE_USER);
        integer = (integer == null ? 0 : integer);
        //创建Session数
        redisTemplate.opsForValue().set(TOTAL_ONLINE_USER,integer + 1);
    }

    @Override
    public void sessionDestroyed(HttpSessionEvent se) {
        logger.info("=============Session监听:Session 失效===============");
        Integer integer = redisTemplate.opsForValue().get(TOTAL_ONLINE_USER);
        redisTemplate.opsForValue().set(TOTAL_ONLINE_USER,integer - 1);
    }
}
```
**备注**：此方式下在自定义的 HttpSessionListener 实现类上使用 `@WebListener` 注解是无效的，**@WebListener**注解由**Servlet**提供,注解的监听器不会启动 。


### Servlet 容器管理Session ###
HttpSession 默认是由 Servlet 容器管理。 使用`@WebListene`注解的自定义 Session 监听器随 Servlet 容器启动而启动并保持一直监听, 比 Spring IoC 容器启动的更早，在自定义Session 监听器类里使用 注入 Spring Bean 的 **@Autowired** 注解无效, 该注解注释的属性值为 `null`, 这时要使用 Spring IoC 容器里的 Bean(如:RedisTemplate)时，必须手动调用容器根据 Bean 的 Id 或 Bean 的 Class 对象 获取 Bean 实例。

可以使用 @Component, @Configuration 注解将该监听器注册为 Spring Bean; 也可用 @ServletComponentScan 和 @WebListener注解, 会扫描 @WebListener 注解的类注册为Spring Bean。

``` java
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.web.context.WebApplicationContext;
import org.springframework.web.context.support.WebApplicationContextUtils;

import javax.servlet.ServletContext;
import javax.servlet.annotation.WebListener;
import javax.servlet.http.HttpSession;
import javax.servlet.http.HttpSessionEvent;
import javax.servlet.http.HttpSessionListener;

/**
 * @Name: CountOnlineUserListener
 * @Desc: 统计在线用户数
 **/

@WebListener
public class CountOnlineUserListener implements HttpSessionListener {

    private static final Logger logger = LogManager.getLogger(CountOnlineUserListener.class);

    private static final String TOTAL_ONLINE_USER="totalOnlineUser";

    private RedisTemplate<String, Integer> redisTemplate;

    @Override
    public void sessionCreated(HttpSessionEvent event) {
        logger.info("=============Session监听:创建 Session===============");
        HttpSession session = event.getSession();
        session.setMaxInactiveInterval(1 * 60);
        ServletContext servletContext = session.getServletContext();
        WebApplicationContext webApplicationContext = WebApplicationContextUtils.getWebApplicationContext(servletContext);
        redisTemplate = (RedisTemplate<String, Integer>) webApplicationContext.getBean("redisTemplate");
        Integer integer = redisTemplate.opsForValue().get(TOTAL_ONLINE_USER);
        integer = (integer == null ? 0 : integer);
        //创建Session数
        redisTemplate.opsForValue().set(TOTAL_ONLINE_USER,integer + 1);
    }

    @Override
    public void sessionDestroyed(HttpSessionEvent se) {
	logger.info("=============Session监听:Session 失效===============");
        Integer integer = redisTemplate.opsForValue().get(TOTAL_ONLINE_USER);
        redisTemplate.opsForValue().set(TOTAL_ONLINE_USER,integer - 1);
    }
}
```
**备注：**如果使用 Spring Boot 内嵌的 Servlet 容器，自定义的 session 监听器上使用了 @WebListener注解，还需要在使用 @Configuration 注解的类上添加@ServletComponentScan注解, 并设置要扫描的包的路径为自定义的监听器所在的包路径，所以大多会把该注解放在 Spring Boot 的入口类上。
关于`@WebListener`和`@ServletComponentScan`注解的使用，可参考[Spring Boot实践系列(二十七)：Listener, Filter, Interceptor ](http://112.74.59.39/2018/08/14/springboot-app-27-listener-filter-interceptor/)

**其它**
[关于Tomcat禁用Session探讨](http://zyqwst.iteye.com/blog/2309004)
