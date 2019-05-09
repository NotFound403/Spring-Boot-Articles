---
title: Spring Boot 2实践系列(十)：Spring 缓存体系
comments: true
date: 2018-05-31 23:09:11
tags: [cache,redis]
categories: [Spring Boot 2实践系列]
---
　　Spring 对各种缓存技术抽象成了统一的接口和常用的操作方法，对不同的缓存技术，如 redis, ehcache 等透明地添加缓存的支持。

　　只要使用了缓存注解`@EnableCaching`，Spring Boot就会自动配置缓存基本设置。
<!-- more -->
## Spring 缓存支持 ##
Spring 提供了 org.springframework.cache.CacheManager 接口来统一管理缓存技术，org.springframework.cache.Cache接口包含缓存常用的**CRUD**操作方法，但一般不会使用此接口。
### CacheManager类型 ###
|Cache              |CacheManager        |    描述                    |
|:---------------|:-------------------------|:--------------------------|
|Generic            |通用缓存			 |如果定义了Cache Bean，则所有该Bean类型的 CacheManager都会被创建|
|JCache (JSR-107)   |JCacheCacheManager  |支持JSR-107标准的实现作为缓存技术,如 EhCache 3, Hazelcast, Infinispan等|
|EhCache 2.x        |EhCacheCacheManager |使用EhCache 2.x作用存储缓存|
|Couchbase          |CouchbaseCacheManager|使用分布式Couchbase作为存储缓存|
|Redis              |RedisCacheManager   |使用Redis作为存储缓存|
|Caffeine           |CaffeineCacheManager|Caffeine是对Guava缓存的Java 8重写，取代了对Guava的支持。Caffeine 2.1 or higher|
|Simple             |SimpleCacheManager  |使用给定的collection来作为存储缓存，常用于测试或作简单声明，如果没有提供缓存，则使用ConcurrentHashMap 来作为缓存的简单实现|
### CacheManager实现 ###
使用 **CacheManager** 时，需要注册该实现该接口的 **Bean**。
``` java
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.ehcache.EhCacheCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableCaching //开启声名式缓存
public class AppConfig {

    @Bean
    public EhCacheCacheManager cacheManager(CacheManager ehCacheCacheManager){
        return new EhCacheCacheManager();
    }
}
```
### 声名式缓存注解 ###
Spring 提供了 4个注解来声明缓存规则(使用AOP实现)。
1. **@Cacheable**
方法的结果可以被缓存。查询时会先从缓存中取，如果没有再执行方法返回数据并缓存。
2. **@CachePut**
总是执行方法返回数据并缓存，不会跳过方法的执行。
3. **@CacheEvit**
如果存在缓存数据，则清除所有。
4. **@Caching**
可通过该注解来组合多个注解(上面三个注解)，元注解作为该注解的属性来使用。

