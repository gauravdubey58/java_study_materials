[← Chapter 1: Introduction](./01-introduction.md) | [Index](../index.md)

---

# Chapter 2 — Core Concepts & Terminology

This chapter defines every fundamental term you will encounter when working with Maven. Understanding these concepts clearly is the foundation for everything else.

---

## Artifact

An **artifact** is any file produced or consumed by Maven — most commonly a JAR, WAR, EAR, or POM file. Every artifact in the Maven ecosystem is uniquely identified by a set of coordinates.

---

## Coordinates (GAV)

**Coordinates** are the three-part address that uniquely identifies any artifact in the Maven world:

```
groupId : artifactId : version
com.google.guava : guava : 32.1.3-jre
```

This notation is often abbreviated as **GAV** (Group, Artifact, Version).

### groupId
Uniquely identifies the organisation or project group. By convention, it uses the **reverse domain name** pattern:
- `org.springframework` → springframework.org
- `com.fasterxml.jackson` → jackson project at fasterxml.com
- `io.netty` → netty.io

The groupId maps directly to a directory path in the repository:
`org/springframework/spring-core/`

### artifactId
The name of the specific artifact within the group. It must be unique within its groupId:
- `spring-core`, `spring-web`, `spring-context` (all under `org.springframework`)
- `guava`, `gson` (both under `com.google`)

### version
Identifies the specific release. Maven recognises several version patterns:

| Pattern | Example | Meaning |
|---|---|---|
| Semantic version | `3.12.0` | Stable release |
| SNAPSHOT | `3.12.0-SNAPSHOT` | In-development build |
| Qualifier | `3.12.0-beta-1` | Pre-release |
| RELEASE | `RELEASE` | Latest non-snapshot (deprecated) |

---

## SNAPSHOT vs Release

A **SNAPSHOT** version (e.g., `1.5.0-SNAPSHOT`) signals that this is a moving target — the artifact is still being developed and may change with each download. Maven will re-check the remote repository for a newer SNAPSHOT version on every build (or according to the `updatePolicy`).

A **Release** version (e.g., `1.5.0`) is immutable. Once published to a release repository, it never changes. Maven caches it locally forever.

---

## POM (Project Object Model)

The **POM** is the fundamental unit of Maven configuration. It is the `pom.xml` file at the root of every Maven project. The POM describes:
- Project coordinates and metadata
- Dependencies the project needs
- Plugins that perform build tasks
- Build configuration (source directories, output directories, etc.)
- Profiles for environment-specific configuration

See [Chapter 3 — The POM](./03-pom.md) for the full anatomy.

---

## Super POM

The **Super POM** is the implicit parent of every Maven project. All projects inherit from it, even if no `<parent>` is declared. It defines:
- The default Central repository location
- Default plugin versions bound to lifecycle phases
- Default directory structure paths (`src/main/java`, `target/`, etc.)
- Default resource filtering configuration

To view what the Super POM contributes to your project:
```bash
mvn help:effective-pom
```

---

## Effective POM

The **Effective POM** is what Maven actually uses during a build. It is the fully merged result of:
1. The Super POM (base defaults)
2. All ancestor POMs (parent chain)
3. The project's own `pom.xml`

```bash
mvn help:effective-pom          # print effective POM to console
mvn help:effective-pom -Doutput=effective-pom.xml   # write to file
```

---

## Mojo

A **Mojo** stands for **M**aven **O**ld **J**ava **O**bject. It is the Java class implementation of a single Maven plugin goal. Each Mojo:
- Extends `AbstractMojo` from the Maven Plugin API
- Is annotated with `@Mojo` to declare its goal name and lifecycle binding
- Implements the `execute()` method that does the actual work

As a Maven user, you rarely write Mojos — you consume them through plugins.

---

## Goal

A **goal** is the smallest unit of work in Maven — a single, specific task. Goals are provided by plugins:

```
plugin-prefix:goal-name

compiler:compile        → the 'compile' goal of the maven-compiler-plugin
surefire:test           → the 'test' goal of the maven-surefire-plugin
jar:jar                 → the 'jar' goal of the maven-jar-plugin
dependency:tree         → the 'tree' goal of the maven-dependency-plugin
```

Goals can be run directly on the command line or bound to lifecycle phases.

---

## Phase

