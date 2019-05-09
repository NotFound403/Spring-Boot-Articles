---
title: Spring Boot 2实践系列(三十七)：集成 Swagger2 生成 API 文档
comments: true
date: 2018-12-31 19:19:10
tags: [swagger,api]
categories: [Spring Boot 2实践系列]
---
　　**Swagger**官方宣称是最好的**API**文档构建工具，也确实好用，易用。Swagger 生成的 API 接口文档是 **Restful** 风格的，会自动识别传参方式(path , query , body )，参数类型等；会自动识别允许的请求方式，若接口没有限制请求方式，Swagger 则会生成 **Restful** 风格的所有请求(get,post,put,patch,options,head,delete)。

　　项目的 **API** 接口需要更新，需要维护，直接通过 **Swagger** 工具生成非常的方便，不需要维护一个单独的**API**文档。单独的**API**文档非常容易出错和遗漏，特别是时间长了修改的人多的情况下，更是难以保证准确性。

　　[Swagger GitHub](https://github.com/swagger-api)，[Swagger 官网](https://swagger.io/)，[SpringFox GitHub](https://github.com/springfox/springfox)
<!-- more -->
下面汇总了集成 Swagger 三种方式，集成都非常简单，根据所需和习惯选择。

## 方式一：springfox-swagger ##
### 添加依赖包 ###
添加`swagger`依赖和`swagger-ui`包
``` xml
<!--Swagger-->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.9.9</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.9.2</version>
</dependency>
```
**springfox-swagger2**最新的依赖包版本是`2.9.2`，在此最新版本上使用`@ApiImplicitParam`注解指定接口参数类型是数字时(**Long，Integer**)，会报：**java.lang.NumberFormatException: For input string: ""**的错误。
解决此问题可通过降一个版本到`2.8.0`解决，也可替换里面的依赖包，手动导入解决，如下：
``` xml
<!--Swagger-->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.9.2</version>
	<!--排除-->
    <exclusions>
        <exclusion>
            <groupId>io.swagger</groupId>
            <artifactId>swagger-annotations</artifactId>
        </exclusion>
        <exclusion>
            <groupId>io.swagger</groupId>
            <artifactId>swagger-models</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.9.2</version>
</dependency>
<!--手动导入-->
<dependency>
    <groupId>io.swagger</groupId>
    <artifactId>swagger-annotations</artifactId>
    <version>1.5.21</version>
</dependency>
<dependency>
    <groupId>io.swagger</groupId>
    <artifactId>swagger-models</artifactId>
    <version>1.5.21</version>
</dependency>
```

### 添加 Swagger 配置 ###
```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {
    //swagger2的配置文件，这里可以配置swagger2的一些基本的内容，比如扫描的包等等
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                //为当前包路径
                .apis(RequestHandlerSelectors.basePackage("com.xxx.controller"))
                .paths(PathSelectors.any())
                .build();
    }
    //构建 api文档的详细信息函数,注意这里的注解引用的是哪个
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                //页面标题
                .title("xxx project title")
                //创建人
                .contact(new Contact("your name", "url", "email"))
                //版本号
                .version("1.0")
                //描述
                .description("xxx project description")
                .build();
    }
}
```

做到这里，实际就可以编写`Controller`层接口，运行程序访问接口文档了：http://localhost:8080/swagger-ui.html

可以使用**Swagger**注解对实体类、属性、Controller 类、**API** 接口方法、接口参数进行注释，在阅读生成的 **API** 文档时可以更直观的理解；也可以不用任务注解，**Swagger** 也能生成 **API** 文档。

此方式访问 **API** 文档是在项目根目下，若使用了 `Shiro` 进行**认证**和**权限**管理，则需要登录才可访问，若想不需要登录提供开放访问，则需要做如下配置。

### Shiro放行swagger-ui资源 ###
``` java
@Configuration
public class ShiroConfig {
    private static final Logger logger = LogManager.getLogger(ShiroConfig.class);

    @Bean
    public ShiroFilterFactoryBean shirFilter(SecurityManager securityManager) {
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();

        // 必须设置 SecurityManager
        shiroFilterFactoryBean.setSecurityManager(securityManager);

        // 如果不设置默认会自动寻找Web工程根目录下的"/login.jsp"页面
        shiroFilterFactoryBean.setLoginUrl("/login");
        // 登录成功后要跳转的链接
        shiroFilterFactoryBean.setSuccessUrl("/index");
        // 未授权界面;
        shiroFilterFactoryBean.setUnauthorizedUrl("/error");

        // 过滤器.配置不被拦截路径
        Map<String, String> filterChainDefinitionMap = new LinkedHashMap<String, String>();
        filterChainDefinitionMap.put("/","anon");
        filterChainDefinitionMap.put("/doLogin","anon");
        filterChainDefinitionMap.put("/login", "anon");
        // 放行swagger-ui资源
        filterChainDefinitionMap.put("/swagger-ui.html", "anon");
        filterChainDefinitionMap.put("/swagger-resources/**", "anon");
        filterChainDefinitionMap.put("/v2/api-docs", "anon");
        filterChainDefinitionMap.put("/webjars/springfox-swagger-ui/**", "anon");

        filterChainDefinitionMap.put("/**", "authc");

        shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
        return shiroFilterFactoryBean;
    }
}
```

此集成方式访问的是项目根目录下的`swagger-ui`资源，不好修改访问**AIP**文档的`URI`路径，在本地开发环境倒没什么问题，若项目在测试环境或生产环境配置了使用了`Nginx`对某个`URI`路径进行代理跳转的，而又要看线上环境的**API**接口文档，此方式则不能满足。可使用下面的**方式二**配置。

## 方式二：可修改API文档路径 ##
前端访问后端服务的接口，特别是前后端分离项目，往往会通过 **Nginx** 对某个路径进行代理跳转，也就需要对` API `文档路径进行自定义，若项目使用了 `Shiro`，可能需要对 `API` 文档路径放行，自定义路径也更好操作。
### 添加依赖包 ###
``` xml
<!--springfox-swagger2-->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.8.0</version>
</dependency>
```
这里用的是 **2.8.0** 的版本，而最新的是 **2.9.2 **，若使用最新的版，在`@ApiImplicitParam`注解指定接口参数类型是数字时(**Long，Integer**)，会报：**java.lang.NumberFormatException: For input string: ""**的错误，并且解决此问题的处理不适合此种集成方式，解决方案见文底**【相关参考】**。

### 添加 swagger-ui ###
要修改 **Swagger** 生成的 **API** 文档的访问路径，需要使用原生的`swagger-ui`。
1. 从 GitHub 下载 `swagger-ui`，版本介于**v2.0.0**与**v3.0.0**之间，最大的版本是[swagger-ui v2.2.10](https://github.com/swagger-api/swagger-ui/tree/v2.2.10)。
2. 将下载包中的`dist`目录复制到 **Spring Boot** 工程的 **/resources/swagger**目录下，没有 **swagger** 目录则创建该目录，最终目录是**/resources/swagger/dist**
3. 修改访问路径，打开`dist`目录下的 **index.html** 文件，做如下修改
	``` html
	<script type="text/javascript">
	    $(function () {
	      var url = window.location.search.match(/url=([^&]+)/);
	      if (url && url.length > 1) {
	        url = decodeURIComponent(url[1]);
	      } else {
	        url = "http://petstore.swagger.io/v2/swagger.json";
	      }
	```
	改成：
	```
	<script type="text/javascript">
	    $(function () {
	      var url = window.location.search.match(/url=([^&]+)/);
	      if (url && url.length > 1) {
	        url = decodeURIComponent(url[1]);
	      } else {
			/*这里*/
	        url = "/open/api/doc";
	      }
	
	```
	API 文档的 **URI** 路径是`/open/api/doc/index.html`

### 添加 Swagger 配置 ###
1. 在 **resources** 下创建 **swagger.properties**文件，在文件中添加如下配置。
> springfox.documentation.swagger.v2.path=/open/api/doc

2. 创建 **swagger** 配置文件
	``` java
	package com.leopard.config;
	
	import org.springframework.context.annotation.Bean;
	import org.springframework.context.annotation.Configuration;
	import org.springframework.context.annotation.PropertySource;
	import springfox.documentation.builders.ApiInfoBuilder;
	import springfox.documentation.builders.PathSelectors;
	import springfox.documentation.builders.RequestHandlerSelectors;
	import springfox.documentation.service.ApiInfo;
	import springfox.documentation.spi.DocumentationType;
	import springfox.documentation.spring.web.plugins.Docket;
	import springfox.documentation.swagger2.annotations.EnableSwagger2;
	
	@Configuration
	@EnableSwagger2
	@PropertySource("classpath:swagger.properties")
	public class SwaggerConfig {
	    //swagger2的配置文件，这里可以配置swagger2的一些基本的内容，比如扫描的包等等
	    @Bean
	    public Docket createRestApi() {
	        return new Docket(DocumentationType.SWAGGER_2)
	                .apiInfo(apiInfo())
	                .select()
	                //为当前包路径
	                .apis(RequestHandlerSelectors.basePackage("com.xxx.controller"))
	                .paths(PathSelectors.any())
	                .build();
	    }
	    //构建 api文档的详细信息函数,注意这里的注解引用的是哪个
	    private ApiInfo apiInfo() {
	        return new ApiInfoBuilder()
	                //页面标题
	                .title("xxx 项目API文档")
	                //创建人
	                .contact(new Contact("your name", "url", "your email"))
	                //版本号
	                .version("1.0")
	                //描述
	                .description("xxx 项目管理后台接口")
	                .build();
	    }
	
	}
	```
3. 添加 **API URI** 路径映射到静态资源目录
	``` java
	@Configuration
	public class WebConfig implements WebMvcConfigurer {
	
	    @Override
	    public void addResourceHandlers(ResourceHandlerRegistry registry) {
	        registry.addResourceHandler("/open/api/doc/**").addResourceLocations("classpath:/swagger/dist/");
	    }
	}
	```

### Shiro 放行 API URI ###
项目若使用了 `Shiro` 进行认证和权限管理，需要对 `API` 文档路径放行时可添加配置：`filterChainDefinitionMap.put("/open/**", "anon");`
``` java
@Configuration
public class ShiroConfig {
    private static final Logger logger = LogManager.getLogger(ShiroConfig.class);

    @Bean
    public ShiroFilterFactoryBean shirFilter(SecurityManager securityManager) {
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();

        // 必须设置 SecurityManager
        shiroFilterFactoryBean.setSecurityManager(securityManager);

        // 如果不设置默认会自动寻找Web工程根目录下的"/login.jsp"页面
        shiroFilterFactoryBean.setLoginUrl("/login");
        // 登录成功后要跳转的链接
        shiroFilterFactoryBean.setSuccessUrl("/index");
        // 未授权界面;
        shiroFilterFactoryBean.setUnauthorizedUrl("/error");

        // 拦截器.配置不被拦截路径
        Map<String, String> filterChainDefinitionMap = new LinkedHashMap<String, String>();
        filterChainDefinitionMap.put("/","anon");
        filterChainDefinitionMap.put("/doLogin","anon");
        filterChainDefinitionMap.put("/login", "anon");
		// API 文档路径
        filterChainDefinitionMap.put("/open/**", "anon");
        
        filterChainDefinitionMap.put("/**", "authc");


        shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
        return shiroFilterFactoryBean;
    }
}
```


### 创建 API 接口并访问 ###
1. 在`Controller`创建接口
``` java
@RestController
@RequestMapping(value = "/user")
@Api(value = "用户API")
public class UserController {

    @Autowired
    private UserService userService;

    /**
     * @desc: 分页查询
     * @return: ResultBean
     **/
    @RequestMapping(value = "/queryAll", method = RequestMethod.GET)
    @ApiOperation(value = "查询用户列表", notes = "分页查询")
    public ResultBean queryByPage(UserVo userVo) {
        List<User> userList = userService.queryByPage(userVo);
        return new ResultBean(userList).setSuccessCodeAndState();
    }
}
```
2. 运行应用，访问 `API` 文档
访问路径：http://localhost:8080/open/api/doc/index.html

## 方式三：使用 swagger-boot ##
上面的**方式一、方式二**已基本可满足需求了，但基于 Spring Boot框架的集，有些大神在原始**Swagger**基础上创建了`swagger-boot`包，更加方便。
1. 由 SpringForAll 社区提供在[spring-boot-starter-swagger](https://github.com/SpringForAll/spring-boot-starter-swagger)。
	只要导入这个包即可始用，更详细的配置参考 GitHub 上的说明。
2. 还有款[swagger-spring-boot](https://github.com/battcn/swagger-spring-boot)工具,
	这款工作和`spring-boot-starter-swagger`极其相似，但作者自定义了前端页面，根据个人是否习惯选择。只要导入包即可始用，更详细的配置参考 GitHub 上的说明。

## Swagger 常用注解使用 ##
Swagger 提供了多种注解来对接口进行注释，非常方便。
### 接口层注解 ###
Swagger 接口层注解指作用在**Controller**类上或里面的接口方法上的注解。
1. **@Api()**
	作用在**Controller**类上，标识这个类是个**Swagger**资源
2. **@ApiOperation()**
	作用在接口方法上，表示这是一个 HTTP 方法
3. **@ApiParam**
	作用在接口方法上，对接口参数添加元素据，如参数名,说明,是否必填等。只适用于`JAX-RS`环境。
4. **@ApiImplicitParam()**
	作用在接口方法上，对接口参数进注释定义，支持 `Servlet` 和非 `JAX-RS` 环境使用。
5. **@ApiImplicitParams()**
	作用在接口方法上，括号里是多个**@ApiImplicitParam()**注解。
6. **@ApiIgnore()**
	作用在**Controller**类或方法上，指定 **Swagger** 忽略此类和方法。

### 实体类注解 ###
2. **@ApiModel()**
	作用在实例类上，对实体类进行说明，表示这个一个 Swagger models，即该实例类会被用于接口方法接收参数。
3. **@ApiModelProperty()**
	作用在属性或set/get 方法上，对属性添加注释说明


## 相关参考 ##
[Documenting your REST API with Swagger and Springfox](https://g00glen00b.be/documenting-rest-api-swagger-springfox/)
[Spring Boot中使用Swagger2构建强大的RESTful API文档](https://www.jianshu.com/p/8033ef83a8ed)