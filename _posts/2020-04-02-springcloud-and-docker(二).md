---
title: springcloud and docker(二)
tags: ReadingNotes

---

### 第七章、使用Hystrix实现服务的容错处理

#### 通用方式整合hystrix

添加依赖

```xml
<!--Hystrix-->
<!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-starter-netflix-hystrix -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    <version>2.2.2.RELEASE</version>
</dependency>
```

在启动类上加上`@EnableHystrix`

controller

```java
@GetMapping("/user/{id}")
@HystrixCommand(fallbackMethod = "findByIdFallback", commandProperties = {
    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",
                     value = "5000"),//超时时间，默认为1秒
}, threadPoolProperties = {
    @HystrixProperty(name = "coreSize", value = "1"),//设置核心线程池的大小，默认为10
}, ignoreExceptions = {//配置不想执行回退的异常类
    MyBusinessException.class, IllegalAccessException.class
})
public User findById(@PathVariable Long id){
    return this.restTemplate.getForObject("http://microservice-provider-user/" + id,
                                          User.class);
}
/**
* 如需获得导致fallback的原因，只需在fallback方法上添加Throwable参数即可
* @param id
* @param throwable
* @return 用户
*/
public User findByIdFallback(Long id, Throwable throwable) {
    LOGGER.error("进入回退方法，异常：", throwable);
    User user = new User();
    user.setId(-1L);
    user.setUsername("RestTemplate默认用户");
    return user;
}
```

