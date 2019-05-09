---
title: Spring Boot 2实践系列(十九)：JdbcTemplate 的使用
comments: true
date: 2018-06-25 11:56:40
tags: [jdbc,jdbcTemplate,jpaRepository]
categories: [Spring Boot 2实践系列]
---
　　Spring Boot 的 JdbcTemplate 完全依赖于 Spring Data 的 spring-jdbc 支持, JdbcTemplate 类是 **JDBC** 核心包中的核心类。它处理资源创建和释放，**Statement** 创建和执行，使应用代码获取结果；JdbcTemplate 类执行`CRUD`操作和**存储过程**调用，对 **ResultSets** 执行迭代并返回，捕获**JDBC**异常。
　　
　　JdbcTemplate 是非常经典和流行的数据库访问技术，但现在已经有成熟`ORM`框架可以提供更简结的操作和更优的代码结构来支持对数据库的访问。
　　
　　**Spring Boot** 中使用`JdbcTemplate`，需要导入`spring-boot-starter-jdbc`依赖, 实际开发中已使用不多。 **Spring Boot** 提供了更便捷的[JpaRepository](http://112.74.59.39/2018/05/25/springboot-app-4-data-jpa/)来对数据库的访问。

　　[Spring Data -> spring-jdbc -> JdbcTemplate 官方文档](https://docs.spring.io/spring/docs/5.0.5.RELEASE/spring-framework-reference/data-access.html#jdbc)
<!-- more -->
## JdbcTemplate ##
Spring Boot 项目导入 spring-boot-starter-jdbc 包后，会自动配置 `JdbcTemplate`，在`DAO`层直接注入即可使用。

``` java
package com.springboot.jdbc.service.impl;

import com.springboot.jdbc.entity.Actor;
import com.springboot.jdbc.entity.User;
import com.springboot.jdbc.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.stereotype.Service;

import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.List;

/**
 * 演示省略DAO层
 */

@Service
public class UserServiceImpl implements UserService {

	@Autowired
	private JdbcTemplate jdbcTemplate;


    /**
     * 增-insert
     * @param user
     * @return
     */
	@Override
	public int addUser(User user) {
		String sql = "insert into user (id,name,age,address) values (?,?,?,?)";
		return jdbcTemplate.update(sql, user.getId(),user.getName(), user.getAge(), user.getAddress());
	}

    /**
     * 改-update
     * @param name
     * @return
     */
    @Override
    public int updateUser(String name, Long id) {
        String sql = "update user set name = ? where id = ?";
        return jdbcTemplate.update(sql, name, id);
    }

    /**
     * 删除-delete
     * @param id
     * @return
     */
    @Override
    public int deleteUser(Long id) {
        String sql = "delete from user where id = ?";
        return jdbcTemplate.update(sql,id);
    }

    /**
     * rowCount
     * @return
     */
    @Override
    public int queryCount() {
        String sql = "select count(*) from user";
        return jdbcTemplate.queryForObject(sql, Integer.class);
    }

    /**
     * count by
     * @return
     */
    @Override
    public int queryCountByLastName(String lastName) {
        String sql = "select count(*) from actor where last_name = ?";
        return jdbcTemplate.queryForObject(sql,Integer.class,lastName);
    }

    /**
     * 查询结果是字符串
     * @param actorId
     * @return
     */
    @Override
    public String queryLastName(Long actorId) {
        String sql = "select last_name from actor where actor_id = ?";
        Object[] objArr = new Object[1];
        objArr[0] = actorId;
        return jdbcTemplate.queryForObject(sql, objArr, String.class);
    }

    /**
     * 返回对象
     * @param actorId
     * @return
     */
    @Override
    public Actor queryByActorId(Long actorId) {
        String sql = "select * from actor where actor_id = ?";
        Object[] objArr = new Object[1];
        objArr[0] = actorId;

        RowMapper<Actor> actorRowMapper = new RowMapper<Actor>() {
            @Override
            public Actor mapRow(ResultSet resultSet, int i) throws SQLException {
                Actor actor = new Actor()
                        .setFirstName(resultSet.getString("first_name"))
                        .setLastName(resultSet.getString("last_name"))
                        .setActorId(resultSet.getLong("actor_id"))
                        .setLastUpdate(resultSet.getDate("last_update"));
                return actor;
            }
        };
        return jdbcTemplate.queryForObject(sql, objArr, actorRowMapper);
    }

    /**
     * 查询对象集合
     * @return
     */
    @Override
    public List<Actor> queryActorList() {
        String sql = "select * from actor";
        RowMapper<Actor> actorRowMapper = new RowMapper<Actor>() {
            @Override
            public Actor mapRow(ResultSet resultSet, int i) throws SQLException {
                Actor actor = new Actor()
                        .setFirstName(resultSet.getString("first_name"))
                        .setLastName(resultSet.getString("last_name"))
                        .setActorId(resultSet.getLong("actor_id"))
                        .setLastUpdate(resultSet.getDate("last_update"));
                return actor;
            }
        };
        return jdbcTemplate.query(sql,actorRowMapper);
    }

}

```

[源码 -> Github](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-jdbc-template)
