---
title: Spring Boot 2实践系列(四十)：源码分析启动类上 @SpringBootApplication 注解
comments: true
date: 2019-04-27 08:23:57
tags: [main,Application,SpringBootApplication]
categories: [Spring Boot 2实践系列]
---
　　Spring Boot 都会有一个名为 xxxApplication 的启动类，里面有一个标准的 *java* 应用的入口 *main* 方法，用于启动 Spring Boot 应用项目。

　　*@SpringBootApplication* 是 Spring Boot 的核心注解，作用在 xxxApplication 的启动类上，*SpringBoot* 会自动扫描 *@SpringBootApplication* 所在类的同级包及下级包里的所有 Bean 。

　　通过 [Spring Initializr](https://start.spring.io/) 或 IDE 支持创建的 Spring Boot 应用的在 groupId + arctifactID 组合的包名下会创建一个 xxxApplication 启动类。
<!-- more -->

## 注解源码 ##

1. @SpringBootApplication 注解源码	
	``` java
	@Target(ElementType.TYPE)
	@Retention(RetentionPolicy.RUNTIME)
	@Documented
	@Inherited
	@SpringBootConfiguration
	@EnableAutoConfiguration
	@ComponentScan(excludeFilters = {
			@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
			@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
	public @interface SpringBootApplication {
	
		@AliasFor(annotation = EnableAutoConfiguration.class)
		Class<?>[] exclude() default {};
	
		@AliasFor(annotation = EnableAutoConfiguration.class)
		String[] excludeName() default {};
	
		@AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
		String[] scanBasePackages() default {};
	
		@AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
		Class<?>[] scanBasePackageClasses() default {};
	}
	
	```
	
2. @SpringBootConfiguration 注解源码
   ``` java
   @Target(ElementType.TYPE)
   @Retention(RetentionPolicy.RUNTIME)
   @Documented
   @Configuration
   public @interface SpringBootConfiguration {
   }
   ```
   该注解标识这是一个 Spring Boot 项目。

@SpringBootApplication 是一个复合注解，其定义使用了多个元注解，但重要的只有三个 **@Configuration**、**@EnableAutoConfiguration**、**@ComponentScan**。
## 注解分析

### @Configuration

标识当前类是个 IoC 容器的配置类，应用启动时创建为 Bean 存放到 IoC 容器，在这里起引导启动作用。

### @EnableAutoConfiguration

``` java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
    
	Class<?>[] exclude() default {};

	String[] excludeName() default {};
}
```

这也是个复合注解，该注解可以帮助 Spring Boot 应用将所有符合条件的 @Configuration 配置都加载到当前 Spring Boot 创建并使用的 IoC 容器中。

该注解的的核心要属于 *@Import(AutoConfigurationImportSelector.class)* ，AutoConfigurationImportSelector 是自动配置选择器。

``` java
public class AutoConfigurationImportSelector
		implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware,
		BeanFactoryAware, EnvironmentAware, Ordered {

	private static final AutoConfigurationEntry EMPTY_ENTRY = new AutoConfigurationEntry();

	private static final String[] NO_IMPORTS = {};

	private static final Log logger = LogFactory
			.getLog(AutoConfigurationImportSelector.class);

	private static final String PROPERTY_NAME_AUTOCONFIGURE_EXCLUDE = "spring.autoconfigure.exclude";

	private ConfigurableListableBeanFactory beanFactory;
	// 环境
	private Environment environment;
	// 类加载
	private ClassLoader beanClassLoader;
	// 资源加载
	private ResourceLoader resourceLoader;

	//----省略方法--------
}
```

该类实现了类加载器，资源加载器，BeanFactoryAware 和 EnvironmentAware，借助 SpringFactoriesLoader 工具类查找配置类的功能，实现了从 classpath 中搜寻所有 META-INF/spring.factories 配置文件，并将其中 org.springframework.boot.autoconfigure.EnableAutoConfiguration 对应的配置项通过反射实例化为标注了 @Configuration 的 IoC 容器配置类，然后汇总加载到 IoC 容器。

### @ComponentScan

该注解默认会扫描当前类所在包及所有下级包中使用了 @Component、@Repository、@Service、@Controller 注解的类并将其作为 Spring Bean 加载到 IoC 容器中。

也可手动单个注册每个 Bean。