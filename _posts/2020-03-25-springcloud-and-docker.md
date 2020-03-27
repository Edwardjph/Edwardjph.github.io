---
title: springcloud and docker
tags: ReadingNotes
---

### 第三章、开始使用spring cloud实战微服务

#### spring data jpa

```xml
<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-data-jpa -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
    <version>2.2.5.RELEASE</version>
</dependency>
```

配置

```yml
spring:
  jpa:
    database: mysql
    show-sql: true
    hibernate:
      ddl-auto: none
 	#可选参数
	#create 启动时删数据库中的表，然后创建，退出时不删除数据表
	#create-drop 启动时删数据库中的表，然后创建，退出时删除数据表 如果表不存在报错
	#update 如果启动时表格式不一致则更新表，原有数据保留
	#validate 项目启动表结构进行校验 如果不一致则报错
    #none 啥都不干
```

实体类

```java
@Entity //标识这个类是实体类
@Data
@Table(name = "User")//对应数据库内的表，表名与类名相同可以省略
public class User {
    @Id//主键
    @GeneratedValue(strategy = GenerationType.AUTO)//自增策略
    private Long id;

    @Column//变量对应列名
    private String username;
    
    @Column(name = "compete_status")
    private String competeStatus;
}
```

dao层

```java
@Repository//将类标识为Bean
@Transactional//事务
public interface UserRepository extends JpaRepository<User, Long> {
    @Modifying//通知Spring Data 这是一个DELETE或UPDATE操作。
    @Query("update Competitions c set c.filePath = ?1, c.status = ?2 where c.url = ?3")
    int updateByUrl(String filePath, String status, String url);
}
```

@Service用于标注业务层组件

@Controller用于标注控制层组件

@Repository用于标注数据访问组件，即DAO组件

@Component泛指组件，当组件不好归类的时候，我们可以使用这个注解进行标注。

#### lombok

先安装插件

![2020-03-26-12-02](\assets\images\springcloud-and-docker\2020-03-26-12-02.jpg)

```xml
<dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
</dependency>
```

```java
@Entity
@Data//@ToString+@EqualsAndHashCode+@Getter / @Setter+@RequiredArgsConstructor
@Table(name = "User")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
}
```

@ToString：自动生成tostring方法。可以这样设置不包含哪些字段：`@ToString(exclude = {"id","name"})`，如果继承的有父类的话，可以设置让其调用父类的toString()方法：`@ToString(callSuper = true)`

@Getter / @Setter：自动生成get、set方法，默认生成的方法是public的，可以通过`@Getter(access = AccessLevel.PROTECTED)`修改

@EqualsAndHashCode：生成hashCode()和equals()方法，可以这样设置不包含哪些字段：`@EqualsAndHashCode(exclude={"id", "name"})`

@RequiredArgsConstructor：生成构造方法，可以带参数也可以不带参数，如果带参数，这参数只能是以final修饰的未经初始化的字段，或者是以@NonNull注解的未经初始化的字段

@NoArgsConstructor：生成一个无参构造方法

@AllArgsConstructor：生成一个全参数的构造方法

#### log4j2

```xml
	<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
      <exclusions>
        <exclusion><!-- 去除默认配置 -->
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-log4j2 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-log4j2</artifactId>
        <version>2.2.5.RELEASE</version>
    </dependency>
    <dependency>  <!-- 支持识别yml配置 -->
        <groupId>com.fasterxml.jackson.dataformat</groupId>
        <artifactId>jackson-dataformat-yaml</artifactId>
    </dependency>
```

编写log4j2配置，文件放在resources中

