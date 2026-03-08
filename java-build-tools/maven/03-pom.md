[← Chapter 2: Core Concepts](./02-core-concepts.md) | [Index](../index.md)

---

# Chapter 3 — The Project Object Model (POM)

The `pom.xml` is the heart of every Maven project. It is the single source of truth for everything Maven needs to know: what your project is, what it depends on, how to build it, and where to deploy it.

---

## The Minimal POM

The absolute minimum `pom.xml` that Maven can work with:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>1.0.0</version>
</project>
```

Every other element is optional — Maven inherits all missing configuration from the Super POM.

---

## The Full POM — Annotated

Below is a comprehensive `pom.xml` with every major section explained:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <!-- Always 4.0.0 for Maven 2 and Maven 3 -->
    <modelVersion>4.0.0</modelVersion>

    <!-- ═══════════════════════════════════════════════
         SECTION 1: Project Identity
    ═══════════════════════════════════════════════ -->
    <groupId>com.example</groupId>
    <artifactId>my-application</artifactId>
    <version>2.1.0-SNAPSHOT</version>
    <packaging>jar</packaging>             <!-- jar | war | ear | pom -->

    <name>My Application</name>
    <description>A sample enterprise application</description>
    <url>https://github.com/example/my-application</url>
    <inceptionYear>2024</inceptionYear>

    <!-- ═══════════════════════════════════════════════
         SECTION 2: Parent / Inheritance
    ═══════════════════════════════════════════════ -->
    <!--
        Inheriting from a parent POM means this project gets all
        dependencyManagement, pluginManagement, and properties from the parent.
        Spring Boot's parent also configures sensible plugin defaults.
    -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.3</version>
        <relativePath />   <!-- look up from repository, not local filesystem -->
    </parent>

    <!-- ═══════════════════════════════════════════════
         SECTION 3: Properties
         Reusable variables. Reference with ${property.name}.
         Command-line: mvn install -Dproperty.name=value
    ═══════════════════════════════════════════════ -->
    <properties>
        <java.version>17</java.version>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>

        <!-- Custom properties for version alignment -->
        <jackson.version>2.15.2</jackson.version>
        <testcontainers.version>1.19.3</testcontainers.version>
    </properties>

    <!-- ═══════════════════════════════════════════════
         SECTION 4: Dependency Management
         Declares versions centrally. Does NOT add dependencies.
         Child modules can omit <version> for managed deps.
    ═══════════════════════════════════════════════ -->
    <dependencyManagement>
        <dependencies>
            <!-- Import a BOM — pulls in all its version definitions -->
            <dependency>
                <groupId>org.testcontainers</groupId>
                <artifactId>testcontainers-bom</artifactId>
                <version>${testcontainers.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <!-- ═══════════════════════════════════════════════
         SECTION 5: Dependencies
         These are actually added to the project's classpath.
    ═══════════════════════════════════════════════ -->
    <dependencies>

        <!-- compile scope (default): on all classpaths, included in artifact -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <!-- version omitted — inherited from parent BOM -->
        </dependency>

        <!-- provided: compile + test classpath, NOT packaged (container provides at runtime) -->
        <dependency>
            <groupId>jakarta.servlet</groupId>
            <artifactId>jakarta.servlet-api</artifactId>
            <version>6.0.0</version>
            <scope>provided</scope>
        </dependency>

        <!-- runtime: NOT needed to compile, but needed to run -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <version>42.6.0</version>
            <scope>runtime</scope>
        </dependency>

        <!-- test: only for test compilation and execution -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <!-- optional: not inherited by downstream projects -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>${jackson.version}</version>
            <optional>true</optional>
        </dependency>

        <!-- Dependency with exclusion of a transitive dep -->
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

    </dependencies>

    <!-- ═══════════════════════════════════════════════
         SECTION 6: Build Configuration
    ═══════════════════════════════════════════════ -->
    <build>
        <!-- Override default source / output directories if needed -->
        <sourceDirectory>src/main/java</sourceDirectory>
        <testSourceDirectory>src/test/java</testSourceDirectory>
        <outputDirectory>target/classes</outputDirectory>
        <testOutputDirectory>target/test-classes</testOutputDirectory>

        <!-- Name of the output artifact (without extension) -->
        <finalName>${project.artifactId}-${project.version}</finalName>

        <!-- Resources (non-Java files copied to output) -->
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>  <!-- replace ${variables} in resource files -->
            </resource>
        </resources>
        <testResources>
            <testResource>
                <directory>src/test/resources</directory>
            </testResource>
        </testResources>

        <!-- Plugin Management: declare versions without activating plugins -->
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.11.0</version>
                </plugin>
            </plugins>
        </pluginManagement>

        <!-- Plugins: these actually run during the build -->
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

    <!-- ═══════════════════════════════════════════════
         SECTION 7: Repositories
         Additional repositories beyond Maven Central.
    ═══════════════════════════════════════════════ -->
    <repositories>
        <repository>
            <id>spring-milestones</id>
            <url>https://repo.spring.io/milestone</url>
            <snapshots><enabled>false</enabled></snapshots>
        </repository>
    </repositories>

    <!-- ═══════════════════════════════════════════════
         SECTION 8: Distribution Management
         Where to deploy artifacts.
    ═══════════════════════════════════════════════ -->
    <distributionManagement>
        <repository>
            <id>nexus-releases</id>
            <url>https://nexus.company.com/repository/maven-releases/</url>
        </repository>
        <snapshotRepository>
            <id>nexus-snapshots</id>
            <url>https://nexus.company.com/repository/maven-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>

    <!-- ═══════════════════════════════════════════════
         SECTION 9: Profiles
         See Chapter 9 for full details.
    ═══════════════════════════════════════════════ -->
    <profiles>
        <profile>
            <id>production</id>
            <properties>
                <log.level>WARN</log.level>
            </properties>
        </profile>
    </profiles>

    <!-- ═══════════════════════════════════════════════
         SECTION 10: Project Information (optional metadata)
    ═══════════════════════════════════════════════ -->
    <licenses>
        <license>
            <name>Apache License 2.0</name>
            <url>https://www.apache.org/licenses/LICENSE-2.0</url>
        </license>
    </licenses>

    <developers>
        <developer>
            <id>jdoe</id>
            <name>Jane Doe</name>
            <email>jane.doe@example.com</email>
            <roles><role>Lead Developer</role></roles>
        </developer>
    </developers>

    <scm>
        <connection>scm:git:git://github.com/example/my-app.git</connection>
        <developerConnection>scm:git:ssh://github.com/example/my-app.git</developerConnection>
        <url>https://github.com/example/my-app</url>
    </scm>

</project>
```

