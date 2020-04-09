---
title: springcloud and docker(一)
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

eureka server

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

eureka client

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
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

自我保护模式的激活条件是：上一分钟，`Renews(last min)<Renews threshold`，默认等待五分钟（可通过`eureka.server.wait-time-in-ms-when-sync-empty`配置）后可看到如上信息

`Renews threshold`:期望收到的心跳数，等于服务数乘以每分钟心跳数（默认每分钟两次，可通过`eureke.instance.lease-renewal-interval-in-seconds`配置）乘以续约比例阀值（默认0.85，可通过`eureka.server.renewal-percent-threshold`配置），eg：一共有五个服务，`Renews threshold`=5×2×0.85=8

`Renews(last min)`:上一分钟收到的心跳数，等于服务数乘以每分钟心跳数（默认每分钟两次，可通过`eureke.instance.lease-renewal-interval-in-seconds`配置），eg：一共五个服务，`Renews(last min)`=5×2=10

自我保护模式被激活后，它不会从注册列表中剔除因长时间没收到心跳导致租期过期的服务，而是等待修复，直到心跳恢复正常之后，它自动退出自我保护模式。自我保护模式可有效避免网络分区故障导致服务不可用问题

但在单机环境下，很容易触发自我保护模式，导致以关闭的服务无法及时剔除，使用ribbon时访问到已关闭的服务，解决办法：

1、禁用自我保护模式  (eureka server)

```
eureka.server.enable-self-preservation = false
```

2、降低阀值，原理：降低`Renews threshold`   (eureka server)

```
eureka.server.renewal-percent-threshold = 0.5
```

3、加快eureka client发送心跳频率，原理：增大`Renews(last min)`  (eureka client)

```
eureke.instance.lease-renewal-interval-in-seconds = 10
```

同时由于eureka server清理无效节点的时间间隔默认为60秒，依然容易出现以上情况，可以适当减小  (eureka server)

```
eureka.server.eviction-interval-timer-in-ms = 30000
```

也可设置eureka server至上一次收到client的心跳之后，等待下一次心跳的超时时间，在这个时间内若没收到下一次心跳，则将移除该instance

```
eureke.instance.lease-expiration-duration-in-seconds = 6
```

#### 多网卡环境下的ip选择

1. 忽略指定名称的网卡

   ```yml
   spring:
     cloud:
       inetutils:
         ignored-interfaces:
           - docker0
           - veth.*
   ```

   这样就忽略了docker0以及veth开头的网卡

2. 使用正则表达式，指定使用的网络地址

   ```yml
   spring:
     cloud:
       inetutils:
           preferred-networks:
               - 192.168
               - 10.0
   ```

3. 只使用站点本地地址

   ```yml
   spring:
     cloud:
       inetutils:
       	use-only-site-local-interfaces: true
   ```

4. 手动指定ip

   ```yml
   eureka:
   	instance:
   		ip-address: 127.0.0.1
   ```

以上都需开启

```yml
eureka:
    instance:
        prefer-ip-address: true
```

#### eureka的健康检查（需整合actuator）

应用程序将主动向eureka发送健康状况（来自/actuator/health）

```yml
eureka:
  client:
    healthcheck:
      enabled: true
```

### 第五章、使用Ribbon实现客户端侧负载均衡

#### 为服务消费者整合Ribbon

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
```

由于在`spring-cloud-starter-netflix-eureka-client`依赖中已包含`spring-cloud-starter-netflix-ribbon`，因此无须再次引入

为RestTemplate添加@LoadBalanced注解

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate(){
    return new RestTemplate();
}
```

Controller

```java
@RestController
public class MoviceController {
    private static final Logger LOGGER = LoggerFactory.getLogger(MoviceController.class);

    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    private LoadBalancerClient loadBalancerClient;

    @GetMapping("/user/{id}")
    public User findById(@PathVariable Long id){
        return this.restTemplate.getForObject("http://microservice-provider-user/" + id,
                User.class);
    }

    @GetMapping("/log-user/instance")
    public void logUserInstance(){
        ServiceInstance serviceInstance = this.loadBalancerClient.choose(
                "microservice-provider-user");
        //打印当前选择的是哪个节点
        MoviceController.LOGGER.info("{}:{}:{}", serviceInstance.getServiceId(),
                serviceInstance.getHost(), serviceInstance.getPort());
    }
}
```

