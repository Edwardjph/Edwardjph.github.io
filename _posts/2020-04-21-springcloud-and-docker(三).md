---
title: springcloud and docker(三)
tags: ReadingNotes
---

### 第九章、使用spring cloud config统一管理微服务配置

#### 新建一个git仓库（[码云](https://gitee.com/)）

拉取到本地

```
git clone https://gitee.com/mynszm/spring-cloud-config-repo.git
```

新建几个配置文件

`microservice-foo.properties`

`microservice-foo-dev.properties`

`microservice-foo-test.properties`

`microservice-foo-production.properties`

内容分别是

`profile=default-1.0`

`profile=dev-1.0`

`profile=test-1.0`

`profile=production-1.0`

提交到远程仓库

```
git status
git add *
git commit -m "info"
git push origin master
```

为了测试版本控制，新建`config-label-v2.0`分支

```
#创建分支
git branch config-label-v2.0
#切换分支
git checkout config-label-v2.0
```

将各个配置文件中的1.0改为2.0，再次上传

#### 编写Config Server

新建一个工程，ArtifactId是`microservice-config-server`

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

启动类添加`@EnableConfigServer`注解

```yml
server:
  port: 8080

spring:
  application:
    name: microservice-config-server
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/mynszm/spring-cloud-config-repo.git
          username: 1194824079@qq.com
          password: nszm6796
```

启动项目

端口与配置文件的映射规则，label对应仓库分支，默认是master

```
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties
```

访问http://localhost:8080/microservice-foo/dev

![2020-04-24-11-04](\assets\images\springcloud-and-docker\2020-04-24-11-04.png)

访问http://localhost:8080/microservice-foo-dev.properties

```
profile: dev-1.0
```

访问http://localhost:8080/config-label-v2.0/microservice-foo-dev.properties

```
profile: dev-2.0
```

#### 编写config client

新建项目，ArtifactId是`microservice-config-client`

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

application.yml

```yml
server:
  port: 8081
```

新建配置文件bootstrap.yml

```yml
spring:
  application:
    #对应config server所获取的配置文件的{application}
    name: microservice-foo
  cloud:
    config:
      uri: http://localhost:8080/
      #profile对应config server所获取的配置文件中的{profile}
      profile: dev
      #指定Git仓库的分支，对应config server所获取的配置文件的{label}
      label: master
```

bootstrap.yml在application.yml之前加载，用于应用程序上下文的引导阶段。

从运行效果来看，bootstrap.yml在程序运行起来之前就已经加载，可以理解为，在程序运行之前拉取配置，再运行程序

```java
@RestController
public class ConfigClientController {

    @Value("${profile}")
    private String profile;

    @GetMapping("/profile")
    public String hello() {
        return this.profile;
    }
}
```

运行程序，访问http://localhost:8081/profile

```
dev-1.0
```

#### Config Server的Git仓库配置详解

##### 1、占位符支持

```yml
spring:
  application:
    name: microservice-config-server
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/mynszm/{application}
```

使用这种方式，即可实现一个应用对应一个Git仓库

##### 2、模式匹配

```yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/mynszm/spring-cloud-config-repo.git
          repos:
          	simple: https://gitee.com/simple/spring-cloud-config-repo.git
          	special: 
          	  pattern: special*/dev*,*special*/dev*
          	  uri: https://gitee.com/special/spring-cloud-config-repo.git
          	local:
          	  pattern: local*
          	  uri: file:/home/configsvc/config-repo
```

对于simple仓库，只匹配所有配置文件中名为simple的应用程序。local仓库则匹配所有配置文件中以local开头的所有应用程序的名称。如果不匹配任何模式，则使用spring.cloud.config.server.git.uri

##### 3、搜索模式

```yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/mynszm/spring-cloud-config-repo.git
          search-paths: foo,bar*
```

这样就会在Git仓库根目录、foo子目录以及所有以bar开头的子目录中查找配置文件

