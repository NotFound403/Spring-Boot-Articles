---
title: Spring Boot 2实践系列(一)：整合Mybatis + 通用Mapper + PageHelper分页 + Druid数据库连接池
comments: true
date: 2018-05-18 16:47:11
tags: [mybatis,mapper,pageHelper,datasource,druid]
categories: [Spring Boot 2实践系列]
---
　　**前言：**必须要感谢伟大的开源思想，感谢所有的开源团队和个人，他们提供了如此多的优秀开源项目，这是一件伟大的事。

　　[Spring Boot 开发模板：spring-boot-template](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-template)，此模板基于`Spring Boot 2.0.4 Release` 版本的快速开发模板, 尽量采用 `boot-starter` 依赖, 使用最精简的配置。

　　集成了 `MySQL, Druid, Mybatis, Common Mapper, pageHelper, Log4j2, Shiro, JSTL`, Web页面支持 `JSP`, 支持打 `war` 包, 支持打 `Docker` 镜像。
<!-- more -->
## 使用描述 ##
1. 如果是接口数据,建议统一返回 `ResultBean` 对象。若是分页查询,将查询数据装到 `PageInfo` 里, `pageInfo` 对象封装到 `ResultBean` 的 `data` 属性中。
2. 将 java 配置文件和组件分开存放,让项目结构更清晰。
3. 实体类
    - 所有实体类建议都实现序列化接口, 生成序列化版本号。
    - 实体类 `entity` 包下的 `base` 目录里有个实体类的基类 `BaseEntity` , 基类里面包含 `id,pageNum,pageSize` 三个公用属性,
其中 `pageNum(页码)` 和 `pageSize(每页显示条数)` 是分页参数, 这两个属性不参与持久化。
    - 让实体类继承 `BaseEntity` 基类, 在 `properties` 配置文件添加支持方法参数分页配置(`pagehelper.supportMethodsArguments=true`),
在实体类作为查询参数时就会根据 `pageNum` 和 `pageSize` 这两个属性执行自动分页, 可以省略查询方法前的 `PageHelper.startPage(pageNum, pageSize)` 设置。
    - 若查询参数和和分页参数不封装在实体类里, 是直接作为方法参数传递, 在执行查询方法前需要设置分页参数(`PageHelper.startPage(pageNum, pageSize)`)。
    - 通用 Mapper 默认是根据实体类名首字母小写来找对应的表,若实体类名与表名无法对应,可在实体类上添加注解(`@Table(name = "table_name")`)来映射表名。
    - 通用 Mapper 默认支持下划线和驼峰规则转换,若属性名与表字段无法匹配, 可在属性上添加注解(`@Column(name = "column_name")`)来映射表字段。
4. MyBatis
    - 在配置文件开启了对实体类别名支持, 别名默认是实体类名首字母小写,在 XML 接收实体类参数和返回实体类时,可直接写别名而不用写全限定实体类名。
    - 开启了实体类属性名驼峰规则和下划线的转换, 若转换后可对应表字段, 在 XML 里的字段和属性映射可省略。
    - MyBatis 的自动配置默认开启了 Mapper 级别(二级)缓存的支持, 若需要使用缓存, 可在业务 mapper 接口上添加缓存引用注解 `@CacheNamespaceRef(XxxMapper.class)`,
        在 `mapper.xml` 文件中添加缓存配置 `<cache />`, 有多个属性支持详细配置,例如 `flushInterval` 设置缓存刷新间隔,单位是毫秒。
	- 缓存的配置还有另一种方式：若只使用通用 Mapper 接口中的方法, 只需在业务接口上添加 `@CacheNamespace` 注解来启用缓存, 如果要和 XML 混合使用缓存, 需要在 XML 文件添加缓存引用`<cache-ref namespace="业务mapper接口的全限定名,即该mapper接口在mapper文件的命名空间" />`。 通用 Mapper 的作者发现此方式会报错并给 **Mybatis** 提交了 `issue`, 但在此模板项目上验证未出现异常。
