[← Chapter 6: Dependency Management](./06-dependency-management.md) | [Index](../index.md)

---

# Chapter 7 — Repositories

A **repository** is a storage location for Maven artifacts. When Maven needs a dependency, it searches repositories in a defined order. Understanding this lookup chain is key to troubleshooting missing artifacts and configuring enterprise setups.

---

## The Repository Lookup Chain

```
Build requests artifact X
        │
        ▼
1. Local Repository (~/.m2/repository)
        │ not found?
        ▼
2. Repositories declared in pom.xml
        │ not found?
        ▼
3. Repositories declared in settings.xml
        │ not found?
        ▼
4. Maven Central (default)
        │ not found?
        ▼
BUILD FAILURE — dependency not found
```

---

## Local Repository

The **local repository** is a cache on your machine. Every artifact Maven downloads is stored here, so subsequent builds don't need to re-download it.

**Default location:** `~/.m2/repository`

```
~/.m2/
├── repository/
│   ├── org/
│   │   └── springframework/
│   │       └── spring-core/
│   │           └── 6.1.2/
│   │               ├── spring-core-6.1.2.jar
│   │               ├── spring-core-6.1.2.pom
│   │               └── spring-core-6.1.2.jar.sha1
│   └── com/
│       └── google/
│           └── guava/
└── settings.xml            ← user-specific Maven settings
```

### Customising the Local Repository Path

In `~/.m2/settings.xml`:
```xml
<settings>
    <localRepository>/data/maven-repo</localRepository>
</settings>
```

Or per-build:
```bash
mvn install -Dmaven.repo.local=/tmp/ci-repo
```

---

## Maven Central

**Maven Central** is the default public repository that Maven queries when artifacts aren't found locally. It hosts hundreds of thousands of open-source Java libraries.

- URL: `https://repo.maven.apache.org/maven2`
- Search: https://search.maven.org
- No credentials required — it's publicly accessible

Maven Central is defined in the **Super POM**, so it is always available without any configuration.

---

## Remote Repositories in pom.xml

You can add extra repositories for libraries not hosted on Maven Central (e.g., Spring Milestones, JBoss, GitHub Packages):

```xml
<repositories>
    <repository>
        <id>spring-milestones</id>
        <name>Spring Milestones</name>
        <url>https://repo.spring.io/milestone</url>
        <releases>
            <enabled>true</enabled>
            <updatePolicy>never</updatePolicy>
        </releases>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>

    <repository>
        <id>jboss-public</id>
        <url>https://repository.jboss.org/nexus/content/groups/public</url>
    </repository>
</repositories>

<!-- Separate section for plugin repositories -->
<pluginRepositories>
    <pluginRepository>
        <id>central</id>
        <url>https://repo.maven.apache.org/maven2</url>
    </pluginRepository>
</pluginRepositories>
```

### Update Policies

The `<updatePolicy>` element controls how often Maven checks for newer SNAPSHOTs:

| Value | Behaviour |
|---|---|
| `always` | Check on every build |
| `daily` | Check once per day (default) |
| `interval:60` | Check every 60 minutes |
| `never` | Never check — always use cached version |

---

## Private Repository Managers

In enterprise environments, teams host a private repository manager that acts as a **proxy and cache** for all external repositories. Developers only talk to the internal server.

### Why Use a Repository Manager?

- **Speed** — cached artifacts are served from your internal network
- **Reliability** — builds don't fail because Maven Central is down
- **Security** — control which libraries are allowed (licence compliance, vulnerability scanning)
- **Private artifacts** — publish your own company libraries internally

### Popular Repository Managers

| Product | Vendor | Notes |
|---|---|---|
| **Nexus Repository Manager** | Sonatype | Most common in Java shops. Free OSS edition available |
| **JFrog Artifactory** | JFrog | Supports many package formats. Free and enterprise editions |
| **Apache Archiva** | Apache | Simple, free, open-source |

---

## Configuring a Mirror

A **mirror** redirects all Maven repository requests to a different URL. In enterprise setups, you set the internal Nexus/Artifactory as a mirror for `*` (all repositories):

In `~/.m2/settings.xml`:
```xml
<settings>
    <mirrors>
        <mirror>
            <id>company-nexus</id>
            <name>Company Nexus</name>
            <mirrorOf>*</mirrorOf>           <!-- mirror ALL repositories -->
            <url>https://nexus.company.com/repository/maven-public/</url>
        </mirror>
    </mirrors>
</settings>
```

`mirrorOf` patterns:

| Pattern | Mirrors |
|---|---|
| `*` | All repositories |
| `central` | Maven Central only |
| `!central,*` | All except Central |
| `external:*` | All except localhost |

---

## Authentication

For repositories requiring credentials, configure them in `settings.xml` (never in `pom.xml` — that would expose passwords in source control):

```xml
<settings>
    <servers>
        <server>
            <!-- Must match the <id> in the repository declaration -->
            <id>nexus-releases</id>
            <username>deploy-user</username>
            <password>secret123</password>
        </server>
        <server>
            <id>nexus-snapshots</id>
            <username>deploy-user</username>
            <password>secret123</password>
        </server>
    </servers>
</settings>
```

For CI/CD systems, use environment variables or Maven password encryption:
```bash
mvn --encrypt-master-password myMasterPassword
mvn --encrypt-password myRepositoryPassword
```

---

## Deploying to a Repository

To publish your artifact to a remote repository, configure `<distributionManagement>` in the POM:

```xml
<distributionManagement>
    <repository>
        <id>nexus-releases</id>   <!-- must match server id in settings.xml -->
        <url>https://nexus.company.com/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
        <id>nexus-snapshots</id>
        <url>https://nexus.company.com/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

Then run:
```bash
mvn deploy      # packages, installs locally, and deploys to remote repository
```

Maven automatically routes SNAPSHOT versions to the `<snapshotRepository>` and release versions to `<repository>`.

---

## Manually Installing a JAR into the Local Repository

When you have a JAR that isn't in any public repository:

```bash
mvn install:install-file \
  -Dfile=/path/to/vendor.jar \
  -DgroupId=com.vendor \
  -DartifactId=vendor-lib \
  -Dversion=2.0.0 \
  -Dpackaging=jar
```

This copies the JAR into `~/.m2/repository` so other local projects can declare it as a dependency.

---

| | |
|---|---|
| [← Chapter 6: Dependency Management](./06-dependency-management.md) | [Next → Chapter 8: Plugins](./08-plugins.md) |
