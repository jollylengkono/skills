# Oracle GoldenGate Performance Tuning

## Overview

Use this skill to tune Oracle GoldenGate Extract, Distribution, and Replicat for high-throughput and low-latency CDC pipelines. Verify tuning options against release-specific documentation using `references/goldengate-source-map.md`.

## Tuning Workflow

1. Establish a baseline: capture Extract lag, trail write rate, delivery rate, and Replicat apply rate.
2. Locate the bottleneck stage: capture, trail delivery (Distribution/Pump), or apply (Replicat).
3. Apply one tuning change at a time; re-measure lag and throughput before the next change.
4. Validate data correctness after any parallelism increase.

## Extract Tuning

### Integrated Extract (Oracle source, 11.2.0.3+)

Integrated Extract uses Oracle LogMiner infrastructure and is preferred over Classic Extract for Oracle 11.2.0.3+ environments.

- `CACHEMGR CACHESIZE <size>` — allocate cache for large transaction buffering; default is system-dependent
- `TRANSMEMORY DIRECTORY <path>` — specify a fast local disk path for transaction spill files; avoid NFS
- `FETCHOPTIONS NOUSESNAPSHOT` — reduce fetch overhead when supplemental logging covers all required columns

### Classic Extract

- Co-locate the trail write directory with fast local storage to reduce redo read latency.
- Set `SPLIT TRANLOG` to improve large transaction handling on busy redo log volumes.
- Avoid unnecessary `TRAILCHARSET` conversions; character set conversion adds CPU overhead.

## Trail File Tuning

- Keep trail files on fast local storage; avoid NFS for high-throughput pipelines.
- Tune trail segment size: `ADD EXTRACT <name>, EXTTRAIL <path>, MEGABYTES <n>` — default 500 MB; size based on volume and purge frequency.
- For WAN delivery with a dedicated Pump: enable compression with the `COMPRESS` parameter (GoldenGate 12c+) to reduce network transfer volume.

## Distribution Path and Pump Tuning

- Run a dedicated Pump Extract between source and target systems for isolation.
- For WAN environments: tune TCP buffer behavior with `TCPBUFSIZE` and `TCPFLUSHBYTES` parameters.
- Use parallel Pump processes when delivering to multiple independent Replicat processes.

## Replicat Tuning

### Integrated Replicat (Oracle target, 12.1.2+)

- Enable parallelism: `PARALLELISM <n>` — align apply thread count with target database CPU count; tune up incrementally.
- `BATCHSQL` is enabled by default; disable only when target constraint ordering issues require serial apply.
- `REPERROR DEFAULT, ABEND` is the safe default; use `REPERROR DEFAULT, DISCARD` only with active discard file monitoring.

### Parallel Replicat (19.1+)

- Parallel Replicat applies transactions concurrently using dependency metadata embedded in the trail.
- Tune `PARALLELISM` incrementally; monitor for apply server contention with `INFO REPLICAT <name>, DETAIL`.
- Ensure target tables have appropriate indexes; missing indexes on high-volume target tables degrade apply throughput significantly.

### Classic Replicat

- Use `BATCHSQL` to batch DML for insert-heavy workloads.
- Set `GROUPTRANSOPS <n>` to combine smaller source transactions into larger apply batches.
- `MAXTRANSOPS <n>` caps the number of operations per apply transaction; use to control commit frequency.

## Supplemental Logging Impact on Performance

- `ADD SUPPLEMENTAL LOG DATA` (minimal supplemental logging) adds the least redo overhead.
- ALL COLUMNS supplemental logging (`ADD SUPPLEMENTAL LOG DATA COLUMNS ALL TABLES`) increases redo volume significantly; use only when required by the Extract capture configuration.
- Identify tables without primary keys early; they require ALL COLUMNS logging by default and generate higher redo volume.

## Monitoring Throughput

| Command | Purpose |
|---|---|
| `STATS EXTRACT <name>` | Extract throughput (records/sec, bytes/sec) |
| `STATS REPLICAT <name>` | Replicat apply throughput |
| `LAG EXTRACT <name>` | Current Extract lag from source |
| `LAG REPLICAT <name>` | Current Replicat apply lag |

For GoldenGate Microservices (19c+): use the Performance Metrics Service in the Admin Server UI for real-time throughput dashboards.

## Oracle Version Notes (19c vs 26ai)

- For 19c source: Integrated Extract is standard; Classic Extract is supported but not preferred for new deployments.
- For 26ai targets: verify Parallel Replicat certification before enabling high-parallelism configurations; do not assume parameters from 19c apply deployments are valid unchanged.
- Do not carry forward 19c Replicat parameter files to 26ai apply targets without explicit compatibility review.

## Sources

- `references/goldengate-source-map.md`
- GoldenGate 23 core documentation: https://docs.oracle.com/en/middleware/goldengate/core/23/coredoc/toc.htm
- GoldenGate 21.3 core documentation: https://docs.oracle.com/en/middleware/goldengate/core/21.3/ggcab/overview-oracle-goldengate.html
- GoldenGate 19.1 documentation: https://docs.oracle.com/en/middleware/goldengate/core/19.1/
- GoldenGate 12.3.0.1 documentation: https://docs.oracle.com/en/middleware/goldengate/core/12.3.0.1/books.html
- GoldenGate certifications: https://www.oracle.com/integration/goldengate/certifications/
