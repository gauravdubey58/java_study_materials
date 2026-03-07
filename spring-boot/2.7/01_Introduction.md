# 01 — Introduction & What's New in Spring Boot 2.7

[← Back to Index](./README.md)

---

## Table of Contents
- [What is Spring Boot?](#what-is-spring-boot)
- [Core Philosophy](#core-philosophy)
- [Spring Boot 2.7 Highlights](#spring-boot-27-highlights)
- [Application Startup Process](#application-startup-process)
- [Spring Boot vs Spring Framework](#spring-boot-vs-spring-framework)

---

## What is Spring Boot?

Spring Boot is an **opinionated, convention-over-configuration** extension of the Spring Framework that simplifies the bootstrapping and development of Spring applications. It removes the need for boilerplate XML configuration and enables developers to create **standalone, production-grade** Spring-based applications with minimal setup.

```java
// This is all you need to start a Spring Boot web application
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

---

## Core Philosophy

| Principle | Meaning |
|-----------|---------|
| **Convention over Configuration** | Sensible defaults — you only configure what differs |
| **Opinionated Defaults** | Pre-configured starters and auto-configuration |
| **Standalone** | Embeds Tomcat/Jetty/Undertow; no WAR deployment needed |
| **Production-Ready** | Metrics, health checks, and externalized configuration included |

---

## Spring Boot 2.7 Highlights

### 🔐 Security Changes (Breaking)
- `WebSecurityConfigurerAdapter` is **deprecated** — migrate to `SecurityFilterChain` beans
- New `@EnableMethodSecurity` replaces `@EnableGlobalMethodSecurity`
- Spring Security updated to **5.7.x**

### ⚙️ Auto-Configuration
- New registration file: `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
- `spring.factories` still works but is deprecated for auto-config registration
- New `@AutoConfiguration` annotation (replaces `@Configuration` on auto-config classes)

### 🖥️ GraalVM Native Image
- Improved AOT (Ahead-Of-Time) processing
- `RuntimeHintsRegistrar` API for registering reflection, resource, and proxy hints
- `spring-aot-maven-plugin` for build-time processing

### 📦 Dependency Updates
- Spring Framework **5.3.x**
- Spring Data **2021.2**
- Hibernate **5.6.x**
- Kotlin **1.6.x**
- Tomcat **9.x**

### 🩺 Actuator Improvements
- Liveness and Readiness probes for Kubernetes out of the box
- New `startup` endpoint
- Improved metric tagging

### 🧪 Testing
- `@SpringBootTest` improvements for Docker Compose integration
- Improved `@DataJpaTest` with better defaults

---

## Application Startup Process

```
SpringApplication.run(MyApp.class, args)
         │
         ▼
1. Creates SpringApplication instance
         │
         ▼
2. Determines ApplicationContext type
   (Servlet / Reactive / None)
         │
         ▼
3. Loads ApplicationContextInitializers
   and ApplicationListeners from spring.factories
         │
         ▼
4. Publishes ApplicationStartingEvent
         │
         ▼
5. Prepares Environment
   (loads application.properties / .yml)
         │
         ▼
6. Prints Banner 🍃
         │
         ▼
7. Creates ApplicationContext
         │
         ▼
8. Runs BeanFactoryPostProcessors
         │
         ▼
9. Registers and instantiates Beans
   (Component scan + Auto-configuration)
         │
         ▼
10. Runs BeanPostProcessors
    (AOP proxies, @Autowired, @Value injection)
         │
         ▼
11. Publishes ApplicationStartedEvent
         │
         ▼
12. Runs CommandLineRunner / ApplicationRunner beans
         │
         ▼
13. Publishes ApplicationReadyEvent
         │
         ▼
14. Embedded Server starts, listens for requests ✅
```

---

## Spring Boot vs Spring Framework

| Aspect | Spring Framework | Spring Boot |
|--------|-----------------|-------------|
| Configuration | Explicit XML or Java config | Auto-configured, minimal config |
| Server | Deploy to external server (WAR) | Embedded server (JAR) |
| Dependencies | Manage versions manually | Starters manage compatible versions |
| Boilerplate | High | Very low |
| Startup time | Slower (more config) | Faster to get started |
| Production features | Manual setup | Built-in (Actuator) |
| Learning curve | Steeper | Gentler |
| Flexibility | Maximum | Opinionated (but escapable) |

---

[Next → Core Concepts & Terminology](./02_Core_Concepts.md)
