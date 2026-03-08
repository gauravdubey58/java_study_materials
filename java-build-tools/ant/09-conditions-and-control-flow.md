[← Chapter 8: Properties & Filtering](./08-properties-and-filtering.md) | [Index](../index.md)

---

# Chapter 9 — Conditions & Control Flow

Ant provides several mechanisms for conditional execution. This chapter covers all condition types and the patterns for using them.

---

## `<condition>` Task

Sets a property to a value if a condition evaluates to true:

```xml
<!-- Set property if condition is true -->
<condition property="is.windows">
    <os family="windows" />
</condition>

<!-- Set one of two values (if/else) -->
<condition property="exec.extension" value=".bat" else="">
    <os family="windows" />
</condition>

<!-- Complex condition -->
<condition property="build.ready" value="yes" else="no">
    <and>
        <available file="${src.dir}" />
        <available file="${lib.dir}" />
        <not>
            <isset property="build.skip" />
        </not>
    </and>
</condition>
```

---

## Condition Types

### File and Resource Conditions

```xml
<!-- File or directory exists -->
<available file="config.xml" property="config.present" />
<available file="${lib.dir}" type="dir" property="lib.dir.exists" />

<!-- Java class is on the classpath -->
<available classname="org.junit.Test"
           classpathref="compile.classpath"
           property="junit.available" />

<!-- Classpath resource exists -->
<available resource="META-INF/spring.factories" property="spring.present" />
```

### String Conditions

```xml
<!-- Equality -->
<equals arg1="${env}" arg2="production" />

<!-- Contains -->
<contains string="${version}" substring="SNAPSHOT" />

<!-- Matches regex -->
<matches string="${version}" pattern="^\d+\.\d+\.\d+$" />

<!-- Property is set (to any value) -->
<isset property="skip.tests" />
```

### OS Conditions

```xml
<os family="windows" />
<os family="unix" />
<os family="mac" />
<os name="Linux" />
<os arch="amd64" />
```

### Network Conditions

```xml
<!-- HTTP server is responding -->
<http url="https://nexus.company.com/" />

<!-- TCP port is open -->
<socket server="db.example.com" port="5432" />
```

### Logical Combinators

```xml
<and>
    <available file="src" />
    <not><os family="windows" /></not>
</and>

<or>
    <equals arg1="${env}" arg2="dev" />
    <equals arg1="${env}" arg2="staging" />
</or>

<not>
    <isset property="skip.tests" />
</not>

<xor>
    <isset property="flag.a" />
    <isset property="flag.b" />
</xor>
```

---

## `<fail>` — Fail the Build

```xml
<!-- Fail if property IS set -->
<fail if="build.errors" message="Build errors detected — check logs" />

<!-- Fail if property is NOT set -->
<fail unless="deploy.env" message="Set -Ddeploy.env=dev|staging|prod" />

<!-- Fail with a condition -->
<fail message="Java 17+ is required">
    <condition>
        <not>
            <javaversion atLeast="17" />
        </not>
    </condition>
</fail>
```

---

## Conditional Target Execution

The `if` and `unless` attributes on `<target>` provide basic conditional execution:

```xml
<!-- Run only if property IS set -->
<target name="sign-artifacts" if="signing.enabled">
    <exec executable="jarsigner" ... />
</target>

<!-- Run only if property is NOT set -->
<target name="use-defaults" unless="custom.config">
    <copy file="default-config.xml" tofile="config.xml" />
</target>

<!-- Combine in a depends chain -->
<target name="deploy" depends="package, sign-artifacts, use-defaults">
    ...
</target>
```

---

## `<uptodate>` — Incremental Builds

Sets a property if the target files are newer than the source files. Used to skip work that doesn't need redoing:

```xml
<uptodate property="jar.is.current"
          targetfile="${dist.dir}/myapp.jar">
    <srcfiles dir="${src.dir}"     includes="**/*.java" />
    <srcfiles dir="${classes.dir}" includes="**/*.class" />
</uptodate>

<target name="package-if-needed" unless="jar.is.current">
    <jar destfile="${dist.dir}/myapp.jar" ... />
</target>
```

---

## Advanced Control with Ant-Contrib

When you need full if/else and loops (not just property-based conditionals), add the Ant-Contrib library (see [Chapter 7](./07-antlibs.md)):

```xml
<!-- Full if/else/elseif -->
<if>
    <equals arg1="${env}" arg2="production" />
    <then>
        <property name="server.url" value="https://app.example.com" />
    </then>
    <elseif>
        <equals arg1="${env}" arg2="staging" />
        <then>
            <property name="server.url" value="https://staging.example.com" />
        </then>
    </elseif>
    <else>
        <property name="server.url" value="http://localhost:8080" />
    </else>
</if>

<!-- For loop -->
<for list="${module.list}" param="module" delimiter=",">
    <sequential>
        <echo message="Processing: @{module}" />
        <ant dir="@{module}" target="package" inheritAll="false" />
    </sequential>
</for>
```

---

| | |
|---|---|
| [← Chapter 8: Properties & Filtering](./08-properties-and-filtering.md) | [Next → Chapter 10: Macros & Custom Tasks](./10-macros-and-custom-tasks.md) |
