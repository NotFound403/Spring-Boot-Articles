---
title: Spring Boot 2实践系列(九)：Transaction(事务)的支持
comments: true
date: 2018-05-30 23:20:29
tags: [transaction]
categories: [Spring Boot 2实践系列]
---
　　要理解 Spring Boot 的事务，必须先理解 Spring 的事务机制，可以参考[Spring事务机制](http://112.74.59.39/2018/05/30/spring-transaction/)。
<!-- more -->
## 自动配置的事务管理器 ##
### JDBC访问 ###
Spring Boot 已经定义了实现 **PlatformTransactionManager**接口的 **DataSourceTransactionManager** 的 **Bean**, 配置在**org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration**类文件中。
DataSourceTransactionManager 继承自抽象类 AbstractPlatformTransactionManager，抽象类实现了 PlatformTransactionManager 这个统一的接口。
``` java
@Bean
@ConditionalOnMissingBean(PlatformTransactionManager.class)
public DataSourceTransactionManager transactionManager(
		DataSourceProperties properties) {
	DataSourceTransactionManager transactionManager = new DataSourceTransactionManager(
			this.dataSource);
	if (this.transactionManagerCustomizers != null) {
		this.transactionManagerCustomizers.customize(transactionManager);
	}
	return transactionManager;
}
```

### JPA访问 ### 
Spring Boot Jpa 是基于 Hibernate 实现的，在org.springframework.boot.autoconfigure.orm.jpa.JpaBaseConfiguration类文件里定义了 PlatformTransactionManager 接口的实现 JpaTransactionManager 的 Bean 。
``` java
@Bean
@ConditionalOnMissingBean
public PlatformTransactionManager transactionManager() {
	JpaTransactionManager transactionManager = new JpaTransactionManager();
	if (this.transactionManagerCustomizers != null) {
		this.transactionManagerCustomizers.customize(transactionManager);
	}
	return transactionManager;
}
```
在自动配置类 org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration 引入了 HibernateJpaConfiguration，自动配置了 `JPA`的事务。

### Transactional自动配置 ###
org.springframework.boot.autoconfigure.transaction.TransactionAutoConfiguration事务自动配置依赖于 JtaAutoConfiguration,HibernateJpaAutoConfiguration,DataSourceTransactionManagerAutoConfiguration自动配置的事务,还开启了声明式事务的支持。
``` java
@Configuration
@ConditionalOnClass(PlatformTransactionManager.class)
@AutoConfigureAfter({ JtaAutoConfiguration.class, HibernateJpaAutoConfiguration.class,
		DataSourceTransactionManagerAutoConfiguration.class,
		Neo4jDataAutoConfiguration.class })
@EnableConfigurationProperties(TransactionProperties.class)
public class TransactionAutoConfiguration {

	...................

	//开启了声名式事务的支持
	@Configuration
	@ConditionalOnBean(PlatformTransactionManager.class)
	@ConditionalOnMissingBean(AbstractTransactionManagementConfiguration.class)
	public static class EnableTransactionManagementConfiguration {

		@Configuration
		@EnableTransactionManagement(proxyTargetClass = false)
		@ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false", matchIfMissing = false)
		public static class JdkDynamicAutoProxyConfiguration {

		}

		@Configuration
		@EnableTransactionManagement(proxyTargetClass = true)
		@ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true", matchIfMissing = true)
		public static class CglibAutoProxyConfiguration {

		}
	}
}
```

## 事务使用 ##
在 Spring Boot中，无须显式使用`@EnableTransactionManagement`注解来开启事务支持，Spring Boot 通过自动配置，默认就提供了事务的支持，在实际开发中，只需在业务层的类或方法上添加`@Transactional`注解即可启用事务(相当于XML中的`<tx:annotation-driven/>`配置方式)。

**事务**一般是在一条`sqlSession`里执行多表或多条SQL操作时需要用到。在程序执行遇到**运行时异常**并**抛出异常**时才会触发事务，在实际开发中抛出异常还得在`Controller`捕获异常并做最终的处理。

开发时，一般在业务层使用`try...catch....`捕获异常，在`catch`里面抛出异常来触发事务，在控制层同样使用`try...catch....`接收异常，在`catch`里处理异常并转换为用户可识别的异常消息。

### 业务层 ###
1. 业务层
方法上添加**@Transactional**注解启用事务,捕获并抛出异常来触发事务
``` java
@Service
public class UserServerImpl{

	@Autowired
	private UserMapper userMapper;

	@Override
    @Transactional
    public void addUser(User user) {
        try {
            userMapper.insertUser(user);
			userMapper.insertUserAddress(user.getAddress());
           
        } catch (Exception e) {
            throw e;
        }
    }
}
```
2. 控制层
捕获业务层抛出的异常,在**catch**里处理异常转换为可识别的异常消息。
``` java
@RestController
@RequestMapping(value = "/user")
public class UserController {
	
	@Autowired
    private UserService userService;

	@RequestMapping("/saveUser")
    public RespResult saveUser(User user) {
        RespResult respResult = new RespResult();
        try {   
				userService.saveUser(user);
                respResult.setState(true).setCodeAndMsg(ResultCodeAndMsg.SUCCESS);
        } catch (Exception e) {
            respResult.setState(false).setCode(4001).setMsg("保存用户失败");
        }
        return respResult;
    }
}
```

## 分布式事务框架 ##
在此备注，后续研究
1. spring-boot-starter-jta-atomikos
2. spring-boot-stater-jta-bitronix
