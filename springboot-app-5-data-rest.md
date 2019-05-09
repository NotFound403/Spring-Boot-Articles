---
title: Spring Boot 2实践系列(五)： Data Rest 集成详解和使用
comments: true
date: 2018-05-25 12:36:13
tags: [data,rest]
categories: [Spring Boot 2实践系列]
---
　　Spring Data Rest 依赖于 `JPA` , 支持将 JPA,MongoDB，Neo4j，GemFire和Cassandra的 Repository自动转换为`Rest`服务。

　　只需定义**实体类**和`Repository`,就可以直接将查出的数据以`Rest`服务方式返回,可以对数据库执行`CRUD`操作,省略了`Controller`层和`Service`。此方式适合于只需对数据库进行`CRUD`操作的项目，不适合需要对业务逻辑判断封装操作的项目。

　　Spring Data Rest 官网:https://docs.spring.io/spring-data/rest/docs/current/reference/html/
<!-- more -->
## Data Rest ##
1. Spring Data Rest的配置类是`RepositoryRestMvcConfiguration`，位于**org.springframework.data.rest.webmvc.config.RepositoryRestMvcConfiguration**包下。
2. 自定义 Data Rest
给Spring MVC 应用添加 Spring Data REST功能，可以通过继承**RepositoryRestConfigurerAdapter**适配器，或在自己的配置类`@Import`上引入**RepositoryRestMvcConfiguration**类。
3. Spring-boot-starter-data-rest 的自动配置类已经引入了 RepositoryRestMvcConfiguration 配置，SpringBootRepositoryRestConfigurer 继承了 RepositoryRestConfigurerAdapter，所以就不需要人为手动配置。
4. Spring Boot项目引入依赖包
Spring-boot-starter-data-rest 依赖于 **jpa**, 所以要使用 data rest必须添加**rest**和**jpa**这两个依赖。
``` xml
<!-- jpa 依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<!-- rest 依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-rest</artifactId>
</dependency>  
```
## 传统集成Data Rest ##
### 继承方式 ###
``` java
//Spring Boot 2.0.2.Release版本
@Component
public class MyApplicationConfig extends RepositoryRestConfigurerAdapter {

  @Override
  public void configureRepositoryRestConfiguration(RepositoryRestConfiguration config) {
    config.setBasePath("/api");	//定制请求路径
  }
}
```

### 注解引入 ###
``` java
import org.springframework.context.annotation.Import;
import org.springframework.data.rest.webmvc.RepositoryRestMvcConfiguration;

@Configuration
@Import(RepositoryRestMvcConfiguration.class)
public class MyApplicationConfig {
  //…
}
```

### XML配置 ###
``` xml
<bean class="org.springframework.data.rest.webmvc.config.RepositoryRestMvcConfiguration"/>
```

## SpringBoot分析 ##
### 自动配置 ###
spring-boot-starter-data-rest 的自动配置在**org.springframework.boot.autoconfigure.data.rest**包下。
自动配置类：RepositoryRestMvcAutoConfiguration.class
``` java
@Configuration
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnMissingBean({RepositoryRestMvcConfiguration.class})
@ConditionalOnClass({RepositoryRestMvcConfiguration.class})
@AutoConfigureAfter({HttpMessageConvertersAutoConfiguration.class, JacksonAutoConfiguration.class})
@EnableConfigurationProperties({RepositoryRestProperties.class})
@Import({RepositoryRestMvcConfiguration.class})
public class RepositoryRestMvcAutoConfiguration {
    public RepositoryRestMvcAutoConfiguration() {
    }

    @Bean
    public SpringBootRepositoryRestConfigurer springBootRepositoryRestConfigurer() {
        return new SpringBootRepositoryRestConfigurer();
    }
}
```
从自动配置类可以看到注入了消息转换器，Jackson。RepositoryRestMvcConfiguration配置类使用了Spring MVC的适配器和解析器，也就是数据输出是通过Spring MVC实现的。

