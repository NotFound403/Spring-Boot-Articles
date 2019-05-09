---
title: Spring Boot 2实践系列(二十)： 配置文件加载及参数绑定
comments: true
date: 2018-06-25 15:00:46
tags: [spring boot,properties,参数绑定,yml]
categories: [Spring Boot 2实践系列]
---
　　**Spring Boot**提倡的是零配置，提供了强大的自动配置功能；只要装配了相应的功能，默认就会启用通用的配置(**习惯优于配置**)。

　　但实际开发中，除了默认的配置外，根据需求难免需要定制配置文件；**SpringBoot**默认的配置文件`application.properties`会被自动加载，但定制的配置文件则需要我们自己手动加载。

　　**SpringBoot**加载配置文件是将文件内容读取到**Spring**容器中，目的是为了给**Spring**容器中的上下文对象调用，达到一次加载，随处调用。
<!-- more -->
## SpringBoot配置文件 ##
　　Spring Boot可以自动识别`yml 和 properties`两种文件，都是**键值对(key/value)**形式的配置；`properties`可直接使用注解`@PropertySource`加载自定义的文件；而`yml`文件需要从代码层面加载，不如前者方便，但`yml`是**面向对象**的一种格式，更易于理解。

　　默认的配置文件`application.properties`随项目的(**Maven**)创建而存在在`src/main/resources`目录中，会被自动加载。而我们自定义的配置文件，一般也放在些目录下，或在此目录建立子目录存放。

## 配置文件加载 ##
### 公共配置文件 ###
`application.properties`文件也称为公共配置文件，在**SpringBoot**启动时被自动加载，优先级最高。
``` properties
server.port=80
server.servlet.context-path=/

book.name=spring boot
```

### 定制配置文件 ###
定制配置文件`xxx.properties`文件，使用`@PropertySource(value = "classpath:xxx.properties", encoding = "UTF-8")`注解加载,作用在类上,基于一次加载，到处使用原则，因一份配置文件可能有多种类型参数，或有多个配置文件，一般会建一个配置类，将加载配置文件的注解统一放在该配置类上。`value`属性是个字符串数组，可以引入多个配置文件。
1. mysql.properties
	```
	#mysql.properties
	mysql.url=http://localhost:3306
	mysql.username=root
	mysql.password=123456
	```
2. PropertiesConfig.java
	``` java
	import org.springframework.context.annotation.Configuration;
	import org.springframework.context.annotation.PropertySource;
	
	@Configuration
	@PropertySource(value = {"classpath:mysql.properties"}, encoding = "utf-8")
	public class PropertiesConfig {
	
	}
	```
**备注：**默认编码是`ISO 8859-1`，故在默认的配置文件中使用中文时，会出现乱码，可以将中文转成**Unicode**编码或者使用`yml`配置格式(默认就支持utf-8), 或在配置**java**类上利用`@PropertySource`注解的 `encoding` 属性指定编码。

## SpringBoot参数绑定 ##
SpringBoot的参数绑定支持两种方式：
1. 使用`@Value("${key}")`注解绑定每一项属性，每一项属性都要使用该注解。
2. 使用`@ConfigurationProperties(prefix="xxx")`注解，匹配前辍，配置文件参数后辍与**JavaBean**的属性名一致，可实现批量自动绑定。

### @Value绑定到属性 ###
`@Value`注解是将每一项参数绑定到属性上。
``` 
@Value("${book.name}")
private String bookName;
```

### @ConfigurationProperties绑定JavaBean ###
`@ConfigurationProperties(prefix="xxx")`注解作用在实体为上，将实体类注册到**Spring**容器，需要用到该实体类时，通过`@Autowired`注入到调用的类里。
1. 实体类
	在类上添加`@Component`注解，将该类注册到到**Spring**容器，添加`@PropertySource`注解加载定制配置文件。
	``` java
	@Component
	@PropertySource(value = {"classpath:mysql.properties"})
	@ConfigurationProperties(prefix="mysql")
	public class MysqlPro {
		
		private String url;
		private String account;
		private String password;
	
		//---set/get方法省略---
	}
	```
2. 定制配置文件：`src/main/resources/mysql.properties`
3. 在调用实体类的业务类里面注入实体类`Bean`。
	``` java
	@Autowired
	private MysqlPro mysqlPro;
	```

### 数组参数绑定 ###
Spring Boot配置文件还支持数组类型的参数绑定，如下配置：
``` 
user.hobby[0]=看电影
user.hobby[1]=旅游
```
可以使用 `List<String>` 获取取**hobby**的值。


## @EnableConfigurationProperties ##
**@ConfigurationProperties**：作用在属性配置类上，可以通过`prefix`指定前缀。
**@EnableConfigurationProperties**：表示指定的类(JavaBean)注册为 Spring 容器的 Bean，并注入到本类 。

当调用类通过**@EnableConfigurationProperties**来注入**@ConfigurationProperties**注解的属性类时，属性类可以省略`@Component`或`@Configuration`注解；调用类可以省略注入注解`@Autowired`，属性类作为调用方的一个属性参数来使用。

查看Spring Boot的自动配置源码，此方式使用也是比较多的。

1. 属性类,通过前缀注入值
``` java
//省略@Component注解
@ConfigurationProperties(prefix = "rest.server")
public class RestServerProperties {

    private String url;

    public String getUrl() {
        return url;
    }

    public RestServerProperties setUrl(String url) {
        this.url = url;
        return this;
    }
}
```
2. 调用方
``` java
@Service
@EnableConfigurationProperties({RestServerProperties.class})
public class CallRemoteServiceImpl implements CallRemoteService {

    private static final Logger logger = LogManager.getLogger(CallRemoteServiceImpl.class);

	//省略@Autowired注解
    private RestServerProperties restServerProperties;

	//------方法--------------

}
```

调用使用`@ConfigurationProperties`注解的`JavaBean`时，在调用类上使用`@EnableConfigurationProperties({xxxxProperties.class})`注解指定该`JavaBean`需要注册为**Spring**容器的`Bean`，即在调用前声明注册为`Bean`,这样在`JavaBean`上就不需要使用`@Component`或`@Configuration`注解。

## 生成随机数 ##
Spring Boot的属性配置文件中可以通过`${random}`来产生 `int` 值、 `long` 值或者 `string` 字符串，来支持属性的随机值。
``` 
//随机字符串
blog.value=${random.value}
//随机int
blog.number=${random.int}
//随机long
blog.bignumber=${random.long}
//10以内的随机数
blog.test1=${random.int(10)}
//1-20的随机数
blog.test2=${random.int[1,20]}
```
