[← Chapter 8: Plugins](./08-plugins.md) | [Index](../index.md)

---

# Chapter 9 — Profiles

Profiles allow you to conditionally activate different build configurations based on the environment, operating system, JDK version, or custom properties. A profile can override properties, add dependencies, and change plugin configuration.

---

## Declaring a Profile

```xml
<profiles>
    <profile>
        <id>production</id>
        <properties>
            <log.level>WARN</log.level>
            <db.url>jdbc:postgresql://prod-db:5432/app</db.url>
        </properties>
        <dependencies>
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>prod-monitoring</artifactId>
                <version>1.0.0</version>
            </dependency>
        </dependencies>
    </profile>
</profiles>
```

---

## Profile Activation Strategies

### 1. Explicit — command line flag

```bash
mvn install -Pproduction
mvn install -Pproduction,integration   # activate multiple profiles
mvn install -P!development             # deactivate a profile
```

### 2. Property value

```bash
mvn install -Denv=prod
```

```xml
<activation>
    <property>
        <n>env</n>
        <value>prod</value>
    </property>
</activation>
```

### 3. Property existence (any value)

```xml
<activation>
    <property>
        <n>performRelease</n>    <!-- activated if -DperformRelease is present -->
    </property>
</activation>
```

### 4. Operating System

```xml
<activation>
    <os>
        <family>windows</family>    <!-- windows | unix | mac -->
        <name>Windows 11</name>     <!-- exact os.name match -->
        <arch>amd64</arch>
    </os>
</activation>
```

### 5. JDK Version

```xml
<activation>
    <jdk>[17,21)</jdk>    <!-- activated for JDK 17, 18, 19, 20 (not 21) -->
</activation>

<activation>
    <jdk>17</jdk>         <!-- activated for any JDK 17.x -->
</activation>
```

### 6. File Existence / Absence

```xml
<activation>
    <file>
        <exists>integration-env.properties</exists>
    </file>
</activation>

<activation>
    <file>
        <missing>skip-deploy.flag</missing>
    </file>
</activation>
```

### 7. Default Activation

```xml
<activation>
    <activeByDefault>true</activeByDefault>
</activation>
```

A profile with `activeByDefault` is activated unless any other profile in the same POM is explicitly activated.

---

## Listing Active Profiles

```bash
mvn help:active-profiles
mvn help:active-profiles -Pproduction
```

---

## Profile Location

Profiles can be declared in three places:

| Location | Scope |
|---|---|
| `pom.xml` | Project-specific; committed to source control |
| `~/.m2/settings.xml` | User-specific; not committed |
| `$M2_HOME/conf/settings.xml` | Global for all users on the machine |

---

| | |
|---|---|
| [← Chapter 8: Plugins](./08-plugins.md) | [Next → Chapter 10: Multi-Module Projects](./10-multi-module.md) |
