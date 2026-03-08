[← Chapter 10: Macros & Custom Tasks](./10-macros-and-custom-tasks.md) | [Index](../index.md)

---

# Chapter 11 — Ivy — Dependency Management for Ant

Apache Ivy is a standalone dependency manager that plugs into Ant. It resolves dependencies from Maven-compatible repositories using a Maven-like `ivy.xml` descriptor. Ivy is Ant's answer to Maven's built-in dependency management.

---

## Core Ivy Concepts

| Term | Definition |
|---|---|
| `ivy.xml` | Ivy's dependency descriptor (analogous to Maven's `pom.xml`) |
| `ivysettings.xml` | Configures Ivy's behaviour: resolvers, caches, patterns |
| **Resolver** | A component that knows how to locate and fetch artifacts (local filesystem, URL, Maven repo, etc.) |
| **Configuration** | A named scope that groups dependencies (like Maven's `compile`, `test`, `runtime` scopes) |
| **Organisation** | Ivy's term for Maven's `groupId` |
| **Module** | Ivy's term for Maven's `artifactId` |
| **Revision** | Ivy's term for Maven's `version` |
| **Retrieve** | Download resolved artifacts and copy them to a local directory |
| **Publish** | Upload your artifact to a repository |
| **Cache** | Ivy's local artifact cache (separate from Maven's `~/.m2`) |

---

## The `ivy.xml` File

```xml
<ivy-module version="2.0">
    <info organisation="com.example"
          module="my-application"
          revision="1.5.0"
          status="release" />

    <!--
        Configurations are like Maven scopes.
        'extends' means: this config includes everything from the listed config(s).
    -->
    <configurations>
        <conf name="compile"  description="Compile-time dependencies" />
        <conf name="runtime"  extends="compile" description="Runtime dependencies" />
        <conf name="test"     extends="runtime" description="Test-only dependencies" />
        <conf name="provided" description="Provided by the container" />
    </configurations>

    <dependencies>
        <!-- conf="compile->default": our 'compile' config uses the dep's 'default' config -->
        <dependency org="org.slf4j"               name="slf4j-api"       rev="2.0.9"
                    conf="compile->default" />

        <!-- Runtime logging implementation -->
        <dependency org="ch.qos.logback"          name="logback-classic"  rev="1.4.11"
                    conf="runtime->default" />

        <!-- JDBC driver: needed at runtime, not compile time -->
        <dependency org="org.postgresql"          name="postgresql"       rev="42.6.0"
                    conf="runtime->default" />

        <!-- Test-only dependencies -->
        <dependency org="org.junit.jupiter"       name="junit-jupiter"    rev="5.10.1"
                    conf="test->default" />
        <dependency org="org.mockito"             name="mockito-core"     rev="5.7.0"
                    conf="test->default" />

        <!-- Provided by servlet container -->
        <dependency org="jakarta.servlet"         name="jakarta.servlet-api" rev="6.0.0"
                    conf="provided->default" />

        <!-- Exclude a transitive dependency -->
        <dependency org="org.springframework" name="spring-core" rev="6.1.2"
                    conf="compile->default">
            <exclude org="commons-logging" name="commons-logging" />
        </dependency>
    </dependencies>

</ivy-module>
```

---

## The `ivysettings.xml` File

```xml
<ivysettings>
    <settings defaultResolver="chain" />

    <caches defaultCacheDir="${user.home}/.ivy2/cache" />

    <resolvers>
        <!--
            Chain resolver: tries each resolver in order.
            Returns the first successful result.
        -->
        <chain name="chain" returnFirst="true">

            <!-- 1. Local file system (fastest — check here first) -->
            <filesystem name="local-repo">
                <ivy  pattern="${user.home}/.ivy2/local/[organisation]/[module]/[revision]/ivy-[revision].xml" />
                <artifact pattern="${user.home}/.ivy2/local/[organisation]/[module]/[revision]/[artifact]-[revision].[ext]" />
            </filesystem>

            <!-- 2. Company Nexus (internal proxy) -->
            <ibiblio name="nexus"
                     root="https://nexus.company.com/repository/maven-public/"
                     m2compatible="true" />

            <!-- 3. Maven Central (fallback) -->
            <ibiblio name="central"
                     root="https://repo.maven.apache.org/maven2/"
                     m2compatible="true" />

        </chain>
    </resolvers>

    <!-- Separate resolver for publishing -->
    <resolvers>
        <url name="nexus-releases">
            <artifact pattern="https://nexus.company.com/repository/maven-releases/[organisation]/[module]/[revision]/[artifact]-[revision].[ext]" />
        </url>
    </resolvers>

</ivysettings>
```

