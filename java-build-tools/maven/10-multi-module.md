[← Chapter 9: Profiles](./09-profiles.md) | [Index](../index.md)

---

# Chapter 10 — Multi-Module Projects

A **multi-module project** (also called a reactor build) groups several related Maven projects under a single parent. One `mvn install` at the root builds them all in the correct dependency order.

---

## Directory Structure

```
my-project/                          ← root (aggregator)
├── pom.xml                          ← parent/aggregator POM
├── my-core/
│   ├── pom.xml
│   └── src/
├── my-service/
│   ├── pom.xml
│   └── src/
└── my-web/
    ├── pom.xml
    └── src/
```

---

## The Parent / Aggregator POM

The root POM must use `<packaging>pom</packaging>` and list all child modules:

```xml
<project>
    <groupId>com.example</groupId>
    <artifactId>my-project</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>   <!-- REQUIRED for parent/aggregator -->

    <modules>
        <module>my-core</module>
        <module>my-service</module>
        <module>my-web</module>
    </modules>

    <!-- Shared dependency versions for all children -->
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.slf4j</groupId>
                <artifactId>slf4j-api</artifactId>
                <version>2.0.9</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <!-- Shared plugin config for all children -->
    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.11.0</version>
                    <configuration>
                        <source>17</source>
                        <target>17</target>
                    </configuration>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
```

---

## Child Module POM

```xml
<project>
    <!-- Inherit from parent -->
    <parent>
        <groupId>com.example</groupId>
        <artifactId>my-project</artifactId>
        <version>1.0.0-SNAPSHOT</version>
        <!-- Default: looks for ../pom.xml. Override if needed: -->
        <relativePath>../pom.xml</relativePath>
    </parent>

    <!-- Only artifactId is needed — groupId and version inherited -->
    <artifactId>my-service</artifactId>

    <dependencies>
        <!-- Depend on a sibling module -->
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>my-core</artifactId>
            <version>${project.version}</version>
        </dependency>

        <!-- Version omitted — managed by parent -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
        </dependency>
    </dependencies>
</project>
```

---

## Aggregation vs Inheritance

These are two distinct (though often combined) concepts:

| Concept | Mechanism | Purpose |
|---|---|---|
| **Aggregation** | `<modules>` in parent | Build all child modules together as a group |
| **Inheritance** | `<parent>` in child | Share configuration, dependencies, plugins |

A POM can do both (the most common pattern), or either one independently.

---

## Reactor Build Order

Maven's reactor calculates the correct build order automatically based on inter-module dependencies. If `my-service` depends on `my-core`, the reactor always builds `my-core` first.

```bash
mvn install
# [INFO] Reactor Build Order:
# [INFO]   my-project                                                        [pom]
# [INFO]   my-core                                                           [jar]
# [INFO]   my-service                                                        [jar]
# [INFO]   my-web                                                            [war]
```

---

## Selective Module Builds

```bash
# Build only one module (and its dependencies in the reactor)
mvn install -pl my-service -am

# Build a module and everything that depends on it
mvn install -pl my-core -amd

# Exclude a module from the build
mvn install -pl '!my-web'

# Resume a failed build from a specific module
mvn install -rf my-service
```

| Flag | Meaning |
|---|---|
| `-pl` | Project list — specify which module(s) to build |
| `-am` | Also make — build dependencies of `-pl` modules |
| `-amd` | Also make dependents — build modules that depend on `-pl` modules |
| `-rf` | Resume from — restart a failed build from a specific module |

---

| | |
|---|---|
| [← Chapter 9: Profiles](./09-profiles.md) | [Next → Chapter 11: Use Cases](./11-use-cases.md) |
