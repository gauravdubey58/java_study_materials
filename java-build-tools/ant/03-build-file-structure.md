[← Chapter 2: Core Concepts](./02-core-concepts.md) | [Index](../index.md)

---

# Chapter 3 — Build File Structure

A well-structured `build.xml` follows a logical order. This chapter presents a complete, annotated template you can use as a starting point for any Java project.

---

## Recommended File Layout

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project name="MyApp" default="package" basedir=".">

    <!-- ── 1. Description ────────────────────────────── -->
    <description>
        Build file for MyApp — a sample Java application.
        Run 'ant -projecthelp' to list available targets.
    </description>

    <!-- ── 2. Property Definitions ───────────────────── -->
    <!-- Load external properties first (they win over defaults below) -->
    <property file="${user.home}/myapp-build.properties" />
    <property file="build.properties" />

    <!-- Directory layout -->
    <property name="src.dir"      value="src/main/java" />
    <property name="test.dir"     value="src/test/java" />
    <property name="res.dir"      value="src/main/resources" />
    <property name="lib.dir"      value="lib" />
    <property name="build.dir"    value="build" />
    <property name="classes.dir"  value="${build.dir}/classes" />
    <property name="test.classes" value="${build.dir}/test-classes" />
    <property name="dist.dir"     value="dist" />
    <property name="reports.dir"  value="${build.dir}/reports" />

    <!-- Build metadata -->
    <property name="app.version"  value="1.0.0" />
    <property name="jar.name"     value="myapp-${app.version}.jar" />
    <property name="main.class"   value="com.example.Main" />
    <property name="java.source"  value="17" />

    <!-- ── 3. Timestamp ──────────────────────────────── -->
    <tstamp>
        <format property="build.date" pattern="yyyy-MM-dd" />
        <format property="build.time" pattern="HH:mm:ss" />
    </tstamp>

    <!-- ── 4. Classpath Definitions ─────────────────── -->
    <path id="compile.classpath">
        <fileset dir="${lib.dir}/compile" includes="**/*.jar" />
    </path>

    <path id="test.classpath">
        <path refid="compile.classpath" />
        <fileset dir="${lib.dir}/test"    includes="**/*.jar" />
        <pathelement location="${classes.dir}" />
        <pathelement location="${test.classes}" />
    </path>

    <path id="runtime.classpath">
        <path refid="compile.classpath" />
        <fileset dir="${lib.dir}/runtime" includes="**/*.jar" />
        <pathelement location="${classes.dir}" />
    </path>

    <!-- ── 5. Targets ────────────────────────────────── -->

    <target name="init"
            description="Create build output directories">
        <mkdir dir="${classes.dir}" />
        <mkdir dir="${test.classes}" />
        <mkdir dir="${dist.dir}" />
        <mkdir dir="${reports.dir}" />
        <echo message="[${app.version}] Build initialised — ${build.date} ${build.time}" />
    </target>

    <target name="compile"
            depends="init"
            description="Compile main source code">
        <javac srcdir="${src.dir}"
               destdir="${classes.dir}"
               classpathref="compile.classpath"
               source="${java.source}"
               target="${java.source}"
               encoding="UTF-8"
               includeantruntime="false"
               debug="true"
               debuglevel="lines,vars,source" />
        <!-- Copy non-Java resources -->
        <copy todir="${classes.dir}">
            <fileset dir="${res.dir}" />
        </copy>
    </target>

    <target name="compile-tests"
            depends="compile"
            description="Compile test source code">
        <javac srcdir="${test.dir}"
               destdir="${test.classes}"
               classpathref="test.classpath"
               source="${java.source}"
               target="${java.source}"
               includeantruntime="false" />
    </target>

    <target name="test"
            depends="compile-tests"
            description="Run unit tests and generate HTML report">
        <junit printsummary="yes"
               haltonfailure="yes"
               fork="yes">
            <classpath refid="test.classpath" />
            <formatter type="xml" />
            <batchtest todir="${reports.dir}">
                <fileset dir="${test.classes}">
                    <include name="**/*Test.class" />
                </fileset>
            </batchtest>
        </junit>
        <junitreport todir="${reports.dir}">
            <fileset dir="${reports.dir}" includes="TEST-*.xml" />
            <report todir="${reports.dir}/html" />
        </junitreport>
    </target>

    <target name="package"
            depends="test"
            description="Create the distributable JAR">
        <jar destfile="${dist.dir}/${jar.name}"
             basedir="${classes.dir}">
            <manifest>
                <attribute name="Main-Class"               value="${main.class}" />
                <attribute name="Implementation-Version"   value="${app.version}" />
                <attribute name="Built-By"                 value="${user.name}" />
                <attribute name="Build-Date"               value="${build.date}" />
            </manifest>
        </jar>
        <echo message="Created: ${dist.dir}/${jar.name}" />
    </target>

    <target name="javadoc"
            depends="compile"
            description="Generate API documentation">
        <javadoc sourcepath="${src.dir}"
                 destdir="${build.dir}/javadoc"
                 classpathref="compile.classpath"
                 windowtitle="MyApp ${app.version} API"
                 access="protected" />
    </target>

    <target name="clean"
            description="Delete all generated build output">
        <delete dir="${build.dir}" />
        <delete dir="${dist.dir}" />
    </target>

    <target name="rebuild"
            depends="clean, package"
            description="Full clean build" />

    <target name="all"
            depends="clean, package, javadoc"
            description="Full build including Javadoc" />

</project>
```

---

## Viewing Available Targets

```bash
ant -projecthelp
# Buildfile: build.xml
#
# Main targets:
#  all          Full build including Javadoc
#  clean        Delete all generated build output
#  compile      Compile main source code
#  javadoc      Generate API documentation
#  package      Create the distributable JAR
#  rebuild      Full clean build
#  test         Run unit tests and generate HTML report
#
# Default target: package
```

Only targets with a `description` attribute appear in the listing.

---

| | |
|---|---|
| [← Chapter 2: Core Concepts](./02-core-concepts.md) | [Next → Chapter 4: Lifecycle & Execution Model](./04-lifecycle-execution.md) |