##### 4、启动时加载配置文件

默认情况下，在配置首次请求时，config server才会clone git仓库。

```
spring.cloud.config.server.git.clone-on-start=true
```

此配置即可让config server启动时就clone git仓库

#### Config Server的健康状况指示器

```yml
spring:
  application:
    name: microservice-config-server
  security:
    user:
      name: user
      password: admin
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/mynszm/spring-cloud-config-repo.git
          username: 1194824079@qq.com
          password: nszm6796
          clone-on-start: true
        health:
          repositories:
            a-foo:
              #默认为master
              label: config-label-v2.0
              name: microservice-foo
              #默认是default
              profiles: dev
              
management:
  endpoint:
    health:
      show-details: always
```

访问http://localhost:8080/actuator/health

![2020-04-24-11-47](\assets\images\springcloud-and-docker\2020-04-24-11-47.png)

#### 配置内容的加解密

##### 安装JCE

[下载地址](https://www.oracle.com/technetwork/cn/java/javase/downloads/jce8-download-2133166-zhs.html)

下载解压，将解压出来的两个jar文件替换JDK/jre/lib/security目录中的jar文件

##### 对称加密

修改Config Server项目，新建bootstrap.yml文件

```yml
encrypt:
  key: foo	#对称加密密钥
```

**以encrypt开头的配置，必须放在bootstrap.yml文件中**

运行程序

**此时不要配置security，否则curl没有任何反应**

```
curl http://localhost:8080/encrypt -d mysecret
```

返回`37c031735585beda5a210f912e146d2dba9e6eca588a7d80cff5210e7428f4dc`， 加密成功

```
curl http://localhost:8080/decrypt -d 37c031735585beda5a210f912e146d2dba9e6eca588a7d80cff5210e7428f4dc
```

返回mysecret，解密成功

##### 存储加密文件

准备一个配置文件encryption.yml

```yml
spring:
  datasource:
    username: dbuser
    password: '{cipher}37c031735585beda5a210f912e146d2dba9e6eca588a7d80cff5210e7428f4dc'
```

如果使用的是properties格式，则不能使用单引号

```
spring.datasource.username=dbuser
spring.datasource.password={cipher}37c031735585beda5a210f912e146d2dba9e6eca588a7d80cff5210e7428f4dc
```

将其push到仓库

访问http://localhost:8080/encryption-default.yml

```
spring:
  datasource:
    username: dbuser
    password: mysecret
```

Config Server自动解密配置内容，如果想返回密文本身

```
spring.cloud.config.server.encrypt.enabled=false
```

##### 非对称加密

执行一下命令，创建一个Key Store

```
keytool -genkeypair -alias mytestkey -keyalg RSA -dname "CN=WebServer,OU=Unit,O=Organization,L=City,S=State,C=US" -keypass changeme -keystore server.jks -storepass letmein
```

将生成的server.jks文件复制到resource下

```yml
encrypt:
  key-store:
    location: classpath:/server.jks #jks文件路径
    password: letmein               #storepass
    alias: mytestkey                #alias
    secret: changeme                #keypass
```

运行程序

```
curl http://localhost:8080/encrypt -d mysecret
```

返回`AQBXie4FL0UZLgg+XdIC03DfVPouKSrKJMiceRCBujZs0xK0kHYiVk9F0cQ/1l+Jqk7gC4jCCZwzJ9n8KKSfhyXlYsBlhZ6u9FVOgPvfLG23QMNkyYNDnJzwP4IW7e3AM6RLBpJWYEvAnoJK67w5YcvCNI8HRfZSXWEk5sOydIGND5163aEJ8W7nfeRjoY4n42kYoxvcJ6iXYpUpu6YhpBdHaVWFlvRtb8PyrAjeX3gOTrxAR0/ZeKLwQDXfdgnJhth+07WoG8Y3Ubb+8onOcCdfSAoc9cTjqfWlkaOpVjYVJZMO7fxpDLRGxSBXVbPu+05MG1dVufZv53pgXBOnOv3zqXETCOi/jWnr+oSW4qXIyUZXSLs0VGR1zX4FAAKSltM=`， 加密成功

```
curl http://localhost:8080/decrypt -d AQBXie4FL0UZLgg+XdIC03DfVPouKSrKJMiceRCBujZs0xK0kHYiVk9F0cQ/1l+Jqk7gC4jCCZwzJ9n8KKSfhyXlYsBlhZ6u9FVOgPvfLG23QMNkyYNDnJzwP4IW7e3AM6RLBpJWYEvAnoJK67w5YcvCNI8HRfZSXWEk5sOydIGND5163aEJ8W7nfeRjoY4n42kYoxvcJ6iXYpUpu6YhpBdHaVWFlvRtb8PyrAjeX3gOTrxAR0/ZeKLwQDXfdgnJhth+07WoG8Y3Ubb+8onOcCdfSAoc9cTjqfWlkaOpVjYVJZMO7fxpDLRGxSBXVbPu+05MG1dVufZv53pgXBOnOv3zqXETCOi/jWnr+oSW4qXIyUZXSLs0VGR1zX4FAAKSltM=
```

返回mysecret，解密成功

#### 使用/refresh端点手动刷新配置

修改`microservice-config-client`项目，将/refresh端口暴露出来

```yml
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

需要手动刷新的类加上`@RefreshScope`注解

```java
@RestController
@RefreshScope
public class ConfigClientController {

    @Value("${profile}")
    private String profile;

    @GetMapping("/profile")
    public String hello() {
        return this.profile;
    }
}
```

启动项目，访问http://localhost:8081/profile，获得结果dev-1.0；

修改`microservice-foo-dev.properties`文件内容：`profile=dev-1.0-change`

再次访问http://localhost:8081/profile，依然是dev-1.0；

```
curl -X POST http://localhost:8081/actuator/refresh
```

返回结果`["profile"]`，表示profile这个配置已经被刷新

再次访问http://localhost:8081/profile，获得dev-1.0-change；

#### 使用Spring Cloud Bus自动刷新配置

将`microservice-config-server`以及`microservice-config-client`项目都加入消息总线，两个项目分别添加

```xml
<!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-starter-bus-amqp -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
    <version>2.2.1.RELEASE</version>
</dependency>
```

在bootstrap.yml中

```yml
spring:
  rabbitmq:
    host: 192.168.80.5
    port: 5672
    username: root
    password: admin
```

启动`microservice-config-server`以及多个`microservice-config-client`

修改`microservice-foo-dev.properties`配置内容`profile=dev-1.0-bus`

发送post请求到Config Server

```
curl -X POST http://localhost:8080/actuator/bus-refresh
```

**注：发送到Config Client也可，但这样破坏了微服务的职责单一原则，以及微服务各节点的对等性**

此时访问各个微服务，发现配置都已经更新

配合Git、码云仓库的WebHooks

![2020-04-24-16-12](\assets\images\springcloud-and-docker\2020-04-24-16-12.png)

可实现配置的自动刷新

##### 局部刷新

可使用destination参数来定位要刷新的应用程序，例如：`/actuator/bus-refresh?destination=customers:9000`，其中`customers`是`spring.application.name`，

`9000`是`server.port`

##### 跟踪总线事件

```yml
spring:
  cloud:
    bus:
      trace:
        enabled: true
```

`/actuator/bus-refresh`端点被请求后通过访问`/actuator/trace`跟踪总线事件

#### spring cloud config与Eureka配合使用

将`microservice-config-server`以及`microservice-config-client`项目都注册到Eureka上

Config Client的bootstrap.yml配置

```yml
spring:
  application:
    name: microservice-foo
  cloud:
    config:
      profile: dev
      label: master
      discovery:
        enabled: true
        service-id: microservice-config-server

eureka:
  client:
    #导致注册中心UNKNOWN的罪魁祸首，该配置应放在application.yml中
#    healthcheck:
#      enabled: true
    service-url:
      defaultZone: http://user:123456@peer1:8761/eureka/,http://user:123456@peer2:8762/eureka/
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 20
```

**注：其中`microservice-config-client`关于eureka的配置必须放置在bootstrap.yml，个人理解为discovery.enable此时已经需要在eureka中找到service-id，所以必须现在注册；`microservice-config-server`无此限制**

**！！！关于健康检查，如果放在bootstrap.yml将导致注册中心此服务处于UNKNOWN状态**

#### Spring Cloud Config的用户认证

修改`microservice-config-server`项目

```xml
<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
    <version>2.2.5.RELEASE</version>
</dependency>
```

application.yml

```yml
spring:
  security:
    user:
      name: user
      password: admin
```

Config Client有两种方式使用需要用户认证的Config Server

```yml
spring:
  cloud:
    config:
      uri: http://user:admin@localhost:8080/
```

```yml
spring:
  cloud:
    config:
      username: user
      password: admin
```

### 第十章、使用Spring Cloud Sleuth实现微服务跟踪

修改`microservice-provider-user`以及`microservice-consumer-movice`项目

```xml
<!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-starter-sleuth -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
    <version>2.2.2.RELEASE</version>
</dependency>
```

这样就整合Sleuth成功

#### spring cloud sleuth与Zipkin配合使用

**关于 Zipkin 的服务端，在使用 Spring Boot 2.x 版本后，官方就不推荐自行定制编译了，直接提供了编译好的 jar 包**

下载，运行

```
curl -sSL https://zipkin.io/quickstart.sh | bash -s
java -jar zipkin.jar
```

访问http://localhost:9411/zipkin/将看到

![2020-04-24-16-46](\assets\images\springcloud-and-docker\2020-04-24-16-46.png)

修改`microservice-provider-user`以及`microservice-consumer-movice`项目

```xml
<!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-sleuth-zipkin -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-zipkin</artifactId>
    <version>2.2.2.RELEASE</version>
</dependency>
```

```yml
spring:
  zipkin:
    base-url: http://localhost:9411
  sleuth:
    sampler:
      percentage: 1.0
```

- `sleuth.sampler.percentage`：指定需采样的请求的百分比，默认是0.1，即10%，Sleuth会忽略大量的span，建议在开发，测试时，将此属性设大一点

启动后，访问一会`microservice-provider-user`以及`microservice-consumer-movice`

再访问http://localhost:9411/zipkin/

![2020-04-24-16-53](\assets\images\springcloud-and-docker\2020-04-24-16-53.png)

#### 使用消息中间件收集数据

我们可以通过环境变量让 Zipkin 从 RabbitMQ 中读取信息，就像这样：

```
java -jar zipkin.jar --zipkin.collector.rabbitmq.addresses=192.168.80.5 --zipkin.collector.rabbitmq.username=root --zipkin.collector.rabbitmq.password=admin
```

[zipkin.jar的yml配置文件内容可在此处查看](https://github.com/openzipkin/zipkin/blob/master/zipkin-server/src/main/resources/zipkin-server-shared.yml)

关于 Zipkin 的 Client 端，删除Sleuth及Zipkin相关依赖，添加

```xml
<!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-starter-zipkin -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
    <version>2.2.2.RELEASE</version>
</dependency>

<!-- https://mvnrepository.com/artifact/org.springframework.amqp/spring-rabbit -->
<dependency>
    <groupId>org.springframework.amqp</groupId>
    <artifactId>spring-rabbit</artifactId>
    <version>2.2.5.RELEASE</version>
</dependency>
```

```yml
spring:
  zipkin:
    sender:
      type: rabbit
  sleuth:
    sampler:
      percentage: 1.0
  rabbitmq:
    host: 192.168.80.5
    port: 5672
    username: root
    password: admin
```

