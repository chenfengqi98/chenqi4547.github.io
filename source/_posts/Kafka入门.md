---
title: Kafka入门
date: 2020-04-19 21:30:07
tags:
- Kafka
categories:
- Kafka
---

### 使用Docker搭建Kafka环境

1. Zookeeper安装

   ```sh
   docker pull wurstmeister/zookeeper
   docker run -d --name zookeeper -p 2181:2181 -v wurstmeister/zookeeper
   ```

   {% asset_img result-1.png %}

2. Kafka安装

   ```sh
   docker pull wurstmeister/kafka
   docker run -d --name kafka -p 9092:9092 -e KAFKA_BROKER_ID=0 -e KAFKA_ZOOKEEPER_CONNECT={ip}:2181/kafka -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://{ip}:9092 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 wurstmeister/kafka
   ```

   {% asset_img result-2.png %}

   - KAFKA__BROKER_ID：kafka都有一个BROKER_ID来区分自己
   - KAFKA_ZOOKEEPER_CONNECT：连接Zookeeper地址，{ip}改为宿主机ip
   - KAFKA_ADVERTISED_LISTENERS：把kafka的地址端口注册给zookeeper，{ip}改为宿主机ip
   - KAFKA_LISTENERS：配置kafka的监听端口

3. 测试

   进入Kafka镜像，使用`kafka-console-producer.sh`命令向test主题发送消息

   ```sh
   docker exec -it kafka /bin/sh
   # 使用
   kafka-console-producer.sh --broker-list localhost:9092 --topic test
   ```

   使用`kafka-console-consumer.sh`命令订阅test主题

   ```sh
   kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test 
   ```

   {% asset_img result-3.png %}

   可以在Zookeeper中看到相关信息

   {% asset_img result-4.png %}

### Kafka在Spring Boot中使用

1. 新建Spring Boot项目，引入Kafka依赖

   ```pom
   		<dependencies>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-web</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.kafka</groupId>
               <artifactId>spring-kafka</artifactId>
           </dependency>
   
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-test</artifactId>
               <scope>test</scope>
               <exclusions>
                   <exclusion>
                       <groupId>org.junit.vintage</groupId>
                       <artifactId>junit-vintage-engine</artifactId>
                   </exclusion>
               </exclusions>
           </dependency>
           <dependency>
               <groupId>org.springframework.kafka</groupId>
               <artifactId>spring-kafka-test</artifactId>
               <scope>test</scope>
           </dependency>
           <dependency>
               <groupId>org.projectlombok</groupId>
               <artifactId>lombok</artifactId>
               <version>1.18.10</version>
           </dependency>
       </dependencies>
   ```

2. 配置文件

   ```properties
   server.port=9090
   
   spring.kafka.consumer.bootstrap-servers=localhost:9092
   # 配置消费者消息offset是否自动重置
   spring.kafka.consumer.auto-offset-reset=earliest
   
   spring.kafka.producer.bootstrap-servers=localhost:9092
   spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
   spring.kafka.producer.retries=3
   
   # 自定义属性
   kafka.topic.mytopic:my-topic
   kafka.topic.mytopic2:my-topic2
   ```

3. 代码实现

   Kafka配置类

   ```java
   @Configuration
   public class KafkaConfig {
   
       @Value("${kafka.topic.my-topic}")
       private String myTopic;
       
       @Value("${kafka.topic.my-topic2}")
       private String myTopic2;
   
       @Bean
       public RecordMessageConverter jsonConverter() {
           return new StringJsonMessageConverter();
       }
   
       @Bean
       public NewTopic myTopic() {
           return new NewTopic(myTopic, 2, (short) 1);
       }
   
       @Bean
       public NewTopic myTopic2() {
           return new NewTopic(myTopic2, 1, (short) 1);
       }
   }
   ```

   消息生产者

   ```java
   @Service
   public class UserProducerService {
   
       private static final Logger logger = LoggerFactory.getLogger(UserProducerService.class);
   
       private final KafkaTemplate<String, Object> kafkaTemplate;
   
       public UserProducerService(KafkaTemplate<String, Object> kafkaTemplate) {
           this.kafkaTemplate = kafkaTemplate;
       }
   
       public void sendMessage(String topic, Object o) {
           ListenableFuture<SendResult<String, Object>> future = kafkaTemplate.send(topic, o);
           future.addCallback(result ->
                   logger.info("producer send message to topic:{},partition:{} success", result.getRecordMetadata().topic(),
                           result.getRecordMetadata().partition()),
                   ex -> { logger.error("producer send message failed, exception:{}", ex.getMessage());
           });
       }
   }
   ```

   消费者，通过在方法上使用 `@KafkaListener` 注解监听消息，当有消息的时候就会拉取消费。

   ```java
   @Service
   public class UserConsumerService {
   
       private static final Logger logger = LoggerFactory.getLogger(UserConsumerService.class);
   
       private final ObjectMapper objectMapper = new ObjectMapper();
   
       @KafkaListener(topics = {"${kafka.topic.my-topic}"}, groupId = "group1")
       public void consumeMessage(ConsumerRecord<String, String> userConsumerRecord) {
           try {
               User user = objectMapper.readValue(userConsumerRecord.value(), User.class);
               logger.info("消费者消费topic:{} partition:{}的消息 -> {}", userConsumerRecord.topic(), userConsumerRecord.partition(), user.toString());
           } catch (JsonProcessingException e) {
               e.printStackTrace();
           }
       }
   
       @KafkaListener(topics = {"${kafka.topic.my-topic2}"}, groupId = "group2")
       public void consumeMessage2(ConsumerRecord<String, String> userConsumerRecord) throws JsonProcessingException {
           User user = objectMapper.readValue(userConsumerRecord.value(), User.class);
           logger.info("消费者消费{}的消息 -> {}", userConsumerRecord.topic(), user.toString());
       }
   }
   ```

   User实体

   ```java
   @Data
   @ToString
   public class User {
       private Long id;
       private String username;
   
       public User() {
       }
   
       public User(Long id, String username) {
           this.id = id;
           this.username = username;
       }
   }
   ```

   UserController

   ```java
   @RestController
   @RequestMapping("/user")
   public class UserController {
       @Value("${kafka.topic.my-topic}")
       private String myTopic;
       
       @Value("${kafka.topic.my-topic2}")
       private String myTopic2;
   
       @Autowired
       private UserProducerService producerService;
   
       @RequestMapping("/send")
       public void sendMessage(@RequestParam String name){
           producerService.sendMessage(myTopic, new User(1L,name));
           producerService.sendMessage(myTopic2, new User(2L,name));
       }
   }
   ```

4. 测试

   访问http://localhost:9090/user/send?name=werw

   {% asset_img result-5.png %}

   查看Zookeeper节点

   {% asset_img result-6.png %}