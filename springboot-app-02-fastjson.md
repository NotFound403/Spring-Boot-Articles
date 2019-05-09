---
title: Spring Boot 2实践系列(二)：FastJson集成和使用
comments: true
date: 2018-05-19 9:44:16
tags: [fastjson]
categories: [Spring Boot 2实践系列]
---
　　FastJson　是阿里巴巴的开源　**JSON**　解析库，它可以解析　**JSON**　格式的字符串，支持将　Java Bean　序列化为　JSON　字符串，也可以从　JSON　字符串反序列化到　JavaBean。

　　FastJson 速度快、功能完备、API简洁。
<!-- more -->
## 集成FastJson ##
Spring Boot 已自带了 Jackson 来帮助序列化和反序列化，集成 FastJson 替换掉 Jackson 也非常简单，只要在 Java 配置类中注入 HttpMessageConverters Bean即可,如下：
1. Java配置文件注入HttpMessageConverters Bean,该Bean中添加了 FastJson
``` java
@Configuration
public class MyWebConfig implements WebMvcConfigurer {

    @Bean
    public HttpMessageConverters fastJsonConfigure(){
        FastJsonHttpMessageConverter converter = new FastJsonHttpMessageConverter();
        FastJsonConfig fastJsonConfig = new FastJsonConfig();
        fastJsonConfig.setSerializerFeatures(SerializerFeature.PrettyFormat);
//		fastJsonConfig.setCharset(Charset.forName("UTF-8"));
        //日期格式化
        fastJsonConfig.setDateFormat("yyyy-MM-dd HH:mm:ss");
        converter.setFastJsonConfig(fastJsonConfig);

		//处理中文乱码问题 与@JSONField注解冲突
//        List<MediaType> fastMediaTypes = new ArrayList<>();
//        fastMediaTypes.add(MediaType.APPLICATION_JSON_UTF8);
//        fastJsonHttpMessageConverter.setSupportedMediaTypes(fastMediaTypes);

        return new HttpMessageConverters(converter);
    }
}
```
2. 实体类
``` java
public class City {
    @Id
    private Long cityId;
    private String city;
    private Long countryId;
    @JSONField(serialize = false)
    private Date lastUpdate;

	//------get/set------

}
```
## FastJson使用 ##
[FastJson 在 GitHub 上的链接](https://github.com/alibaba/fastjson),使用可参考[SpringMVC整合FastJson](http://112.74.59.39/%2F2018%2F01%2F19%2Fspringmvc-fastJSON%2F), [FastJson 常见问题](https://github.com/alibaba/fastjson/wiki/%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98),[FastJson 使用示例](https://github.com/alibaba/fastjson/wiki/Samples-DataBind)。


[源码 -> GitHub](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-web),见 MyWebConfig 类中的配置。