[更多配置信息](https://github.com/Netflix/Hystrix/wiki/Configuration)

正常运行将得到

![2020-04-10-14-15](\assets\images\springcloud-and-docker\2020-04-10-14-15.png)

关闭`microservice-provider-user`

![2020-04-10-14-17](\assets\images\springcloud-and-docker\2020-04-10-14-17.png)

**当请求失败、被拒绝、超时或者断路器打开时，都会进入回退方法。但是进入回退方法并不意味着断路器已经被打开。**

通过为项目引入[Actuator](https://nszm-jph.github.io/2020/03/25/springcloud-and-docker(%E4%B8%80).html#%E4%B8%BA%E9%A1%B9%E7%9B%AE%E6%95%B4%E5%90%88spring-boot-actuator)，我们可以通过http://localhost:8010/actuator/health来查看断路器状态

此时，由于`microservice-provider-user`关闭，已经返回默认用户，但是断路器依然关闭

![2020-04-10-14-23](\assets\images\springcloud-and-docker\2020-04-10-14-23.png)

疯狂刷新http://localhost:8010/user/1（默认5s内失败20次，将开启断路器）

![2020-04-10-14-31](\assets\images\springcloud-and-docker\2020-04-10-14-31.png)

#### Hystrix的隔离策略：

- THREAD（线程隔离）：使用该方法，HystrixCommand将在单独的线程上执行，并发请求受到线程池中的线程数量的限制
- SEMAPHORE（信号量隔离）：使用该方法，HystrixCommand将在调用线程上执行，开销相对较小，并发请求受到信号量个数的限制

默认使用THREAD，正常情况下保持默认即可，如果发生找不到上下文的运行时异常，可以考虑将隔离策略设置为SEMAPHORE

#### Feign使用Hystrix

springcloud默认已经为Feign整合了Hystrix

```yml
feign:
  hystrix:
    enabled: true
```

```java
/**
 * 使用@FeignClient的fallback指定回退类
 */
@FeignClient(name = "microservice-provider-user",
        configuration = FeignConfiguration.class,
        fallback = FeignClientFallback.class)
public interface UserFeignClient {

    @RequestMapping(value = "/{id}", method = RequestMethod.GET)
    public User findById(@PathVariable("id") Long id);
```

```java
/**
 * 回退类FeignClientFallback需要实现Feign Client接口
 * FeignClientFallback也可以是public class，没有区别
 */
@Component
public class FeignClientFallback implements UserFeignClient {
    @Override
    public User findById(Long id) {
        User user = new User();
        user.setId(-1L);
        user.setUsername("Feign默认用户");
        return user;
    }
}
```

##### 通过FallbackFactory检查回退原因

```java
@FeignClient(name = "microservice-provider-user",
        configuration = FeignConfiguration.class,
        fallbackFactory = FeignClientFallbackFactory.class)
public interface UserFeignClient {

    @RequestMapping(value = "/{id}", method = RequestMethod.GET)
    public User findById(@PathVariable("id") Long id);
}
```

```java
/**
 * UserFeignClient的FallbackFactory类
 */
@Component
public class FeignClientFallbackFactory implements FallbackFactory<UserFeignClient> {
    private static final Logger LOGGER = LoggerFactory.getLogger(
            FeignClientFallbackFactory.class);

    @Override
    public UserFeignClient create(Throwable throwable) {
        return new UserFeignClient() {
            @Override
            public User findById(Long id) {
                //日志最好放在各个fallback的方法里，而不要直接放在create方法中
                //否则启动时就会打印该日志
                FeignClientFallbackFactory.LOGGER.info("fallback; reason was:", throwable);
                User user = new User();
                user.setId(-1L);
                user.setUsername("FeignFallbackFactory默认用户");
                return user;
            }
        }
    }
}
```

##### 为Feign禁用Hystrix

```java
public class FeignDisableHystrixConfiguration {
    @Bean
    @Scope("prototype")
    public Feign.Builder feignBuilder() {
        return Feign.builder();
    }
}
```

想要禁用Hystrix的@FeignClient，引用该配置即可

#### Hystrix的监控

引入[Actuator](https://nszm-jph.github.io/2020/03/25/springcloud-and-docker(%E4%B8%80).html#%E4%B8%BA%E9%A1%B9%E7%9B%AE%E6%95%B4%E5%90%88spring-boot-actuator)，访问http://localhost:8010/actuator/hystrix.stream，将看到

```html
data: {"type":"HystrixCommand","name":"UserFeignClient#findById(Long)","group":"microservice-provider-user","currentTime":1586501651458,"isCircuitBreakerOpen":false,"errorPercentage":100,"errorCount":1,"requestCount":1,"rollingCountBadRequests":0,"rollingCountCollapsedRequests":0,"rollingCountEmit":0,"rollingCountExceptionsThrown":0,"rollingCountFailure":0,"rollingCountFallbackEmit":0,"rollingCountFallbackFailure":0,"rollingCountFallbackMissing":0,"rollingCountFallbackRejection":0,"rollingCountFallbackSuccess":1,"rollingCountResponsesFromCache":0,"rollingCountSemaphoreRejected":0,"rollingCountShortCircuited":0,"rollingCountSuccess":0,"rollingCountThreadPoolRejected":0,"rollingCountTimeout":1,"currentConcurrentExecutionCount":0,"rollingMaxConcurrentExecutionCount":0,"latencyExecute_mean":0,"latencyExecute"...}
```

##### feign项目监控

需要引入

```xml
<!--Hystrix-->
<!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-starter-netflix-hystrix -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    <version>2.2.2.RELEASE</version>
</dependency>
```

并在启动类上添加`@EnableCircuitBreaker`

##### 使用HystrixDashboard可视化监控数据

新建一个项目，artifactId为`microservice-hystrix-dashboard`，并添加依赖

```java
<!--hystrix-dashboard-->
<!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-starter-netflix-hystrix-dashboard -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
    <version>2.2.2.RELEASE</version>
</dependency>
```

在启动类上添加`@EnableHystrixDashboard`

```yml
server:
  port: 8030

eureka:
  client:
    healthcheck:
      enabled: true
    service-url:
      defaultZone: http://user:123456@peer1:8761/eureka/,http://user:123456@peer2:8762/eureka/
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 20

spring:
  application:
    name: microservice-hystrix-dashboard
```

启动，访问http://localhost:8030/hystrix

![2020-04-10-15-03](\assets\images\springcloud-and-docker\2020-04-10-15-03.png)

在url一栏输入http://localhost:8010/actuator/hystrix.stream，点击Monitor Stream

![2020-04-10-15-05](\assets\images\springcloud-and-docker\2020-04-10-15-05.png)

![2020-04-11-11-52](\assets\images\springcloud-and-docker\2020-04-11-11-52.png)

##### 使用Turbine聚合监控数据

为`microservice-hystrix-dashboard`项目添加依赖

```xml
<!--turbine-->
<!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-starter-netflix-turbine -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-turbine</artifactId>
    <version>2.2.2.RELEASE</version>
</dependency>
<!--actuator-->
<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-actuator -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
    <version>2.2.5.RELEASE</version>
</dependency>
```

启动类添加`@EnableTurbine`注解

```yml
turbine:
  cluster-name-expression: "'default'"
  app-config: microservice-consumer-movice, microservice-consumer-movier-feign-hystrix
```

启动，访问http://localhost:8030/hystrix，在url一栏输入http://localhost:8030/turbine.stream，点击Monitor Stream

![2020-04-11-12-37](\assets\images\springcloud-and-docker\2020-04-11-12-37.png)

### 第八章、使用Zuul构建微服务网关

#### 编写Zuul微服务网关

新建项目，添加以下依赖

```xml
<!--zuul-->
<!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-starter-netflix-zuul -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
    <version>2.2.2.RELEASE</version>
</dependency>
```

启动类添加`@EnableZuulProxy`

配置文件

```yml
server:
  port: 8040
spring:
  application:
    name: microservice-gateway-zuul
eureka:
  client:
    service-url:
      defaultZone: http://user:123456@peer1:8761/eureka/,http://user:123456@peer2:8762/eureka/
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 20
```

启动服务，访问http://localhost:8040/microservice-consumer-movice/user/1请求会被转发到http://localhost:8010/user/1，默认情况下，Zuul会代理所有注册到Eureka Server的微服务；

Zuul整合了Ribbon，启动多个`microservice-provider-user`，访问http://localhost:8040/microservice-provider-user/1，`microservice-provider-user`都会打印sql；

Zuul整合了Hystrix，启动`microservice-hystrix-dashboard`，访问http://localhost:8030/hystrix，输入http://localhost:8040/actuator/hystrix.stream

![2020-04-13-11-22](\assets\images\springcloud-and-docker\2020-04-13-11-22.png)

**注：需暴露actuator/hystrix.stream端点才能访问**

```yml
management:
  endpoints:
    web:
      exposure:
        include: hystrix.stream
```

**由于Thread Pools栏没有东西，可以看出默认情况下，Zuul的hystrix隔离策略是SEMAPHORE，可以配置为THREAD**

```yml
zuul:
  ribbon-isolation-strategy: thread
```

再次访问http://localhost:8030/hystrix

![2020-04-13-16-46](\assets\images\springcloud-and-docker\2020-04-13-16-46.png)

可以看出现在隔离策略变为了THREAD，但HystrixThreadPoolKey默认为RibbonCommand，这意味着所有路由的HystrixCommand都会在相同的Hystrix线程池中执行，可以使用以下配置，让每个路由使用独立的线程池

```yml
zuul:
  thread-pool:
    use-separate-thread-pools: true
```

对于本例，则是`microservice-consumer-movice`，还可以加上前缀

```yml
zuul:
  thread-pool:
    use-separate-thread-pools: true
    thread-pool-key-prefix: prefix-
```

对于本例，则是`prefix-microservice-consumer-movice`

#### 管理端点

/routes端点

- 使用GET方法访问该端点，即可返回Zuul当前映射的路由列表
- 使用POST方法访问就会强制刷新Zuul当前映射到路由列表
- 访问/routes/details可以查看更多与路由相关的详情设置

/filter端点

访问该端点即可返回Zuul中当前所有过滤器的详情

首先我们应该暴露这两个端点

```yml
management:
  endpoints:
    web:
      exposure:
        include: hystrix.stream, routes, filters
```

启动服务，访问http://localhost:8040/actuator/routes

![2020-04-13-11-33](\assets\images\springcloud-and-docker\2020-04-13-11-33.png)

访问http://localhost:8040/actuator/routes/details

![2020-04-13-11-34](\assets\images\springcloud-and-docker\2020-04-13-11-34.png)

访问http://localhost:8040/actuator/filters

![2020-04-13-11-35](\assets\images\springcloud-and-docker\2020-04-13-11-35.png)

#### 路由配置详解

1. 自定义指定微服务的访问路径

   ```yml
   zuul:
     routes:
       microservice-consumer-movice: /user/**
   ```

   `microservice-consumer-movice`被映射到`/user/**`路径，访问http://localhost:8040/user/user/1，请求会被转发到http://localhost:8010/user/1

2. 忽略指定微服务

   ```yml
   zuul:
     ignored-services: microservice-consumer-movice
   ```

   这样Zuul就不会再代理`microservice-consumer-movice`服务

3. 忽略所有微服务，只路由指定微服务

   ```yml
   zuul:
     ignored-services: '*'
     routes:
       microservice-consumer-movice: /user/**
   ```

4. 同时指定微服务的serverId和对应路径

   ```yml
   zuul:
     routes:
       user-route:				#该配置中，user-route只是给路由的一个名字，可以随便取
         service-id: microservice-consumer-movice
         path: /movice/**		
   ```

   效果同示例1

5. 同时指定path和URL

   ```yml
   zuul:
     routes:
       user-route:				#该配置中，user-route只是给路由的一个名字，可以随便取
         url: http://localhost:8000/
         path: /movice/**		
   ```

   这样就可以将/movice/**映射到http://localhost:8000/，需要注意的是，使用这种方式配置路由不会使用Ribbon以及Hystrix

6. 同时指定path和URL，并且不破坏Zuul的Ribbon、Hystrix特性

   ```yml
   zuul:
     routes:
       user-route:				#该配置中，user-route只是给路由的一个名字，可以随便取
         service-id: microservice-consumer-movice
         path: /movice/**	
   ribbon:
     eureka:
       enable: false 			#为Ribbon禁用Eureka
   microservice-consumer-movice:
     ribbon:
       listOfServers: localhost:8000, localhost:8001
   ```

7. 使用正则表达式指定Zuul的路由匹配规则

   ```java
   @Bean
       public PatternServiceRouteMapper serviceRouteMapper() {
           //servicePattern指定微服务正则
           //routePattern指定路由正则
           return new PatternServiceRouteMapper("(?<name>^.+)-(?<version>v.+$)",
                   "${version}/${name}");
       }
   ```

   将microservice-consumer-movice-v1请求转发到/v1/microservice-consumer-movice/**

8. 路由前缀

   ```yml
   zuul:
     #  此处默认有strip-prefix: true，所以/api被丢弃    
     prefix: /api
     routes:
       microservice-consumer-movice:
         path: /user/**
         #放在哪里就管理哪里，/user被保留
         strip-prefix: false
   ```

   strip-prefix默认为true，将抛弃代理前缀，因此访问http://localhost:8040/api/user/1，请求会被转发到http://localhost:8010/user/1

9. 忽略某些路径

   ```yml
   zuul:
     ignored-patterns: /**/admin/**	#忽略所有包含/admin/的路径
   ```

10. 本地转发

    ```yml
    zuul:
      routes:
        user-route:	
          url: forward:/path-b
          path: /path-a/**	
    ```

    访问Zuul的`/path-a/**`路径，将转发到Zuul的`/path-b/**`

打印Zuul转发的具体细节

```yml
logging:
  level:
    com.netflix: DEBUG
```

#### Zuul的安全与Header

##### 忽略Header

```yml
zuul:
  ignored-headers: Header1, Header2
```

这样设置后，Header1, Header2将不会转播到其他微服务中；默认情况下，ignored-headers是空值，但如果Spring Security在项目的classpath中，ignored-headers默认值就是Pragma, Cache-Control, X-Frame-Options, X-Content-Type-Options, X-XSS-Protection, Expires。所以，当Spring Security在项目的classpath中，同时又需要使用下游微服务的Spring Security的Header时，可以

```yml
zuul:
  ignore-security-headers: false
```

##### 敏感Header的设置

```yml
zuul:
  routes:
    microservice-consumer-movice:
      sensitive-headers: Cookie
```

也可以设置全局敏感Header

```yml
zuul:
  sensitive-headers: Cookie, Set-Cookie, Authorization #默认是Cookie, Set-Cookie, Authorization
```

事实上，`sensitive-headers`会被添加到`ignored-headers`中

#### 使用Zuul上传文件

对于小文件（1M以内）无须任何处理，即可正常上传；对于大文件（10M以上）上传，需要为上传路径添加/zuul前缀。对于超大文件（500M以上）需要提升超时设置

```yml
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 60000

ribbon:
  ConnectTimeout: 3000
  ReadTimeout: 60000
  # 同一实例最大重试次数，不包括首次调用
  MaxAutoRetries: 1
  # 重试其他实例的最大重试次数，不包括首次所选的server
  MaxAutoRetriesNextServer: 0
```

**注：此时运行服务将会出现以下WARN**

```
The Hystrix timeout of 60000ms for the command microservice-consumer-movice is set lower than the combination of the Ribbon read and connect timeout, 126000ms.
```

可以看出是因为Hystrix的超时时间小于Ribbon的超时时间，hystrix熔断了以后，ribbon的重试就都没有意义了；

ribbon超时的计算公式：ribbonTimeout = (ribbonReadTimeout + ribbonConnectTimeout) * (maxAutoRetries + 1) * (maxAutoRetriesNextServer + 1);

因此我们需要适当降低ribbon的超时设置

```yml
ribbon:
  ConnectTimeout: 15000
  ReadTimeout: 15000
  # 同一实例最大重试次数，不包括首次调用
  MaxAutoRetries: 1
  # 重试其他实例的最大重试次数，不包括首次所选的server
  MaxAutoRetriesNextServer: 0
```

创建一个新工程，artifactId是`microservice-file-upload`

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<!--actuator-->
<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-actuator -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
    <version>2.2.5.RELEASE</version>
</dependency>
```

```yml
server:
  port: 8050

spring:
  application:
    name: microservice-file-upload
  servlet:
    multipart:	#配置文件上传大小的限制
      max-file-size: 2000MB #默认1M
      max-request-size: 2500MB #默认10M

eureka:
  client:
    healthcheck:
      enabled: true
    service-url:
      defaultZone: http://user:123456@peer1:8761/eureka/,http://user:123456@peer2:8762/eureka/
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 20
```

```java
@Controller
public class FileUploadController {

    @RequestMapping(value = "/upload", method = RequestMethod.POST)
    public @ResponseBody String handleFileUpload(@RequestParam(value = "file",
        required = true)MultipartFile file) throws IOException {
        //当文件过大时，会发生内存溢出
        byte[] bytes = file.getBytes();
        File fileToSave = new File("G:\\"+file.getOriginalFilename());
        FileCopyUtils.copy(bytes,fileToSave);
        return fileToSave.getAbsolutePath();
    }
}
```

**此时直接上传大文件时会发生`java.lang.OutOfMemoryError: Java heap space`异常**，可以改为如下

```java
@Controller
public class FileUploadController {

    @RequestMapping(value = "/upload", method = RequestMethod.POST)
    public @ResponseBody String handleFileUpload(@RequestParam(value = "file",
        required = true)MultipartFile file) throws IOException {
//        byte[] bytes = file.getBytes();
        File fileToSave = new File("G:\\"+file.getOriginalFilename());
//        FileCopyUtils.copy(bytes,fileToSave);
        FileCopyUtils.copy(file.getInputStream(), new FileOutputStream(fileToSave));
        return fileToSave.getAbsolutePath();
    }
}
```

这样解决了大文件上传的问题，但是因为文件比较大，copy的耗时可能会比较长，用户体验度有待改善

启动服务，直接上传

```
curl -F "file=@G:\hacker\OWASP_Broken_Web_Apps_VM_1.2.7z" localhost:8050/upload
```

此时就可以上传成功，小文件也可以上传成功

通过zuul上传小文件

```
curl -v -H "Transfer-Encoding: chunked" -F "file=@F:\Pictures\Camera Roll\tnsnames.ora" localhost:8040/api/microservice-file-upload/upload
```

通过zuul上传大文件，不添加/zuul前缀

```
>curl -v -H "Transfer-Encoding: chunked" -F "file=@F:\Pictures\Camera Roll\-2ad026e748ce65ab.jpg" localhost:8040/api/microservice-file-upload/upload
```

将会报错`org.apache.tomcat.util.http.fileupload.impl.FileSizeLimitExceededException: The field file exceeds its maximum permitted size of 1048576 bytes.`

通过zuul上传大文件，添加/zuul前缀

```
curl -v -H "Transfer-Encoding: chunked" -F "file=@F:\Pictures\Camera Roll\tnsnames.ora" localhost:8040/zuul/api/microservice-file-upload/upload
```

文件上传成功！

#### Zuul过滤器

##### 过滤器类型与请求生命周期

- PRE：这种过滤器在请求被路由之前调用。可用于身份验证、在集群中选择请求的微服务、记录调试信息等
- ROUTING：这种过滤器将请求路由到微服务。这种过滤器用于构建发送给微服务的请求
- POST：这种过滤器在路由到微服务以后执行。可用来为响应添加标准的HTTP Header、收集统计信息和指标、将响应从微服务发送给客户端等
- ERROR：在其他阶段发生错误时执行该过滤器

##### 编写Zuul过滤器

```java
@Component
public class PreRequestLogFilter extends ZuulFilter {
    private static final Logger LOGGER = LoggerFactory.getLogger(PreRequestLogFilter.class);

    //定义过滤器的类型
    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }
    
    //定义过滤器的执行顺序
    @Override
    public int filterOrder() {
        return FilterConstants.PRE_DECORATION_FILTER_ORDER - 1;
    }
    
	//判断该过滤器是否执行
    @Override
    public boolean shouldFilter() {
        return true;
    }

    //过滤器的具体逻辑
    @Override
    public Object run() throws ZuulException {
        RequestContext currentContext = RequestContext.getCurrentContext();
        HttpServletRequest request = currentContext.getRequest();
        PreRequestLogFilter.LOGGER.info(String.format("send %s request to %s",
                request.getMethod(), request.getRequestURL().toString()));
        return null;
    }
}
```

启动服务，访问http://localhost:8040/api/user/1，将得到如下日志

```
send GET request to http://localhost:8040/api/user/1
```

##### 禁用Zuul过滤器

只需设置`zuul.<SimpleClassName>.<filterType>.disable=true`即可

```yml
zuul:
  PreRequestLogFilter:
    pre:
      disable: true
```

##### Zuul的容错与回退

```java
@Component
public class MyFallbackProvider implements FallbackProvider {

    private static final Logger LOGGER = LoggerFactory.getLogger(MyFallbackProvider.class);

    @Override
    public String getRoute() {
        //表明这是为哪个微服务提供回退，×表示为所有微服务提供回退
        return "*";
    }

    @Override
    public ClientHttpResponse fallbackResponse(String route, Throwable cause) {
        MyFallbackProvider.LOGGER.error(String.format("throwable is %s", cause));
        return new ClientHttpResponse() {
            //设置回退时的状态码
            @Override
            public HttpStatus getStatusCode() throws IOException {
                return HttpStatus.BAD_REQUEST;
            }

            //设置数字类型状态码
            @Override
            public int getRawStatusCode() throws IOException {
                return HttpStatus.BAD_REQUEST.value();
            }

            //设置状态文本
            @Override
            public String getStatusText() throws IOException {
                return HttpStatus.BAD_REQUEST.getReasonPhrase();
            }

            @Override
            public void close() {

            }

            //响应体
            @Override
            public InputStream getBody() throws IOException {
                return new ByteArrayInputStream("服务不可用，请稍后尝试".getBytes());
            }

            //返回的响应头
            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders httpHeaders = new HttpHeaders();
                MediaType mediaType = new MediaType("application", "json", Charset.forName("UTF-8"));
                httpHeaders.setContentType(mediaType);
                return httpHeaders;
            }
        };
    }
}
```

启动服务，关闭`microservice-consumer-movice`，访问http://localhost:8040/api/user/1

![2020-04-13-16-39](\assets\images\springcloud-and-docker\2020-04-13-16-39.png)

##### 饥饿加载

```yml
zuul:
  ribbon:
    eager-load:
      enabled: true
```

##### Query String编码

当处理请求时，query param会被编码，这些参数在route过滤器中构建请求时，将被重新编码。如果query param使用JavaScript的encodeURIComponent()方法进行编码，那么重新编码的结果可能与原始值不同；

要强制让query string与HttpServletRequest.getQueryString()保持一致，可使用以下配置

```yml
zuul:
  force-original-query-string-encoding: true
```

但由于query string在原始的HttpServletRequest上获取，将无法使用RequestContext.getCurrentContext().set-RequestQueryParams(someOverriddenParameters)重写query param