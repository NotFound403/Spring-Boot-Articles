---
title: Spring Boot 2实践系列(三十六)：集成 JavaMail 发送邮件
comments: true
date: 2018-10-29 13:24:05
tags: [mail,JavaMail]
categories: [Spring Boot 2实践系列]
---
　　发送电子邮件到用户是项目中一个很常见的功能，如邮件通知、邮件营销、通过邮件激活账号、通过邮件找回密码等。

　　Spring Framework 提供了一个用于发送电子邮件的实用库，保护用户免受底层邮件系统的细节影响。提供了发送电子邮件的简单抽象 JavaMailSender 接口，Spring Boot 为 JavaMail提供了自动配置和启动模块。

　　[Spring Boot 对 Email 的支持官方说明](https://docs.spring.io/spring-boot/docs/2.0.6.RELEASE/reference/htmlsingle/#boot-features-email),有关使用 **JavaMail** 的详细说明，可参阅官方的[参考文档](https://docs.spring.io/spring/docs/5.0.10.RELEASE/spring-framework-reference/integration.html#mail), [JavaMail 参考实现](https://javaee.github.io/javamail/)。
<!-- more -->
## 基本概念 ##
**邮件协议**：目前常用的客户端/服务器协议有两种，分别是 `POP3/SMTP`和`IMAP/SMTP`：
- **SMTP**(Simple Mail Transfer Protocol)：即简单邮件传输协议(发送邮件),一组用于由**源地址**到**目的地址**传送邮件的规范，由它来控制邮件的中转方式，帮助计算机在发送或中转邮件时找到下一个目的地。属于**TCP/IP**协议簇，默认端口是`25`。
- **POP3**(Post Office Protocol - Version 3)：即**邮局协议版本3**，主要用于支持使用客户端管理在服务器上的电子邮件。属于**Tcp/IP**家族，默认端口是`110`。
- **IMAP**(Internet Mail Access Protocol)：Internet邮件访问协议，主要用于支持客户端通过该协议从邮件服务器获取邮件信息、下载邮件等。 **IMAP**协议运行在 **Tcp/IP** 协议之上，默认端口是`143`。

开启 **POP3/SMTP** 服务、**IMAP/SMTP** 服务。通常邮件服务商提供的邮箱设置页面可设置是否开启服务支持，并可查看服务器地址。例如 163 的邮箱设置：
![POP3/IMAP/SMTP](http://112.74.59.39/images/1540889971089.png)

## 自动配置分析 ##
Spring Boot 为 JavaMail 提供的自动配置源码位于 org.springframework.boot.autoconfigure.mail 包下。
自动配置的属性配置以`spring.mail`为前缀，配置如下：
> spring.mail.default-encoding=UTF-8 # Default MimeMessage encoding.
> spring.mail.host= # SMTP server host. For instance, `smtp.example.com`.
> spring.mail.port= # SMTP server port.
> spring.mail.username= # Login user of the SMTP server.
> spring.mail.password= # Login password of the SMTP server.
> spring.mail.properties.*= # Additional JavaMail Session properties.
> spring.mail.protocol=smtp # Protocol used by the SMTP server.
> spring.mail.jndi-name= # Session JNDI name. When set, takes precedence over other Session settings.
> spring.mail.test-connection=false # Whether to test that the mail server is available on startup.

自动配置提供的邮件默认编码是`UTF-8`，默认的协议是`smtp`,源码文件是 MailProperties.java。 若配置文件包含 **spring.mail.host** 属性就会自动注册`mailSender` **Bean**，在使用时可直接注入**mailSender**来发送邮件。

默认的邮件超时是无限的，可以自定义超时时长来避免线程被无响应的邮件服务器阻塞，配置如下：
> spring.mail.properties.mail.smtp.connectiontimeout=5000
> spring.mail.properties.mail.smtp.timeout=3000
> spring.mail.properties.mail.smtp.writetimeout=5000


## 集成JavaMail ##
1. pom.xml 添加依赖
``` xml
<!--mail-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```
2. application.properties 添加配置
``` properties
spring.mail.default-encoding=UTF-8
spring.mail.host=smtp.xxx.com
spring.mail.port=25
spring.mail.username=xxxx@xxx.com
spring.mail.password=xxxxxx
#默认是smtp
spring.mail.protocol=smtp
spring.mail.test-connection=true
spring.mail.properties.mail.smtp.connectiontimeout=5000
spring.mail.properties.mail.smtp.timeout=3000
spring.mail.properties.mail.smtp.writetimeout=5000

# 启动SSL时的配置
#spring.mail.smtp.socketFactory.class=javax.net.ssl.SSLSocketFactory
#spring.mail.smtp.socketFactory.fallback=false
#spring.mail.smtp.socketFactory.port=465
```
3. 发送邮件

发送邮件有些基本元素是需要需要填写：
> **from**：邮件发送者，即源地址，一般设置在配置文件中或做为常量配置。
> **to**：邮件接收者，此参数一般从用户信息中获取，参数可为 String 数组，同时发送多人。
> **subject**：邮件主题。
> **content**：邮件正文(内容)。
> **cc**：抄送给用户，值是邮箱地址(可选)。

构建邮件对象有两种方式：SimpleMailMessage，MimeMessagePreparator。
> **SimpleMailMessage**：简单邮件模型，只包含 from,to,cc,subject,text等几个字段。
> **MimeMessagePreparator**：MIME 邮件的回调接口,当邮件内容出现特殊字符编码时,此接口非常有用。还有一个重要的帮助类**MimeMessageHelper**，该类是 MimeMessageHelper 的帮助类，用于构建复杂的邮件对象，如带附件的邮件，HTML邮件等。

### 简单文本邮件 ###
``` java
/**
 * @name: SimpleOrderManager
 * @desc: 发送邮件
 **/
@Service
public class sendEmailImpl implements sendEmail {

    private static final Logger logger = LogManager.getLogger(sendEmailImpl.class);

    @Autowired
    private JavaMailSender mailSender;

    @Value("${spring.mail.username}")
    private String from;

    /**
     * 发送简单文本邮件
     */
    @Override
    public void sendSimpleMail() {
        User user = new User().setName("Andy").setEmailAddress("xxxx@163.com");

        SimpleMailMessage message = new SimpleMailMessage();
        message.setFrom(from);
        message.setTo(user.getEmailAddress());
        message.setSubject("账号注册激活确认");
        message.setText("请点击下面链接激活注册的账号："
                + "\n\t" + "http://xxxx.xxxx.com?code=xxxxxx");
        message.setCc("xxxx@qq.com");
        try {
            mailSender.send(message);
            logger.info("邮件发送成功！");
        } catch (MailException e) {
            logger.error("邮件发送异常:{}", e);
            e.printStackTrace();
        }
    }

    /**
     * MimeMessagePreparator发送
     */
    @Override
    public void sendMailUseMimeMessagePreparator() {
        User user = new User().setName("Andy").setEmailAddress("xxxx@163.com");

        MimeMessagePreparator preparator = new MimeMessagePreparator() {
            public void prepare(MimeMessage mimeMessage) throws Exception {
                mimeMessage.setRecipient(Message.RecipientType.TO,
                        new InternetAddress(user.getEmailAddress()));
                mimeMessage.setFrom(from);
                mimeMessage.setSubject("幸运观众");
                mimeMessage.setText("你已被抽中为幸运观众,可获得8888元的奖品大礼包！");
            }
        };
        try {
            mailSender.send(preparator);
            logger.info("邮件发送成功！");
        } catch (MailException e) {
            logger.error("邮件发送异常:{}", e);
            e.printStackTrace();
        }
    }

    /**
     * MimeMessageHelper发送
     */
    @Override
    public void sendMailUseMimeMessageHelper() {
        User user = new User().setName("Andy").setEmailAddress("xxxx@163.com");

        MimeMessage message = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(message);
        try {
            helper.setFrom(new InternetAddress(from));
            helper.setTo(user.getEmailAddress());
            helper.setSubject("双色球中奖");
            helper.setText("恭喜你中了双色球一等奖！！");
            mailSender.send(message);
            logger.info("邮件发送成功！");
        } catch (MessagingException e) {
            logger.error("邮件发送异常:{}", e);
            e.printStackTrace();
        }
    }
}
```
### 带附件的邮件 ###
只能使用`MimeMessageHelper`，并开启 **multipart** 模式。注意附件的大小，过大有可能被邮件服务商的拦截掉。
``` java
/**
 * 发送带附件的邮件
 */
@Override
public void sendMailWithAttachments() {
    User user = new User().setName("Andy").setEmailAddress("xxxx@163.com");
    MimeMessage message = mailSender.createMimeMessage();
    try {
        //开启multipart模式
        MimeMessageHelper helper = new MimeMessageHelper(message, true);

        String path = sendEmailImpl.class.getClassLoader().getResource("").getPath();
        File file = new File(path + "/happy.jpg");
        FileSystemResource resource = new FileSystemResource(file);

        helper.setFrom(new InternetAddress(from));
        helper.setTo(user.getEmailAddress());
        helper.setSubject("设计图");
        helper.setText("设计图见附件");
        helper.addAttachment("happy.jpg", resource);
        mailSender.send(message);
        logger.info("邮件发送成功！");
    } catch (MessagingException e) {
        logger.error("邮件发送异常:{}", e);
        e.printStackTrace();
    }
}
```
### HTML邮件 ###
发送 HTML 邮件，并在邮件正文内嵌静态文件(如显示图片)。
``` java
/**
 * HTML邮件,内联静态资源附件
 * helper.setText(content, true);第二个参数设置为true表示html邮件
 */
@Override
public void sendMailInlineResource() {
    User user = new User().setName("Andy").setEmailAddress("xxxx@163.com");
    MimeMessage message = mailSender.createMimeMessage();

    String mailContent = "<html><body>" +
            "<h1>Hello World!</h1>" +
            "<img src='cid:happy.jpg'>" +
            "</body></html>";
    try {
        //开启multipart模式
        MimeMessageHelper helper = new MimeMessageHelper(message, true);

        String path = sendEmailImpl.class.getClassLoader().getResource("").getPath();
        File file = new File(path + "/happy.jpg");
        FileSystemResource resource = new FileSystemResource(file);

        helper.setFrom(new InternetAddress(from));
        helper.setTo(user.getEmailAddress());
        helper.setSubject("设计图");
        helper.setText("设计图见附件");
        //邮件正文显示附件
        helper.setText(mailContent, true);
        helper.addInline("happy.jpg", resource);
        mailSender.send(message);
        logger.info("邮件发送成功！");
    } catch (MessagingException e) {
        logger.error("邮件发送异常:{}", e);
        e.printStackTrace();
    }
}
```

### thymeleaf模板邮件 ###
为固定的场景设置邮件模板，如重置密码、账号激活、营销活动，给每个用户发送的内容只有小部分变化。
这类需求可以使用模板引擎来为各类邮件设置成模板，通过**模板引擎将小部分变化的地方参数化**，返回的是 HTML 代码的字符串，即实际发送的是 HTML 邮件。
1. pom.xml 添加模板依赖
``` java
<!--thymeleaf-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```
2. 创建模板 emailTemplate.html
``` html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>邮件模板</title>
</head>
<body>
淘宝 11.11，打折、优惠、红包、抽奖不断，大礼包就等你。</br>
<a href="#" th:href="@{http://www.taobao.com/{name}(name=${name})}">开始抽奖</a>
</body>
</html>
```
3. 发送模板邮件
**注意：**发送模板邮件的业务类里，必须使用 Spring Boot 为模板提供自动配置时注册的模板引擎 Bean，若自己创建模板引擎对象会读不到模板文件。
``` java
/**
 * @name: SimpleOrderManager
 * @desc: 发送邮件
 **/
@Service
public class SendEmailImpl implements SendEmail {

    private static final Logger logger = LogManager.getLogger(SendEmailImpl.class);

    @Autowired
    private JavaMailSender mailSender;

    @Autowired
    private TemplateEngine templateEngine;

    @Value("${spring.mail.username}")
    private String from;   

    /**
     * 发送模板(Thymeleaf)邮件
     */
    @Override
    public void sendMailTemplate() {
        User user = new User().setName("Andy").setEmailAddress("xxxx@163.com");
        try {
            //获取Thymeleaf模板内容(邮件内容)
            Context context = new Context();
            //设置模板需要的参数
            context.setVariable("name", user.getName());
            //执行模板引擎
            String emailContent = templateEngine.process("emailTemplate", context);
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true);
            helper.setFrom(from);
            helper.setTo(user.getEmailAddress());
            helper.setSubject("淘宝 11.11 活动");
            helper.setText(emailContent,true);

            mailSender.send(message);
            logger.info("邮件发送成功！");
        } catch (MessagingException e) {
            e.printStackTrace();
        }
    }
}

```
[spring-boot-email 源码 -> GitHub](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-email)


## 邮件系统 ##
若是微服务架构，可将发送邮件抽出为一个独立的微服务来使用，邮件系统作为平台的基础服务来支持整个系统。
作为一个完整的邮件系统，需要完整的记录发送邮件的任务和执行情况，包括发送失败的处理等。

### 异步发送邮件 ###
通常主业务并不需要发送邮件及时响应执行结果，比如营销类、通知类、提醒类的业务可以允许延时或者失败。这时可采用异步的方式发送邮件，以加快主业务的执行速度。

在实际项目中，邮件发送可采用**消息中间件MQ**来实现异步发送，主业务将发送邮件任务入库，通过 MQ 通知邮件系统，邮件系统收到通知就从库中取出任务执行发送逻辑；或者创建一个邮件发送的消息队列，给对应消息队列按照规定参数发送消息，邮件系统监听此队列，当有消息时对消息进行解析，按消息所带参数处理邮件发送的逻辑。

### 发送失败处理 ###
1. 将发送邮件任务入库，执行任务后同时标记任务的处理结果。
2. 根据需要，可启用定时任务来扫描发送失败的邮件，对重发次数小于 3 次的邮件执行再次发送。
3. 重发邮件的时间间隔，建议以 2 的次方间隔时间，如:2、4、8、16....

### 独立的管理后台 ###
一个完善的邮件系统，可以设计一个独立的邮件管理后台，提供图形化界面方便公司运营，对发送邮件业务进行统计等。也可以设置一些代码钩子，统计用户点击固定链接次数，方便公司营销人员监控邮件营销转化率。

还可设置白名单、黑名单来做邮件接收人的过滤机制。

【参考】
[如何使用 Spring Boot 开发邮件系统？](https://mp.weixin.qq.com/s/_uCByid-J9MwPTbhzjDlJg)
