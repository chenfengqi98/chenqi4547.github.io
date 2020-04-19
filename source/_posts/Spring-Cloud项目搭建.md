---
title: Spring Cloud项目搭建
date: 2020-04-12 19:43:58
tags:
 - Spring Cloud
categories:
 - Spring Cloud
---

### 1 注册中心搭建

1. 新建Spring Boot项目，选择`eureka`依赖

   ```xml
   				<properties>
               <java.version>1.8</java.version>
               <spring-cloud.version>Greenwich.SR5</spring-cloud.version>
       		</properties>
   				<dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-actuator</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
           </dependency>
   
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-test</artifactId>
               <scope>test</scope>
           </dependency>
   ```

2. 配置文件

   ```yml
   # 端口
   server:
     port: 8080
   # eureka配置
   eureka:
     instance:
       hostname: localhost
     client:
       register-with-eureka: true
       fetch-registry: true
       service-url:
         defaultZone: http://localhost:8080/eureka/
   spring:
     application:
       name: eureka-server
   ```

3. 在启动类上加上`@EnableEurekaServer`注解

   EurekaServerApplication.java

   ```java
   @SpringBootApplication
   @EnableEurekaServer
   public class EurekaServerApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(EurekaServerApplication.class, args);
       }
   
   }
   
   ```

4. 测试，访问8080端口

   {% asset_img result-1.png %}

---

### 2 客户端搭建

1. 新建SpringBoot项目，引入`eureka、Spring Web`依赖

   ```xml
   				<dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-web</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-test</artifactId>
               <scope>test</scope>
           </dependency>
   ```

2. 配置文件

   ```yml
   # 端口
   server:
     port: 8081
   # 应用名
   spring:
     application:
       name: eureka-client
   # eureka配置
   eureka:
     client:
       service-url:
         defaultZone: http://localhost:8080/eureka/
       fetch-registry: true
   ```

3. 在启动类上加上`@EnableEurekaClient`注解，并新建一个Controller。

   EurekaClientApplication.java

   ```java
   @SpringBootApplication
   @EnableEurekaClient
   public class EurekaClientApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(EurekaClientApplication.class, args);
       }
   
   }
   ```

   UserController.java

   ```java
   @RestController
   public class UserController {
   
       @Value("${server.port}")
       private String port;
   
       @RequestMapping("index")
       public String index() {
           return port + " : Hello Spring Cloud";
       }
   }
   ```

4. 设置`EurekaClientApplication`可以同时启动多个实例，然后分别在8081端口和8082端口启动。

   {% asset_img result-2.png %}

5. 测试，查看注册中心的服务列表

   {% asset_img result-3.png %}

---

### 3 服务消费者搭建

1. 新建Spring Boot项目，引入`eureka、ribbon、Spring Web`依赖

   ```xml
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
               <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
           </dependency>
   ```

2. 配置文件

   ```yml
   server:
     port: 8090
   spring:
     application:
       name: eureka-consumer
   eureka:
     client:
       service-url:
         defaultZone: http://localhost:8080/eureka/
   ```

3. 在启动类上加上`@EnableEurekaClient`注解，并新建一个Controller

   EurekaCustomerApplication.java

   ```java
   @SpringBootApplication
   @EnableEurekaClient
   public class EurekaCustomerApplication {
   
   		// LoadBalanced ribbon负载均衡
       @Bean
       @LoadBalanced
       public RestTemplate restTemplate() {
           return new RestTemplate();
       }
   
       public static void main(String[] args) {
           SpringApplication.run(EurekaCustomerApplication.class, args);
       }
   
   }
   ```

   CustomerController.java

   ```java
   @RestController
   public class CustomerController {
   
       @Autowired
       RestTemplate restTemplate;
   
       @RequestMapping("index")
       public String index(){
           return restTemplate.getForObject("http://eureka-client/index", String.class);
       }
   }
   ```

4. 测试服务和负载均衡，在第一次访问的时候日志打印出了两个现有的服务提供者信息，并且会均衡的请求这两个服务

   {% asset_img result-4.png %}

   {% asset_img result-5.png %}

   {% asset_img result-6.png %}

---

#### 4 实现自定义负载均衡

1. 修改配置文件，新增`eureka-client.ribbon.listOfServers`配置

   ```yml
   server:
     port: 8090
   spring:
     application:
       name: eureka-consumer
   eureka:
     client:
       service-url:
         defaultZone: http://localhost:8080/eureka/
   # id:ribbon:listOfServers
   eureka-client:
     ribbon:
       listOfServers: localhost:8081,localhost:8082
   ```

2. 修改启动类和Controller代码

   EurekaCustomerApplication.java

   ```java
   @SpringBootApplication
   @EnableEurekaClient
   @RibbonClients(
           @RibbonClient(value = "eureka-client")
   )
   public class EurekaCustomerApplication {
   
       @Bean
       //@LoadBalanced
       public RestTemplate restTemplate() {
           return new RestTemplate();
       }
   
       public static void main(String[] args) {
           SpringApplication.run(EurekaCustomerApplication.class, args);
       }
   		// 自定义负载均衡策略
       @Bean
       public IRule myRule(){
           return new RandomRule();
       }
   }
   ```

   `@RibbonClient(value = "eureka-client")`中的value字段为配置文件中的id。

   CustomerController.java

   ```java
   @RestController
   public class CustomerController {
   
       @Autowired
       RestTemplate restTemplate;
   
       @Autowired
       LoadBalancerClient loadBalancerClient;
   
       @RequestMapping("index")
       public String index() {
           ServiceInstance choose = loadBalancerClient.choose("eureka-client");
           String url = "http://" + choose.getHost() + ":" + choose.getPort() + "/index";
           return restTemplate.getForObject(url, String.class);
       }
   }
   ```

   通过`loadBalancerClient`获取服务的ip和端口。

3. 测试后发现每次调用的服务是随机的

