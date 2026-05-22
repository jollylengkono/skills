# Oracle SOA Suite Performance Tuning

## Overview

Use this skill to tune Oracle SOA Suite runtime performance for BPEL, Mediator, OSB, and infrastructure components across SOA 11g through 14.1.2. Load `references/soa-version-product-matrix.md` to confirm component availability before applying product-specific tuning.

## Tuning Workflow

1. Establish a baseline: measure instance throughput (instances/sec), response latency, JMS queue depths, and JVM heap/GC metrics.
2. Identify the bottleneck: BPEL dehydration overhead, Mediator routing latency, OSB pipeline processing, or JVM GC pressure.
3. Apply one tuning change at a time and re-measure before the next change.
4. Monitor the SOA infra database schema (SOAINFRA) for table growth and query latency.

## BPEL Tuning

### Dehydration Store

The dehydration store persists BPEL instance state to the SOAINFRA schema. Heavy dehydration is the most common SOA throughput bottleneck.

- Reduce dehydration frequency by lowering audit level:
  - Set audit level to `Production` (not `Development`) for production workloads
  - Set audit level to `Off` only when audit trail data is explicitly not required
- Tune the BPEL thread pool: `bpel.config.threadCount` in Enterprise Manager > SOA > SOA Infrastructure > Administration > System MBean Browser.
- Set `nonBlockingInvoke=true` for asynchronous invoke activities to avoid tying up a BPEL thread during remote calls.
- Keep the SOAINFRA schema tablespace on fast storage; monitor for growth and purge completed instances regularly.

### BPEL Instance Purging

- Configure purge jobs via Enterprise Manager > SOA > SOA Infrastructure > Administration > Purge Instance.
- Purge closed and faulted instances on a regular schedule to prevent SOAINFRA table bloat from degrading dehydration query performance.
- Run purge jobs during off-peak hours; large purge operations against the SOAINFRA schema can contend with active instance processing.

## Mediator Tuning

- Mediator routing is synchronous by default; use asynchronous routing for downstream calls with variable or long response times.
- Tune Mediator thread count: `mediator.config.threadCount` in the SOA MBean configuration.
- Reduce routing rule complexity for high-volume composites; each additional conditional routing rule adds evaluation overhead.
- Monitor routing service response time via Enterprise Manager > SOA > Mediator metrics.

## Oracle Service Bus (OSB) Tuning

- Tune OSB message flow thread pool via WebLogic Work Manager constraints on the OSB Managed Server; see `../weblogic/performance-tuning.md`.
- Use result caching for business services where upstream responses are stable over a short time window.
- Minimize transformation complexity in pipeline stages; split complex XSLT transforms into multiple simpler stages to improve readability and reduce single-stage overhead.
- Enable endpoint URI load balancing for business services with multiple available endpoint targets.
- Monitor OSB business service response times via OSB Console > Operations > Service Health.

## JMS Tuning

- Tune JMS server thread count on the SOA Managed Server to match the inbound message rate.
- Set message expiration on SOA-internal JMS queues to prevent unbounded queue growth from stuck or undelivered messages.
- Avoid the default JMS file store for high-throughput workloads; use JDBC persistence store or tune file store I/O path to fast local disk.

## JVM Tuning for SOA

- See `../java/performance-tuning.md` for JVM-level tuning guidance.
- SOA workloads with large XML payloads are memory-intensive: size heap conservatively with headroom for JAXB and DOM object churn.
- Use G1GC on JDK 9+ with a modest pause target for SOA workloads: `-XX:+UseG1GC -XX:MaxGCPauseMillis=200`.
- Tune Metaspace for deployments with many SOA composites and OSB configurations:
  `-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m`

## JDBC Tuning for SOA Infrastructure

- Set `InitialCapacity` equal to `MaxCapacity` on the SOAINFRA DataSource to pre-warm the pool.
- Set a short `ConnectionReserveTimeoutSeconds` on the SOAINFRA DataSource; BPEL dehydration failures caused by connection wait time are difficult to distinguish from application errors.
- Keep optimizer statistics current on the SOAINFRA schema: `DBMS_STATS.GATHER_SCHEMA_STATS('<soainfra_schema>')` — run periodically or after large purge operations.
- Enable statement caching on the SOAINFRA DataSource: `StatementCacheSize=20` minimum for complex SOA topologies.

## Oracle Version Notes (19c vs 26ai)

- For Oracle Database 19c backends: standard JDBC tuning and SOAINFRA schema management applies.
- For 26ai database targets: validate SOA infrastructure JDBC driver compatibility and SOAINFRA schema behavior before migrating the database.
- Keep database and SOA upgrade tracks separate; reconcile at integration testing.

## Sources

- `references/soa-version-product-matrix.md`
- `../java/performance-tuning.md`
- `../weblogic/performance-tuning.md`
- Oracle SOA Suite documentation: https://docs.oracle.com/en/middleware/soa-suite/
- SOA Suite 11g documentation: https://www.oracle.com/middleware/technologies/soa-suite-11gdocumentation.html
- Oracle Middleware certification: https://www.oracle.com/middleware/technologies/fusion-certification.html
