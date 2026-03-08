[← Chapter 1: Introduction](./01-introduction.md) | [Index](../index.md)

---

# Chapter 2 — Core Concepts & Terminology

This chapter defines every fundamental term you will encounter in Ant. Mastering this vocabulary is the foundation for reading and writing Ant build files.

---

## Build File

The XML configuration file that defines the entire build process. By convention it is named **`build.xml`**, but any name is valid — you specify it with `-f`:

```bash
ant                          # uses build.xml in current directory
ant -f custom-build.xml      # uses a custom-named file
ant -f ../shared/build.xml   # uses a file in another directory
```

A project has exactly one active build file at a time (though it can `<import>` or call others).

---

## Project

The **root element** of every Ant build file. There is exactly one `<project>` per build file.

```xml
<project name="MyApp" default="package" basedir=".">
    ...
</project>
```

| Attribute | Required | Description |
|---|---|---|
| `name` | No | Display name of the project |
| `default` | No | Target to run when no target is specified on the command line |
| `basedir` | No | Root directory for resolving relative paths. Defaults to the directory of the build file |

---

## Target

A **named group of tasks** that performs a specific part of the build (e.g., compile, test, package). Targets can depend on other targets via the `depends` attribute.

```xml
<target name="compile"
        depends="init"
        description="Compile all Java source files"
        if="src.present"
        unless="already.compiled">
    <javac ... />
</target>
```

| Attribute | Description |
|---|---|
| `name` | Unique name (required) |
| `depends` | Comma-separated list of targets that must run first |
| `description` | Shown in `ant -projecthelp` output |
| `if` | Only run this target if the named property IS set |
| `unless` | Only run this target if the named property is NOT set |

---

## Task

The **smallest unit of work** in Ant. A task is an XML element that Ant executes — it performs one concrete action (create a directory, compile files, copy a file, etc.).

```xml
<mkdir dir="${build.dir}/classes" />
<javac srcdir="${src.dir}" destdir="${classes.dir}" />
<copy file="config.xml" todir="${dist.dir}" />
```

Ant ships with approximately 150 built-in tasks. More are available as optional tasks or third-party Antlibs.

---

## Built-in Task vs Optional Task

**Built-in tasks** are included with every Ant installation and work immediately. Examples: `<mkdir>`, `<copy>`, `<delete>`, `<javac>`, `<jar>`, `<echo>`, `<exec>`.

**Optional tasks** ship with Ant but require additional JAR files to be placed in `$ANT_HOME/lib`. Examples: `<junit>`, `<ftp>`, `<scp>`, `<telnet>`. The required JARs are listed in Ant's documentation.

---

## Antlib

An "Ant Library" — a packaged collection of custom tasks distributed as a JAR with an `antlib.xml` descriptor inside. Antlibs are the Ant equivalent of Maven plugins. Examples: Apache Ivy, JaCoCo's Ant tasks, Ant-Contrib.

```xml
<!-- Load an Antlib via namespace -->
<project xmlns:ivy="antlib:org.apache.ivy.ant">
    ...
    <ivy:retrieve />
</project>
```

---

## Property

A **key-value pair** used as a variable throughout the build file. The single most important rule about Ant properties:

> **Properties are immutable — the first definition wins.**

This immutability enables command-line overrides: properties set with `-D` on the command line are set before the build file is read, so they take precedence over anything declared in the file.

```xml
<property name="debug"        value="true" />
<property name="build.dir"    value="build" />
<property name="lib.dir"      location="lib" />    <!-- 'location' = absolute path -->
<property file="build.properties" />               <!-- load from file -->
<property environment="env" />                     <!-- env.PATH, env.JAVA_HOME, etc. -->
```

---

## Built-in Properties

Ant pre-defines these properties automatically:

| Property | Value |
|---|---|
| `ant.version` | Ant version string |
| `ant.home` | Ant installation directory |
| `ant.file` | Absolute path of the current build file |
| `ant.project.name` | Value of the `name` attribute on `<project>` |
| `basedir` | Absolute path of the project base directory |
| `java.version` | JVM version (from system) |
| `java.home` | JVM installation directory |
| `os.name` | Operating system name |
| `os.family` | OS family (`windows`, `unix`, `mac`) |
| `user.name` | Current OS username |
| `user.home` | Current user's home directory |

---

## FileSet

A **collection of files** selected by include/exclude patterns within a base directory. FileSets are the primary way to specify groups of files in Ant tasks.

```xml
<fileset id="java.sources" dir="${src.dir}">
    <include name="**/*.java" />
    <exclude name="**/generated/**" />
    <exclude name="**/*Test.java" />
</fileset>
```

The `**` pattern matches any number of directory levels. `*` matches within a single directory.

---

## Path and Classpath

An **ordered list of filesystem locations** (directories and JAR files) used to define classpaths for compilation and execution. Paths can be defined once with an `id` and reused by reference.

```xml
<path id="compile.classpath">
    <fileset dir="${lib.dir}" includes="**/*.jar" />
    <pathelement location="${classes.dir}" />
</path>

<!-- Reference in javac -->
<javac classpathref="compile.classpath" ... />
```

---

## PatternSet

A reusable set of include/exclude filename patterns that can be nested inside FileSets and other data types.

```xml
<patternset id="no.tests">
    <include name="**/*.java" />
    <exclude name="**/*Test.java" />
</patternset>
```

---

## Mapper

A **transformation rule** that converts source file names into target file names. Used internally by tasks like `<copy>` and `<move>`.

| Mapper Type | Example |
|---|---|
| `identity` | `Foo.java` → `Foo.java` (unchanged) |
| `flatten` | `com/example/Foo.java` → `Foo.java` |
| `glob` | `*.java` → `*.class` |
| `regexp` | Full regular expression substitution |
| `merge` | All files → single named file |

---

## Filter / FilterSet

A **token replacement rule** applied to file content during copy operations. Replaces `@TOKEN@` placeholders with property values.

```xml
<filterset id="app.filters">
    <filter token="VERSION"    value="${app.version}" />
    <filter token="BUILD_DATE" value="${build.date}" />
</filterset>
```

---

## Listener and Logger

A **Listener** monitors Ant build events (build started/finished, target started/finished, task started/finished) and reacts to them. The default listener prints output to the console.

A **Logger** specifically controls how build output is formatted. Common loggers:
- `NoBannerLogger` — suppresses the Ant banner
- `AnsiColorLogger` — colourised output
- `XmlLogger` — writes build events as XML (for CI integration)

```bash
ant -logger org.apache.tools.ant.NoBannerLogger compile
ant -logger org.apache.tools.ant.XmlLogger -logfile build-log.xml compile
```

---

## BuildException

The Java exception class thrown when an Ant task encounters an error. It causes the build to fail immediately unless the task has `failonerror="false"`:

```xml
<javac srcdir="${src.dir}" destdir="${classes.dir}" failonerror="true" />
<exec executable="nonexistent-tool" failonerror="false" />   <!-- won't stop build -->
```

---

| | |
|---|---|
| [← Chapter 1: Introduction](./01-introduction.md) | [Next → Chapter 3: Build File Structure](./03-build-file-structure.md) |
