[← Chapter 12: Use Cases](./12-use-cases.md) | [Index](../index.md)

---

# Chapter 13 — Quick Reference

A condensed cheat sheet of the most-used Ant commands, tasks, attributes, and patterns.

---

## Command Line

```bash
ant                              # run default target
ant compile                      # run 'compile' target (+ its dependencies)
ant clean package                # run 'clean', then 'package'
ant -f custom.xml test           # use a custom build file
ant -D key=value compile         # set a property (wins over build.xml declarations)
ant -Denv=production deploy      # common pattern for env selection
ant -projecthelp                 # list targets with descriptions
ant -p                           # shorthand for -projecthelp
ant -verbose                     # verbose output
ant -debug                       # maximum debug output
ant -quiet                       # errors only
ant -keep-going (-k)             # continue on failure
ant -lib /path/to/jars compile   # add extra JARs to Ant's classpath
ant -logger org.apache.tools.ant.NoBannerLogger compile
ant -logfile build.log package   # write output to file
```

---

## Most-Used Tasks

```xml
<!-- Directories -->
<mkdir dir="${build.dir}" />
<delete dir="${build.dir}" />
<delete file="old.jar" />

<!-- Files -->
<copy file="src.xml" tofile="dest.xml" />
<copy todir="${dest.dir}"><fileset dir="${src.dir}" /></copy>
<move file="a.jar" tofile="b.jar" />
<touch file="build.marker" />

<!-- Java -->
<javac srcdir="${src.dir}" destdir="${classes.dir}"
       classpathref="compile.classpath"
       source="17" target="17"
       includeantruntime="false" />

<java classname="com.example.Main" fork="true" classpathref="runtime.classpath">
    <arg value="--config" /><arg value="app.properties" />
    <jvmarg value="-Xmx512m" />
</java>

<javadoc sourcepath="${src.dir}" destdir="${build.dir}/javadoc"
         packagenames="com.example.*" />

<!-- Archives -->
<jar  destfile="${dist.dir}/app.jar"  basedir="${classes.dir}" />
<war  destfile="${dist.dir}/app.war"  webxml="src/main/webapp/WEB-INF/web.xml" />
<zip  destfile="${dist.dir}/app.zip">...</zip>
<unzip src="lib.zip" dest="${lib.dir}" />

<!-- Output -->
<echo message="Building ${app.version}" />
<echo level="warning" message="Deprecated!" />

<!-- Processes -->
<exec executable="git" outputproperty="git.hash">
    <arg line="rev-parse --short HEAD" />
</exec>
```

---

## Data Types Cheat Sheet

```xml
<!-- FileSet -->
<fileset id="sources" dir="${src.dir}" includes="**/*.java" excludes="**/*Test.java" />

<!-- Path -->
<path id="cp">
    <fileset dir="${lib.dir}" includes="**/*.jar" />
    <pathelement location="${classes.dir}" />
</path>

<!-- PatternSet -->
<patternset id="no.tests">
    <include name="**/*.java" />
    <exclude name="**/*Test.java" />
</patternset>

<!-- FilterSet -->
<filterset id="tokens">
    <filter token="VERSION" value="${app.version}" />
</filterset>

<!-- Referencing -->
<javac classpathref="cp" ... />
<copy todir="${dest}"><fileset refid="sources" /><filterset refid="tokens" /></copy>
```

---

## Condition Cheat Sheet

```xml
<condition property="is.windows"><os family="windows" /></condition>
<condition property="src.exists"><available file="${src.dir}" /></condition>
<condition property="class.ok"><available classname="org.junit.Test" classpathref="cp" /></condition>
<condition property="port.open"><socket server="db.host" port="5432" /></condition>
<condition property="all.good">
    <and>
        <available file="${src.dir}" />
        <not><isset property="skip.all" /></not>
    </and>
</condition>

<!-- On targets -->
<target name="run-if" if="flag" ... />
<target name="run-unless" unless="flag" ... />

<!-- Fail guard -->
<fail unless="deploy.env" message="Must set -Ddeploy.env=dev|staging|prod" />
<fail if="tests.failed"   message="Tests failed!" />
```

---

## Timestamp & Build Info

```xml
<tstamp>
    <format property="build.date"      pattern="yyyy-MM-dd" />
    <format property="build.timestamp" pattern="yyyyMMdd-HHmmss" />
</tstamp>

<propertyfile file="${classes.dir}/build-info.properties">
    <entry key="version"    value="${app.version}" />
    <entry key="date"       type="date" value="now" pattern="yyyy-MM-dd" />
    <entry key="build.num"  type="int" operation="+" value="1" />
</propertyfile>
```

---

## Macro Skeleton

```xml
<macrodef name="my-macro">
    <attribute name="param1" />
    <attribute name="param2" default="default-value" />
    <element   name="extra"  optional="true" />
    <sequential>
        <echo message="Running with @{param1} and @{param2}" />
        <extra />
    </sequential>
</macrodef>

<my-macro param1="value1">
    <extra><echo message="extra content" /></extra>
</my-macro>
```

---

## Conventional Target Depends Chain

```
init
  └── compile
        └── compile-tests
              └── test
                    └── package
                          └── javadoc
                                └── dist
```

```xml
<target name="dist" depends="package, javadoc">
    <zip destfile="${dist.dir}/release-${app.version}.zip">
        ...
    </zip>
</target>
```

---

## Pattern Quick Reference

| Pattern | Matches |
|---|---|
| `**/*.java` | All `.java` files anywhere |
| `*.java` | `.java` files in root only |
| `com/**` | Everything under `com/` |
| `**/*Test.class` | Test class files anywhere |
| `**` | Everything, including subdirectories |

---

*Official Ant documentation: https://ant.apache.org/manual/*
*Apache Ivy documentation: https://ant.apache.org/ivy/*
*Ant-Contrib tasks: http://ant-contrib.sourceforge.net/*

---

| | |
|---|---|
| [← Chapter 12: Use Cases](./12-use-cases.md) | [↑ Back to Index](../index.md) |
