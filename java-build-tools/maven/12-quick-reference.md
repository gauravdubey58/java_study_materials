[← Chapter 11: Use Cases](./11-use-cases.md) | [Index](../index.md)

---

# Chapter 12 — Quick Reference

A condensed cheat sheet of the most-used Maven commands, flags, and patterns. Bookmark this page.

---

## Core Commands

```bash
# Lifecycle phases (each runs all preceding phases too)
mvn validate
mvn compile
mvn test
mvn package
mvn verify
mvn install
mvn deploy

# Combined
mvn clean install              # most common — full clean build
mvn clean package              # build artifact without installing
mvn clean verify               # full build including integration tests
mvn clean deploy               # full CI/CD pipeline

# Clean only
mvn clean
```

---

## Test Commands

```bash
mvn test                                         # run all tests
mvn test -Dtest=UserServiceTest                  # run one test class
mvn test -Dtest=UserServiceTest#myMethod         # run one method
mvn test -Dtest="*ServiceTest"                   # wildcard
mvn package -DskipTests                          # skip test execution
mvn package -Dmaven.test.skip=true               # skip compile + execution
mvn verify                                       # also run integration tests
```

---

## Dependency Commands

```bash
mvn dependency:tree                     # full dependency tree
mvn dependency:tree -Dverbose           # show conflict resolution
mvn dependency:tree -Dincludes=log4j    # filter by artifact
mvn dependency:analyze                  # unused / undeclared deps
mvn dependency:list                     # flat sorted dependency list
mvn dependency:copy-dependencies        # download all JARs locally
mvn dependency:go-offline               # pre-cache for offline builds
```

---

## Versions Commands

```bash
mvn versions:display-dependency-updates   # show available updates
mvn versions:display-plugin-updates       # show available plugin updates
mvn versions:use-latest-releases          # update all to latest release
mvn versions:set -DnewVersion=2.0.0       # set project version
mvn versions:revert                        # undo POM version changes
```

---

## Project Information

```bash
mvn help:effective-pom                         # merged POM Maven actually uses
mvn help:effective-settings                   # merged settings.xml
mvn help:active-profiles                      # active profiles for this build
mvn help:describe -Dplugin=compiler           # describe all goals of plugin
mvn help:describe -Dplugin=compiler -Dgoal=compile -Ddetail  # full goal detail
mvn help:system                               # all system properties
```

---

## Multi-Module

```bash
mvn install -pl my-module                     # build one module only
mvn install -pl my-module -am                 # build module + its dependencies
mvn install -pl my-module -amd                # build module + its dependents
mvn install -pl '!my-web'                     # exclude a module
mvn install -rf my-module                     # resume from module after failure
mvn install -T 4                              # 4 parallel threads
mvn install -T 1C                             # 1 thread per CPU core
```

---

## Release

```bash
mvn release:prepare                            # bump version, tag SCM
mvn release:perform                            # deploy tagged release
mvn release:rollback                           # undo failed prepare
mvn release:clean                              # remove temp files
```

---

## Debug & Troubleshooting

```bash
mvn install -X                                 # full debug output
mvn install -e                                 # show stack traces
mvn install -q                                 # quiet mode (errors only)
mvn install -V                                 # print Maven version at start
mvn install -o                                 # offline mode
mvn install -U                                 # force update SNAPSHOTs
mvn install -s /path/settings.xml             # use custom settings file
mvn install -Dmaven.repo.local=/tmp/repo       # use custom local repo
```

---

## Useful Flags Summary

| Flag | Meaning |
|---|---|
| `-P profile-id` | Activate a profile |
| `-D key=value` | Set a system property |
| `-pl module` | Limit to specific module(s) |
| `-am` | Also build dependencies |
| `-amd` | Also build dependents |
| `-rf module` | Resume from module |
| `-T N` | Use N threads |
| `-o` | Offline mode |
| `-U` | Force SNAPSHOT update |
| `-q` | Quiet output |
| `-X` | Debug output |
| `-e` | Show exceptions |
| `-V` | Show version |
| `--batch-mode` | Non-interactive (CI) |
| `--no-transfer-progress` | Hide download progress |
| `-s file` | Custom settings.xml |
| `-f file` | Custom POM file |

---

## Common POM Snippets

```xml
<!-- Skip tests in a specific plugin execution -->
<configuration>
    <skipTests>true</skipTests>
</configuration>

<!-- Force Java version -->
<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
</properties>

<!-- Use UTF-8 everywhere -->
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
</properties>
```

---

*Official Maven docs: https://maven.apache.org/guides/*
*Plugin index: https://maven.apache.org/plugins/*
*Dependency search: https://search.maven.org*

---

| | |
|---|---|
| [← Chapter 11: Use Cases](./11-use-cases.md) | [↑ Back to Index](../index.md) |
