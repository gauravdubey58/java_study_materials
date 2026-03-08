[← Chapter 4: Lifecycles](./04-lifecycles.md) | [Index](../index.md)

---

# Chapter 5 — Phases & Goals

Chapter 4 explained what lifecycles and phases *are*. This chapter explains how to work with them practically — how to run phases and goals from the command line, how goals bind to phases, and how to call goals directly without going through the lifecycle.

---

## Phases vs Goals — The Key Distinction

| Concept | What it is | Example |
|---|---|---|
| **Phase** | A named step in a lifecycle | `compile`, `test`, `package` |
| **Goal** | A specific task provided by a plugin | `compiler:compile`, `surefire:test` |

Running a **phase** triggers the lifecycle up to and including that phase, executing all goals bound to each phase along the way.

Running a **goal** directly executes only that one task, bypassing the lifecycle entirely.

---

## Running Phases

```bash
# Each command runs all lifecycle phases up to and including the named one
mvn validate
mvn compile
mvn test
mvn package
mvn verify
mvn install
mvn deploy

# Combine lifecycles and phases in one command
mvn clean package           # clean lifecycle + default up to package
mvn clean verify            # clean lifecycle + default up to verify
mvn clean install           # most common — used in CI/CD
mvn clean deploy            # full production pipeline
```

### What Happens When You Run mvn package

```
mvn package
  Runs in order:
  [1]  validate             → check POM is valid
  [2]  initialize           → set up build state
  [3]  generate-sources     → (no goals bound by default)
  [4]  process-sources      → (no goals bound by default)
  [5]  generate-resources   → (no goals bound by default)
  [6]  process-resources    → resources:resources (copies + filters resources)
  [7]  compile              → compiler:compile (compiles .java files)
  [8]  process-classes      → (no goals bound by default)
  [9]  generate-test-sources → (no goals bound by default)
  [10] process-test-sources → (no goals bound by default)
  [11] generate-test-resources → (no goals bound by default)
  [12] process-test-resources → resources:testResources
  [13] test-compile         → compiler:testCompile
  [14] process-test-classes → (no goals bound by default)
  [15] test                 → surefire:test (runs unit tests)
  [16] prepare-package      → (no goals bound by default)
  [17] package              → jar:jar (creates the JAR)
  ── DONE ──
```

---

## Running Goals Directly

Plugin goals are written in the format `pluginPrefix:goalName` or the full form `groupId:artifactId:version:goalName`.

```bash
# Short form (using plugin prefix — works for well-known plugins)
mvn compiler:compile
mvn surefire:test
mvn jar:jar
mvn dependency:tree
mvn help:effective-pom

# Full form (always works, even for unknown plugins)
mvn org.apache.maven.plugins:maven-compiler-plugin:3.11.0:compile
```

### Useful Goals to Know

```bash
# Dependency inspection
mvn dependency:tree                      # show full dependency tree
mvn dependency:tree -Dverbose            # include conflict resolution details
mvn dependency:tree -Dincludes=log4j     # filter tree to specific artifact
mvn dependency:analyze                   # find unused / undeclared dependencies
mvn dependency:resolve                   # resolve and list all dependencies
mvn dependency:copy-dependencies         # copy all JARs to target/dependency/

# Project information
mvn help:effective-pom                   # show the merged effective POM
mvn help:effective-settings             # show effective settings.xml
mvn help:describe -Dplugin=compiler     # describe all goals of a plugin
mvn help:describe -Dplugin=compiler -Dgoal=compile -Ddetail   # full detail

# Version management
mvn versions:display-dependency-updates  # show newer available dependency versions
mvn versions:display-plugin-updates      # show newer available plugin versions
mvn versions:use-latest-releases        # auto-update all dependencies

# Archetype (project generation)
mvn archetype:generate                   # interactive project generator
```

---

## Binding a Goal to a Phase

You can bind any plugin goal to any lifecycle phase by configuring it in the POM:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-antrun-plugin</artifactId>
            <version>3.1.0</version>
            <executions>
                <execution>
                    <id>print-build-start</id>
                    <phase>validate</phase>          <!-- attach to validate phase -->
                    <goals>
                        <goal>run</goal>             <!-- run this goal when phase executes -->
                    </goals>
                    <configuration>
                        <target>
                            <echo message="Build started for ${project.artifactId}!" />
                        </target>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

Multiple executions of the same plugin, each bound to a different phase, are possible by using different `<id>` values:

```xml
<executions>
    <execution>
        <id>start-server</id>
        <phase>pre-integration-test</phase>
        <goals><goal>start</goal></goals>
    </execution>
    <execution>
        <id>stop-server</id>
        <phase>post-integration-test</phase>
        <goals><goal>stop</goal></goals>
    </execution>
</executions>
```

---

## Default Phase Bindings by Packaging

The goals that Maven binds automatically differ depending on your project's `<packaging>` type:

### jar packaging (default)

| Phase | Bound Goal |
|---|---|
| `process-resources` | `resources:resources` |
| `compile` | `compiler:compile` |
| `process-test-resources` | `resources:testResources` |
| `test-compile` | `compiler:testCompile` |
| `test` | `surefire:test` |
| `package` | `jar:jar` |
| `install` | `install:install` |
| `deploy` | `deploy:deploy` |

### war packaging

Same as `jar`, except:
- `package` → `war:war` (instead of `jar:jar`)

### pom packaging (parent/aggregator)

| Phase | Bound Goal |
|---|---|
| `install` | `install:install` |
| `deploy` | `deploy:deploy` |

Note: `pom` packaging does not bind `compile`, `test`, or `package` — there is no source code to compile or package.

---

## Multi-Goal Execution

You can execute multiple goals or phases in a single Maven invocation — they run left to right:

```bash
mvn clean package dependency:tree     # clean, then package, then print dep tree
mvn compiler:compile surefire:test    # compile then test (no lifecycle)
```

---

## Skipping Phases

You cannot skip a phase in the middle of a lifecycle, but you can skip specific goals that are bound to phases:

```bash
mvn package -DskipTests                         # skip test execution (still compiles tests)
mvn package -Dmaven.test.skip=true              # skip test compilation AND execution
mvn package -Dmaven.javadoc.skip=true           # skip Javadoc generation
mvn package -Dcheckstyle.skip=true             # skip Checkstyle
mvn package -Dspotbugs.skip=true               # skip SpotBugs
```

---

## Parallel Builds

Maven can run independent module builds in parallel using the `-T` flag (requires Maven 3+):

```bash
mvn -T 4 install           # 4 threads
mvn -T 1C install          # 1 thread per CPU core
mvn -T 2C clean install    # 2 threads per CPU core
```

Parallel builds only work for multi-module projects where modules have no inter-dependencies.

---

| | |
|---|---|
| [← Chapter 4: Lifecycles](./04-lifecycles.md) | [Next → Chapter 6: Dependency Management](./06-dependency-management.md) |
