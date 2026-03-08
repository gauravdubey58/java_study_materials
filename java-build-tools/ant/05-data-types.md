[← Chapter 4: Lifecycle & Execution Model](./04-lifecycle-execution.md) | [Index](../index.md)

---

# Chapter 5 — Data Types

Ant **data types** are reusable structures that describe collections of files, paths, and transformation rules. They can be defined once with an `id` attribute and referenced anywhere with `refid`.

---

## `<fileset>` — File Collections

A `<fileset>` selects a group of files from a directory using include/exclude patterns.

```xml
<!-- Define with id for reuse -->
<fileset id="main.sources" dir="${src.dir}">
    <include name="**/*.java" />
    <exclude name="**/generated/**" />
    <exclude name="**/*Test.java" />
</fileset>

<!-- Inline (used directly inside a task) -->
<copy todir="${dist.dir}">
    <fileset dir="${res.dir}" includes="**/*.properties" />
</copy>

<!-- Reference a defined fileset -->
<javac destdir="${classes.dir}">
    <src>
        <fileset refid="main.sources" />
    </src>
</javac>
```

### Pattern Syntax

| Pattern | Matches |
|---|---|
| `**/*.java` | All `.java` files in any subdirectory |
| `*.java` | `.java` files in the fileset root only |
| `com/**` | Everything under the `com/` subdirectory |
| `**/*Test.java` | Files ending in `Test.java` anywhere |
| `!**/skip/**` | Exclude pattern (prefix with `!` in some tasks) |

---

## `<path>` — Classpath Definitions

A `<path>` defines an ordered list of locations for use as a Java classpath. Always define paths with an `id` so they can be reused.

```xml
<!-- Compile classpath: all JARs in lib/compile/ -->
<path id="compile.classpath">
    <fileset dir="${lib.dir}/compile" includes="**/*.jar" />
</path>

<!-- Test classpath: compile classpath + test JARs + compiled classes -->
<path id="test.classpath">
    <path refid="compile.classpath" />         <!-- reference another path -->
    <fileset dir="${lib.dir}/test" includes="**/*.jar" />
    <pathelement location="${classes.dir}" />   <!-- single directory -->
    <pathelement location="${test.classes}" />
    <pathelement path="${env.CLASSPATH}" />     <!-- OS environment variable -->
</path>
```

### Using Paths

```xml
<javac classpathref="compile.classpath" ... />
<java classpathref="runtime.classpath" ... />
```

### Converting a Path to a String

```xml
<pathconvert property="classpath.str"
             pathsep=":"
             refid="runtime.classpath" />
<echo message="Classpath: ${classpath.str}" />
```

---

## `<patternset>` — Reusable Patterns

A `<patternset>` captures include/exclude patterns that can be applied to multiple filesets.

```xml
<patternset id="java.files">
    <include name="**/*.java" />
    <exclude name="**/*Test.java" />
    <exclude name="**/package-info.java" />
</patternset>

<!-- Apply the same patterns to different directories -->
<fileset dir="${src.dir}">
    <patternset refid="java.files" />
</fileset>

<fileset dir="${gen.src.dir}">
    <patternset refid="java.files" />
</fileset>
```

---

## `<dirset>` — Directory Collections

Like `<fileset>` but matches directories instead of files.

```xml
<dirset id="module.dirs" dir="${modules.dir}">
    <include name="*" />             <!-- all immediate subdirectories -->
    <exclude name="deprecated-*" />
</dirset>
```

---

## `<filelist>` — Explicit File Lists

An explicit, ordered list of named files. Unlike `<fileset>`, files don't need to exist at definition time.

```xml
<filelist id="config.files" dir="${res.dir}">
    <file name="database.properties" />
    <file name="logging.xml" />
    <file name="app.properties" />
</filelist>

<!-- Concatenate in declared order -->
<concat destfile="${dist.dir}/all-config.properties">
    <filelist refid="config.files" />
</concat>
```

---

## `<mapper>` — File Name Transformations

A `<mapper>` converts source file names to target file names. Used inside `<copy>`, `<move>`, `<apply>`, and others.

```xml
<!-- identity (default): src name = dest name -->
<mapper type="identity" />

<!-- flatten: strips directory path -->
<!-- com/example/Foo.java → Foo.java -->
<mapper type="flatten" />

<!-- glob: wildcard substitution -->
<!-- Foo.java → Foo.class -->
<mapper type="glob" from="*.java" to="*.class" />

<!-- regexp: full regex substitution -->
<mapper type="regexp"
        from="^(.*)/([^/]+)\.java$$"
        to="\1/\2.class" />

<!-- package: replaces '.' with '/' in file stem (or vice versa) -->
<mapper type="package" from="*.java" to="*.class" />

<!-- merge: all files map to one file -->
<mapper type="merge" to="combined-output.txt" />
```

### Using a Mapper in `<copy>`

```xml
<!-- Copy all .java files, renaming to .java.bak -->
<copy todir="${backup.dir}">
    <fileset dir="${src.dir}" includes="**/*.java" />
    <mapper type="glob" from="*.java" to="*.java.bak" />
</copy>
```

---

## `<filterset>` — Token Replacement

A `<filterset>` defines `@TOKEN@`-style replacements applied to file content during copy operations.

```xml
<!-- Define once -->
<filterset id="version.filters">
    <filter token="APP_VERSION"  value="${app.version}" />
    <filter token="BUILD_DATE"   value="${build.date}" />
    <filter token="BUILD_USER"   value="${user.name}" />
</filterset>

<!-- Apply during copy -->
<copy todir="${dist.dir}">
    <fileset dir="${res.dir}" includes="**/*.properties" />
    <filterset refid="version.filters" />
</copy>

<!-- Load filters from a properties file -->
<filterset>
    <filtersfile file="env-production.properties" />
</filterset>
```

Source file:
```
app.version=@APP_VERSION@
built.by=@BUILD_USER@
```

After copy, becomes:
```
app.version=1.5.0
built.by=jdoe
```

---

## `<filterchain>` — Stream Processing Pipeline

A `<filterchain>` applies a sequence of content transformations during a copy. More powerful than `<filterset>` — it can filter lines, replace strings, strip comments, etc.

```xml
<copy tofile="${dist.dir}/clean-log.txt" file="raw-log.txt">
    <filterchain>
        <!-- Only include lines containing ERROR -->
        <linecontains>
            <contains value="ERROR" />
        </linecontains>
        <!-- Replace DEBUG with TRACE -->
        <tokenfilter>
            <replacestring from="DEBUG" to="TRACE" />
        </tokenfilter>
        <!-- Remove trailing whitespace -->
        <tokenfilter>
            <trim />
        </tokenfilter>
    </filterchain>
</copy>
```

Common filter chain elements: `<linecontains>`, `<linecontainsregexp>`, `<tokenfilter>`, `<striplinecomments>`, `<striplinebreaks>`, `<expandproperties>`.

---

| | |
|---|---|
| [← Chapter 4: Lifecycle & Execution Model](./04-lifecycle-execution.md) | [Next → Chapter 6: Built-in Tasks](./06-built-in-tasks.md) |
