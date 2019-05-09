---
title: Spring Boot 2实践系列(十五)：开发部署和热部署Spring Boot应用
comments: true
date: 2018-06-13 09:10:33
tags: [linux,deploy,war,jar]
categories: [Spring Boot 2实践系列]
---
　　此篇是作为[Spring Boot实践系列(三)：打war包部署到外部Tomcat](http://112.74.59.39/2018/05/24/springboot-app-3-outside-tomcat/)和[Spring Boot实践系列(十四)：配置文件profile属性和部署jar包](http://112.74.59.39/2018/06/10/springboot-app-14-profile-jar-deploy/)的延续。
<!-- more -->
## 热部署 ##
Spring Boot 支持热部署，官方提供了热部署的模块`spring-boot-devtools`，如需在项目`pom.xml`文件添加该依赖。
``` xml
<!-- 热部署 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional>
</dependency>
```
热部署在开发环境非常的方便高效，不用手动编译重启。而在生产中大多是直接更新整个`jar`包的情况比较多,可以在热部署依赖配置添加作用范围`<scope>provided</scope>`

`spring-boot-devtools`是监视类路径下的文件发生更改时就重启应用，某些资源文件(如静态资源文件和视图模板)的更改是不会重启应用的。

还有一些其它热部署的组件，如 Spring Loaded, JRebel(收费)。

## 模板缓存 ##
Spring Boot 支持的一些库使用了缓存来提高性能，例如，模板引擎缓存已编译的模板以避免重复解析模板文件。缓存静态资源在生产中非常有用，但在开发中会造成不便，在修改静态资源文件后得不到想要的更新后的内容。而`spring-boot-devtools`默认是禁用缓存的，所以推荐使用`devtools`来做开发环境的热部署,就不用针对单项功能去控制缓存的启用/禁用。

当然 Spring Boot 也支持缓存选项的设置，通常在`application.properties`文件中进行配置，也可在开发配置文件和生产配置文件分别配置，如下：
``` properties
# Thymeleaf Templates 
spring.thymeleaf.cache = false/true

# FreeMarker Templates 
spring.freemarker.cache = false/true

# Groovy Templates
spring.groovy.template.cache = false/true
```

## 作为Linux 服务运行 ##
Spring Boot应用除了使用**java -jar xxx.jar**运行外，还可以创建在**Linux**环境下可执行的应用，可以像**Linux**环境的**.sh**执行文件来运行，也可通过`init.d`或`systemd`来注册为 Linux 服务。这样可以使 Spring Boot 应用在生产环境中来安装和管理变得非常简单。

以下操作在 Ubuntu 16.04.4版本上操作

### 创建可执行应用 ###
Maven 管理的项目，在`pom.xml`文件中的**spring-boot-maven-plugin**插件下添加开启可执行文件的配置。
1. 打包成可执行文件
``` xml
<plugin>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-maven-plugin</artifactId>
	<configuration>
		<executable>true</executable>
	</configuration>
</plugin>
```
2. 通过`rz`或`sftp`将`jar`包上传到 Linux 服务器。
3. 修改包的`jar`包的权限
> chmod 777 myapp.jar
这个`jar`就变成了 Linux 环境下的一个可执行文件了。
4. 以可执行文件的方式运行 **jar**包
> ./myapp.jar

### init.d方式 ###
1. 创建软链接
	> ln -s /usr/local/java-app/myapp.jar /etc/init.d/myapp
	
	官文档指示操作：https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#deployment-install

2. 给服务授权为可执行
	> /etc/init.d# chmod +x myapp

3. 查看服务可用命令
	> /etc/init.d# ./myapp
	> Usage: ./myapp {start|stop|force-stop|restart|force-reload|status|run}

4. 可执行命令
	> /etc/init.d# ./myapp start
	> /etc/init.d# ./myapp stop
	> /etc/init.d# ./myapp force-stop
	> /etc/init.d# ./myapp restart
	> /etc/init.d# ./myapp force-reload
	> /etc/init.d# ./myapp status
	> /etc/init.d# ./myapp run
	
	以上操作还不算是将 `jar` 注册为服务来启动.
	服务启动命令应该是：service service_name [start|stop|reload|status]，或者是:systemctl [tart|stop|reload|status] service_name.service 
5. 以service方式运行
	> service myapp start
	> 提示：Failed to start myapp.service: Unit myapp.service not found.

6. 将服务注册为系统默认服务
	> update-rc.d myapp defaults 
	> 再次启动服务
	> service myapp start
	> 提示：Job for myapp.service failed because the control process exited with error code. See "systemctl status myapp.service" and "journalctl -xe" for details.

7. 根据提示查找问题
	> systemctl status myapp.service
	> 提示：Loaded: loaded (/etc/init.d/myqpp; bad; vendor preset: enabled)
	> Active: failed (Result: exit-code) since Wed 2018-06-13 18:21:42 CST; 51s ago

8. 继承查找
	> journalctl -xe
	> 提示：Unable to find Java
	
	提示是找不到 java，提示内容是服务文件输出的，服务文件中使用的是`$JAVA_HOME`来查找 java。实际已经配置了 Java 环境，在**/etc/profile** 文件配置了 export JAVA_HOME=/usr/local/java/jdk1.8.0_172 ，问题出在服务文件中的`$JAVA_HOME`定位不到**jdk**目录，把文件中的**$JAVA_HOME**用绝对路径显式指定，如下：
	```
	# Find Java
	if [[ -n "/usr/local/java/jdk1.8.0_172" ]] && [[ -x "/usr/local/java/jdk1.8.0_172/bin/java" ]]; then
	    javaexe="/usr/local/java/jdk1.8.0_172/bin/java"
	elif type -p java > /dev/null 2>&1; then
	    javaexe=$(type -p java)
	elif [[ -x "/usr/bin/java" ]];  then
	    javaexe="/usr/bin/java"
	else
	    echo "Unable to find Java"
	    exit 1
	```
9. 以服务方式启动
	> service myapp start                     	 //启动成功
	> service myapp status                     	 //状态：活动
	> systemctl status myapp.service             //状态：活动
	
	目前来看以服务方式启动是成功了，现在访问下服务
10. 访问服务
http://ip:port/mapping_path                 //无响应，访问失败。
使用 `ps -ef|grep java` 没有看到Java进程，不知具体原因，问题仍未解决。

11. CentOS 7环境验证
在 CentOS 7 环境下，截止到第 **5** 步的操作可正常部署运行可访问，这也是 Spring Boot 官方文档支持的方式。在 Ubuntu 16.04.4版本上就比较诡异了，其它版本是否有差异会造成异常还不确定。
官方文档：https://docs.spring.io/spring-boot/docs/2.0.2.RELEASE/reference/htmlsingle/

### systemd方式 ###
1. Spring Boot 项目 jar 包是安装在** /var/myapp** 目录下(安装目录根据自己需要而选择)
2. 在 **/etc/systemd/system** 目录下创建`myapp.service`脚本文件
> /etc/systemd/system# touch myqpp.service

3. 编辑脚本文件内容如下：
	``` java
	[Unit]
	Description=myapp
	After=syslog.target
	
	[Service]
	#如果没有添加指定用户权限,注释User限制
	#User=myapp
	#jar文件绝对路径
	ExecStart=/var/myapp/myapp.jar
	SuccessExitStatus=143
	
	[Install]
	WantedBy=multi-user.target
	```
	`ExecStart=/var/myapp/myapp.jar`此设置可能会提示：Unable to find Java; 也是找不到 `java` 执行命令；
	改为`java`命令的绝对路径：	`ExecStart=/usr/local/java/jdk1.8.0_172/bin/java -jar /var/myapp/myapp.jar`
4. 启用服务
	> systemctl enable myapp.service
5. 查看服务状态
	> systemctl status myapp.service
	绿色 `active(running)`表示启动成功，活动的状态。
6. 浏览器访问服务地址
	返回成功表示此方式部署的服务成功。
7. systemctl常用命令
	> systemctl start myapp.service         //启动
	> systemctl restart myapp.service       //重启
	> systemctl status myapp.service        //查看状态
	> systemctl stop myapp.service          //停止

**问题：**`init.d`和`systemd`两种方式部署的应用，启动时如何指定配置文件是个问题(像java命令指定配置文件，如：java -jar xxx.jar - -spring.profiles.active=prod)，现还没找到方法。

