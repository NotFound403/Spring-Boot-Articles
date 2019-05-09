---
title: Spring Boot 2实践系列(八)：log4jdbc-log4j2集成和使用
comments: true
date: 2018-05-27 09:08:20
tags: [log4j,log4j2,log4jdbc]
categories: [Spring Boot 2实践系列]
---
　　通常项目中打印 SQL 执行语句参数位置是被占位符替换的，查询用到的参数并不在 SQL 语句中而是额外显示，也无法看到 SQL 的执行结果。

　　可以使用`Log4jdbc-log4j2`神器，在打印中直接看到完整的 SQL 语句，并格式化打印出执行结果。

　　**Log4jdbc-log4j2**是`Log4jdbc`的增强版，官网：http://log4jdbc.brunorozendo.com/
<!-- more -->
## 前提 ##
Spring Boot 已配置使用 Log4j2 为日志工具([Spring Boot实践系列(七)：log4j2集成和使用](http://112.74.59.39/2018/05/27/springboot-app-7-log4j2/)。

以下是在`log4j2.xml`文件配置的 **jdbc** 日志级别。
``` xml
<Loggers>
    <logger name="jdbc.sqlonly" level="INFO" />
    <logger name="jdbc.sqltiming" level="OFF"/>
    <logger name="jdbc.resultsettable" level="INFO"/>
    <logger name="jdbc.resultset" level="OFF"/>
    <logger name="jdbc.connection" level="OFF"/>
    <logger name="jdbc.audit" level="OFF"/>
</Loggers>
```

## 引入Log4jdbc-log4j2依赖 ##
``` xml
<!-- 监听sql执行详情
     用于数据库链接：datasource.url=jdbc:log4jdbc:mysql://
     jdbc驱动配置为：net.sf.log4jdbc.sql.jdbcapi.DriverSpy
     官方提示配置了4或4.1版本，会自动查找并加载该驱动,集成到SpringBoot未验证
-->
<dependency>
    <groupId>org.bgee.log4jdbc-log4j2</groupId>
    <artifactId>log4jdbc-log4j2-jdbc4.1</artifactId>
    <version>1.16</version>
</dependency>
```

## datasource.url ##
1. 修改数据源`url`地址
> 常规
> spring.datasource.url=jdbc:mysql://localhost:3306/sakila?characterEncoding=utf-8
> 修改后：主是在是前缀使用log4jdbc
> spring.datasource.url=jdbc:log4jdbc:mysql://localhost:3306/sakila?characterEncoding=utf-8

2. 修改数据源驱动
> 常规
> spring.datasource.driver-class-name=com.mysql.jdbc.Driver
> 修改后：使用lo4jdbc数据驱动
> spring.datasource.driver-class-name=net.sf.log4jdbc.sql.jdbcapi.DriverSpy

## slf4j记录SQL ##
使用`slf4j`来记录SQL日志，输出的日志会更精简。也可以不做此配置，使用`log4j2`打印。
在资源根目录`resources`下新建一个`log4jdbc.log4j2.properties`文件，内容如下:
> log4jdbc.spylogdelegator.name=net.sf.log4jdbc.log.slf4j.Slf4jSpyLogDelegator

## SQL日志示例 ##
> select city0_.city_id as city_id1_0_0_, city0_.city as city2_0_0_, city0_.country_id as country_3_0_0_, city0_.last_update as last_upd4_0_0_ from city city0_ where city0_.city_id=200 
> 2018-05-28 00:28:05,518 INFO [n.s.l.l.s.Slf4jSpyLogDelegator->resultSetCollected:610] [http-nio-8080-exec-1] 
> |---------|---------|-----------|----------------------|
> |city_id  |city     |country_id |last_update           |
> |---------|---------|-----------|----------------------|
> |[unread] |Hamilton |68         |2006-02-15 04:45:25.0 |
> |---------|---------|-----------|----------------------|


## log4jdbc ##
log4jdbc 是版本的 jar 包
``` xml
<!-- jdbc输出完整带参的SQL语句，而不是占位符 -->
<dependency>
    <groupId>com.googlecode.log4jdbc</groupId>
    <artifactId>log4jdbc</artifactId>
    <version>1.2</version>
</dependency>
```