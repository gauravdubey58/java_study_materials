[← Chapter 7: Repositories](./07-repositories.md) | [Index](../index.md)

---

# Chapter 8 — Plugins

Maven itself does almost nothing — all real work is delegated to plugins. This chapter covers how plugins work and documents the most important core and ecosystem plugins.

---

## Plugin Anatomy

A plugin is a JAR file containing one or more **Mojos** (Maven Old Java Objects). Each Mojo implements one goal. Plugins are configured in the `<build><plugins>` section of the POM.

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.11.0</version>
            <configuration>
                <!-- plugin-specific settings -->
                <source>17</source>
                <target>17</target>
            </configuration>
            <executions>
                <!-- bind additional goals to lifecycle phases -->
                <execution>
                    <id>my-extra-compile</id>
                    <phase>generate-sources</phase>
                    <goals><goal>compile</goal></goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

---

## Core Apache Maven Plugins

---

### maven-compiler-plugin
**Purpose:** Compiles Java source files.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.11.0</version>
    <configuration>
        <source>17</source>              <!-- Java source compatibility -->
        <target>17</target>              <!-- Java bytecode target -->
        <encoding>UTF-8</encoding>
        <compilerArgs>
            <arg>-Xlint:all</arg>        <!-- enable all warnings -->
            <arg>-Werror</arg>           <!-- treat warnings as errors -->
        </compilerArgs>
        <annotationProcessorPaths>       <!-- annotation processors (e.g., Lombok) -->
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>1.18.30</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```
**Default goals:** `compiler:compile` (bound to `compile`), `compiler:testCompile` (bound to `test-compile`)

---

### maven-surefire-plugin
**Purpose:** Runs unit tests during the `test` phase. Supports JUnit 4, JUnit 5, and TestNG.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.2</version>
    <configuration>
        <!-- Run tests in parallel -->
        <parallel>classes</parallel>
        <threadCount>4</threadCount>
        <!-- Include / exclude patterns -->
        <includes>
            <include>**/*Test.java</include>
            <include>**/*Tests.java</include>
        </includes>
        <excludes>
            <exclude>**/*IntegrationTest.java</exclude>
        </excludes>
        <!-- JVM args for test process -->
        <argLine>-Xmx512m -Dfile.encoding=UTF-8</argLine>
        <!-- Show test output on failure -->
        <useFile>false</useFile>
    </configuration>
</plugin>
```
**Key goal:** `surefire:test`

---

### maven-failsafe-plugin
**Purpose:** Runs integration tests. Guarantees the `post-integration-test` phase always runs (teardown), even if tests fail.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <version>3.2.2</version>
    <executions>
        <execution>
            <goals>
                <goal>integration-test</goal>   <!-- runs at integration-test phase -->
                <goal>verify</goal>             <!-- reports results at verify phase -->
            </goals>
        </execution>
    </executions>
    <configuration>
        <includes>
            <include>**/*IT.java</include>
            <include>**/*IntegrationTest.java</include>
        </includes>
    </configuration>
</plugin>
```
**Key goals:** `failsafe:integration-test`, `failsafe:verify`

---

### maven-jar-plugin
**Purpose:** Creates the JAR artifact during the `package` phase.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>3.3.0</version>
    <configuration>
        <archive>
            <manifest>
                <mainClass>com.example.Application</mainClass>
                <addClasspath>true</addClasspath>
                <classpathPrefix>lib/</classpathPrefix>
            </manifest>
            <manifestEntries>
                <Built-By>${user.name}</Built-By>
                <Implementation-Version>${project.version}</Implementation-Version>
            </manifestEntries>
        </archive>
        <excludes>
            <exclude>**/*.properties</exclude>
        </excludes>
    </configuration>
</plugin>
```
**Key goals:** `jar:jar`, `jar:test-jar`

---

### maven-war-plugin
**Purpose:** Packages web applications as WAR files.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-war-plugin</artifactId>
    <version>3.4.0</version>
    <configuration>
        <warName>myapp</warName>
        <failOnMissingWebXml>false</failOnMissingWebXml>
        <webResources>
            <resource>
                <directory>src/main/webapp</directory>
                <filtering>true</filtering>
            </resource>
        </webResources>
    </configuration>
</plugin>
```
**Key goals:** `war:war`, `war:exploded`

---

### maven-clean-plugin
**Purpose:** Deletes the `target/` directory and any additional configured directories.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-clean-plugin</artifactId>
    <version>3.3.2</version>
    <configuration>
        <filesets>
            <fileset>
                <directory>generated-sources</directory>
            </fileset>
        </filesets>
    </configuration>
</plugin>
```
**Key goal:** `clean:clean`

---

### maven-resources-plugin
**Purpose:** Copies and optionally filters (performs token replacement in) resource files.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-resources-plugin</artifactId>
    <version>3.3.1</version>
    <configuration>
        <encoding>UTF-8</encoding>
        <delimiters>
            <delimiter>@</delimiter>   <!-- use @variable@ syntax instead of ${} -->
        </delimiters>
    </configuration>
</plugin>
```
**Key goals:** `resources:resources`, `resources:testResources`, `resources:copy-resources`

---

### maven-install-plugin
**Purpose:** Installs the project artifact into the local repository (`~/.m2/repository`).

**Key goals:** `install:install`, `install:install-file`

```bash
# Install a third-party JAR not in any repository
mvn install:install-file -Dfile=vendor.jar -DgroupId=com.vendor \
    -DartifactId=vendor-lib -Dversion=1.0 -Dpackaging=jar
