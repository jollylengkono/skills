# Oracle Enterprise Manager 13c Performance Tuning

## Overview

This skill covers performance tuning for Oracle Enterprise Manager (OEM) 13c deployments. It targets administrators who need to resolve OEM slowness, high CPU and memory consumption, upload backlogs, or sluggish console response.

OEM 13c performance depends on five interacting components: the Oracle Management Service (OMS) JVM, the WebLogic Server domain hosting OMS, the Oracle Management Repository (OMR) database, the deployed Management Agents, and the network path between them. Tuning one component in isolation rarely resolves systemic problems — always work through the tuning workflow first.

The baseline used in this guide is OEM 13c Release 5 (13.5). Where behavior or recommended values differ by release, this is stated explicitly.

---

## Tuning Workflow

Before changing any parameter, identify which component is the bottleneck:

1. **Check OEM Self-Diagnostics first.** Navigate to Setup > Manage Cloud Control > Health Overview. Review OMS heap usage, loader thread backlog, repository session counts, and agent upload queue lengths.
2. **Check the OMR AWR report.** Run an AWR report on the OMR database (SYSMAN schema) and look for top wait events, high-parse SQL, and lock contention on MGMT$ tables.
3. **Review OMS logs.** The files `emoms.log` and `emoms.trc` in `$OMS_HOME/sysman/log/` contain actionable errors and warnings. Look for `OutOfMemoryError`, `GC overhead limit exceeded`, or `loader timeout` messages.
4. **Establish a sizing baseline.** Confirm OMS and OMR hardware meets Oracle's published sizing guidelines for your target count before changing tuning parameters.
5. **Change one parameter at a time** and measure impact before making additional changes.

---

## OMS JVM Heap Tuning

The OMS runs as a WebLogic managed server (`EMGC_OMS1`) inside the `GCDomain` WebLogic domain. Its JVM heap is configured via the `emctl` utility and stored in `$OMS_HOME/sysman/config/emoms.properties` and `$OMS_HOME/bin/emctl.ini`. Do not edit WebLogic start scripts directly — changes are overwritten during patching.

### Checking current heap settings

```bash
# View current OMS JVM heap settings (includes min, max, and actual usage)
$OMS_HOME/bin/emctl status oms -details
```

The output includes `JVM Heap Size (MB)` for both minimum (`Xms`) and maximum (`Xmx`) values. Also check the OMS Health Overview dashboard for real-time heap utilization percentage.

### Changing OMS JVM heap size

```bash
# Stop OMS before changing heap
$OMS_HOME/bin/emctl stop oms -all

# Set new heap values (example: 4 GB min, 8 GB max)
$OMS_HOME/bin/emctl set property -name EM_JAVA_HEAP_MIN -value 4096
$OMS_HOME/bin/emctl set property -name EM_JAVA_HEAP_MAX -value 8192

# Restart OMS to apply
$OMS_HOME/bin/emctl start oms
```

### Heap sizing guidelines by target count

Oracle's sizing guidance from the OEM Advanced Installation and Configuration Guide recommends the following heap ranges:

| Monitored targets | OMS heap min | OMS heap max |
|-------------------|-------------|-------------|
| Up to 500 | 2 GB | 4 GB |
| 500–1 000 | 4 GB | 8 GB |
| 1 000–2 000 | 6 GB | 12 GB |
| 2 000+ | 8 GB | 16 GB |

In multi-OMS deployments, each OMS node should be sized for its share of total target load plus 20% overhead.

### GC tuning

OEM 13c uses the G1GC garbage collector by default on JDK 8 and later. Do not change the GC algorithm without Oracle Support guidance. Key indicators of GC-related heap pressure:

- `GC overhead limit exceeded` in `emoms.log` — GC is consuming > 98% of elapsed time.
- Full GC events more frequent than once per hour — the JVM is struggling to reclaim memory.
- Heap after Full GC consistently above 80% of `Xmx` — the maximum heap is too small.

When GC issues appear, increase `EM_JAVA_HEAP_MAX` before adjusting GC-specific flags.

