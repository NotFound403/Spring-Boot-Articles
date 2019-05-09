---
title: Spring Boot 2实践系列(二十五)：WebSocket详解与使用
comments: true
date: 2018-07-25 18:41:44
tags: [websocker,socket,stomp,http,sockjs]
categories: [Spring Boot 2实践系列]
---
　　Spring Boot为内嵌的Tomcat 8.5，Jetty 9和Undertow提供了 WebSockets 自动配置。 如果打包 war 文件部署到独立容器，则Spring Boot会认为容器负责其对WebSocket支持的配置。

　　Spring Framework提供了丰富的WebSocket支持，可以通过spring-boot-starter-websocket模块轻松的集成使用。

　　[Spring Boot -> WebSockets](https://docs.spring.io/spring-boot/docs/2.0.3.RELEASE/reference/htmlsingle/#boot-features-websockets), [Spring Framework -> Web on Servlet Stack -> WebSockets](https://docs.spring.io/spring/docs/5.0.7.RELEASE/spring-framework-reference/web.html#websocket)。
<!-- more -->
## 前言 ##
　　**B/S**架构的 Web 应用是基于 HTTP 协议通信的，而 HTTP 协议具有`非连接、无状态`的特性，即每次的连接只处理一个请求，客户端发起请求，服务器处理请求响应给客户端后，连接就会断开(请求时建立连接，请求完释放连接), 释放的资源可以服务其它请求，这可以充分利用资源; 基于HTTP协议的通信, 请求只能由 Web 客户端主动发起, 服务器被动响应完后并不知道客户端的状态。

　　在 HTTP 和 REST中，服务器应用程序被建模成多个URL。客户端要与服务器交互，请求访问这些 URL，请求-响应的模式。服务器根据 HTTP URL，方式和头信息将请求路由到适当的处理器。

　　但某些的业务场景需要服务器主动给客户端发送消息, 如：给Web页面发送消息通知, 或在页面进行的即时通信等, HTTP就不能满足, 需求推动技术创新，于是 WebSocket 技术诞生了。　　

## WebSockets ##
`WebSocket`协议[**RFC 6455**](https://tools.ietf.org/html/rfc6455)提供了一种标准化方法，可通过单个 **TCP** 连接在客户端和服务器之间建立全双工双向通信通道进行通信。 它是与 **HTTP**不同的**TCP**协议, 但基于 HTTP协议来工作, 使用 **80** 和 **443** 并允许重用现有防火墙规则。

WebSocket 交互以一条 HTTP 请求开始，该请求使用 HTTP 的头属性`Upgrade`来指定传输协议或切换成 WebSocket 协议。在 WebSockets中，通常只有一个URL用于初始连接，随后所有交互消息都在同一条TCP连接上流动。

WebSocket 是一种低级传输协议，与HTTP不同，它没有规定消息内容的任何语义，意味着客户端和服务器对交互的消息语意需要达成一致，即需要用户自定义应用层面的通信协议，否则无法路由和处理消息。

WebSocket 客户端和服务器可以通过 HTTP 握手请求上的"Sec-WebSocket-Protocol"标头协商使用更高级别的消息传递协议（例如STOMP），或自定义协议。

WebSocket头信息示例：
1. **General**
``` 
Request URL: ws://localhost/websocket/218/1whk0bls/websocket
Request Method: GET
Status Code: 101 
```
2. **Request Headers:请求头**
``` 
GET ws://localhost/websocket/218/1whk0bls/websocket HTTP/1.1
Host: localhost
Connection: Upgrade
Pragma: no-cache
Cache-Control: no-cache
User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Safari/537.36
Upgrade: websocket
Origin: http://localhost
Sec-WebSocket-Version: 13
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Cookie: JSESSIONID=03831878435AD7A7D489400925651B26
Sec-WebSocket-Key: v34AniZxaloYVda1yJKcXw==
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
```
3. WebSocket广播模式的**Response Headers:响应头**，不是通常的200状态码
```
HTTP/1.1 101
Upgrade: websocket
Connection: upgrade
Sec-WebSocket-Accept: xa3vVWA3TnIDR+lG2GcBWMFMvLU=
Sec-WebSocket-Extensions: permessage-deflate;client_max_window_bits=15
Date: Thu, 23 Aug 2018 23:07:29 GMT
```
4. WebSocket点对点模式的**Response Headers**
``` 
HTTP/1.1 101
Upgrade: websocket
Connection: upgrade
Sec-WebSocket-Accept: GdjMsIjRgN8BAbgkk7c1gsLhs64=
Sec-WebSocket-Extensions: permessage-deflate;client_max_window_bits=15
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
X-Frame-Options: DENY
Date: Wed, 29 Aug 2018 05:41:43 GMT
```
Spring Framework提供了一个WebSocket API，可用于编写处理WebSocket消息的客户端和服务器端应用程序。

## STOMP ##
在实际开发中，我们较少使用原生的 WebSocket，因为会比较麻烦并繁琐，需要处理浏览器兼容性问题，通常会基于的`STOMP`做开发， STOMP提供更丰富的编程模型。

STOMP是一种简单的，面向文本的消息传递协议。STOMP 可用于任何可靠的双向流网络协议，如TCP和WebSocket。 虽然STOMP是面向文本的协议，也可以传输文本和二进制消息数据。STOMP是在 HTTP 协议基础上的一种基于帧(frame)的格式的消息定义。

在使用 Spring 的 STOMP 支持时，Spring WebSocket 应用程序充当客户端的STOMP代理。消息被路由到 @Controller 消息处理方法或者路由到简单的内部消息中间件，该代理跟踪订阅并向订阅用户广播消息。或可使用消息中间件(RabbitMQ，ActiveMQ等)做为 STOMP 代理来实现消息的广播，Spring维护与代理的TCP连接，向其中继消息，并将消息从其传递到连接的WebSocket客户端。

消息交互有两种模式：
1. 广播模式(topic)：发布订阅(一对多)。
2. 点对点模式(queue)：一对一消息交互。

来自服务器的所有消息必须响应特定的客户端订阅，并且服务器消息的“subscription-id”头必须与客户端订阅的“id”头匹配。

关于 WebSocket 使用 STOMP 可通过集成 spring-messaging和spring-websocket模块获得支持，集成了这些依赖，就可以通过带有SockJS Fallback的WebSocket公开 STOMP 端点。在 Spring Boot项目中，只需添加spring-boot-starter-websocket依赖，此依赖包含了 spring-messaging 和 spring-websocket 模块。

### topic(一对多) ###
服务器发送消息，所有订阅到此消息路径的客户端都可以收到该消息。 例：同一个订阅了消息页面在浏览器开启多个窗口，多个窗口都能收到服务器发送的消息。
1. **建立连接**
```
Opening Web Socket...
Web Socket Opened...

>>> CONNECT
accept-version:1.1,1.0
heart-beat:10000,10000

<<< CONNECTED
version:1.1
heart-beat:0,0

connected to server undefined
connected>>>>:CONNECTED
heart-beat:0,0
version:1.1
```
2. **开启订阅**
```
>>> SUBSCRIBE
id:sub-0
destination:/topic/getRespMsg
```
3. **发送消息**
```
>>> SEND
destination:/app/hello
content-length:22

{"msg":"Hello World!"}
```
4. **接收消息**
```
<<< MESSAGE
destination:/topic/getRespMsg
content-type:application/json;charset=UTF-8
subscription:sub-0
message-id:a51ctb4d-6
content-length:66

{
  "resultMsg" : "收到消息：Hello World!! 1535246405591"
}
```
5. **帧消息**
``` 
发送：["SEND\ndestination:/app/hello\ncontent-length:22\n\n{\"msg\":\"Hello World!\"}\u0000"]

接收：a["MESSAGE\ndestination:/topic/getRespMsg\ncontent-type:application/json;charset=UTF-8\nsubscription:sub-0\nmessage-id:a51ctb4d-7\ncontent-length:66\n\n{\r\n  \"resultMsg\" : \"收到消息：Hello World!! 1535246560183\"\r\n}\u0000"]
```

### queue(一对一) ###
消息由谁发送，由谁接收，点对点通信。
1. **建立连接**
```
Opening Web Socket...
Web Socket Opened...
>>> CONNECT
login:guest
passcode:guest
accept-version:1.1,1.0
heart-beat:10000,10000

<<< CONNECTED
version:1.1
heart-beat:0,0
user-name:tom								//连接时带上了用了名
```
2. **开启订阅**
``` 
connected to server undefined
>>> SUBSCRIBE
id:sub-0
destination:/user/queue/notifications		//订阅地址
```
3. **发送消息**
```
>>> SEND
destination:/im/chat						//发送消息目标地址
content-length:12

Hello World!
```
4. **接收消息**
```
<<< MESSAGE
destination:/user/queue/notifications
content-type:text/plain;charset=UTF-8
subscription:sub-0
message-id:lm5q38dv-1
content-length:22

kitty-发送:Welcome!!							//消息内容
```
5. **帧数据**
```
发起连接：["CONNECT\nlogin:guest\npasscode:guest\naccept-version:1.1,1.0\nheart-beat:10000,10000\n\n\u0000"]
连接返回：a["CONNECTED\nversion:1.1\nheart-beat:0,0\nuser-name:tom\n\n\u0000"]
发起订阅：["SUBSCRIBE\nid:sub-0\ndestination:/user/queue/notifications\n\n\u0000"]
发送消息：["SEND\ndestination:/im/chat\ncontent-length:12\n\nHello World!\u0000"]
接收消息：a["MESSAGE\ndestination:/user/queue/notifications\ncontent-type:text/plain;charset=UTF-8\nsubscription:sub-0\nmessage-id:lm5q38dv-1\ncontent-length:22\n\nkitty-发送:Welcome!!\u0000"]
```
5. **WebSocket连接端点的响应信息**
```
info：http://localhost/endpointChat/info

Response：{"entropy":-422932604,"origins":["*:*"],"cookie_needed":true,"websocket":true}
```