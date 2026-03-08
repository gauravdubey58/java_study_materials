[← Chapter 3: Build File Structure](./03-build-file-structure.md) | [Index](../index.md)

---

# Chapter 4 — Lifecycle & Execution Model

This is the most conceptually important chapter for anyone coming from Maven. Ant's execution model is fundamentally different — understanding it prevents confusion and mistakes.

---

## No Built-In Lifecycle

**Ant has no built-in lifecycle.**

There are no pre-defined phases like `compile`, `test`, or `package` that Ant knows about. There is no implicit ordering. If you want compilation to happen before testing, you must declare that dependency explicitly using the `depends` attribute.

This is the single biggest conceptual difference from Maven.

```xml
<!-- Maven: 'test' always runs after 'compile' — it's in the lifecycle -->

<!-- Ant: YOU define this relationship -->
<target name="test" depends="compile">   <!-- 'compile' runs first BECAUSE YOU SAID SO -->
    ...
</target>
```

---

## Dependency Graph (DAG)

Ant resolves target dependencies as a **Directed Acyclic Graph (DAG)**. Before running any target, Ant computes the full execution order by following all `depends` chains recursively.

### Example: Running `ant package`

```
Targets defined:
  init          (no depends)
  compile       depends="init"
  compile-tests depends="compile"
  test          depends="compile-tests"
  package       depends="test"

Execution when you run 'ant package':
  Ant computes: package → test → compile-tests → compile → init
  Reverses to get execution order:
  1. init
  2. compile
  3. compile-tests
  4. test
  5. package
```

If you run `ant compile`, only `init` and `compile` run. Ant does not know or care about `test` or `package`.

---

## Rules of Target Execution

**Each target runs at most once per build**, even if multiple targets depend on it:

```
If A depends on C, and B depends on C:
  ant A B    →  C runs once, then A, then B
```

**Targets always run in dependency order**, not in the order they appear in the build file.

**Circular dependencies cause an immediate error.**

---

## Conventional Target Names

Although Ant enforces nothing, the community has settled on these conventional names. Following them makes your build file understandable to other Ant users:

| Target Name | Convention |
|---|---|
| `init` | Create directories, set properties, load timestamps |
| `compile` | Compile main Java source code |
| `compile-tests` | Compile test source code |
| `test` | Run unit tests |
| `package` or `jar` | Create JAR artifact |
| `war` | Create WAR artifact |
| `dist` | Create distribution archive (ZIP, TAR) |
| `javadoc` | Generate API documentation |
| `clean` | Delete `build/` and `dist/` directories |
| `rebuild` | `clean` then full build |
| `all` | Run everything (build + docs + tests) |
| `deploy` | Deploy to application server or remote host |
| `check` | Run static analysis / code quality |
| `release` | Full release pipeline |

---

## Running Targets

```bash
ant                          # run the default target (set in <project default="...">)
ant compile                  # run 'compile' (and its dependencies)
ant clean package            # run 'clean', then 'package'
ant -f build.xml test        # explicit build file
ant -projecthelp             # list described targets
ant -verbose                 # verbose output (more detail)
ant -debug                   # debug output (maximum detail)
ant -quiet                   # suppress most output (errors only)
ant -keep-going              # continue even if a target fails (-k)

# Pass a property on the command line (overrides any <property> in build.xml)
ant -Denv=production deploy
ant -Ddebug=false compile
```

---

## Importing Other Build Files

Large projects can split build logic across multiple files using `<import>`:

```xml
<!-- build.xml -->
<project name="MyApp" default="package">
    <import file="build-common.xml" />      <!-- shared targets/properties -->
    <import file="build-deploy.xml" />      <!-- deployment targets -->
    ...
</project>
```

Imported targets become part of the same project. If a name conflicts, the importing file wins.

---

## Calling Sub-Builds

```xml
<!-- Call a target in a completely separate build file -->
<ant dir="modules/core" antfile="build.xml" target="package" inheritAll="false">
    <property name="version" value="${app.version}" />
</ant>

<!-- Call targets in multiple sub-directories -->
<subant target="compile">
    <fileset dir="." includes="modules/*/build.xml" />
</subant>
```

`inheritAll="false"` is strongly recommended — it prevents your parent build's properties from polluting the child build's environment.

---

| | |
|---|---|
| [← Chapter 3: Build File Structure](./03-build-file-structure.md) | [Next → Chapter 5: Data Types](./05-data-types.md) |
