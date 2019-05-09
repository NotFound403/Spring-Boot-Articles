---
title: Spring Boot 2实践系列(二十四)：内嵌Tomcat SSL配置
comments: true
date: 2018-07-18 09:58:45
tags: [tomcat,ssl]
categories: [Spring Boot 2实践系列]
---
　　现在`https`的请求链接已非常普遍了，**https**可理解为 **http**的安全版，是基于 http 下加入了**SSL**(Secure Sockets Layer 安全套接层)层，为数据通讯提供安全支持。

　　SSL 配置是基于服务端的，Nginx 和 Tomcat 都可以配置 SSL，SSL证书可以自己通过JDK工具生成，也可以购买认证的证书。

　　SSL 协议可分为两层： SSL记录协议（SSL Record Protocol）：它建立在可靠的传输协议（如TCP）之上，为高层协议提供数据封装、压缩、加密等基本功能的支持。 SSL握手协议（SSL Handshake Protocol）：它建立在SSL记录协议之上，用于在实际的数据传输开始前，通讯双方进行身份认证、协商加密算法、交换加密密钥等(- -百度百科)。

　　Java Web 项目的部署环境通常在 Servlet容器(Tomcat)外面一层使用 Nginx 来做返向代理，在 Nginx 服务器里面配置 SSL 并将 HTTP 重定向到 HTTPS，或者在 Tomcat 容器里配置 SSL，将HTTP 请求重定向 到 HTTPS。

　　本篇针对 Spring Boot 内嵌 Tomcat 配置 SSL 进行讲解，实际使用可能并不多，但可以了解Spring Boot 内嵌 Tomcat 对 SSL 的支持和配置。
<!-- more -->
## 生成证书 ##
调用 jdk 自带的 keytool 工具可以生成自签名的证书。
```
C:\Users\Administrator>keytool -genkey -v -alias tomcat -keyalg RSA
输入密钥库口令:
再次输入新口令:
您的名字与姓氏是什么?
  [Unknown]:  Zheng
您的组织单位名称是什么?
  [Unknown]:  Administrator
您的组织名称是什么?
  [Unknown]:  Admin
您所在的城市或区域名称是什么?
  [Unknown]:  ShenZhen
您所在的省/市/自治区名称是什么?
  [Unknown]:  GuangDong
该单位的双字母国家/地区代码是什么?
  [Unknown]:  CN
CN=Zheng, OU=Rocky, O=Admin, L=ShenZhen, ST=GuangDong, C=CN是否正确?
  [否]:  y

输入 <tomcat> 的密钥口令
        (如果和密钥库口令相同, 按回车):
```
在当前目录下会生成一个`.keystore`证书文件。
**备注**：如果不指定证书算法(`-keyalg RSA`)，默认生成的SSL证书的版本会比较底，已被现在主流的浏览器弃用。

## 配置SSL ##
将`.keystore`拷贝到项目根目录，在`application.properties`中做如下配置。
```
server.port=8443
server.ssl.key-store=classpath:.keystore
server.ssl.key-password=123456
server.ssl.key-store-type=JKS
server.ssl.key-store-password=123456
server.ssl.key-alias=tomcat
```
重启项目，浏览器访问：https://localhost:8443/，浏览器会提示不安全，是因为没有生成给客户端用的证书，点击继承链接。

## http跳转https ##
Spring Boot 2.0.x版本在配置 HTTP 强制跳转到 HTTPS 时，重写 `TomcatServletWebServerFactory`类的`postProcessContext(Context context)`方法时一直存在问题报错，查了官网和搜索相关问题暂未找到解决思路, 网上文章绝大部分是基本 1.5.x版本写的，Spring Boot 2.0.x版本在创建内置的Tomcat容器时与 1.5.x的版本存在较大的差异。
``` java
@Configuration
public class SSLConfig {

    /*同时支持http和https访问*/
    @Bean
    public ServletWebServerFactory servletContainer() {
        TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory();
        tomcat.addAdditionalTomcatConnectors(createStandardConnector());
        return tomcat;
    }


    /*htt强制跳转到https: ----> 以下配置存在问题,暂未能解决*/
    /*@Bean
    public ServletWebServerFactory servletContainer() {
        TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory() {

            @Override
            protected void postProcessContext(Context context) {
                SecurityConstraint securityConstraint = new SecurityConstraint();
                securityConstraint.setUserConstraint("CONFIDENTIAL");

                SecurityCollection securityCollection = new SecurityCollection();
                securityCollection.addPattern("/*");

                securityConstraint.addCollection(securityCollection);
                context.addConstraint(securityConstraint);
            }

        };
        tomcat.addAdditionalTomcatConnectors(createStandardConnector());
        return tomcat;
    }*/

    /*创建连接器*/
    public Connector createStandardConnector() {
        Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
        connector.setScheme("http");
        connector.setPort(8080);
        connector.setSecure(false);
        //重定向
        connector.setRedirectPort(8443);
        return connector;
    }
}
```
在创建 TomcatServletWebServerFactory 的 Bean 时报的错：
> Exception encountered during context initialization - cancelling refresh attempt: org.springframework.beans.FatalBeanException: ServletWebServerFactory implementation com.springboot.web.configure.SSLConfig$1 cannot be instantiated. To allow a separate management port to be used, a top-level class or static inner class should be used instead

字面理解是`SSLConfig`无法实例化，如果不重写 postProcessContext(Context context) 则正常，而该方法在源码里是一个空的方法，源码上的注释是给子类重写的方法, 但在使用时不可能继承 TomcatServletWebServerFactory，否则会报多个 Servlet 容器错误，所以暂未找到解决方案...
若有人踩过同样的坑并解决了的，或有解决思路的，请赐教！

## Nginx SSL ##
个人建议，生产环境使用 Nginx 的返向代理, 在 Nginx 配置 SSL比较简单。

**参考：**
[JDK自带工具keytool生成ssl证书](https://www.cnblogs.com/green-hand/p/6514597.html)
[外部Tomcat配置SSL](https://www.cnblogs.com/luchangyou/p/5889161.html)