---
title: Spring Boot 2实践系列(三十八)：全局异常统一处理
comments: true
date: 2019-01-30 09:35:51
tags: [exception,error]
categories: [Spring Boot 2实践系列]
---
　　Spring Boot 在默认情况下，提供了 `/error` 映射来处理所有错误，在 Servlet 容器里注册了全局的错误页面(**Whitelabel Error Page**)并返回客户端。

　　也可以自定义替换默认的异常处理, 通过实现 **ErrorController** 接口并注册为 **Bean**；或者添加 **ErrorAttributes** 类型的 **Bean** 来替换内容。

　　通常情况下，都会自定义全局异常统一处理，返回统一的消息结构体，便于了解和快速定位问题，Spring 提供了 **@ControllerAdvice** 注解，非常好实现。

　　Spring Boot 官方文档，错误处理：[28.1.11 Error Handling](https://docs.spring.io/spring-boot/docs/2.1.2.RELEASE/reference/htmlsingle/#boot-features-error-handling)

<!-- more -->
Spring Boot 自动配置还提供了实现 **ErrorController** 接口异常处理的基类 **BasicErrorController**，默认是处理 **text/html**类型请求的错误，可以继承该基类自定义处理更多的请求类型，添加公共方法并使用 **@RequestMapping** 注解的 **produce**属性指定处理类型。

还可以定义一个使用 **@ControllerAdvice** 注解的类，返回指定控制器的指定的异常类型的 JSON 格式的消息。

可了解[ SpringMVC注解之@ControllerAdvice ](http://112.74.59.39:90/%2F2018%2F03%2F31%2Fspringmvc-controllerAdvice%2F)，本篇示例下 **@ControllerAdvice** 注解 Controller 层异常的全局处理。

## 错误消息 ##
API 接口项目，异常处理统一返回 JSON 格式的消息。

1. 统一消息结构体
	``` java
	/**
	 * @desc: 统一响应消息结构体
	 * @date: 2019/1/30 11:26
	 **/
	public class ResultBean implements Serializable {
	
	    private static final long serialVersionUID = -8332309757143905140L;
	
	    private static final boolean SUCCESS = true;
	    private static final boolean FAIL = false;
	
	    private Boolean state;
	    private Integer code;
	    private String msg;
	    private Object data;
	
	    public ResultBean successResult(){
	        this.state = SUCCESS;
	        return this;
	    }
	
	    public ResultBean failResult(){
	        this.state = FAIL;
	        return this;
	    }
	
	    //.....set/get方法......
	}
	```
2. 全局异常处理类
	``` java
	/**
	 * @desc: 全局异常处理
	 * @date: 2019/1/30 10:29
	 **/
	@ControllerAdvice
	public class GlobalExceptionHandler {
	    private Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);
	
	    @ExceptionHandler(value = Exception.class)
	    @ResponseBody
	    public ResultBean defaultErrorHandler(HttpServletRequest req, Exception e) {
	        logger.error("", e);
	
	        ResultBean resultBean = new ResultBean();
	        resultBean.setMsg(e.getMessage());
	        if (e instanceof NoHandlerFoundException) {
	            resultBean.setCode(404);
	        } else {
	            resultBean.setCode(500);
	        }
	        resultBean.setState(false);
	        resultBean.setData(null);
	        return resultBean;
	    }
	}
	```
3. 在 application.properties 添加配置
	Spring MVC 开启抛出找不到映射处理的异常，关闭静态资源映射
	```
	spring.mvc.throw-exception-if-no-handler-found=true
	spring.resources.add-mappings=false
	```
4. 官方示例
	``` java
	@ControllerAdvice(basePackageClasses = AcmeController.class)
	public class AcmeControllerAdvice extends ResponseEntityExceptionHandler {
	
		@ExceptionHandler(YourException.class)
		@ResponseBody
		ResponseEntity<?> handleControllerException(HttpServletRequest request, Throwable ex) {
			HttpStatus status = getStatus(request);
			return new ResponseEntity<>(new CustomErrorType(status.value(), ex.getMessage()), status);
		}
	
		private HttpStatus getStatus(HttpServletRequest request) {
			Integer statusCode = (Integer) request.getAttribute("javax.servlet.error.status_code");
			if (statusCode == null) {
				return HttpStatus.INTERNAL_SERVER_ERROR;
			}
			return HttpStatus.valueOf(statusCode);
		}
	}
	```

## 错误页面 ##
还可以根据状态码来自定义 HTML 错误页面，将页面文件添加静态资源的 `error` 目录。

错误页面可以是静态 HTML (可以添加到静态资源目录)，也可以使用模板构建。文件名须以 HTTP 状态码或系列掩码命名(如：5xx)。
1. 示例：404 静态 HTML 页面
```
src/
 +- main/
     +- java/
     |   + <source code>
     +- resources/
         +- public/
             +- error/
             |   +- 404.html
             +- <other public assets>
```
2. 示例：系列掩码命名模板文件
```
src/
 +- main/
     +- java/
     |   + <source code>
     +- resources/
         +- templates/
             +- error/
             |   +- 5xx.ftl
             +- <other templates>
```
3. 对于列复杂的映射，还可以添加实现 ErrorViewResolver 接口的 Bean
``` java
public class MyErrorViewResolver implements ErrorViewResolver {

	@Override
	public ModelAndView resolveErrorView(HttpServletRequest request,
			HttpStatus status, Map<String, Object> model) {
		// Use the request or status to optionally return a ModelAndView
		return ...
	}
}
```

## 相关参考 ##
1. [SpringBoot拦截全局异常并发送邮件给指定邮箱](https://blog.csdn.net/tianyaleixiaowu/article/details/77868200)
2. [零代码如何打造自己的实时监控预警系统](https://yq.aliyun.com/articles/293209?spm=a2c4e.11153940.blogcont637783.17.4cd824b8EBmmV2)
3. [从零开始搭建ELK+GPE监控预警系统](https://yq.aliyun.com/articles/261432?spm=a2c4e.11153940.blogcont293209.23.1c0b3931VmYYYZ)
4. [Java 小应用日志级别异常处理最佳实践](https://mp.weixin.qq.com/s/h2Af6sRKFkCCCu4xbOxi5A)
5. [小应用日志和异常信息聚合开源方案 stackify-log-log4j2](https://github.com/stackify/stackify-log-log4j2)
6. [SpringBoot全局异常与数据校验](https://mp.weixin.qq.com/s/RIDJ4KYatasuV24dHacacg)