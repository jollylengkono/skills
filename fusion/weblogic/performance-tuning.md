# WebLogic Server Performance Tuning

## Overview

Use this skill to tune WebLogic Server for throughput, latency, and resource efficiency across versions 11g through 15c. Load `references/weblogic-version-matrix.md` to confirm version-specific settings before applying tuning changes.

## Tuning Workflow

1. Establish a baseline: capture thread pool utilization, JDBC wait counts, heap usage, and GC pause metrics.
2. Identify the bottleneck category: thread starvation, connection pool exhaustion, GC pressure, or I/O latency.
3. Apply one tuning change at a time and re-measure before the next change.
4. Document the rationale for each change and the observed delta.

## JVM Heap and GC Tuning

- Set `-Xms` equal to `-Xmx` for server workloads to avoid runtime heap resizing.
- Use G1GC as the default for WebLogic 12.2.1.4+ running on JDK 9+:
  `-XX:+UseG1GC -XX:MaxGCPauseMillis=200`
- Enable GC logging to guide tuning decisions:
  - JDK 8: `-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:<path>`
  - JDK 9+: `-Xlog:gc*:file=<path>:time,uptime`
- For WebLogic 11g on JDK 6/7: tune PermGen explicitly with `-XX:PermSize=256m -XX:MaxPermSize=512m`.
- Avoid over-allocating heap on nodes with multiple Managed Servers; leave OS headroom.
- See `../java/references/java-gc-reference.md` for the full collector selection matrix.

## Default Execute Queue and Work Managers

WebLogic uses a self-tuning thread pool managed by Work Managers.

- The default thread pool adapts based on throughput statistics.
- Use Work Managers to partition threads for request class differentiation:
  - `MaxThreadsConstraint` — cap threads for a given Work Manager
  - `MinThreadsConstraint` — floor to prevent starvation
  - `RequestClass` (Fair Share or Response Time) — control scheduling weight
- Avoid hardcoding a thread pool size with `-Dweblogic.threadpool.size`; prefer Work Manager constraints.
- Monitor via Admin Console > Environment > Servers > Monitoring > Threads.

## JDBC Connection Pool Tuning

- Set `InitialCapacity` and `MinCapacity` equal to `MaxCapacity` to pre-warm the pool on server start.
- Set `MaxCapacity` based on the database's maximum connection limit and application concurrency needs.
- Enable `TestConnectionsOnReserve` for production workloads where stale connections are a risk; use a lightweight `TestTableName` query.
- Set `ConnectionReserveTimeoutSeconds` to match the application SLA; avoid unlimited wait.
- Use Statement Cache to reduce parse overhead:
  - `StatementCacheSize` (default 10) — increase for workloads with high query variety
  - `StatementCacheType = LRU`

## JTA and Transaction Timeouts

- Default JTA timeout is 30 seconds; tune via Admin Console > Domain > Configuration > JTA.
- Align timeout with the longest expected transaction SLA; keep it as tight as the SLA allows.
- Long-running transactions that hold locks are a common cause of throughput degradation.

## HTTP Session and Cluster Tuning

- Set `InvalidationIntervalSecs` for in-memory sessions to reduce lock contention.
- For cluster deployments, prefer JDBC-based session persistence over in-memory replication for large clusters.
- Tune `TimeoutSecs` for idle session expiration to reclaim memory from abandoned sessions.

## Monitoring and Observability

- Enable WebLogic Diagnostics Framework (WLDF) watches and notifications for threshold-based alerting.
- Use WLST to periodically capture metrics for trend analysis.
- Use JFR for JVM-level profiling when GC log analysis alone is insufficient.

## Oracle Version Notes (19c vs 26ai)

When tuning WebLogic instances connected to Oracle Database:
- 19c baseline: standard JDBC thin driver configurations apply.
- 26ai targets: validate JDBC driver version compatibility before tuning DataSource settings.
- Keep database connection pool sizing and JTA timeout tuning distinct from WebLogic thread pool tuning.

## Sources

- `references/weblogic-version-matrix.md`
- `../java/references/java-gc-reference.md`
- WebLogic 15.1.1 documentation: https://docs.oracle.com/en/middleware/standalone/weblogic-server/15.1.1/wlsig/index.html
- WebLogic 14.1.2 documentation: https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/14.1.2/index.html
- WebLogic 12.2.1.4 documentation: https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/12.2.1.4/index.html
- WebLogic 14.1.1 documentation: https://docs.oracle.com/en/middleware/standalone/weblogic-server/14.1.1.0/index.html
- WebLogic 11g documentation: https://docs.oracle.com/middleware/11119/wls/index.html
