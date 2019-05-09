---
title: Spring Boot 2实践系列(二十一)：RestTemplate远程调用 REST 服务
comments: true
date: 2018-06-26 11:24:10
tags: [RestTemplate,WebClient,HttpClient]
categories: [Spring Boot 2实践系列]
---
　　互联网项目经常存在远程调用的情况,如果调用的是是`REST`远程服务，可以使用 Spring Framework 提供的`RestTemplate`, Spring Boot没有提供 **RestTemplate**的自动配置，需要自定义，但可以通过自动配置的`RestTemplateBuilder`来创建**RestTemplate**对象，并且 HttpMessageConverters 会自动应用到 RestTemplate 实例中。

　　**RestTemplateBuilder**的 `Bean` 创建和自动配置在 Spring Boot 自动配置包(org.springframework.boot:spring-boot-autoconfigure.2.0.0.RELEASE)里 Web 路径下：org.springframework.boot.autoconfigure.web.client.RestTemplateAutoConfiguration。

　　在使用`RestTemplate`之前, 必须先对`Rest`服务有个基本的了解, 否则在调用**RestTemplate**的方法时会存在困惑。可参考一个简单的[Restful Server](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-restful-service)示例项目。

　　如果是 `WebFlux` 项目，可以使用`WebClient`来远程调用**REST**服务，相比`RestTemplate`,**WebClient**拥有更多的功能，并且是完全响应式, 后续再对**WebClient**进行详解。

