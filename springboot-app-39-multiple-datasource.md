---
title: Spring Boot 2实践系列(三十九)：Spring Boot 2.x + Mybatis + Druid + Common Mapper 配置多数据源
comments: true
date: 2019-02-15 17:30:41
tags: [druid,mybatis,通用Mapper,数据源]
categories: [Spring Boot 2实践系列]
---
　　项目需要连接多个数据库时，在不使用数据库中间件的情况下，就需要配置多个数据源，如**主从数据库**、**新旧数据库**(结构不一致)。

　　手头项目新需求需要连接两个数据库，需要配置两个数据源。项目的基本结构是基于 Spring Boot 2.x + Mybatis + Druid + Common Mapper + PageHelper，所以本篇也是在此基础实现多数据源的配置，留个记录。　　
<!-- more -->
## 数据源 ##
使用了数据库连接池(Druid)来对数据源进行管理。数据源的工作主要是读取数据库配置参数创建数据源，基本参数有 driverClassName,url,username,password等等。

[**强烈注意**：使用 Druid 连接池，Spring Boot 2.X 版本不再支持配置继承，多数据源的话每个数据源的所有配置都需要单独配置，否则配置不会生效](https://github.com/alibaba/druid/tree/master/druid-spring-boot-starter)。

自定义每个数据源：
1. 创建自定义数据源 **DataSource** 实例，注入数据源配置参数。
2. 创建用于生产 **sqlSession** 的 **SqlSessionFactory** 实例，需要注入自定义的数据源。
3. 创建 **SqlSessionTemplate**，需要注入自定义的 **SqlSessionFactory**, 管理 **sqlSession**。 
4. 创建自定义的事务管理 **DataSourceTransactionManager**，需要注入自定义的数据源。

## 配置实现 ##
1. 数据库连接参数和 Mybatis 配置
	``` properties
	#=============DataSource One=================
	spring.datasource.source1.driver-class-name=net.sf.log4jdbc.sql.jdbcapi.DriverSpy
	spring.datasource.source1.url=jdbc:log4jdbc:mysql://localhost:3306/sakila?useUnicode=true&characterEncoding=utf-8&allowMultiQueries=true&autoReconnect=true&serverTimezone=GMT%2B8
	spring.datasource.source1.username=panda
	spring.datasource.source1.password=123456
	spring.datasource.source1.name=master
	spring.datasource.source1.type=com.alibaba.druid.pool.DruidDataSource
	spring.datasource.source1.initial-size=1
	spring.datasource.source1.max-active=30
	spring.datasource.source1.min-idle=1
	spring.datasource.source1.max-wait=60000
	spring.datasource.source1.validation-query=select 1
	spring.datasource.source1.validation-query-timeout=1
	spring.datasource.source1.test-while-idle=true
	spring.datasource.source1.async-init=true
	spring.datasource.source1.async-close-connection-enable=true
	
	#=============DataSource Two=================
	spring.datasource.source2.driver-class-name=net.sf.log4jdbc.sql.jdbcapi.DriverSpy
	spring.datasource.source2.url=jdbc:log4jdbc:mysql://localhost:3306/sakila_slave?useUnicode=true&characterEncoding=utf-8&allowMultiQueries=true&autoReconnect=true&serverTimezone=GMT%2B8
	spring.datasource.source2.username=panda
	spring.datasource.source2.password=123456
	spring.datasource.source2.name=slave
	spring.datasource.source2.type=com.alibaba.druid.pool.DruidDataSource
	spring.datasource.source2.initial-size=1
	spring.datasource.source2.max-active=30
	spring.datasource.source2.min-idle=1
	spring.datasource.source2.max-wait=60000
	spring.datasource.source2.validation-query=select 1
	spring.datasource.source2.validation-query-timeout=1
	spring.datasource.source2.test-while-idle=true
	spring.datasource.source2.async-init=true
	spring.datasource.source2.async-close-connection-enable=true

	#============MyBatis配置===================
	#mybatis.mapper-locations=classpath:mapper/*.xml
	#mybatis.type-aliases-package=com.springboot.template.entity
	#mybatis.configuration.map-underscore-to-camel-case=true
	
	#===========通用Mapper=====================
	mapper.mappers=com.springboot.template.mapper.base.BaseMapper
	mapper.safe-update=true
	mapper.safe-delete=true
	mapper.not-empty=true
	mapper.check-example-entity-class=true
	```
2. 两个自定义数据源配置类
	**DataSourceOneConfig：**
	``` java
	package com.springboot.template.common.datasource;
	
	import com.alibaba.druid.pool.DruidDataSource;
	import com.alibaba.druid.spring.boot.autoconfigure.DruidDataSourceBuilder;
	import com.springboot.template.mapper.base.BaseMapper;
	import org.apache.ibatis.session.SqlSessionFactory;
	import org.mybatis.spring.SqlSessionFactoryBean;
	import org.mybatis.spring.SqlSessionTemplate;
	import org.springframework.beans.factory.annotation.Qualifier;
	import org.springframework.boot.context.properties.ConfigurationProperties;
	import org.springframework.context.annotation.Bean;
	import org.springframework.context.annotation.Configuration;
	import org.springframework.context.annotation.Primary;
	import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
	import org.springframework.core.io.support.ResourcePatternResolver;
	import org.springframework.jdbc.datasource.DataSourceTransactionManager;
	import tk.mybatis.spring.annotation.MapperScan;
	
	import javax.sql.DataSource;
	
	/**
	 * @name: DataSourceOneConfig
	 * @desc: 多数据源配置之数据源1
	 **/
	@Configuration
	@MapperScan(basePackages = "com.springboot.template.mapper.source1",
	        sqlSessionTemplateRef = "sqlSessionTemplateOne", markerInterface = BaseMapper.class)
	public class DataSourceOneConfig {
	
	    /**
	     * @desc: 数据源 1
	     * @param: []
	     * @return: javax.sql.DataSource
	     **/
	    @Bean(name = "dataSourceOne")
	    @ConfigurationProperties(prefix = "spring.datasource.source1")
	    @Primary
	    public DataSource dataSourceOne() {
	        //一定要用DruidDataSource类，不然的话druid数据库连接池是不会起作用的
	//        DruidDataSource dataSourceOne = new DruidDataSource();
	        DruidDataSource dataSourceOne = DruidDataSourceBuilder.create().build();
	        return dataSourceOne;
	    }
	
	    /**
	     * @desc: sqlSession 配置
	     * @param: []
	     * @return: org.apache.ibatis.session.Configuration
	     **/
	    @Bean(name = "configurationOne")
	    public org.apache.ibatis.session.Configuration configurationOne() {
	        org.apache.ibatis.session.Configuration configuration = new org.apache.ibatis.session.Configuration();
	        //开启驼峰映射
	        configuration.setMapUnderscoreToCamelCase(true);
	        return configuration;
	    }
	
	    /**
	     * @desc: SqlSessionFactory 1
	     * @param: [dataSource]
	     * @return: org.apache.ibatis.session.SqlSessionFactory
	     **/
	    @Bean(name = "sqlSessionFactoryOne")
	    @Primary
	    public SqlSessionFactory sqlSessionFactoryOne(@Qualifier("dataSourceOne") DataSource dataSource) throws Exception {
	        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
	        sqlSessionFactoryBean.setDataSource(dataSource);
	        //指定mapper xml目录
	        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
	        sqlSessionFactoryBean.setMapperLocations(resolver.getResources("classpath:mapper/*.xml"));
	        sqlSessionFactoryBean.setTypeAliasesPackage("com.springboot.template.entity");
	        sqlSessionFactoryBean.setConfiguration(configurationOne());
	        return sqlSessionFactoryBean.getObject();
	    }
	
	    /**
	     * @desc: SqlSessionTemplate
	     * @param: [sqlSessionFactory]
	     * @return: SqlSessionTemplate
	     **/
	    @Bean(name = "sqlSessionTemplateOne")
	    @Primary
	    public SqlSessionTemplate sqlSessionTemplateOne(@Qualifier("sqlSessionFactoryOne") SqlSessionFactory sqlSessionFactory) {
	        return new SqlSessionTemplate(sqlSessionFactory);
	    }
	
	    /**
	     * @desc: 数据源1的事务管理
	     * @param: [dataSource]
	     * @return: org.springframework.jdbc.datasource.DataSourceTransactionManager
	     **/
	    @Bean(name = "dataSourceTransactionManagerOne")
	    @Primary
	    public DataSourceTransactionManager dataSourceTransactionManagerOne(@Qualifier("dataSourceOne") DataSource dataSource) {
	        return new DataSourceTransactionManager(dataSource);
	    }
	}
	```
	**DataSourceTwoConfig：**
	``` java
	package com.springboot.template.common.datasource;
	
	import com.alibaba.druid.pool.DruidDataSource;
	import com.alibaba.druid.spring.boot.autoconfigure.DruidDataSourceBuilder;
	import com.springboot.template.mapper.base.BaseMapper;
	import org.apache.ibatis.session.SqlSessionFactory;
	import org.mybatis.spring.SqlSessionFactoryBean;
	import org.mybatis.spring.SqlSessionTemplate;
	import org.springframework.beans.factory.annotation.Qualifier;
	import org.springframework.boot.context.properties.ConfigurationProperties;
	import org.springframework.context.annotation.Bean;
	import org.springframework.context.annotation.Configuration;
	import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
	import org.springframework.core.io.support.ResourcePatternResolver;
	import org.springframework.jdbc.datasource.DataSourceTransactionManager;
	import tk.mybatis.spring.annotation.MapperScan;
	
	import javax.sql.DataSource;
	
	/**
	 * @name: DataSourceTwoConfig
	 * @desc: 多数据源配置之数据源2
	 **/
	@Configuration
	@MapperScan(basePackages = "com.springboot.template.mapper.source2",
	        sqlSessionTemplateRef = "sqlSessionTemplateTwo", markerInterface = BaseMapper.class)
	public class DataSourceTwoConfig {
	
	    /**
	     * @desc: 数据源2
	     * @param: []
	     * @return: javax.sql.DataSource
	     **/
	    @Bean(name = "dataSourceTwo")
	    @ConfigurationProperties(prefix = "spring.datasource.source2")
	    public DataSource dataSourceTwo() {
	        //一定要用DruidDataSource类，不然的话druid数据库连接池是不会起作用的
	//        DruidDataSource dataSourceTwo = new DruidDataSource();
	        DruidDataSource dataSourceTwo = DruidDataSourceBuilder.create().build();
	        return dataSourceTwo;
	    }
	
	    /**
	     * @desc: sqlSession 配置
	     * @param: []
	     * @return: org.apache.ibatis.session.Configuration
	     **/
	    @Bean(name = "configurationTwo")
	    public org.apache.ibatis.session.Configuration configurationTwo() {
	        org.apache.ibatis.session.Configuration configuration = new org.apache.ibatis.session.Configuration();
	        //开启驼峰映射
	        configuration.setMapUnderscoreToCamelCase(true);
	        return configuration;
	    }
	
	    /**
	     * @desc: SqlSessionFactory 2
	     * @param: [dataSource]
	     * @return: org.apache.ibatis.session.SqlSessionFactory
	     **/
	    @Bean(name = "sqlSessionFactoryTwo")
	    public SqlSessionFactory sqlSessionFactoryTwo(@Qualifier("dataSourceTwo") DataSource dataSource) throws Exception {
	        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
	        sqlSessionFactoryBean.setDataSource(dataSource);
	        //指定mapper xml目录
	        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
	        sqlSessionFactoryBean.setMapperLocations(resolver.getResources("classpath:mapper/*.xml"));
	        sqlSessionFactoryBean.setTypeAliasesPackage("com.springboot.template.entity");
	        sqlSessionFactoryBean.setConfiguration(configurationTwo());
	        return sqlSessionFactoryBean.getObject();
	    }
	
	    /**
	     * @desc: SqlSessionTemplate
	     * @param: [sqlSessionFactory]
	     * @return: SqlSessionTemplate
	     **/
	    @Bean(name = "sqlSessionTemplateTwo")
	    public SqlSessionTemplate sqlSessionTemplateTwo(@Qualifier("sqlSessionFactoryTwo") SqlSessionFactory sqlSessionFactory) {
	        return new SqlSessionTemplate(sqlSessionFactory);
	    }
	
	    /**
	     * @desc: 数据源2的事务管理
	     * @param: [dataSource]
	     * @return: org.springframework.jdbc.datasource.DataSourceTransactionManager
	     **/
	    @Bean(name = "dataSourceTransactionManagerTwo")
	    public DataSourceTransactionManager dataSourceTransactionManagerTwo(@Qualifier("dataSourceTwo") DataSource dataSource) {
	        return new DataSourceTransactionManager(dataSource);
	    }
	}
	```
	**配置说明：**
	@Configuration：说明是配置为，注册为 Spring Bean。
	@MapperScan：扫描 Mapper 接口，区分不同的数据源。
	@Primary：指定主数据源，有且只能有一个主数据源，否则报错。 
	@ConfigurationProperties：读取 properties 配置文件中的数据库连接参数, 用于设置 DataSourceProperties 的属性。
	
	**特别注意：**
	因此 demo 集成的是 mybatis-spring-boot-starter 包,在默认的单数据源时，可以在 properties 文件配置 mybatis，如下：
	``` properties
	#============MyBatis配置===================
	mybatis.mapper-locations=classpath:mapper/*.xml
	mybatis.type-aliases-package=com.springboot.template.entity
	mybatis.configuration.map-underscore-to-camel-case=true
	```
	而多数据源则不支持此方式配置, 每个数据源必须有自己的配置, 包括指定 xxxMapper.xml 文件路径, 定义实体类别名, 开启驼峰规则，否则会报无法绑定 Mapper 接口方法的错误。
	
3. 两个数据源对应的 Mapper 接口路径
	``` properties
	com.springboot.template.mapper.source1.UserMapperOne.java
	com.springboot.template.mapper.source2.UserMapperTwo.java
	```
4. 与两个数据源 Mapper 接口对应的 xxxMapper.xml文件。
	``` properties
	src/resources/mapper/UserMapperOne.xml
	src/resources/mapper/UserMapperTwo.xml
	```
5. 多数据源使用
在业务层，根据需要注入多个数据源的 Mapper, 调用 Mapper 接口。
6. 配置了两个数据源，若要配置更多数据源，同样的操作。

## 源码 ##
[GitHub -> Spring-Boot-Example/spring-boot-multiple-datasource/](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-multiple-datasource)

## 相关参考 ##
1. [Mybatis Common Mapper - Mybatis通用Mapper](https://github.com/abel533/Mapper)
2. [Mybatis-PageHelper - Mybatis 通用分页插件](https://github.com/pagehelper/Mybatis-PageHelper)
3. [druid/druid-spring-boot-starter](https://github.com/alibaba/druid/tree/master/druid-spring-boot-starter)
3. [spring boot整合mybatis通用mapper实现Druid多数据源](https://blog.csdn.net/aa456aaxxx/article/details/80346703)