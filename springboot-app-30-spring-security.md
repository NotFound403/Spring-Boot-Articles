---
title: Spring Boot 2实践系列(三十)：Spring Security 安全框架详解和集成
comments: true
date: 2018-08-31 16:05:07
tags: [security,spring,web]
categories: [Spring Boot 2实践系列]
---
Spring Security 为基于 Spring 的应用程序提供安全保护，是一个功能强大且可高度自定义的自份验证和访问控制框架。

应用程序安全性的两个主要方面是`身份验证`(认证：Authentication)和`授权`(访问控制：Authorization), 这也是 Spring Security 目标的两个主要领域。

**身份验证：**认证，即确认用户可以访问系统，可理解为用户账号密码正确且有效。
**授权**：访问控制，即用户在当前系统下所拥有的功能权限。


Spring Boot 关于 Spring Security 官方说明[Security](https://docs.spring.io/spring-boot/docs/2.0.4.RELEASE/reference/htmlsingle/#boot-features-security), [Spring Security 官方文档 -> learn](https://spring.io/projects/spring-security#overview), [Spring Boot 集成 Spring Security 官方说明](https://docs.spring.io/spring-security/site/docs/5.0.7.RELEASE/guides/html5/helloworld-boot.html), [Spring Security -> Samples and Guides (Start Here)](https://docs.spring.io/spring-security/site/docs/5.0.7.RELEASE/reference/htmlsingle/#samples)
<!-- more -->

## Spring Security ##
Spring Security 是通过过滤器来实现所有安全的功能。 Spring Security 提供了 AbstractSecurityWebApplicationInitializer 抽象类，实现了 WebApplicationInitializer 接口, 重写了 onStartup(ServletContext servletContext) 方法, 在方法里调用了 insertSpringSecurityFilterChain(servletContext)方法, 将 springSecurityFilterChain 过滤器注册到 Servlet 容器。
### 源码分析 ###
1. springSecurityFilterChain 过滤器, 在所有其它过滤器之前执行。
``` java
@Configuration
public class WebSecurityConfiguration implements ImportAware, BeanClassLoaderAware {

	private List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers;

	/**
	 * Creates the Spring Security Filter Chain (创建 springSecurityFilterChain 过滤器)
	 * @return
	 * @throws Exception
	 */
	@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
	public Filter springSecurityFilterChain() throws Exception {
		
		//判断是否有自定义的WebSecurityConfig
		boolean hasConfigurers = webSecurityConfigurers != null
				&& !webSecurityConfigurers.isEmpty();
		if (!hasConfigurers) {

			//使用默认配置,在 WebSecurityConfigurerAdapter 的getHttp方法里
			WebSecurityConfigurerAdapter adapter = objectObjectPostProcessor
					.postProcess(new WebSecurityConfigurerAdapter() {
					});
			webSecurity.apply(adapter);
		}
		return webSecurity.build();
	}

	/**
	 * 该方法会被提前执行,用于提前设置安全配置
	 * 1. 获取自定义的 WebSecurityConfig 配置
	 * 2. 创建 webSecurity 实体
	 * 3. 将自定义 WebSecurityConfig 配置添加到 webSecurity
	 */
	@Autowired(required = false)
	public void setFilterChainProxySecurityConfigurer(
			ObjectPostProcessor<Object> objectPostProcessor,			
			
			//获取所有 WebSecurityConfigurer 接口类类型的 Bean(包括自定义的WebSecurityConfig配置)
			@Value("#{@autowiredWebSecurityConfigurersIgnoreParents.getWebSecurityConfigurers()}") List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers)
			throws Exception {
		webSecurity = objectPostProcessor
				.postProcess(new WebSecurity(objectPostProcessor));
		if (debugEnabled != null) {
			webSecurity.debug(debugEnabled);
		}

		Collections.sort(webSecurityConfigurers, AnnotationAwareOrderComparator.INSTANCE);

		Integer previousOrder = null;
		Object previousConfig = null;
		for (SecurityConfigurer<Filter, WebSecurity> config : webSecurityConfigurers) {
			Integer order = AnnotationAwareOrderComparator.lookupOrder(config);
			if (previousOrder != null && previousOrder.equals(order)) {
				throw new IllegalStateException(
						"@Order on WebSecurityConfigurers must be unique. Order of "
								+ order + " was already used on " + previousConfig + ", so it cannot be used on "
								+ config + " too.");
			}
			previousOrder = order;
			previousConfig = config;
		}
		for (SecurityConfigurer<Filter, WebSecurity> webSecurityConfigurer : webSecurityConfigurers) {
			//将自定义配置添加到 webSecurity里
			webSecurity.apply(webSecurityConfigurer);
		}
		this.webSecurityConfigurers = webSecurityConfigurers;
	}
}
```
2. AbstractSecurityWebApplicationInitializer 抽像类
继承了 WebApplicationInitializer 接口,重写了 onStartup()方法; 该类由 SpringServletContainerInitializer 类通过 Servlet 的 @HandlesTypes注解自动扫描, SpringServletContainerInitializer 实现了 ServletContainerInitializer 接口, Servlet(Tomcat)容器启动时会调用其onStartup()操作。 
``` java
public abstract class AbstractSecurityWebApplicationInitializer
		implements WebApplicationInitializer {

	public static final String DEFAULT_FILTER_NAME = "springSecurityFilterChain";

	// onStartup()方法, Servlet容器初始化时执行
	public final void onStartup(ServletContext servletContext) throws ServletException {
		beforeSpringSecurityFilterChain(servletContext);
		if (this.configurationClasses != null) {
			AnnotationConfigWebApplicationContext rootAppContext = new AnnotationConfigWebApplicationContext();
			rootAppContext.register(this.configurationClasses);
			
			//添加ContextLoaderListener监听器
			servletContext.addListener(new ContextLoaderListener(rootAppContext));
		}
		if (enableHttpSessionEventPublisher()) {
			servletContext.addListener(
					"org.springframework.security.web.session.HttpSessionEventPublisher");
		}
		servletContext.setSessionTrackingModes(getSessionTrackingModes());
		
		//注册 springSecurityFilterChain 过滤器
		insertSpringSecurityFilterChain(servletContext);
		afterSpringSecurityFilterChain(servletContext);
	}

	
	/**
	 * Registers the springSecurityFilterChain(注册 springSecurityFilterChain 过滤器)
	 * @param servletContext the {@link ServletContext}
	 */
	private void insertSpringSecurityFilterChain(ServletContext servletContext) {
		String filterName = DEFAULT_FILTER_NAME;
	
		//从Spring 容器中获取过滤器
		DelegatingFilterProxy springSecurityFilterChain = new DelegatingFilterProxy(
				filterName);
		String contextAttribute = getWebApplicationContextAttribute();
		if (contextAttribute != null) {
			springSecurityFilterChain.setContextAttribute(contextAttribute);
		}
		
		//注册 Filter 到 Servlet容器
		registerFilter(servletContext, true, filterName, springSecurityFilterChain);
	}
	

	// 将过滤器添加到 Servlet容器,并提供异步支持
	private final void registerFilter(ServletContext servletContext,
			boolean insertBeforeOtherFilters, String filterName, Filter filter) {
	
		//添加过滤器到容器
		Dynamic registration = servletContext.addFilter(filterName, filter);
		if (registration == null) {
			throw new IllegalStateException(
					"Duplicate Filter registration for '" + filterName
							+ "'. Check to ensure the Filter is only configured once.");
		}
		registration.setAsyncSupported(isAsyncSecuritySupported());
		EnumSet<DispatcherType> dispatcherTypes = getSecurityDispatcherTypes();
		registration.addMappingForUrlPatterns(dispatcherTypes, !insertBeforeOtherFilters,
				"/*");
	}
}
```
### 集成使用 ###
1. 自定义初始化类，继承 AbstractSecurityWebApplicationInitializer 抽像类, 开启。
``` java
import org.springframework.security.web.context.AbstractSecurityWebApplicationInitializer;

/**
 * @Name: WebAppInitializer
 * @Desc: 开启 Spring Security 过滤器的支持
 **/
public class WebAppInitializer extends AbstractSecurityWebApplicationInitializer {

}
```
2. 自定义 WebSecurityConfig 类, 添加 `@EnableWebSecurity` 注解, 继承 WebSecurityConfigurerAdapter 抽像类，重写里面的方法。
``` java
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.builders.WebSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;

/**
 * @Name: WebSecurityConfig
 * @Desc: 自定义 WebSecurity 配置
 **/
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity httpSecurity) throws Exception {
		super.configure(httpSecurity);          
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
       	super.configure(auth);
    }

    @Override
    public void configure(WebSecurity webSecurity) throws Exception {
       super.configure(webSecurity);
    }
}
```


## Spring Boot 集成 ##
### 自动配置 ###
Spring Boot 为 Spring Security 提供了自动配置, 在 org.springframework.boot.autoconfigure.security 包下。
Spring Security 自动配置的核心类有 SecurityProperties(属性类), SecurityAutoConfiguration(Security自动配置类), SecurityFilterAutoConfiguration(过滤器自动配置类)。

1. SecurityProperties 获取在 application.properties 自定义的安全属性, 自定义属性前缀是 **spring.security** 。

    在该文件中有个静态内部类(`User`), 里面定义了默认的用户名是`user`, 默认的密码是`UUID.randomUUID().toString()`, 如果集成了 Spring Security 但又没有自定义**登录页面**和**用户认证**, 则会输出默认的登录页面, 并使用此默认用户名和密码进行登录认证, 密码会随项目启动输出打印。

    默认的登录页面是由 **DefaultLoginPageGeneratingFilter** 过滤器处理，判断是否有自定义登录页面的URL路径, 不存在时则在该过滤器里拼接了 HTML 代码来输出页面。

2. SecurityAutoConfiguration 导入了 SpringBootWebSecurityConfiguration 类, 对于 SpringBootWebSecurityConfiguration 的使用，在 Spring Boot v1.5.x 版本和 2.0.x版本上存在较大差异, 详情请对比两个版本的源码。 

    SpringBootWebSecurityConfiguration 注入了 WebSecurityConfigurerAdapter 抽象类；自定义的安全配置继承 WebSecurityConfigurerAdapter 类。

3. SecurityFilterAutoConfiguration 创建 springSecurityFilterChain 实例,注册到 Spring 容器中。 

4. Spring Security 默认提交用户认证的路径是`/login`, 请求方式默认也只能是`POST`; 退出登录路径是`/logout`; 提交认证的用户名属性是`username`, 密码属性是`password`。用户认证是由 UsernamePasswordAuthenticationFilter 过滤器处理。

    Spring Security 的用户认证是通过**14个条件过滤器**组成的**过滤器链**来实现的, 14个过滤器是(WebAsyncManagerIntegrationFilter, SecurityContextPersistenceFilter, HeaderWriterFilter, CsrfFilter, LogoutFilter(退出登录过滤器), UsernamePasswordAuthenticationFilter, DefaultLoginPageGeneratingFilter(登录用户认证过滤器), BasicAuthenticationFilter, RequestCacheAwareFilter, SecurityContextHolderAwareRequestFilter, AnonymousAuthenticationFilter, SessionManagementFilter, ExceptionTranslationFilter, FilterSecurityInterceptor)

5. 封装用户认证的对象 authentication 的数据格式如下，可通过 `SecurityContextHolder.getContext().getAuthentication();` 来获取该对像实例。
  ``` json
  {
  	"authenticated": true,
  	"authorities": [{
  		"authority": "ADMIN"
  	}],
  	"details": {
  		"remoteAddress": "0:0:0:0:0:0:0:1",
  		"sessionId": "19CACF37586038F3DE14F808A891D483"
  	},
  	"name": "admin",
  	"principal": {
  		"accountNonExpired": true,
  		"accountNonLocked": true,
  		"address": "中国",
  		"age": 21,
  		"authorities": [{
  			"authority": "ADMIN"
  		}],
  		"credentialsNonExpired": true,
  		"enabled": true,
  		"id": 3,
  		"password": "{bcrypt}$2a$10$/.4eK1JTNF9h6jBzPh94ROgdgsj6KBVNAmg3I7pNBx1wWbckq97jG",
  		"role": "ADMIN",
  		"state": true,
  		"username": "admin"
  	}
  }
  ```

6. 默认提供一个基于 HTTP Basic 认证的安全防护策略，提供了默认的用户名和密码，也可通过以下属性设置：
  ``` properties
  #认证账号密码
  spring.security.user.name=admin
  spring.security.user.password=123456
  ```

7. 默认启用了一些必要的 Web 安全策略，比如针对 XSS、CSRF 等常见针对 Web 的攻击，同时，也会将一些常见的静态资源路径排除在安全防护之外。

### JSP标签库 ###

Spring Security 还提供了支持 JSP 的标签库，[Spring Security -> JSP Tag Libraries](https://docs.spring.io/spring-security/site/docs/5.0.7.RELEASE/reference/htmlsingle/#taglibs)
1. 导入标签库包
	``` xml
	<!--Spring Security 支持JSP的标签库-->
	<dependency>
	    <groupId>org.springframework.security</groupId>
	    <artifactId>spring-security-taglibs</artifactId>
	</dependency>
	```
2. 标签库使用
	在 JSP 文件顶部添加 taglib 支持:`<%@ taglib uri="http://www.springframework.org/security/tags" prefix="sec" %>`
	``` html
	<div>
		<h1>欢迎来到首页:
	        <%--从authentication中取值--%>
	        <sec:authentication property="principal.username" />
	
	        <%--从model中取值--%>
	        ${username}
	    </h1>
	
	    <a href="/admin">管理页面</a><br>
	    <a href="/user">USER页面</a><br>
	    <a href="/logout">退出登录</a>
	
	    <sec:authorize url="/admin">
	        <p>有权向 /admin 路径发送请求才可显示</p>
	    </sec:authorize>
	</div>
	
	<div>
	    <h1>USER, ADMIN 角色页面</h1>
	    <sec:authorize access="hasAuthority('ADMIN')">
	        <p>只有 ADMIN 角色可看</p>
	    </sec:authorize>
	
	    <sec:authorize access="hasAuthority('USER')">
	        <p>只有 USER 角色可看</p>
	    </sec:authorize>
	</div>
	```

### 集成示例 ###
1. 导入 Spring Security 依赖, 此示例使用 Spring Boot 2.0.2 Release版本
``` xml
<!--Spring Security-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```
2. WebConfig：Web配置
``` java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/login").setViewName("login");
//        registry.addViewController("/index").setViewName("index");
        registry.addViewController("/admin").setViewName("admin");
        registry.addViewController("/user").setViewName("user");
        registry.addViewController("/error").setViewName("error");
        registry.addViewController("/404").setViewName("404");
        //根目录默认定位到首页
        registry.addRedirectViewController("/","/index");
    }
}
```
3. WebSecurityConfig：Web安全配置
``` java
/**
 * @Name: WebSecurityConfig
 * @Desc: WebSecurity配置类
 **/
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private DataSource dataSource;

    /*@Autowired
    private CustomUserDetailsService customUserService;*/

    /**
     * 自定义请求授权
     *
     * @param httpSecurity
     * @throws Exception
     */
    @Override
    protected void configure(HttpSecurity httpSecurity) throws Exception {
        httpSecurity.csrf().disable()                                   //关闭CSRF防护
                .authorizeRequests()                                    //开启请求权限配置
//                    .antMatchers("/admin/**").hasRole("ADMIN")
//                    .antMatchers("/user/**").hasAnyRole("ADMIN","USER")
                    .antMatchers("/admin/**").hasAuthority("ADMIN")             //是否授权
                    .antMatchers("/user/**").hasAnyAuthority("ADMIN", "USER")
                    .antMatchers("/", "/login").permitAll()                     //匹配路径,允许任何人访问(不拦截,放行)
                    .anyRequest().authenticated()                               //其它请求,要求登录验证后可访问
                .and()
                    .formLogin()
                    .loginPage("/login")                //自定义登录
                    .defaultSuccessUrl("/index")        //登录成功后跳转url
                    .failureUrl("/error")               //登录失败跳转url(无法跳转到/error路径,提示无映射,实际有映射,换其它路径正常)
                    .permitAll()
                .and()
                    .rememberMe()                   //开启Cookie存储用户信息
                    .tokenValiditySeconds(604800)   //Cookie有效期
                    .key("myKey")                   //Cookie中的私钥
                .and()
                    .logout()                       //注销用户
                    .logoutUrl("/logout")           //注销用户url
                    .logoutSuccessUrl("/login")     //注解成功后跳转url
                    .permitAll()
                .and()
                .httpBasic();
    }

    @Bean
    public CustomUserDetailsService customUserDetailsService() {
        return new CustomUserServiceImpl();
    }

    /**
     * 用户认证方法
     *
     * @param auth
     * @throws Exception
     */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        /**
         * 1. 内置允许通过认证的用户
         * Spring Boot 2.0.x集成的是 Spring Security 5.0.x
         * 对认证的密码需要加密处理，否则会报错
         * */
        /*auth.inMemoryAuthentication().passwordEncoder(new BCryptPasswordEncoder())
                .withUser("tom").password(new BCryptPasswordEncoder().encode("123456")).roles("USER")
                .and()
                .withUser("kitty").password(new BCryptPasswordEncoder().encode("112233")).roles("USER");
        */

        /**
         * 2. JDBC用户
         * 默认的密码格式是: {id}encodedPassword, jdbc的用户密码必须以这种格式存储才能正确处理
         * 如 123456 使用 bcrypt 加密后存储的密码: {bcrypt}$2a$10$/.4eK1JTNF9h6jBzPh94ROgdgsj6KBVNAmg3I7pNBx1wWbckq97jG
         * 否则会报错误: There is no PasswordEncoder mapped for the id “null”
         */
        /*auth.jdbcAuthentication().dataSource(dataSource)
                .usersByUsernameQuery("select username, password, true from user where username = ?")
                .authoritiesByUsernameQuery("select username, role from user where username = ?");
        */

        /**
         * 3. 自定义通用用户
         */
        auth.userDetailsService(customUserDetailsService());
//        auth.userDetailsService(customUserService);

    }

    /**
     * 自定义Web安全策略
     *
     * @param webSecurity
     * @throws Exception
     */
    @Override
    public void configure(WebSecurity webSecurity) throws Exception {
        //设置忽略不拦截路径
        webSecurity.ignoring().antMatchers("/resources/static/**");
    }
}
```
4. SysUser：用户实体类
``` java
/**
 * @Name: SysUser
 * @Desc: 实现UserDetails接口,该用户实体类即为Spring Security 所使用的用户
 **/
@Entity
@Table(name = "user")
//public class SysUser{
public class SysUser implements UserDetails {           //实现 UserDetails,重写里面方法

    private static final long serialVersionUID = -1L;

    @Id
    @GeneratedValue
    private Long id;
    private String username;
    private String password;
    private String role;
    private String address;
    private Integer age;
    private Boolean state;

    //----get/set方法-----

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }

    /**
     * 重写getAuthorities方法,将用户角色做为权限
     * @return
     */
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        //添加授权
        List<GrantedAuthority> authorityList = new ArrayList<>();
        authorityList.add(new SimpleGrantedAuthority(getRole()));
        return authorityList;
    }
}
```
5. 控制器:IndexController
``` java
/**
 * @Name: IndexController
 * @Desc: 首页控制器
 **/
@Controller
public class IndexController {

    @RequestMapping("/index")
    public String indexPage(Model model){
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        logger.info("authentication:{}", JSON.toJSONString(authentication));
        //返回的是个用户实体
        SysUser sysUser = (SysUser) authentication.getPrincipal();
//        Object principal = authentication.getPrincipal();
//        Map<String, String> stringMap = ObjectToMapUtil.obj2Map(principal);
        model.addAttribute("username", sysUser.getUsername());
        return "index";
    }
}
```
6. 用户业务层接口:CustomUserDetailsService
``` java
import org.springframework.security.core.userdetails.UserDetailsService;

/**
 * @Name: CustomUserDetailsService
 * @Desc: 用户业务层接口
 **/

public interface CustomUserDetailsService extends UserDetailsService {
}
```
7. 用户业务实现:CustomUserServiceImpl
``` java
/**
 * @Name: CustomUserServiceImpl
 * @Desc: 用户业务处理
 **/
@Service
public class CustomUserServiceImpl implements CustomUserDetailsService {

    @Autowired
    private SysUserRepository sysUserRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Example<SysUser> example = Example.of(new SysUser().setUsername(username));
        SysUser sysUser = sysUserRepository.findOne(example).get();

//        List<GrantedAuthority> authorityList = new ArrayList<>();
//        authorityList.add(new SimpleGrantedAuthority(sysUser.getRole()));
//        return new User(sysUser.getUsername(), sysUser.getPassword(),authorityList);
        return  sysUser;
    }
}
```
8. 数据访问层:SysUserRepository
``` java
@Repository
public interface SysUserRepository extends JpaRepository<SysUser, Long> {
}
```
9. 登录页面：login
``` html
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/fmt" prefix="fmt" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/functions" prefix="fn" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<!DOCTYPE html>
<html>
<title>登录页面</title>
<head>
</head>

<body onload='document.f.username.focus();'>
    <h3>欢迎登录</h3>
    <form name='f' action='/login' method='POST'>
        <table>
            <tr>
                <td>User:</td>
                <td><input type='text' name='username' value=''></td>
            </tr>
            <tr>
                <td>Password:</td>
                <td><input type='password' name='password'/></td>
            </tr>
            <tr>
                <td colspan='2'><input name="submit" type="submit" value="Login"/></td>
            </tr>
            <%--<input name="_csrf" type="hidden" value="a029a020-6e8f-4bce-9155-23545f36275d"/>--%>
        </table>
    </form>
</body>

</html>
```
10. 首页
``` html
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/fmt" prefix="fmt" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/functions" prefix="fn" %>
<%@ taglib prefix="sec" uri="http://www.springframework.org/security/tags" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ page isELIgnored="false"%>

<!DOCTYPE html>
<html>
<title>首页</title>
<head>

</head>

<body>

<div>
    <h1>欢迎来到首页:
        <sec:authentication property="principal.username" />
        ---- ${username}
    </h1>

    <a href="/admin">管理页面</a><br>
    <a href="/user">USER页面</a><br>
    <a href="/logout">退出登录</a>

    <sec:authorize url="/admin">
        <p>有权向 /admin 路径发送请求才可显示</p>
    </sec:authorize>
</div>

</body>
</html>
```
11. 管理员角色页面
``` html
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/fmt" prefix="fmt" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/functions" prefix="fn" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ page isELIgnored="false"%>

<!DOCTYPE html>
<html>
<title>首页</title>
<head>

</head>

<body>

<div>
    <h1>ADMIN 角色页面</h1>
</div>

</body>
</html>
```
12. 普通用户角色页面
``` html
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/fmt" prefix="fmt" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/functions" prefix="fn" %>
<%@ taglib uri="http://www.springframework.org/security/tags" prefix="sec" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ page isELIgnored="false"%>


<!DOCTYPE html>
<html>
<title>首页</title>
<head>

</head>

<body>

<div>
    <h1>USER, ADMIN 角色页面</h1>
    <sec:authorize access="hasAuthority('ADMIN')">
        <p>只有 ADMIN 角色可看</p>
    </sec:authorize>

    <sec:authorize access="hasAuthority('USER')">
        <p>只有 USER 角色可看</p>
    </sec:authorize>
</div>

</body>
</html>
```
13. 错误页面
``` html
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/fmt" prefix="fmt" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/functions" prefix="fn" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ page isELIgnored="false"%>

<!DOCTYPE html>
<html>
<title>ERROR</title>
<head>

</head>

<body>

<div>
    <h1>不好意思,出错了！！！</h1>
</div>

</body>
</html>
```
14. 404页面
``` html
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/fmt" prefix="fmt" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/functions" prefix="fn" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ page isELIgnored="false"%>

<!DOCTYPE html>
<html>
<title>ERROR</title>
<head>

</head>

<body>

<div>
    <h1>不好意思,出错了！！！</h1>
</div>

</body>
</html>
```

## 拒绝访问错误页 ##
1. Java 配置
``` java
@Override
    protected void configure(HttpSecurity httpSecurity) throws Exception {
        http.authorizeRequests()
                .antMatchers("/admin/*")
                .hasAnyRole("ROLE_ADMIN")
                ...
                .and()
                .exceptionHandling()
                .accessDeniedPage("/my-error-page");
    }
```
2. 自定义AccessDeniedHandler
``` java
@Component
public class CustomAccessDeniedHandler implements AccessDeniedHandler {
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException ex) throws IOException, ServletException {
        response.sendRedirect("/my-error-page");
    }
}


@Autowired
private CustomAccessDeniedHandler accessDeniedHandler;

@Override
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
            .antMatchers("/admin/*")
            .hasAnyRole("ROLE_ADMIN")
            ...
            .and()
            .exceptionHandling()
            .accessDeniedHandler(accessDeniedHandler);
}
```
3. 自定义异常响应消息
``` java
@ControllerAdvice
public class RestResponseEntityExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler({AccessDeniedException.class})

    public ResponseEntity<Object> handleAccessDeniedException(Exception ex, WebRequest request) {

        return new ResponseEntity<Object>("Access denied message here", new HttpHeaders(), HttpStatus.FORBIDDEN);
    }
    ...
}
```

[源码 -> GitHub](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-spring-security)