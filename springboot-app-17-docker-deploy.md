---
title: Spring Boot 2实践系列(十七)：Docker部署Spring Boot应用
comments: true
date: 2018-06-20 16:52:02
tags: [docker,jar,deploy]
categories: [Spring Boot 2实践系列]
---
　　现在的互联网应用系统在需要分布式部署时，会遇到环境配置的问题，在容器技术出来之前，需要在每台服务器系统上配置应用系统的运行环境，容易出错或环境不一致导致各种问题。

　　容器技术的出现很好的解决了环境配置问题，配置一次应用运行环境，打成镜像，到处部署使用，当然还有其它的用途。
<!-- more -->
## 环境准备 ##
### 安装 Docker ###
Docker 的安装不在此详述，可参考[Docker系列(一)：Centos 7下安装Docker](http://112.74.59.39/2018/04/06/docker-1-centos7-install/)，其它操作系统的安装参考 Docker 官方文档：[CentOS](https://docs.docker.com/install/linux/docker-ce/centos/)，[Ubuntu](https://docs.docker.com/install/linux/docker-ce/ubuntu/)，[Debian](https://docs.docker.com/install/linux/docker-ce/debian/)，[Fedora](https://docs.docker.com/install/linux/docker-ce/fedora/)

国内可参考：[Docker 入门教程](http://www.ruanyifeng.com/blog/2018/02/docker-tutorial.html)，只是有个别命令的使用不是官方推荐的方式。

### 安装 Maven ###
Maven 安装参考[Maven安装和使用(Linux, Windows)](http://112.74.59.39/2018/06/20/maven-install-linux-windows/)

### 安装 Git ###
Git 安装参考[Getting Started - Installing Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

## 部署 jar 包设置 ##
1. 有一个包含 jdk1.8 的基础镜像。
2. Spring Boot项目添加 docker 插件。
	``` xml	
	<artifactId>myapp</artifactId>
	<packaging>jar</packaging>
	<properties>
	    <docker.image.prefix>local_repos</docker.image.prefix>
	</properties>
	
	<!-- Docker maven plugin -->
	<plugin>
	    <groupId>com.spotify</groupId>
	    <artifactId>docker-maven-plugin</artifactId>
	    <version>1.1.1</version>
	    <configuration>
	        <imageName>${docker.image.prefix}/${project.artifactId}</imageName>
	        <dockerDirectory>src/main/docker</dockerDirectory>
	        <resources>
	            <resource>
	                <targetPath>/</targetPath>
	                <directory>${project.build.directory}</directory>
	                <include>${project.build.finalName}.${project.packaging}</include>
	            </resource>
	        </resources>
	    </configuration>
	</plugin>
	```
	**说明：** ${docker.image.prefix}:镜像前辍，一般用于标识镜像库名称。
3. 在项目`src/main`创建`docker`目录，在**docker**目录里创建 `Dockerfile`文件。
4. 编辑 Dockerfile 文件。更多参考[Docker系列(四)：Dockerfile 配置详解](http://112.74.59.39/2018/06/20/docker-4-dockerfile/)
``` 
#指定基础镜像
FROM repos_local/centos-jdk1.8:1.0

#作者
MAINTAINER gxing

#数据卷
VOLUME /tmp

#添加当前目录文件到镜像的工作目录
ADD myapp.jar myapp.jar

#运行容器后执行
ENTRYPOINT ["java","-jar","myapp.jar","--spring.profiles.active=prod","&"]
```

## 部署 war 包设置##
部署`war`包的基础镜像必须包含`JDK`和`Tomcat`。
1. 修改`pom.xml`文件。
``` xml
<artifactId>myapp</artifactId>
<packaging>war</packaging>
<properties>
    <docker.image.prefix>local_repos</docker.image.prefix>
</properties>

<!-- 打war包, 覆盖掉内嵌的Tomcat -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
    <scope>provided</scope>
</dependency>

<!-- 打war包，排除web.xml,使用java config -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-war-plugin</artifactId>
    <version>2.6</version>
    <configuration>
        <failOnMissingWebXml>false</failOnMissingWebXml>
    </configuration>
</plugin>

<!-- Docker maven plugin -->
<plugin>
    <groupId>com.spotify</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>1.1.1</version>
    <configuration>
        <imageName>${docker.image.prefix}/${project.artifactId}</imageName>
        <dockerDirectory>src/main/docker</dockerDirectory>
        <resources>
            <resource>
                <targetPath>/</targetPath>
                <directory>${project.build.directory}</directory>
                <include>${project.build.finalName}.${project.packaging}</include>
            </resource>
        </resources>
    </configuration>
</plugin>
```
2. 添加 Dockerfile 文件
``` xml
FROM centos-jdk1.8-tomcat8:1.0

MAINTAINER gxing

VOLUME /tmp

#---------jar-----------------
#ADD myapp.war myapp.jar

#---------war-----------------
ADD myapp.war /usr/local/tomcat/webapps/ROOT.war
WORKDIR /usr/local/tomcat/bin
CMD ["catalina.sh", "run"]
```
## 打Docker应用镜像 ##
1. 将修改后的项目推到 Git仓库。
2. 在 Linux 使用 Git 命令拉取项目工程代码。
> git pull https://xxxxx/xxx/xxx.git

3. 进入到项目工程根目录，执行 Maven 打包，出现**BUILD SUCCESS**表示成功。
> mvn package -DskipTests docker:build

4. 查看镜像,会有显打包成功的镜像。
> docker images

5. 创建容器并启动。
> docker run -d -p 8080:8080 --name app local_repos/myapp

6. 访问容器：http://xxxxxxx:8080/xxx 。
访问成功表示正常。

7. 映射项目日志到外部(-v：共享文件)
> docker run -d -p 8080:8080 -v /data/qyd-server/logs:/usr/local/tomcat/bin/logs --name myapp local_repos/myapp
> 基础镜像里配置了镜像里的工作目录是`/work`，在打Spring Boot 应用镜像时，指定了添加应用到根目录(工作目录)，项目使用 Log4j2 配置了日志输出到文件，运行容器时添加`-v`参来将容器中的应用日志共享到本地主要的目录中。