---
title: Spring Cloud之Hystrix使用
date: 2020-04-13 10:16:50
tags:
 - Spring Cloud
categories:
 - Spring Cloud
---

### Hystrix简介

​		在分布式环境中，不可避免地会有许多服务依赖项中的某些失败。Hystrix是一个库，可通过添加等待时间容限和容错逻辑来帮助您控制这些分布式服务之间的交互。Hystrix通过隔离服务之间的访问点，停止服务之间的级联故障并提供后备选项来实现此目的，所有这些都可以提高系统的整体弹性。

​		总体来说`[Hystrix]`就是一个能进行**熔断**和**降级**的库，通过使用它能提高整个系统的弹性。

Spring Cloud Hystrix实现了**断路器、线程隔离**等一系列服务保护功能。

- Fallback(失败快速返回)：当某个服务单元发生故障（类似用电器发生短路）之后，通过断路器的故障监控（类似熔断保险丝）， **向调用方返回一个错误响应， 而不是长时间的等待**。这样就不会使得线程因调用故障服务被长时间占用不释放，**避免**了故障在分布式系统中的**蔓延**。
- 资源/依赖隔离(线程池隔离)：它会为**每一个依赖服务创建一个独立的线程池**，这样就算某个依赖服务出现延迟过高的情况，也只是对该依赖服务的调用产生影响， 而**不会拖慢其他的依赖服务**。

---

### Hystrix服务搭建

> Spring Cloud搭建可以查看上篇文章：<a href="/2020/04/12/Spring-Cloud项目搭建/">Spring Cloud项目搭建</a>

1. 新建Spring Boot项目，引入如下依赖

   ```xml
   <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-web</artifactId>
   </dependency>
   <!-- actuator 监控-->
   <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-actuator</artifactId>
   </dependency>
   <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
   </dependency>
   <!-- hystrix -->
   <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
   </dependency>
   <!-- hystrix-dashboard 监控面板 -->
   <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
   </dependency>
   ```

2. 配置文件

   ```yml
   server:
     port: 8091
   spring:
     application:
       name: eureka-consumer-hystrix
   eureka:
     client:
       service-url:
         defaultZone: http://localhost:8080/eureka/
   # actuator暴露节点
   management:
     endpoints:
       web:
         exposure:
           include: "*"
   ```

---

### 具体代码实现

1. 启动类`EurekaCustomerHystrixApplication.java`

   ```
   @SpringBootApplication
   @EnableHystrixDashboard
   @EnableHystrix
   public class EurekaCustomerHystrixApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(EurekaCustomerHystrixApplication.class, args);
       }
   
       @Bean
       @LoadBalanced
       public RestTemplate restTemplate() {
           return new RestTemplate();
       }
   }
   ```

   需要在启动类上加上`@EnaableHystrix`注解，才会在actuator有`hystrix.stream`节点信息，`@EnableHystrixDashboard`用于启用监控面板

2. 实现一个`HystrixCommand`

   ```java
   public class MyCustomerCommand extends HystrixCommand {
   
       private final Logger logger = LoggerFactory.getLogger(MyCustomerCommand.class);
   
       private RestTemplate restTemplate;
   
       public MyCustomerCommand(RestTemplate restTemplate) {
           super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("hystrix"))
                   //选择Controller
                   .andCommandKey(HystrixCommandKey.Factory.asKey("HystrixController"))
                   // 设置100ms超时时间，5s熔断重试间隔
                   .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
                           .withExecutionTimeoutInMilliseconds(100)
                           .withCircuitBreakerSleepWindowInMilliseconds(5000))
                   //线程池参数
                   .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("hystrixThreadPool"))
                   .andThreadPoolPropertiesDefaults(HystrixThreadPoolProperties.Setter()
                           .withCoreSize(1).withMaxQueueSize(2)));
   
           this.restTemplate = restTemplate;
       }
   
       /**
        * 核心代码实现，具体逻辑
        *
        * @return
        * @throws Exception
        */
       @Override
       protected Object run() throws Exception {
           return restTemplate.getForObject("http://eureka-client/index", String.class);
       }
   
       /**
        * 熔断逻辑
        *
        * @return
        */
       @Override
       protected Object getFallback() {
           String reslut = "服务降级";
           logger.warn(reslut);
           return reslut;
       }
   }
   ```

3. 新建Controller

   ```java
   @RestController
   public class HystrixController {
       @Autowired
       RestTemplate restTemplate;
   
       @RequestMapping("index")
       public Object index() {
           return new MyCustomerCommand(restTemplate).execute();
       }
   }
   ```

4. eureka-client模拟网络延迟

   ```java
   @RestController
   public class UserController {
   
       private final Logger logger = LoggerFactory.getLogger(UserController.class);
   
       @Value("${server.port}")
       private String port;
   
       @RequestMapping("index")
       public String index() throws InterruptedException {
           Random random = new Random();
           // 模拟网络延迟
           int timeOut = random.nextInt(100) + 50;
           Thread.sleep(timeOut);
           logger.info("当前延迟时间：{} ms", timeOut);
           return port + " : Hello Spring Cloud";
       }
   }
   ```

---

### 测试

1. 服务降级测试

   {% asset_img result-1.png %}

   {% asset_img result-2.png %}

​	在控制台可以看到日志信息

```sh
2020-04-12 10:03:31.529  WARN 817 --- [ HystrixTimer-4] c.c.c.e.command.MyCustomerCommand        : 服务降级
```

2. HystrixDashboard测试

   {% asset_img result-3.png %}

获取当前服务的Hystrix监控节点，也就是Single Hystrix App：https://hystrix-app:port/actuator/hystrix.stream{% asset_img result-4.png %}

然后在HystrixDashboard输入节点信息，既可以监控Hystrix信息

{% asset_img result-5.png %}

