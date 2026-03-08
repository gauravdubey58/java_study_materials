[← Chapter 11: Ivy — Dependency Management](./11-ivy-dependency-management.md) | [Index](../index.md)

---

# Chapter 12 — Common Use Cases

Practical, copy-paste-ready recipes for the most frequent Ant build scenarios.

---

## Building a Fat JAR (Uber-JAR)

An executable JAR that includes all dependencies merged inside it:

```xml
<target name="fat-jar" depends="package" description="Create self-contained executable JAR">
    <jar destfile="${dist.dir}/myapp-fat.jar">
        <!-- Main classes -->
        <fileset dir="${classes.dir}" />
        <!-- Unpack all runtime dependency JARs into the fat JAR -->
        <zipgroupfileset dir="${lib.dir}/runtime" includes="**/*.jar" />
        <!-- Exclude signature files (required for signed JARs) -->
        <zipfileset excludes="META-INF/*.SF,META-INF/*.DSA,META-INF/*.RSA" />
        <manifest>
            <attribute name="Main-Class" value="${main.class}" />
        </manifest>
    </jar>
    <echo message="Fat JAR: ${dist.dir}/myapp-fat.jar" />
</target>
```

---

## Environment-Specific Builds

Load the right configuration based on a command-line property:

```xml
<property name="env" value="dev" />   <!-- default environment -->

<target name="load-env-config">
    <echo message="Loading config for environment: ${env}" />
    <property file="config/${env}.properties" />
    <fail unless="db.url" message="config/${env}.properties must define db.url" />
</target>

<target name="deploy" depends="package, load-env-config">
    <scp file="${dist.dir}/${jar.name}"
         todir="${deploy.user}@${deploy.host}:${deploy.dir}"
         password="${deploy.password}" trust="true" />
</target>
```

```bash
ant -Denv=production deploy
ant -Denv=staging    deploy
ant deploy           # uses dev defaults
```

---

## OS-Conditional Builds

```xml
<condition property="exec.ext" value=".bat" else="">
    <os family="windows" />
</condition>
<condition property="path.sep" value=";" else=":">
    <os family="windows" />
</condition>

<target name="run" depends="package">
    <exec executable="${scripts.dir}/start${exec.ext}" failonerror="true">
        <arg value="${app.version}" />
    </exec>
</target>
```

---

## Capturing Git Metadata in Builds

```xml
<target name="git-info" description="Capture Git metadata">
    <exec executable="git" outputproperty="git.commit.hash" failonerror="true">
        <arg line="rev-parse --short HEAD" />
    </exec>
    <exec executable="git" outputproperty="git.branch" failonerror="true">
        <arg line="rev-parse --abbrev-ref HEAD" />
    </exec>
    <exec executable="git" outputproperty="git.commit.message" failonerror="true">
        <arg line="log -1 --pretty=%B" />
    </exec>

    <propertyfile file="${classes.dir}/build-info.properties">
        <entry key="version"     value="${app.version}" />
        <entry key="git.hash"    value="${git.commit.hash}" />
        <entry key="git.branch"  value="${git.branch}" />
        <entry key="build.date"  type="date" value="now" pattern="yyyy-MM-dd'T'HH:mm:ss" />
        <entry key="built.by"    value="${user.name}" />
    </propertyfile>
</target>
```

---

## Multi-Module Build (Manual Coordination)

```xml
<property name="modules" value="core,service,web" />

<target name="build-all" description="Build all modules in order">
    <!-- Build each module's 'package' target -->
    <ant dir="modules/core"    target="package" inheritAll="false">
        <property name="app.version" value="${app.version}" />
    </ant>
    <ant dir="modules/service" target="package" inheritAll="false">
        <property name="app.version" value="${app.version}" />
        <!-- Make core classes available to service -->
        <property name="core.classes" value="${basedir}/modules/core/build/classes" />
    </ant>
    <ant dir="modules/web"     target="package" inheritAll="false">
        <property name="app.version" value="${app.version}" />
    </ant>
    <echo message="All modules built successfully." />
</target>

<target name="clean-all" description="Clean all modules">
    <subant target="clean">
        <fileset dir="modules" includes="*/build.xml" />
    </subant>
</target>
```

---

## Deploying to Tomcat

```xml
<property name="tomcat.home" value="/opt/tomcat" />
<property name="deploy.dir"  value="${tomcat.home}/webapps" />

<target name="deploy-tomcat" depends="package">
    <!-- Stop Tomcat -->
    <exec executable="${tomcat.home}/bin/shutdown.sh" osfamily="unix" failonerror="false" />

    <!-- Wait for shutdown -->
    <sleep seconds="5" />

    <!-- Copy WAR -->
    <copy file="${dist.dir}/myapp.war"
          todir="${deploy.dir}"
          overwrite="true" />

    <!-- Start Tomcat -->
    <exec executable="${tomcat.home}/bin/startup.sh" osfamily="unix" />
    <echo message="Deployed to Tomcat at ${deploy.dir}" />
</target>
```

---

## Generating a Versioned Release Archive

```xml
<target name="release-zip" depends="package, javadoc"
        description="Create a versioned release archive">
    <zip destfile="${dist.dir}/myapp-${app.version}-release.zip">
        <zipfileset src="${dist.dir}/${jar.name}"
                    prefix="myapp-${app.version}" />
        <zipfileset dir="${build.dir}/javadoc"
                    prefix="myapp-${app.version}/docs/api" />
        <zipfileset dir="."
                    includes="README.md,LICENSE.txt,CHANGELOG.md"
                    prefix="myapp-${app.version}" />
    </zip>
    <checksum file="${dist.dir}/myapp-${app.version}-release.zip"
              algorithm="SHA-256"
              property="release.sha256" />
    <echo file="${dist.dir}/myapp-${app.version}-release.zip.sha256"
          message="${release.sha256}" />
    <echo message="Release archive created: myapp-${app.version}-release.zip" />
    <echo message="SHA-256: ${release.sha256}" />
</target>
```

---

| | |
|---|---|
| [← Chapter 11: Ivy — Dependency Management](./11-ivy-dependency-management.md) | [Next → Chapter 13: Quick Reference](./13-quick-reference.md) |