---

## POM Inheritance

Every POM implicitly inherits from the **Super POM**. You can also explicitly declare a parent:

```xml
<parent>
    <groupId>com.example</groupId>
    <artifactId>company-parent</artifactId>
    <version>1.0.0</version>
    <relativePath>../company-parent/pom.xml</relativePath>
</parent>
```

### What is inherited from parent?
- `<groupId>` and `<version>` (if not declared in child)
- `<dependencies>`
- `<dependencyManagement>`
- `<pluginManagement>`
- `<build>` configuration
- `<properties>`
- `<repositories>`
- `<profiles>`

### What is NOT inherited?
- `<artifactId>` — must be unique per project
- `<name>` and `<description>`

---

## pluginManagement vs plugins

These two sections are commonly confused:

**`<pluginManagement>`** — Declares plugin configuration (versions, settings) that child modules can inherit. It does **not** execute the plugin.

**`<plugins>`** — Actually binds the plugin to the build. The plugin runs during the build.

Think of `<pluginManagement>` as a lookup table of "if any module uses this plugin, use these settings". Child modules then add the plugin to `<plugins>` without repeating the version or configuration.

---

## Built-in POM Properties

Maven provides several pre-defined properties you can use anywhere in the POM:

| Property | Value |
|---|---|
| `${project.groupId}` | The project's groupId |
| `${project.artifactId}` | The project's artifactId |
| `${project.version}` | The project's version |
| `${project.name}` | The project's name |
| `${project.basedir}` | The project root directory |
| `${project.build.directory}` | `target/` |
| `${project.build.outputDirectory}` | `target/classes` |
| `${project.build.sourceDirectory}` | `src/main/java` |
| `${settings.localRepository}` | Path to local Maven repo |
| `${java.version}` | JVM version |
| `${env.VARIABLE_NAME}` | OS environment variable |

---

| | |
|---|---|
| [← Chapter 2: Core Concepts](./02-core-concepts.md) | [Next → Chapter 4: Lifecycles](./04-lifecycles.md) |
