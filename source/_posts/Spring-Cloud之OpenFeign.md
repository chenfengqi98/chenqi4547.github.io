---
title: Spring Cloud之OpenFeign
date: 2020-04-19 19:51:26
tags:
 - Spring Cloud
categories:
 - Spring Cloud
---

### Feign简介

Feign是一种声明式、模板化的HTTP客户端。在Spring Cloud中使用Feign, 我们可以做到使用HTTP请求远程服务时能与调用本地方法一样的编码体验，开发者完全感知不到这是远程方法，更感知不到这是个HTTP请求。它基于 Netflix Feign 实现，整合了 Spring Cloud Ribbon 与 Spring Cloud Hystrix,  除了整合这两者的强大功能之外，它还提供了声明式的服务调用(不再通过RestTemplate调用方式)。

---

### Feign使用

> Spring Cloud搭建可以查看上篇文章：<a href="/2020/04/12/Spring-Cloud项目搭建/">Spring Cloud项目搭建</a>

1. 新建Spring Boot项目，引入下面的依赖

   ```xml
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-actuator</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-web</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-openfeign</artifactId>
           </dependency>
   ```

2. 配置文件

   ```yml
   server:
     port: 8092
   spring:
     application:
       name: eureka-feign
   eureka:
     client:
       service-url:
         defaultZone: http://localhost:8080/eureka/
   # 启用Hystrix
   feign:
     hystrix:
       enabled: true
   hystrix:
     command:
       default:
         execution:
           isolation:
             thread:
               timeoutInMilliseconds: 100
   ```

---

### 具体代码实现

1. 启动类上加上`@EnableFeignClients`注解

   ```java
   @SpringBootApplication
   @EnableFeignClients
   public class EurekaFeignApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(EurekaFeignApplication.class, args);
       }
   
   }
   ```

2. 新建一个`FeignService`接口，并指定调用`eureka-client`服务

   ```java
   @Component
   @FeignClient(name = "eureka-client", fallback = FeignServiceFallback.class)
   public interface FeignService {
   
       @RequestMapping(value = "/index")
       String index();
   
       @RequestMapping(value = "/getName")
       String getName(@RequestParam("name") String name);
   
   }
   ```

3. 新建`FeignServiceFallback`实现上面的接口，用于实现Hystrix熔断的回调。

   ```java
   @Component
   public class FeignServiceFallback implements FeignService {
   
       private final Logger logger = LoggerFactory.getLogger(FeignServiceFallback.class);
   
       @Override
       public String index() {
           String reslut = "服务降级";
           logger.warn(reslut);
           return reslut;
       }
   
       @Override
       public String getName(String name) {
           String reslut = "服务降级..";
           logger.warn(reslut);
           return reslut;
       }
   }
   ```

4. 新建`FeignController`

   ```java
   @RestController
   public class FeignController {
   
       @Autowired
       FeignService feignService;
   
       @RequestMapping("/index")
       public String index() {
           return feignService.index();
       }
   
       @RequestMapping("/getName")
       public String getName(@RequestParam("name") String name) {
           return feignService.getName(name);
       }
   
   }
   ```

5. 修改eureka-client

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
   
       @GetMapping("getName")
       public String getName(@RequestParam("name") String name) {
           return port + " : " + name;
       }
   }
   ```

   

---

### 测试

1. 调用测试

   {% asset_img result-1.png %}

   {% asset_img result-2.png %}

   {% asset_img result-3.png %}

