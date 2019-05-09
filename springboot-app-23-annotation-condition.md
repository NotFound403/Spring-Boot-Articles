---
title: Spring Boot 2实践系列(二十三)：自动配置之@Conditional条件注解
comments: true
date: 2018-07-10 09:40:48
tags: [spring,condition,conditional,条件注解]
categories: [Spring Boot 2实践系列]
---
　　要理解 Spring Boot 的自动配置，就必须先理解 Spring 的 `@Conditional`注解，在自动配置类中常看到该注解的使用。

　　该注解指定了在什么条件下创建 Bean 进行配置。 Spring Boot包含多个 `@Conditional` 注释，可以在`@Configuration`注解的类和`@Bean`注解方法中使用。

　　`@Conditional`类型的注解，可以注解在类上，可以注解在`Bean`方法上，可以允许基于Spring Environment属性包含配置，可以仅允许在存在特定资源时包含配置。

　　[官方文档 -> Spring Boot自动配置之条件注解](https://docs.spring.io/spring-boot/docs/2.0.3.RELEASE/reference/htmlsingle/#boot-features-condition-annotations)

　　也可自定义，通过实现`Condition`接口，并重写其`matches`方法来构造判断条件。
<!-- more -->
## @Conditional ##
###  Class Conditions ###
`@ConditionalOnClass`和`@ConditionalOnMissingClass` 两个在类上的注解：
判断指定的类是否存在来构建自动配置，也可使用`name`属性名来指定类名。

### Bean Conditions ###
`@ConditionalOnBean`和`@ConditionalOnMissingBean` 两个在 **Bean** 方法上的注解：
判断指定的 Bean 是否存在来构建自动配置，可以使用`value`属性来按类型或名称(id)指定 **Bean**, 可以使用`search`属性指定 **ApplicationContext** 层次结构来搜索`Bean`。
``` java
@Configuration
public class MyAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean
	public MyService myService() { ... }

}
```
要添加注意添加 Bean 时的顺序，官方建议在自动配置类上仅使用 `@ConditionalOnBean`和`@ConditionalOnMissingBean`注释，因为可以保证这些注释在添加用户定义的Bean后执行。
`@ConditionalOnBean`和`@ConditionalOnMissingBean`注解作用在`@Configuration`注释的类上，等同于在作用在每个包含`@Bean`的方法上。

### Property Conditions ###
`@ConditionalOnProperty`注解可以基于Spring Environment属性包含的配置进判断，再决定自动配置的执行，使用 `prefix` 和 `name` 属性指定应检查的属性。默认情况下，匹配**存在**且不等于**false**的任何属性。 您还可以使用`havingValue`和`matchIfMissing`属性创建更高级的检查。

### Resource Conditions ###
`@ConditionalOnResource`注释仅允许在存在特定资源时执行自动配置。 可以使用常用的 Spring 约定来指定资源，如以下示例所示：file：/home/user/test.dat。

### Web Application Conditions ###
`@ConditionalOnWebApplication`和`@ConditionalOnNotWebApplication`注解用于判断应用程序是否为`Web应用程序`，Web应用程序是使用**Spring WebApplicationContext**，定义会话范围或具有**StandardServletEnvironment**的任何应用程序。

### SpEL Expression Conditions ###
`@ConditionalOnExpression`注解允许根据**SpEL**表达式的结果来执行配置。

## 示例 ##
条件：根据操作系统返回对应的内容。
1. 定义判断条件
``` java
//Windows系统的判断条件
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.type.AnnotatedTypeMetadata;

public class WindowsCondition implements Condition {

	@Override
	public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
		return context.getEnvironment().getProperty("os.name").contains("Windows");
	}
}


//Linux系统的判断条件
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.type.AnnotatedTypeMetadata;

public class LinuxCondition implements Condition {

	@Override
	public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
		return context.getEnvironment().getProperty("os.name").contains("Linux");
	}
}

```
2. 不同系统下的Bean类
``` java
//接口
public interface ListService {
	
	public String showListCmd();
}

//Windows下的实现类
public class WindowsListServiceImpl implements ListService {

	@Override
	public String showListCmd() {
		return "dir";
	}
}

//Linux下的实现类
public class LinuxListServiceImpl implements ListService {

	@Override
	public String showListCmd() {
		return "ls";
	}
}
```
3. 配置类
**@Conditional()**注解调用条件判断的类并根据返回的结果来创建`Bean`
``` java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Conditional;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ConditionConfig {
	
	@Bean
	@Conditional(WindowsCondition.class)
	public ListService windowsListService() {
		return new WindowsListServiceImpl();
	}
	
	@Bean
	@Conditional(LinuxCondition.class)
	public ListService linuxListService() {
		return new LinuxListServiceImpl();
	}
}
```
4. 运行类
``` java
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class ConditionMain {
	
	public static void main(String[] args) {
		AnnotationConfigApplicationContext context = 
				new AnnotationConfigApplicationContext(ConditionConfig.class);
		
		ListService listService = context.getBean(ListService.class);
		System.out.println(context.getEnvironment().getProperty("os.name")
				+ "系统下的列表命令："
				+ listService.showListCmd());
		
		//context.close();
	}
}
```