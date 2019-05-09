---
title: Spring Boot 2实践系列(三十三)：JMS 和 AMQP 消息服务及支持的消息组件
comments: true
date: 2018-10-16 16:38:33
tags: [JMS,MQ,AMQP,ActiveMQ,RabbitMQ,Kafka]
categories: [Spring Boot 2实践系列]
---
　　**消息组件**在现在的互联网应用系统已广泛使用，特别是在大型的、分布式或微服务架构中，要协调系统之间的通信，消息组件几乎是不可或缺的。

　　使用**消息中间件**可实现系统之间的**异步通信**、可对服务之间的调用进行**解耦**、可对并发请求实现**流量消峰**、可用于**消息通讯**。

　　Spring Framework 为与消息组件的集成提供了广泛的支持, 从简化使用`JMS API`的`JmsTemplate`到完整的异步接收消息的基础架构。 Spring AMQP还为高级消息队列协议提供了类似的功能集。

　　Spring Boot 默认就为 ActiveMQ、 RabbitMQ、 Kafka、 Artemis 提供自动配置支持。[Spring AMQP 官方文档](http://spring.io/projects/spring-amqp#learn)，[Spring Boot Message 文档](https://docs.spring.io/spring-boot/docs/2.0.5.RELEASE/reference/htmlsingle/#boot-features-messaging)。
<!-- more -->
## 异步消息 ##
1. **异步消息：**即消息发送者无须等待消息接收者的处理及返回。
2. 两个重要概念：**消息代理(message broke)**和**目的地(destination)**。当消息发送者发送消息后，消息由**消息代理**负责传递到**目的地**。
3. 消息通信两种模式：**队列(queue)**和**主题(topic)**。队列用于**点对点**通信，主题用于**发布/订阅(publish/subscribe)**的消息通信。
> **点对点模式：**
消息代理获得消息后将消息放入一个`队列(queue)`中，当有消息接收者来接收消息时，将消息从队列中取出传递给消费者，队列里就没有了这条消息。点对点模式确保每一条消息只有唯一的发送者和接收者，消息一旦被消费，消息就消失，消息在被消费之前会一直等待直到被消费，如果有多个消费同时接收一条消息，遵循先来后到原则。
>
> **发布/订阅模式：**
消息发送者将消息以`主题`方式发送，允许有多个消息消费者监听这个**主题**，订阅一个主题的消费者只能消费自它文本阅之后发布的消息，类似于广播的方式，接收者必须先订阅。

## JMS & AMQP ##
**JMS**(Java Message Service:Java消息服务)：是基于`JVM`消息代理的规范，而 ActiveMQ、 HornetQ 是一个 JMS 消息代理的实现。

JMS 的基本组成部分：
1. **连接工厂**：是客户用来创建连接的对象。
javax.jms.ConnectionFactory 接口提供了一种创建 **javax.jms.Connection** 来与**JMS代理**进行交互的标准方法。虽然Spring需要一个 ConnectionFactory 来与 JMS 一起工作，但通常不需要直接使用它，而是可以依赖更高级别的消息传递抽象。 Spring Boot 还自动配置了发送和接收消息所需的基础结构。如为 ActiveMQ 提供的 ActiveMQConnectionFactory。
2. **连接**：JMS Connection 封装了 JMS 客户端到 JMS Provider 的连接与 JMS 提供者(消息代理服务器)之前的一个虚拟连接。
3. **会话**：JMS Session 是生产和消费消息的一个单线程上下文。会话用于创建消息的生产者(producer)，消费者(consumer)，消息(message)等。
4. **生产者：**MessageProducer,由 Session 对象创建的用来发布消息的对象。
5. **消费者：**MessageConsumer,由 Session 对象创建的用来消费消息的对象。
6. **消息：**Message,JMS消息包括消息头和消息体以及其它的扩展属性，定义的类型有 TextMessage,MapMessage,BytesMessage,StreamMessage和ObjectMessage。
7. **目的地：**Destination，消息的目的地，发布的消息的目的地，消费的消息的来源对象。
8. **消息队列：**Queue,点对点的消息队列。
9. **消息主题：**Topic,发布/订阅的消息队列。

**AMQP**(Advanced Message Queuing Protocol)：是一个更高级的消息代理规范(协议)，它兼容**JMS**，支持跨语言和平台。**AMQP**的主要实现有`RabbitMQ`。
Spring AMQP 项目将 Spring 核心概念应用于基于 **AMQP** 的消息传递解决方案的开发，它提供了一个**模板**作为发送和接收消息的高级抽象。此项目由两部分：spring-amqp 是基础抽象，spring-rabbit 是 RabbitMQ 实现。

Spring 为 **JMS、AMQP** 提供了 **@JmsListener、@RabbitListener**注解在方法上监听消息代理发布的消息。需要通过 **@EnableJms、@EnableRabbit**开启支持。

## 自动配置支持 ##
Spring Boot 为 JMS、 AMQP、 ActiveMQ、 Kafka 提供了自动配置支持，分别在自动配置 org.springframework.boot.autoconfigure 包的 **jms、 amqp、 kafka**的三个目录里。

在 **jms** 包下的 JmsAutoConfiguration 文件注册了 **jmsTemplate**，还开启了注解式消息监听的支持，即自动开启`@EnableJms`注解。

### ActiveMQ ###
Spring Boot 为 ActiveMQ 提供自动配置在**org.springframework.boot.autoconfigure.jms.activemq**包下。自动配置属性以`spring.activemq`为前缀，必配的属性有3个：**broker-url、user、password**, 还支持连接池的配置, 只是连接池不符合 **JMS 2.0**标准，也可自定义实现 **ActiveMQConnectionFactoryCustomizer**接口来获得更高级的功能，还定义了用于连接的 **ActiveMQConnectionFactory** Bean。 [Apache ActiveMQ 官网](http://activemq.apache.org/)。
> spring.activemq.broker-url=tcp://192.168.1.210:9876
> spring.activemq.user=admin
> spring.activemq.password=secret
> spring.activemq.pool.enabled=true
> spring.activemq.pool.max-connections=50

### RabbitMQ ###
Spring Boot 为 RabbitMQ 提供自动配置在**org.springframework.boot.autoconfigure.amqp**包下。自动配置属性以`spring.rabbitmq`为前缀，必配的属性有4个：**host、port、username、password**。
在自动配置`RabbitAutoConfiguration`文件中，创建了用于连接的 **rabbitConnectionFactory** Bean, 创建了模板`rabbitTemplate` Bean。自动配置还开启了注解式消息监听的支持，即自动开启了`@EnableRabbit`注解。
> spring.rabbitmq.host=localhost
> spring.rabbitmq.port=5672
> spring.rabbitmq.username=admin
> spring.rabbitmq.password=secret

### Kafka ###
Spring Boot 通过 spring-kafka 项目的自动配置来支持 Apache Kafka, 自动配置在**org.springframework.boot.autoconfigure.kafka**包下，属性配置前缀是`spring.kafka`。

自动配置创建了`kafkaTemplate` Bean，可以在项目中拿来使用；创建了发送消息时异常的监听器`ProducerListener`；创建了默认的消息消费者`kafkaConsumerFactory`；如果定义了属性`spring.kafka.producer.transaction-id-prefix`，则会自动配置事务管理`kafkaTransactionManager` 。

自动配置开启了注解式消息监听的支持，即自动开启`@EnableKafka`注解。
> spring.kafka.bootstrap-servers=localhost:9092
> spring.kafka.consumer.group-id=myGroup

**Kafka** 可配置的属性比较多，**KafkaProperties**类只提供了部分 **Kafka** 支持的属性，如果想使用不直接支持的其它属性配置生产者或消费者，可以使用以下属性：
> spring.kafka.properties.prop.one=first
> spring.kafka.admin.properties.prop.two=second
> spring.kafka.consumer.properties.prop.three=third
> spring.kafka.producer.properties.prop.four=fourth

还可以按如下方式配置 Spring Kafka JsonDeserializer(反序列化):
> spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
> spring.kafka.consumer.properties.spring.json.value.default.type=com.example.Invoice
> spring.kafka.consumer.properties.spring.json.trusted.packages=com.example,org.acme

**注意：**如果使用如上配置则完全覆盖掉 Spring Boot 自动配置直接支持的属性配置。

【其它参考：】
[RabbitMQ ，RabbitMQ简介，各种MQ选型对比](https://www.sojson.com/blog/48.html)