---

## Common Ivy Targets in build.xml

```xml
<project xmlns:ivy="antlib:org.apache.ivy.ant">

    <!-- ── Bootstrap: download Ivy itself ────────────── -->
    <property name="ivy.version"  value="2.5.2" />
    <property name="ivy.jar"      value="${user.home}/.ant/lib/ivy-${ivy.version}.jar" />

    <target name="bootstrap-ivy">
        <mkdir dir="${user.home}/.ant/lib" />
        <get src="https://repo.maven.apache.org/maven2/org/apache/ivy/ivy/${ivy.version}/ivy-${ivy.version}.jar"
             dest="${ivy.jar}"
             skipexisting="true" />
    </target>

    <!-- ── Resolve: download dependencies ────────────── -->
    <target name="resolve" depends="bootstrap-ivy"
            description="Resolve and retrieve project dependencies">
        <ivy:settings file="ivysettings.xml" />
        <ivy:resolve file="ivy.xml" />
        <!-- Copy JARs to lib/[conf]/ directories -->
        <ivy:retrieve pattern="${lib.dir}/[conf]/[artifact]-[revision].[ext]"
                      sync="true"
                      type="jar,bundle" />
    </target>

    <!-- ── After resolve, build paths from retrieved JARs ── -->
    <target name="init-classpath" depends="resolve">
        <path id="compile.classpath">
            <fileset dir="${lib.dir}/compile"  includes="**/*.jar" />
            <fileset dir="${lib.dir}/provided" includes="**/*.jar" />
        </path>
        <path id="runtime.classpath">
            <fileset dir="${lib.dir}/runtime" includes="**/*.jar" />
            <pathelement location="${classes.dir}" />
        </path>
        <path id="test.classpath">
            <fileset dir="${lib.dir}/test"    includes="**/*.jar" />
            <path refid="runtime.classpath" />
        </path>
    </target>

    <!-- ── Or use ivy:cachepath directly (no file copy) ─── -->
    <target name="classpath-from-cache" depends="resolve">
        <ivy:cachepath pathid="compile.classpath" conf="compile" />
        <ivy:cachepath pathid="test.classpath"    conf="test" />
    </target>

    <!-- ── Report: show dependency graph ─────────────── -->
    <target name="ivy-report" depends="resolve"
            description="Generate dependency report">
        <ivy:report todir="${reports.dir}/ivy"
                    graph="true"
                    xml="true" />
    </target>

    <!-- ── Publish: deploy to repository ─────────────── -->
    <target name="publish" depends="package"
            description="Publish artifact to internal Nexus">
        <ivy:publish resolver="nexus-releases"
                     pubrevision="${app.version}"
                     status="release"
                     overwrite="false"
                     forcedeliver="true">
            <artifacts pattern="${dist.dir}/[artifact]-[revision].[ext]" />
        </ivy:publish>
    </target>

    <!-- ── Clean Ivy cache ────────────────────────────── -->
    <target name="ivy-clean-cache"
            description="Clear the local Ivy cache">
        <ivy:cleancache />
    </target>

</project>
```

---

## Ivy vs Maven Dependency Management

| Feature | Ivy | Maven |
|---|---|---|
| Configuration file | `ivy.xml` | `pom.xml` |
| Repository format | Maven-compatible | Maven native |
| Scopes/Configurations | Fully customisable | Fixed (compile/test/runtime/etc.) |
| Integration | Plugs into Ant | Built into Maven |
| Transitive deps | Yes | Yes |
| BOM import | Not native | Yes (`scope=import`) |
| Learning curve | Moderate | Low (for basic use) |

---

| | |
|---|---|
| [← Chapter 10: Macros & Custom Tasks](./10-macros-and-custom-tasks.md) | [Next → Chapter 12: Use Cases](./12-use-cases.md) |
