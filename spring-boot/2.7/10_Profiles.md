# 10 — Profiles & Configuration Management

[← Back to Index](./README.md)

---

## Table of Contents
- [What Are Profiles?](#what-are-profiles)
- [Defining Profile Configurations](#defining-profile-configurations)
- [Activating Profiles](#activating-profiles)
- [Profile-Specific Beans](#profile-specific-beans)
- [Multi-Document YAML Files](#multi-document-yaml-files)

---

## What Are Profiles?

Profiles allow you to have **environment-specific** configuration. Common profiles: `dev`, `test`, `staging`, `prod`.

---

## Defining Profile Configurations

### Separate Files (Recommended)

```
src/main/resources/
├── application.yml            ← Shared across all profiles
├── application-dev.yml        ← Overrides for dev
├── application-test.yml       ← Overrides for test
├── application-staging.yml    ← Overrides for staging
└── application-prod.yml       ← Overrides for prod
```

```yaml
# application.yml (shared)
app:
  name: My Application
spring:
  jpa:
    open-in-view: false
server:
  port: 8080
```

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:h2:mem:devdb
  h2:
    console:
      enabled: true
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
logging:
  level:
    com.example: DEBUG
    org.hibernate.SQL: DEBUG
```

```yaml
# application-prod.yml
spring:
  datasource:
    url: ${DB_URL}             # From environment variable
    username: ${DB_USER}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate       # Never modify schema in prod
server:
  port: ${PORT:8080}
logging:
  level:
    root: WARN
    com.example: INFO
```

---

## Activating Profiles

```bash
# 1. Command-line argument (highest priority)
java -jar app.jar --spring.profiles.active=prod

# 2. Environment variable
export SPRING_PROFILES_ACTIVE=prod
java -jar app.jar

# 3. In application.properties (lowest priority, mostly for dev convenience)
# spring.profiles.active=dev

# 4. Programmatically
SpringApplication app = new SpringApplication(MyApp.class);
app.setAdditionalProfiles("dev", "local");
app.run(args);
```

> ⚠️ **Never hardcode** `spring.profiles.active=prod` in `application.properties`.  
> It gets bundled into the JAR and is hard to override.

---

## Profile-Specific Beans

```java
// Only active in dev
@Configuration
@Profile("dev")
public class DevConfig {

    @Bean
    public DataSource h2DataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }

    @Bean
    public CommandLineRunner devDataLoader(UserRepository repo) {
        return args -> {
            repo.save(new User("dev-user@example.com", "password"));
            log.info("Dev data loaded!");
        };
    }
}

// Active in staging AND prod
@Configuration
@Profile({"staging", "prod"})
public class CloudConfig {
    @Bean
    public AmazonS3 s3Client() { ... }
}

// Active in everything EXCEPT dev
@Component
@Profile("!dev")
public class RealEmailService implements EmailService { ... }

@Component
@Profile("dev")
public class MockEmailService implements EmailService { ... }
```

---

## Multi-Document YAML Files

You can use `---` separator to define profile sections within a single `application.yml`:

```yaml
# Shared config (applies to all profiles)
app:
  name: My App
server:
  port: 8080

---
# Dev config
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:devdb

---
# Prod config
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: ${DB_URL}
```

---

[← Testing](./09_Testing.md) &nbsp;|&nbsp; [Next → Exception Handling & REST](./11_Exception_Handling_REST.md)
