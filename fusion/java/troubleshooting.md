# Java Troubleshooting (Fusion Middleware Context)

## Overview

Use this skill to diagnose Java runtime issues in Oracle Fusion Middleware environments (WebLogic, SOA Suite, GoldenGate). For raw JVM commands and artifact-level diagnostics, load `references/java-troubleshooting-reference.md`. This skill adds middleware-specific context and routing guidance.

## Fast Triage Workflow

1. Confirm Java version and runtime: `java -version`, startup script, `JAVA_HOME`.
2. Classify issue: startup failure, memory/GC pressure, thread starvation, classloading, or module error.
3. Load `references/java-troubleshooting-reference.md` for the matching diagnostic section.
4. Correlate with the middleware container log (WebLogic Managed Server, SOA Managed Server) for application context.

## Startup Failures in Middleware

Common causes:
- Obsolete or removed JVM flags after a JDK upgrade (especially after Java 9, 14, or 17 boundary)
- Module accessibility errors (`--illegal-access` removed in Java 17+)
- JDK version incompatible with the installed middleware version

Diagnostic steps:
- Run the middleware startup script directly from the command line to capture full JVM output
- Look for `Unrecognized option:` or `Error occurred during initialization of VM` in the output
- Use `java -XshowSettings:all -version` to confirm effective JVM settings
- Cross-reference the JDK version with middleware compatibility using `references/java-version-matrix.md`

## OutOfMemoryError in Middleware

Classify by OOM message — see `references/java-troubleshooting-reference.md` for diagnostic commands.

Middleware-specific context:
- `Java heap space`: common in SOA Suite workloads processing large XML payloads; check BPEL instance state retention and dehydration frequency
- `Metaspace`: common after repeated hot-deploy cycles in WebLogic; check for classloader leaks in application code
- `unable to create new native thread`: check OS-level thread limits (`ulimit -u`) and the total number of running Managed Servers on the host

## Thread Issues in Middleware

- WebLogic stuck threads: correlate `BEA-000337` log entries with thread dump output; see `../weblogic/troubleshooting.md`
- SOA dehydration blocking: identify threads blocked on SOAINFRA JDBC calls in thread dumps
- GoldenGate JVM threads: apply `jcmd <pid> Thread.print` to the GoldenGate JVM process when Extract or Replicat JVM hangs are suspected

## Classloading and Module Issues

Middleware-specific context:
- WebLogic uses a hierarchical classloader; application classes can shadow or conflict with system classes
- SOA composite classloading errors typically appear as `ClassNotFoundException` for adapter or schema classes
- After a JDK 9+ migration: split packages between application and system classloaders cause `IllegalAccessError`

Use diagnostic commands from `references/java-troubleshooting-reference.md`:
- `jdeps --jdk-internals <app.jar>`
- `java --show-module-resolution ...`
- Add `--add-exports` or `--add-opens` only after identifying the specific module boundary violation; do not use broadly

## Version-Specific Middleware Notes

- WebLogic 11g (10.3.6) + Java 6/7: PermGen-based memory; use `jmap -heap` and PermGen flags; see `references/java-troubleshooting-reference.md`.
- WebLogic 12.2.1.4 + Java 8 or 11: Metaspace applies; CMS GC is available on Java 8, removed in Java 14+.
- WebLogic 14.1.2 and 15c + Java 17/21: strong module encapsulation enforced; audit `--add-opens` entries at each JDK major upgrade.
- SOA 14.1.2 + Java 17: validate all JCA adapters for module compatibility before migration.

## Reference Files

- `references/java-troubleshooting-reference.md` — JVM commands, OOM classification, thread and classloading diagnostics
- `references/java-version-matrix.md` — Java version timeline and LTS planning
- `references/java-gc-reference.md` — GC collector availability and migration gotchas
- `../weblogic/troubleshooting.md` — WebLogic-specific troubleshooting

## Oracle Version Notes (19c vs 26ai)

When Java troubleshooting intersects with Oracle Database connectivity:
- JDBC driver version compatibility must be validated separately from JVM troubleshooting.
- For 26ai database targets: verify the JDBC driver supports the target database version before attributing connectivity issues to JVM configuration.

## Sources

- `references/java-troubleshooting-reference.md`
- `references/java-version-matrix.md`
- JDK 26 troubleshooting guide: https://docs.oracle.com/en/java/javase/26/troubleshoot/index.html
- Oracle Java SE support roadmap: https://www.oracle.com/java/technologies/java-se-support-roadmap.html
