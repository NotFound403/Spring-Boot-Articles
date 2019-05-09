---
title: Spring Boot 2实践系列(七)：log4j2集成和使用
comments: true
date: 2018-05-27 09:03:24
tags: [log4j2]
categories: [Spring Boot 2实践系列]
---
　　**官方说明：**Apache `Log4j 2`是对 **Log4j** 的升级，并提供了许多 **Logback** 可用的改进，同时解决了 **Logback** 体系结构中的一些固有问题。
	
　　官网：http://logging.apache.org/log4j/2.x/
<!-- more -->
## 排除Logback依赖 ##
Spring Boot默认已经集成 Logback 日志工具，要使用 **Log4j2**必须先排除 Logback。
``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <exclusions>
         <!--排除logback，后面配置log4j2-->
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

## 引入Log4j2依赖 ##
``` xml
<!--log4j2 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

## 配置Log4j2.xml文件 ##
创建`log4j2.xml`文件，放在工程`resources`目录里。
``` xml
<?xml version="1.0" encoding="UTF-8"?>

<!-- Configuration后面的status, 这个用于设置log4j2自身内部的信息输出, 可以不设置, 当设置成trace时,
    会看到log4j2内部各种详细输出, 可以设置成OFF(关闭) 或 Error(只输出错误信息), 30s 刷新此配置。
-->
<Configuration status="INFO" monitorInterval="30">
    <Properties>
        <Property name="log.path.root">logs</Property>
        <Property name="log.file">project_name</Property>
        <Property name="log.pattern">%d %p [%C{1.}->%M:%L] [%t] %m%n</Property>
    </Properties>

    <Appenders>
        <!-- 输出控制台日志 -->
        <Console name="Console" target="SYSTEM_OUT" follow="true">
            <!--控制台只输出level及以上级别的信息（onMatch），其他的直接拒绝（onMismatch）-->
            <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY"/>
            <PatternLayout>
                <charset>UTF-8</charset>
                <!-- 输出日志格式 -->
                <pattern>${log.pattern}</pattern>
            </PatternLayout>
        </Console>

        <!-- INFO 日志 -->
        <RollingFile name="INFO-LOG" fileName="${log.path.root}/info/${log.file}.log"
                     immediateFlush="false"
                     filePattern="${log.path.root}/info/${log.file}-%d{yyyy-MM-dd}_%i.log.gz">

            <!-- 日志过滤:只记录INFO级别的信息 -->
            <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY"/>
            <PatternLayout pattern="${log.pattern}"/>

            <!--志日文件打包策略-->
            <Policies>
                <!-- 按日期打包:
                     interval:int类型,日志文件滚动时间间隔,默认是1,单位精度与filePattern中配置的日期格式精度一致。
                              如上d%{yyyy-MM-dd}是精确到天,默认的interval=1表示每天滚动打包;
                                 d%{yyyy-MM-dd HH:mm}精确到分钟, 设置interval=30,表示每30分钟滚动打包
                     modulate:boolean类型,默认是false,是否对滚动文件的时间进行调制。
                            若modulate=true，则滚动时间将以0点为边界进行偏移计算调制时间,
                            如现在是20分钟,interval=30,下次文件滚动时间是30分钟,再下次是60分钟。
                -->
                <TimeBasedTriggeringPolicy modulate="true" interval="24"/>
                <!-- 按日志文件大小打包,达到20M打包 -->
                <!--<SizeBasedTriggeringPolicy size="20 MB"/>-->
            </Policies>
            <!--指定打包文件个数,默认7个,超过则覆盖之前的-->
            <!--<DefaultRolloverStrategy max="20"/>-->
        </RollingFile>

        <!-- ERROR 日志 -->
        <RollingFile name="ERROR-LOG" fileName="${log.path.root}/error/${log.file}.log"
                     filePattern="${log.path.root}/info/${log.file}-%d{yyyy-MM-dd}_%i.log.gz">
            <!--日志过-->
            <Filters>
                <ThresholdFilter level="ERROR" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
            <PatternLayout pattern="${log.pattern}"/>
            <Policies>
                <TimeBasedTriggeringPolicy modulate="true" interval="1"/>
            </Policies>
        </RollingFile>
    </Appenders>

    <Loggers>
        <logger name="jdbc.sqlonly" level="INFO"/>
        <logger name="jdbc.sqltiming" level="OFF"/>
        <logger name="jdbc.resultsettable" level="INFO"/>
        <logger name="jdbc.resultset" level="OFF"/>
        <logger name="jdbc.connection" level="OFF"/>
        <logger name="jdbc.audit" level="OFF"/>
        <Root level="INFO">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="INFO-LOG"/>
            <AppenderRef ref="ERROR-LOG"/>
        </Root>
    </Loggers>

</Configuration>
```
关于`Log4j2`的配置，详解见[Log：log4j2.xml 配置示例和异步日志详解 ](http://112.74.59.39/2018/03/31/log-log4j2-setting-sync-logger/)。

## 调用Logger ##
``` java
import com.alibaba.fastjson.JSON;
import com.springboot.example.entity.User;
import com.springboot.example.service.UserService;
//log4j2 依赖
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping(value = "/user",method = RequestMethod.POST)
public class UserController {
	// 定义静态不可变的Logger变量，传入当前类的Class对象。
    private static final Logger logger = LogManager.getLogger(UserController.class);

    @Autowired
    private UserService userService;

    @RequestMapping("/query")
    public User addUser(User user){
        user.setAddress("中国").setAge(11).setName("Kitty");
        logger.info("user:{}", JSON.toJSONString(user));
        int i = userService.addUser(user);
        return user;
    }
}
```
logger.info输入示例：city:{"city":"Hamilton","cityId":200,"countryId":68,"lastUpdate":1139949925000}