## Spring Boot的支持 ##
Spring Boot自动配置了多个 CacheManager 的实现，自动配置在 org.springframework.boot.autoconfigure.cache 包下。
![](http://112.74.59.39/images/1527784560897.png)

缓存的属性类是 **CacheProperties**，以`spring.cache`为前缀的属性来配置缓存。

Spring Boot环境下，使用缓存只需要导入相应的缓存依赖包，并在配置类上使用`@EnableCaching`开启缓存支持。
``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

**备注：**查看 spring-boot-starter-cache 的依赖，依赖了 spring-boot-starter, context, context-support包，这些包也被包含在在 Sping Boot其它组件里，没有独有的依赖包，所以如果项目已存在这三个依赖是可以不添加**spring-boot-starter-cache**也能使用默认缓存。如果要使用默认的 ConcurrentHashMap 来做缓存，就不能添加第三方缓存依赖(ehcache,redis)，否则会启用第三方缓存技术。

### 示例一：Simple ###
示例依赖了`JPA`，需要导入** spring-boot-starter-data-jpa **；使用 `Log4j2`和`fastJson`，需要导入依赖。

1. 开启缓存支持
``` java
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableCaching
public class CacheConfig {

}
```

2. Category实体类
``` java
import com.fasterxml.jackson.annotation.JsonFormat;

import javax.persistence.Entity;
import javax.persistence.Id;
import java.util.Date;

@Entity
public class Category {

    @Id
    private Long CategoryId;
    private String name;
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private Date lastUpdate;
	
	//....get/set.......

}
```

3. Controller
``` java
import com.alibaba.fastjson.JSON;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Date;

@RestController
@RequestMapping(value = "/cache/category")
public class CategoryController {
	//添加fastJson依赖
    private final static Logger logger = LogManager.getLogger(CategoryController.class);

    @Autowired
    private CategoryService categoryService;

    @RequestMapping("/queryById")
    public Category queryById(Long CategoryId){
       Category category = categoryService.queryById(CategoryId);
       logger.info(JSON.toJSONString(category));
       return category;
    }

    @RequestMapping("/queryByCategoryId")
    public Category queryByCategoryId(Category category){
        category = categoryService.queryByCategoryId(category);
        logger.info(JSON.toJSONString(category));
        return category;
    }

    @RequestMapping("/saveCategory")
    public Category saveCategory(){
        Category category = new Category().setCategoryId(18L).setLastUpdate(new Date()).setName("Hello Kitty");
        category = categoryService.saveCategory(category);
        logger.info(JSON.toJSONString(category));
        return category;
    }

    @RequestMapping("/deleteById")
    public void deleteById(Long categoryId){
        categoryService.deleteById(categoryId);
    }

    @RequestMapping("/queryByName")
    public Category queryByName(Category category){
        category = categoryService.queryByName(category);
        logger.info(JSON.toJSONString(category));
        return category;
    }

    @RequestMapping("/queryByCategoryName")
    public Category queryByCategoryName(Category category){
        category = categoryService.queryByCategoryName(category);
        logger.info(JSON.toJSONString(category));
        return category;
    }

}
```

4. Service接口
``` java
public interface CategoryService {
    Category queryById(Long categoryId);

    Category queryByCategoryId(Category category);

    Category queryByName(Category category);

    Category queryByCategoryName(Category category);

    Category saveCategory(Category category);

    void deleteById(Long categoryId);
}
```

5. ServiceImpl接口实现
``` java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.data.domain.Example;
import org.springframework.data.domain.ExampleMatcher;
import org.springframework.stereotype.Service;

@Service
public class CategoryServiceImpl implements CategoryService {

    @Autowired
    private CategoryRepository categoryRepository;

    /**
     * Cacheable 先从缓存中查找，没有再找数据库
     * @param categoryId
     * @return
     */
    @Override
    @Cacheable(key = "#categoryId", value = "category")
    public Category queryById(Long categoryId) {
        return categoryRepository.findById(categoryId).get();
    }

    /**
     * 先查缓存，再查数据库
     * @param category
     * @return
     */
    @Override
    @Cacheable(key = "#category.CategoryId", value = "category")
    public Category queryByCategoryId(Category category) {
        return categoryRepository.findById(category.getCategoryId()).get();
    }

    /**
     * 保存数据并存入缓存
     * @param category
     * @return
     */
    @Override
    @Cacheable(key = "#category.CategoryId", value = "category")
    public Category saveCategory(Category category) {
        return categoryRepository.save(category);
    }


    /**
     * 删除数据同时删除缓存
     * @param categoryId
     */
    @Override
    @CacheEvict(value = "category")
    public void deleteById(Long categoryId) {
        categoryRepository.deleteById(categoryId);
    }

    /**
     * 查数据并存缓存
     * @param category
     * @return
     */
    @Override
    @CachePut(key = "#category.name", value = "category")
    public Category queryByName(Category category) {
        Example<Category> example = new Example<Category>(){

            @Override
            public Category getProbe() {
                return category;
            }

            @Override
            public ExampleMatcher getMatcher() {
                ExampleMatcher matcher = ExampleMatcher.matchingAny();
                return matcher;
            }
        };

        return categoryRepository.findOne(example).get();
    }

    /**
     * 先从缓存中查，没有再从库里查
     * @param category
     * @return
     */
    @Override
    @Cacheable(key = "#category.name", value = "category")
    public Category queryByCategoryName(Category category) {
        Example<Category> example = new Example<Category>(){

            @Override
            public Category getProbe() {
                return category;
            }

            @Override
            public ExampleMatcher getMatcher() {
                ExampleMatcher matcher = ExampleMatcher.matchingAny();
                return matcher;
            }
        };
        return categoryRepository.findOne(example).get();
    }
}
```

6. CategoryRepository
``` java
import org.springframework.data.jpa.repository.JpaRepository;

public interface CategoryRepository extends JpaRepository<Category, Long> {
}
```

**参考：**[Cache抽象详解](http://jinnianshilongnian.iteye.com/blog/2001040)