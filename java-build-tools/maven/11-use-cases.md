[← Chapter 10: Multi-Module Projects](./10-multi-module.md) | [Index](../index.md)

---

# Chapter 11 — Common Use Cases

Practical recipes for the most frequent Maven tasks you'll encounter on real projects.

---

## Testing

### Skip Tests
```bash
mvn package -DskipTests               # compile tests but don't run them
mvn package -Dmaven.test.skip=true    # don't even compile tests
```

### Run a Single Test Class
```bash
mvn test -Dtest=UserServiceTest
mvn test -Dtest=UserServiceTest#shouldCreateUser   # specific method
mvn test -Dtest="UserServiceTest,OrderServiceTest" # multiple classes
mvn test -Dtest="*ServiceTest"                     # wildcard
```

### Run Tests in Parallel
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <parallel>classes</parallel>
        <threadCount>4</threadCount>
        <forkCount>2</forkCount>
    </configuration>
</plugin>
```

### Integration Tests with Failsafe
```bash
mvn verify                          # runs both unit tests and integration tests
mvn failsafe:integration-test       # run only integration tests
```

---

## Dependency Diagnostics

```bash
mvn dependency:tree                          # full transitive tree
mvn dependency:tree -Dverbose                # include conflict details
mvn dependency:tree -Dincludes=org.slf4j     # filter by group
mvn dependency:analyze                       # find unused / undeclared deps
mvn dependency:list                          # flat sorted list
mvn dependency:copy-dependencies             # download all JARs to target/dependency/

# Check for available updates
mvn versions:display-dependency-updates
mvn versions:display-plugin-updates
```

---

## Building Docker Images

### Using Jib (no Docker daemon required)
```xml
<plugin>
    <groupId>com.google.cloud.tools</groupId>
    <artifactId>jib-maven-plugin</artifactId>
    <version>3.4.0</version>
    <configuration>
        <to>
            <image>gcr.io/my-project/my-app:${project.version}</image>
        </to>
    </configuration>
</plugin>
```

```bash
mvn compile jib:build           # push directly to registry
mvn compile jib:dockerBuild     # build to local Docker daemon
```

---

## Generating a New Project

```bash
# Interactive mode — Maven shows a list of archetypes
mvn archetype:generate

# Non-interactive — specify everything up front
mvn archetype:generate \
  -DgroupId=com.example \
  -DartifactId=my-service \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DarchetypeVersion=1.4 \
  -DinteractiveMode=false
```

---

## Offline Builds

```bash
mvn install -o                   # fail if any artifact is missing locally
mvn dependency:go-offline        # pre-download everything needed for offline use
```

---

## Build Debugging

```bash
mvn install -X                   # full debug output (very verbose)
mvn install -e                   # print exception stack traces on failure
mvn install -q                   # quiet — only errors
mvn install -V                   # show Maven version at start of build

# Check what profile is active
mvn help:active-profiles

# Check the effective POM (merged with parent + Super POM)
mvn help:effective-pom

# Check effective settings
mvn help:effective-settings

# Describe a plugin
mvn help:describe -Dplugin=surefire -Ddetail
```

---

## Releasing a Library

```bash
# Step 1: Check nothing is dirty
git status

# Step 2: Prepare the release (bumps version, creates Git tag, updates to next SNAPSHOT)
mvn release:prepare \
  -DreleaseVersion=2.0.0 \
  -DdevelopmentVersion=2.1.0-SNAPSHOT

# Step 3: Perform the release (checks out tag, builds, deploys to remote repo)
mvn release:perform

# If something goes wrong:
mvn release:rollback
mvn release:clean
```

---

## Updating Dependency Versions

```bash
# See what can be updated (does not change the POM)
mvn versions:display-dependency-updates

# Automatically update all dependencies to their latest release
mvn versions:use-latest-releases

# Set a specific version for one dependency
mvn versions:set-property -Dproperty=jackson.version -DnewVersion=2.15.3

# Set the project version
mvn versions:set -DnewVersion=3.0.0

# Revert uncommitted version changes
mvn versions:revert
```

---

## Resource Filtering

Replace `${variable}` placeholders in resource files with actual property values:

```xml
<!-- pom.xml -->
<properties>
    <app.version>${project.version}</app.version>
    <db.url>jdbc:h2:mem:testdb</db.url>
</properties>

<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>
        </resource>
    </resources>
</build>
```

```properties
# src/main/resources/application.properties  (before filtering)
app.version=@app.version@
db.url=${db.url}
```

After `mvn process-resources`, `target/classes/application.properties` contains the actual values.

---

## CI/CD Pipeline Command

A standard CI/CD pipeline command:

```bash
mvn clean verify \
  --batch-mode \
  --no-transfer-progress \
  -Dmaven.test.failure.ignore=false \
  -Dstyle.color=always
```

| Flag | Purpose |
|---|---|
| `--batch-mode` | Disable interactive prompts (required in CI) |
| `--no-transfer-progress` | Suppress download progress bars (cleaner logs) |
| `-Dmaven.test.failure.ignore=false` | Fail build immediately on test failure |

---

| | |
|---|---|
| [← Chapter 10: Multi-Module Projects](./10-multi-module.md) | [Next → Chapter 12: Quick Reference](./12-quick-reference.md) |
