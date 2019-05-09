---
title: Spring Boot 2实践系列(四十三)：源码分析 SpringApplication 执行流程
comments: true
date: 2019-05-03 22:52:26
tags: [SpringApplication]
categories: [Spring Boot 2实践系列]
---

　　源码分析 SpringApplication 执行流程，深入理解 Spring Boot 应用的启动流程。

　　本篇分析基于 Spring Boot 2.1.4.RELEASE 版本。

<!-- more -->

## 源码分析

1. Spring Boot 启动类

   ``` java
   @SpringBootApplication
   public class ConsumerService1Application {
       // main 方法启动类
       public static void main(String[] args) {
           // run 方法执行类
           SpringApplication.run(ConsumerService1Application.class, args);
       }
   }
   ```

2. run 方法源码，以下展示 run 方法调用关系

   ``` java
   // 1
   public static ConfigurableApplicationContext run(Class<?> primarySource,
                                                    String... args) {
       return run(new Class<?>[] { primarySource }, args);
   }
   // 2
   public static ConfigurableApplicationContext run(Class<?>[] primarySources,
                                                    String[] args) {
       return new SpringApplication(primarySources).run(args);
   }
   // 3
   public SpringApplication(Class<?>... primarySources) {
       this(null, primarySources);
   }
   // 4 做一些初始化判断和赋值的操作
   public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
       this.resourceLoader = resourceLoader;
       Assert.notNull(primarySources, "PrimarySources must not be null");
       this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
       // 判断 webApplication 类型
       this.webApplicationType = WebApplicationType.deduceFromClasspath();
       // 执行初始化
       setInitializers((Collection) getSpringFactoriesInstances(
           ApplicationContextInitializer.class));
       // 加载监听器
       setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
       this.mainApplicationClass = deduceMainApplicationClass();
   }
   ```

   run 方法里首先创建一个 SpringApplication 对象实例，在 SpringApplication  实例初始化前，会提完做几件事
   
3. deduceFromClasspath() 方法：对 Web 容器全限定类名进行反射获取 Class 对象来判断 classpath 是否存在某个 Web 容器类型来决定创建一个 Web 应用使用的 Application Context 类型。

   **enum WebApplicationType.java**

     ``` java
     public enum WebApplicationType {
     	NONE,
     
     	SERVLET,
     
     	REACTIVE;
     
     	private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet",
     			"org.springframework.web.context.ConfigurableWebApplicationContext" };
     
     	private static final String WEBMVC_INDICATOR_CLASS = "org.springframework."
     			+ "web.servlet.DispatcherServlet";
     
     	private static final String WEBFLUX_INDICATOR_CLASS = "org."
     			+ "springframework.web.reactive.DispatcherHandler";
     
     	private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";
     
     	private static final String SERVLET_APPLICATION_CONTEXT_CLASS = "org.springframework.web.context.WebApplicationContext";
     
     	private static final String REACTIVE_APPLICATION_CONTEXT_CLASS = "org.springframework.boot.web.reactive.context.ReactiveWebApplicationContext";
     
     	static WebApplicationType deduceFromClasspath() {
             // 根据全限定类名，判断是那种 WebApplication 类型
     		if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null)
     				&& !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
     				&& !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
     			return WebApplicationType.REACTIVE;
     		}
     		for (String className : SERVLET_INDICATOR_CLASSES) {
     			if (!ClassUtils.isPresent(className, null)) {
     				return WebApplicationType.NONE;
     			}
     		}
           // servlet 类型
     		return WebApplicationType.SERVLET;
     	}
     }
     ```
   **ClassUtils.java**

     ``` java
   // 判断提供的 className 是否存在(Web 容器类名)
   public static boolean isPresent(String className, @Nullable ClassLoader classLoader) {
       try {
           forName(className, classLoader);
           return true;
       }
       catch (IllegalAccessError err) {
           throw new IllegalStateException("Readability mismatch in inheritance hierarchy of class [" +
                                           className + "]: " + err.getMessage(), err);
       }
       catch (Throwable ex) {
           // Typically ClassNotFoundException or NoClassDefFoundError...
           return false;
       }
   }
   
   // 条件判断，调用 Class.forName() 方法反射实例化
   public static Class<?> forName(String name, @Nullable ClassLoader classLoader)
       throws ClassNotFoundException, LinkageError {
   
       //-----省略------
       try {
           // 反射实例化
           return Class.forName(name, false, clToUse);
       }
       catch (ClassNotFoundException ex) {
           int lastDotIndex = name.lastIndexOf(PACKAGE_SEPARATOR);
           if (lastDotIndex != -1) {
               String innerClassName =
                   name.substring(0, lastDotIndex) + INNER_CLASS_SEPARATOR + name.substring(lastDotIndex + 1);
               try {
                   return Class.forName(innerClassName, false, clToUse);
               }
               catch (ClassNotFoundException ex2) {
                   // Swallow - let original exception get through
               }
           }
           throw ex;
       }
   }
     ```

