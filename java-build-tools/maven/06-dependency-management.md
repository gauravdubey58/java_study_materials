[← Chapter 5: Phases & Goals](./05-phases-and-goals.md) | [Index](../index.md)

---

# Chapter 6 — Dependency Management

Automatic dependency management is one of Maven's most powerful features. You declare what your project needs, and Maven fetches everything — including all transitive dependencies — from remote repositories.

---

## Declaring a Dependency

Every dependency requires at minimum: `groupId`, `artifactId`, and `version` (version may be omitted if managed by a parent or BOM).

```xml
<dependencies>
    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-lang3</artifactId>
        <version>3.12.0</version>
    </dependency>
</dependencies>
```

---

## Dependency Scopes

**Scope** controls when a dependency is available and whether it is included in the final packaged artifact.

### Scope Reference Table

| Scope | Compile CP | Test CP | Runtime CP | Packaged | Description |
|---|---|---|---|---|---|
| `compile` | ✅ | ✅ | ✅ | ✅ | Default. Needed everywhere. |
| `provided` | ✅ | ✅ | ❌ | ❌ | Compile-time only. Container provides at runtime. |
| `runtime` | ❌ | ✅ | ✅ | ✅ | Not needed to compile but needed to run. |
| `test` | ❌ | ✅ | ❌ | ❌ | Only for tests. Never packaged. |
| `system` | ✅ | ✅ | ❌ | ❌ | Like `provided` but path is specified manually. Avoid. |
| `import` | N/A | N/A | N/A | N/A | Only in `<dependencyManagement>`. Imports a BOM. |

### scope: compile (default)

Used for libraries your main code compiles against and ships with:
```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>32.1.3-jre</version>
    <!-- scope defaults to compile -->
</dependency>
```

### scope: provided

The dependency is needed to compile the code, but the runtime environment (e.g., Tomcat, Kubernetes) will supply it. It is NOT included in the final WAR or JAR:
```xml
<dependency>
    <groupId>jakarta.servlet</groupId>
    <artifactId>jakarta.servlet-api</artifactId>
    <version>6.0.0</version>
    <scope>provided</scope>
</dependency>
```

### scope: runtime

Not needed to compile (no import statements reference it), but needed at runtime. Common for JDBC drivers and logging implementations:
```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.6.0</version>
    <scope>runtime</scope>
</dependency>
```

### scope: test

Only available during `test-compile` and `test` phases. Not included in any artifact. The most commonly used test-scope libraries:
```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.1</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.7.0</version>
    <scope>test</scope>
</dependency>
```

---

## Transitive Dependencies

If your project depends on Library A, and Library A depends on Library B, Maven automatically resolves and downloads Library B. You don't need to list it yourself.

```
Your Project
  ├── spring-boot-starter-web (declared)
  │     ├── spring-webmvc (transitive)
  │     │     ├── spring-core (transitive)
  │     │     └── spring-beans (transitive)
  │     ├── jackson-databind (transitive)
  │     │     ├── jackson-core (transitive)
  │     │     └── jackson-annotations (transitive)
  │     └── tomcat-embed-core (transitive)
  └── spring-boot-starter-test (declared)
        ├── junit-jupiter (transitive)
        └── mockito-core (transitive)
```

Visualise this for your own project:
```bash
mvn dependency:tree
mvn dependency:tree -Dverbose    # include omitted/conflicting entries
```

---

## Version Conflict Resolution

When multiple paths in the dependency tree pull in different versions of the same library, Maven must choose one. The rules are:

### Rule 1: Nearest Wins

Maven picks the version that is **closest to the root** in the dependency graph (fewest hops):

```
Your Project (root)
  ├── A → requires commons-lang3:3.8     (2 hops from root)
  └── B → requires commons-lang3:3.12   (2 hops from root — same depth)
```

When depth is equal, the **first declaration** in the POM wins. Here A is declared first, so `3.8` wins.

### Rule 2: Explicit Declaration Always Wins

If you declare the dependency directly in your own POM, that version is always chosen — it is at depth 1 (nearest possible):

```xml
<!-- This overrides any transitive version -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.12.0</version>
</dependency>
```

---

## Excluding Transitive Dependencies

Sometimes a library pulls in a transitive dependency you don't want — for example, a library that uses `commons-logging` when you prefer SLF4J. Use `<exclusions>`:

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <exclusions>
        <exclusion>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

To exclude ALL transitive dependencies from an artifact (get only the JAR itself):
```xml
<exclusions>
    <exclusion>
        <groupId>*</groupId>
        <artifactId>*</artifactId>
    </exclusion>
</exclusions>
```

---

## Optional Dependencies

Mark a dependency as `<optional>true</optional>` when it is needed to compile your library but should NOT be automatically pulled in by downstream users:

```xml
<!-- In your library's POM -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.15.2</version>
    <optional>true</optional>
</dependency>
```

Projects that depend on your library will need to explicitly declare Jackson if they want to use the Jackson-related features of your library.

---

## Dependency Management (Version Locking)

`<dependencyManagement>` locks versions centrally without adding the dependency to the project. Child modules or other `<dependencies>` entries can then omit the version:

```xml
<!-- In parent POM -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>2.0.9</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

```xml
<!-- In child POM — version is inherited from parent's dependencyManagement -->
<dependencies>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <!-- no <version> needed -->
    </dependency>
</dependencies>
```

---

## Importing a BOM

A **Bill of Materials (BOM)** is a POM that contains a `<dependencyManagement>` section with a curated, tested set of versions. Import it using `scope=import`:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.2.3</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

After importing, you can use all Spring Boot managed dependencies without specifying versions:
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <!-- version comes from the BOM -->
    </dependency>
</dependencies>
```

---

## Analysing Dependencies

```bash
mvn dependency:tree                     # full tree
mvn dependency:tree -Dverbose           # show omitted duplicates and conflicts
mvn dependency:tree -Dincludes=log4j    # filter to specific group/artifact
mvn dependency:analyze                  # find unused declared / used undeclared deps
mvn dependency:list                     # flat list of resolved dependencies
```

---

| | |
|---|---|
| [← Chapter 5: Phases & Goals](./05-phases-and-goals.md) | [Next → Chapter 7: Repositories](./07-repositories.md) |
