---
title: Spring Boot 2实践系列(二十九)：集成 WebSocket 广播和点对点两种通信方式
comments: true
date: 2018-08-29 14:16:17
tags: [websocket,topic,queue,广播,点对点]
categories: [Spring Boot 2实践系列]
---
要了解 WebSocket, 可先看[Spring Boot实践系列(二十五)：WebSocket详解与使用](http://112.74.59.39/2018/07/25/springboot-app-25-websocket/),可参考官方文档[Spring Boot -> WebSockets](https://docs.spring.io/spring-boot/docs/2.0.4.RELEASE/reference/htmlsingle/#boot-features-websockets), Spring MVC提供了对 WebSockets 的支持[Spring Web MVC -> WebSockets](https://docs.spring.io/spring/docs/5.0.8.RELEASE/spring-framework-reference/web.html#websocket)。

本篇主要记录广播(topic)和点对点(queue)两种模式集成的Demo代码。
<!-- more -->
## topic(广播) ##
1. pom 文件添加 websocket 依赖
``` xml
<!-- websocket -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```
2. WebSocket配置
``` java
package com.springboot.websocket.config;

import com.springboot.websocket.common.WebSocketInterceptor;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.*;

/**
 * @Name: WebSocketConfig
 * @Desc: WebSocket配置
 **/
@Configuration
@EnableWebSocketMessageBroker   //开启使用STOMP协议来传输基于代理(message broker)的消息
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer{               //v2.0.x
//public class WebSocketConfig extends AbstractWebSocketMessageBrokerConfigurer{        //v1.5.x

    //注册 WebSocketp客户端连接端点(路径)
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        //"portfolio"是WebSocket（或SockJS）客户端为了进行WebSocket握手而需要连接的端点的HTTP URL。
        //并指定使用SocketJS协议
        registry.addEndpoint("/websocket")
                .addInterceptors(new WebSocketInterceptor())
                .setAllowedOrigins("*")
                .withSockJS();
    }

    //配置消息代理(MessageBroker)
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        //目标路径"/app"开头的STOMP消息将路由到@Controller类中的@MessageMapping方法。
        config.setApplicationDestinationPrefixes("/app");

        //使用内置的消息代理进行订阅和广播; 将目标标头以“/ topic”或“/ queue”开头的消息路由到代理。
        //"/topic"广播; "/queue"订阅
        config.enableSimpleBroker("/topic");
//        config.enableSimpleBroker("/topic", "/queue");
    }

}
```
3. WebSocket拦截器(可要可不要)
``` java
package com.springboot.websocket.common;

import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.http.server.ServletServerHttpRequest;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.server.support.HttpSessionHandshakeInterceptor;

import javax.servlet.http.HttpSession;
import java.util.Map;

/**
 * @Name: WebSocketInterceptor
 * @Desc: 连接拦截器
 **/
public class WebSocketInterceptor extends HttpSessionHandshakeInterceptor {

    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Map<String, Object> attributes) throws Exception {
        System.out.println("=====================握手前拦截===========================");
        if (request instanceof ServletServerHttpRequest) {
            ServletServerHttpRequest servletRequest = (ServletServerHttpRequest) request;
            HttpSession session = servletRequest.getServletRequest().getSession(false);  
			
			//---------------------------------------------------------          
        }

        return super.beforeHandshake(request, response, wsHandler, attributes);
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Exception exception) {
        super.afterHandshake(request, response, wsHandler, exception);
    }
}
```
4. 发送消息和接收消息实体类
``` java
package com.springboot.websocket.entity;

/**
 * @Name: WebSocketMessage
 * @Desc: 接收WebSocket客户端的消息
 **/
public class ClientMsg {

    private String msg;

    public String getMsg() {
        return msg;
    }

    public ClientMsg setMsg(String msg) {
        this.msg = msg;
        return this;
    }
}

//--------------------------------------------

package com.springboot.websocket.entity;

/**
 * @Name: ResposeMessage
 * @Desc: 服务器响应消息
 **/
public class RespResult {

    private String resultMsg;

    public RespResult(String resultMsg) {
        this.resultMsg = resultMsg;
    }

    public String getResultMsg() {
        return resultMsg;
    }

    public RespResult setResultMsg(String resultMsg) {
        this.resultMsg = resultMsg;
        return this;
    }
}

```
5. WebSocketController
``` java
package com.springboot.websocket.controller;

import com.springboot.websocket.entity.ClientMsg;
import com.springboot.websocket.entity.RespResult;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

/**
 * @Name: WebSocketController
 * @Desc: WebSocket控制器
 **/
@Controller
public class WebSocketController {

    @RequestMapping("/stomp")
    public String stomp(){
        return "stomp";
    }

    @MessageMapping("/hello")
    @SendTo("/topic/getRespMsg")
    public RespResult handle(ClientMsg clientMsg) throws InterruptedException {
        System.out.println("--------------" + clientMsg.getMsg());
        return new RespResult("收到消息：" + clientMsg.getMsg() + "! " + System.currentTimeMillis());
    }
}
```
6. 此Demo页面用的是jsp, 在 application.properties 配置文件添加 jsp 前后缀映射
```
server.port=80

spring.mvc.view.prefix=/WEB-INF/jsp/
spring.mvc.view.suffix=.jsp

//jsp开发模式,修改动态更新
server.servlet.jsp.init-parameters.development=true
```
7. 编写jsp页面(stomp.jsp), 记得添加支持 jsp 的依赖。
``` html
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/fmt" prefix="fmt" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/functions" prefix="fn" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<!DOCTYPE html>
<html lang="en">
<title>STOMP</title>
<head>
    <meta charset="UTF-8">
    <title>Home</title>
</head>
<script type="text/javascript" src="https://cdn.staticfile.org/sockjs-client/1.1.5/sockjs.min.js"></script>
<script type="text/javascript" src="https://cdn.staticfile.org/stomp.js/2.3.3/stomp.min.js"></script>
<script type="text/javascript" src="https://cdn.staticfile.org/jquery/3.3.1/jquery.min.js"></script>

<body>
<div>
    <button id="connect" onclick="connect()">连接</button>
    <button id="disConnect" disabled="disabled" onclick="disConnect()">断开连接</button>
</div>
<div id="conversationDiv">
    <label>消息</label>
    <input type="text" name="msg" id="msgId" value=""/>
    <button id="sendMsg" onclick="sendMsg()">发送</button>
    <p id="response"></p>
</div>
</body>

<script type="text/javascript">

    var stompClient = null;

    function setConnected(connected) {
        document.getElementById("connect").disabled = connected;
        document.getElementById("disConnect").disabled = !connected;
        document.getElementById("conversationDiv").style.visibility = connected ? 'visible' : 'hidden';
        $("#response").html();
    }

    function connect() {
        var socket = new SockJS("http://localhost/websocket");
        stompClient = Stomp.over(socket);

        stompClient.connect({}, function (frame) {
            setConnected(true);
            console.log("connected>>>>:" + frame);//connected>>>>:CONNECTED

            stompClient.subscribe("/topic/getRespMsg", function (response) {
                var msg = JSON.parse(response.body).resultMsg;
                showResponse(msg);
            })
        });
    }

    function disConnect() {
        if (stompClient != null) {
            stompClient.disconnect();
        }
        setConnected(false);
        console.log("DisConnect-------------------------")

    }


    function sendMsg() {
        var msg = $("#msgId").val();
        //以'/app'开头被路由到 @MessageMapping 注解的路径
        stompClient.send("/app/hello", {}, JSON.stringify({"msg": msg}));
    }

    function showResponse(msg) {
        $("#response").html(msg);
    }


</script>
</html>
```
8. 依赖的静态 js 文件,可添加到 resources/static/js 目录, Demo使用的是开源在cdn上的静态资源。
```
jquery-3.3.1.min.js
sockjs.min.js
stomp-1.7.1.js
webstomp.js
```
9. 开启多个浏览器访问 http://localhost/stomp, 给服务器发送消息
可以看到服务器响应的消息在多个浏览器上显示。

## queue(点对点) ##
1. pom 文件添加依赖: thymeleaf模板, websocket, spring security
``` xml
<!-- thymeleaf -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>

<!-- websocket -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>

<!-- spring security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```
2. WebMvcConfig 配置,可不配置此文件,视图控制放在在Controller的方法里返回。
``` java
package com.springboot.websocket.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ViewControllerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

/**
 * @Name: WebMvcConfig
 * @Desc: Web相关配置
 **/
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/login").setViewName("/login");
        registry.addViewController("/chat").setViewName("/chat");
    }
}
```
3. WebSocket 配置
``` java
package com.springboot.websocket.config;

import com.springboot.websocket.common.WebSocketInterceptor;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

/**
 * @Name: WebSocketConfig
 * @Desc: WebSocket配置
 **/

@Configuration
@EnableWebSocketMessageBroker   //开启使用STOMP协议来传输基于代理(message broker)的消息
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    //注册 WebSocketp客户端连接端点(路径)
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        //注册一个WebSocket（或SockJS）客户端进行WebSocket握手而需要连接的端点, 使用SockJS协议。
        registry.addEndpoint("/endpointChat")
                .addInterceptors(new WebSocketInterceptor())
                .withSockJS();
    }

    //配置消息代理(MessageBroker)
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        //目标路径"/im"开头的STOMP消息将路由到@Controller类中的@MessageMapping方法;
        //此处若配置了, 则前端客户端发送消息的路径需加上前缀作为开头
        registry.setApplicationDestinationPrefixes("/im");

        //使用内置的消息代理进行订阅和广播; 将目标标头以“/ topic”或“/ queue”开头的消息路由到代理。
        //"/topic"广播; "/queue"点对点
        registry.enableSimpleBroker("/queue");

        //自定义发送消息路径前缀, 服务端使用SimpMessagingTemplate发送消息的路径默认前缀是/user, 客户端订阅消息的路径必须加上此前缀
//        registry.setUserDestinationPrefix("/msg");

    }
}
```
4. WebSecurity 配置
项目中用了 Spring Security 来做用户验证, 需添加 WebSecurity 配置
``` java
package com.springboot.websocket.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.builders.WebSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;

@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity httpSecurity) throws Exception {
        httpSecurity.authorizeRequests()
                .antMatchers("/", "/login").permitAll()      //匹配路径,允许任何人访问(不拦截,放行)
                .anyRequest().authenticated()                //任何请求映射需要身份认证才可访问
                .and()
                .formLogin().loginPage("/login")             //表单认证,定义登录url
                .defaultSuccessUrl("/chat")                  //认证成功后默认跳转访问的url
                .permitAll()
                .and()
                .logout()                                    //退出登录
                .permitAll();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        //内置允许通过认证的用户, Spring Boot 2.0.x集成的是 Spring Security 5.0.x , 对认证的密码需要加密处理，否则会报错
        auth.inMemoryAuthentication().passwordEncoder(new BCryptPasswordEncoder())
                .withUser("tom").password(new BCryptPasswordEncoder().encode("123456")).roles("USER")
                .and()
                .withUser("kitty").password(new BCryptPasswordEncoder().encode("112233")).roles("USER");
    }

    @Override
    public void configure(WebSecurity webSecurity) throws Exception {
        //忽略静态资源目录,不拦截
        webSecurity.ignoring().antMatchers("/resources/static/**");
    }
}
```
5. WebSocket拦截器(可不用)
``` java
package com.springboot.websocket.common;

import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.http.server.ServletServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.server.support.HttpSessionHandshakeInterceptor;

import javax.servlet.http.HttpSession;
import java.util.Map;

/**
 * @Name: WebSocketInterceptor
 * @Desc: TODO
 **/
@Component
public class WebSocketInterceptor extends HttpSessionHandshakeInterceptor {

    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Map<String, Object> attributes) throws Exception {
        System.out.println("=====================握手前拦截===========================");
        if (request instanceof ServletServerHttpRequest) {
            ServletServerHttpRequest servletRequest = (ServletServerHttpRequest) request;
            HttpSession session = servletRequest.getServletRequest().getSession(false);
           
			//-----------------------------------
        }

        return super.beforeHandshake(request, response, wsHandler, attributes);
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Exception exception) {
        super.afterHandshake(request, response, wsHandler, exception);
    }
}
```
6. WebSocketController 控制器
``` java
package com.springboot.websocket.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;

import java.security.Principal;

/**
 * @Name: WebSocketController
 * @Desc: 消息控制器
 **/
@Controller
public class WebSocketController {

    //注入简单消息模板给客户端,用于发送消息,默认有个目标前缀/user,可自定义
    @Autowired
    private SimpMessagingTemplate simpMessagingTemplate;

    @MessageMapping("/chat")
    public void handleChat(Principal principal,String msg){
        //下面的代码就是如果发送人是Michael，接收人就是Janet，发送的信息是message，反之亦然。
        if(principal.getName().equals("tom")){
            //通过SimpMessagingTemplate的convertAndSendToUser向用户发送消息。
            //第一参数表示接收信息的用户，第二个是浏览器订阅的地址，第三个是消息本身
            simpMessagingTemplate.convertAndSendToUser("kitty","/queue/notifications",
                    principal.getName() + "-发送:" + msg);
        } else {
            simpMessagingTemplate.convertAndSendToUser("tom","/queue/notifications",
                    principal.getName() + "-发送:" + msg);
        }
    }
}
```
7. 在 resources/static/js 目录下添加静态资源
``` 
sockjs.min.js
stomp.js
jquery-3.1.1.js
```
8. 在 resouces/templates 下添加模板页面,Demo 中使用的是 thymeleaf 模板
	**登录页面：**login.html
	``` html
	<!DOCTYPE html>
	<html lang="en" xmlns:th="http://www.w3.org/1999/xhtml">
	<head>
	    <meta charset="UTF-8"/>
	    <title>聊天室登录页面</title>
	</head>
	<body>
	<div th:if="${param.error}">
	    无效的账号和密码
	</div>
	<div th:if="${param.logout}">
	    您已注销
	</div>
	<form th:action="@{/login}" method="post">
	    <div><label>账号：<input type="text" name="username"/></label></div>
	    <div><label>密码：<input type="password" name="password"/></label></div>
	    <div><input type="submit" value="登录"/></div>
	</form>
	</body>
	</html>
	```
	**聊天室页面：**chat.html
	``` html
	<!DOCTYPE html>
	<html xmlns:th="http://www.thymeleaf.org">
	<head>
	    <meta charset="UTF-8"/>
	    <title>聊天页面</title>
	    <script th:src="@{js/sockjs.min.js}"></script>
	    <script th:src="@{js/stomp.js}"></script>
	    <script th:src="@{js/jquery-3.1.1.js}"></script>
	</head>
	<body>
	<p>
	    聊天室
	</p>
	<p>
	    <input type="button" id="disConnect" name="disConnect" onclick="disConnect()" value="断开连接">
	</p>
	
	<form id="chatForm">
	    <textarea rows="4" cols="60" id="chatMsgText" name="text"></textarea>
	    <input type="submit"/>
	</form>
	
	<div id="output"></div>
	
	</body>
	<script th:inline="javascript">
	    $('#chatForm').submit(function (e) {
	        e.preventDefault();
	        // var text = $('#chatForm').find('textarea[name="text"]').val();
	        var text = $("#chatMsgText").val();
	        console.log("msg content:" + text);
	        sendSpittle(text);
	    });
	
	    // 连接endpoint为"/endpointChat"的节点
	    var sock = new SockJS("/endpointChat");
	    var stomp = Stomp.over(sock);
	
	    //连接WebSocket服务端
	    stomp.connect('guest', 'guest', function (frame) {
	        // 订阅/user/queue/notifications发送的消息, 多了一个/user，并且这个user是必须的，
	        // 因为SimpMessagingTemplate发送消息默认会在路径最前面添加前缀,默认的前缀就是 /user
	        stomp.subscribe("/user/queue/notifications", handleNotification);
	    });
	
	    function handleNotification(message) {
	        $('#output').append("<b>收到了:" + message.body + "</b><br/>")
	    }
	
	    function sendSpittle(text) {
	        // 表示向后端路径发送消息请求,
	        // /im 是后端配置的路由前缀, 会被路由到@Controller类下 @MessageMapping 注解的控制器,
	        // /chat 是映射到控制器的路径
	        stomp.send("/im/chat", {}, text);
	    }
	
	    function disConnect() {
	        sock.close();
	    }
	
	</script>
	</html>
	```
9. 开启两个浏览器，登录 http://localhost:8080/login
输入 WebSecurityConfig 中内置认置的两个用户名和密码进行登录, 登录成功会跳转到**聊天室(chat)**页面, 发送消息;两个浏览器的页面可以收到并显示对方发送的消息。

10. 问题
把页面改造成 JSP 或纯 HTML，在点击登录后不跳转到**聊天室**页面，跳转回到登录页面, 没有报错, 试过多种方式未能解决，待详细研究 Spring Security 后再回头来看问题。
**解决**：研究 Spring Security 终于确定问题了, Spring Security 默认是开启 CSRF 防护, 是通过给前端隐藏属性(`_csrf`)传递一个 Token 值, 提交表单时把这个值一起提交来确认是否同一个站点, 自定义 WebSecurityConfig 配置若没有设置 CSRF 则请求不会通过, 可以关闭`csrf`或配置`CSRF`的 Token 生成策略。
```
httpSecurity.csrf().disable()
         .authorizeRequests()
```


【源码 -> GitHub】
1. [spring-boot-websocket-queue](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-websocket-queue)
2. [spring-boot-websocket-topic](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-websocket-topic)