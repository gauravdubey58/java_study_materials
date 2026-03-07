# 03 — Architecture & Auto-Configuration

[← Back to Index](./README.md)

---

## Table of Contents
- [Spring Boot Architecture Overview](#spring-boot-architecture-overview)
- [How Auto-Configuration Works](#how-auto-configuration-works)
- [All @ConditionalOn* Annotations](#all-conditionalon-annotations)
- [Writing Custom Auto-Configuration](#writing-custom-auto-configuration)
- [Excluding Auto-Configuration](#excluding-auto-configuration)
- [Debugging Auto-Configuration](#debugging-auto-configuration)
- [spring.factories vs AutoConfiguration.imports](#springfactories-vs-autoconfigurationimports)

---

## Spring Boot Architecture Overview

```
┌────────────────────────────────────────────────────────────────┐
│                     Spring Boot Application                    │
├─────────────────────────┬──────────────────────────────────────┤
│   Presentation Layer    │  @RestController  @Controller        │
│                         │  @RequestMapping  @ExceptionHandler  │
├─────────────────────────┼──────────────────────────────────────┤
│   Service Layer         │  @Service  Business Logic            │
│                         │  @Transactional  @Async              │
├─────────────────────────┼──────────────────────────────────────┤
│   Data Access Layer     │  @Repository  Spring Data JPA        │
│                         │  JpaRepository  @Query               │
├─────────────────────────┼──────────────────────────────────────┤
│   Database              │  MySQL / PostgreSQL / H2 / MongoDB   │
├─────────────────────────┴──────────────────────────────────────┤
│   Cross-Cutting Concerns:  AOP · Security · Transactions        │
│   Infrastructure:          Auto-Config · Embedded Server       │
└────────────────────────────────────────────────────────────────┘
```

### Layer Responsibilities

| Layer | Annotation | Responsibility |
|-------|-----------|---------------|
| Presentation | `@RestController` | Handle HTTP requests/responses, input validation |
| Service | `@Service` | Business logic, transaction management |
| Repository | `@Repository` | Database operations, exception translation |
| Entity | `@Entity` | Data model mapping to DB tables |
| Config | `@Configuration` | Bean definitions, infrastructure setup |

---

## How Auto-Configuration Works

Spring Boot auto-configuration is **not magic** — here is exactly what happens:

### Step-by-Step Mechanism

```
@SpringBootApplication
       │
       ├── @EnableAutoConfiguration
       │          │
       │          └── AutoConfigurationImportSelector
       │                      │
       │          Reads: META-INF/spring/
       │          org.springframework.boot.autoconfigure
       │          .AutoConfiguration.imports
       │                      │
       │          For each class listed:
       │          ┌────────────────────────────┐
       │          │  Evaluate @ConditionalOn*  │
       │          │  annotations               │
       │          └────────────────────────────┘
       │                      │
       │          ✅ Conditions met → apply config
       │          ❌ Conditions not met → skip
       │
       └── @ComponentScan  → scan @Component, @Service, @Repository, @Controller
```

### Key Insight: User Beans Win

Auto-configuration always uses `@ConditionalOnMissingBean` for the beans it creates. This means if **you** define a bean of the same type, Spring Boot's auto-configured bean is skipped — your definition takes precedence.

```java
// Auto-configuration class (simplified)
@AutoConfiguration
@ConditionalOnClass(DataSource.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean   // ← Only created if you haven't defined a DataSource bean
    public DataSource dataSource(DataSourceProperties props) {
        return props.initializeDataSourceBuilder().build();
    }
}
```

---

## All @ConditionalOn* Annotations

| Annotation | Applied When... | Example Use Case |
|------------|----------------|-----------------|
| `@ConditionalOnClass` | Class is present on classpath | Configure Redis if Lettuce/Jedis is on classpath |
| `@ConditionalOnMissingClass` | Class is NOT on classpath | Provide fallback if a lib is absent |
| `@ConditionalOnBean` | A specific bean already exists | Only create X if Y already defined |
| `@ConditionalOnMissingBean` | Bean of the type does NOT exist | Allow user to override default impl |
| `@ConditionalOnProperty` | Property has a specific value | Feature flags |
| `@ConditionalOnResource` | A resource file exists | Only if `schema.sql` exists on classpath |
| `@ConditionalOnWebApplication` | Running as a web application | Web-only configuration |
| `@ConditionalOnNotWebApplication` | NOT a web application | Batch/CLI-only configuration |
| `@ConditionalOnExpression` | SpEL expression is `true` | Complex conditions |
| `@ConditionalOnJava` | JVM version matches | Java 17+ features |
| `@ConditionalOnSingleCandidate` | Exactly one bean of the type exists | Auto-configure only with one datasource |
| `@ConditionalOnCloudPlatform` | Running on specified cloud (K8s, PCF) | Cloud-specific config |
| `@ConditionalOnJndi` | JNDI location available | Enterprise/app-server deployment |

### Examples

```java
// Only configure if Redis client is on classpath
@ConditionalOnClass(name = "io.lettuce.core.RedisClient")

// Only create if property is set to "true"
@ConditionalOnProperty(name = "feature.notifications.enabled", havingValue = "true", matchIfMissing = false)

// Only in a web context
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)

// Only on Java 17+
@ConditionalOnJava(JavaVersion.SEVENTEEN)

// SpEL expression
@ConditionalOnExpression("${app.feature.enabled:false} && '${app.env}' == 'prod'")
```

---

## Writing Custom Auto-Configuration

Use this when building a shared library or internal starter.

### Step 1: Create the Auto-Configuration Class

```java
package com.mycompany.autoconfigure;

@AutoConfiguration                              // Spring Boot 2.7+ annotation
@ConditionalOnClass(MyService.class)            // Only if our service class is present
@ConditionalOnProperty(
    prefix = "mycompany",
    name = "enabled",
    havingValue = "true",
    matchIfMissing = true
)
@EnableConfigurationProperties(MyProperties.class)
public class MyServiceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean                   // Allow user override
    public MyService myService(MyProperties props) {
        return new MyService(props.getApiKey());
    }
}
```

### Step 2: Register the Auto-Configuration

Create this file in your library JAR:

```
src/main/resources/META-INF/spring/
org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

Content:
```
com.mycompany.autoconfigure.MyServiceAutoConfiguration
```

### Step 3: Create Properties Class

```java
@ConfigurationProperties(prefix = "mycompany")
public class MyProperties {
    private String apiKey;
    private boolean enabled = true;
    private Duration timeout = Duration.ofSeconds(30);

    // getters and setters
}
```

### Step 4: User Configuration (in their `application.yml`)

```yaml
mycompany:
  enabled: true
  api-key: abc123
  timeout: 60s
```

---

## Excluding Auto-Configuration

Sometimes you want to prevent a specific auto-configuration from running:

```java
// Option 1: In @SpringBootApplication
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    SecurityAutoConfiguration.class
})
public class MyApp { }

// Option 2: In application.properties
// spring.autoconfigure.exclude=\
//   org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
//   org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration
```

---

## Debugging Auto-Configuration

### Method 1: `--debug` flag
```bash
java -jar app.jar --debug
```
Prints a detailed **Conditions Evaluation Report** showing which auto-configurations were applied and which were skipped (and why).

### Method 2: Property
```properties
debug=true
```

### Method 3: Actuator
```
GET /actuator/conditions
```
Returns the conditions report in JSON format.

### Sample Output
```
============================
CONDITIONS EVALUATION REPORT
============================

Positive matches:
-----------------
   DataSourceAutoConfiguration matched:
      - @ConditionalOnClass found required class 'javax.sql.DataSource' (OnClassCondition)

Negative matches:
-----------------
   MongoAutoConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required class 'com.mongodb.MongoClient' (OnClassCondition)
```

---

## spring.factories vs AutoConfiguration.imports

This is a **Spring Boot 2.7 specific change**.

| | Old (`spring.factories`) | New (2.7+) |
|-|--------------------------|------------|
| File | `META-INF/spring.factories` | `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` |
| Format | Key=Value pairs | One class per line |
| Class annotation | `@Configuration` | `@AutoConfiguration` |
| Status | Deprecated for auto-config | Preferred |

### Old Format (`spring.factories`)
```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.FooAutoConfiguration,\
  com.example.BarAutoConfiguration
```

### New Format (2.7+) — `AutoConfiguration.imports`
```
com.example.FooAutoConfiguration
com.example.BarAutoConfiguration
```

> **Both formats work** in Boot 2.7 for backward compatibility, but new code should use the new file format.

---

[← Core Concepts](./02_Core_Concepts.md) &nbsp;|&nbsp; [Next → Starters & Configuration](./04_Starters_and_Configuration.md)
