---
title: Spring Boot 2实践系列(十四)：配置文件profile属性和部署jar包
comments: true
date: 2018-06-10 10:43:18
tags: [spring boot,profile,application,dev,prod,test]
categories: [Spring Boot 2实践系列]
---
　　Spring Boot 创建时会生成默认的配置文件：`application.properties`，该文件中的配置优先级最高。
　　
　　**Profile**是**Spring**为不同的环境来激活相应的的配置文件提供支持， **profile** 全局配置：`application-{profile}.properties`。
<!-- more -->

## profile ##
Spring Boot项目实际开发中常会用以下三个配置文件：
1. 开发环境：**application-dev.properties**
2. 生产环境：**application-prod.properties**
3. 测试环境：**application-test.properties**

然后在`application.properties`中设置`spring.profiles.active=profile`来指定活动的`Profile`。
如：**spring.profiles.active=dev**

### 配置示例 ###
在`src/main/resources`里创建`profile`配置文件。
1. 开发环境：**application-dev.properties**： server.port=81
2. 生产环境：**application-prod.properties**：server.port=82
3. 运行：**application.properties**: spring.profiles.active=dev
这里就启用了开发模式下的配置，服务器端口为`81`。

## jar包部署 ##
Spring Boot除了在配置文件中指定参数外，还可以在启动时指定参数。

1. 简单运行:**java -jar xxx.jar**
2. 指定端口号:**java -jar xxx.jar - -server.port=9090**
3. 指定激活的配置文件：application-prod.properties
	> java -jar xxx.jar - -spring.profiles.active=prod

5. Debug环境运行
	> java -jar xxx.jar –debug

## 外置配置文件 ##
Spring Boot加载 Application.properties配置文件优先级：
1. 当前目录下的 /config 目录
2. 当前目录
3. classpath 下的 /config 包目录
4. classpath 目录

**加载外部配置文**
> java -jar springboot-v1.0.jar --spring.config.location = classpath：/default.properties,classpath：/override.properties
> java -jar -Dspring.config.location=D:\springboot\dev.properties springboot-v1.0.jar

## Linux 后台运行jar ##
如果在 Linux 服务器上只用 **java -jar xxx.jar** 来过运行应用，当**SSH**终端关闭，程序也会退出运行；实际部署应用到 Linux 会让程序在后台运行，方法如下：
> $ nohup java -jar test.jar &
> nohup：不挂断地运行命令；&：在后台运行,这两个命令大我一起使用。
> 默认情况下日志输出被重定向到**nohup.out**文件中。
>
> $ nohu java -jar myapp.jar > myapp_log.txt &
> 输入信息重定向到文件,
>
> $ jobs
> 列出所有在后台执行作业的应用，每个作业前有个编号。
> $ fg 编号，如：fg 1
> 是将该编号的作用调回到前台控制,但关闭前台终端则应用退出


**参考:**
[1][Spring Boot加载配置文件](https://www.cnblogs.com/moonandstar08/p/7368292.html)
[2][Spring Boot配置文件放在jar外部](https://www.cnblogs.com/xiaoqi/p/6955288.html)
