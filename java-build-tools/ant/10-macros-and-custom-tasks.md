[← Chapter 9: Conditions & Control Flow](./09-conditions-and-control-flow.md) | [Index](../index.md)

---

# Chapter 10 — Macros & Custom Tasks

Ant provides three ways to create reusable build logic: `<macrodef>` for XML templates, `<scriptdef>` for inline scripts, and Java-based custom tasks for full programmatic control.

---

## `<macrodef>` — XML Macros

A `<macrodef>` packages a sequence of tasks into a reusable, parameterised template. It is Ant's primary code-reuse mechanism and the simplest way to avoid copy-pasting repeated build logic.

### Defining a Macro

```xml
<macrodef name="compile-module">
    <!-- String attributes (referenced as @{name} inside the macro) -->
    <attribute name="src"     />
    <attribute name="dest"    />
    <attribute name="name"    default="module" />
    <attribute name="version" default="${app.version}" />

    <!-- Element: a placeholder for XML content passed at call site -->
    <element name="extra-classpath" optional="true" />

    <!-- The macro body -->
    <sequential>
        <echo message="─── Compiling @{name} @{version} ───" />
        <mkdir dir="@{dest}" />
        <javac srcdir="@{src}"
               destdir="@{dest}"
               source="${java.source}"
               target="${java.source}"
               includeantruntime="false">
            <classpath refid="compile.classpath" />
            <extra-classpath />    <!-- caller's classpath additions inserted here -->
        </javac>
        <echo message="@{name} compiled → @{dest}" />
    </sequential>
</macrodef>
```

> Note: Inside a macro body, attributes are referenced with `@{attr}` not `${attr}`. This avoids conflicts with project properties.

### Using the Macro

```xml
<!-- Simple invocation -->
<compile-module src="${core.src}" dest="${core.classes}" name="core" />

<!-- With element content (extra-classpath) -->
<compile-module src="${web.src}" dest="${web.classes}" name="web">
    <extra-classpath>
        <pathelement location="${core.classes}" />
        <pathelement location="${lib.dir}/servlet-api.jar" />
    </extra-classpath>
</compile-module>
```

### A More Practical Macro — Environment-Aware Deploy

```xml
<macrodef name="deploy-artifact">
    <attribute name="env"      />
    <attribute name="artifact" />
    <attribute name="dest-dir" />

    <sequential>
        <echo message="Deploying @{artifact} to @{env} → @{dest-dir}" />
        <scp file="${dist.dir}/@{artifact}"
             todir="${deploy.user}@${deploy.host.@{env}}:@{dest-dir}"
             password="${deploy.password}" trust="true" />
        <sshexec host="${deploy.host.@{env}}"
                 username="${deploy.user}"
                 password="${deploy.password}" trust="true"
                 command="sudo systemctl restart myapp" />
    </sequential>
</macrodef>

<!-- Usage -->
<deploy-artifact env="staging"    artifact="myapp-1.5.jar" dest-dir="/opt/myapp/lib" />
<deploy-artifact env="production" artifact="myapp-1.5.jar" dest-dir="/opt/myapp/lib" />
```

---

## `<scriptdef>` — Inline Script Tasks

Define a custom task using an inline script (JavaScript, Groovy, BeanShell, etc.). Useful for one-off logic that doesn't justify a full Java task.

```xml
<!-- JavaScript (built into JDK via Nashorn/Rhino) -->
<scriptdef name="reverse-string" language="javascript">
    <attribute name="value"    />
    <attribute name="property" />
    <![CDATA[
        var reversed = attributes.get("value")
                            .split("").reverse().join("");
        project.setProperty(attributes.get("property"), reversed);
    ]]>
</scriptdef>

<reverse-string value="hello world" property="reversed" />
<echo message="${reversed}" />   <!-- dlrow olleh -->
```

```xml
<!-- Multi-line property expansion -->
<scriptdef name="format-version" language="javascript">
    <attribute name="major" />
    <attribute name="minor" />
    <attribute name="patch" />
    <attribute name="property" />
    <![CDATA[
        var v = attributes.get("major") + "."
              + attributes.get("minor") + "."
              + attributes.get("patch");
        project.setProperty(attributes.get("property"), v);
    ]]>
</scriptdef>
```

---

## Writing a Java Custom Task

For complex or reusable tasks, implement a Java class and register it with `<taskdef>`.

### Step 1: Write the Task Class

```java
package com.example.ant;

import org.apache.tools.ant.BuildException;
import org.apache.tools.ant.Task;
import java.util.ArrayList;
import java.util.List;

/**
 * Custom Ant task: <validateConfig file="..." requiredKeys="key1,key2" />
 */
public class ValidateConfigTask extends Task {

    private String file;
    private String requiredKeys;

    // Setters are called by Ant for each XML attribute
    public void setFile(String file)               { this.file = file; }
    public void setRequiredKeys(String keys)       { this.requiredKeys = keys; }

    @Override
    public void execute() throws BuildException {
        if (file == null) {
            throw new BuildException("'file' attribute is required", getLocation());
        }

        log("Validating config: " + file, org.apache.tools.ant.Project.MSG_INFO);

        java.util.Properties props = new java.util.Properties();
        try (java.io.FileInputStream fis = new java.io.FileInputStream(file)) {
            props.load(fis);
        } catch (java.io.IOException e) {
            throw new BuildException("Cannot read config file: " + file, e, getLocation());
        }

        if (requiredKeys != null) {
            List<String> missing = new ArrayList<>();
            for (String key : requiredKeys.split(",")) {
                if (!props.containsKey(key.trim())) {
                    missing.add(key.trim());
                }
            }
            if (!missing.isEmpty()) {
                throw new BuildException(
                    "Config file missing required keys: " + missing, getLocation());
            }
        }

        log("Config validation passed.", org.apache.tools.ant.Project.MSG_INFO);
    }
}
```

### Step 2: Compile and Package

```xml
<!-- In a separate build step, compile the task and package it -->
<javac srcdir="tasks/src" destdir="tasks/classes">
    <classpath>
        <pathelement location="${ant.home}/lib/ant.jar" />
    </classpath>
</javac>
<jar destfile="${lib.dir}/my-tasks.jar" basedir="tasks/classes" />
```

### Step 3: Register and Use

```xml
<taskdef name="validateConfig"
         classname="com.example.ant.ValidateConfigTask"
         classpath="${lib.dir}/my-tasks.jar" />

<validateConfig file="production.properties"
                requiredKeys="db.url,db.password,app.secret" />
```

---

## Best Practices

Use `<macrodef>` when you want to avoid duplicating the same sequence of tasks across targets. Use `<scriptdef>` for small, self-contained string or property manipulation that doesn't justify a Java class. Write a Java task when you need full IDE support, unit testability, or logic that would be awkward in XML or a script.

---

| | |
|---|---|
| [← Chapter 9: Conditions & Control Flow](./09-conditions-and-control-flow.md) | [Next → Chapter 11: Ivy — Dependency Management](./11-ivy-dependency-management.md) |
