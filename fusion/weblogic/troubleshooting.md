# WebLogic Server Troubleshooting

## Overview

Use this skill to diagnose and resolve common WebLogic Server operational issues across versions 11g through 15c. Load `references/weblogic-version-matrix.md` when issue resolution depends on version-specific behavior.

## Fast Triage Workflow

1. Confirm WebLogic version, JDK, and platform (`weblogic.version` property, `java -version`).
2. Check server state in Admin Console or via WLST (`server.state`).
3. Locate the relevant log: `<domain>/servers/<name>/logs/<name>.log`.
4. Classify issue: startup, deployment, connectivity, memory, or thread.

## Startup Failures

Common causes:
- Port conflicts (default Admin port 7001; Managed Server ports vary)
- Missing or corrupt `boot.properties`
- Invalid JVM flags after JDK upgrade (check for removed or obsolete flags)
- Node Manager misconfiguration

Diagnostic steps:
- Check `<domain>/servers/AdminServer/logs/AdminServer.log`
- Run `startWebLogic.sh` from the command line to capture full output
- Confirm `JAVA_HOME` and `WL_HOME` environment variables
- Use `-Dweblogic.Stdout` and `-Dweblogic.Stderr` redirection to capture full server output

## Deployment Failures

Common causes:
- Missing shared libraries or modules required by the deployment
- Classloader conflicts between application and system classpath
- Deployment descriptor validation errors
- Insufficient permissions on deployment directories

Diagnostic steps:
- Check `<domain>/servers/<name>/logs/` for deployment task output
- Review Admin Console > Deployments > deployment state and task log
- Use `weblogic.Deployer` with `-verbose` flag for command-line deploys
- Inspect `weblogic-application.xml` and `weblogic.xml` descriptor settings

## JDBC Connection Pool Issues

Common symptoms:
- `Cannot get a connection, pool error Timeout waiting for connection`
- Connection leaks causing pool exhaustion
- Stale connections returning errors after database restart

Diagnostic steps:
- Admin Console > Services > Data Sources > Monitoring tab (active connections, wait count)
- Check `TestConnectionsOnReserve` and `TestTableName` settings
- Enable connection leak profiling in Development mode only; disable in Production
- Verify database listener and firewall allow connections from all Managed Server hosts

## Stuck Threads

WebLogic detects stuck threads when a thread exceeds the configured stuck-thread timeout (default: 600 seconds).

Diagnostic steps:
- Check `<domain>/servers/<name>/logs/` for `BEA-000337` (stuck thread detected)
- Capture a thread dump: `kill -3 <pid>` on Linux; `Ctrl+Break` on Windows; or `jcmd <pid> Thread.print`
- Identify blocked call stacks in the dump
- Admin Console > Environment > Servers > Monitoring > Threads

Remediation patterns:
- Increase the stuck thread timeout only if the operation is legitimately long-running
- Investigate the upstream dependency causing the block (database, JMS broker, remote EJB)
- Tune Work Manager constraints if thread contention is resource-specific

## OutOfMemoryError

Classify by the OOM message text:
- `Java heap space` → heap sizing issue
- `GC overhead limit exceeded` → GC tuning or heap growth needed
- `Metaspace` → class loader leak or insufficient Metaspace allocation
- `unable to create new native thread` → thread count limit or OS resource exhaustion

Diagnostic steps:
- Enable `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=<path>` before incidents recur
- Capture a heap dump on demand: `jcmd <pid> GC.heap_dump /path/heap.hprof`
- Use `jstat -gcutil <pid> 5000` to observe GC pressure over time

## Log File Locations

| Component | Default log path |
|---|---|
| Admin Server | `<domain>/servers/AdminServer/logs/AdminServer.log` |
| Managed Server | `<domain>/servers/<name>/logs/<name>.log` |
| Node Manager | `<NodeManager_home>/nodemanager.log` |
| HTTP access log | `<domain>/servers/<name>/logs/access.log` |
| Domain log | `<domain>/logs/<domain>.log` |

## Oracle Version Notes (19c vs 26ai)

This skill targets WebLogic operational troubleshooting. For database-layer issues:

- WebLogic 11g (10.3.6) runs on JDK 6/7 with PermGen; tune `-XX:MaxPermSize` instead of `-XX:MaxMetaspaceSize`.
- WebLogic 12.2.1.4+ supports JDK 8 and JDK 11; verify JDK compatibility before troubleshooting JVM flags.
- WebLogic 14.1.2 and 15c support JDK 17 and JDK 21; CMS GC is removed in JDK 14+, update GC flags accordingly.
- For estates connected to Oracle Database 19c or 26ai: treat JDBC connectivity issues as a separate track from WebLogic runtime troubleshooting.

## Sources

- `references/weblogic-version-matrix.md`
- WebLogic 15.1.1 documentation: https://docs.oracle.com/en/middleware/standalone/weblogic-server/15.1.1/wlsig/index.html
- WebLogic 14.1.2 documentation: https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/14.1.2/index.html
- WebLogic 12.2.1.4 documentation: https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/12.2.1.4/index.html
- WebLogic 14.1.1 documentation: https://docs.oracle.com/en/middleware/standalone/weblogic-server/14.1.1.0/index.html
- WebLogic 11g documentation: https://docs.oracle.com/middleware/11119/wls/index.html
