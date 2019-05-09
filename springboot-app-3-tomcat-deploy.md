---
title: Spring Boot 2实践系列(三)：Tomcat 配置、部署、随机端口
comments: true
date: 2018-05-24 09:31:06
tags: [tomcat,jar,war,servlet]
categories: [Spring Boot 2实践系列]
---
　　**Spring Boot** Web应用默认内置了精简版的**Tomcat**服务器，可以直接执行**jar**来启动运行应用。如果需要将应用部署到外部**Tomcat**服务器就需要修改部份配置。

　　项目通过**Maven**来管理依赖。Spring Boot默认支持 `jar`包方式，并可直接通过 `jar`命令来运行项目应用。若需要将项目打成**war**部署在外部的**Tomcat**上运行，需要做些修改。
<!-- more -->
## 内嵌Servlet容器(Tomcat) ##
1. Servlet容器的配置属性在 org.springframework.boot.autoconfigure.web.ServerProperties 文件中。
	```
	server.port=8080 # Server HTTP port.
	server.connection-timeout= # 连接超时时长.
	server.servlet.path=/ # 配置访问路径，默认为 /.
	server.max-http-header-size=0 # http消头最大字节数, 单位：bytes
	server.server-header= # 设置响应头的值，为空则不发送
	server.use-forward-headers= # Whether X-Forwarded-* headers should be applied to the HttpRequest.
	server.servlet.context-parameters.*= # Servlet context init parameters.
	server.servlet.context-path= # Context path of the application.
	server.servlet.application-display-name=application # Display name of the application.
	
	server.tomcat.accept-count=0 # 排队中的请求最大连接数
	server.tomcat.max-connections=0 # 接受和处理的最大连接数
	server.tomcat.max-threads=0 # 最大线程数
	server.tomcat.uri-encoding=UTF-8 # 编码
	server.compression.enabled=fals # 是否开启响应数据压缩
	server.tomcat.resource.cache-ttl＝＃ 静态资源有效时长
	```
	默认端口是 **8080**，可以使用 **server.port** 设置(例如, 在 application.properties 中作为 System 属性)。关闭端口使用 **server.port=-1**,对测试有用。

2. 在代码中配置 Tomcat:
	``` java
	import org.springframework.boot.web.server.WebServerFactoryCustomizer;
	import org.springframework.boot.web.servlet.server.ConfigurableServletWebServerFactory;
	import org.springframework.stereotype.Component;
	
	@Component
	public class CustomizationBean implements WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> {
	
		@Override
		public void customize(ConfigurableServletWebServerFactory server) {
			server.setPort(9000);
		}
	}
	```
3. 自己注册 TomcatServletWebServerFactory, JettyServletWebServerFactory, UndertowServletWebServerFactory 
	``` java
	@Bean
	public ConfigurableServletWebServerFactory webServerFactory() {
		TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
		factory.setPort(9000);
		factory.setSessionTimeout(10, TimeUnit.MINUTES);
		factory.addErrorPages(new ErrorPage(HttpStatus.NOT_FOUND, "/notfound.html"));
		return factory;
	}
	```
4. 使用随机端口
	当在一台服务器上部署多个实例应用时，需要配置不同的端口，可以配置随机端口，也可以自定义检查端口冲突使用未被占用的端口。
	生成随机数来作为端口,如下例
	> //扫描空闲端口，防止端口冲突
	> server.port=0
	> 
	> 或者 server.port=${random.int[2000,20000]}
	> 随机数生成方式可能存在端口冲突，可以写个启动类，配置系统变量。

	



## 部署到外部Tomcat ##
### 修改打包方式 ###
在`pom.xml`文件里修改打包方式为`war`包。
> 默认是打 **jar** 包
> `<packaging>jar</packaging>`
> 改为打 **war** 包
> `<packaging>war</packaging>`

### 排除内置Tomcat ### 
1. 方式一
	Spring Boot 内嵌的`Tomcat`是被集成在`spring-boot-starter-web`包里，因使用外部**Tomcat**,需要将内嵌的**Tomcat**依赖排除掉(发布环境)。
	``` xml
	<dependency>
	    <groupId>org.springframework.boot</groupId>
	    <artifactId>spring-boot-starter-web</artifactId>
	    <exclusions>
	        <exclusion>
	            <groupId>org.springframework.boot</groupId>
	            <artifactId>spring-boot-starter-tomcat</artifactId>
	        </exclusion>
	    </exclusions>
	</dependency>
	```
	此方式需要添加`javax.servlet-api`依赖, Application 继承 SpringBootServletInitializer 依赖于 servlet。
	``` xml
	<dependency>
	    <groupId>javax.servlet</groupId>
	    <artifactId>javax.servlet-api</artifactId>
	    <version>3.1.0</version>
	    <scope>provided</scope>
	</dependency>
	```
