---
title: Spring Boot 2实践系列(二十二)：Web相关配置详解
comments: true
date: 2018-07-09 23:35:44
tags: [web,web mvc,spring boot]
categories: [Spring Boot 2实践系列]
---
　　Spring Boot对 Web 项目提供了很好的支持，对视图解析、静态资源、格式化和转器器、消息转换器提供了自动配置，自动映射了静态首页。

　　Web 相关自动配置在 Spring Boot 自动配置包(org.springframework.boot:spring-boot-autoconfigure.2.0.0.RELEASE)里 Web 路径下：org.springframework.boot.autoconfigure.web。在该路径下可以看到提供了**Web**相关的自动配置，如 RestTemplate, embedded(内嵌的应用服务器：jetty,tomcat,undertow), format，reactive，Servlet等。
<!-- more -->
## Servlet ##
Spring Boot Servlet 提供了默认的并自动配置的错误解析器(ErrorViewResolver)、自动配置了 DispatcherServlet, HttpEncoding(默认是UTF-8),还包括 JspTemplate，Multipart, WebMvc的自动配置。

查看自动配置属性配置类的原码，可以看到支持配置的属性前缀和属性名称：
WebMvc属性配置类：WebMvcProperties.java，配置的前缀是`spring.mvc`，设置WebMvc相关，如前缀，后缀等。
静态资源属性配置类：ResourceProperties.java，配置的前缀是`spring.resources`，可以设置一些对静态资源的控制。
容器服务器的属性配置类：ServerProperties.java，配置的前缀是`server`，可设置端口，编码，context-path等。

## ViewResolver ##
`WebMvcAutoConfiguration`创建多个视图解析器：
1. **ContentNegotiatingViewResolver**：自己并不进行视图解析，而是代理给其它 ViewResolver 来执行，并且优先级最高，该视图解析器会根据请求的MediaType 来选择合适的 ViewResolver。
2. **InternalResourceViewResolver**：用于解析配置了前后缀的视图，返回视图名的字符串来得到实际的页面。
`spring.mvc.view.prefix` 配置视图前辍，`spring.mvc.view.suffix`配置视图后缀。 
JSP模板`JspTemplateAvailabilityProvider`会通过环境变量直接获取配置的前后辍，为空则返回 **WebMvcAutoConfiguration** 自动配置默认的前后缀(也为空:"")。看源码分析，如果配置了视图前后缀，通过 **JspTemplateAvailabilityProvider**直接获取的前后缀与从**WebMvcAutoConfiguration**中拿到的前后缀是一致的，都是从配置文件中同一个属性获取值。
3. **BeanNameViewResolver**：根据视图名(Controller 返回 String 视图)解析视图。

## 静态资源 ##
自动配置类`WebMvcAutoConfiguration`的`addResourceHandlers`方法中配置了默认静态资源目录，而默认的静态资源目录是在资源属性类(`ResourceProperties`)中定义的。也可在项目的配置文件中，通过使用前缀`spring.resources`来自定义资源路径，`staticLocations`是个数组, 静态资源的路径会被映射为(`/**`), 可以通过 `http://localhost/**`业访问(省略静态资源的目录,直接定位到下一级路径)。

默认静态资源目录
``` java
private static final String[] CLASSPATH_RESOURCE_LOCATIONS = {
			"classpath:/META-INF/resources/", "classpath:/resources/",
			"classpath:/static/", "classpath:/public/" };

private String[] staticLocations = CLASSPATH_RESOURCE_LOCATIONS;
```

## Formatter与Converter ##
关于`Formatter与Converter`的自动配置，是在`WebMvcAutoConfiguration`的`addFormatters`方法中对`converter`和`formatter`类型进行判断, 符合的会被添加到`FormatterRegistry`中。
**WebMvcAutoConfiguration**中创建了`WebConversionService Bean`，调用的构造方法，默认是从`mvcProperties`属性配置中获取日期格式化样式添加到格式器中执行。

``` java
@Override
public void addFormatters(FormatterRegistry registry) {
	for (Converter<?, ?> converter : getBeansOfType(Converter.class)) {
		registry.addConverter(converter);
	}
	for (GenericConverter converter : getBeansOfType(GenericConverter.class)) {
		registry.addConverter(converter);
	}
	for (Formatter<?> formatter : getBeansOfType(Formatter.class)) {
		registry.addFormatter(formatter);
	}
}
```
在使用时，自定义的格式器或转换器，只需要实现`Converter，GenericConverter，Formatter`接口，就会被添加到`FormatterRegistry`自动注册为Bean。

