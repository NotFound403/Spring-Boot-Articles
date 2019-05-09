---
title: Spring Boot 2实践系列(十三)：MongoDB集成详解与使用
comments: true
date: 2018-06-07 00:28:05
tags: [mongodb,MongoTemplate,MongoRepository]
categories: [Spring Boot 2实践系列]
---
　　MongoDB的介绍和简单使用请参考[MongoDB(一)：Linux 环境安装MongoDB与简单使用](http://112.74.59.39:90/2018/06/10/mongodb-1-install/)

　　Spring Boot 对 MongoDB 的通过 Spring Data MongoDB 来实现的,Spring Data MongoDB 提供了类似 **JPA**的操作和注解。
<!-- more -->
## OR映射注解 ##
提供了类似于 **JPA** 的注解来映射实体类与数据库之间的关系, **@Id**注解是 Spring Data提供，其它注解是MongoDB自有，在 org.springframework.data.mongodb.core.mapping包下。

|注解                  |描述                                             |
|:--------------------|:------------------------------------------------|
|@Document            |映射实体对象与MongoDB的数据库中的一个集合(表)         |
|@Id                  |映射当前属性是ID                                   |
|@Field               |为文档属性定义名称(实体类属性别名)                   |
|@Version             |将当前属性作为文档版本                              |
|@DbRef               |当前属性将参考其他的文档                            |

## MongoTemplate ##
Spring Data MongoDB 提供了一个`MongoTemplate`, MongoTemplate 提供了基本的数据访问的方法。

配置 `MongoTemplate` **Bean**：
``` java
package com.springboot.nosql.config;

import com.mongodb.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.mongodb.MongoDbFactory;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.SimpleMongoDbFactory;

//MongoClient及MongoDbFactory
//配置数据库连接属性
@Configuration
public class MongodbConfig {
    String database = null;

     //注解MongoClient为Bean
    @Bean
    public MongoClient mongoClient(){
        MongoClientURI mongoClientURI = new MongoClientURI("mongodb://root:123456@10.0.3.4:27017/admin");
        this.database = mongoClientURI.getDatabase();
        return new MongoClient(mongoClientURI);
    }

    @Bean
    public MongoDbFactory mongoDbFactory(){
        return new SimpleMongoDbFactory(mongoClient(),database);
    }

    @Bean
    public MongoTemplate mongoTemplate(MongoDbFactory mongoDbFactory){
        return new MongoTemplate(mongoDbFactory);
    }
}
```
Spring Boot 集成**MongoDB**，添加`spring-boot-starter-data-mongodb`包后，Spring Boot会自动将`MongoTemplate`注册为**Bean**，在业务层可直接用于注入即可使用,不需要手动添加此配置类。

## MongoRepository ##
创建实体类的 Repository 接口，继承 MongoRepository，即可调用里面的方法来操作数据。
**备注：** MongoRepository 没有提供 update 方法。

## Spring Boot支持MongoDB ##
Spring Boot也配置了些默认的参数，如端口为：27017，服务器地址为：127.0.0.1，数据库为：test，这些参数都可以在配置文件中设置。
Spring Boot自动开启了对 **Repository**的支持,即已经自动配置了**@EnableMongoRepositories**。Spring Boot 1.5.x 与 2.0.x 版本在属性配置上有些不同,不可通用。
```
#-------------- mongodb ------------------------
##------------- mongodb 3.x --------------------
spring.data.mongodb.uri=mongodb://root:123@192.168.220.129:27017/admin
## 多个IP集群连接
#spring.data.mongodb.uri=mongodb://username:password@ip1:port1,ip2:port2/database

##-----------mongodb 2.x------------------------
#spring.data.mongodb.uri=mongodb://192.168.220.129:27017/admin
#spring.data.mongodb.username=root
#spring.data.mongodb.password=123

spring.data.mongodb.authentication-database=admin
spring.data.mongodb.repositories.type=auto
```

## 示例 ##
示例包含了调用 `MongoTemplate` 的方法和 `MongoRepository`的方法来执行`CRUD`的操作。
以下代码只贴下主要的CRUD代码，[Github 源码](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-nosql-mongodb)
1. 控制层
``` java
import com.mongodb.client.result.DeleteResult;
import com.mongodb.client.result.UpdateResult;
import com.springboot.nosql.entity.Actor;
import com.springboot.nosql.repository.ActorRepository;
import com.springboot.nosql.service.ActorService;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Date;
import java.util.List;

@RestController
@RequestMapping(value = "/actor")
public class ActorController {
    private final static Logger logger = LogManager.getLogger(ActorController.class);

    @Autowired
    private ActorService actorService;

    @Autowired
    private ActorRepository actorRepository;

    //----------MongoTemplate-------------------------

    /**
     * 保存
     * @param actor
     */
    @RequestMapping("/saveActor")
    public void saveActor(Actor actor ){
        actor.setLastUpdate(new Date());
        actorService.saveActor(actor);
    }

    /**
     * 根据主键id查
     * @param actorId
     * @return
     */
    @RequestMapping("/queryByActorId")
    public Actor queryByActorId(Long actorId ){
        return actorService.queryByActorId(actorId);
    }

    /**
     * 根据用户名查
     * @param firstName
     * @return
     */
    @RequestMapping("/queryByFirstName")
    public List<Actor> queryByFirstName(String firstName ){
        return actorService.queryByFirstName(firstName);
    }

    /**
     * 更新
     * @param Actor
     * @return
     */
    @RequestMapping("/updateActor")
    public UpdateResult updateActor(Actor Actor ){
        return actorService.updateActor(Actor);
    }

    /**
     * 删除
     * @param actorId
     * @return
     */
    @RequestMapping("/deleteByActorId")
    public DeleteResult deleteByActorId(Long actorId ){
        return actorService.deleteByActorId(actorId);
    }


    //------ActorRepository extends MongoRepository-------
    //------以下示例代码,省略了Service层----------------------
    @RequestMapping("/addActor")
    public Actor addActor(Actor actor ){
        actor.setLastUpdate(new Date());
        return actorRepository.save(actor);
    }

    @RequestMapping("/findAll")
    public List<Actor> findAll( ){
        return actorRepository.findAll();
    }

    @RequestMapping("/findById")
    public Actor findById(Long actorId){
        return actorRepository.findById(actorId).get();
    }

    @RequestMapping("/deleteById")
    public void deleteById(Long actorId){
        actorRepository.deleteById(actorId);
    }

    //MongoRepository没有update方法
}

```
2. 业务层
``` java
import com.mongodb.client.result.DeleteResult;
import com.mongodb.client.result.UpdateResult;
import com.springboot.nosql.dao.ActorDao;
import com.springboot.nosql.entity.Actor;
import com.springboot.nosql.service.ActorService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class ActorServiceImpl implements ActorService {

    @Autowired
    private ActorDao actorDao;

    @Override
    public void saveActor(Actor actor) {
        actorDao.saveActor(actor);
    }

    @Override
    public Actor queryByActorId(Long actorId) {
        return actorDao.queryByActorId(actorId);
    }

    @Override
    public List<Actor> queryByFirstName(String firstName) {
        return actorDao.queryByFirstName(firstName);
    }

    @Override
    public UpdateResult updateActor(Actor actor) {
        return actorDao.updateActor(actor);
    }

    @Override
    public DeleteResult deleteByActorId(Long actorId) {
        return actorDao.deleteByActorId(actorId);
    }
}
```
3. Repository
``` java
import com.springboot.nosql.entity.Actor;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface ActorRepository extends MongoRepository<Actor, Long> {
}
```

4. MongoTemplate

``` java
package com.springboot.nosql.dao.impl;

import com.mongodb.client.result.DeleteResult;
import com.mongodb.client.result.UpdateResult;
import com.springboot.nosql.dao.ActorDao;
import com.springboot.nosql.entity.Actor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.data.mongodb.core.query.Query;
import org.springframework.data.mongodb.core.query.Update;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public class ActorDaoImpl implements ActorDao {

    @Autowired
    private MongoTemplate mongoTemplate;

    @Override
    public void saveActor(Actor actor) {
        mongoTemplate.save(actor);
    }

    @Override
    public Actor queryByActorId(Long actorId) {
        return mongoTemplate.findById(actorId, Actor.class);
    }

    @Override
    public List<Actor> queryByFirstName(String firstName) {
        Query query = new Query(Criteria.where("firstName").is(firstName));
        return mongoTemplate.find(query, Actor.class);
    }

    @Override
    public UpdateResult updateActor(Actor actor) {
        Query query = new Query(Criteria.where("actorId").is(actor.getActorId()));

        Update update = new Update().set("firstName", actor.getFirstName());
        //更新查询返回结果集的第一条
        return mongoTemplate.updateFirst(query, update, Actor.class);

        //更新查询返回结果集的所有
//        return mongoTemplate.updateMulti(query, update, Actor.class);
    }

    @Override
    public DeleteResult deleteByActorId(Long actorId) {
        Query query = new Query(Criteria.where("actorId").is(actorId));
        return mongoTemplate.remove(query, Actor.class);
    }
}
```