5. Mapper
    - 在 mapper 包下的 base 目录里, 有个 `BaseMapper<T>` 基础接口, 此接口继承了通用 mapper 中的 `Mapper<T>, MySqlMapper<T>` 两个接口, 
        此基础接口不能被 Spring 扫描到,否则会报错, 此项目配置扫描 `mapper` 包时设置了 `markerInterface` 参数, 即只扫描该参数值的父级包下的 mapper 接口文件。
    - 若不需要使用 `MySqlMapper<T>` 中的独有方法, 则业务方法可直接继承通用 `mapper` 的 `Mapper<T>`接口。
6. Shiro
    - 使用了 Shiro 的认证功能对登录用户身份进行认证, 用户登录密码使用 Shiro 的 `Md5Hash` 进行加密码。
7. Docker
    - 在pom.xml文件里集成了将应用打包成 docker 镜像的插件, `Dockerfile` 文件指向 `src/main/docker` 目录。
    - 打镜像命令见 `document` 目录下的 `打Docker镜像命令.txt` 文件内容。

## 项目改造
拉取项目模板, 改造成实际要开发的项目。以下描述基于 IDEA 开发工具。

1. 修改项目名称
    File → Project Structure → Project Settings → Project, 修改 Project name 为实际开发的项目名称; 
    点击 Modules , 修改 Name 名称为实际模块名称。
2. 修改包路径名
	将 com.springboot.template 改为实际需要的包路径名, 包括测试的包路径, 同步修改包下的 Java 文件中的引用包路径。
3. 修改 `pom.xml`文件
	将文件中的 `groupId、 artifactId、 name`三个标签的值，改为实际项目信息。
4. 修改日志配置文件 `log4j2.xml`
	将日志文件里 `log.file` 名称改为实际项目名称。
5. 修改数据源信息
	根据实际的开发、测试、生产环境的数据库修改链接、账号、密码信息。
6. 替换Web页面
7. 需要的话修改登录路径

## pom.xml ##
``` xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.springboot</groupId>
    <artifactId>template</artifactId>
    <version>v1.0.0</version>
    <packaging>war</packaging>

    <name>template</name>
    <description>Template project for Spring Boot</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.4.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <shiro.version>1.3.2</shiro.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
            <!--排除依赖-->
            <exclusions>
                <exclusion>
                    <!--排除默认的日志管理:logback-->
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <!--日志管理使用lo4j2-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-log4j2</artifactId>
        </dependency>
        <!--Spring MVC Web-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!--数据库ORM相关-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <!--log4jdbc:输出完整sql语句,以表格方式格式化输出结果-->
        <dependency>
            <groupId>org.bgee.log4jdbc-log4j2</groupId>
            <artifactId>log4jdbc-log4j2-jdbc4.1</artifactId>
            <version>1.16</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.10</version>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.2</version>
        </dependency>
        <!--通用Mapper-->
        <dependency>
            <groupId>tk.mybatis</groupId>
            <artifactId>mapper-spring-boot-starter</artifactId>
            <version>2.0.4</version>
        </dependency>
        <!-- 分页插件 -->
        <dependency>
            <groupId>com.github.pagehelper</groupId>
            <artifactId>pagehelper-spring-boot-starter</artifactId>
            <version>1.2.7</version>
        </dependency>

        <!-- fastjson -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.49</version>
        </dependency>

        <!--打WAR包,覆盖内嵌的Tomcat-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
            <scope>provided</scope>
        </dependency>

        <!--JSP支持-->
        <dependency>
            <groupId>javax.servlet.jsp</groupId>
            <artifactId>javax.servlet.jsp-api</artifactId>
            <version>2.3.3</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.tomcat.embed</groupId>
            <artifactId>tomcat-embed-jasper</artifactId>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>javax.servlet.jsp.jstl</groupId>
            <artifactId>javax.servlet.jsp.jstl-api</artifactId>
            <version>1.2.2</version>
        </dependency>

        <!--Shiro-->
        <dependency>
            <groupId>org.apache.shiro</groupId>
            <artifactId>shiro-core</artifactId>
            <version>${shiro.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.shiro</groupId>
            <artifactId>shiro-web</artifactId>
            <version>${shiro.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.shiro</groupId>
            <artifactId>shiro-spring</artifactId>
            <version>${shiro.version}</version>
        </dependency>

        <!--Apache Commons-->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.8.1</version>
        </dependency>
        <dependency>
            <groupId>com.github.ulisesbocchio</groupId>
            <artifactId>jasypt-spring-boot-starter</artifactId>
            <version>2.1.0</version>
        </dependency>




        <!--热重启-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
        </dependency>
        <!--扫描自动配置元数据文件,提高应用启动速度-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        <!--测试-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>



    <build>
        <!-- 最终war包名:默认是 ${project.artifactId}-${project.version} -->
        <finalName>${project.name}</finalName>

        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <!-- docker打包插件 -->
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>1.1.1</version>
                <executions>
                    <execution>
                        <phase>install</phase>
                        <goals>
                            <goal>build</goal>
                        </goals>
                    </execution>
                </executions>

                <configuration>
                    <!--禁用构建缓存-->
                    <noCache>true</noCache>
                    <!-- 镜像名称 -->
                    <imageName>${project.name}:latest</imageName>
                    <!-- Dockerfile文件地址 -->
                    <dockerDirectory>${basedir}/src/main/docker</dockerDirectory>
                    <resources>
                        <resource>
                            <!-- <targetPath>/</targetPath> -->
                            <!-- 项目构建输出目录，默认为target/ -->
                            <directory>${project.build.directory}</directory>
                            <!-- 打包出来的war名称，默认为${project.artifactId}-${project.version} -->
                            <include>${project.build.finalName}.war</include>
                        </resource>
                    </resources>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

## properties ##
1. application.properties
``` java
spring.profiles.active=dev