当Ribbon与eureka配合使用的时候，会自动将虚拟主机名映射成微服务的网络地址，所以可以直接使用`http://microservice-provider-user/`虚拟主机名不能包含`_`

**补充：**

`@PathVariable`：当使用someUrl/{paramId}样式映射时，这时的paramId可通过 @Pathvariable注解绑定它传过来的值到方法的参数上。

`@RequestHeader`：可以把Request请求header部分的值绑定到方法的参数上

```java
public void displayHeaderInfo(@RequestHeader("Accept-Encoding") String encoding,
                              @RequestHeader("Keep-Alive") long keepAlive)  {
}
```

`@CookieValue`：可以把Request header中关于cookie的值绑定到方法的参数上

```java
public void displayHeaderInfo(@CookieValue("JSESSIONID") String cookie)  {
}
```

`@RequestParam`：用来处理Content-Type: 为 `application/x-www-form-urlencoded`编码的内容，提交方式GET、POST

```java
public String setupForm(@RequestParam("petId") int petId, ModelMap model) {
}
```

`@RequestBody`：用来处理Content-Type: 不是`application/x-www-form-urlencoded`编码的内容，例如application/json, application/xml等

```java
public void handle(@RequestBody String body, Writer writer) throws IOException {
}
```

#### Ribbon配置自定义

1、用Java代码自定义Ribbon配置

创建Ribbon的配置类

```java
/**
 * 该类为Ribbon的配置类
 * 注意：该类不应该在主应用程序上下文的@ComponentScan所扫描的包中
 */
@Configuration
@ExcludeFromComponentScan
public class RibbonConfiguration {
    @Bean
    public IRule ribbonRule(){
        //负载均衡规则，改为随机
        return new RandomRule();
    }
}
```

启动类

```java
@SpringBootApplication
@RibbonClient(name = "microservice-provider-user", configuration =
        RibbonConfiguration.class)
public class MicroserviceSimpleConsumerMoviceApplication {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }

    public static void main(String[] args) {
        SpringApplication.run(MicroserviceSimpleConsumerMoviceApplication.class, args);
    }

}
```

必须注意的是，本例中的RibbonConfiguration类不能存放在主应用程序上下文的@componentScan所扫描的保中，否则该类中的配置信息将被所有的@RibbonClient共享

2、使用属性自定义Ribbon配置

```yml
microservice-provider-user:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
```

#### 脱离Eureka使用Ribbon

在没有eureka的服务上，引入

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
```

```yml
microservice-provider-user:
  ribbon:
  	listOfServers: localhost:8000,localhost:8001
```

有eureka但想单独是用ribbon，不使用eureka的服务发现功能

```yml
ribbon:
  eureka:
    enabled: false
```

只让指定名称的ribbon client使用指定的url，其他依旧与eureka配合使用：

```
<clientName>.ribbon.NIWSServerListClassName = com.netflix.loadbalancer.ConfigurationBasedServerList
<clientName>.ribbon.listOfServers = localhost:8000,localhost:8001
```

#### 饥饿加载

指定名称的Ribbon Client第一次请求的时候，对应的上下文才会加载，因此，首次请求会比较慢，可以配置饥饿加载：

```yml
ribbon:
  eager-load:
    clients: microservice-provider-user, client2
    enabled: true
```

### 第六章、使用Feign实现声明式REST调用

#### 为服务消费者整合Feign

```xml
<!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-starter-openfeign -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
    <version>2.2.2.RELEASE</version>
</dependency>
```

Feign接口

```java
@FeignClient(name = "microservice-provider-user")
public interface UserFeignClient {
    @RequestMapping(value = "/{id}", method = RequestMethod.GET)
    public User findById(@PathVariable("id")Long id);
}

```

controller

```java
@RestController
public class MoviceController {
    @Autowired
    private UserFeignClient userFeignClient;