**JDK version note:** OEM 13.4 ships with JDK 8; OEM 13.5 ships with JDK 11 or JDK 17 depending on the release update. The `EM_JAVA_HEAP_MIN` and `EM_JAVA_HEAP_MAX` parameters apply to all supported JDK versions. The `XX:MaxPermSize` JVM flag (PermGen) has no effect on JDK 11 and later; Metaspace is self-sizing on those releases.

---

## OMR Database Tuning

The Oracle Management Repository (OMR) is the Oracle Database that stores all monitoring data, metric history, job definitions, and OEM configuration. Its performance directly affects console response time, loader throughput, and report generation speed.

### Key initialization parameters

The following parameters are documented for OMR sizing and performance in Oracle's OEM configuration guides and MOS Note 1080065.1. Apply changes to the OMR `spfile`:

| Parameter | Recommended starting value | Notes |
|-----------|---------------------------|-------|
| `sga_target` | 4–8 GB (scale with target count) | Use ASMM; do not use AMM (`memory_target`) |
| `pga_aggregate_target` | 2–4 GB | Supports large sort and hash operations in SYSMAN purge and aggregation jobs |
| `db_cache_size` | At least 2 GB | Minimum buffer cache for MGMT$ table blocks; Oracle can grow this under ASMM |
| `shared_pool_size` | 1–2 GB | Minimum shared pool for OMS cursor activity |
| `undo_retention` | 10800 (3 hours) minimum | Prevents `ORA-01555` during long report queries and slow purge operations |
| `open_cursors` | 600 | OEM repository procedures and OMS connection pool open many cursors per session |
| `cursor_sharing` | `EXACT` | **Do not set to `FORCE`.** OEM SQL uses bind variables; cursor sharing causes parse errors and performance regression |
| `parallel_max_servers` | Constrain to a defined value | Uncontrolled parallelism on the OMR degrades loader and query performance |
| `recyclebin` | `OFF` | Prevents SYSMAN purge-generated objects from accumulating in the recyclebin |

```sql
-- Apply recyclebin and cursor_sharing changes immediately (no restart required)
ALTER SYSTEM SET RECYCLEBIN = OFF SCOPE = BOTH;
ALTER SYSTEM SET CURSOR_SHARING = EXACT SCOPE = BOTH;
```

> **Important:** Do not use `MEMORY_TARGET` (Automatic Memory Management) for the OMR on Linux. Use `SGA_TARGET` + `PGA_AGGREGATE_TARGET` (ASMM) instead. AMM is incompatible with HugePages, and HugePages are strongly recommended for the OMR when SGA exceeds 4 GB.

### Index maintenance on SYSMAN tables

OEM's SYSMAN schema uses B-tree and bitmap indexes on interval-partitioned metric tables. These tables experience continuous high-churn inserts and deletes from loader jobs and purge operations, which leads to index bloat and eventually to unusable index partitions.

Check for unusable indexes:

```sql
-- Check for unusable non-partitioned SYSMAN indexes
SELECT owner, index_name, status
FROM   dba_indexes
WHERE  owner = 'SYSMAN'
  AND  status = 'UNUSABLE'
ORDER  BY index_name;

-- Check for unusable partitioned SYSMAN indexes
SELECT index_owner, index_name, partition_name, status
FROM   dba_ind_partitions
WHERE  index_owner = 'SYSMAN'
  AND  status = 'UNUSABLE'
ORDER  BY index_name, partition_name;
```

Rebuild unusable indexes during a maintenance window:

```sql
-- Rebuild a non-partitioned index online
ALTER INDEX SYSMAN.<index_name> REBUILD ONLINE;

-- Rebuild a specific partition of a partitioned index
ALTER INDEX SYSMAN.<index_name> REBUILD PARTITION <partition_name> ONLINE;
```

Oracle recommends rebuilding SYSMAN indexes after major OEM upgrades or repository backup and restore operations. Check `BLEVEL` for B-tree depth — values consistently above 4 on high-traffic indexes suggest a rebuild is warranted.

After index maintenance, refresh optimizer statistics:

```sql
-- As SYSMAN or DBA with ANALYZE ANY privilege
EXEC DBMS_STATS.GATHER_SCHEMA_STATS(
  ownname      => 'SYSMAN',
  cascade      => TRUE,
  no_invalidate => FALSE
);
```

