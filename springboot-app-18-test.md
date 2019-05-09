---
title: Spring Boot 2实践系列(十八)：Spring Boot的测试
comments: true
date: 2018-06-20 23:36:50
tags: [test,junit,runwith]
categories: [Spring Boot 2实践系列]
---
　　Spring Boot，每次新建项目都会自动加上`spring-boot-starter-test`的依赖，同时在`src/test/java`生成当前项目的测试类。

　　Spring Boot 的测试由两个模块提供支持：spring-boot-test 提供测试的核心功能，spring-boot-test-autoconfigure 提供测试的自动配置。

　　Spring Boot 提供了 `@SpringBootTest`注解， 用于替换 spring-test 的 `@ContextConfiguration`注解, 该注解通过 SpringApplication 创建用于测试的 ApplicationContext, 就可以调用 Spring Boot 的功能。

　　Spring Boot 关于使用 Mock 测试 Spring MVC 可以参考[SpringMVC使用MockMvc和Junit进行单元测试](http://112.74.59.39/2018/02/24/springmvc-unit-test-mockmv/)，[官方文档-测试特性(boot-features-testing)](https://docs.spring.io/spring-boot/docs/2.0.3.RELEASE/reference/htmlsingle/#boot-features-testing)。
<!-- more -->
  Spring Boot 提供了三种测试请求类型，分别是 `WebTestClient`， `MockMvc`，`TestRestTemplate`。其中 WebTestClient 的自动配置由 spring-boot-starter-webflux 提供。
``` java
package com.springboot.jpa;

import com.alibaba.fastjson.JSON;
import com.springboot.jpa.entity.User;
import com.springboot.jpa.repository.UserRepository;
import org.junit.Assert;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.reactive.server.WebTestClient;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.context.WebApplicationContext;

import java.util.List;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Transactional
public class JpaApplicationTests {

    String userListStr;

    @Autowired
    private UserRepository userRepository;
    @Autowired
    private TestRestTemplate testRestTemplate;
    @Autowired
    private WebTestClient webTestClient;

    MockMvc mockMvc;

    @Autowired
    private WebApplicationContext webApplicationContext;

    /**
     * 在测试之前初始化些数据
     */
    @Before
    public void setUp(){
        User user1 = new User().setId(11L).setAge(11).setName("Linker").setAddress("深圳。。。。。");
        User user2 = new User().setId(12L).setAge(12).setName("Kinger").setAddress("广州。。。。。");
        userRepository.save(user1);
        userRepository.save(user2);

        List<User> userList = userRepository.findAll();
        userListStr = JSON.toJSONString(userList);

        mockMvc = MockMvcBuilders.webAppContextSetup(webApplicationContext).build();
        System.out.println(userListStr);
    }

    @TestConfiguration
    static class Config {

        @Bean
        public RestTemplateBuilder restTemplateBuilder() {
            return new RestTemplateBuilder().setConnectTimeout(1000).setReadTimeout(1000);
        }
    }

    /**
     * webTestClientDemo
     */
    @Test
    public void webTestClientDemo() {
        String responseBody = webTestClient.get().uri("/user/queryAll").exchange().expectStatus().isOk()
                .expectBody(String.class).returnResult().getResponseBody();
        System.out.println("--------------webTestClientDemo-------------" + responseBody);
    }


    /**
     * TestRestTemplate
     * @throws Exception
     */
    @Test
    public void testRestTemplateDemo() throws Exception {
        ResponseEntity<String> entity = this.testRestTemplate.getForEntity("/user/queryAll", String.class);
        System.out.println("------------" + entity.getBody().toString());
        System.out.println("------------" + entity.getStatusCode());
        System.out.println("------------" + entity.getStatusCodeValue());

        System.out.println("------------" + JSON.toJSONString(entity));
    }

    /**
     * MockMvc
     * @throws Exception
     */
    @Test
    public void mockMvcDemo() throws Exception {
        MvcResult result = mockMvc.perform(MockMvcRequestBuilders.get("/user/queryAll")
                .accept(MediaType.APPLICATION_JSON_UTF8)).andReturn();
        int status = result.getResponse().getStatus();
        String content = result.getResponse().getContentAsString();
        Assert.assertEquals("error,错误消息", 200, status);
    }
}
```

[源码 -> Github](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-test)