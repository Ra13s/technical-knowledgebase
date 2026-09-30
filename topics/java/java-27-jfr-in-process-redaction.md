# Java 27 JFR in-process redaction

## What it is

JDK 27 adds in-process Java Flight Recorder (JFR) filtering for selected startup arguments, environment variables, and system properties before matching values are written into supported JFR events.

## Use when

Use this as defense in depth when JFR recordings are exported, attached to incidents, or retained outside the JVM.

## Implementation

Inspect the runtime defaults first:

```bash
java -XX:FlightRecorderOptions:help
```

JDK 27 provides two relevant filters:

- `redact-key` for matching environment-variable and system-property names;
- `redact-argument` for matching command-line arguments.

Both accept case-insensitive glob patterns with `*` and `?`, can load patterns from a file with `@filename`, and use a leading `+` to extend rather than replace the runtime defaults.

Keep the defaults, add organization-specific patterns only where needed, then create a short non-production recording containing harmless canary values and verify the supported startup events contain `[REDACTED]` rather than the canary.

## Coverage limits

This is best-effort filtering, not universal recording sanitization.

`redact-argument` covers matching startup data in `jdk.JVMInformation`, `java.command` in `jdk.InitialSystemProperty`, and matching argument text in `jdk.InitialEnvironmentVariable`. It does not cover every event; for example, child-process command lines in `jdk.ProcessStart` are outside this filter.

`redact-key` covers `jdk.InitialSystemProperty`, `jdk.InitialEnvironmentVariable`, and `-D` values in `jdk.JVMInformation`. It does not cover `jdk.InitialSecurityProperty`.

Application-defined JFR events need their own data-minimization rules.

## Decision rules

- Prefer the runtime defaults plus small additions rather than replacing the defaults.
- Canary-test the exact filter set during a JDK 27 rollout.
- Keep recordings protected as sensitive operational data even after redaction.
- Prefer avoiding sensitive startup arguments entirely when possible.

## Why it is useful

Once sensitive startup data enters a recording, every downstream export, support bundle, and retention system has to protect it correctly. In-process filtering removes common values at the earliest JFR boundary.

## Version

JDK 27 / JEP 536.

## Sources

- Oracle JDK 27 `java` command: https://docs.oracle.com/en/java/javase/27/docs/specs/man/java.html
- Inside Java, *JDK 27 Runtime Updates Release Notes* (2026-09-12): https://inside.java/2026/09/12/jdk-27-runtime-updates/
- OpenJDK JEP 536: https://openjdk.org/jeps/536