### 依赖包分析 ###
![spring-data-rest](http://112.74.59.39/images/1527576826067.png)
从图中的包依赖关系可以看出使用了**jackson**来做数据绑定和转换，**webmvc**用来输出数据。

## SpringBoot集成使用 ##
### 实体类 ###
``` java
@Entity
public class Actor {

    @Id
    @GeneratedValue(strategy=GenerationType.IDENTITY)
    private Long actorId;
    private String firstName;
    private String lastName;
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private Date lastUpdate;
	
	// set/get方法
}
```
### Repository ###
``` java
import com.springboot.example.entity.Actor;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ActorRepository extends JpaRepository<Actor, Long> {

}
```
### 访问 ###
1. 获取 Rest 访问路径信息，直接访问项目根目录，如：localhost:8080
	``` json
	{
	    "_links": {        
	        "actors": {
	            "href": "http://localhost:8080/actors{?page,size,sort}",
	            "templated": true
	        },
	        "profile": {
	            "href": "http://localhost:8080/profile"
	        }
	    }
	}
	```
2. 获取所有,访问路径里拼接实体类的复数(默认的规则)，如：http://localhost:8080/actors
	``` json
	{
	    "_embedded": {
	        "actors": [
	            {
	                "firstName": "PENELOPE",
	                "lastName": "GUINESS",
	                "lastUpdate": "2006-02-14T20:34:33.000+0000",
	                "_links": {
	                    "self": {
	                        "href": "http://localhost:8080/actors/1"
	                    },
	                    "actor": {
	                        "href": "http://localhost:8080/actors/1"
	                    }
	                }
	            },
				{..中间数据省略...},   
	            {
	                "firstName": "LUCILLE",
	                "lastName": "TRACY",
	                "lastUpdate": "2006-02-14T20:34:33.000+0000",
	                "_links": {
	                    "self": {
	                        "href": "http://localhost:8080/actors/20"
	                    },
	                    "actor": {
	                        "href": "http://localhost:8080/actors/20"
	                    }
	                }
	            }
	        ]
	    },
	    "_links": {
	        "first": {
	            "href": "http://localhost:8080/actors?page=0&size=20"
	        },
	        "self": {
	            "href": "http://localhost:8080/actors{?page,size,sort}",
	            "templated": true
	        },
	        "next": {
	            "href": "http://localhost:8080/actors?page=1&size=20"
	        },
	        "last": {
	            "href": "http://localhost:8080/actors?page=9&size=20"
	        },
	        "profile": {
	            "href": "http://localhost:8080/profile/actors"
	        }
	    },
	    "page": {
	        "size": 20,
	        "totalElements": 200,
	        "totalPages": 10,
	        "number": 0
	    }
	}
	```
3. 根据ID获取单个实体类,Rest方式路径拼接ID，如：http://localhost:8080/actors/20
	``` java
	{
	    "firstName": "PENELOPE",
	    "lastName": "GUINESS",
	    "lastUpdate": "2006-02-14T20:34:33.000+0000",
	    "_links": {
	        "self": {
	            "href": "http://localhost:8080/api/actors/1"
	        },
	        "actor": {
	            "href": "http://localhost:8080/api/actors/1"
	        }
	    }
	}
	```
4. 分页查询，在请求URL中拼接分页参数，如：http://localhost:8080/actors?page=2&size=3
5. 查询并排序，如查询所有并按名倒序排：http://localhost:8080/actors?sort=firstName,desc
6. Rest 的**CRUD**操作说明
Data Rest遵循的是 Restful 风格来处理资源，查询是`get`请求，新增是`post`请求,更新是`put`请求,删除是`delete`请求。
7. 新增记录
post请求：http://localhost:8080/actors, JSON字符串参数如下：
``` json
{
	"firstName":"Fly",
	"lastName":"bird",
	"lastUpdate":"2018-05-29 01:10:11"
}
```
8. 更新记录,根据主键ID更新
put请求：http://localhost:8080/actors/201, JSON字符串参数如下：
``` json
{
	"firstName":"Fly1111",
	"lastName":"bird2222",
	"lastUpdate":"2018-05-29 11:10:11"
}
```
9. 删除记录，根据主键ID删除
delete请求：http://localhost:8080/actors/201

**备注：**从访问返回的结果看出，此 Rest 方式与 Spirng Boot 的监控组件(Actuator)提供的端点访问非常相似, 可以暴露总的访问端点，返回子端点列表，再访问子端点获取具体的信息。

## 自定义配置 ##
### 自定义属性 ###
从自动配置的属性类文件可以看到是获取`spring.data.rest`为前辍属性参数。
1. 定制请求路径
> spring.data.rest.base-path=/api
请求路径就变成了：http://localhost:8080/api/actors/201

2. 定制用于排序的属性/列
> spring.data.rest.sort-param-name=age

3. 定制分页每页显示条数
> spring.data.rest.default-page-size=10

### 定制节点路径 ###
在实体类名后加`s`组成的路径是默认的规则，如果需要对此映射路径修改的话，可在`Repository`上使用注解。
``` java
import com.springboot.example.rest.entity.Actor;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.repository.query.Param;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;
import org.springframework.data.rest.core.annotation.RestResource;

import java.util.List;

@RepositoryRestResource(path = "actor")
public interface ActorRepository extends JpaRepository<Actor, Long> {

    /**
     * @RestResource注解常用属性
     * path: 映射路径
     * rel: 引用别名,不设置默认是方法名
     * exported: 两个值,不设置默认是true,设置为false时表示不作为Rest资源暴露;也可作用在类上
     */

    // 路径是方法名：http://localhost:8080/api/actor/search/findByFirstName?firstName=BOB
    Actor findByFirstName(@Param("firstName") String firstName);

    // 定制路径：http://localhost:8080/api/actor/search/firstNameStartsWith?firstName=NICK
    @RestResource(path = "firstNameStartsWith")
    List<Actor> findByFirstNameStartsWith(@Param("firstName") String firstName);

    // 定制路径：http://localhost:8080/api/actor/search/firstNameStartsWith?firstName=NICK
    @RestResource(path = "findByLikeFirstName", rel = "likeFirstName")
    List<Actor> findByFirstNameContains(@Param("firstName") String firstName);

    //方法不作为Rest资源暴露,该属性也可作用在类,即类的所有方法都不暴露
    @RestResource(exported = false)
    Actor findByLastName(@Param("lastName") String lastName);

}
```