　　[Spring Boot 2.0.3 -> Calling REST Services with RestTemplate](https://docs.spring.io/spring-boot/docs/2.0.3.RELEASE/reference/htmlsingle/#boot-features-resttemplate)
　　[Spring Framework 5.7 -> RestTemplate](https://docs.spring.io/spring-framework/docs/5.0.7.RELEASE/spring-framework-reference/integration.html#rest-resttemplate)
<!-- more -->
## RestTemplate ##
RestTemplate 通过 HTTP client 库来提代供更高级别的 API, 可以轻松的调用远程`Rest`服务, 
RestTemplate 提供了多个内部重载的方法组，其中`xxxForObject`和`xxxForEntity`两组是用的比较多的，两者在使用上基本没有区别，区别是在返回的内容上：
**xxxForObject：**返回的消息只有响应体，没有响应头。
**xxxForEntity：**返回的消息中有包含响应头和响应体; 返回值是一个 ResponseEntity<T>，ResponseEntity<T>是Spring对HTTP请求响应的封装，包括了几个重要的元素，如响应码、contentType、contentLength、响应消息体等。

|       方法组       |      描述                             |
|:------------------|:--------------------------------------|
|getForObject       |GET 方式, 返回数据的类型是 Object，只有实体数据,不包含响应头数据|
|getForEntity       |GET 方式, 返回数据的类型是 ResponseEntity,包含状态码,响庆头,响应体数据|
|headForHeaders     |通过 HEAD 获取或设置头部信息                 |
|postForLocation    |POST 方式, 创建新资源并返回新资源的 URI |
|postForObject      |POST 方式, 创建新资源并返回响应体|
|postForEntity		|POST 方式, 创建新资源并返回 ResponseEntity<T>|
|put				|PUT 方式， 创建或更新资源|
|patchForObject     |更新资源并返回响应体, JDK自带的HttpURLConnection 不支持, 但Apache HttpComponents和其它支持|
|delete				|删除资源|
|optionsForAllow    |检查资源允许HTTP访问的请求方式|
|exchange           |较少使用，提供额外的操作，可接收RequestEntity，返回ResponseEntity|
|execute            |执行请求最常用的方式，通过回调接口完全控制请求前的准备工作和取出响应消息|

### 初始化 ###
RestTemplate 默认使用的是** java.net.HttpURLConnection **来执行请求，可以切换成不同的实现了**ClientHttpRequestFactory**接口的**HTTP 库**，如：Apache HttpComponents， Netty， OkHttp

示例：使用 Apache HttpComponents
``` java
RestTemplate template = new RestTemplate(new HttpComponentsClientHttpRequestFactory());
```

每个**ClientHttpRequestFactory**公开的配置项都是基于 HTTP Client 底层库的支持，如：凭据，连接池等。

**注意：** java.net 实现的 HTTP请求可能在访问表示错误的响应状态时会引发异常，如：401。如果是这个问题，切换到其它的 HTTP Client库。

### URIs ###
``` java
Many of the RestTemplate methods accepts a URI template and URI template variables, either as a String vararg, or as Map<String,String>.

接收 URI模板和字符串参数，或 Map<String,String>。
String result = restTemplate.getForObject("http://example.com/hotels/{hotel}/bookings/{booking}", String.class, "42", "21");
Map<String, String> vars = Collections.singletonMap("hotel", "42");
String result = restTemplate.getForObject("http://example.com/hotels/{hotel}/rooms/{hotel}", String.class, vars);
```

2. **注意：**URI templates are automatically encoded（自动编码），如：
``` java
restTemplate.getForObject("http://example.com/hotel list", String.class);
Results in request to "http://example.com/hotel%20list"  （实际请求连接）
```

3. 也可以使用 RestTemplate 的 uriTemplateHandler属性来自定义 URI 的编码方式，或创建一个 java.net.URI 传给接收 URI 的 RestTemplate方法。更多 [URI编码请详见](https://docs.spring.io/spring-framework/docs/5.0.7.RELEASE/spring-framework-reference/web.html#mvc-uri-building)。

### headers ###
可以使用`exchange()`方法来指定请求头信息，如下：
``` java
String uriTemplate = "http://example.com/hotels/{hotel}";
URI uri = UriComponentsBuilder.fromUriString(uriTemplate).build(42);

RequestEntity<Void> requestEntity = RequestEntity.get(uri)
        .header(("MyRequestHeader", "MyValue")
        .build();

ResponseEntity<String> response = template.exchange(requestEntity, String.class);

String responseHeader = response.getHeaders().getFirst("MyResponseHeader");
String body = response.getBody();
```
通过 RestTemplate 灵活的方法，可以让返回的 ResponseEntity 会包含响应头。

### body ###
传入并从 RestTemplate 方法返回的对象，是通过 HttpMessageConverter 使原始内容和对象之间实现相互转换。
在一个 **POST** 请求中，一个输入对象被序列化到请求体中：
``` java
URI location = template.postForLocation("http://example.com/people", person);
```

默认情况下，可以不显式设置请求的`Content-Type`, 底层会根据**源对象类型**找到适配的消息转换器来设置 **Content-Type**，若有特殊需求，可以使用`exchange`方法来显式设置**Content-Type**, 相当于显式指定了消息转换器。
在一个**GET**请求中，响应体被反序列化为输出对象。
``` java
Person person = restTemplate.getForObject("http://example.com/people/{id}", Person.class, 42);
```

## RestTemplateBuilder ##
### build ###
RestTemplateBuilder 包含多个有用的方法来快速的配置 RestTemplate ，如添加 BASIC auth 支持，可以使用：`builder.basicAuthorization("user", "password").build();`

RestTemplate默认会注册所有内置的消息转换器,具体的选择取决于类路径下有那些转换库可被选择使用。可以人为显式设置消息转换器。
``` java
@Service
public class MyService {

	private final RestTemplate restTemplate;

	public MyService(RestTemplateBuilder restTemplateBuilder) {
		this.restTemplate = restTemplateBuilder.build();
	}

	public Details someRestCall(String name) {
		return this.restTemplate.getForObject("/{name}/details", Details.class, name);
	}
}
```

### 自定义 ###
Spring Boot提供了RestTemplateBuilder 的自动配置, 几乎不需要通过自定义来创建 RestTemplateBuilder, 如果自定义则会关闭RestTemplateBuilder 的自动配置。

示例：为除 192.168.0.5 之外的所有主机配置代理请求。
``` java
static class ProxyCustomizer implements RestTemplateCustomizer {

	@Override
	public void customize(RestTemplate restTemplate) {
		HttpHost proxy = new HttpHost("proxy.example.com");
		HttpClient httpClient = HttpClientBuilder.create()
				.setRoutePlanner(new DefaultProxyRoutePlanner(proxy) {

					@Override
					public HttpHost determineProxy(HttpHost target,
							HttpRequest request, HttpContext context)
							throws HttpException {
						if (target.getHostName().equals("192.168.0.5")) {
							return null;
						}
						return super.determineProxy(target, request, context);
					}

				}).build();
		restTemplate.setRequestFactory(
				new HttpComponentsClientHttpRequestFactory(httpClient));
	}
}
```

## RestTemplate示例 ##
1. RestTemplateConfig
``` java
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate customRestTemplate(){

	//设置连接和读超时时间,毫秒
	// 2.1.0 版本方式
	return new RestTemplateBuilder().setConnectTimeout(Duration.ofMillis(1000)).setReadTimeout(Duration.ofMillis(1000)).build();
	// 2.0.x 版本方式
	//return new RestTemplateBuilder().setConnectTimeout(1000).setReadTimeout(1000).build();
    }
}
```
2. RestTemplate方法调用
``` java
import com.alibaba.fastjson.JSON;
import com.springboot.rest.template.config.RestServerProperties;
import com.springboot.rest.template.entity.City;
import com.springboot.rest.template.service.CallRemoteService;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClientException;
import org.springframework.web.client.RestTemplate;

import java.net.URI;

@Service
public class CallRemoteServiceImpl implements CallRemoteService {

    private static final Logger logger = LogManager.getLogger(CallRemoteServiceImpl.class);

    @Autowired
    private RestServerProperties restServerProperties;

    @Autowired
    private RestTemplate restTemplate;

    /**
     * getForEntity()
     * 获取数据-所有
     * 包含响应头,响应体数据
     */
    @Override
    public String getForEntityForAll() {
        String uri = "http://localhost:8080/city";
        ResponseEntity<Object> actorResponseEntity = null;
        try {
            actorResponseEntity = restTemplate.getForEntity(uri, Object.class);
        } catch (RestClientException e) {
            e.printStackTrace();
        }
        String cityListStr = JSON.toJSONString(actorResponseEntity);
        return cityListStr;
    }

    /**
     * getForObject()
     * 返回的只有实体数据,没有响应头的数据
     */
    @Override
    public String getForObjectForAll() {
        String uri = restServerProperties.getUrl() + "/city";
        Object object = restTemplate.getForObject(uri, Object.class);
        String cityListStr = JSON.toJSONString(object);
        logger.info("cityListStr:{}" + cityListStr);
        return cityListStr;
    }


    /**
     * getForEntity()
     * 获取数据-参数
     */
    public City getForEntityByCityId(Long cityId) {
        String uri = "http://localhost:8080/city/" + cityId;
        ResponseEntity<Object> responseEntity = restTemplate.getForEntity(uri, Object.class);

        //Body是LinkHashMap类型
        /*Object body = responseEntity.getBody();
        String botyStr = JSON.toJSONString(body);
        City city = JSON.parseObject(botyStr, City.class);*/

        //从响应体中拿取实体类数据
        City city = JSON.parseObject(JSON.toJSONString(responseEntity.getBody()), City.class);

        return city;
    }

    /**
     * postForLocation()
     * 添加数据
     */
    public String postForLocationForCity(City city) {
        String uriStr = null;
        HttpHeaders headers = new HttpHeaders();
        MediaType type = MediaType.parseMediaType("application/json; charset=UTF-8");
        headers.setContentType(type);
        headers.add("Accept", MediaType.APPLICATION_JSON.toString());

        HttpEntity<String> formEntity = new HttpEntity<String>(JSON.toJSONString(city), headers);

        String url = restServerProperties.getUrl() + "/city";

        //rest服务中Controller方法使用对象无法接收参数
//        URI uri = restTemplate.postForLocation(url, city);
        //rest服务中Controller方法中 @RequestBody 注解不支持默认的text/plain;charset=ISO-8859-1
//        URI uri = restTemplate.postForLocation(url, JSON.toJSONString(city));

        //1.只能发送对象字符串,否则Rest服务接收不到参数
        //2.字符串默认数据类型和编码是text/plain;charset=ISO-8859-1,
        // 调用的Rest服务的Controller方法中使用 @RequestBody 解析参数封装到对象中,
        // 而@RequestBody注解解析的是JSON数据,所以需要设置消息头告诉数据类型和编码
        URI uri = restTemplate.postForLocation(url, formEntity);
        if(uri != null){
            uriStr = JSON.toJSONString(uri);
        }
        return uriStr;
    }

    /**
     * postForEntity()
     * 添加数据
     */
    public String postForEntityForCity(City city) {
        String url = restServerProperties.getUrl() + "/city";
        ResponseEntity<City> cityResponseEntity = restTemplate.postForEntity(url, city, City.class);
        String str = JSON.toJSONString(cityResponseEntity);
        return str;
    }

    /**
     * postForObject()
     * 添加数据
     */
    public String postForObjectForCity(City city) {
        String url = restServerProperties.getUrl() + "/city";
        City city1 = restTemplate.postForObject(url, city, City.class);
        String str = JSON.toJSONString(city1);
        return str;
    }

    /**
     * put()
     * 更新数据
     */
    public void putForCity(Long cityId, String cityName) {
        String url = restServerProperties.getUrl() + "/city" + "/" + cityId + "/" + cityName;
        HttpHeaders headers = new HttpHeaders();
        MediaType type = MediaType.parseMediaType("application/json; charset=UTF-8");
        headers.setContentType(type);
        headers.add("Accept", MediaType.APPLICATION_JSON.toString());
        HttpEntity<String> formEntity = new HttpEntity<String>(JSON.toJSONString(
                new City().setCityId(cityId).setCityName(cityName)), headers);
        restTemplate.put(url,null);
    }

    /**
     * delete删除
     * @param cityId
     */
    @Override
    public void deleteById(Long cityId) {
        String url = restServerProperties.getUrl() + "/city" + "/" + cityId;
        restTemplate.delete(url);
    }

}
```
[示例源码 -> GitHub](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-restTemplate)