---
title: springcloud and docker(三)
tags: ReadingNotes
---

### 第九章、使用spring cloud config统一管理微服务配置

#### 

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

