---
title: Spring Boot 2实践系列(三十二)：Spring Boot 配置文件密码加密两种方案
comments: true
date: 2018-09-19 15:58:07
tags: [encyptor,md5,加密,密码]
categories: [Spring Boot 2实践系列]
---
　　Spring Boot 项目常把连接数据库的密码明文放在配置文件里，这是非常不安全的，数据是IT企业的核心资产，即使应用服务器被玫击破坏也不能影响到数据库中的数据，更不能因为**明文密码**被窃取而导致数据库被随意连接和不安全操作的可能存在，所以需要对**密码**进行加密来增加安全性。

　　介绍两种加密方式：jasypt 可加密配置文件中所有属性值; druid 自带了加解密,可对数据库密码进行加密。 
<!-- more -->
## jasypt 加解密 ##
**jasypt** 是一个简单易用的加解密Java库，可以快速集成到 Spring 项目中。可以快速集成到 Spring Boot 项目中，并提供了自动配置，使用非常简单。
jasypt 库已上传到 Maven 中央仓库, [在 GitHub 上有更详细的使用说明](https://github.com/ulisesbocchio/jasypt-spring-boot)。
jasypt 的实现原理是实现了 ApplicationContextInitializer 接口，重写了获取环境变量的方法，在容器初始化时对配置文件中的属性进行判断，若包含前后缀(`ENC(`的`)`)表示是加密属性值，则进行解密并返回。
1. 添加依赖
``` xml
<!-- jasypt加密 -->
<dependency>
    <groupId>com.github.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>2.1.0</version>
</dependency>
```
2. 环境变量设置 jasypt 密钥
> jasypt.encryptor.password=vh^onsYFUx^DMCKK
>
> 或不让密钥在配置文件中显示，可以在启动应用时将密码做为启动参数传递给应用
> 在运行应用时设置环境变量，如下：
> `java -jar target/jasypt-spring-boot-demo-0.0.1-SNAPSHOT.jar --jasypt.encryptor.password=password`
> 
> 或将密钥作为命令行参数传递运行应用
> `java -Djasypt.encryptor.password=password -jar target/jasypt-spring-boot-demo-0.0.1-SNAPSHOT.jar`

3. 数据源密码先用明文显示来连接
``` 
spring.datasource.username=root
spring.datasource.password=123456
```
4. 编写测试代码生成密文
	``` java
	@RunWith(SpringRunner.class)
	@SpringBootTest
	public class ApplicationTests {
	
	    @Autowired
	    StringEncryptor encryptor;
	
	
	    @Test
	    public void jacketEncrypt() {
	
	        //加密
	        String name = encryptor.encrypt("admin");
	        String password = encryptor.encrypt("123456");
	        System.out.println("name 密文: " + name);
	        System.out.println("password 密文: " + password);
	
	        //解密
	        String decrypt1 = encryptor.decrypt(name);
	        String decrypt2 = encryptor.decrypt(password);
	        System.out.println(decrypt1 + "------------" + decrypt2);
	    }
	}
	
	#-------结果--------
	name 密文: bcT3uohCdQXHD58iEOG/sw==
	password 密文: LzogLizdqSBR2gg8Gb72Qw==
	```
	密码密文也可以通过以下命令来生成：
	```
	java -cp jasypt-1.9.2.jar org.jasypt.intf.cli.JasyptPBEStringEncryptionCLI input="123456" password=Hello@World algorithm=PBEWithMD5AndDES
	input的值是原明文密码
	password的值是加密密钥，是 jasypt.encryptor.password= 的值
	```
5. 替换用户名和密码,密文使用ENC进行标识, 如下
```
#方案一:jasypt加解密
spring.datasource.username=ENC(bcT3uohCdQXHD58iEOG/sw==)
spring.datasource.password=ENC(LzogLizdqSBR2gg8Gb72Qw==)
```
6. 重启应用,查看数据源初始化连接是否成功。
 
## druid 非对称加密 ##
数据库连接池 Druid 自身支持对数据库密码的加密解密, [是通过 ConfigFilter 实现的，在 GitHub 有官方的指导说明](https://github.com/alibaba/druid/wiki/%E4%BD%BF%E7%94%A8ConfigFilter)。
1. 添加依赖,
``` xml
<dependency>
   <groupId>com.alibaba</groupId>
   <artifactId>druid-spring-boot-starter</artifactId>
   <version>1.1.10</version>
</dependency>
```
2. 配置数据源,先用明文密码连接数据库
```
spring.datasource.username=root
spring.datasource.password=123456
```
3. 编写测试代码生成非对称加密的**公钥和私钥**
	``` java
	@RunWith(SpringRunner.class)
	@SpringBootTest
	public class ApplicationTests {
	
	    @Test
	    public void druidEncrypt() throws Exception {
	        //密码明文
	        String password = "123456";
	        System.out.println("明文密码: " + password);
	        String[] keyPair = ConfigTools.genKeyPair(512);
	        //私钥
	        String privateKey = keyPair[0];
	        //公钥
	        String publicKey = keyPair[1];
	
	        //用私钥加密后的密文
	        password = ConfigTools.encrypt(privateKey, password);
	
	        System.out.println("privateKey:" + privateKey);
	        System.out.println("publicKey:" + publicKey);
	
	        System.out.println("password:" + password);
	
	        String decryptPassword = ConfigTools.decrypt(publicKey, password);
	        System.out.println("解密后:" + decryptPassword);
	    }
	}
	
	#------------结果------------------
	明文密码: 123456
	privateKey:MIIBVgIBADANBgkqhkiG9w0BAQEFAASCAUAwggE8AgEAAkEAlgDJ+BjPrmzXfnZ3DYddy7LyVqvyWkbDkVuw+hhsKPZNJRpuCjAGj9omHoj4EJ5ZMsW8emKapCPZaKKUtw1DhQIDAQABAkAgpdtPnFbXZ+kfJTmUQDox86i7JIGDFJPMN2C1jks8PsoKRuMwbSSXd3owdGyEQ28bJa3EOEdkGex+2IqsfZwBAiEAx7aclTD+MVsx9dkOcp5oWpCDpQCK0gbnyIeS5arUcyECIQDAR5Czh8ejceRRcG7yH13+FcC2GIgtLxYmi691hrBn5QIhAJuRCcPFGByGNxKUc4ahEhSJwaIEHB6iNmakBK9WNItBAiEAtXBSmTadKhxEyJyB9LOorCS2rp5Dke+GxWS2cv5f5AkCIQCwhGIq7dmtg12cK4S63zD9/SIbLMTW89ph4rgQFEsoMg==
	
	publicKey:MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBAJYAyfgYz65s1352dw2HXcuy8lar8lpGw5FbsPoYbCj2TSUabgowBo/aJh6I+BCeWTLFvHpimqQj2WiilLcNQ4UCAwEAAQ==
	
	password:CFJ5PUOf0GLY56E27pCPI12eHFqtFzVk/XcBN49qr1e/ya/X1eN4FtGLnaEe/7VPefF40UKPgSqFMbnfPLKAiA==
	```
	生成公钥和私钥，还可使用命令生成：**java -cp druid-1.0.16.jar com.alibaba.druid.filter.config.ConfigTools you_password**
4. 配置文件增加解密支持,并替换明文密码
``` 
#---------密码加密------------------------
spring.datasource.username=panda
spring.datasource.password=CFJ5PUOf0GLY56E27pCPI12eHFqtFzVk/XcBN49qr1e/ya/X1eN4FtGLnaEe/7VPefF40UKPgSqFMbnfPLKAiA==
#---------开启ConfigFilter支持-----------
spring.datasource.druid.filter.config.enabled=true
#---------设置公钥------------------------
spring.datasource.publicKey=MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBAINRom1IY639dDMD0FFw7zMsxRVABYGJnKxSpO84dyJgXaIkoTZkE1JaWE2/gtgli28vgM72UHf2EGhxbLZwzhsCAwEAAQ==
#---------设置连接属性---------------------
spring.datasource.druid.connection-properties=config.decrypt=true;config.decrypt.key=${spring.datasource.publicKey}
```
5. 重启应用, 查看数据源初始化时的连接是否成功。
再谨慎点是把测试生成密文的java文件也删除。

## 完整配置文件 ##
``` java
#=============jdbc dataSource=========================
spring.datasource.name=druidDataSource
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
spring.datasource.url=jdbc:mysql://localhost:3306/sakila?characterEncoding=utf-8&allowMultiQueries=true&autoReconnect=true

#账号密码明文显示
#spring.datasource.username=panda
#spring.datasource.password=123456

#方案一:jasypt加解密
#spring.datasource.username=ENC(ocj4Go8I46th0NOUs2BdGg==)
#spring.datasource.password=ENC(QA8zJh3woJEjyJjaKCpsiQ==)
#jasypt加密
#jasypt.encryptor.password=vh^onsYFUx^DMCKK

#方案二:druid自带非对称加密
spring.datasource.username=root
spring.datasource.password=ai9lB7h4oR9AHrQzU8H38umcelX9dBmx4aSycDOgJWa/2sv5U0GzbyI9sx54sL3nJ0kGayGrTHl3N/Bp1sSJ4w==

spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.druid.initial-size=5
spring.datasource.druid.max-active=20
spring.datasource.druid.min-idle=5
spring.datasource.druid.max-wait=10
spring.datasource.druid.validationQuery=SELECT 1
spring.datasource.druid.filter.config.enabled=true
#spring.datasource.druid.filters=stat,wall,log4j2,config
spring.datasource.publicKey=MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBAINRom1IY639dDMD0FFw7zMsxRVABYGJnKxSpO84dyJgXaIkoTZkE1JaWE2/gtgli28vgM72UHf2EGhxbLZwzhsCAwEAAQ==
spring.datasource.druid.connection-properties=config.decrypt=true;config.decrypt.key=${spring.datasource.publicKey}
```
**注意：**最好将**密钥**或**私钥**作为环境变量参数在执行应用的启动命令时传入，而不是放在配置文件中。

[示例源码 -> GitHub](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-password-encrypt)