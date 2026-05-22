# Java Performance Tuning (Fusion Middleware Context)

## Overview

Use this skill to tune Java runtime performance for Oracle Fusion Middleware workloads (WebLogic, SOA Suite, GoldenGate). For GC-specific tuning and the collector selection matrix, load `references/java-gc-reference.md`. This skill adds middleware-specific framing and broader JVM performance guidance.

## Tuning Workflow

1. Baseline: collect GC log, heap histogram, thread utilization, and CPU profile before changing anything.
2. Identify bottleneck: GC pause frequency, heap sizing, thread starvation, or I/O wait (JDBC, JMS, remote calls).
3. Apply one JVM change at a time; re-measure under representative load.
4. See `references/java-gc-reference.md` for collector selection and GC-specific flags.

## Heap Sizing

- Set `-Xms` equal to `-Xmx` in production to avoid heap resize pauses.
- For WebLogic Managed Servers: size heap per server; avoid a single large heap when multiple Managed Servers are possible on the same host.
- SOA Suite with XML-heavy workloads: plan for a higher allocation rate due to DOM and JAXB object churn; monitor live heap usage with `jstat -gcutil <pid> 5000`.
- Leave OS headroom: do not allocate more than ~75% of available RAM to JVM heap across all JVMs on the host.

## GC Selection and Tuning

Load `references/java-gc-reference.md` for the collector decision matrix and specific flags.

Middleware shorthand:
- Java 8 + WebLogic 12.2.1.4: G1GC is a safe choice; CMS is available but deprecated.
- Java 11+ + WebLogic 14.x/15c: G1GC is the default; ZGC or Shenandoah are options for latency-sensitive workloads.
- Java 8 SOA workloads: switch from ParallelGC to G1GC to reduce pause times for XML-heavy processing: `-XX:+UseG1GC -XX:MaxGCPauseMillis=200`.
- Never carry CMS flags forward to Java 14+; see the migration gotchas section in `references/java-gc-reference.md`.

## Thread Sizing

- Do not set `-Dweblogic.threadpool.size` manually in recent WebLogic versions; use Work Manager constraints instead.
- For SOA Suite: tune BPEL and Mediator thread counts via SOA MBeans, not JVM thread count flags.
- For GoldenGate JVM processes: monitor thread usage with `jcmd <pid> Thread.print` and `jstat -gcutil`.

## JVM Flag Hygiene

- Start with a minimal flag set and add flags only from measured evidence.
- After each major JDK upgrade, review all JVM flags for removed or deprecated options: check startup output for `WARNING: Using obsolete option`.
- Common flags that become obsolete at upgrade boundaries:
  - `-XX:MaxPermSize` / `-XX:PermSize` — removed in Java 8 (use `-XX:MaxMetaspaceSize`)
  - `-XX:+UseConcMarkSweepGC` — removed in Java 14
  - `-XX:+PrintGCDetails` — replaced by `-Xlog:gc*` in Java 9+

## JVM Profiling

Use JFR (Java Flight Recorder) for low-overhead production profiling:

```
jcmd <pid> JFR.start name=perf settings=profile duration=120s filename=perf.jfr
jcmd <pid> JFR.stop name=perf
```

Analyze JFR output with JDK Mission Control (JMC). Use `jstack` or `jcmd Thread.print` for quick thread-level snapshots before committing to a full JFR capture.

## Native Memory Monitoring

Enable native memory tracking when suspecting non-heap growth:

```
-XX:NativeMemoryTracking=summary
jcmd <pid> VM.native_memory summary
```

Common native memory growth causes in middleware: DirectByteBuffer use in NIO channels, JNI code, or accumulating class metadata from repeated deployments.

## Metaspace Tuning

- Set an explicit `MaxMetaspaceSize` to prevent unbounded native memory consumption in environments with many deployments:
  `-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m`
- Metaspace growth that is not bounded by application deployment count typically indicates a classloader leak; diagnose with heap dump analysis.

## Oracle Version Notes (19c vs 26ai)

- Separate JVM tuning from JDBC connection tuning; poor JDBC connection health (stale connections, pool exhaustion) can manifest as thread or memory pressure in the JVM.
- For 26ai database targets: validate JDBC driver version and connection behavior independently before attributing application-layer performance issues to JVM configuration.

## Reference Files

- `references/java-gc-reference.md` — collector selection, availability matrix, and flag reference
- `references/java-troubleshooting-reference.md` — JVM diagnostic commands
- `references/java-version-matrix.md` — LTS planning and version timeline
- `../weblogic/performance-tuning.md` — WebLogic-specific tuning
- `../soa/performance-tuning.md` — SOA Suite-specific tuning

## Sources

- `references/java-gc-reference.md`
- JDK 26 GC Tuning Guide: https://docs.oracle.com/en/java/javase/26/gctuning/index.html
- JDK 26 troubleshooting guide: https://docs.oracle.com/en/java/javase/26/troubleshoot/index.html
- JDK Mission Control: https://docs.oracle.com/en/java/java-components/jdk-mission-control/
- Oracle Java SE support roadmap: https://www.oracle.com/java/technologies/java-se-support-roadmap.html