#=============SERVER=====================
server.tomcat.uri-encoding=utf-8

#=============DATASOURCE=================
#Druid 数据源配置，继承spring.datasource.* 配置，相同则覆盖
spring.datasource.name=master
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
spring.datasource.druid.initial-size=1
spring.datasource.druid.max-active=30
spring.datasource.druid.min-idle=1
spring.datasource.druid.max-wait=60000
spring.datasource.druid.validation-query=select 1
spring.datasource.druid.validation-query-timeout=1
spring.datasource.druid.test-while-idle=true
spring.datasource.druid.async-init=true
spring.datasource.druid.async-close-connection-enable=true

#============MyBatis配置===================
mybatis.mapper-locations=classpath:mapper/*.xml
mybatis.type-aliases-package=com.springboot.template.entity
mybatis.configuration.map-underscore-to-camel-case=true

#===========通用Mapper=====================
mapper.mappers=com.springboot.template.mapper.base.BaseMapper
mapper.safe-update=true
mapper.safe-delete=true
mapper.not-empty=true
mapper.check-example-entity-class=true

#=============pagehelper==================
pagehelper.helperDialect=mysql
pagehelper.reasonable=true
pagehelper.supportMethodsArguments=true
pagehelper.params=count=countSql

#===========SpringMVC Web=================
spring.mvc.view.prefix=/WEB-INF/jsp/
spring.mvc.view.suffix=.jsp

#===========Jackson======================
spring.jackson.time-zone=GMT+8
spring.jackson.date-format=yyyy-MM-dd HH:mm:ss

```
2. application-dev.properties
```
#============datasource==========================
spring.datasource.driver-class-name=net.sf.log4jdbc.sql.jdbcapi.DriverSpy
spring.datasource.url=jdbc:log4jdbc:mysql://localhost:3306/sakila?useUnicode=true&characterEncoding=utf-8&allowMultiQueries=true&autoReconnect=true
spring.datasource.username=root
spring.datasource.password=123456

#====JSP 开发环境动态即时更新页面,不需重新编译===========
server.servlet.jsp.init-parameters.development=true
```
2. application-pro.properties
	```
	#============datasource==========================
	#=====这里的 driver-classs配置可省略,数据源的自动配置会根据连接类型来匹配驱动=====
	spring.datasource.driver-class-name=com.mysql.jdbc.Driver
	spring.datasource.url=jdbc:mysql://localhost:3306/sakila?useUnicode=true&characterEncoding=utf-8&allowMultiQueries=true&autoReconnect=true
	spring.datasource.username=root
	spring.datasource.password=123456
	```
	备注：生产环境的数据源的账号密码应加密显示, 连接没有使用 log4jdbc, 否则会打印太多的查询结果数据降低性能。


[源码：spring-boot-template -> Github](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-template)