# Java Garbage Collection Reference (Java 6 to Java 26)

## Scope

Use this reference for operational GC choices and upgrade planning across Java 6 through the current GA release line.

Latest checked: **Java 26 GA (March 17, 2026)**.

## Collector Timeline (What Exists When)

| Collector | Java 6-8 | Java 9-13 | Java 14-20 | Java 21-23 | Java 24-26 | Notes |
|---|---|---|---|---|---|---|
| Serial (`-XX:+UseSerialGC`) | Yes | Yes | Yes | Yes | Yes | Small/simple workloads |
| Parallel/Throughput (`-XX:+UseParallelGC`) | Yes | Yes | Yes | Yes | Yes | Throughput-oriented |
| CMS (`-XX:+UseConcMarkSweepGC`) | Yes | Yes (deprecated in 9) | **Removed in 14+** | No | No | Replace with G1/ZGC/Shenandoah |
| G1 (`-XX:+UseG1GC`) | Available | Available | Available | Available | Available | Default since Java 9 on server-class machines |
| Epsilon (`-XX:+UseEpsilonGC`) | No | Added in 11 (experimental) | Yes | Yes | Yes | No-op GC; exits on heap exhaustion |
| ZGC (`-XX:+UseZGC`) | No | Added in 11 (experimental) | Product since 15 | Generational introduced in 21 | Non-generational mode removed in 24 | Low-latency focus |
| Shenandoah (`-XX:+UseShenandoahGC`) | No | Added in 12 (experimental) | Product since 15 | Yes | Yes | Low-pause concurrent collector |

Note: exact availability can vary by JDK distribution/platform; verify for your runtime build.

## Default Collector by Era

| Era | Default on server-class machines | Server-class criterion in docs |
|---|---|---|
| Java 6-8 | Parallel/Throughput | Java 8 docs: >=2 CPUs and >=2 GB RAM |
| Java 9-26 | G1 | Java 26 docs: >=2 CPUs and >=1792 MB RAM |
| Non-server-class (all eras) | Typically Serial | Ergonomics fallback |

## Tuning Knobs (Operational Shortlist)

Start with defaults, then tune only from evidence (GC logs + latency/throughput SLOs).

### Cross-collector

- Heap bounds: `-Xms`, `-Xmx`
- Pause target: `-XX:MaxGCPauseMillis=<ms>`
- Throughput target: `-XX:GCTimeRatio=<n>`
- Worker threads: `-XX:ParallelGCThreads=<n>`, `-XX:ConcGCThreads=<n>`

### Logging

- Java 8 and older style: `-XX:+PrintGCDetails -Xloggc:<file>`
- Java 9+ unified logging: `-Xlog:gc*[:file=...]`

### G1

- Start marking threshold: `-XX:InitiatingHeapOccupancyPercent=<pct>`
- Region size: `-XX:G1HeapRegionSize=<size>`
- Optional memory reduction for duplicated strings: `-XX:+UseStringDeduplication`

### CMS (legacy only)

- Trigger threshold: `-XX:CMSInitiatingOccupancyFraction=<pct>`
- Fixed trigger policy: `-XX:+UseCMSInitiatingOccupancyOnly`

### ZGC

- Enable: `-XX:+UseZGC`
- Proactive interval: `-XX:ZCollectionInterval=<sec>`
- Uncommit behavior: `-XX:+ZUncommit`, `-XX:ZUncommitDelay=<sec>`

### Shenandoah

- Enable: `-XX:+UseShenandoahGC`
- Mode: `-XX:ShenandoahGCMode=<mode>`
- Heuristics: `-XX:ShenandoahGCHeuristics=<heuristic>`

## Migration Gotchas

1. **Java 8 memory model change**
- PermGen is removed in Java 8 (JEP 122). Old PermGen flags become obsolete.

2. **Java 9 default collector change**
- Default moved from Parallel to G1 (JEP 248). Re-baseline latency/throughput after upgrade.

3. **Java 9+ logging change**
- GC logging moved to unified logging (`-Xlog`) under JEP 158/JEP 271.

4. **Java 14 CMS removal**
- `-XX:+UseConcMarkSweepGC` is not usable in Java 14+ (JEP 363).

5. **Java 24 ZGC mode change**
- Non-generational ZGC removed; `-XX:+UseZGC` uses generational mode (JEP 490).

6. **Do not bulk-carry old flags forward**
- Upgrade by starting from defaults + minimal SLO-driven flags; many old flags are deprecated/ignored/obsolete.

## Practical Upgrade Pattern

1. Record current GC flags and baseline p95/p99 latency + throughput.
2. Upgrade JDK with minimal flags (`-Xms/-Xmx`, optional pause target).
3. Capture GC logs under load.
4. Add one GC change at a time; re-measure.
5. Remove obsolete flags during each major JDK step.

## Sources

- Java 26 GA announcement: https://blogs.oracle.com/java/the-arrival-of-java-26
- Java 26 GC Tuning Guide index: https://docs.oracle.com/en/java/javase/26/gctuning/index.html
- Java 26 Ergonomics (defaults, goals): https://docs.oracle.com/en/java/javase/26/gctuning/ergonomics.html
- Java 26 `java` command options: https://docs.oracle.com/en/java/javase/26/docs/specs/man/java.html
- Java 8 Ergonomics (default throughput on server-class): https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/ergonomics.html
- Java 6 GC ergonomics: https://docs.oracle.com/javase/6/docs/technotes/guides/vm/gc-ergonomics.html
- Java 6 GC tuning guide: https://www.oracle.com/java/technologies/javase/gc-tuning-6.html
- JEP 248 (G1 default in Java 9): https://openjdk.org/jeps/248
- JEP 291 (CMS deprecated): https://openjdk.org/jeps/291
- JEP 363 (CMS removed in Java 14): https://openjdk.org/jeps/363
- JEP 122 (PermGen removal in Java 8): https://openjdk.org/jeps/122
- JEP 318 (Epsilon GC): https://openjdk.org/jeps/318
- JEP 333 (ZGC experimental in Java 11): https://openjdk.org/jeps/333
- JEP 377 (ZGC product in Java 15): https://openjdk.org/jeps/377
- JEP 439 (Generational ZGC in Java 21): https://openjdk.org/jeps/439
- JEP 490 (Remove non-generational ZGC in Java 24): https://openjdk.org/jeps/490
- JEP 189 (Shenandoah experimental in Java 12): https://openjdk.org/jeps/189
- JEP 379 (Shenandoah product in Java 15): https://openjdk.org/jeps/379
- JEP 158 (Unified JVM logging): https://openjdk.org/jeps/158
- JEP 271 (Unified GC logging): https://openjdk.org/jeps/271
