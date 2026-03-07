# 04 — Starters & Configuration

[← Back to Index](./README.md)

---

## Table of Contents
- [All Spring Boot Starters](#all-spring-boot-starters)
- [application.properties vs application.yml](#applicationproperties-vs-applicationyml)
- [Important Properties Reference](#important-properties-reference)
- [@ConfigurationProperties](#configurationproperties)
- [Custom Configuration with Validation](#custom-configuration-with-validation)

---

## All Spring Boot Starters

### Core

| Starter | What It Brings |
|---------|---------------|
| `spring-boot-starter` | Core auto-configuration, logging (Logback), YAML |
| `spring-boot-starter-web` | Spring MVC + Jackson + Tomcat |
| `spring-boot-starter-webflux` | Reactive web with Project Reactor + Netty |
| `spring-boot-starter-aop` | AspectJ + Spring AOP |
| `spring-boot-starter-validation` | Hibernate Validator (JSR-380 Bean Validation) |

### Data

| Starter | What It Brings |
|---------|---------------|
| `spring-boot-starter-data-jpa` | Spring Data JPA + Hibernate + HikariCP |
| `spring-boot-starter-data-mongodb` | Spring Data MongoDB + Mongo driver |
| `spring-boot-starter-data-redis` | Spring Data Redis + Lettuce client |
| `spring-boot-starter-data-elasticsearch` | Spring Data Elasticsearch |
| `spring-boot-starter-data-cassandra` | Spring Data Cassandra |
| `spring-boot-starter-data-neo4j` | Spring Data Neo4j |
| `spring-boot-starter-data-r2dbc` | Reactive relational DB (R2DBC) |
| `spring-boot-starter-jdbc` | Spring JDBC + HikariCP (without JPA) |

### Security

| Starter | What It Brings |
|---------|---------------|
| `spring-boot-starter-security` | Spring Security (authentication + authorization) |
| `spring-boot-starter-oauth2-client` | OAuth 2.0 / OIDC client support |
| `spring-boot-starter-oauth2-resource-server` | JWT / opaque token resource server |

### Messaging

| Starter | What It Brings |
|---------|---------------|
| `spring-boot-starter-amqp` | Spring AMQP + RabbitMQ client |
| `spring-boot-starter-kafka` | Spring for Apache Kafka |
| `spring-boot-starter-websocket` | WebSocket + STOMP support |
| `spring-boot-starter-activemq` | Apache ActiveMQ |
| `spring-boot-starter-artemis` | Apache Artemis (ActiveMQ next-gen) |

### Monitoring & Operations

| Starter | What It Brings |
|---------|---------------|
| `spring-boot-starter-actuator` | Health, metrics, info, env endpoints |
| `spring-boot-starter-logging` | Logback (included by default) |
| `spring-boot-starter-log4j2` | Log4j2 (replace default Logback) |

### Developer Tools

| Starter | What It Brings |
|---------|---------------|
| `spring-boot-devtools` | Hot restart, LiveReload, property defaults |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ, MockMvc, TestRestTemplate |

### Integrations & Others

| Starter | What It Brings |
|---------|---------------|
| `spring-boot-starter-cache` | Spring Cache abstraction |
| `spring-boot-starter-mail` | JavaMailSender (SMTP) |
| `spring-boot-starter-quartz` | Quartz Scheduler |
| `spring-boot-starter-batch` | Spring Batch |
| `spring-boot-starter-thymeleaf` | Thymeleaf templating engine |
| `spring-boot-starter-freemarker` | Freemarker templating engine |
| `spring-boot-starter-mustache` | Mustache templating engine |
| `spring-boot-starter-graphql` | Spring GraphQL |
| `spring-boot-starter-hateoas` | HATEOAS + HAL support |

---

## application.properties vs application.yml

Both work identically — choose based on team preference. YAML is more readable for nested configs.

```properties
# application.properties — flat key=value
server.port=8080
server.servlet.context-path=/api

spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=secret

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

logging.level.org.springframework=DEBUG
logging.level.com.example=INFO
```

```yaml
# application.yml — nested, human-friendly
server:
  port: 8080
  servlet:
    context-path: /api

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: secret
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true

logging:
  level:
    org.springframework: DEBUG
    com.example: INFO
```

---

## Important Properties Reference

### Server

```properties
server.port=8080
server.servlet.context-path=/api
server.ssl.enabled=true
server.ssl.key-store=classpath:keystore.jks
server.ssl.key-store-password=secret
server.compression.enabled=true
server.compression.min-response-size=2048
server.error.include-message=always
server.shutdown=graceful                         # Graceful shutdown (Boot 2.3+)
spring.lifecycle.timeout-per-shutdown-phase=20s
```

### Database / JPA

```properties
# DataSource
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.username=root
spring.datasource.password=secret

# H2 in-memory (dev/test)
spring.datasource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console

# JPA / Hibernate
spring.jpa.hibernate.ddl-auto=none         # none|validate|update|create|create-drop
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect
spring.jpa.open-in-view=false              # ⚠️ Disable for better performance

# Connection Pool (HikariCP)
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
```

### Spring Security

```properties
spring.security.user.name=admin
spring.security.user.password=admin123
spring.security.user.roles=ADMIN
```

### Spring Cache / Redis

```properties
spring.cache.type=redis
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.password=
spring.redis.database=0
spring.redis.timeout=2000ms
spring.cache.redis.time-to-live=600000      # 10 minutes in ms
spring.cache.redis.cache-null-values=false
```

### Spring Kafka

```properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=my-group
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.apache.kafka.common.serialization.StringSerializer
```

### Mail

```properties
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=you@gmail.com
spring.mail.password=your-app-password
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
```

### Actuator

```properties
management.endpoints.web.exposure.include=health,info,metrics,env,loggers
management.endpoints.web.base-path=/actuator
management.endpoint.health.show-details=always
management.endpoint.health.show-components=always
management.server.port=9090                 # Separate port for actuator
info.app.name=My Application
info.app.version=1.0.0
info.app.environment=production
```

### Logging

```properties
logging.level.root=INFO
logging.level.com.example=DEBUG
logging.level.org.springframework.security=DEBUG
logging.file.name=logs/app.log
logging.file.max-size=10MB
logging.file.max-history=30
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{36} - %msg%n
logging.pattern.file=%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{36} - %msg%n
```

### Flyway

```properties
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
spring.flyway.baseline-on-migrate=true
spring.flyway.validate-on-migrate=true
```

### Async / Scheduling

```properties
spring.task.execution.pool.core-size=5
spring.task.execution.pool.max-size=20
spring.task.execution.pool.queue-capacity=100
spring.task.execution.thread-name-prefix=task-
spring.task.scheduling.pool.size=5
```

---

## @ConfigurationProperties

Bind a group of related properties to a type-safe POJO — preferred over multiple `@Value` annotations.

### Basic Example

```java
@ConfigurationProperties(prefix = "app")
@Component  // or register via @EnableConfigurationProperties
public class AppProperties {

    private String name = "DefaultApp";       // Default value
    private String version;
    private boolean featureEnabled = false;
    private Duration sessionTimeout = Duration.ofMinutes(30);
    private final Mail mail = new Mail();
    private final List<String> allowedOrigins = new ArrayList<>();

    // ---- Nested class ----
    public static class Mail {
        private String host;
        private int port = 587;
        private String from;
        // getters + setters
    }

    // getters + setters for all fields
}
```

```yaml
app:
  name: ShopApp
  version: 2.1.0
  feature-enabled: true         # relaxed binding: maps to featureEnabled
  session-timeout: 1h           # Duration format
  mail:
    host: smtp.gmail.com
    port: 587
    from: noreply@shopapp.com
  allowed-origins:
    - http://localhost:3000
    - https://shopapp.com
```

### Registering Without `@Component`

```java
// In a @Configuration class
@EnableConfigurationProperties(AppProperties.class)

// Or in main class
@SpringBootApplication
@EnableConfigurationProperties({AppProperties.class, DatabaseProperties.class})
public class MyApp { }
```

---

## Custom Configuration with Validation

Combine `@ConfigurationProperties` with Bean Validation to validate config at startup:

```java
@ConfigurationProperties(prefix = "payment")
@Validated
public class PaymentProperties {

    @NotBlank(message = "Payment API key must not be blank")
    private String apiKey;

    @NotNull
    @Min(1) @Max(10)
    private Integer maxRetries;

    @NotNull
    @DurationMin(seconds = 1)
    @DurationMax(minutes = 5)
    private Duration timeout;

    @URL
    private String gatewayUrl;

    // getters + setters
}
```

If validation fails, the application **refuses to start** with a clear error message — fail-fast behaviour.

---

[← Architecture & Auto-Config](./03_Architecture_AutoConfiguration.md) &nbsp;|&nbsp; [Next → Annotations Reference](./05_Annotations_Reference.md)