**WebMvcAutoConfiguration**中还创建了**RequestMappingHandlerAdapter**，**Validator**，**ExceptionHandlerExceptionResolver**等**Bean**。

![](https://i.loli.net/2019/04/16/5cb573114650e.png)

## HttpMessageConverter ##
``` java
private final HttpMessageConverters messageConverters;

public WebMvcAutoConfigurationAdapter(ResourceProperties resourceProperties,
				WebMvcProperties mvcProperties, ListableBeanFactory beanFactory,
				@Lazy HttpMessageConverters messageConverters,
				ObjectProvider<ResourceHandlerRegistrationCustomizer> resourceHandlerRegistrationCustomizerProvider) {
			this.resourceProperties = resourceProperties;
			this.mvcProperties = mvcProperties;
			this.beanFactory = beanFactory;
			this.messageConverters = messageConverters;
			this.resourceHandlerRegistrationCustomizer = resourceHandlerRegistrationCustomizerProvider
					.getIfAvailable();
		}

public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
			converters.addAll(this.messageConverters.getConverters());
		}
```
在`WebMvcAutoConfiguration`中，通过 WebMvcAutoConfigurationAdapter 注入了 HttpMessageConverters 的 Bean, 这个 Bean 是在 org.springframework.boot.autoconfigure.http.HttpMessageConvertersAutoConfiguration 中定义的。
``` java
@Bean
@ConditionalOnMissingBean
public HttpMessageConverters messageConverters() {
	return new HttpMessageConverters(
			this.converters != null ? this.converters : Collections.emptyList());
}
```
如果需要新增自定义 HttpMessageConverters，则只需要创建自定义的 HttpMessageConverters 的 Bean，然后在此 Bean 中注册自定义的 HttpMessageConverter 即可,如下：
``` java
@Bean
public HttpMessageConverters customConverters(){
	HttpMessageConverter<?> customConverter1 = new CustomConverter1();
	HttpMessageConverter<?> customConverter2 = new CustomConverter2();
	
	return new HttpMessageConverters(customConverter1,customConverter2);
}
```

在 Spring Boot 自动配置的 jar包中的 org.springframework.boot.autoconfigure.http 路径下，还配置了 Gson消息转换`GsonHttpMessageConvertersConfiguration`，Http编码`HttpEncodingProperties`默认为 UTF-8，Jackson消息转换`JacksonHttpMessageConvertersConfiguration`, Jsonb消息转换`JsonbHttpMessageConvertersConfiguration`(postgresql 的数据格式)。

## CROS 跨域支持

跨源资源共享(CROS：[Cross-origin resource sharing](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing)) 是大多数浏览器实现的 [W3C CROS](https://www.w3.org/TR/cors/) 标准，允许以灵活的方式指定授权何种跨域请求，而不是使用一些安全性较低且功能较弱的方法，如 IFRAME 或 JSONP。

从版本 Spring 4.2 开始，Spring MVC 支持 CORS。在 Spring Boot 应用程序中可以在 Controller 方法上使用 @CrossOrigin 注解来配置 CORS。

从 Spring Boot 2.x 版本开始，还可以定义实现  WebMvcConfigurer  的 Bean 重写 addCorsMappings(CorsRegistry) 方法来定义全局 CROS 配置，如下示例：

```java
@Configuration
public class MyConfiguration {

	@Bean
	public WebMvcConfigurer corsConfigurer() {
		return new WebMvcConfigurer() {
			@Override
			public void addCorsMappings(CorsRegistry registry) {
				registry.addMapping("/api/**");
			}
		};
	}
}
```

更多示例如下：

```java
/**
 * @name: WebConfig
 * @desc: Web配置
 * @date: 2018-09-26 17:30
 **/
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {

        String[] ignorePath = {"/index", "/dict/**", "/login/**", "/code/imgCode", "/code/loginPhoneCode",
                "/webTemplate/**", "/error", "/callback/**"};

        //登录拦截器
        registry.addInterceptor(new LoginInterceptor())
                .excludePathPatterns(ignorePath);
    }

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("*")
                .allowedHeaders("*")
                .allowedMethods("*")
                .allowCredentials(true);
    }

    @Bean
    public RestTemplate customRestTemplate() {
        //设置超时时间,毫秒
        return new RestTemplateBuilder().setConnectTimeout(1000).setReadTimeout(1000).build();
    }
}
```

## 欢迎首页 ##

项目默认的欢迎首页(`index.html`)也是`WebMvcAutoConfiguration`中进行了自动配置，在**WebMvcAutoConfiguration**中注册了个欢迎页面的处理映射器`WelcomePageHandlerMapping`，源码如下：
``` java
//首页的处理映射
@Bean
public WelcomePageHandlerMapping welcomePageHandlerMapping(
		ApplicationContext applicationContext) {
	return new WelcomePageHandlerMapping(
			new TemplateAvailabilityProviders(applicationContext),
			applicationContext, getWelcomePage(),
			this.mvcProperties.getStaticPathPattern());
}

//获取首页,这里的locations 就是上面提到的默认的静态资源目录
private Optional<Resource> getWelcomePage() {
	String[] locations = getResourceLocations(
			this.resourceProperties.getStaticLocations());
	return Arrays.stream(locations).map(this::getIndexHtml)
			.filter(this::isReadable).findFirst();
}

//定义默认的首页
private Resource getIndexHtml(String location) {
	return this.resourceLoader.getResource(location + "index.html");
}
```
如果项目根目录存在`index.html`页面，直接访问项目的域名( http://localhost:8080/ )会映射到 index.html 页面。

## 自定义Web配置 ##
如果 Spring Boot 提供的 Spring MVC 配置不符合要求，可以通过创建一个配置类(有 `@Configuration` 注解)，加上`@EnableWebMvc`注解来实现完全自己控制的`MVC`配置。

通常情况下，Spring Boot 提供的自动配置是能满足大多数需求的，当需要增加自己的额外配置时，可以定义一个类实现`WebMvcConfigurer`(spring 5.0),无须`@EnableWebMvc`注解。
``` java
@Configuration
public class MyWebConfig implements WebMvcConfigurer {

    public void addViewControllers(ViewControllerRegistry registry) {
		//添加URL映射
        registry.addViewController("/home").setViewName("home");
		//重定向
        registry.addRedirectViewController("/selectAll", "/city/queryAll");
    }
}
```

## Servlet&Filter&Listener ##
[官网 -> Servlets, Filters, and listeners](https://docs.spring.io/spring-boot/docs/2.0.3.RELEASE/reference/htmlsingle/#boot-features-embedded-container-servlets-filters-listeners)
当使用Spring Boot内嵌的Servlet容器(Tomcat,Jetty)时，可将自定义的`Servlet，Filter，Listener`声明为Spring Bean或 Servlet 的组件，依照Servlet规范注册到 Servlet容器中。

Spring Boot 提供了一些定义 Filter Beans 的自动配置，以下是一些过滤器和及其顺序的示例，如果多个过滤器是无序的也是安全的。

|   Servlet Filter  |   Order      |
|:------------------|:-------------|
|OrderedCharacterEncodingFilter|Ordered.HIGHEST_PRECEDENCE|
|WebMvcMetricsFilter|Ordered.HIGHEST_PRECEDENCE + 1|
|ErrorPageFilter|Ordered.HIGHEST_PRECEDENCE + 1|
|HttpTraceFilter|Ordered.LOWEST_PRECEDENCE - 10|

当Servlet，Filter，Listener作为Servlet组件时，可使用`@ServletComponentScan`来扫描启用`@WebServlet，@WebFilter，@WebListener `注释实现自动注册。
**备注：**`@ServletComponentScan`对独立的容器没有影响，独立容器有内部的发现机制来代替此注解。

自定义ServletWebServer
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

## Favicon ##
`Favicon` 指的是站点在浏览器标签上的**Logo**图标。 Spring Boot提供了一个默认的 Favicon，就是 Spring 的图标, 默认是开启的，在浏览器上可以看到。
1. 关闭 Favicon
> spring.mvc.favicon.enabled=false

2. 自定义 Favicon
将自己的`favicon.ico`(文件名固定不可变)文件放到默认的静态资源路径下，重启项目访问就可以看到浏览器标签显示自定义的**logo**。

