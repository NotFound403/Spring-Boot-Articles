---
title: Spring Boot 2实践系列(四)： Data Jpa 集成详解和使用
comments: true
date: 2018-05-25 10:09:11
tags: [data,jpa]
categories: [Spring Boot 2实践系列]
---
　　JPA 即 Java Persistence API，是一个基于**O/R**映射的标准规范。Hibernate 是该规范的实现，Hibernate 是非常流行的对象关系映射框架(`ORM`),是`SSH`组合开发框架重要组件。

　　Spring Data JPA 是 Spring Data 的一个子项目，它提供了基于 JPA 的 Repository 减少了大量的数据访问操作的代码。JpaRepository 接口提供了通用的数据访问方法。

　　始终建议看官方文档:
　　[Spring Boot 2.0.3 > Use Spring Data Repositories](https://docs.spring.io/spring-boot/docs/2.0.3.RELEASE/reference/htmlsingle/#howto-use-spring-data-repositories)
　　[Spring Data JPA - Reference Documentation](https://docs.spring.io/spring-data/jpa/docs/2.0.8.RELEASE/reference/html/#jpa.repositories)


<!-- more -->
## JDBC自动配置 ##
Spring Boot 自动配置依赖于 spring-boot-autoconfigure-2.0.1.RELEASE.jar 包,默认支持的自动配置都在org.springframework.boot.autoconfigure路径下。
### 自动配置 ###
spring-boot-starter-data-jpa 依赖于spring-boot-starter-jdbc, Spring Boot 为 JDBC 做了些自动配置，并自动开启了注解事务的支持。JDBC 自动配置源码在 org.springframework.boot.autoconfigure.jdbc 路径下。

1. 自动配置文件：**DataSourceAutoConfiguration.class**自动配置类会被注册为`Bean`并执行里面的配置。
2. 配置属性文件：**DataSourceProperties.class**，可以通过`spring.datasource`为前辍的属性自动配置数据源。
3. 自动开启注解事务文件，**DataSourceTransactionManagerAutoConfiguration.class**自动配置事务管理，并配置了一个`JdbcTemplate`。

### 表结构初始化 ###
Spring Boot 提供了初始化数据的功能：放置在类路径(/src/main/resources)下的`schema.sql`文件会自动用来初始化表结构；放置在类路径下的`data.sql`文件会自动用来填充表数据。

**实际项目中几乎不会使用，也不建议使用；实际开发是先根据业务逻辑设计好数据库，再定实体类，再进行业务逻辑代码开发；数据库设计先行能更好更深入的理解业务逻辑和关联关系。**

## JPA自动配置分析 ##
1. Spring Boot 对 JPA 的自动配置在 org.springframework.boot.autoconfigure.orm.jpa 下，从该包下的HibernateJpaAutoConfiguration.class 可以看出，Spring Boot 默认支持的 JPA 实现是Hibernate, HibernateJpaAutoConfiguration 依赖于 `JDBC`的自动配置类DataSourceAutoConfiguration。
2. 从 JpaProperties.class 属性配置类可以看到，配置 JPA 可以使用 spring.jpa 为前缀的属性在 application.properties 中配置。
3. JpaBaseConfiguration.class 类配置了 transactionManager，jpaVendorAdapter，entityManagerFactory 等 Bean。还提供了getPackagesToScan() 方法用于自动扫描注解 @Entity 的实体类。

### Spring Data JPA自动配置 ###
1. 对 Spring Data JPA 的自动配置放在 org.springframework.boot.autoconfigure.data.jpa 包下。
2. 从 JpaRepositoriesAutoConfiguration.class 配置类中可以看到该自动配置依赖于 HibernateJpaAutoConfiguration 配置。该配置类还引入了JpaRepositoriesAutoConfigureRegistrar.class 注册类，该注册启用了 EnableJpaRepositories，所以在 Spring Boot 项目中配置类就无须声明@EnableJpaRepositories。

### Spring Boot JPA配置 ###
在项目中添加`spring-boot-starter-data-jpa`依赖，会自动添加`jdbc`依赖，然后只需定义`DataSource`、实体类和数据访问层，并在需要使用数据访问的地方注入数据访问层的 Bean 即可，无须额外的配置。

### JpaRepositories类图 ###
![](http://112.74.59.39/images/1527427088051.png)

## Spring Boot JPA使用示例 ##
### 项目示例 ###
1. 新建**Spring Boot**项目，引入依赖
``` xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
</dependency>
```
2. 在`application.properties`配置数据源。
``` properties
#-----------data source-----------------
server.port=80
spring.profiles.active=dev
logging.config=classpath:log4j2.xml

#-----------data source-----------------
#spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
#spring.datasource.driverClassName=com.mysql.jdbc.Driver
spring.datasource.driverClassName = net.sf.log4jdbc.DriverSpy 
#spring.datasource.url=jdbc:mysql://localhost:3306/mytest?characterEncoding=utf-8&allowMultiQueries=true&autoReconnect=true
spring.datasource.url=jdbc:log4jdbc:mysql://localhost:3306/mytest?characterEncoding=utf-8
spring.datasource.username=admin
spring.datasource.password=123456

#spring.datasource.tomcat.max-active=20
#spring.datasource.tomcat.test-while-idle=true
#spring.datasource.tomcat.validation-query=select 1
#spring.datasource.tomcat.default-auto-commit=false
#spring.datasource.tomcat.min-idle=15
#spring.datasource.tomcat.initial-size=15

#spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.database-platform=org.hibernate.dialect.MySQL5Dialect
spring.jackson.serialization.indent-output=true
```
3. 定义实体类
``` java
@Entity	//和数据库表映射的实体类
public class Actor {
    @Id	//映射为数据库主键
    @GeneratedValue(strategy=GenerationType.IDENTITY)
    private Long actorId;
//    @Column(name = "realName")        //字段名映射
    private String firstName;
    private String lastName;
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "GMT+8")
    private Date lastUpdate;
	
	//...........set/get方法................

}
```
4. Controller层代码
``` java
import com.springboot.jpa.entity.Actor;
import com.springboot.jpa.service.ActorService;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Date;
import java.util.List;

@RestController
@RequestMapping(value = "/actor")
public class ActorController {
    private static final Logger logger = LogManager.getLogger(ActorController.class);

    @Autowired
    private ActorService actorService;

    /**
     * 添加
     * @param actor
     * @return
     */
    @RequestMapping(value = "/addActor")
    public Actor addActor(Actor actor){
        actor.setLastUpdate(new Date());
        return actorService.addActor(actor);
    }

    /**
     * 删除
     * @param actorId
     * @return
     */
    @RequestMapping(value = "/deleteByActorId")
    public void deleteByActorId(Long actorId){
        actorService.deleteByActorId(actorId);
    }

    /**
     * 改
     * @param actorId
     * @param firstName
     * @return
     */
    @RequestMapping(value = "/updateFirstName")
    public int updateFirstName(Long actorId, String firstName){
        return actorService.updateFirstName(actorId, firstName);
    }

    /**
     * 查所有
     * @return
     */
    @RequestMapping("/queryAll")
    public List<Actor> queryAll(){
        return actorService.queryAll();
    }

    /**
     * 主键条件查询
     * @param actorId
     * @return
     */
    @RequestMapping("/queryByActorId")
    public Actor queryByActorId(Long actorId){
        return actorService.queryByActorId(actorId);
    }

    /**
     * 非主键条件查询
     * @param firstName
     * @return
     */
    @RequestMapping("/queryByFirstName")
    public List<Actor> queryByFirstName(String firstName){
        return actorService.queryByFirstName(firstName);
    }

    /**
     * HQL语句查询
     * @param lastName
     * @return
     */
    @RequestMapping("/queryByLastName")
    public List<Actor> queryByLastName(String lastName){
        return actorService.queryByLastName(lastName);
    }

    /**
     * 条件查询,自定匹配规则
     * @param firstName
     * @param lastName
     * @return
     */
    @RequestMapping("/queryByFirstNameAndLastName")
    public List<Actor> queryByFirstNameAndLastName(String firstName, String lastName){
        return actorService.queryByFirstNameAndLastName(firstName,lastName);
    }

    /**
     * 排序查询
     * @return
     */
    @RequestMapping("/queryByFirstNameWithSortDesc")
    public List<Actor> queryByFirstNameWithSortDesc(String firstName){
//        Sort sort = new Sort(Sort.Direction.DESC, "firstName");
        Sort sort = new Sort(Sort.Direction.ASC, "firstName");
        return actorService.queryByFirstNameWithSortDesc(sort);
    }

    /**
     * 分页查询-sql:从第(page * size) + 1条开始查,查 size 条
     * @param page 页码
     * @param size 每页显示条数
     * @return
     */
    @RequestMapping("/queryByPage")
    public Page<Actor> queryByPage(int page, int size){
        PageRequest pageRequest = PageRequest.of(page, size);
        return  actorService.queryByPage(pageRequest);
    }

    /**
     * 统计
     * @return
     */
    @RequestMapping("/count")
    public Long countActor(){
        return actorService.countActor();
    }

    /**
     * 条件统计
     * @param actor
     * @return
     */
    @RequestMapping("/countBy")
    public Long countBy(Actor actor){
        return actorService.countBy(actor);
    }
}
```
5. 业务接口
``` java
import com.springboot.jpa.entity.Actor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;

import java.util.List;

public interface ActorService {

    Actor addActor(Actor actor);

    void deleteByActorId(Long actorId);

    List<Actor> queryAll();

    Actor queryByActorId(Long actorId);

    Long countActor();

    Long countBy(Actor actor);

    List<Actor> queryByFirstName(String firstName);

    int updateFirstName(Long actorId, String firstName);

    List<Actor> queryByLastName(String lastName);

    List<Actor> queryByFirstNameAndLastName(String firstName, String lastName);

    List<Actor> queryByFirstNameWithSortDesc(Sort sort);

    Page<Actor> queryByPage(PageRequest pageRequest);
}
```
6. 业务接口实现类代码
``` java
import com.springboot.jpa.entity.Actor;
import com.springboot.jpa.repository.ActorRepository;
import com.springboot.jpa.service.ActorService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.*;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class ActorServiceImpl implements ActorService {

    @Autowired
    private ActorRepository actorRepository;

    /**
     * 增
     *
     * @param actor
     * @return
     */
    @Override
    public Actor addActor(Actor actor) {
        return actorRepository.save(actor);
    }

    /**
     * 删: deleteById()
     *
     * @param actorId
     * @return
     */
    @Override
    public void deleteByActorId(Long actorId) {
        actorRepository.deleteById(actorId);
    }

    /**
     * 改
     *
     * @param firstName
     * @return
     */
    @Override
    public int updateFirstName(Long actorId, String firstName) {
        return actorRepository.updateFirstName(actorId, firstName);
    }

    /**
     * 查: 所有, findAll()
     *
     * @return
     */
    @Override
    public List<Actor> queryAll() {
        return actorRepository.findAll();
    }

    /**
     * 查: 条件-主键, findById(primary key)
     *
     * @param actorId
     * @return
     */
    @Override
    public Actor queryByActorId(Long actorId) {
        //springboot 1.5.3.Release
//        return actorRepository.findOne(actorId);

        //springboot 2.0.0.Release及以上
        return actorRepository.findById(actorId).get();
    }

    /**
     * 查: 条件-非主键字段
     * 非主键查询都是匹配对象属性查询
     *
     * @param firstName
     * @return
     */
    @Override
    public List<Actor> queryByFirstName(String firstName) {
        Example<Actor> example = Example.of(new Actor().setFirstName(firstName));
        return actorRepository.findAll(example);
    }

    /**
     * 查: 定制查询条件和匹配规则
     * @param firstName
     * @param lastName
     * @return
     */
    @Override
    public List<Actor> queryByFirstNameAndLastName(String firstName, String lastName) {

        Actor actor = new Actor().setFirstName(firstName).setLastName(lastName);

        Example<Actor> example = new Example<Actor>() {
            //构造参与查询的参数
            @Override
            public Actor getProbe() {
                return actor;
            }
            //设置查询匹配规则
            @Override
            public ExampleMatcher getMatcher() {
                return ExampleMatcher.matchingAny();
            }
        };
//        return actorRepository.findAll(example);

        return actorRepository.findAll(Example.of(actor, ExampleMatcher.matchingAll()));
    }

    /**
     * 排序查询
     * @param sort
     * @return
     */
    @Override
    public List<Actor> queryByFirstNameWithSortDesc(Sort sort) {
        return actorRepository.findAll(sort);
    }

    /**
     * 分页查询
     * @param pageRequest
     * @return
     */
    @Override
    public Page<Actor> queryByPage(PageRequest pageRequest) {
        return actorRepository.findAll(pageRequest);
    }

    /**
     * 使用HQL查询语句
     * @param lastName
     * @return
     */
    @Override
    public List<Actor> queryByLastName(String lastName) {
        return actorRepository.queryByLastName(lastName);
    }

    /**
     * 统计: 所有, count()
     *
     * @return
     */
    @Override
    public Long countActor() {
        return actorRepository.count();
    }

    /**
     * 统计: 条件,全匹配实体类属性,count(example)
     *
     * @param actor
     * @return
     */
    @Override
    public Long countBy(Actor actor) {
//        Actor actor = new Actor().setFirstName(firstName);
//        Example<Actor> example = Example.of(actor);
        return actorRepository.count(Example.of(actor));
    }
}
```
7. 定义数据访问接口
继承`JpaRepository`接口，指定映射的实体类和主键类型。
``` java
import com.springboot.jpa.entity.Actor;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Repository
public interface ActorRepository extends JpaRepository<Actor, Long> {

    /**
     * @Query注解的SQL语句是HQL,是对对象的查询
     * 更新和删除操作必需开启事务,否则会报:
     * nested exception is javax.persistence.TransactionRequiredException
     * @param actorId
     * @param firstName
     * @return
     */
    @Modifying
    @Transactional(timeout = 1000)
    @Query("update Actor set firstName = ?2 where actorId = ?1")
    int updateFirstName(Long actorId, String firstName);

    /**
     * @Query注解传参有两种方式
     * 1. 第一种是按参数的序号传参。
     * 2. 第二种是绑定参数名传参,方法参数必须使用@Param注解来与HQL入参绑定
     * @param lastName
     * @return
     */
//    @Query(value = "select a from Actor a where a.lastName = ?1")
    @Query(value = "select a from Actor a where a.lastName = :lastName")
    List<Actor> queryByLastName(@Param("lastName") String lastName);
}
```

## JPA常用注解 ##
1. **@Entity：**表示这是一个与数据库映射的实体类(作用在类上)。
2. **@Table：**表示该实体类映射到的表名(作用在类上)。
3. **@Id：**表示该属性映射为数据库表的主键(作用在属性上)。
4. **@GeneratedValue：**指示主键生成策略(作用在属性上)。
	- GenerationType.TABLE
	- GenerationType.SEQUENCE
	- GenerationType.IDENTITY
	- GenerationType.AUTO
5. **@Column：**表示该属性映射到表的字段名(作用在属性上)。
6. **@Temporal：** Date类型属性映射到表字段的具体日期时间类型(作用在属性上),可设置如下值：
	- TemporalType.DATE
	- TemporalType.TIME
	- TemporalType.TIMESTAMP
7. **@Inheritance：**指定继承(父子)关系的实体类映射到表的策略(作用在类上)。
	- InheritanceType.SINGLE_TABLE
	- InheritanceType.TABLE_PER_CLASS
	- InheritanceType.JOINED

## 示例代码>github ##
[文章源码(springboot-data-jpa) -> GitHub](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-data-jpa)

【参考】
[@Query注解的用法(Spring Data JPA)](http://www.cnblogs.com/zj0208/p/6008627.html)
[@Query注解及@Modifying注解](https://www.cnblogs.com/zhaobingqing/p/6864223.html)
