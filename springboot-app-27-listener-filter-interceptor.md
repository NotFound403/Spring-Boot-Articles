---
title: Spring Boot 2实践系列(二十七)：Listener, Filter, Interceptor
comments: true
date: 2018-08-14 15:40:01
tags: [listener,filter,interceptor,监听器,过滤器,拦截器]
categories: [Spring Boot 2实践系列]
---
　　Listener 监听器,Filter 过滤器,Interceptor 拦截器是 Java Web领域非常重要的三大神器(组件)，会经常使用到。

　　关注这三个的知识点本篇不做描述，主要记录在Spring Boot框架中这三大组件的使用。

　　前面文章也有提到:[Spring事件监听 ](http://112.74.59.39/%2F2018%2F02%2F22%2Fspring-event%2F),[SpringMVC之HandlerInterceptor拦截器](http://112.74.59.39/%2F2018%2F02%2F28%2Fspringmvc-interceptor%2F), [Spring Boot实践系列(二十二)：Web相关配置详解 ](http://112.74.59.39/2018/07/09/springboot-app-22-web/#Servlet-amp-Filter-amp-Listener)。

　　官方文档：[Spring Boot -> Application Events and Listeners](https://docs.spring.io/spring-boot/docs/2.0.4.RELEASE/reference/htmlsingle/#boot-features-application-events-and-listeners),[Spring Boot -> Servlets, Filters, Listeners](https://docs.spring.io/spring-boot/docs/2.0.4.RELEASE/reference/htmlsingle/#boot-features-application-events-and-listeners), [Spring Boot -> Add a Servlet, Filter, or Listener to an Application](https://docs.spring.io/spring-boot/docs/2.0.4.RELEASE/reference/htmlsingle/#howto-add-a-servlet-filter-or-listener-using-scanning)。
<!-- more -->
## Spring Boot ##
在 Spring Boot 框架中，将自定义的 Servlet, Filter, Listener 注册为 Spring Bean有两种方式：
1. Spring Boot框架中在自定义的 Servlet, Filter, Listener 三大组件类上添加`@Component 或 @Configuration`注解将监听器注册为 Spring Bean, 可调用 IoC 容器的产资源(Bean)。
2. 在自定义的 Servlet, Filter, Listener 类上添加`@WebServlet, @WebFilter, @WebListener`注解，会自动将自定义三大给件类注册为 Servlet 组件并启动。

**注意：**如果使用的是 Spring Boot 内嵌的 Servlet 容器，还需要在使用了`@Configuration`注解的类上添加`@ServletComponentScan`注解; 如果部署在外部独立的Servlet 容器，则不需要**@ServletComponentScan**注解，由独立容器的内部发现机制将自定义的三大组件类注册为 Servlet组件。

**@ServletComponentScan**注解，是在使用 Spring Boot 内嵌的 Servlet 容器时需要添加，用于是启用扫描使用了**@WebServlet, @WebFilter, @WebListener**注解的类并将之注册为 Servlet 组件, 可通过`value`或`basePackages`属性指定自定义的 Servlet, Filter, Listener 所在的包路径, 就会自动将该包下使用了 @WebServlet, @WebFilter, @WebListener 注解的自定义类注册 Servlet 组件; 默认情况下会从 `@ServletComponentScan`注解所在类的包开始扫描，所以很多时候直接把这个注解放在启动入口类上。

## Listener ##
**Listener**是 Servlet 规范，由 Servlet 容器提供支持，随Servlet容器启动就启动监听, 监听到事件时，在监听器里面相应的方法里执行处理。
Servlet 提供的3个基本监听器接口：ServletContextListener，ServletRequestListener，HttpSessionListener，使用时实现用到的接口，重写里面的方法 。 

1. 根据创建的Session数**统计在线人数**
根据 Session 统计 Web 站点的计在线人数并不十分准确, 若用户离线了但 Session 仍在有效期内则认为这个用户仍在线。
``` java
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.web.context.WebApplicationContext;
import org.springframework.web.context.support.WebApplicationContextUtils;

import javax.servlet.ServletContext;
import javax.servlet.annotation.WebListener;
import javax.servlet.http.HttpSession;
import javax.servlet.http.HttpSessionEvent;
import javax.servlet.http.HttpSessionListener;

/**
 * @Name: CountOnlineUserListener
 * @Desc: 统计在线用户数
 **/

@WebListener
public class CountOnlineUserListener implements HttpSessionListener {

    private static final Logger logger = LogManager.getLogger(CountOnlineUserListener.class);

    private static final String TOTAL_ONLINE_USER="totalOnlineUser";

    private RedisTemplate<String, Integer> redisTemplate;

    @Override
    public void sessionCreated(HttpSessionEvent event) {

        HttpSession session = event.getSession();
        //session默认有效时间是30分钟,可不设置
        session.setMaxInactiveInterval(30 * 60);
        ServletContext servletContext = session.getServletContext();
        WebApplicationContext webApplicationContext = WebApplicationContextUtils.getWebApplicationContext(servletContext);
        redisTemplate = (RedisTemplate<String, Integer>) webApplicationContext.getBean("redisTemplate");
        Integer integer = redisTemplate.opsForValue().get(TOTAL_ONLINE_USER);
        integer = (integer == null ? 0 : integer);
        //创建Session数
        redisTemplate.opsForValue().set(TOTAL_ONLINE_USER,integer + 1);
    }

    @Override
    public void sessionDestroyed(HttpSessionEvent se) {
        Integer integer = redisTemplate.opsForValue().get(TOTAL_ONLINE_USER);
        redisTemplate.opsForValue().set(TOTAL_ONLINE_USER,integer - 1);
    }
}
```
2. 容器启动监听器，执行初始化操作
因上面的统计在线用户数是存在 `Redis` 中, 如果系统宕机或崩溃是不会清空引在线用户数的值的，会存在较大误差。所在在系统起动时重置下。
如果是单机系统，可以将计算的用户数放到`ServletContext`域中。
``` java
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.web.context.WebApplicationContext;
import org.springframework.web.context.support.WebApplicationContextUtils;

import javax.servlet.ServletContext;
import javax.servlet.ServletContextEvent;
import javax.servlet.ServletContextListener;
import javax.servlet.annotation.WebListener;

/**
* @Desc: 重写ServletContext
* @Date: 2018/8/14 11:29
**/
@WebListener
public class DydContextLoaderListener implements ServletContextListener {

    private RedisTemplate<String, Integer> redisTemplate;

    /**
    * @Desc: 容器启动增加初始化操作
    * @Param: [sce]
    * @Return: void
    **/
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        ServletContext servletContext = sce.getServletContext();
        WebApplicationContext webApplicationContext = WebApplicationContextUtils.getWebApplicationContext(servletContext);
        redisTemplate = (RedisTemplate<String, Integer>) webApplicationContext.getBean("redisTemplate");
        //重置在线人数
        redisTemplate.opsForValue().set("totalOnlineUser",0);
    }

    @Override
    public void contextDestroyed(ServletContextEvent sce) {

    }
}
```


## Filter ##
**Filter**过滤器是 Servlet 规范，由 Servlet 容器提供支持，在 Servlet 之前和之后起作用，无法调用 Spring IOC中的资源。

基础知识，参考[Servlet 编写过滤器](http://www.runoob.com/servlet/servlet-writing-filters.html)

## Interceptor ##
**Interceptor**是 Spring 的一个组件，由 Spring 框架提供支持，由 Spring 管理，可在拦截器中注入 IoC 中的资源。
1. 登录拦截器
	``` java
	/**
	 * @ClassName LoginInterceptor
	 * @Description 登录验证拦截器
	 **/
	@Component
	public class LoginInterceptor implements HandlerInterceptor {
	
	    private static final Logger log = LoggerFactory.getLogger(LoginInterceptor.class);
	
	    @Override
	    public boolean preHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o) throws Exception {
	        HttpSession session = httpServletRequest.getSession();
	        // 判断是否登录
	        if (null != session.getAttribute("SESSION_USER_OBJ")) {
	            return true;
	        } else {
	            httpServletResponse.sendRedirect("/user/toLogin");
	            return false;
	        }
	    }
	
	    @Override
	    public void postHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, ModelAndView modelAndView) throws Exception {
	
	    }
	
	    @Override
	    public void afterCompletion(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, Exception e) throws Exception {
	
	    }
	}
	```
2. 配置放行路径
	**注：**新的 Spring 5.0 是实现 `WebMvcConfigurer` 接口，**WebMvcConfigurerAdapter**已标注为过期。
	``` java
	/**
	 * @ClassName WebAppConfig
	 * @Description Web配置
	 **/
	@SpringBootConfiguration
	public class WebAppConfig extends WebMvcConfigurerAdapter {
	
	    @Override
	    public void addInterceptors(InterceptorRegistry registry) {
	        //注册自定义拦截器，添加拦截路径和排除拦截路径
	        registry.addInterceptor(new LoginInterceptor())
	                .addPathPatterns("/**")
					.excludePathPatterns("/index")
					.excludePathPatterns("/user/register")
	                .excludePathPatterns("/user/login")
	                .excludePathPatterns("/error");
	    }
	}
	```

三大神器的执行顺序：监听电话 -> 过滤垃圾信号 -> 拦截恐怖事件，任务处理完则原路返回。

【参考：】
[Java三大器之监听器](https://blog.csdn.net/reggergdsg/article/details/52891311)