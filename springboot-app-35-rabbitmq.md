---
title: Spring Boot 2实践系列(三十五)：集成 RabbitMQ 消息中间件
comments: true
date: 2018-10-27 08:35:48
tags: [MQ,JMS,AMQP,RabbitMQ]
categories: [Spring Boot 2实践系列]
---
　　Spring AMQP 默认支持 RabbitMQ 作为 AMQP 协议的实现，因为...RabbitMQ 和 Spring 是同一家软件公司开发的。

　　Spring Boot 对 RabbitMQ 的支持也是基于 Spring AMQP。为 RabbitMQ 提供了自动配置，可以直接使用`rabbitTemplate`，自动开启了消息监听注解`@EnableRabbit`。

　　更多关于消息服务概念和支持的组件可阅读[Spring Boot 2实践系列(三十三)：JMS 和 AMQP 消息服务及支持的消息组件](http://112.74.59.39/2018/10/16/springboot-app-33-spring-jms-mq/)。
　　
<!-- more -->
## RabbitMQ 简介 ##
　　RabbitMQ 是一个基于 AMQP 协议的轻量级，可靠，可扩展且可移植的消息代理; 支持多种消息传输协议、消息队列、传输确认、到队列的灵活路由、多种交换类型等。

　　支持多种语言开发的客户端，可集群部署以实现高可用和大吞吐量。是部署最广泛、最受欢迎的开源消息代理【- -译自官方描述】。

　　[Spring AMQP 参考](https://docs.spring.io/spring-amqp/docs/current/reference/htmlsingle/)，[RabbitMQ 官网](http://www.rabbitmq.com/)，[RabbitMQ Download](http://www.rabbitmq.com/download.html)，[RabbitMQ Doc](http://www.rabbitmq.com/documentation.html)，[RabbitMQ Tutorials](http://www.rabbitmq.com/getstarted.html), [Management Plugin](http://www.rabbitmq.com/management.html)。

## RabbitMQ 安装 ##
以 Ubuntu 18.x 安装 RabbitMQ 为例，根据官方安装说明来执行安装操作。

1. 下载软件包前先在服务器添加用于签署 RabbitMQ 版本的密钥,否则会报警告
```
apt-key adv --keyserver "hkps.pool.sks-keyservers.net" --recv-keys "0x6B73A36E6026DFCA"
wget -O - "https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc" | sudo apt-key add -
```

2. RabbitMQ 依赖面向高并发的语言 Erlang,所以需要安装 Erlang 运行环境。
官方有提供安装 Erlang 运行环境的安装步骤。但个人不建议手动安装 Erlang 运行环境，因为后续安装完 RabbitMQ 后，各种依赖和版本冲突问题，也不好定位，rabbitmq-server 无法启动，所以建议下载 RabbitMQ 全量包(rabbitmq-server_3.7.8-1_all.deb)，会自动下载 RabbitMQ 和所有依赖。

3. 下载 RabbitMQ 全量包，以[rabbitmq-server_3.7.8-1_all.deb](https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.7.8/rabbitmq-server_3.7.8-1_all.deb)为例。
> 执行安装：
> dpkg -i rabbitmq-server_3.7.8-1_all.deb
> 
> 会报缺少依赖，执行全量安装，会自动下载所有依赖并安装
> apt-get install -f

4. 安装完后，rabbitmq-server 服务会自动运行, 查看 rabbitmq-server 进程和状态
> ps -ef|grep rabbitmq
> service rabbitmq-server status //查看状态，显示 Active 表示活动运行中。

5. 服务启动、停止、重启、查看状态操作
> service rabbitmq-server start
> service rabbitmq-server status
> service rabbitmq-server restart
> service rabbitmq-server stop
>
> 查看 rabbitmq 状态，包括当前的配置信息
> rabbitmqctl status

6. 安装 Rabbitmq 管理插件，提供基于 HTTP API 对 Rabbitmq 节点和集群进行管理和监控。管理插件包括基于浏览器的 UI 和命令行工具 rabbitmqadmin。
> 启用管理插件
> rabbitmq-plugins enable rabbitmq_management

6. 登录 Rabbitmq Web 管理
Web 管理的端口是 **15672**，为本地登录提供了`guest/guest`账号密码，若要远程登录管理，需要添加账号。
> http://rabbitmq_server_ip:15672

7. 添加个 admin 账号用于远程登录
> 添加账号密码
> rabbitmqctl add_user admin 123456  
>
> 查看添加的账号
> rabbitmqctl list_users
>
> 添加权限为管理员,管理员账号可用于 Spring Boot 的连接
> rabbitmqctl set_user_tags admin administrator 
> 
> 分配权限(赋予virtual host中所有资源的配置、写、读权限以便管理其中的资源
```
rabbitmqctl set_permissions -p / admin '.*' '.*' '.*'
```
8. 关于 rabbitmqctl 命令的使用，可使用 man 命令来查看
> man rabbitmqctl

9. 也可以直接使用 Rabbitmq 的 Docker 镜像，做好端口映射；省略手动安装 Rabbitmq 服务。

## RabbitMQ 集成 ##
RabbitMQ 的具体使用可参考官方的[RabbitMQ Tutorials](http://www.rabbitmq.com/getstarted.html)。RabbitMQ 通信有六种模式： **Simplest**(一对一的简单队列模式)、**Work queues**(一对多的队列模式)、**Publish/Subscribe**(发布-订阅模式)、**Routing**(路由模式)、**Topics**(主题模式)、**RPC**(RPC调用模式)。

此集成演示最常见的`Queue`队列模式和`Topic`主题模式。RabbitMQ的功能非常强大，可以单独开个系列文章来分析和实践。
### 配置 ###
1. **pom.xml 添加依赖**
``` xml
<!--AMQP-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```
2. application.properties 配置
``` properties
#---------RabbitMQ-------------------------
spring.rabbitmq.host=192.168.220.128
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=123456
spring.rabbitmq.virtual-host=/
```

### Queues ###
1. 添加 Queue Bean 配置
``` java
/**
 * @name: QueueRabbitMQConfig
 * @desc: 消息组件
 **/
@Configuration
public class QueueRabbitMQConfig {

    private static final String QUEUE_NAME_1 = "my-queue";
    private static final String QUEUE_NAME_2 = "object";

    @Bean
    public Queue strQueue(){
        return new Queue(QUEUE_NAME_1);
    }

    @Bean
    public Queue objQueue(){
        return new Queue(QUEUE_NAME_2);
    }
}
```
2. 消息生产者
``` java
/**
 * @name: QueueProducerServiceImpl
 * @desc: 生产发送消息
 **/
@Service
public class QueueProducerServiceImpl implements QueueProducerService {

    @Autowired
    private RabbitTemplate rabbitTemplate;


    /**
     * 发送普通消息到队列-queue
     * @param msg
     */
    @Override
    public void rabbitMQSendStr(String msg) {
        for (int i = 0; i < 100; i++) {
            System.out.println(i);
            rabbitTemplate.convertAndSend("my-queue", System.nanoTime() + "__" +i);
        }
    }

    /**
     * 发送对象,对象必须实现序列化接口
     * @param user
     */
    @Override
    public void rabbitMQSendObj() {
        this.rabbitTemplate.convertAndSend("object", new User("Andy",33));
    }
}
```
3. 消息消费者
``` java
/**
 * @name: MQConsumerServiceImpl
 * @desc: 消息消费者
 **/
@Service
public class QueueConsumerServiceImpl {

    /**
     * 多个消费者订阅 queue 类型同一个渠道的消息，
     * 接收消息的顺序与发送消息的顺序并不相同，
     * 多个消费者平均消费消息。
     */

    /**
     * 监听my-queue 文本消息
     */
    @RabbitListener(queues = "my-queue")
    public void rabbitMQReceive1(String msg) {
        System.out.println("client1 receive my-queue msg:" + msg);
    }

    @RabbitListener(queues = "my-queue")
    public void rabbitMQReceive2(String msg) {
        System.out.println("client2 receive my-queue msg:" + msg);
    }

    @RabbitListener(queues = "my-queue")
    public void rabbitMQReceive3(String msg) {
        System.out.println("client3 receive my-queue msg:" + msg);
    }

    /**
     * 监听user 对象消息消息
     */
    @RabbitListener(queues = "object")
    public void rabbitMQReceiveUser(User user) {
        System.out.println("receive user msg:" + user);
    }
}
```
多个消费者订阅 `queue` 类型同一个渠道的消息，多个消费者平均消费消息(轮询)，接收消息的顺序与发送消息的顺序并不相同。
Spring Boot 集成 RabbitMQ，只要对象实现了序列化接口，可直接发布对象消息。 

### Topics ###
**Topic**主题模式在消息生产者和消息消费者之间增加了`Exchange`(交换器)，将消息生产者和消息消费者进行了解耦，消息生产者将消息发送给交换器，交换器根据**调度策略**把消息再给到目标队列。

`Exchange` 用于转发消息，不做存储。若没有将 Queue bind 到 Exchange，交换器将直接丢弃消息，若开启了 ack 模式，在找不到队列时会返回错误。

将 Queue 绑定到 Exchange ，涉及三个参数的关联，分别是**queue、exchange、routing_key**，如`BindingBuilder.bind(queueNewsNba).to(exchange).with("topic.news.nba")`。

Queue 绑定到 Exchang 状态可在 Web 管理界面 Queues 选项点击队列名称，在队列概述界面有个 **Bindings** 项，点击展开可查看到当前 queue 绑定到的路由键。

**routing_key**：路由密钥由**点**分隔的多个单词组成，最多可达 255 个字节。例：**news.nba**,**sys.log.error**等。同样 **queue** 名也采用相同的形式。此路由键支持两种模式匹配：
> *(start)可匹配一个单词。
> #(hash)可替换零个或多个单词。

1. Topic 配置(注册 Queue,TopicExchange,Binding)
``` java
/**
 * @name: TopicRabbitMQConfig
 * @desc: Topic 配置
 **/
@Configuration
public class TopicRabbitMQConfig {

    //新闻主题
    private static final String TOPIC_NEWS = "topic.news";
    //NBA新闻主题
    private static final String TOPIC_NEWS_NBA = "topic.news.nba";

    @Bean
    public Queue queueNews() {
        return new Queue(TopicRabbitMQConfig.TOPIC_NEWS);
    }

    @Bean
    public Queue queueNewsNba() {
        return new Queue(TopicRabbitMQConfig.TOPIC_NEWS_NBA);
    }

    /**
     * 注册交换器
     *
     * @return
     */
    @Bean
    public TopicExchange exchange() {
        TopicExchange exchange = new TopicExchange("exchange");
        return exchange;
    }

	/**
     * 队列绑定到交换器,设置匹配的routing_key
     * /

    /**
     * 订阅新闻主题可以收到所有新闻包括NBA
     * @param queueNews
     * @param exchange
     * @return
     */
    @Bean
    public Binding bindingExchangeNews(Queue queueNews, TopicExchange exchange) {
        return BindingBuilder.bind(queueNews).to(exchange).with("topic.news.#");
    }

    /**
     * 订阅新闻下的NBA主题只可以收到NBA新闻
     * @param queueNewsNba
     * @param exchange
     * @return
     */
    @Bean
    public Binding bindingExchangeNewsNba(Queue queueNewsNba, TopicExchange exchange) {
        return BindingBuilder.bind(queueNewsNba).to(exchange).with("topic.news.nba");
    }
}
```
2. 消息生产者
``` java
/**
 * @name: TopicProducerServiceImpl
 * @desc: Topic 生产者
 */
@Service
public class TopicProducerServiceImpl implements TopicProducerService {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Autowired
    private TopicExchange topicExchange;

    @Override
    public void rabbitMQSendStr(String msg) {
        rabbitTemplate.convertAndSend(topicExchange.getName(), "topic.news", "Top Ten News");
    }

    @Override
    public void rabbitMQSendObj(User andy) {
        rabbitTemplate.convertAndSend(topicExchange.getName(), "topic.news.nba", "2018-2019 NBA First Battle Start");
    }
}
```
3. 消息消费者
``` java
/**
 * @name: TopicConsumerServiceImpl
 * @desc: Topic 消费者
 **/
@Service
public class TopicConsumerServiceImpl {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @RabbitListener(queues = "topic.news")
    public void rabbitMQReceive1(String msg) {
        System.out.println("client1 receive topic.news     msg: " + msg);
    }

    @RabbitListener(queues = "topic.news.nba")
    public void rabbitMQReceive2(String msg) {
        System.out.println("client2 receive topic.news.nba msg: " + msg);
    }
}
```
[springboot-rabbitmq -> 源码Github](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-jms-rabbitmq)