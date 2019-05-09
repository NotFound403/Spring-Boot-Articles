---
title: Spring Boot 2实践系列(四十二)：源码分析自动配置之编码过滤器
comments: true
date: 2019-04-28 08:47:28
tags: [filter,autoconfig]
categories: [Spring Boot 2实践系列]
---
　　在 *SSM* 框架中，通常会在 *web.xml* 中配置编码过滤器 *CharacterEncodingFilter*，但在 Spring Boot 应用中却没有要求人为配置编码过滤器，是因为 Spring Boot 基于 *习惯优于配置*  的原则，默认情况下自动配置了编码过滤器，采用的 *UTF-8* 编码。

　　本篇分析 *Spring Boot* 的编码过滤器的自动配置，也更详细的理解和体会 *Spring Boot* 自动配置的使用。

<!-- more -->
在 *spring-boot-start* 包默认依赖了 *spring-boot-aotuconfig* 包，*Spring Boot* 默认支持的组件的自动配置全在 *spring-boot-aotuconfig* 包里面，只要引入这些组件就可以基于很少的配置来使用。

## HTTP编码属性配置 ##
自动配置少不了属性配置类，在 *org.springframework.boot.autoconfigure.http* 包下有个 *HttpEncodingProperties* 的属性配置类，原码及解读如下：

``` java
// //支持 spring.http.encoding 属性设置编码
@ConfigurationProperties(prefix = "spring.http.encoding")
public class HttpEncodingProperties {

	public static final Charset DEFAULT_CHARSET = StandardCharsets.UTF_8;
	// HTTP请求和响应默认编码UTF-8
	private Charset charset = DEFAULT_CHARSET;
	// HTTP请求和响应是否强制编码，相当于xml文件中编码过滤器参数forceEncoding=true
	private Boolean force;
	// HTTP请求是否强制编码，当 force 未指定时为 true(该值是进过判断后的默认值)
	private Boolean forceRequest;
	// HTTP响应是否强制编码
	private Boolean forceResponse;

	private Map<Locale, Charset> mapping;

	public Charset getCharset() {
		return this.charset;
	}

	public void setCharset(Charset charset) {
		this.charset = charset;
	}

	public boolean isForce() {
		return Boolean.TRUE.equals(this.force);
	}

	public void setForce(boolean force) {
		this.force = force;
	}

	public boolean isForceRequest() {
		return Boolean.TRUE.equals(this.forceRequest);
	}

	public void setForceRequest(boolean forceRequest) {
		this.forceRequest = forceRequest;
	}

	public boolean isForceResponse() {
		return Boolean.TRUE.equals(this.forceResponse);
	}

	public void setForceResponse(boolean forceResponse) {
		this.forceResponse = forceResponse;
	}

	public Map<Locale, Charset> getMapping() {
		return this.mapping;
	}

	public void setMapping(Map<Locale, Charset> mapping) {
		this.mapping = mapping;
	}
	// force 默认未赋初始值而为 null, 编码过滤器在调用此方法时，传入了请求类型进行比较判断
    // 请求类型为 Type.REQUEST 时，force 为 true，即为请求强制编码
    // 请求类型为 Type.RESPONSE 时，force 为 false，即响应不强制编码
	public boolean shouldForce(Type type) {
		Boolean force = (type != Type.REQUEST ? this.forceResponse : this.forceRequest);
		if (force == null) {
			force = this.force;
		}
		if (force == null) {
			force = (type == Type.REQUEST);
		}
		return force;
	}

	public enum Type {
		REQUEST, RESPONSE
	}
}
```

也可在 *application.properties* 配置文件使用 *spring.http.encoding* 前缀指定编码格式：
```properties
spring.http.encoding.charset=UTF-8
spring.http.encoding.force=true
```

## HTTP编码自动配置 ##
编码过滤器的类 *CharacterEncodingFilter* 还是要用到的，自动配置类里注入了属性配置类，用于给编码过滤器设置参数并将 *CharacterEncodingFilter* 注册为 *Bean*。
*Http* 编码自动配置类在 *org.springframework.boot.autoconfigure.web.servlet* 包下。

``` java
// 配置类
@Configuration		
// 开启属性配置	
@EnableConfigurationProperties(HttpEncodingProperties.class)
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
// 当CharacterEncodingFilter在类路径的条件下则注册Bean
@ConditionalOnClass(CharacterEncodingFilter.class)	
// 当spring.http.encoding的值为enabled，如果没有设置则默认为true,即条件符合
@ConditionalOnProperty(prefix = "spring.http.encoding", value = "enabled", matchIfMissing = true)
public class HttpEncodingAutoConfiguration {

	private final HttpEncodingProperties properties;
	// 将属性配置作为自动配置构造方法的参数传入
	public HttpEncodingAutoConfiguration(HttpEncodingProperties properties) {
		this.properties = properties;
	}

	@Bean
    // 如果CharacterEncodingFilter Bean不存在则创建
	@ConditionalOnMissingBean(CharacterEncodingFilter.class)	
	public CharacterEncodingFilter characterEncodingFilter() {
        // 创建了字符编码过滤器对象
		CharacterEncodingFilter filter = new OrderedCharacterEncodingFilter();
		// 过滤器编码参数，调用的是属性配置类的属性值
		filter.setEncoding(this.properties.getCharset().name());
		filter.setForceRequestEncoding(this.properties.shouldForce(Type.REQUEST));
		filter.setForceResponseEncoding(this.properties.shouldForce(Type.RESPONSE));
		return filter;
	}
｝
```

## 字符编码过滤器

``` java
public class CharacterEncodingFilter extends OncePerRequestFilter {

	@Nullable
	private String encoding;

	private boolean forceRequestEncoding = false;

	private boolean forceResponseEncoding = false;

	public CharacterEncodingFilter() {
	}

	public CharacterEncodingFilter(String encoding) {
		this(encoding, false);
	}

	public CharacterEncodingFilter(String encoding, boolean forceEncoding) {
		this(encoding, forceEncoding, forceEncoding);
	}

	public CharacterEncodingFilter(String encoding, boolean forceRequestEncoding, boolean forceResponseEncoding) {
		Assert.hasLength(encoding, "Encoding must not be empty");
		this.encoding = encoding;
		this.forceRequestEncoding = forceRequestEncoding;
		this.forceResponseEncoding = forceResponseEncoding;
	}

	public void setEncoding(@Nullable String encoding) {
		this.encoding = encoding;
	}

	@Nullable
	public String getEncoding() {
		return this.encoding;
	}

	public void setForceEncoding(boolean forceEncoding) {
		this.forceRequestEncoding = forceEncoding;
		this.forceResponseEncoding = forceEncoding;
	}

	public void setForceRequestEncoding(boolean forceRequestEncoding) {
		this.forceRequestEncoding = forceRequestEncoding;
	}

	public boolean isForceRequestEncoding() {
		return this.forceRequestEncoding;
	}

	public void setForceResponseEncoding(boolean forceResponseEncoding) {
		this.forceResponseEncoding = forceResponseEncoding;
	}

	public boolean isForceResponseEncoding() {
		return this.forceResponseEncoding;
	}

    // 在此方法中对 request 和 response 执行编码，此方法是核心
	@Override
	protected void doFilterInternal(
			HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {

		String encoding = getEncoding();
		if (encoding != null) {
			if (isForceRequestEncoding() || request.getCharacterEncoding() == null) {
				request.setCharacterEncoding(encoding);
			}
			if (isForceResponseEncoding()) {
				response.setCharacterEncoding(encoding);
			}
		}
		filterChain.doFilter(request, response);
	}
}
```

