[← Chapter 6: Built-in Tasks](./06-built-in-tasks.md) | [Index](../index.md)

---

# Chapter 7 — Antlibs & Optional Tasks

Antlibs extend Ant with additional tasks beyond the built-in set. Each Antlib is a JAR file containing both the task implementations and an `antlib.xml` descriptor. This chapter covers the most important Antlibs used in real Java projects.

---

## Loading an Antlib

There are two ways to load an Antlib's tasks:

### Method 1: `<taskdef>` with classpath

```xml
<taskdef resource="org/apache/ivy/ant/antlib.xml"
         classpath="${ivy.jar.path}"
         uri="antlib:org.apache.ivy.ant" />
```

### Method 2: XML namespace (preferred for Antlibs)

```xml
<project xmlns:ivy="antlib:org.apache.ivy.ant">
    <!-- Ant auto-loads the Antlib from the classpath -->
    <ivy:retrieve />
</project>
```

Place the Antlib JAR in `$ANT_HOME/lib` (global) or reference it via the `-lib` command-line flag:

```bash
ant -lib path/to/antlib.jar target-name
```

---

## JUnit — Unit Testing

JUnit support is included with Ant but requires `junit.jar` and `ant-junit.jar` in `$ANT_HOME/lib`.

### `<junit>` — Run Tests

```xml
<junit printsummary="withOutAndErr"
       haltonfailure="false"
       failureproperty="tests.failed"
       fork="true"
       forkmode="perBatch"
       maxmemory="512m">

    <classpath refid="test.classpath" />

    <!-- Output formatters -->
    <formatter type="xml" />                            <!-- for junitreport -->
    <formatter type="brief" usefile="false" />          <!-- to console -->

    <!-- Run all *Test.class files -->
    <batchtest todir="${reports.dir}">
        <fileset dir="${test.classes}">
            <include name="**/*Test.class" />
            <exclude name="**/Abstract*.class" />
        </fileset>
    </batchtest>

    <!-- Or run a single test -->
    <test name="com.example.UserServiceTest" todir="${reports.dir}" />

    <jvmarg value="-Dfile.encoding=UTF-8" />
    <sysproperty key="db.url" value="jdbc:h2:mem:testdb" />
</junit>

<!-- Generate HTML report from XML results -->
<junitreport todir="${reports.dir}">
    <fileset dir="${reports.dir}" includes="TEST-*.xml" />
    <report format="frames" todir="${reports.dir}/html" />
</junitreport>

<!-- Fail the build if any tests failed -->
<fail if="tests.failed" message="Unit tests FAILED — check ${reports.dir}/html" />
```

Key `<junit>` attributes:

| Attribute | Description |
|---|---|
| `printsummary` | `yes`, `no`, `withOutAndErr` — how to display results |
| `haltonfailure` | Stop on first test failure |
| `failureproperty` | Set this property if any test fails (check it later) |
| `fork` | Run in separate JVM (`true` recommended) |
| `forkmode` | `once` (one JVM for all tests), `perBatch`, `perTest` |

---

## Apache Ivy — Dependency Management

Ivy is a standalone dependency manager that integrates with Ant and resolves Maven-compatible dependencies. See [Chapter 11](./11-ivy-dependency-management.md) for the full Ivy reference.

```xml
<project xmlns:ivy="antlib:org.apache.ivy.ant">
    <target name="resolve">
        <ivy:retrieve pattern="${lib.dir}/[conf]/[artifact]-[revision].[ext]"
                      sync="true" />
    </target>
</project>
```

---

## Checkstyle — Code Style Enforcement

Checks Java source code against a style configuration (Google Style, Sun Style, or custom).

```xml
<taskdef resource="com/puppycrawl/tools/checkstyle/ant/checkstyle-ant-task.properties"
         classpath="${lib.dir}/checkstyle-all.jar" />

<target name="checkstyle" description="Enforce code style">
    <checkstyle config="checkstyle.xml"
                failOnViolation="true"
                maxErrors="0"
                maxWarnings="10">
        <fileset dir="${src.dir}" includes="**/*.java" />
        <formatter type="plain" />
        <formatter type="xml" tofile="${reports.dir}/checkstyle.xml" />
    </checkstyle>
</target>
```

---

## SpotBugs (formerly FindBugs) — Static Analysis

Analyses compiled bytecode for common bug patterns.

```xml
<taskdef name="spotbugs"
         classname="edu.umd.cs.findbugs.anttask.FindBugsTask"
         classpath="${spotbugs.home}/lib/spotbugs-ant.jar" />

<target name="spotbugs" depends="compile" description="Static bug analysis">
    <spotbugs home="${spotbugs.home}"
              output="xml"
              outputFile="${reports.dir}/spotbugs.xml"
              effort="max"
              threshold="Low"
              failOnError="true">
        <sourcePath path="${src.dir}" />
        <class location="${classes.dir}" />
        <auxClasspath refid="compile.classpath" />
    </spotbugs>
</target>
```

---

## JaCoCo — Code Coverage

Instruments test runs and generates coverage reports in multiple formats.

