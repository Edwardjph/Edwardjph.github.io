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