---
title: Spring Boot 2实践系列(二十六)：JPA @GeneratedValue四种主键生成策略
comments: true
date: 2018-07-26 11:50:15
tags: [jpa,primary key]
categories: [Spring Boot 2实践系列]
---
　　**JPA**规范中主键生成策略`@GeneratedValue`四种用法：`TABLE,SEQUENCE,IDENTITY,AUTO`。　

　　Spring Boot集成 JPA，在实体类映射表主键列的属性上使用**@GeneratedValue**注解来指示主键生成策略，主键生成策略的类型由枚举类`GenerationType`的值确定。
<!-- more -->
## @GeneratedValue ##
实体类上主键生成策略注解`@GeneratedValue`。
``` java
package javax.persistence;

import java.lang.annotation.Retention;
import java.lang.annotation.Target;

import static java.lang.annotation.ElementType.FIELD;
import static java.lang.annotation.ElementType.METHOD;
import static java.lang.annotation.RetentionPolicy.RUNTIME;
import static javax.persistence.GenerationType.AUTO;

@Target({METHOD, FIELD})
@Retention(RUNTIME)

public @interface GeneratedValue {
	
	//没有显式指定主键策略，默认使用 AUTO，JPA自动选择合适的策略
    GenerationType strategy() default AUTO;

    String generator() default "";
}
```

## GenerationType ##
主键生成类型源码.
``` java
package javax.persistence;

public enum GenerationType {
    //使用一个特定的数据库表格来保存主键,在使用时从该表取数据做为主键(在分布式系统需要全局唯一有序的主键时常使用)。
    TABLE,
    //由底层数据库的序列来生成主键，条件是数据库支持序列。
    SEQUENCE,
    //由数据库自动生成主键,主要在自增ID列使用 
    IDENTITY,
    //主键由程序控制，在代码层必须给映射主键的字段set值
    AUTO
}
```

### 数据库支持类型 ###
1. **MySQL：**支持**TABLE，IDENTITY，AUTO**；不支持**SEQUENCE**。
2. **Oracle：**支持**TABLE，SEQUENCE，AUTO**；不支持**IDENTITY**。

[**参考：**]
[理解JPA注解@GeneratedValue](https://blog.csdn.net/canot/article/details/51455967)
[@GeneratedValue 四种标准用法](https://blog.csdn.net/ethan_fu/article/details/47832907)
[JPA主键生成器和主键生成策略](http://xiaoyaocao.iteye.com/blog/1874412)


