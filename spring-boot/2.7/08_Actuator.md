# 08 — Actuator & Monitoring

[← Back to Index](./README.md)

---

## Table of Contents
- [Setup](#setup)
- [All Built-in Endpoints](#all-built-in-endpoints)
- [Configuration Reference](#configuration-reference)
- [Custom Health Indicator](#custom-health-indicator)
- [Custom Endpoint](#custom-endpoint)
- [Kubernetes Probes](#kubernetes-probes)
- [Metrics with Micrometer](#metrics-with-micrometer)

---

## Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

---

## All Built-in Endpoints

| Endpoint | HTTP Method | Description |
|----------|------------|-------------|
| `/actuator/health` | GET | App health status (UP/DOWN) |
| `/actuator/info` | GET | Arbitrary app information |
| `/actuator/metrics` | GET | All available metric names |
| `/actuator/metrics/{name}` | GET | Specific metric value |
| `/actuator/env` | GET | All environment properties |
| `/actuator/env/{key}` | GET | Specific property |
| `/actuator/beans` | GET | All Spring beans in context |
| `/actuator/mappings` | GET | All `@RequestMapping` routes |
| `/actuator/loggers` | GET | All logger levels |
| `/actuator/loggers/{name}` | GET / POST | Get or change log level at runtime |
| `/actuator/conditions` | GET | Auto-configuration evaluation report |
| `/actuator/configprops` | GET | All `@ConfigurationProperties` |
| `/actuator/scheduledtasks` | GET | All `@Scheduled` tasks |
| `/actuator/caches` | GET | All caches |
| `/actuator/heapdump` | GET | JVM heap dump (binary) |
| `/actuator/threaddump` | GET | Thread dump |
| `/actuator/httptrace` | GET | Last 100 HTTP request traces |
| `/actuator/startup` | GET | Startup step timeline |
| `/actuator/shutdown` | POST | Graceful shutdown (disabled by default) |

---

## Configuration Reference

```yaml
management:
  # Which endpoints are exposed over HTTP
  endpoints:
    web:
      exposure:
        include: health, info, metrics, env, beans, loggers, conditions
        # include: "*"   ← exposes ALL (not recommended in prod)
        exclude: shutdown
      base-path: /actuator

  # Per-endpoint configuration
  endpoint:
    health:
      show-details: always        # never | when-authorized | always
      show-components: always
    loggers:
      enabled: true
    shutdown:
      enabled: false              # Explicitly disabled by default

  # Separate port for actuator (security best practice)
  server:
    port: 9090
    address: 127.0.0.1            # Only local access

  # Info contributor
  info:
    env:
      enabled: true               # Enable info.* properties
    build:
      enabled: true               # Show build info (requires META-INF/build-info.properties)
    git:
      enabled: true               # Show git info (requires git.properties)

# Custom info values
info:
  app:
    name: My Application
    version: "@project.version@"  # Injected from Maven pom.xml
    description: Sample Spring Boot app

# Change log level at runtime
# POST /actuator/loggers/com.example
# Body: { "configuredLevel": "DEBUG" }
```

---

## Custom Health Indicator

```java
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {

    private final ExternalApiClient client;

    public ExternalApiHealthIndicator(ExternalApiClient client) {
        this.client = client;
    }

    @Override
    public Health health() {
        try {
            boolean reachable = client.ping();
            if (reachable) {
                return Health.up()
                    .withDetail("service", "External Payment API")
                    .withDetail("url", client.getBaseUrl())
                    .withDetail("responseTime", "45ms")
                    .build();
            }
            return Health.down()
                .withDetail("error", "Service unreachable")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("exception", e.getMessage())
                .build();
        }
    }
}
```

Response at `/actuator/health`:
```json
{
  "status": "UP",
  "components": {
    "db": { "status": "UP" },
    "externalApi": {
      "status": "UP",
      "details": { "service": "External Payment API", "responseTime": "45ms" }
    },
    "redis": { "status": "UP" }
  }
}
```

---

## Custom Endpoint

```java
@Component
@Endpoint(id = "feature-flags")  // Accessible at /actuator/feature-flags
public class FeatureFlagsEndpoint {

    private final Map<String, Boolean> flags = new ConcurrentHashMap<>();

    @ReadOperation          // GET /actuator/feature-flags
    public Map<String, Boolean> getFlags() {
        return Collections.unmodifiableMap(flags);
    }

    @ReadOperation          // GET /actuator/feature-flags/{name}
    public Boolean getFlag(@Selector String name) {
        return flags.getOrDefault(name, false);
    }

    @WriteOperation         // POST /actuator/feature-flags
    public void setFlag(String name, boolean enabled) {
        flags.put(name, enabled);
    }

    @DeleteOperation        // DELETE /actuator/feature-flags/{name}
    public void removeFlag(@Selector String name) {
        flags.remove(name);
    }
}
```

---

## Kubernetes Probes

Spring Boot 2.7 has built-in liveness and readiness probes.

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

| Probe | URL | Meaning |
|-------|-----|---------|
| Liveness | `/actuator/health/liveness` | Is the app alive? (restart if DOWN) |
| Readiness | `/actuator/health/readiness` | Is the app ready for traffic? (remove from LB if DOWN) |

```yaml
# Kubernetes deployment.yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 5
```

---

## Metrics with Micrometer

Spring Boot auto-configures Micrometer metrics.

```xml
<!-- Prometheus export -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: prometheus
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active}
```

### Custom Metrics

```java
@Service
public class OrderService {

    private final Counter orderCounter;
    private final Timer orderTimer;

    public OrderService(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.created")
            .description("Total orders created")
            .tag("type", "standard")
            .register(registry);

        this.orderTimer = Timer.builder("orders.processing.time")
            .description("Time to process an order")
            .register(registry);
    }

    public Order placeOrder(OrderRequest request) {
        return orderTimer.record(() -> {
            Order order = processOrder(request);
            orderCounter.increment();
            return order;
        });
    }
}
```

---

[← Back to Index](./README.md)