2. 方式二
	添加**spring-boot-starter-tomcat**依赖，覆盖内嵌的**Tomcat**, 设置作用范围是`provided`(编译和测试)。
	``` java
	<dependency>
	    <groupId>org.springframework.boot</groupId>
	    <artifactId>spring-boot-starter-tomcat</artifactId>
	    <scope>provided</scope>
	</dependency>
	```
### 入口类继承Servlet初始化类 ###
如果 Spring Boot 应用打 **war** 包，部署在外部 Servlet 容器(Tomcat,jetty等)，Application 入口类可以继承`SpringBootServletInitializer`类并覆盖配置方法(**configure()**), 这样就可以利用 Spring Framework Servlet 3.0的支持，在程序由 servlet容器启动时进行自定义配置，如关闭 Whitelabel Error Page 页面。
``` java
@SpringBootApplication
public class Application extends SpringBootServletInitializer {

	@Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        //关闭默认的Whitelabel Error Page
        setRegisterErrorPageFilter(false);
        return application.sources(Application.class);
    }

	public static void main(String[] args) throws Exception {
		SpringApplication.run(Application.class, args);
	}
}
```
## 创建war包工程 ##
在使用 **Maven** 构建项目时，选择 `war`包方式的工程。

## 排除缺少web.xml报错 ##
如果是**SSM**框架手动修改依赖包转为**Spring Boot**，并且没有**web.xml**文件，需要在 Maven 插件管理里，添加关闭`web.xml`的检查报错。
``` xml
<!-- 打war包，排除web.xml,使用java config -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-war-plugin</artifactId>
    <version>2.6</version>
    <configuration>
        <failOnMissingWebXml>false</failOnMissingWebXml>
    </configuration>
</plugin>
```

## 自定义实现随机端口 ##
1. 启动配置,设置随机端口
``` java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.util.StringUtils;

public class StartConfig {

    private static Logger logger = LoggerFactory.getLogger(StartConfig.class);

    public StartConfig(String[] args) {
        Boolean isServerPort = false;
        String serverPort = "";
        if(args != null){
            for (String arg : args) {
                if(StringUtils.hasText(arg) && arg.startsWith("--server.port")){
                    isServerPort = true;
                    serverPort = arg;
                    break;
                }
            }
        }
        if(!isServerPort){
            int port = ServerPortUtil.getAvailablePort();
            logger.info("current server.port=" + port);
            System.setProperty("server.port",String.valueOf(port));
        }else {
            logger.info("current server.port=" + serverPort.split("=")[1]);
            System.setProperty("server.port", serverPort.split("=")[1]);
        }

    }
}
```
2. 服务端口工具类，生成随机数端口
``` java
import java.util.Random;

public class ServerPortUtil {
	
    public static int getAvailablePort() {
        int max = 65535;
        int min = 2000;
        Random random = new Random();
        int nextInt = random.nextInt(max);
        int port = nextInt % (max - min + 1) + min;
        boolean using = NetUtils.isLoclePortUsing(port);
        if (using) {
            return getAvailablePort();
        } else {
            return port;
        }
    }
}
```
3. 网络工具类，检查本地端口是否可用
``` java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.net.InetAddress;
import java.net.Socket;

public class NetUtils {

    private static Logger logger = LoggerFactory.getLogger(NetUtils.class);

    public static boolean isLoclePortUsing(int port) {
        boolean flag = true;
        try {
            flag = isPortUsing("127.0.0.1", port);
        } catch (Exception e) {
            logger.error("error:", e.getMessage());
        }
        return flag;
    }

    public static boolean isPortUsing(String host, int port) {
        boolean flag = false;
        try {
            InetAddress theAddress = InetAddress.getByName(host);
            // 如果该本地端口创建 Socket 成功,说明该端口已被使用; 而实际需要未被使用的端口,给未被使用的端口创建 Socket 会报连接被拒绝的错误
            Socket socket = new Socket(theAddress, port);
            flag = true;
        } catch (Exception e) {
            logger.error("error:", e.getMessage());
        }
        return flag;
    }
}
```
4. 启动类加载启动配置
``` java
@SpringBootApplication
public class Application {
    private static Logger logger = LoggerFactory.getLogger(Application.class);

    public static void main(String[] args) {
        logger.info("args:", JSON.toJSONString(args));
        new StartConfig(args);
        SpringApplication.run(Application.class, args);
    }
}
```

## 相关参考 ##
1. [Spring Boot Reference Guide 2.1.2 ：the HTTP Port](https://docs.spring.io/spring-boot/docs/2.1.2.RELEASE/reference/htmlsingle/#howto-change-the-http-port)
