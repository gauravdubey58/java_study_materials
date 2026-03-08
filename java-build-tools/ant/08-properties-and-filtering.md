[← Chapter 7: Antlibs & Optional Tasks](./07-antlibs.md) | [Index](../index.md)

---

# Chapter 8 — Properties & Filtering

Properties are Ant's variable system. Understanding their immutability rule and loading order is essential for building flexible, configurable build files.

---

## The Immutability Rule

**The first definition of a property wins. Later definitions are silently ignored.**

This is the opposite of most programming languages and catches beginners off guard. It exists to allow overrides from the command line:

```bash
ant -Ddebug=false compile    # -D properties are set FIRST, before build.xml is read
```

In `build.xml`:
```xml
<property name="debug" value="true" />   <!-- silently ignored — already set to 'false' -->
```

This rule is what makes the layered property loading pattern work.

---

## Property Loading Order (Layered Configuration)

The recommended pattern is to load properties from least-specific to most-specific. Since the first definition wins, more-specific files override defaults:

```xml
<!-- 1. Load user-specific overrides FIRST (highest priority) -->
<property file="${user.home}/my-project-local.properties" />

<!-- 2. Load environment-specific settings -->
<property file="${env.name}.properties" />    <!-- e.g., production.properties -->

<!-- 3. Load project defaults LAST (lowest priority) -->
<property file="build.properties" />

<!-- 4. Hard-coded fallback defaults in build.xml (lowest of all) -->
<property name="debug"        value="true" />
<property name="build.dir"    value="build" />
```

---

## Property Declarations

```xml
<!-- Simple string value -->
<property name="app.version" value="1.5.0" />

<!-- Location: resolves relative path to absolute path -->
<property name="lib.dir" location="lib" />
<!-- If basedir is /home/user/project, lib.dir = /home/user/project/lib -->

<!-- From an external .properties file -->
<property file="build.properties" />

<!-- From a properties file on the classpath -->
<property resource="default-config.properties" />

<!-- From OS environment variables (prefix: env.) -->
<property environment="env" />
<echo message="Java home: ${env.JAVA_HOME}" />
<echo message="System path: ${env.PATH}" />

<!-- Conditional: set only if not already set -->
<property name="log.level" value="DEBUG" />  <!-- use immutability as a guard -->
```

---

## Reading Properties in Tasks

Reference any property with `${property.name}`:

```xml
<echo message="Building ${app.name} version ${app.version}" />
<mkdir dir="${build.dir}/classes" />
<javac destdir="${classes.dir}" ... />
```

Nested property references work:
```xml
<property name="env.name"    value="production" />
<property name="db.url.production" value="jdbc:postgresql://prod/app" />
<property name="db.url" value="${db.url.${env.name}}" />
<!-- db.url = jdbc:postgresql://prod/app -->
```

---

## The `<propertyfile>` Task

Creates or updates a Java-format `.properties` file. Useful for incrementing build numbers:

```xml
<propertyfile file="build.properties">
    <entry key="build.number" type="int"  operation="+" value="1" />
    <entry key="build.date"   type="date" value="now" pattern="yyyy-MM-dd" />
    <entry key="build.user"   value="${user.name}" />
    <entry key="app.version"  value="${app.version}" />
</propertyfile>
```

---

## Resource Filtering

Token replacement transforms `@TOKEN@` placeholders in files during copy operations.

### Using a FilterSet

```xml
<filterset id="build.info">
    <filter token="APP_VERSION"  value="${app.version}" />
    <filter token="BUILD_DATE"   value="${build.date}" />
    <filter token="DB_URL"       value="${db.url}" />
</filterset>

<copy todir="${dist.dir}" filtering="true">
    <fileset dir="${res.dir}" includes="**/*.properties,**/*.xml" />
    <filterset refid="build.info" />
</copy>
```

Source file (`app.properties`):
```properties
version=@APP_VERSION@
built=@BUILD_DATE@
db.url=@DB_URL@
```

After copy:
```properties
version=1.5.0
built=2024-06-15
db.url=jdbc:postgresql://prod-db:5432/app
```

### Loading Filters from a File

```xml
<filterset>
    <filtersfile file="env-production.properties" />
</filterset>
```

`env-production.properties`:
```properties
DB_HOST=prod-db.internal
DB_PORT=5432
LOG_LEVEL=WARN
```

---

| | |
|---|---|
| [← Chapter 7: Antlibs & Optional Tasks](./07-antlibs.md) | [Next → Chapter 9: Conditions & Control Flow](./09-conditions-and-control-flow.md) |
