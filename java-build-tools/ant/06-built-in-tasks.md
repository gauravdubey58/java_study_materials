[← Chapter 5: Data Types](./05-data-types.md) | [Index](../index.md)

---

# Chapter 6 — Built-in Tasks

Ant ships with approximately 150 built-in tasks. This chapter documents the most important ones grouped by category.

---

## File & Directory Operations

### `<mkdir>` — Create Directories

Creates directories, including all missing parents.

```xml
<mkdir dir="${build.dir}/classes" />
<mkdir dir="${reports.dir}/html" />
```

### `<delete>` — Delete Files or Directories

```xml
<!-- Delete a directory tree -->
<delete dir="${build.dir}" />

<!-- Delete specific files -->
<delete file="old-artifact.jar" />

<!-- Delete by pattern -->
<delete>
    <fileset dir="." includes="**/*.tmp,**/*.bak" />
</delete>

<!-- Delete but keep the directory itself -->
<delete includeEmptyDirs="false">
    <fileset dir="${dist.dir}" includes="**/*" />
</delete>
```

### `<copy>` — Copy Files or Directories

```xml
<!-- Copy a single file -->
<copy file="src/config.xml" tofile="dist/config.xml" />

<!-- Copy a directory tree -->
<copy todir="${dist.dir}/resources">
    <fileset dir="${res.dir}" />
</copy>

<!-- Copy and overwrite only if source is newer -->
<copy todir="${dist.dir}" overwrite="false">
    <fileset dir="${src.dir}" includes="**/*.properties" />
</copy>

<!-- Copy with token filtering -->
<copy todir="${dist.dir}" filtering="true">
    <fileset dir="${res.dir}" includes="**/*.xml" />
    <filterset>
        <filter token="VERSION" value="${app.version}" />
    </filterset>
</copy>

<!-- Copy with file renaming -->
<copy todir="${backup.dir}">
    <fileset dir="${src.dir}" includes="**/*.java" />
    <mapper type="glob" from="*.java" to="*.java.bak" />
</copy>
```

### `<move>` — Move or Rename Files

```xml
<move file="temp.jar" tofile="${dist.dir}/myapp.jar" />
<move todir="${archive.dir}">
    <fileset dir="${dist.dir}" includes="*.jar" />
</move>
```

### `<chmod>` — Change File Permissions (Unix/Linux/macOS)

```xml
<chmod file="${dist.dir}/run.sh" perm="755" />
<chmod dir="${scripts.dir}" perm="755" includes="**/*.sh" />
```

### `<touch>` — Create File or Update Timestamp

```xml
<touch file="build.marker" />
<touch>
    <fileset dir="${src.dir}" includes="**/*.java" />
</touch>
```

### `<replace>` — In-place String Replacement

```xml
<replace file="version.properties"
         token="@VERSION@"
         value="${app.version}" />

<!-- Replace across many files -->
<replace dir="${src.dir}" token="TODO" value="DONE">
    <include name="**/*.java" />
</replace>
```

### `<fixcrlf>` — Normalise Line Endings

```xml
<!-- Convert to Unix line endings (LF) -->
<fixcrlf srcdir="${src.dir}" eol="lf" includes="**/*.sh" />

<!-- Convert to Windows line endings (CRLF) -->
<fixcrlf srcdir="${src.dir}" eol="crlf" includes="**/*.bat" />
```

### `<concat>` — Concatenate Files

```xml
<concat destfile="${build.dir}/all-sql.sql" append="false">
    <fileset dir="sql" includes="*.sql" />
</concat>
```

---

## Java Compilation & Execution

### `<javac>` — Compile Java Source Files

The most important task in Java builds.

```xml
<javac srcdir="${src.dir}"
       destdir="${classes.dir}"
       classpathref="compile.classpath"
       source="17"
       target="17"
       encoding="UTF-8"
       debug="true"
       debuglevel="lines,vars,source"
       deprecation="true"
       includeantruntime="false"
       failonerror="true"
       verbose="false">
    <include name="**/*.java" />
    <exclude name="**/generated/**" />
    <compilerarg value="-Xlint:all" />
    <compilerarg value="-proc:full" />
</javac>
```

Key attributes:

| Attribute | Description |
|---|---|
| `srcdir` | Source directory (required) |
| `destdir` | Output directory for `.class` files |
| `classpathref` | Reference to a `<path>` for compilation |
| `source` / `target` | Java version compatibility (use same value) |
| `debug` | Include debug info in bytecode (`true` / `false`) |
| `includeantruntime` | Set to `false` to exclude Ant's own JARs |
| `encoding` | Source file character encoding |
| `failonerror` | Stop build on compile error (default: `true`) |

### `<java>` — Run a Java Program

```xml
<java classname="com.example.DataMigrator"
      fork="true"
      failonerror="true">
    <classpath refid="runtime.classpath" />
    <arg value="--config" />
    <arg value="production.properties" />
    <jvmarg value="-Xmx1g" />
    <jvmarg value="-Dfile.encoding=UTF-8" />
    <sysproperty key="log.level" value="DEBUG" />
    <env key="APP_ENV" value="production" />
    <redirector output="${build.dir}/migration.log"
                alwayslog="true" />
</java>
```

`fork="true"` runs the program in a new JVM process — required for setting JVM args, changing system properties, or running with a different classpath.

### `<javadoc>` — Generate API Documentation

```xml
<javadoc sourcepath="${src.dir}"
         destdir="${build.dir}/javadoc"
         classpathref="compile.classpath"
         packagenames="com.example.*"
         windowtitle="MyApp ${app.version} API"
         doctitle="&lt;h1&gt;MyApp&lt;/h1&gt;"
         access="protected"
         author="true"
         version="true"
         use="true"
         encoding="UTF-8"
         charset="UTF-8">
    <link href="https://docs.oracle.com/en/java/javase/17/docs/api/" />
</javadoc>
```

