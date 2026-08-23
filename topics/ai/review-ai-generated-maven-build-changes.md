# Review AI-generated Maven build changes

## What it is

A compact review recipe for changes made by coding agents to Maven build configuration. The goal is to inspect the **resolved build**, not only the lines the agent changed in `pom.xml`.

## Use when

Run this whenever an AI coding agent adds or changes any of the following:

- dependencies or BOMs
- Maven plugins
- parent POMs
- repositories or plugin repositories
- profiles
- Maven settings/mirrors
- wrapper/build tooling

This is especially important when the agent introduces a dependency in an ecosystem the reviewer does not know well.

## How to use it

After the agent changes the build, run:

```bash
mvn dependency:tree
mvn help:effective-pom
mvn help:effective-settings
mvn dependency:resolve-plugins
```

Review each output for a different class of change:

### `mvn dependency:tree`

Inspect the resolved application dependency graph, including transitive dependencies. Check that the intended library/version is actually selected and look for unexpected additions or conflict-resolution changes.

### `mvn help:effective-pom`

Inspect the effective POM after inheritance and active profiles. This exposes build configuration that may not be obvious from the local `pom.xml`, including inherited plugins, repositories, dependency management and parent configuration.

For provenance of inherited elements, use verbose mode when useful:

```bash
mvn help:effective-pom -Dverbose
```

### `mvn help:effective-settings`

Inspect merged Maven settings for mirrors, profiles, proxies and repository configuration actually affecting the build.

Do not enable password display when capturing this output for logs or agent context.

### `mvn dependency:resolve-plugins`

Resolve/report project plugins and their dependencies. A Maven plugin executes as part of the build lifecycle, so treat a newly introduced plugin as executable build tooling, not merely another library.

## Agent rule

A useful harness instruction is:

```text
If you modify pom.xml, a parent/BOM, Maven plugin, repository, profile, settings, or wrapper configuration, do not finish after tests pass. Inspect the resolved dependency tree, effective POM, effective settings, and plugin dependencies. Report every newly introduced direct dependency, transitive dependency, plugin, repository, parent/BOM, or dynamic/SNAPSHOT version and justify why it is needed.
```

For high-risk repositories, make unexplained build-graph additions a submission blocker.

## Why it is useful

Reviewing only the textual POM diff misses important supply-chain changes:

- one direct dependency can bring many transitive artifacts;
- parent POMs/BOMs/profiles can alter versions and configuration elsewhere;
- Maven plugins can execute code during the build;
- mirrors/settings can change where artifacts are fetched from.

The commands above give both humans and agents a repeatable way to inspect what Maven will actually use.

## Caveats / when not to use

- These commands reveal configuration and resolved artifacts; they do not prove that a dependency is safe.
- Follow with vulnerability, provenance and support/EOL checks according to project policy.
- `effective-settings` can contain sensitive configuration; Maven hides passwords by default. Do not ask agents to expose secrets.
- Multi-module builds may produce large output. Scope/filter results for review, but do not skip modules affected by the change.

## Sources

- https://foojay.io/today/vibe-coding-maven-and-the-dependencies-you-didnt-choose/
- https://maven.apache.org/plugins/maven-dependency-plugin/usage.html
- https://maven.apache.org/plugins/maven-dependency-plugin/resolve-plugins-mojo.html
- https://maven.apache.org/plugins/maven-help-plugin/
- https://maven.apache.org/plugins/maven-help-plugin/effective-pom-mojo.html
- https://maven.apache.org/plugins/maven-help-plugin/effective-settings-mojo.html
- Discovery: https://www.baeldung.com/java-weekly-660
