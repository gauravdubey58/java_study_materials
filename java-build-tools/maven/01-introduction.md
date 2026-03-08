[← Index](../index.md)

---

# Chapter 1 — Introduction to Maven

## What Is Apache Maven?

Apache Maven is an open-source **build automation and project management tool** primarily used for Java projects. It was created by Jason van Zyl in 2002 while working on the Apache Turbine project, and donated to the Apache Software Foundation shortly after.

Before Maven, every Java project had its own ad-hoc build scripts — usually a mix of Ant files and shell scripts with no standard structure. If you joined a new project, you had to learn that project's unique build process from scratch. Maven was created to solve this problem by providing a universal, standardised way to build Java applications.

---

## The Core Philosophy — Convention over Configuration

Maven is built around a single guiding principle: **Convention over Configuration (CoC)**.

This means Maven provides sensible defaults for everything. If your project follows Maven's conventions, you need almost zero configuration to get a working build. You only configure things that deviate from the defaults.

The most visible convention is the **Standard Directory Layout**:

```
my-project/
├── pom.xml                          ← Project configuration
├── src/
│   ├── main/
│   │   ├── java/                    ← Application source code
│   │   └── resources/               ← Application resources (properties, XML, etc.)
│   └── test/
│       ├── java/                    ← Test source code
│       └── resources/               ← Test resources
└── target/                          ← All generated output (created by Maven)
    ├── classes/                     ← Compiled .class files
    ├── test-classes/                ← Compiled test .class files
    └── my-project-1.0.jar           ← Packaged artifact
```

If you follow this layout, Maven's compiler knows exactly where to find source files, where to put class files, and where to write the final JAR — no configuration needed.

---

## A Brief History

| Year | Milestone |
|---|---|
| 2002 | Jason van Zyl begins Maven development at Apache |
| 2003 | Maven 1.0 released |
| 2005 | Maven 2.0 released — major rewrite, the foundation of modern Maven |
| 2010 | Maven 3.0 released — improved performance and backwards compatibility |
| 2019 | Maven 3.6.x — current stable branch begins |
| 2024 | Maven 3.9.x — latest stable release |

---

## What Maven Does

Maven is more than a build tool. It manages the full software project lifecycle:

**Build** — Compiles source code, runs tests, and packages the application into a deployable artifact (JAR, WAR, EAR).

**Dependency Management** — Automatically downloads the libraries your project needs from remote repositories. You declare what you need; Maven fetches it and all of its transitive dependencies.

**Project Information** — Manages metadata about your project: name, version, description, developers, licenses, and issue tracking URLs.

**Reporting** — Generates HTML reports for test results, code coverage, static analysis, and Javadoc.

**Release Management** — Automates the process of tagging a release in version control, updating version numbers, and deploying to a repository.

---

## Installation

### Prerequisites
- Java Development Kit (JDK) 8 or higher
- `JAVA_HOME` environment variable set

### Steps

**macOS (Homebrew)**
```bash
brew install maven
```

**Linux (apt)**
```bash
sudo apt install maven
```

**Windows (Chocolatey)**
```bash
choco install maven
```

**Manual installation**
1. Download from https://maven.apache.org/download.cgi
2. Extract to a directory (e.g., `/opt/maven`)
3. Add `M2_HOME=/opt/maven` and `$M2_HOME/bin` to `PATH`

### Verify Installation
```bash
mvn --version
# Apache Maven 3.9.6
# Maven home: /opt/homebrew/Cellar/maven/3.9.6/libexec
# Java version: 17.0.9, vendor: Eclipse Adoptium
```

---

## The Maven Wrapper (mvnw)

The **Maven Wrapper** (`mvnw` / `mvnw.cmd`) is a script bundled with a project that automatically downloads and uses a specific Maven version. This eliminates "works on my machine" issues caused by different Maven versions.

```bash
# Add wrapper to a project
mvn wrapper:wrapper

# Use wrapper instead of system Maven
./mvnw clean install        # Linux / macOS
mvnw.cmd clean install      # Windows
```

The wrapper is configured in `.mvn/wrapper/maven-wrapper.properties` and should be committed to version control.

---

## Maven vs Ant vs Gradle

| Concern | Maven | Ant | Gradle |
|---|---|---|---|
| Configuration | XML (pom.xml) | XML (build.xml) | Groovy/Kotlin DSL |
| Convention | Strong (CoC) | None | Moderate |
| Dependency management | Built-in | Manual (Ivy) | Built-in |
| Standard lifecycle | Yes | No | No |
| Build speed | Moderate | Fast | Fast (daemon + cache) |
| Learning curve | Moderate | Low | High |
| Best for | Standard Java/Java EE | Legacy/custom pipelines | Modern Java, Android |

---

## Next Up

The next chapter dives into every core term and concept you need to understand before working with Maven.

---

| | |
|---|---|
| [← Index](../index.md) | [Next → Chapter 2: Core Concepts](./02-core-concepts.md) |