---

## Archive Tasks

### `<jar>` — Create a JAR File

```xml
<jar destfile="${dist.dir}/${jar.name}"
     basedir="${classes.dir}"
     compress="true"
     index="true">
    <manifest>
        <attribute name="Main-Class"              value="${main.class}" />
        <attribute name="Class-Path"              value="lib/guava.jar" />
        <attribute name="Implementation-Version"  value="${app.version}" />
        <attribute name="Built-By"                value="${user.name}" />
    </manifest>
    <!-- Add resources from another location -->
    <fileset dir="${res.dir}" />
    <!-- Exclude unwanted content -->
    <exclude name="**/*.txt" />
</jar>
```

### `<war>` — Create a WAR File

```xml
<war destfile="${dist.dir}/myapp.war"
     webxml="src/main/webapp/WEB-INF/web.xml">
    <!-- Web content (HTML, JSP, CSS, JS) -->
    <fileset dir="src/main/webapp">
        <exclude name="WEB-INF/web.xml" />
    </fileset>
    <!-- Runtime dependency JARs → WEB-INF/lib/ -->
    <lib dir="${lib.dir}/runtime" />
    <!-- Compiled classes → WEB-INF/classes/ -->
    <classes dir="${classes.dir}" />
    <!-- Extra WEB-INF content -->
    <webinf dir="src/main/webapp/WEB-INF">
        <exclude name="web.xml" />
    </webinf>
</war>
```

### `<zip>` — Create a ZIP Archive

```xml
<zip destfile="${dist.dir}/myapp-${app.version}.zip">
    <zipfileset dir="${dist.dir}" includes="*.jar" prefix="myapp/lib" />
    <zipfileset dir="scripts" includes="*.sh" prefix="myapp/bin" filemode="755" />
    <zipfileset dir="docs"                      prefix="myapp/docs" />
    <zipfileset dir="." includes="LICENSE.txt"  prefix="myapp" />
</zip>
```

### `<unzip>` / `<unjar>` — Extract Archives

```xml
<unzip src="${downloads.dir}/library.zip" dest="${lib.dir}" overwrite="true">
    <patternset>
        <include name="**/*.jar" />
    </patternset>
</unzip>
```

### `<tar>` — Create TAR / TAR.GZ Archives

```xml
<tar destfile="${dist.dir}/myapp-${app.version}.tar.gz"
     compression="gzip"
     longfile="gnu">
    <tarfileset dir="${dist.dir}" includes="*.jar" prefix="myapp" />
    <tarfileset dir="scripts" includes="*.sh" prefix="myapp/bin" mode="755" />
</tar>
```

---

## I/O and Output Tasks

### `<echo>` — Print Messages

```xml
<echo message="Building version ${app.version}" />
<echo level="warning" message="Deprecated dependency detected!" />
<echo level="error"   message="Critical issue!" />
<echo file="${build.dir}/version.txt" message="${app.version}" />
<echo append="true" file="${build.dir}/build.log" message="${build.date}" />
```

### `<input>` — Prompt for User Input

```xml
<input message="Enter environment (dev/staging/prod):"
       validargs="dev,staging,prod"
       addproperty="deploy.env" />
<echo message="Deploying to: ${deploy.env}" />
```

### `<loadfile>` — Load File Content into Property

```xml
<loadfile property="readme.text" srcfile="README.md" />
<loadfile property="version.number" srcfile="VERSION">
    <filterchain>
        <striplinebreaks />
        <trim />
    </filterchain>
</loadfile>
```

---

## Process Execution

### `<exec>` — Run an External Command

```xml
<!-- Simple command -->
<exec executable="git" failonerror="true">
    <arg value="commit" />
    <arg value="-m" />
    <arg value="Release ${app.version}" />
</exec>

<!-- Capture output into property -->
<exec executable="git" outputproperty="git.hash" failonerror="true">
    <arg line="rev-parse --short HEAD" />
</exec>
<echo message="Building at commit: ${git.hash}" />

<!-- Platform-conditional execution -->
<exec executable="cmd" osfamily="windows">
    <arg value="/c" />
    <arg value="scripts/build.bat" />
</exec>
<exec executable="scripts/build.sh" osfamily="unix" />
```

---

## Property and Timestamp Tasks

### `<tstamp>` — Set Timestamp Properties

```xml
<tstamp>
    <format property="build.date"      pattern="yyyy-MM-dd" />
    <format property="build.time"      pattern="HH:mm:ss" />
    <format property="build.timestamp" pattern="yyyyMMdd-HHmmss" />
    <format property="build.year"      pattern="yyyy" />
</tstamp>
```

### `<propertyfile>` — Create/Update a Properties File

```xml
<propertyfile file="${build.dir}/build-info.properties"
              comment="Build metadata — generated by Ant">
    <entry key="version"      value="${app.version}" />
    <entry key="build.date"   type="date" value="now" pattern="yyyy-MM-dd" />
    <entry key="build.number" type="int"  operation="+" value="1" />
    <entry key="built.by"     value="${user.name}" />
</propertyfile>
```

### `<checksum>` — Generate File Checksums

```xml
<checksum file="${dist.dir}/myapp.jar" algorithm="SHA-256"
          property="jar.sha256" />
<echo file="${dist.dir}/myapp.jar.sha256" message="${jar.sha256}" />
```

---

| | |
|---|---|
| [← Chapter 5: Data Types](./05-data-types.md) | [Next → Chapter 7: Antlibs & Optional Tasks](./07-antlibs.md) |