A **phase** is a named step in a Maven lifecycle (e.g., `compile`, `test`, `package`). Phases are ordered — running a phase automatically runs all preceding phases in the lifecycle.

```bash
mvn package    # runs: validate → initialize → ... → compile → test → package
```

See [Chapter 4 — Lifecycles](./04-lifecycles.md) for all phases in order.

---

## Lifecycle

A **lifecycle** is a named, ordered sequence of phases that defines a complete build process. Maven has three built-in lifecycles:
- `default` — the primary build lifecycle (compile → test → package → install → deploy)
- `clean` — removes build output
- `site` — generates project documentation

---

## Plugin

A **plugin** is a collection of related goals packaged together as a JAR file. Maven itself does almost nothing — all actual build work is done by plugins.

Plugins are identified by coordinates, just like dependencies:
```xml
<groupId>org.apache.maven.plugins</groupId>
<artifactId>maven-compiler-plugin</artifactId>
<version>3.11.0</version>
```

---

## Repository

A **repository** is a storage location for Maven artifacts. Maven checks repositories in this order:
1. **Local repository** — your machine (`~/.m2/repository`)
2. **Remote repositories** — declared in the POM or `settings.xml`
3. **Central** — the default public repository

See [Chapter 7 — Repositories](./07-repositories.md) for full details.

---

## Dependency

A **dependency** is an external library or module your project needs to compile, test, or run. You declare dependencies in the `<dependencies>` section of the POM, and Maven downloads them automatically.

```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>2.0.9</version>
</dependency>
```

---

## Transitive Dependency

If your project depends on Library A, and Library A depends on Library B, then **Library B is a transitive dependency** of your project. Maven resolves and downloads transitive dependencies automatically — you don't need to list them yourself.

```
Your Project
  └── spring-web (direct dependency)
        ├── spring-core (transitive)
        ├── spring-beans (transitive)
        └── jackson-databind (transitive)
              └── jackson-core (transitive of transitive)
```

---

## Dependency Mediation

When two paths in the dependency tree require different versions of the same library, Maven applies **dependency mediation** using the **nearest-wins** rule: the version closest to the root project in the dependency graph wins.

```
Your Project
  ├── library-A → requires commons-lang3:3.10
  └── library-B → requires commons-lang3:3.12   ← WINS (same depth, first declaration wins)
```

To override any version, declare it explicitly in your own POM — that is always "nearest".

---

## BOM (Bill of Materials)

A **BOM** is a special POM that defines a curated set of dependency versions in its `<dependencyManagement>` section. Projects import a BOM to get consistent, compatible versions without specifying versions individually.

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

After importing this BOM, you can declare Spring dependencies without version numbers — they are all controlled by the BOM.

---

## Classifier

A **classifier** is an optional suffix on an artifact's coordinates that distinguishes multiple artifacts with the same GAV. Common classifiers:

| Classifier | Purpose | Example filename |
|---|---|---|
| `sources` | Java source code JAR | `guava-32.0-sources.jar` |
| `javadoc` | API documentation JAR | `guava-32.0-javadoc.jar` |
| `tests` | Test classes JAR | `my-lib-1.0-tests.jar` |
| `jdk11` | JDK-specific build | `asm-9.0-jdk11.jar` |

```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>32.0.0-jre</version>
    <classifier>sources</classifier>
</dependency>
```

---

## Packaging

**Packaging** defines the type of artifact Maven produces. It is declared in the POM and defaults to `jar`.

| Packaging | Description |
|---|---|
| `jar` | Standard Java archive (default) |
| `war` | Web Application Archive |
| `ear` | Enterprise Application Archive |
| `pom` | No artifact; used for parent/aggregator POMs |
| `maven-plugin` | Maven plugin project |
| `ejb` | Enterprise JavaBean archive |

The packaging type also determines which goals are bound to which lifecycle phases by default.

---

## Archetype

An **archetype** is a project template that generates a complete Maven project skeleton. Running `mvn archetype:generate` presents a list of hundreds of templates for different project types (Spring Boot app, REST service, multi-module project, etc.).

```bash
mvn archetype:generate \
  -DgroupId=com.example \
  -DartifactId=my-app \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DinteractiveMode=false
```

---

| | |
|---|---|
| [← Chapter 1: Introduction](./01-introduction.md) | [Next → Chapter 3: The POM](./03-pom.md) |
