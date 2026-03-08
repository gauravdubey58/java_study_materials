# Index — Build Tools Notes

> Full table of contents for both the Maven and Ant books. Use this page as your starting point or as a map to jump directly to any topic.

---

## 📘 Book 1 — Apache Maven

| # | Chapter | Key Topics |
|---|---|---|
| 1 | [Introduction to Maven](./maven/01-introduction.md) | What Maven is, philosophy, history, installation |
| 2 | [Core Concepts & Terminology](./maven/02-core-concepts.md) | Artifact, GAV coordinates, POM, Mojo, Goal, Phase, Lifecycle, Plugin, Repository, BOM, Classifier |
| 3 | [Project Object Model (POM)](./maven/03-pom.md) | pom.xml anatomy, inheritance, Super POM, Effective POM, properties, full annotated example |
| 4 | [Build Lifecycles](./maven/04-lifecycles.md) | Default, Clean, and Site lifecycles with all phases explained |
| 5 | [Phases & Goals](./maven/05-phases-and-goals.md) | Running phases vs goals, phase-to-goal bindings, goal syntax |
| 6 | [Dependency Management](./maven/06-dependency-management.md) | Scopes, transitive deps, version conflict resolution, exclusions, optional deps, BOM import |
| 7 | [Repositories](./maven/07-repositories.md) | Local, Central, Remote, private repo managers, mirrors, settings.xml |
| 8 | [Plugins](./maven/08-plugins.md) | Plugin anatomy, 20+ core and ecosystem plugins with full config examples |
| 9 | [Profiles](./maven/09-profiles.md) | Profile declaration, all 5 activation strategies, environment-specific builds |
| 10 | [Multi-Module Projects](./maven/10-multi-module.md) | Parent POM, child modules, aggregation vs inheritance, reactor build order |
| 11 | [Common Use Cases](./maven/11-use-cases.md) | Skipping tests, parallel tests, Docker, releasing, offline builds, debugging |
| 12 | [Quick Reference](./maven/12-quick-reference.md) | Cheat sheet of most-used commands and flags |

---

## 📙 Book 2 — Apache Ant

| # | Chapter | Key Topics |
|---|---|---|
| 1 | [Introduction to Ant](./ant/01-introduction.md) | What Ant is, philosophy, history, installation, vs Maven |
| 2 | [Core Concepts & Terminology](./ant/02-core-concepts.md) | Build file, Project, Target, Task, Antlib, Property, FileSet, Mapper, Filter, Listener |
| 3 | [Build File Structure](./ant/03-build-file-structure.md) | Full annotated build.xml, project element, paths, targets layout |
| 4 | [Lifecycle & Execution Model](./ant/04-lifecycle-execution.md) | No built-in lifecycle, DAG resolution, conventional target names, running targets |
| 5 | [Data Types](./ant/05-data-types.md) | fileset, path, patternset, dirset, filelist, mapper, filterset, filterchain |
| 6 | [Built-in Tasks](./ant/06-built-in-tasks.md) | 40+ tasks: file ops, javac, java, jar/war/zip, echo, exec, XML/property tasks |
| 7 | [Antlibs & Optional Tasks](./ant/07-antlibs.md) | JUnit, Checkstyle, SpotBugs, JaCoCo, Ant-Contrib, SSH/SCP/FTP, xmltask |
| 8 | [Properties & Filtering](./ant/08-properties-and-filtering.md) | Immutability, property files, environment variables, token replacement, filtersets |
| 9 | [Conditions & Control Flow](./ant/09-conditions-and-control-flow.md) | condition types, if/unless on targets, fail, available, uptodate |
| 10 | [Macros & Custom Tasks](./ant/10-macros-and-custom-tasks.md) | macrodef, scriptdef, writing a Java custom task, taskdef |
| 11 | [Ivy — Dependency Management](./ant/11-ivy-dependency-management.md) | ivy.xml, ivysettings.xml, resolvers, configurations, retrieve, publish |
| 12 | [Common Use Cases](./ant/12-use-cases.md) | Fat JAR, OS-conditional builds, multi-module, deploy, build info, env properties |
| 13 | [Quick Reference](./ant/13-quick-reference.md) | Cheat sheet of most-used commands, attributes, and patterns |

---

[← Back to README](./README.md)