Stale optimizer statistics on SYSMAN tables cause poor execution plans for purge jobs, aggregation queries, and OMS loader SQL.

### Purge policies and data retention

OEM accumulates metric data at a high rate. If purge jobs fall behind or are disabled, the OMR grows unbounded and query performance degrades progressively.

**Default retention tiers** (from OEM Administrator's Guide):

| Data tier | Default retention |
|-----------|-----------------|
| Raw metric data (per collection) | 7 days |
| 1-hour rollup data | 31 days |
| 1-day rollup data | 365 days |

For large deployments (2 000+ targets), reducing raw retention to 3–5 days significantly decreases OMR write volume and speeds purge completion.

**Adjust retention via the OEM console:** Setup > Manage Cloud Control > Repository > Metric Data Retention.

**Verify purge jobs are running:**

```sql
-- Check recent purge job execution history
SELECT job_name, status, actual_start_date, run_duration
FROM   dba_scheduler_job_run_details
WHERE  owner = 'SYSMAN'
  AND  job_name LIKE '%PURGE%'
ORDER  BY actual_start_date DESC
FETCH  FIRST 20 ROWS ONLY;

-- Check that the purge job is enabled and scheduled
SELECT job_name, enabled, state, last_start_date, next_run_date
FROM   dba_scheduler_jobs
WHERE  owner = 'SYSMAN'
  AND  job_name LIKE '%PURGE%';
```

Common causes of purge job failure or lag:
- Insufficient `UNDO_RETENTION` — long-running purge DML rolls back with `ORA-01555`; increase `UNDO_RETENTION` to at least 3 hours.
- Row lock contention — OMS loaders and purge jobs contend on SYSMAN tables; schedule purge jobs outside peak upload windows.
- Partition count growth — verify interval partitions are being dropped after purge by checking `dba_tab_partitions` for SYSMAN tables.

```sql
-- Check partition counts on high-volume SYSMAN tables
SELECT table_name, COUNT(*) AS partition_count
FROM   dba_tab_partitions
WHERE  table_owner = 'SYSMAN'
  AND  table_name LIKE 'MGMT$%'
GROUP  BY table_name
ORDER  BY partition_count DESC
FETCH  FIRST 20 ROWS ONLY;
```

---

## OEM Agent Performance

Each managed host runs an Oracle Management Agent that collects metrics and uploads them to OMS. Agent CPU and network overhead is controlled through collection intervals, metric exclusion lists, throttling properties, and heap sizing.

### Agent status and diagnostics

```bash
# Basic agent status
$AGENT_HOME/bin/emctl status agent

# Detailed status including upload queue size and last successful upload time
$AGENT_HOME/bin/emctl status agent -detail
```

A large upload queue (hundreds of files in `$AGENT_HOME/upload/`) means the agent is collecting faster than the OMS can process. Address OMS loader thread capacity first before tuning the agent.

### Collection intervals

Collection intervals are managed per metric and target type in the OEM console: navigate to Targets > [target] > Metrics > Metric and Collection Settings.

Oracle does not recommend intervals below 1 minute for any metric. For non-critical hosts, increase process-level metrics (CPU, memory, process count) to 5–15 minutes. For configuration metrics (which change rarely), 30–60 minute intervals are appropriate.

Via `emcli`:

```bash
# View metric collection properties for a target
$OMS_HOME/bin/emcli get_metric_collection_properties \
  -target_name="mydb" \
  -target_type="oracle_database"
```

### Exclusion lists

To reduce collection scope on targets that do not require full monitoring:

1. Navigate to the target in OEM console.
2. Go to Metrics > Metric and Collection Settings.
3. Set collection frequency to **Disabled** for metric categories not in use (for example, log file monitoring on hosts with no application logs, or tablespace metrics for tablespaces that are manually managed).

Reducing the number of active metric collections per target lowers agent CPU consumption, decreases upload file size, and reduces OMR write volume per collection cycle.

### Agent CPU and memory throttling

Starting with OEM 13c Release 3, the agent supports resource cap properties. Set these in `$AGENT_HOME/sysman/config/emd.properties`:

```
maxCPUPercentage=30
maxMemoryMB=512
```

After editing `emd.properties`, reload the agent to apply:

```bash
$AGENT_HOME/bin/emctl reload agent
```

> **Caution:** Setting `maxCPUPercentage` too low causes the agent to drop metric collections silently. Oracle recommends not setting it below 20% unless the host is severely resource-constrained.

### Agent heap size

The default agent JVM heap is 256 MB. For agents monitoring many targets (multiple databases, middleware servers, and hosts) on a single host, increase the heap:

```bash
# Stop the agent before changing heap
$AGENT_HOME/bin/emctl stop agent

# Set new heap values
$AGENT_HOME/bin/emctl setproperty agent \
  -name agentJavaDefines \
  -value "-Xms256m -Xmx512m"

$AGENT_HOME/bin/emctl start agent
```

### Upload interval and batch size

The agent accumulates metric XML files locally and uploads them in batches. Upload frequency is controlled by `emdUploadInterval` (default: 24 seconds) in `emd.properties`. For agents on high-latency WAN links or where OMS is under load, increase this value to reduce upload rate:

```bash
$AGENT_HOME/bin/emctl setproperty agent -name emdUploadInterval -value 60
$AGENT_HOME/bin/emctl reload agent
```

Ensure `$AGENT_HOME/upload/` has sufficient disk space when increasing upload intervals — more data accumulates locally before each upload cycle.

---

## WebLogic Managed Server Tuning for OMS

The OMS application runs as the `EMGC_OMS1` WebLogic managed server. WebLogic thread pool saturation and JDBC connection pool exhaustion cause console slowness and upload failures even when JVM heap and OMR are healthy.

### Thread pools

OMS uses WebLogic's self-tuning thread pool (default execute queue). Oracle recommends leaving the self-tuning pool at its default and resolving bottlenecks in the OMR or OMS heap first. However, monitoring the thread pool state is useful for diagnosing stuck-thread conditions.

From the WebLogic Administration Console at `https://<oms-host>:7102/console`: Environment > Servers > EMGC_OMS1 > Monitoring > Threads.

Via WLST:

```bash
$MIDDLEWARE_HOME/oracle_common/common/bin/wlst.sh
```

```python
connect('weblogic', '<password>', 't3://<oms-host>:7101')
serverRuntime()
cd('ThreadPoolRuntime/ThreadPoolRuntime')
ls()
# Key attributes to check:
# ExecuteThreadTotalCount  — total thread count
# HoggingThreadCount       — threads stuck longer than MaxStuckThreadTime
# PendingUserRequestCount  — queued requests waiting for a thread (should be 0)
# StandbyThreadCount       — idle threads available
```

`PendingUserRequestCount > 0` sustained over several minutes indicates thread pool saturation. Before increasing thread count, check whether the OMR is the underlying bottleneck causing threads to block on long-running SQL.

### JDBC connection pool (OMR connections)

OMS connects to the OMR through a JDBC data source named `emgc_omsjdk`. Default pool settings are documented in the OEM Advanced Installation and Configuration Guide. For deployments with more than 1 000 targets, the default pool size may require adjustment.

Via WLST:

```python
connect('weblogic', '<password>', 't3://localhost:7101')
edit()
startEdit()
cd('/JDBCSystemResources/emgc_omsjdk/JDBCResource/emgc_omsjdk/JDBCConnectionPoolParams/emgc_omsjdk')
# Check current MaxCapacity
get('MaxCapacity')
# Increase for large deployments
set('MaxCapacity', 100)
set('MinCapacity', 25)
save()
activate()
disconnect()
```

After activation, restart the OMS managed server:

```bash
$OMS_HOME/bin/emctl stop oms
$OMS_HOME/bin/emctl start oms
```

> **Critical constraint:** Do not increase `MaxCapacity` beyond what the OMR `PROCESSES` parameter can accommodate. Each active pool connection consumes one database session on the OMR. The OMR `PROCESSES` value must be at least:
> `MaxCapacity × number_of_OMS_nodes + additional_loader_connections + 50 (headroom)`

### Connection pool for loader threads

OMS maintains a separate connection pool for metric loader threads. The property `oracle.sysman.core.conn.maxConnForLoaders` in `$OMS_HOME/sysman/config/emoms.properties` controls the maximum connections available to loaders. It must be proportional to `em.loader.threadPoolSize`:

```bash
# View current value
$OMS_HOME/bin/emctl get property \
  -name oracle.sysman.core.conn.maxConnForLoaders

# Set loader connection pool size to match loader thread count
$OMS_HOME/bin/emctl set property \
  -name oracle.sysman.core.conn.maxConnForLoaders \
  -value 30

# Restart OMS to apply
$OMS_HOME/bin/emctl stop oms -all
$OMS_HOME/bin/emctl start oms
```

---

## Upload Performance and Loader Thread Tuning

OMS loader threads receive metric files uploaded by agents and write them to the OMR. Loader backlog is the most common cause of delayed metric visibility and missed alerting windows.

### Diagnosing loader backlog

```bash
# On OMS host: count pending upload files in the OMS receive staging area
ls $OMS_HOME/upload/ | wc -l
```

From the console: Setup > Manage Cloud Control > Health Overview > Loader Statistics.

Via `emctl`:

```bash
$OMS_HOME/bin/emctl status oms -loader
```

A healthy deployment processes uploads within seconds of receipt. A queue growing above 200 files that is not draining indicates loader thread or OMR write contention.

### Adjusting loader thread count

```bash
# View current loader thread pool size
$OMS_HOME/bin/emctl get property -name em.loader.threadPoolSize

# Increase loader threads (default is typically 20; Oracle recommends a maximum of 40 per OMS node)
$OMS_HOME/bin/emctl set property -name em.loader.threadPoolSize -value 30

# Restart OMS to apply
$OMS_HOME/bin/emctl stop oms -all
$OMS_HOME/bin/emctl start oms
```

Oracle recommends not exceeding 40 loader threads per OMS instance. Beyond this, OMR write contention becomes the limiting factor rather than thread count. For deployments requiring higher aggregate throughput, add a second OMS instance with a load balancer instead of continuing to increase threads on a single node.

When increasing `em.loader.threadPoolSize`, also increase `oracle.sysman.core.conn.maxConnForLoaders` by the same amount — each active loader thread requires one OMR connection.

### Upload file size management

Large metric payloads from agents monitoring targets with many metrics (for example, an agent monitoring dozens of databases on the same host) can stall individual loader threads on single large files. Reduce maximum upload file size on high-volume agents:

```bash
# Reduce max file size per upload to 1 MB (default is 2 MB)
$AGENT_HOME/bin/emctl setproperty agent -name MaxFileSize -value 1048576
$AGENT_HOME/bin/emctl reload agent
```

Also reduce `MaxFilesPerUpload` if the OMS is receiving too many simultaneous large batches:

```bash
$AGENT_HOME/bin/emctl setproperty agent -name MaxFilesPerUpload -value 2
$AGENT_HOME/bin/emctl reload agent
```

---

## Sizing Guidelines

The following guidelines are derived from the Oracle Enterprise Manager Cloud Control Advanced Installation and Configuration Guide (13c Release 5) and Oracle's published sizing documentation (MOS Note 2073344.1).

### OMS host sizing

| Monitored targets | OMS CPU cores | OMS host RAM | OMS heap (Xmx) | OMS nodes |
|-------------------|--------------|-------------|----------------|-----------|
| Up to 500 | 4 cores | 8 GB | 4 GB | 1 |
| 500–1 000 | 8 cores | 16 GB | 8 GB | 1 |
| 1 000–2 000 | 16 cores | 32 GB | 12 GB | 1–2 |
| 2 000–5 000 | 2× 16 cores | 32 GB each | 12–16 GB each | 2 |
| 5 000+ | Consult Oracle Sizing Guide (MOS 2073344.1) | | | 3+ |

Storage for `$OMS_HOME`: minimum 50 GB. Storage for the upload staging directory (`$OMS_HOME/upload/`): minimum 10 GB, more for large agent counts and high collection frequency.

Multi-OMS deployments (two or more OMS nodes) require a load balancer in front of OMS nodes. All OMS nodes must share the same OMR and the same software home location structure. Refer to the OEM HA configuration guide for load balancer setup.

### OMR database sizing

| Monitored targets | OMR CPU cores | OMR RAM (SGA + PGA) | OMR storage |
|-------------------|--------------|--------------------|-----------| 
| Up to 500 | 4 cores | 8 GB | 200 GB |
| 500–1 000 | 8 cores | 16 GB | 500 GB |
| 1 000–2 000 | 16 cores | 32 GB | 1 TB |
| 2 000–5 000 | 24+ cores | 64 GB | 2–4 TB |
| 5 000+ | Consult Oracle Sizing Guide | | 4+ TB |

Storage estimates assume default retention policies (7 days raw, 31 days hourly, 365 days daily) and average-density metric collection. Reduce storage by approximately 40% if raw retention is reduced to 3 days. Use fast storage — the OMR sustains continuous high-throughput random and sequential I/O from loaders and purge jobs. SSD-backed storage is strongly recommended for `MGMT_TABLESPACE` datafiles.

**OMR character set requirement:** The OMR database must be created with character set `AL32UTF8`. Verify before deploying OEM:

```sql
SELECT value FROM nls_database_parameters WHERE parameter = 'NLS_CHARACTERSET';
-- Must return: AL32UTF8
```

**OMR database version:** OEM 13.5 requires Oracle Database 12.2 or later for the OMR. Oracle 19c (19.20 or later) is the recommended OMR database version and the most tested configuration for OEM 13.5.

---

## Monitoring OEM's Own Performance

OEM can monitor its own components. Enabling self-monitoring provides early warning before performance problems become visible to users.

### Health Overview dashboard

Navigate to Setup > Manage Cloud Control > Health Overview. This dashboard shows:

- OMS heap and CPU usage
- Loader thread backlog and throughput
- Agent upload success and failure rates
- OMR session counts and connection pool utilization
- Notification subsystem and job subsystem status

Review this dashboard at least once daily in large deployments (1 000+ targets).

### Key self-monitoring metrics and thresholds

| Metric | Warning threshold | Critical threshold | Meaning |
|--------|------------------|-------------------|---------|
| OMS Heap Usage (%) | 75% | 90% | Risk of `OutOfMemoryError`; increase `EM_JAVA_HEAP_MAX` |
| Upload Queue Length (files) | 100 | 500 | Loader thread or OMR write bottleneck |
| OMR Active Sessions | 70% of `PROCESSES` | 85% of `PROCESSES` | OMR session exhaustion risk; increase `PROCESSES` or reduce pool sizes |
| Agent Upload Failure Rate (%) | 2% | 5% | Network or OMS availability issue |
| Console Page Response Time (ms) | 5 000 ms | 15 000 ms | OMS thread saturation or OMR latency |

### emctl diagnostic commands

```bash
# Full OMS status and component health
$OMS_HOME/bin/emctl status oms -details

# OMS heap and JVM metrics
$OMS_HOME/bin/emctl status oms -jvm

# Loader queue depth and throughput
$OMS_HOME/bin/emctl status oms -loader

# Repository database connectivity and response time
$OMS_HOME/bin/emctl status oms -bip
```

### OMR diagnostic queries

```sql
-- Active OMS sessions connecting to the OMR
SELECT COUNT(*) AS oms_sessions
FROM   v$session
WHERE  module LIKE '%OMS%';

-- Top wait events on the OMR (last cumulative)
SELECT event,
       total_waits,
       ROUND(time_waited_micro / 1e6, 2) AS time_waited_sec,
       ROUND(average_wait_micro / 1e3, 2) AS avg_wait_ms
FROM   v$system_event
WHERE  wait_class != 'Idle'
ORDER  BY time_waited_micro DESC
FETCH  FIRST 15 ROWS ONLY;

-- Recent SYSMAN job failures
SELECT job_name, status, actual_start_date, run_duration
FROM   dba_scheduler_job_run_details
WHERE  owner = 'SYSMAN'
  AND  status != 'SUCCEEDED'
ORDER  BY actual_start_date DESC
FETCH  FIRST 50 ROWS ONLY;

-- MGMT_TABLESPACE usage
SELECT tablespace_name,
       ROUND(used_space * 8192 / 1024 / 1024 / 1024, 2) AS used_gb,
       ROUND(tablespace_size * 8192 / 1024 / 1024 / 1024, 2) AS total_gb,
       ROUND(used_percent, 1) AS used_pct
FROM   dba_tablespace_usage_metrics
WHERE  tablespace_name IN ('MGMT_TABLESPACE', 'MGMT_AD4J_TS', 'MGMT_ECM_DEPOT_TS')
ORDER  BY used_percent DESC;
```

### Key OEM log files

| Log file | What to look for |
|----------|-----------------|
| `$OMS_HOME/sysman/log/emoms.log` | Loader errors, `OutOfMemoryError`, OMR connectivity failures |
| `$OMS_HOME/sysman/log/emoms.trc` | Detailed OMS trace including SQL errors |
| `$OMS_HOME/sysman/log/emoms_pbs.log` | Purge and batch processing errors, job subsystem |
| `$MIDDLEWARE_HOME/user_projects/domains/GCDomain/servers/EMGC_OMS1/logs/EMGC_OMS1.out` | WebLogic managed server JVM stdout; GC events |
| `$AGENT_HOME/sysman/log/emagent.log` | Agent upload failures, collection errors |
| `$AGENT_HOME/sysman/log/emagent_perl.log` | Agent metric collection script errors |

---

## Oracle Version Notes

**OEM 13c Release 3 (13.3):** Introduced `maxCPUPercentage` and `maxMemoryMB` agent throttling properties. These properties do not exist in OEM 13.1 or 13.2 agents.

**OEM 13c Release 4 (13.4):** Improved self-monitoring dashboards introduced in Health Overview. Ships with JDK 8 for the OMS JVM. The `XX:MaxPermSize` JVM flag applies only to JDK 8; it has no effect on JDK 11+.

**OEM 13c Release 5 (13.5):** Current recommended release. Ships with JDK 11 (some release updates include JDK 17 support). The G1GC configuration differs between JDK 8 and JDK 11 defaults; review `emoms.log` and the WebLogic managed server stdout after an OMS upgrade for GC-related changes in behavior.

**OMR database version support:** OEM 13.5 supports Oracle Database 12.2 or later as the OMR. Oracle 19c (19.20+) is the most tested OMR version. Oracle 11g R2 is not a supported OMR database for any OEM 13c release.

**Optimizer adaptive features on OMR:** On Oracle Database 12.2 and 19c, the parameter `optimizer_adaptive_features` was split into `_optimizer_adaptive_plans` and `_optimizer_adaptive_statistics`. Oracle Support MOS Note 1080065.1 provides specific guidance on which adaptive features to disable for OMR workloads on each database version; follow that note rather than applying blanket disabling of adaptive features.

---

## Sources

- Oracle Enterprise Manager Cloud Control Advanced Installation and Configuration Guide 13c Release 5:
  https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emadv/

- Oracle Enterprise Manager Cloud Control Administrator's Guide 13c Release 5:
  https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emadm/

- Oracle Enterprise Manager Cloud Control Basic Installation Guide 13c Release 5 (hardware sizing reference):
  https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/embsc/

- Oracle Enterprise Manager documentation hub:
  https://docs.oracle.com/en/enterprise-manager/

- Oracle Support Note 1080065.1 — Enterprise Manager 13c Repository (SYSMAN) Database Initialization Parameters *(requires Oracle Support login)*:
  https://support.oracle.com/epmos/faces/DocumentDisplay?id=1080065.1

- Oracle Support Note 2073344.1 — Enterprise Manager 13c Cloud Control Sizing Guide *(requires Oracle Support login)*:
  https://support.oracle.com/epmos/faces/DocumentDisplay?id=2073344.1

- Oracle WebLogic Server Performance and Tuning Guide (12.2.1):
  https://docs.oracle.com/middleware/12213/wls/PERFM/

- Oracle Database 19c Database Administrator's Guide — Managing Memory:
  https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-memory.html

- Oracle Database 19c DBMS_STATS Reference:
  https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_STATS.html

- Oracle Database 19c Reference — DBA_SCHEDULER_JOBS:
  https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_SCHEDULER_JOBS.html