```

---

### maven-deploy-plugin
**Purpose:** Deploys the artifact to a remote repository.

**Key goals:** `deploy:deploy`, `deploy:deploy-file`

Requires `<distributionManagement>` in the POM — see [Chapter 7](./07-repositories.md).

---

### maven-dependency-plugin
**Purpose:** Inspects, analyses, and manages project dependencies.

```bash
mvn dependency:tree                    # full transitive dependency tree
mvn dependency:tree -Dverbose          # include omitted conflicts
mvn dependency:analyze                 # find unused/missing declarations
mvn dependency:copy-dependencies       # copy all JARs to target/dependency/
mvn dependency:unpack-dependencies     # unpack all deps to a directory
```

---

### maven-assembly-plugin
**Purpose:** Creates distributable archives containing the project plus its dependencies.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <version>3.6.0</version>
    <configuration>
        <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
        </descriptorRefs>
        <archive>
            <manifest>
                <mainClass>com.example.Application</mainClass>
            </manifest>
        </archive>
    </configuration>
    <executions>
        <execution>
            <phase>package</phase>
            <goals><goal>single</goal></goals>
        </execution>
    </executions>
</plugin>
```
**Key goal:** `assembly:single`

---

### maven-shade-plugin
**Purpose:** Creates an uber-JAR (fat JAR) with all dependencies merged. Supports class **relocation** to avoid classpath conflicts.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>3.5.1</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals><goal>shade</goal></goals>
            <configuration>
                <createDependencyReducedPom>true</createDependencyReducedPom>
                <relocations>
                    <!-- rename package to avoid conflicts -->
                    <relocation>
                        <pattern>com.google.guava</pattern>
                        <shadedPattern>shaded.com.google.guava</shadedPattern>
                    </relocation>
                </relocations>
                <filters>
                    <filter>
                        <artifact>*:*</artifact>
                        <excludes>
                            <exclude>META-INF/*.SF</exclude>
                            <exclude>META-INF/*.DSA</exclude>
                        </excludes>
                    </filter>
                </filters>
            </configuration>
        </execution>
    </executions>
</plugin>
```

---

### maven-source-plugin
**Purpose:** Attaches a sources JAR to the build (published alongside the main JAR).

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-source-plugin</artifactId>
    <version>3.3.0</version>
    <executions>
        <execution>
            <id>attach-sources</id>
            <goals><goal>jar-no-fork</goal></goals>
        </execution>
    </executions>
</plugin>
```

---

### maven-javadoc-plugin
**Purpose:** Generates HTML Javadoc and attaches a Javadoc JAR.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-javadoc-plugin</artifactId>
    <version>3.6.2</version>
    <configuration>
        <doclint>none</doclint>           <!-- don't fail on Javadoc warnings -->
        <source>17</source>
        <show>protected</show>            <!-- document protected and public members -->
    </configuration>
    <executions>
        <execution>
            <id>attach-javadocs</id>
            <goals><goal>jar</goal></goals>
        </execution>
    </executions>
</plugin>
```

---

### maven-release-plugin
**Purpose:** Automates the full release process: version bump, SCM tag, deploy, and increment to next SNAPSHOT.

```bash
mvn release:prepare    # bump version, tag in Git, update to next SNAPSHOT
mvn release:perform    # checkout tag, build, deploy to remote repository
mvn release:rollback   # undo a failed prepare step
mvn release:clean      # clean up temp files from a failed release
```

---

### maven-enforcer-plugin
**Purpose:** Enforces constraints that must be satisfied for the build to succeed.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-enforcer-plugin</artifactId>
    <version>3.4.1</version>
    <executions>
        <execution>
            <id>enforce-standards</id>
            <goals><goal>enforce</goal></goals>
            <configuration>
                <rules>
                    <requireJavaVersion>
                        <version>[17,)</version>
                        <message>Java 17+ is required</message>
                    </requireJavaVersion>
                    <requireMavenVersion>
                        <version>[3.8,)</version>
                    </requireMavenVersion>
                    <bannedDependencies>
                        <excludes>
                            <exclude>commons-logging:commons-logging</exclude>
                            <exclude>log4j:log4j</exclude>
                        </excludes>
                    </bannedDependencies>
                    <requireNoRepositories>
                        <message>Use only the internal Nexus repository</message>
                    </requireNoRepositories>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

---

## Ecosystem Plugins

| Plugin | groupId / artifactId | Purpose |
|---|---|---|
| Spring Boot | `org.springframework.boot:spring-boot-maven-plugin` | Package as executable JAR/WAR, run Spring Boot app |
| JaCoCo | `org.jacoco:jacoco-maven-plugin` | Code coverage reports |
| Checkstyle | `org.apache.maven.plugins:maven-checkstyle-plugin` | Enforce coding style |
| SpotBugs | `com.github.spotbugs:spotbugs-maven-plugin` | Static bug analysis |
| PMD | `org.apache.maven.plugins:maven-pmd-plugin` | Source code analyser |
| Versions | `org.codehaus.mojo:versions-maven-plugin` | Manage and update version numbers |
| Exec | `org.codehaus.mojo:exec-maven-plugin` | Run Java programs or shell commands |
| Jib | `com.google.cloud.tools:jib-maven-plugin` | Build Docker images without Docker daemon |
| Flyway | `org.flywaydb:flyway-maven-plugin` | Database migrations |
| Liquibase | `org.liquibase:liquibase-maven-plugin` | Database migrations |
| Native | `org.graalvm.buildtools:native-maven-plugin` | GraalVM native image compilation |

---

| | |
|---|---|
| [← Chapter 7: Repositories](./07-repositories.md) | [Next → Chapter 9: Profiles](./09-profiles.md) |
