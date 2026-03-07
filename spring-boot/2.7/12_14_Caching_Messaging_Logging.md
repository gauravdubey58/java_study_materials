# 12 — Caching

[← Back to Index](./README.md)

---

## Table of Contents
- [Cache Providers](#cache-providers)
- [Redis Caching Setup](#redis-caching-setup)
- [Caching Annotations in Practice](#caching-annotations-in-practice)
- [Cache Configuration](#cache-configuration)
- [Caffeine (In-Process Cache)](#caffeine-in-process-cache)

---

## Cache Providers

| Provider | Type | Best For |
|----------|------|---------|
| `Simple` | In-memory (ConcurrentHashMap) | Dev/testing |
| `Caffeine` | In-memory with eviction | Single-node, high-performance |
| `Redis` | Distributed | Multi-node, shared cache |
| `Hazelcast` | Distributed | Clustered |
| `EhCache` | In-memory + disk | Local, complex eviction |

---

## Redis Caching Setup

```yaml
spring:
  cache:
    type: redis
  redis:
    host: localhost
    port: 6379
    password: ""
    timeout: 2000ms
    lettuce:
      pool:
        max-active: 8
        max-idle: 4
        min-idle: 1
```

```java
@SpringBootApplication
@EnableCaching
public class MyApp { }

@Configuration
public class CacheConfig {

    @Bean
    public RedisCacheConfiguration defaultCacheConfig() {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .disableCachingNullValues()
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()));
    }

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        Map<String, RedisCacheConfiguration> cacheConfigs = Map.of(
            "products", RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(5)),
            "users",    RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofHours(1)),
            "settings", RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofDays(1))
        );
        return RedisCacheManager.builder(factory)
            .cacheDefaults(defaultCacheConfig())
            .withInitialCacheConfigurations(cacheConfigs)
            .build();
    }
}
```

---

## Caching Annotations in Practice

```java
@Service
@CacheConfig(cacheNames = "products")
public class ProductService {

    // Cache on read — skip method if cached
    @Cacheable(key = "#id", unless = "#result == null")
    public Product findById(Long id) {
        log.info("Fetching product {} from DB", id);    // Won't print on cache hit
        return productRepository.findById(id).orElseThrow();
    }

    // Always execute + update cache
    @CachePut(key = "#result.id")
    public Product create(CreateProductRequest req) {
        return productRepository.save(toEntity(req));
    }

    // Always execute + update cache
    @CachePut(key = "#id")
    public Product update(Long id, UpdateProductRequest req) {
        Product product = findById(id);
        // update fields
        return productRepository.save(product);
    }

    // Remove from cache on delete
    @CacheEvict(key = "#id")
    public void delete(Long id) {
        productRepository.deleteById(id);
    }

    // Clear entire cache (e.g. bulk update)
    @CacheEvict(allEntries = true)
    public void importAll(List<Product> products) {
        productRepository.saveAll(products);
    }

    // Condition: only cache if category is "ELECTRONICS"
    @Cacheable(key = "#id", condition = "#category == 'ELECTRONICS'")
    public Product findByIdAndCategory(Long id, String category) { ... }

    // Scheduled cache bust
    @CacheEvict(allEntries = true)
    @Scheduled(cron = "0 0 3 * * *")   // Clear at 3 AM daily
    public void scheduledCacheEviction() { }
}
```

---

## Caffeine (In-Process Cache)

Best for **single-node** applications needing a fast local cache with expiry and size limits.

```yaml
spring:
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=1000,expireAfterWrite=10m,recordStats
```

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager("users", "products");
        manager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(Duration.ofMinutes(10))
            .expireAfterAccess(Duration.ofMinutes(5))
            .recordStats());
        return manager;
    }
}
```

---

[← Back to Index](./README.md)

---
---

# 13 — Messaging: Kafka & RabbitMQ

[← Back to Index](./README.md)

---

## Apache Kafka

### Setup

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all              # Wait for all replicas
      retries: 3
    consumer:
      group-id: my-service-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.example.events"
```

### Producer

```java
@Service
public class KafkaProducerService {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    // Send string message
    public void send(String topic, String message) {
        kafkaTemplate.send(topic, message)
            .addCallback(
                result -> log.info("Sent to {} partition {}", topic, result.getRecordMetadata().partition()),
                ex     -> log.error("Failed to send message: {}", ex.getMessage())
            );
    }

    // Send object (JSON serialized)
    public void sendOrderEvent(OrderCreatedEvent event) {
        kafkaTemplate.send("order-events", event.getOrderId().toString(), event);
    }

    // Send and wait (blocking)
    public void sendSync(String topic, String message) throws Exception {
        kafkaTemplate.send(topic, message).get(10, TimeUnit.SECONDS);
    }
}
```

### Consumer

```java
@Service
public class KafkaConsumerService {

    // Simple string consumer
    @KafkaListener(topics = "user-events", groupId = "my-group")
    public void listen(String message,
                       @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
                       @Header(KafkaHeaders.OFFSET) long offset) {
        log.info("Received: '{}' from partition {}, offset {}", message, partition, offset);
    }

    // JSON object consumer
    @KafkaListener(topics = "order-events", groupId = "order-processor",
                   containerFactory = "orderKafkaListenerContainerFactory")
    public void handleOrderEvent(
        @Payload OrderCreatedEvent event,
        @Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) String key,
        Acknowledgment ack
    ) {
        try {
            orderService.process(event);
            ack.acknowledge();   // Manual commit
        } catch (Exception e) {
            log.error("Failed to process order event: {}", e.getMessage());
            // Don't ack → message will be redelivered
        }
    }

    // Multiple topics
    @KafkaListener(topics = {"topic-1", "topic-2"}, groupId = "multi-group")
    public void listenMultiple(ConsumerRecord<String, String> record) {
        log.info("Topic: {}, Key: {}, Value: {}", record.topic(), record.key(), record.value());
    }
}
```

### Dead Letter Topic

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaOperations<Object, Object> template) {
    // Retry 3 times, then send to DLT
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(template,
        (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition()));

    FixedBackOff backOff = new FixedBackOff(1000L, 3L);  // 1s delay, 3 retries
    return new DefaultErrorHandler(recoverer, backOff);
}
```

---

## RabbitMQ

### Setup

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /
    listener:
      simple:
        acknowledge-mode: manual    # Manual ack for reliability
        prefetch: 10                # Process 10 messages at a time
```

### Configuration (Exchange, Queue, Binding)

```java
@Configuration
public class RabbitConfig {

    public static final String EXCHANGE   = "order.exchange";
    public static final String QUEUE      = "order.queue";
    public static final String DLQ        = "order.queue.dlq";
    public static final String ROUTING_KEY = "order.created";

    // Dead Letter Queue first
    @Bean
    public Queue deadLetterQueue() {
        return QueueBuilder.durable(DLQ).build();
    }

    // Main queue pointing to DLQ on failure
    @Bean
    public Queue orderQueue() {
        return QueueBuilder.durable(QUEUE)
            .withArgument("x-dead-letter-exchange", "")
            .withArgument("x-dead-letter-routing-key", DLQ)
            .withArgument("x-message-ttl", 60000)          // 1 min TTL
            .build();
    }

    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange(EXCHANGE);
    }

    @Bean
    public Binding binding(Queue orderQueue, TopicExchange orderExchange) {
        return BindingBuilder.bind(orderQueue).to(orderExchange).with(ROUTING_KEY);
    }

    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();   // JSON serialization
    }
}
```

### Producer

```java
@Service
public class RabbitProducerService {

    private final RabbitTemplate rabbitTemplate;

    public void publishOrderCreated(OrderCreatedEvent event) {
        rabbitTemplate.convertAndSend(
            RabbitConfig.EXCHANGE,
            RabbitConfig.ROUTING_KEY,
            event,
            message -> {
                message.getMessageProperties().setHeader("eventType", "ORDER_CREATED");
                message.getMessageProperties().setExpiration("300000");  // 5 min TTL
                return message;
            }
        );
    }
}
```

### Consumer

```java
@Service
public class RabbitConsumerService {

    @RabbitListener(queues = RabbitConfig.QUEUE)
    public void handleOrderEvent(
        OrderCreatedEvent event,
        Channel channel,
        @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag
    ) throws IOException {
        try {
            orderService.process(event);
            channel.basicAck(deliveryTag, false);    // Acknowledge success
        } catch (BusinessException e) {
            channel.basicNack(deliveryTag, false, false);  // Don't requeue → goes to DLQ
        } catch (Exception e) {
            channel.basicNack(deliveryTag, false, true);   // Requeue for retry
        }
    }
}
```

---

[← Back to Index](./README.md)

---
---

# 14 — Logging, DevTools & Embedded Servers

[← Back to Index](./README.md)

---

## Logging

Spring Boot uses **Logback** + **SLF4J** facade by default.

### Usage

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class UserService {

    private static final Logger log = LoggerFactory.getLogger(UserService.class);
    // With Lombok: @Slf4j on the class, then use log directly

    public void process(User user) {
        log.trace("trace level — finest granularity");
        log.debug("Entering process for userId: {}", user.getId());
        log.info("Processing user: {}", user.getEmail());
        log.warn("User {} has no phone number", user.getId());
        log.error("Failed to process user {}: {}", user.getId(), exception.getMessage(), exception);
    }
}
```

### Log Levels (highest to lowest severity)

```
ERROR > WARN > INFO > DEBUG > TRACE
```

Set in properties:
```properties
logging.level.root=INFO                    # Default for everything
logging.level.com.example=DEBUG            # Your packages
logging.level.org.hibernate.SQL=DEBUG      # Hibernate SQL
logging.level.org.springframework.security=DEBUG
```

### Logback Configuration (`logback-spring.xml`)

```xml
<configuration>

    <!-- Console appender -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- Rolling file appender -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/app.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/app-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>50MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>1GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- Dev profile: verbose console -->
    <springProfile name="dev">
        <root level="DEBUG">
            <appender-ref ref="CONSOLE" />
        </root>
        <logger name="com.example" level="DEBUG" />
    </springProfile>

    <!-- Prod profile: warnings to file -->
    <springProfile name="prod">
        <root level="WARN">
            <appender-ref ref="FILE" />
        </root>
        <logger name="com.example" level="INFO" />
    </springProfile>

</configuration>
```

### Structured Logging (JSON for ELK/Loki)

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.3</version>
</dependency>
```

```xml
<appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder" />
</appender>
```

---

## DevTools

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional>   <!-- Not included in production JAR -->
</dependency>
```

### Features

| Feature | What It Does |
|---------|-------------|
| **Auto Restart** | Restarts app when classpath files change (faster than full restart) |
| **LiveReload** | Triggers browser refresh when resources change (requires browser extension) |
| **Property Defaults** | Disables template caching, enables h2-console, sets DEBUG logging |
| **Remote Restart** | Supports restart of remote application over HTTP |

```properties
# DevTools configuration
spring.devtools.restart.enabled=true
spring.devtools.restart.exclude=static/**,public/**,templates/**
spring.devtools.livereload.enabled=true
spring.devtools.restart.poll-interval=2s
spring.devtools.restart.quiet-period=1s
```

> ℹ️ DevTools uses **two classloaders** — one for libraries (doesn't change), one for your code (reloaded on change). This makes restarts much faster than a full JVM restart.

---

## Embedded Servers

Spring Boot packages an HTTP server inside the JAR — no external deployment needed.

### Default: Apache Tomcat

```yaml
server:
  port: 8080
  tomcat:
    max-threads: 200
    min-spare-threads: 10
    max-connections: 10000
    accept-count: 100
    connection-timeout: 20000
```

### Switch to Jetty

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

### Switch to Undertow

```xml
<!-- Same exclusion as above, then add: -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

### Server Comparison

| Feature | Tomcat | Jetty | Undertow |
|---------|--------|-------|----------|
| Default | ✅ Yes | No | No |
| Threading | Blocking | Blocking + Async | Non-blocking (NIO) |
| Memory usage | Moderate | Low | Low |
| Performance | Good | Good | Excellent |
| HTTP/2 | ✅ | ✅ | ✅ |
| WebSocket | ✅ | ✅ | ✅ |
| Best for | General use | Eclipse ecosystem | High-concurrency |

### HTTPS / SSL

```yaml
server:
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: changeit
    key-store-type: PKCS12
    key-alias: myapp
  port: 8443
```

```bash
# Generate self-signed certificate
keytool -genkeypair -alias myapp -keyalg RSA -keysize 2048 \
  -storetype PKCS12 -keystore keystore.p12 -validity 365
```

---

[← Exception Handling & REST](./11_Exception_Handling_REST.md) &nbsp;|&nbsp; [Next → Interview Questions](./15_Interview_Questions.md)