```xml
<taskdef uri="antlib:org.jacoco.ant"
         resource="org/jacoco/ant/antlib.xml"
         classpath="${jacoco.jar}" />

<target name="coverage" depends="compile-tests">

    <!-- Wrap the junit task with jacoco:coverage -->
    <jacoco:coverage destfile="${reports.dir}/jacoco.exec">
        <junit fork="true" forkmode="once" ...>
            <classpath refid="test.classpath" />
            <formatter type="xml" />
            <batchtest todir="${reports.dir}">
                <fileset dir="${test.classes}" includes="**/*Test.class" />
            </batchtest>
        </junit>
    </jacoco:coverage>

    <!-- Generate HTML and XML reports -->
    <jacoco:report>
        <executiondata>
            <file file="${reports.dir}/jacoco.exec" />
        </executiondata>
        <structure name="MyApp">
            <classfiles>
                <fileset dir="${classes.dir}" />
            </classfiles>
            <sourcefiles encoding="UTF-8">
                <fileset dir="${src.dir}" />
            </sourcefiles>
        </structure>
        <html destdir="${reports.dir}/coverage" />
        <xml  destfile="${reports.dir}/coverage.xml" />
        <csv  destfile="${reports.dir}/coverage.csv" />
        <check failOnViolation="true">
            <rule element="BUNDLE">
                <limit counter="LINE" value="COVEREDRATIO" minimum="0.80" />
            </rule>
        </check>
    </jacoco:report>
</target>
```

---

## Ant-Contrib — Additional Control Flow

Adds `<if>`, `<for>`, `<switch>`, `<trycatch>`, `<foreach>`, and more.

```xml
<taskdef resource="net/sf/antcontrib/antcontrib.properties"
         classpath="${lib.dir}/ant-contrib.jar" />

<!-- if / then / else -->
<if>
    <equals arg1="${env}" arg2="production" />
    <then>
        <echo message="Running production build" />
        <property name="log.level" value="WARN" />
    </then>
    <elseif>
        <equals arg1="${env}" arg2="staging" />
        <then>
            <echo message="Running staging build" />
        </then>
    </elseif>
    <else>
        <echo message="Running development build" />
        <property name="log.level" value="DEBUG" />
    </else>
</if>

<!-- for loop over a comma-separated list -->
<for list="module-core,module-service,module-web" param="module">
    <sequential>
        <echo message="Building @{module}..." />
        <ant dir="@{module}" target="package" inheritAll="false" />
    </sequential>
</for>

<!-- switch statement -->
<switch value="${deploy.env}">
    <case value="dev">
        <property name="server.url" value="http://localhost:8080" />
    </case>
    <case value="staging">
        <property name="server.url" value="https://staging.example.com" />
    </case>
    <default>
        <property name="server.url" value="https://app.example.com" />
    </default>
</switch>

<!-- try / catch / finally -->
<trycatch property="error.message" reference="error.ref">
    <try>
        <java classname="com.example.RiskyTask" fork="true" failonerror="true" />
    </try>
    <catch>
        <echo level="error" message="Task failed: ${error.message}" />
        <antcall target="notify-team" />
    </catch>
    <finally>
        <echo message="Cleanup always runs here" />
        <delete file="${lock.file}" />
    </finally>
</trycatch>
```

---

## SSH/SCP/FTP — Remote Operations

These are **optional tasks** included with Ant, requiring `jsch.jar` in `$ANT_HOME/lib`.

```xml
<!-- Upload artifact via SCP -->
<scp file="${dist.dir}/${jar.name}"
     todir="deploy@app-server.example.com:/opt/myapp/releases"
     password="${deploy.password}"
     trust="true" />

<!-- Execute commands over SSH -->
<sshexec host="app-server.example.com"
         username="deploy"
         password="${deploy.password}"
         trust="true"
         command="sudo systemctl restart myapp" />

<!-- Multiple SSH commands in sequence -->
<sshexec ...
         command="cd /opt/myapp &amp;&amp; ./stop.sh &amp;&amp; ./deploy.sh ${app.version} &amp;&amp; ./start.sh" />

<!-- FTP upload -->
<ftp server="ftp.example.com"
     userid="ftpuser"
     password="${ftp.password}"
     remotedir="/releases/${app.version}"
     action="put"
     passive="true">
    <fileset dir="${dist.dir}" includes="*.jar,*.zip" />
</ftp>
```

---

## `xmltask` — XML File Manipulation

A popular third-party Antlib for reading and modifying XML files using XPath.

```xml
<taskdef name="xmltask"
         classname="com.oopsconsultancy.xmltask.ant.XmlTask"
         classpath="${lib.dir}/xmltask.jar" />

<target name="update-config">
    <xmltask source="config.xml" dest="config-updated.xml">
        <!-- Update a text value -->
        <replace path="/config/version/text()" withText="${app.version}" />
        <!-- Insert a new element -->
        <insert path="/config" position="after"
                xml="&lt;build-date&gt;${build.date}&lt;/build-date&gt;" />
        <!-- Delete an element -->
        <remove path="/config/debug-settings" />
        <!-- Read a value into a property -->
        <copy path="/config/database/url/text()" property="db.url" />
    </xmltask>
</target>
```

---

| | |
|---|---|
| [← Chapter 6: Built-in Tasks](./06-built-in-tasks.md) | [Next → Chapter 8: Properties & Filtering](./08-properties-and-filtering.md) |
