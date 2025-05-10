---
title: 使用 kafka 作为消息队列 做流量削峰
date: 2024-04-21
draft: "false"
tags:
  - kafka
---

## 1.优点

1. **高吞吐量**：Kafka 能够处理非常高的消息吞吐量，适用于需要处理大量数据的场景，比如日志收集、实时数据处理等。
2. **持久性**：Kafka 的消息是持久化存储的，可以保证消息不会丢失。即使消费者消费消息失败，消息也可以被重新消费。
3. **水平扩展性**：Kafka 的设计允许在集群中添加更多的节点来扩展容量和吞吐量，而不需要对现有系统进行太多改动。
4. **分布式存储**：Kafka 使用分布式的日志存储来存储消息，这意味着消息被分割成多个分区并分布在不同的节点上，提高了性能和容错性。
5. **多订阅者**：Kafka 支持多个消费者组，每个消费者组可以独立消费消息，这使得 Kafka 适合于广播消息给多个订阅者的场景。
6. **实时数据处理**：由于 Kafka 具有低延迟和高吞吐量的特点，它非常适合用于实时数据处理和流处理场景，比如事件驱动架构、实时分析等。



## 2.安装，java集成kafka

参考：<https://cloud.tencent.com/developer/article/2393694>

### 

需要注意springboot 和 kafka 版本对应。

### ①依赖

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
    <version>3.0.12</version>
</dependency>

<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>3.3.2</version>
</dependency>
```



### ②yml配置

```yaml
spring: 
  kafka:
    consumer:
      group-id: foo
      auto-offset-reset: earliest
      bootstrap-servers: localhost:9092
    producer:
      bootstrap-servers: localhost:9092
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      properties:
        spring.json.type.mapping: foo:com.vanky.chat.server.kafkatest.Foo,bar:com.vanky.chat.server.kafkatest.Bar
```



### ③kafka 配置类

```java
@Configuration
public class KafkaConfig {

	@Bean
	public ConcurrentKafkaListenerContainerFactory<?, ?> kafkaListenerContainerFactory(
			ConcurrentKafkaListenerContainerFactoryConfigurer configurer,
			ConsumerFactory<Object, Object> kafkaConsumerFactory) {
		ConcurrentKafkaListenerContainerFactory<Object, Object> factory = new ConcurrentKafkaListenerContainerFactory<>();
		configurer.configure(factory, kafkaConsumerFactory);
		//factory.setErrorHandler(new SeekToCurrentErrorHandler(
		//		new DeadLetterPublishingRecoverer(template), 3));
		return factory;
	}

	// 当传输的是个实体类时，进行消息格式转换
	@Bean
	public RecordMessageConverter converter() {
		StringJsonMessageConverter converter = new StringJsonMessageConverter();
		DefaultJackson2JavaTypeMapper typeMapper = new DefaultJackson2JavaTypeMapper();
		typeMapper.setTypePrecedence(Jackson2JavaTypeMapper.TypePrecedence.TYPE_ID);
		typeMapper.addTrustedPackages("com.vanky.chat.server.kafkatest");
		Map<String, Class<?>> mappings = new HashMap<>();
		mappings.put("foo", Foo.class);
		mappings.put("bar", Bar.class);
		typeMapper.setIdClassMapping(mappings);
		converter.setTypeMapper(typeMapper);
		return converter;
	}

	@Bean
	public NewTopic foos() {
		return new NewTopic("foo", 1, (short) 1);
	}

	@Bean
	public NewTopic bars() {
		return new NewTopic("bar", 1, (short) 1);
	}
}
```



### ④保存消息到kafka

```java
@RestController
@RequestMapping("/kafkatest")
@AllArgsConstructor
@Tag(name = "kafka")
@Slf4j
public class Example2Controller {

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @Autowired
    private SendService sendService;

    @PostMapping("/foo")
    @Operation(summary = "foo")
    public void send(@RequestBody Foo foo){
        KafkaCompletableFuture<SendResult<String, Object>> future = (KafkaCompletableFuture<SendResult<String, Object>>)
                kafkaTemplate.send("foo", "modelOne", foo);

        future.thenAccept((result) -> {
            log.info("生产者成功发送消息到" + result.getProducerRecord().topic() + "-> " +
                    result.getProducerRecord().value().toString());
        });
    }

    @PostMapping("/bar")
    @Operation(summary = "bar")
    public void send(@RequestBody Bar bar){
        KafkaCompletableFuture<SendResult<String, Object>> future = (KafkaCompletableFuture<SendResult<String, Object>>)
                kafkaTemplate.send("bar", bar);

        future.thenAccept((result) -> {
            log.info("生产者成功发送消息到" + result.getProducerRecord().topic() + "-> " +
                    result.getProducerRecord().value().toString());
        });
    }

    //异步
    @PostMapping("/ansyc")
    @Operation(summary = "ansyc")
    public void sendAnsyc(@RequestBody Bar bar){
        sendService.sendAnsyc(bar);
    }

    //同步
    @PostMapping("/sync")
    @Operation(summary = "sync")
    public void sendSync(@RequestBody Bar bar){
        sendService.sendSync(bar);
    }
}
```



### ⑤listener  /  handler

1. @KafkaListener：用于将方法标记为 Kafka 消息侦听器。可以在方法上使用此注解来指定要侦听的主题或主题模式，以及其他配置选项，例如组ID和线程数。
2. @KafkaHandler：在使用 @KafkaListener 注解时，如果类中有多个方法，可以使用 @KafkaHandler 注解标记其中一个方法，以指定作为特定消息类型的处理程序。

```java
@Component
public class Example3Listenter {

    @KafkaListener(topics = "ansyc")
    public void listenAnsyc(Bar bar) {
        System.out.println(bar);
    }

    @KafkaListener(topics = "sync")
    public void listenSync(Bar bar) {
        System.out.println(bar);
    }
}


/**
*	或者写成handler形式
*/
@Component
@KafkaListener(id = "handler", topics = {"foo", "bar"})
public class ListenHandler {
    @Autowired
    private KafkaTemplate<Object, Object> kafkaTemplate;

    @KafkaHandler
    public void foo(@Payload Foo foo, @Header(KafkaHeaders.RECEIVED_KEY) String key) {
        System.out.println("key:" + key);
        System.out.println("foo:" + foo.toString());
    }

    @KafkaHandler
    public void foo(Bar bar) {
        System.out.println("bar:" + bar.toString());
    }
}
```