---
title: Spring Boot 2实践系列(四十一)：源码分析自动配置实现及手写自动配置
comments: true
date: 2019-04-27 18:37:41
tags: [autoConfig]
categories: [Spring Boot 2实践系列]
---
　　**Spring Boot** 的一大特性是基于 **习惯优与配置** 原则为很多组件提供了 **自动配置** ，这个强大的特性可以快速将其它功能整合，个人认为这是促成该框架流行的主要原因，特别适合互联网项目的分布式开发(基于业务的单一职责原则)，开发可以将更多精力集中在业务上，而不是配置上。
<!-- more -->

　　**Spring Boot**自动配置的源码在`spring-boot-starter-2.0.0.RELEASE.jar`的依赖包`spring-boot-autoconfigure-2.0.0.RELEASE.jar`里；若需要了解**SpringBoot**为我们做了哪些自动配置，可以看这个包里的源码。
## 原理分析 ##
### autoconfigure-xx.jar ###
查看当前项目已启用和未用启的自动配置报告，在`application.properties`文件添加：`debug=true`,**Positive matches**是已启动的，**Negative matches**是未启动的。
![springboot-42-autoconfig-package.png](https://i.loli.net/2019/04/28/5cc5c05fdcce5.png)
### 自动配置 ###
　　**Spring Boot**自动配置功能分析需要从入口类的`@SpringBootApplication`注解开始，该注解是一个组合注解，组合了核心注解`@EnableAutoConfiguration`，自动配置功能是由`@EnableAutoConfiguration`注解提供的。

1. @EnableAutoConfiguration 注解，该注解是自动配置的入口。
	``` java
	@Target(ElementType.TYPE)
	@Retention(RetentionPolicy.RUNTIME)
	@Documented
	@Inherited
	@AutoConfigurationPackage
	@Import(AutoConfigurationImportSelector.class)
	public @interface EnableAutoConfiguration {
	
		String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
	
		Class<?>[] exclude() default {};
	
		String[] excludeName() default {};
	
	}
	```
	该注解导入了`AutoConfigurationImportSelector`类，该类可理解为**自动配置选择器**，是核心类。
2. AutoConfigurationImportSelector 自动配置选择器
	``` java
	public class AutoConfigurationImportSelector
			implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware,
			BeanFactoryAware, EnvironmentAware, Ordered {
	
		@Override
		public String[] selectImports(AnnotationMetadata annotationMetadata) {
			if (!isEnabled(annotationMetadata)) {
				return NO_IMPORTS;
			}
			try {
				AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
						.loadMetadata(this.beanClassLoader);
				AnnotationAttributes attributes = getAttributes(annotationMetadata);
				List<String> configurations = getCandidateConfigurations(annotationMetadata,
						attributes);//1
				configurations = removeDuplicates(configurations);
				configurations = sort(configurations, autoConfigurationMetadata);
				Set<String> exclusions = getExclusions(annotationMetadata, attributes);
				checkExcludedClasses(configurations, exclusions);
				configurations.removeAll(exclusions);
				configurations = filter(configurations, autoConfigurationMetadata);
				fireAutoConfigurationImportEvents(configurations, exclusions);
				return StringUtils.toStringArray(configurations);
			}
			catch (IOException ex) {
				throw new IllegalStateException(ex);
			}
		}
	
		protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
				AnnotationAttributes attributes) {
			List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
					getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());//2
			Assert.notEmpty(configurations,
					"No auto configuration classes found in META-INF/spring.factories. If you "
							+ "are using a custom packaging, make sure that file is correct.");
			return configurations;
		}
	}
	
	```
	通过`SpringFactoriesLoader.loadFactoryNames()`方法从类路径或系统路径下读取用于自动配置资源文件`META-INF/spring.factories`, 并将其中 org.springframework.boot.autoconfigure.EnableAutoConfiguration 对应的配置项通过反射实例化为对应的标注了 @Configuration 的 JavaConfig 形式的 IoC 容器配置类，并汇总为一个加载到 IoC 容器。
3. loadFactoryNames()方法, 当找到`spring.factories`文件后，`SpringFactoriesLoader`将查询配置文件命名的属性。
	``` java
	public abstract class SpringFactoriesLoader {
	 
		//包根路径下 META-INF/spring.factories 文件
		public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
	
		private static final Log logger = LogFactory.getLog(SpringFactoriesLoader.class);
	
		private static final Map<ClassLoader, MultiValueMap<String, String>> cache = new ConcurrentReferenceHashMap<>();
	
		public static <T> List<T> loadFactories(Class<T> factoryClass, @Nullable ClassLoader classLoader) {
			Assert.notNull(factoryClass, "'factoryClass' must not be null");
			ClassLoader classLoaderToUse = classLoader;
			if (classLoaderToUse == null) {
				classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
			}
			List<String> factoryNames = loadFactoryNames(factoryClass, classLoaderToUse);
			if (logger.isTraceEnabled()) {
				logger.trace("Loaded [" + factoryClass.getName() + "] names: " + factoryNames);
			}
			List<T> result = new ArrayList<>(factoryNames.size());
			for (String factoryName : factoryNames) {
				result.add(instantiateFactory(factoryName, factoryClass, classLoaderToUse));
			}
			AnnotationAwareOrderComparator.sort(result);
			return result;
		}
	
		public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
			String factoryClassName = factoryClass.getName();
			return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
		}
	
		private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
			MultiValueMap<String, String> result = cache.get(classLoader);
			if (result != null)
				return result;
			try {
				//加载文件
				Enumeration<URL> urls = (classLoader != null ?
						classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
						ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
				result = new LinkedMultiValueMap<>();
				while (urls.hasMoreElements()) {
					URL url = urls.nextElement();
					UrlResource resource = new UrlResource(url);
					Properties properties = PropertiesLoaderUtils.loadProperties(resource);
					for (Map.Entry<?, ?> entry : properties.entrySet()) {
						List<String> factoryClassNames = Arrays.asList(
								StringUtils.commaDelimitedListToStringArray((String) entry.getValue()));
						result.addAll((String) entry.getKey(), factoryClassNames);
					}
				}
				cache.put(classLoader, result);
				return result;
			}
			catch (IOException ex) {
				throw new IllegalArgumentException("Unable to load factories from location [" +
						FACTORIES_RESOURCE_LOCATION + "]", ex);
			}
		}
	}
	```
4. spring.factories
`spring.factories`文件声明了有哪些自动配置。
``` java
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer

# Auto Configuration Import Listeners
org.springframework.boot.autoconfigure.AutoConfigurationImportListener=\
org.springframework.boot.autoconfigure.condition.ConditionEvaluationReportAutoConfigurationImportListener

# Auto Configuration Import Filters
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
org.springframework.boot.autoconfigure.condition.OnClassCondition

# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.cloud.CloudAutoConfiguration,\
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,\
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration,\
org.springframework.boot.autoconfigure.dao.PersistenceExceptionTranslationAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraReactiveDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraReactiveRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseReactiveDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseReactiveRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.ldap.LdapDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.ldap.LdapRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoReactiveDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoReactiveRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.neo4j.Neo4jDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.neo4j.Neo4jRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.solr.SolrRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisReactiveAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.rest.RepositoryRestMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.data.web.SpringDataWebAutoConfiguration,\
org.springframework.boot.autoconfigure.elasticsearch.jest.JestAutoConfiguration,\
org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration,\
org.springframework.boot.autoconfigure.freemarker.FreeMarkerAutoConfiguration,\
org.springframework.boot.autoconfigure.gson.GsonAutoConfiguration,\
org.springframework.boot.autoconfigure.h2.H2ConsoleAutoConfiguration,\
org.springframework.boot.autoconfigure.hateoas.HypermediaAutoConfiguration,\
org.springframework.boot.autoconfigure.hazelcast.HazelcastAutoConfiguration,\
org.springframework.boot.autoconfigure.hazelcast.HazelcastJpaDependencyAutoConfiguration,\
org.springframework.boot.autoconfigure.http.HttpMessageConvertersAutoConfiguration,\
org.springframework.boot.autoconfigure.http.codec.CodecsAutoConfiguration,\
org.springframework.boot.autoconfigure.influx.InfluxDbAutoConfiguration,\
org.springframework.boot.autoconfigure.info.ProjectInfoAutoConfiguration,\
org.springframework.boot.autoconfigure.integration.IntegrationAutoConfiguration,\
org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.JdbcTemplateAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.JndiDataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.XADataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.JmsAutoConfiguration,\
org.springframework.boot.autoconfigure.jmx.JmxAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.JndiConnectionFactoryAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.activemq.ActiveMQAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.artemis.ArtemisAutoConfiguration,\
org.springframework.boot.autoconfigure.groovy.template.GroovyTemplateAutoConfiguration,\
org.springframework.boot.autoconfigure.jersey.JerseyAutoConfiguration,\
org.springframework.boot.autoconfigure.jooq.JooqAutoConfiguration,\
org.springframework.boot.autoconfigure.jsonb.JsonbAutoConfiguration,\
org.springframework.boot.autoconfigure.kafka.KafkaAutoConfiguration,\
org.springframework.boot.autoconfigure.ldap.embedded.EmbeddedLdapAutoConfiguration,\
org.springframework.boot.autoconfigure.ldap.LdapAutoConfiguration,\
org.springframework.boot.autoconfigure.liquibase.LiquibaseAutoConfiguration,\
org.springframework.boot.autoconfigure.mail.MailSenderAutoConfiguration,\
org.springframework.boot.autoconfigure.mail.MailSenderValidatorAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.embedded.EmbeddedMongoAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.MongoReactiveAutoConfiguration,\
org.springframework.boot.autoconfigure.mustache.MustacheAutoConfiguration,\
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration,\
org.springframework.boot.autoconfigure.quartz.QuartzAutoConfiguration,\
org.springframework.boot.autoconfigure.reactor.core.ReactorCoreAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.UserDetailsServiceAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.SecurityFilterAutoConfiguration,\
org.springframework.boot.autoconfigure.security.reactive.ReactiveSecurityAutoConfiguration,\
org.springframework.boot.autoconfigure.security.reactive.ReactiveUserDetailsServiceAutoConfiguration,\
org.springframework.boot.autoconfigure.sendgrid.SendGridAutoConfiguration,\
org.springframework.boot.autoconfigure.session.SessionAutoConfiguration,\
org.springframework.boot.autoconfigure.security.oauth2.client.OAuth2ClientAutoConfiguration,\
org.springframework.boot.autoconfigure.solr.SolrAutoConfiguration,\
org.springframework.boot.autoconfigure.thymeleaf.ThymeleafAutoConfiguration,\
org.springframework.boot.autoconfigure.transaction.TransactionAutoConfiguration,\
org.springframework.boot.autoconfigure.transaction.jta.JtaAutoConfiguration,\
org.springframework.boot.autoconfigure.validation.ValidationAutoConfiguration,\
org.springframework.boot.autoconfigure.web.client.RestTemplateAutoConfiguration,\
org.springframework.boot.autoconfigure.web.embedded.EmbeddedWebServerFactoryCustomizerAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.HttpHandlerAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.ReactiveWebServerFactoryAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.WebFluxAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.error.ErrorWebFluxAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.function.client.WebClientAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.HttpEncodingAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.MultipartAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.reactive.WebSocketReactiveAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.servlet.WebSocketServletAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.servlet.WebSocketMessagingAutoConfiguration,\
org.springframework.boot.autoconfigure.webservices.WebServicesAutoConfiguration

# Failure analyzers
org.springframework.boot.diagnostics.FailureAnalyzer=\
org.springframework.boot.autoconfigure.diagnostics.analyzer.NoSuchBeanDefinitionFailureAnalyzer,\
org.springframework.boot.autoconfigure.jdbc.DataSourceBeanCreationFailureAnalyzer,\
org.springframework.boot.autoconfigure.jdbc.HikariDriverConfigurationFailureAnalyzer,\
org.springframework.boot.autoconfigure.session.NonUniqueSessionRepositoryFailureAnalyzer

# Template availability providers
org.springframework.boot.autoconfigure.template.TemplateAvailabilityProvider=\
org.springframework.boot.autoconfigure.freemarker.FreeMarkerTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.mustache.MustacheTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.groovy.template.GroovyTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.thymeleaf.ThymeleafTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.web.servlet.JspTemplateAvailabilityProvider

```

## 自定义自动配置 ##
1. 在元文件里创建`src/main/resources/META-INF/spring.factories`文件，添加开启自动配置并指向自动配置的类。
该文件会被**SpringBoot**自动配置扫描到并加载指定的配置类。
2. 注册配置类：
	- 注册为Bean
	- 开启配置属性，指定业务属性类
	- 按条件指定业务类
	- 注入业务属性类对象
	- 创建业务类对象，并注册为Bean,在创建业务对象的方法中调用业务属性对象加载属性参数
3. 将此自定义的自动配置安装到 Maven 类，其它项目引入添加该依赖，并配置属性。


### 项目1：自动配置Jar ###
**需求：**一个自动配置的项目，被其它主项目引入直接调用；自动配置项目中有业务需要执行，业务参数需要从配置文件中读取，而该参数是在主项目中配置，只要在主项目中按特定的格式配置好参数，自动配置项目就可以获取得并执行。

1. 创建业务类
``` java
public class HelloService {

	private String msg;
	
	public String sayHello() {
		return "Hello " + msg;
	}

	public String getMsg() {
		return msg;
	}

	public void setMsg(String msg) {
		this.msg = msg;
	}
}
```
2. 创建绑定参数类
``` java
package Properties;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix="hello") //1. 参数绑定，指定前辍
public class HelloServiceProperties {
	
	private static final String MSG = "world";
	private String msg = MSG;
	
	public String getMsg() {
		return msg;
	}
	public void setMsg(String msg) {
		this.msg = msg;
	}
}
```
3. 创建自动配置类
``` java
package com.springboot.AutoConfiguration;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import com.springboot.service.HelloService;

import Properties.HelloServiceProperties;

/**
 * 自动配置配置类
 * @author gxing
 * 1. 配置类需要注册为Bean
 * 2. Properties属性类注册为Bean
 * 3. 在自动配置类中注入属性类实例
 * 4. 在自动配置类中创建方法返回业务类对象，在方法中使用属性类实例调用方法
 */

@Configuration	//1. 当前注册为Bean
@EnableConfigurationProperties(HelloServiceProperties.class) //2. HelloServiceProperties注册为Bean
@ConditionalOnClass(HelloService.class)	//3. 指定类，创建对象
@ConditionalOnProperty(prefix="hello",value="enabled",matchIfMissing=true) //4. 对条件进行检查
public class HelloServiceAutoConfiguration {
	
	@Autowired
	private HelloServiceProperties helloServiceProperties;
	
	@Bean
	@ConditionalOnMissingBean(HelloService.class)
	public HelloService helloService() {
		HelloService helloService = new HelloService();
		helloService.setMsg(helloServiceProperties.getMsg());
		return helloService;
	}
}

```
4. 创建自动配置文件`src/main/resources/META-INF/spring.factories`，在文件中输入开启自动配置和指定配置的类的参数。
``` properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.springboot.AutoConfiguration.AutoConfiguration
```
5. 把**自动配置**项目通过`mvn install`安装到`maven`库。

### 项目2: 调用自动配置 ###
1. 添加自动配置jar为依赖, 在业务层注入自动配jar中的业务类。
``` java
@RestController
@SpringBootApplication
public class AutoConfigApplication {
	
	@Autowired
	HelloService helloService;
	
	@RequestMapping("/")
	public String index() {
		return helloService.sayHello();
	}

	public static void main(String[] args) {
		SpringApplication.run(AutoConfigApplication.class, args);
	}
}
```
2. 在`application.properties`添加参数：`hello.msg=Hello Word`。

可以看到，**项目2**在注入`HelloService`之前，没在该项目其它地方声明为`Bean`; 声明`Bean`是在**项目1**中通过自动配置实现的。

### 关闭相关配置 ###
``` java
@RestController
@SpringBootApplication
//@SpringBootApplication(exclude= {DataSourceAutoConfiguration.class})//关闭某项自动配置
public class Application {
	
	public static void main(String[] args) {
		
		//关闭banner方式1
		SpringApplication app = new SpringApplication(Application.class);
		//关闭banner
		app.setBannerMode(Banner.Mode.OFF);
		app.run(args);
		
		//关闭banner方式2
		new SpringApplicationBuilder(Application.class)
				.bannerMode(Banner.Mode.OFF)
				.run(args);

		SpringApplication.run(Application.class, args);
	}
}
```