    @GetMapping("/user/{id}")
    public User findById(@PathVariable Long id){
        return this.userFeignClient.findById(id);
    }
}
```

启动类添加@EnableFeignClients注解

```java
@EnableFeignClients
@SpringBootApplication
public class MicroserviceSimpleConsumerMoviceApplication {
    public static void main(String[] args) {
        SpringApplication.run(MicroserviceSimpleConsumerMoviceApplication.class, args);
    }
}
```

#### 自定义Feign配置

##### 使用java代码配置Feign

配置类

```java
/**
 * 该类可以不写@Configuration注解，如果使用了@Configuration注解，那么
 * 该类不能放在主应用程序上下文@ComponentScan所扫描的包中
 */
public class FeignConfiguration {
    /**
     * 将契约改为feign原生动默认契约，这样就可以使用feign自带的注解了
     * @return 默认的feign契约
     */
    @Bean
    public Contract feignContract(){
        return new feign.Contract.Default();
    }

    @Bean
    Logger.Level feignLoggerLevel(){
        return Logger.Level.FULL;
    }
}
```

Feign接口

```java
@FeignClient(name = "microservice-provider-user", configuration = FeignConfiguration.class)
public interface UserFeignClient {
    /**
     * 使用feign自带的注解@RequestLine
     * @param id 用户id
     * @return 用户信息
     */
    @RequestLine("GET /{id}")
    public User findById(@Param("id") Long id);
}

```

##### 使用属性配置

```yml
#全局配置
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: full
#指定FeignCliend配置
feign:
  client:
    config:
      microservice-provider-user:
        loggerLevel: basic
```

#### Feign对压缩的支持

```yml
feign:
  compression:
    request:
      enabled: true
    response:
      enabled: true
```

#### Feign日志

```yml
feign:
  client:
    config:
      default:
        loggerLevel: full
logging:
  level:
  	#将Feign接口的日志级别设置成DEBUG，因为Feign的Logger.Level只对DEBUG做出响应
    com.itmuch.cloud.microservicesimpleconsumermovice.feign.UserFeignClient: DEBUG
```

loggerLevel的值有：

- NONE：不记录任何日志（默认值）
- BASIC：仅记录请求方法、URL、响应状态代码以及执行时间
- HEADERS：记录BASIC级别的基础上，记录请求和响应的header
- FULL：记录请求和响应的header、body和元数据

#### 使用Feign构造多参数请求

##### GET请求多参数

```java
	/**
     * 两种get请求多参数方法
     *
     */
    @RequestMapping(value = "/get1", method = RequestMethod.GET)
    public User get1(@RequestParam("id") Long id, @RequestParam("username") String username);

    @RequestMapping(value = "/get2", method = RequestMethod.GET)
    public User get2(@RequestParam Map<String, Object> map);
```

##### POST请求多参数

```java
	/**
     * post请求多参数方法
     */
    @RequestMapping(value = "/post", method = RequestMethod.POST)
    public User post(@RequestBody User user);
```

#### 使用Feign上传文件

添加依赖

```xml
<!-- https://mvnrepository.com/artifact/io.github.openfeign.form/feign-form -->
        <dependency>
            <groupId>io.github.openfeign.form</groupId>
            <artifactId>feign-form</artifactId>
            <version>3.8.0</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/io.github.openfeign.form/feign-form-spring -->
        <dependency>
            <groupId>io.github.openfeign.form</groupId>
            <artifactId>feign-form-spring</artifactId>
            <version>3.8.0</version>
        </dependency>
```

Feign Client

```java
@FeignClient(name = "microservice-file-upload", configuration =
        UploadFeignClient.MultiparSupportConfig.class)
public interface UploadFeignClient {
    @RequestMapping(value = "/upload", method = RequestMethod.POST,
                produces = {MediaType.APPLICATION_JSON_VALUE},
                consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    @ResponseBody
    String handleFileUpload(@RequestPart(value = "file")MultipartFile file);

    class MultiparSupportConfig {
        @Bean
        public Encoder feignFormEncoder() {
            return new SpringFormEncoder();
        }
    }
}

```