4.  setInitializers() 和 setListeners() 方法借助 SpringFactoriesLoader 查找 classpath 中并加载所有可用的 ApplicationContextInitializer 和 ApplicationListener。

   实际是通过类加载器读取所有包下的资源文件 *META-INF/spring.factories* 中的内容存到一个 Map 中，然后找出 ApplicationContextInitializer 和 ApplicationListener 用于实例化。

5. SpringApplication 实例完成设置初始化完成后，开始执行 run 方法里面的逻辑。

   **SpringApplication.java**

   ``` java
   public static ConfigurableApplicationContext run(Class<?>[] primarySources,
                                                    String[] args) {
       return new SpringApplication(primarySources).run(args);
   }
   
   // 创建 ApplicationContext 并初始化
   public ConfigurableApplicationContext run(String... args) {
       StopWatch stopWatch = new StopWatch();
       stopWatch.start();
       ConfigurableApplicationContext context = null;
       Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
       configureHeadlessProperty();
       // 从 META-INF/spring.factories 文件获取监听器
       SpringApplicationRunListeners listeners = getRunListeners(args);
       // 告诉启动
       listeners.starting();
       try {
           ApplicationArguments applicationArguments = new DefaultApplicationArguments(
               args);
           // 准备环境
           ConfigurableEnvironment environment = prepareEnvironment(listeners,
                                                                    applicationArguments);
           configureIgnoreBeanInfo(environment);
           // 打印 Banner
           Banner printedBanner = printBanner(environment);
           // 创建 ApplicationContext
           context = createApplicationContext();
           exceptionReporters = getSpringFactoriesInstances(
               SpringBootExceptionReporter.class,
               new Class[] { ConfigurableApplicationContext.class }, context);
           // 将 environment 设置到 context
           prepareContext(context, environment, listeners, applicationArguments,
                          printedBanner);
           // 刷新context 
           refreshContext(context );
           afterRefresh(context, applicationArguments);
           stopWatch.stop();
           if (this.logStartupInfo) {
               new StartupInfoLogger(this.mainApplicationClass)
                   .logStarted(getApplicationLog(), stopWatch);
           }
           listeners.started(context);
           // 查看是否注册 CommandLineRunner，有则遍历
           callRunners(context, applicationArguments);
       }
       catch (Throwable ex) {
           handleRunFailure(context, ex, exceptionReporters, listeners);
           throw new IllegalStateException(ex);
       }
   
       try {
           // 遍历执行 SpringApplicationRunListeners 的 running 方法
           listeners.running(context);
       }
       catch (Throwable ex) {
           handleRunFailure(context, ex, exceptionReporters, null);
           throw new IllegalStateException(ex);
       }
       return context;
   }
   ```

6. 在 ApplicationContext 创建初始化完成后，执行 @EnableAutoConfiguration 注解里的自动配置，并将 IoC 容器配置加载到已准备好的 ApplicationContext 。

## 总体逻辑

1. 创建 SpringApplication，并执行初始化。主要是创建应用类型，查找设置 ApplicationContextInitializer 和 ApplicationListener 。
2. 创建 ApplicationContext，并执行初始化，设置环境，有则遍历 CommandLineRunner，遍历 SpringApplicationRunListeners，ApplicationContext 准备完成。
3. 执行自动配置注解，加载自动配置，并将与 IoC 相关的配置加载到 ApplicationContext 。

SpringApplicationRunListeners：该类调用的是 EventPublishingRunListener，用于在 Spring Boot 启动的不同阶段发布不同的应用事件类型(ApplicationEvent)，如果有那些 ApplicationListener 对这些应用事件感兴趣，则可以接收并处理。