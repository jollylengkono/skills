# Java Troubleshooting Reference (Java 6 to Latest)

## Scope

Use this reference to diagnose JVM runtime and startup issues across Java 6 through latest.
As of 2026-05-20, treat Java 26 as latest feature line and Java 25 as latest LTS line.

## Fast Triage Workflow

1. Confirm runtime and flags (`java -version`, launch args).
2. Classify issue: startup, memory/GC, thread/CPU, classloading/module.
3. Capture low-risk diagnostics first (`jcmd`, thread dump, heap summary).
4. Escalate to richer artifacts (heap dump, JFR, core dump analysis) only when needed.

## Startup Failures

Common symptoms:
- `Could not create the Java Virtual Machine`
- obsolete/removed VM options after upgrade
- module resolution issues in Java 9+

Useful commands:
- `java -version`
- `java -XshowSettings:all -version`
- `jdeps --jdk-internals <app.jar>`

## Memory and GC Incidents

Live checks:
- `jcmd <pid> GC.heap_info`
- `jcmd <pid> GC.class_histogram`
- `jstat -gcutil <pid> 1000`

Artifacts:
- `jcmd <pid> GC.heap_dump /path/heap.hprof`
- Enable `-XX:+HeapDumpOnOutOfMemoryError`

## Thread and CPU Diagnostics

Commands:
- `jcmd <pid> Thread.print -l`
- `jstack -l <pid>`
- JFR capture:
- `jcmd <pid> JFR.start name=triage settings=profile duration=120s filename=triage.jfr`
- `jcmd <pid> JFR.stop name=triage`

## OOM Triage

Classify quickly:
- Java heap
- GC overhead limit
- Metaspace/compressed class space
- Native memory

Native memory path:
- start with `-XX:NativeMemoryTracking=summary`
- run `jcmd <pid> VM.native_memory summary`

## Classloading and Modules

Typical symptoms:
- `ClassNotFoundException` / `NoClassDefFoundError`
- illegal reflective access or module export/open failures

Commands:
- `jdeps --recursive <app.jar>`
- `java --show-module-resolution ...`
- targeted compatibility flags: `--add-exports`, `--add-opens`

## Version-Specific Notes

- Java 6/7/8: legacy tool-driven diagnostics (`jstack`, `jstat`, `jmap`, `jps`) and PermGen-era behavior.
- Java 9+: module system and strong encapsulation drive many migration/runtime issues.
- Java 14+: CMS removed; stale CMS flags can break startup.

## Sources

- Oracle Java SE support roadmap: https://www.oracle.com/java/technologies/java-se-support-roadmap.html
- JDK 26 docs home: https://docs.oracle.com/en/java/javase/26/
- JDK 26 troubleshooting guide: https://docs.oracle.com/en/java/javase/26/troubleshoot/index.html
- JDK 26 migration guide: https://docs.oracle.com/en/java/javase/26/migrate/index.html
- JDK 26 man pages index: https://docs.oracle.com/en/java/javase/26/docs/specs/man/index.html
- Java command man page: https://docs.oracle.com/en/java/javase/26/docs/specs/man/java.html
- jcmd man page: https://docs.oracle.com/en/java/javase/26/docs/specs/man/jcmd.html
- jdeps man page: https://docs.oracle.com/en/java/javase/26/docs/specs/man/jdeps.html
- Java 6 tools overview: https://docs.oracle.com/javase/6/docs/technotes/tools/
- JEP 261 (module system): https://openjdk.org/jeps/261
- JEP 403 (strong encapsulation): https://openjdk.org/jeps/403
- JEP 363 (remove CMS): https://openjdk.org/jeps/363
