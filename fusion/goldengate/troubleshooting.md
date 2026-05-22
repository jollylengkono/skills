# Oracle GoldenGate Troubleshooting

## Overview

Use this skill to diagnose and resolve Oracle GoldenGate Extract, Trail, Distribution, and Replicat issues across GoldenGate 12c through the current release. Load `references/goldengate-source-map.md` when issue resolution requires version-specific documentation.

## Fast Triage Workflow

1. Identify the affected component: Extract, Distribution Path, or Replicat.
2. Check process state: `INFO EXTRACT <name>` / `INFO REPLICAT <name>` in GGSCI or Admin Client.
3. Check the process report file and discard file for error detail.
4. Isolate whether the issue is data quality, resource, connectivity, or configuration.

## Extract Issues

### Extract ABENDs

Common causes:
- Redo log unavailability (archived log purged before Extract read it)
- Supplemental logging not enabled for source tables
- Unsupported DDL operations captured in the redo stream

Diagnostic steps:
- Check `ggserr.log` and the Extract report file: `VIEW REPORT <extract_name>` in GGSCI
- Verify supplemental logging: `SELECT LOG_MODE, SUPPLEMENTAL_LOG_DATA_MIN FROM V$DATABASE`
- Check archived log retention policy: `SHOW PARAMETER LOG_ARCHIVE_DEST`
- Confirm `ADD TRANDATA <schema>.<table>` coverage for all captured tables

### Extract Lag

Common causes:
- Heavy DML load exceeding Extract read throughput
- Redo log I/O bottleneck on the source system
- Long-running transactions holding the low-watermark SCN

Diagnostic steps:
- `LAG EXTRACT <name>` in GGSCI to confirm lag measurement and direction
- Check trail write performance on the source file system
- `SEND EXTRACT <name>, STATUS` for current checkpoint position
- Identify long-running transactions: `SELECT * FROM V$TRANSACTION WHERE START_TIME < SYSDATE - 1/24`

## Replicat Issues

### Replicat ABENDs

Common causes:
- Target table constraint violations (PK/UK conflicts, FK violations)
- DDL mismatch between source and target
- Mapping errors in the Replicat parameter file

Diagnostic steps:
- `VIEW REPORT <replicat_name>` in GGSCI for error detail
- Check the discard file for rejected records (configured with `DISCARDFILE <path>`)
- Review the Replicat parameter file `MAP ... TARGET ...` mappings for correctness
- Use `HANDLECOLLISIONS` only when explicitly intended; remove after the initial load phase

### Replicat Lag Spike

1. Check for apply-side database locks: `SELECT * FROM V$LOCK WHERE BLOCK = 1`
2. Confirm parallelism configuration for Integrated Replicat: `PARALLELISM` parameter value
3. Check for DDL errors blocking the apply queue
4. Review trail file delivery position vs. Replicat apply checkpoint for delivery bottlenecks

## Trail File Issues

- Trail disk exhaustion: verify available disk space on the trail directory; confirm `PURGEOLDEXTRACTS` is configured
- Corrupt trail: do not manually edit trail files; restore from a valid checkpoint or re-extract from source
- Trail accumulation without delivery: review Pump process lag and Replicat lag for delivery delays

## Network and Connectivity

- Classic GoldenGate Collector port: default `7809`; verify firewall rules allow inbound connections on target
- GoldenGate Microservices (19c+): verify Service Manager, Admin Server, and Distribution Server ports; consult deployment documentation for defaults
- TCP tuning for high-latency WAN links: use `TCPFLUSHBYTES` and `TCPBUFSIZE` parameters in the Distribution Path or Pump Extract parameter file

## GGSCI and Admin Client Diagnostic Commands

| Command | Purpose |
|---|---|
| `INFO ALL` | Show all process states |
| `VIEW REPORT <name>` | View current process report file |
| `SEND EXTRACT <name>, STATUS` | Extract checkpoint and current lag |
| `SEND REPLICAT <name>, STATUS` | Replicat checkpoint and current lag |
| `LAG EXTRACT <name>` | Extract lag from source SCN |
| `LAG REPLICAT <name>` | Replicat apply lag |
| `VIEW GGSEVT` | View the GoldenGate error event log |
| `STATS EXTRACT <name>` | Extract throughput statistics |
| `STATS REPLICAT <name>` | Replicat throughput statistics |

## Oracle Version Notes (19c vs 26ai)

- For 19c source databases: verify supplemental logging via `V$DATABASE` and `ALL_LOG_GROUPS`.
- For 26ai target databases: require explicit certification verification before applying GoldenGate configuration; do not assume 19c Replicat parameter files are valid unchanged on a 26ai target.
- For 19c-to-26ai migrations: monitor the Replicat discard file closely in the first 24 hours after cutover.

## Sources

- `references/goldengate-source-map.md`
- GoldenGate 23 core documentation: https://docs.oracle.com/en/middleware/goldengate/core/23/coredoc/toc.htm
- GoldenGate 21.3 core documentation: https://docs.oracle.com/en/middleware/goldengate/core/21.3/ggcab/overview-oracle-goldengate.html
- GoldenGate 19.1 documentation: https://docs.oracle.com/en/middleware/goldengate/core/19.1/
- GoldenGate 12.3.0.1 documentation: https://docs.oracle.com/en/middleware/goldengate/core/12.3.0.1/books.html
- GoldenGate certifications: https://www.oracle.com/integration/goldengate/certifications/