```yml
#共有8个级别，按照从低到高为：ALL < TRACE < DEBUG < INFO < WARN < ERROR < FATAL < OFF
Appenders:
    Console:  #输出到控制台
      name: CONSOLE #Appender命名
      target: SYSTEM_OUT
      PatternLayout:
        pattern: "%d{yyyy-MM-dd HH:mm:ss.SSS} -%5p ${PID:-} [%15.15t] %-30.30C{1.} : %m%n"#日志输出格式
    RollingFile: # 输出到文件，超过256MB归档
      - name: ROLLING_FILE
        ignoreExceptions: false#是否忽略异常
        fileName: /springboot/logs/springboot.log#文件路径
        filePattern: "/springboot/logs/$${date:yyyy-MM}/springboot -%d{yyyy-MM-dd}-%i.log.gz"#文件命名格式
        PatternLayout:
          pattern: "%d{yyyy-MM-dd HH:mm:ss.SSS} -%5p ${PID:-} [%15.15t] %-30.30C{1.} : %m%n"
        Policies:
       		TimeBasedTriggeringPolicy: #指定了基于时间的归档触发策略
       			interval: 1
          	SizeBasedTriggeringPolicy: #指定了基于文件大小的归档触发策略
            	size: "256 MB"
       		DefaultRolloverStrategy: #允许归档的日志文件的最大数量，将对过旧的日志文件进行删除操作
          		max: 1000
Loggers:
    Root:
      level: info
      AppenderRef:
        - ref: CONSOLE	#相当于调用上面的配置
    Logger: #单独设置某些包的输出级别
      - name: app.com.kenho.mapper 
        additivity: false #是否去除重复的log
        level: trace
        AppenderRef:
          - ref: CONSOLE 
          - ref: ROLLING_FILE
```

#### 为项目整合spring boot actuator

监控程序，了解应用程序的运行状态

```xml
<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-actuator -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
    <version>2.2.5.RELEASE</version>
</dependency>
```

访问http://localhost:8010/actuator查看暴露的端口，默认只暴露了health与info

默认情况下，除`shutdown`之外的所有端点均处于启用状态，但需自己暴露

[详细端口信息及操作](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html#production-ready-enabling)

### 第四章、微服务注册与发现

```xml
<modules>
    <module>microservice-provider-user</module>
    <module>microservice-consumer-movice</module>
</modules>
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>        
```

#### eureka高可用

在一台电脑上演示，需修改hosts文件，Windows路径：`C:\Windows\System32\drivers\etc\hosts`

```
127.0.0.1 peer1 peer2
```

application.yml

```yml
spring:
  application:
    name: microservice-discovery-eureka

eureka:
  client:
    service-url:
      defaultZone: http://peer1:8761/eureka/,http://peer2:8762/eureka/


---
spring:
#指定profile=peer1
  profiles: peer1
server:
  port: 8761
eureka:
  instance:
  #指定当profile=peer1时，主机名是peer1
    hostname: peer1

---
spring:
  profiles: peer2
server:
  port: 8762
eureka:
  instance:
    hostname: peer2
```

使用连字符（---）将application文件分为三段，第二第三段指定了properties的值，表示它所在的那段内容应用在哪个profile里。第一段没有指定，对所有profile生效。

分别启动

```
java -jar microservice-discovery-eureka-0.0.1-SNAPSHOT.jar --spring.profiles.active=peer1
java -jar microservice-discovery-eureka-0.0.1-SNAPSHOT.jar --spring.profiles.active=peer2
```

访问http://localhost:8761

![2020-03-27-12-38](\assets\images\springcloud-and-docker\2020-03-27-12-38.png)

将应用注册到eureka集群

```yml
eureka:
  client:
    service-url:
      defaultZone: http://peer1:8761/eureka/,http://peer2:8762/eureka/
```

#### 用户认证

添加security依赖

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
      password: 123456
eureka:
  client:
    service-url:
      defaultZone: http://user:123456@peer1:8761/eureka/,http://user:123456@peer2:8762/eureka/
```

此时运行会报错，因为eureka缺少CSRF（跨站点请求伪造）令牌，需要为`/eureka/**`端点禁用此要求[更多详情](https://cloud.spring.io/spring-cloud-static/spring-cloud-netflix/2.1.2.RELEASE/single/spring-cloud-netflix.html#_securing_the_eureka_server)

```java
@EnableWebSecurity
class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().ignoringAntMatchers("/eureka/**");
        super.configure(http);
    }
}
```

将微服务注册到需要认证的eureka

```yml
eureka:
  client:
    service-url:
      defaultZone: http://user:123456@peer1:8761/eureka/,http://user:123456@peer2:8762/eureka/
```

#### eureka自我保护

有时会看到eureka进入保护模式，如下

![2020-03-27-13-09](\assets\images\springcloud-and-docker\2020-03-27-13-09.png)

因为上一分钟收到的心跳数（Renews）小于期望收到的心跳数（Renews threshold）。但这种情况可能是因为网络故障导致的，微服务其实本身是健康的，eureka就会进入自我保护模式，不再删除服务注册表中的数据，宁可同时保留所有微服务也不盲目注销任何健康的微服务。可手动禁用自我保护模式

```
eureka.server.enable-self-preservation = false
```

