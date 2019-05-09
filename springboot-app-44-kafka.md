---
title: Spring Boot 2实践系列(四十四)：集成 Kafka 消息中间件
comments: true
date: 2019-05-09 08:51:00
tags: [kafka,stream,消息]
categories: [Spring Boot 2实践系列]
---

　　spring-kafka 为支持 Apache Kafka 提供了自动配置。Spring Boot 集成 Kafka 的配置由 `spring.kafka.*` 属性控制。
<!-- more -->

## Kafka 服务

Kafka 服务安装与运行参考 [Kafka系列(一)：Kafka 介绍和安装运行、发布订阅](http://112.74.59.39:90/2019/05/08/kafka-01-install/)。

## 集成 Kafka

### 添加依赖

**pom.xml 导入 spring-kafka 包**

``` xml
<!--Kafka-->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

### 添加配置

在 **application.properties** 配置文件中添加连接 kafka 服务器的配置。

``` properties
spring.kafka.bootstrap-servers=10.0.3.4:9092
# 必须，消费者监听需要指定 group-id
spring.kafka.consumer.group-id=myGroup
```

### 创建主题

创建一个 `NewTopic` 类型的 Bean，如果 topic 已存在，则会忽略。

``` java
@Configuration
public class KafkaConfig {

    @Bean
    public NewTopic newTopic(){
        return new NewTopic("NBA",1, (short) 1);
    }
}
```

### 发送消息

Spring Boot 为 Kafka 提供了 KafkaTemplate 自动配置，可以直接注入使用。

``` java
@RestController
@RequestMapping("/kafka")
public class SendController {

    private static final Logger logger = LogManager.getLogger(SendController.class);

    @Autowired
    private KafkaTemplate kafkaTemplate;

    @RequestMapping("/topic/{msg}")
    public void sendMsg(@PathVariable String msg) {
        ListenableFuture result = kafkaTemplate.send("NBA", msg);
        result.addCallback(o -> System.out.println("send msg success"),
                throwable -> System.out.println("send msg fail"));
    }
}
```

如果定义了 *spring.kafka.producer.transaction-id-prefix* 属性，则会自动配置 *KafkaTransactionManager* 。如果自定义了 *RecordMessageConverter* ，则会自动关联到自动配置的 *KafkaTemplate*。

### 接收消息

如果是个完整的 spring-kafka，则任何 Bean 上可以使用 *@KafkaListener* 注解创建一个监听端点。

``` java
@Component
public class MyConsumer {

    @KafkaListener(topics = "NBA", groupId = "${spring.kafka.consumer.group-id}")
    public void processMessage(String content) {
        System.out.println(content);
    }
}
```

如果定义了 *KafkaTransactionManager* Bean，则会自动关联到容器工厂(ContainerFactory)。同样，如果自定义了 *RecordMessageConverter*，*ErrorHandler*， *AfterRollbackProcessor* Bean，也会自动关联到默认工厂。

自定义 *ChainedKafkaTransactionManager* 必须添加 *@primary* 注解，因为它通常引用自动配置的 *kafktransactionmanager*  Bean。

## Kafka Streams

Spring 为 Kafka 提供了工厂 Bean 来创建 *StreamsBuilder* 对象来管理其生命周期。

只要在类路径下存在 *kafka-streams* 依赖，并且使用 *@EnableKafkaStreams* 注解启用了 Kafka Streams，Spring Boot 就会自动配置必要的 *KafkaStreamsConfiguration* Bean。

``` xml
<!--kafka streams-->
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams</artifactId>
</dependency>
```

启用了 Kafka Streams，意味着必须设置 *application id* 和 *bootstrap server* ，前者可通过 *spring.kafka.streams.application-id* 设置，如果未设置则默认使用 *spring.application.name*；后者可以全局设置或忖为流重写。

要使用 Factory Bean，只需在自定义的 *KStream* 类型的 Bean，使用 *StreamsBuilder* 构建，如下示例：

``` java
@Configuration
@EnableKafkaStreams
static class KafkaStreamsExampleConfiguration {

	@Bean
	public KStream<Integer, String> kStream(StreamsBuilder streamsBuilder) {
		KStream<Integer, String> stream = streamsBuilder.stream("ks1In");
		stream.map((k, v) -> new KeyValue<>(k, v.toUpperCase())).to("ks1Out",
				Produced.with(Serdes.Integer(), new JsonSerde<>()));
		return stream;
	}
}
```

默认情况下，*StreamBuilder* 对象在应用启动中就被创建，就会自动接管 streams。也可使用 *spring.kafka.streams.auto-startup* 属性来自定义此行为。

## 其它属性

请参考官方文档 [Additional Kafka Properties](https://docs.spring.io/spring-boot/docs/2.1.4.RELEASE/reference/htmlsingle/#boot-features-kafka-extra-props)。