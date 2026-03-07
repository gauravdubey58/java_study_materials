# 🍃 Spring Boot 2.7 — Complete Study Notes

> **Spring Boot:** 2.7.x &nbsp;|&nbsp; **Spring Framework:** 5.3.x &nbsp;|&nbsp; **Java:** 8 / 11 / 17  
> **Intended For:** Developers · Interview Prep · GitHub Reference

---

## 📂 How This Repo Is Organised

Each topic lives in its own file so you can navigate directly to what you need.

| # | File | What's Inside |
|---|------|---------------|
| 01 | [Introduction & What's New](./01_Introduction.md) | What is Spring Boot, Boot 2.7 highlights, startup process |
| 02 | [Core Concepts & Terminology](./02_Core_Concepts.md) | IoC, DI, Beans, Scopes, AOP, Transactions, and more |
| 03 | [Architecture & Auto-Configuration](./03_Architecture_AutoConfiguration.md) | Boot architecture, auto-config mechanism, conditional annotations |
| 04 | [Starters & Configuration](./04_Starters_and_Configuration.md) | All starters, `application.properties` / `.yml`, `@ConfigurationProperties` |
| 05 | [Annotations Reference](./05_Annotations_Reference.md) | Every annotation with definition + use case + code snippet |
| 06 | [Spring Data JPA](./06_Spring_Data_JPA.md) | Repositories, queries, pagination, N+1 fix, Specifications, locking |
| 07 | [Spring Security](./07_Spring_Security.md) | SecurityFilterChain, JWT, method security (Boot 2.7 style) |
| 08 | [Actuator & Monitoring](./08_Actuator.md) | Endpoints, custom health indicators, Kubernetes probes |
| 09 | [Testing](./09_Testing.md) | Unit, MockMvc, integration tests, all test annotations |
| 10 | [Profiles & Configuration Management](./10_Profiles.md) | Profile setup, activation, environment-specific config |
| 11 | [Exception Handling & REST APIs](./11_Exception_Handling_REST.md) | `@ControllerAdvice`, error responses, REST best practices |
| 12 | [Caching](./12_Caching.md) | `@Cacheable`, Redis config, eviction strategies |
| 13 | [Messaging — Kafka & RabbitMQ](./13_Messaging.md) | Producers, consumers, listeners |
| 14 | [Logging, DevTools & Embedded Servers](./14_Logging_DevTools_Servers.md) | Logback, log profiles, DevTools, Tomcat/Jetty/Undertow |
| 15 | [100 Interview Questions](./15_Interview_Questions.md) | Q1–Q50 Moderate · Q51–Q100 Tough |

---

## ⚡ Quick-Start Cheat Sheet

```bash
# Create a new project
https://start.spring.io

# Run locally
./mvnw spring-boot:run

# Build executable JAR
./mvnw clean package

# Run JAR
java -jar target/app.jar

# Run with profile
java -jar target/app.jar --spring.profiles.active=prod

# Debug auto-configuration
java -jar target/app.jar --debug
```

---

## 🗂️ Key File Locations

| Purpose | Path |
|---------|------|
| Main class | `src/main/java/.../Application.java` |
| Properties | `src/main/resources/application.properties` |
| Profile properties | `src/main/resources/application-{profile}.properties` |
| Static resources | `src/main/resources/static/` |
| Templates (Thymeleaf) | `src/main/resources/templates/` |
| Flyway migrations | `src/main/resources/db/migration/V1__init.sql` |
| Auto-config registry | `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` |
| Test resources | `src/test/resources/application-test.properties` |

---

## 🏷️ Spring Boot 2.7 at a Glance

| Component | Version |
|-----------|---------|
| Spring Framework | 5.3.x |
| Spring Security | 5.7.x |
| Spring Data | 2021.2 |
| Hibernate | 5.6.x |
| Tomcat | 9.x |
| Kotlin | 1.6.x |
| Java | 8, 11, 17 |

---

> 💡 **Tip:** If you are preparing for interviews, start with [Core Concepts](./02_Core_Concepts.md) and finish with [Interview Questions](./15_Interview_Questions.md).  
> 🐛 Found an error? PRs are welcome!
