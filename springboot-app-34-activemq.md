---
title: Spring Boot 2实践系列(三十四)：集成 AcitveMQ 消息中间件
comments: true
date: 2018-10-17 23:35:31
tags: [mq,ActiveMQ]
categories: [Spring Boot 2实践系列]
---
　　Spring Boot 为 AcitveMQ 提供了自动配置，可以直接使用`jmsTemplate`，自动开启了消息监听注解`@EnableJms`。

　　更多关于消息服务概念和支持的组件可阅读[Spring Boot 2实践系列(三十三)：JMS 和 AMQP 消息服务及支持的消息组件](http://112.74.59.39/2018/10/16/springboot-app-33-spring-jms-mq/)。
<!-- more -->
## ActiveMQ 简介 ##
ActiveMQ 速度快，支持Java，C，C ++，C＃，Ruby，Perl，Python，PHP等多种跨语言客户端和协议。 具有易于使用的企业集成模式和许多高级特性，同时完全支持JMS 1.1 和 J2EE 1.4，支持 AMQP v1.0协议，支持 MQTT v3.1允许在物联网环境中连接。

ActiveMQ 可以轻松嵌入到Spring应用程序中，并使用Spring的XML配置机制进行配置，Spring Boot 为 ActiveMQ 提供了自动配置。

[Apache ActiveMQ 官网](http://activemq.apache.org/), [Download](http://activemq.apache.org/download.html), [Getting Started](http://activemq.apache.org/version-5-getting-started.html),[Using Apache ActiveMQ](http://activemq.apache.org/using-activemq-5.html)。

## AcitveMQ 安装 ##
1. 从官网下载软件包，这里以 Linux 系统为例。
在 Windows 下载 ActiveMQ 软件包上传到 Linux，或 Linux 服务器直接下载(前提是可连外网) 。
2. 解压软件包
> tar zxvf activemq-x.x.x-bin.tar.gz**
> 
> 以 AcitveMQ v5.15.6 为例
> tar zxvf apache-activemq-5.15.6-bin.tar.gz 

3. 运行 AcitveMQ
	进入 AcitveMQ 解压目录,启动 AcitveMQ
	> cd :/usr/local/apache-activemq-5.15.6/bin/
	> ./activemq start
	
	若出现无法启动，查看日志：
	> cd apache-activemq-5.15.6/data/
	> cat activemq.log
	
	查看最近的异常信息，若出现如下异常： 
	> Exception encountered during context initialization - cancelling refresh attempt: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'org.apache.activemq.xbean.XBeanBrokerService#0' defined in class path resource [activemq.xml]: Invocation of init method failed; nested exception is java.net.URISyntaxException: Illegal character in hostname at index 9: **ws://dev_linux**:61614?maximumConnections=1000&wireFormat.maxFrameSize=104857600 | org.apache.activemq.xbean.XBeanBrokerFactory$1 | main
	> 这里面的 **dev_linux** 服务器名，在 **activemq.xml** 配置文件里默认配置的是 **0.0.0.0** 的IP地址，估计是**ws**协议不支持这种转换。
	
	修改配置文件：activemq.xml
	> cd apache-activemq-5.15.6/conf/
	> vim activemq.xml
	> 搜索： /ws://0.0.0.0
	> 修改 ws://0.0.0.0 为 ws://127.0.0.1/ 保存
	
	> 再启动

3. 停止 AcitveMQ
> ./activemq stop

4. 查看 AcitveMQ 相关信息和命令
> ./activemq 不带任何参数

5. AcitveMQ 自带了 Web 管理端，通过浏览器访问，Web 访问端口默认是 **8162**。
> 浏览器输入：http://ip:8162 , 默认账号密码是 **admin/admin**。

7. 配置连接密码，供 Spring Boot 集成访问
修改 conf 目录中 credentials.properties 文件进行密码设置,此配置被 **activemq.xml** 引用。
```
activemq.username=root
activemq.password=123456
guest.password=123456
```

8. 也可以直接使用 AcitveMQ 的 Docker 镜像，做好端口映射；省略手动安装 AcitveMQ 服务。

[**其它参考：**]
[快速搭建ActiveMQ服务-Docker方式](https://mp.weixin.qq.com/s/QRCs_AFzTJNgp9B2E6a5zw)

## AcitveMQ 集成 ##
1. pomlxml 添加依赖
如果要配置连接池，必须添加`activemq-pool`依赖，否则报错：找不到 JmsTemplate Bean
``` xml
<!--ActiveMQ-->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-activemq</artifactId>
</dependency>
<!--activemq-pool-->
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-pool</artifactId>
</dependency>
```
2. properties 配置文件添加配置
Spring Boot 自动配置默认开启的消息模式是`Queue`队列点对点模式；如果要使用`Topic`发布/订阅模式(Pub/Sub),则将 `spring.jms.pub-sub-domain=`改为`true`。
```
#---------Spring JMS-----------------------
##----默认为false,queue(点对点)模式; 修为true,则是topic(发布/订阅模式)
#spring.jms.pub-sub-domain=false
#---------ActiveMQ-------------------------
spring.activemq.broker-url=tcp://10.0.3.4:61616
spring.activemq.user=root
spring.activemq.password=123456
spring.activemq.in-memory=false
spring.activemq.pool.enabled=true
spring.activemq.pool.max-connections=10
spring.activemq.pool.idle-timeout=30
```
3. ActiveMQ 配置类
自动配置的方式不支持`Queue`和`Topic`同时使用,若在一个项目里要同时使用这两种模式，则需要自定义一个 `JmsListenerContainerFactory` Bean，设置 `pub-sub-domain`为`true`，`Topic`监听注解添加`containerFactory`属性，指向自定义开启**Topic**的 **JmsListenerContainerFactory**。
``` java
/**
 * @name: ActiveMQConfig
 * @desc: 配置类
 **/
@Configuration
public class ActiveMQConfig {

    @Bean
    public JmsListenerContainerFactory<?> topicListenerContainer(ConnectionFactory activeMQConnectionFactory) {
        DefaultJmsListenerContainerFactory topicListenerContainer = new DefaultJmsListenerContainerFactory();
        topicListenerContainer.setPubSubDomain(true);
        topicListenerContainer.setConnectionFactory(activeMQConnectionFactory);
        return topicListenerContainer;
    }
}
```
4. 消息生产者:producer
``` java
/**
 * @name: MQSendServiceImpl
 * @desc: TODO
 **/
@Service
public class MQProducerServiceImpl implements MQProducerService {

    @Autowired
    private JmsTemplate jmsTemplate;

    /**
     * 发送queue消息
     *
     * @param msg
     * @throws JMSException
     */
    @Override
    public void activeMQSend(String msg) throws JMSException {

        MessageCreator messageCreator = session -> session.createTextMessage(msg);

        //发布queue
        Destination queueDestination = new ActiveMQQueue("my-queue");
        jmsTemplate.send(queueDestination, messageCreator);
        jmsTemplate.convertAndSend(queueDestination, "Hello Queue");

        //发布topic
        Destination topicDestination = new ActiveMQTopic("my-topic");
        jmsTemplate.send(topicDestination, messageCreator);
        jmsTemplate.convertAndSend(queueDestination, "Hello Topic");

    }
}
```
5. 消费者监听消息:consumer
``` java
/**
 * @name: MQConsumerServiceImpl
 * @desc: TODO
 **/
@Service
public class MQConsumerServiceImpl implements MQConsumerService {

    /**
     * 监听queue消息
     * 
     * @param message
     */
    @Override
    @JmsListener(destination = "my-queue")
    public void activeMQQueueReceive(String message) {
        System.out.println("监听收到my-queue消息:" + message);
    }

    /**
     * 监听topic消息
     * 一个项目同时使用 Queue 和 Topic, Topic 监听注解添加`containerFactory`属性，
     * 指向自定义开启**Topic**的 **JmsListenerContainerFactory**。
     * @param message
     */
    @Override
    @JmsListener(destination = "my-topic", containerFactory = "topicListenerContainer")
    public void activeMQTopicReceive(String message) {
        System.out.println("监听收到my-topic消息:" + message);
    }
}
```

