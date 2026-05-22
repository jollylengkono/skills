# Oracle Enterprise Manager 13c Troubleshooting

## Overview

Use this skill to diagnose and resolve operational issues in Oracle Enterprise Manager (OEM) 13c deployments. Coverage includes OMS startup failures, agent connectivity problems, repository database issues, job and metric collection failures, and console access errors. Commands and procedures apply to OEM 13c (releases 13.4 and 13.5) unless a version-specific note states otherwise.

OEM 13c consists of three primary components:

- **OMS (Oracle Management Service)** — the middle-tier Java EE application running inside a WebLogic domain (`GCDomain`). Hosts the OEM console and dispatches jobs to agents.
- **OMR (Oracle Management Repository)** — an Oracle Database instance with the SYSMAN schema storing all management data, metric history, job definitions, and configuration.
- **Management Agent** — a lightweight process deployed on each monitored host that collects metrics and uploads them to OMS.

All three components must be healthy for OEM to function normally. Triage always starts with status checks across all three.

---

## Fast Triage Workflow

Run these checks in order before diving into component-specific diagnosis.

### 1. Check OMS status

```bash
# Run as the OMS software owner (e.g., oracle)
$OMS_HOME/bin/emctl status oms
```

Expected output includes `Oracle Management Server is Up`. If the OMS is down, proceed to [OMS Startup Failures](#oms-startup-failures) before anything else.

### 2. Check OMS detailed status (OMR connectivity + WebLogic)

```bash
$OMS_HOME/bin/emctl status oms -details
```

The output includes the repository database connect string, the last successful OMR ping time, and the WebLogic server states (`EMGC_ADMINSERVER`, `EMGC_OMS1`). A failing OMR connection here means the console will load partially or not at all — proceed to [OMR Connectivity and Schema Issues](#omr-connectivity-and-schema-issues).

### 3. Check Agent status on a monitored host

```bash
# Run as the agent software owner on the target host
$AGENT_HOME/bin/emctl status agent
```

Look for `Agent is Running and Ready` and verify that `Last successful upload` shows a recent timestamp. Stale uploads or non-running states lead to [Agent Issues](#oem-agent-issues).

### 4. Confirm the console URL is reachable

```
https://<oms-host>:7803/em
```

If the URL is unreachable and OMS reports as running, the problem is almost always in the WebLogic layer or SSL — not the agent or OMR. Proceed to [Console Access Issues](#oem-console-access-issues-weblogic-and-ssl).

### 5. Review OMS logs for recent errors

```
$OMS_HOME/sysman/log/emoms.log
$OMS_HOME/sysman/log/emoms.trc
```

Search for `ERROR` or `FATAL` lines at the timestamp of the reported problem. These logs are the single most useful starting point for any OMS-side failure.

---

## OMS Startup Failures

### Common causes

| Symptom | Likely cause |
|---|---|
| `emctl start oms` hangs or times out | WebLogic AdminServer not started; port conflict |
| `Cannot connect to repository` at startup | OMR database down, TNS resolution failure, or SYSMAN account locked |
| OMS starts but console shows errors | Stale OPSS wallet, corrupted WebLogic domain, or NLS mismatch in OMR |
| `SEVERE` errors in OMS logs | Java heap exhaustion, disk full on OMS host, or corrupted deployment |
| OMS restarts repeatedly | JVM crash (check OMS JVM crash logs in `$OMS_HOME/sysman/log/`) |

### Startup procedure

OEM 13c requires the WebLogic AdminServer to be running before the OMS managed server can start. The `emctl start oms` wrapper handles this, but if it fails midway, start components manually:

```bash
# Step 1 — Start WebLogic AdminServer
$MIDDLEWARE_HOME/user_projects/domains/GCDomain/bin/startWebLogic.sh &

# Step 2 — Wait until AdminServer is RUNNING, then start OMS
$OMS_HOME/bin/emctl start oms
```

### Port conflict

OEM 13c uses several ports. If any are occupied, the corresponding component fails to start.

```bash
netstat -tlnp | grep -E '7803|7301|4889|4903'
```

Default port assignments are recorded in `$OMS_HOME/install/portlist.ini`. Identify which process is using a conflicting port with `lsof -i :<port>` and resolve before retrying startup.

### SYSMAN account locked or expired

Connect as SYSDBA to the OMR database:

```sql
SELECT account_status FROM dba_users WHERE username = 'SYSMAN';
ALTER USER sysman ACCOUNT UNLOCK IDENTIFIED BY "<password>";
```

After unlocking, restart the OMS. If the password was changed outside OEM, also reconfigure the OMS repository connection (see below).

### Reconfiguring the repository connection

If the OMR connect string or SYSMAN password changes (e.g., after a RAC node change, service rename, or out-of-band password reset):

```bash
$OMS_HOME/bin/emctl config oms -store_repos_details \
  -repos_conndesc "(DESCRIPTION=...)" \
  -repos_user sysman
```

You are prompted for the SYSMAN password. After reconfiguring, restart the OMS.

### OPSS wallet corruption

If `emoms.log` contains `OPSS wallet` errors during startup:

1. Stop OMS completely: `$OMS_HOME/bin/emctl stop oms -all`
2. Restore the OPSS wallet from backup. The wallet is stored under:
   `$MIDDLEWARE_HOME/user_projects/domains/GCDomain/config/fmwconfig/`
3. If no backup exists, open an Oracle Support Request — wallet re-creation is destructive and requires guidance from Oracle Support.

---

## OEM Agent Issues

### Agent unreachable

From the OEM console, an agent shows **Unreachable** when the OMS cannot contact the agent's upload URL (default HTTPS port 3872).

Common causes:

1. Agent process is down — verify with `$AGENT_HOME/bin/emctl status agent` on the target host.
2. Firewall or network change blocking port 3872 between OMS and the agent host.
3. Agent SSL certificate mismatch after OMS certificate rotation.

Restart a stopped agent:

```bash
$AGENT_HOME/bin/emctl start agent
```

Test reachability from the OMS host:

```bash
curl -k https://<agent-host>:3872/emd/main/
```

A response containing `Oracle Management Agent` confirms the agent is accessible.

### Agent blocked

An agent shows **Blocked** when it has been explicitly blocked, either manually by an administrator or by OMS policy. Unblock from the console: **Setup > Manage Cloud Control > Agents**, select the agent, then **Agent > Unblock**. Or use emcli:

```bash
emcli unblock_agent -agent="<agent-host>:<port>"
```

### Upload failures

Symptoms: `Last successful upload` is stale; `Upload retry count` is nonzero.

Check the agent upload log:

```
$AGENT_HOME/agent_inst/sysman/log/emagent_upload.log
```

Common causes and fixes:

- **OMS certificate changed** — resecure the agent:
  ```bash
  $AGENT_HOME/bin/emctl secure agent
  ```
- **Upload directory full** — check disk space under `$AGENT_HOME/agent_inst/sysman/emd/upload/`. Remove stale XML files manually only as a last resort; prefer resolving the upload failure so they upload normally.
- **OMS unreachable from agent** — verify the OMS upload URL configured for the agent:
  ```bash
  $AGENT_HOME/bin/emctl status agent | grep -i "OMS URL"
  ```
  Then test connectivity: `curl -k <OMS-upload-URL>`.
- **Agent upload queue too large** — if the backlog is unrecoverable and data loss is acceptable:
  ```bash
  $AGENT_HOME/bin/emctl clearstate agent
  ```
  This discards all pending unuploaded data. Use with caution.

### Re-securing an agent after OMS certificate rotation

```bash
# On the agent host
$AGENT_HOME/bin/emctl secure agent -reg_passwd <reg_password>
```

The registration password is set during OEM installation. Retrieve the current value from the OMS:

```bash
$OMS_HOME/bin/emctl get_reg_pwd oms
```

---

## OMR Connectivity and Schema Issues

### Check OMR reachability from OMS

```bash
$OMS_HOME/bin/emctl status oms -details | grep -i repository
```

Verify the OMR database is open and the listener is running on the OMR host:

```bash
lsnrctl status
```

```sql
-- Connect as SYSDBA to OMR
SELECT status FROM v$instance;
-- Expected: OPEN
```

### NLS character set requirement

OEM 13c requires the OMR database to use `AL32UTF8`. A mismatch causes OMS startup to fail with `NLS_CHARACTERSET mismatch` errors in `emoms.log`.

```sql
SELECT value FROM nls_database_parameters WHERE parameter = 'NLS_CHARACTERSET';
-- Must return: AL32UTF8
```

### SYSMAN schema issues

If the SYSMAN schema is corrupted or version-mismatched with the OMS binary, use the Repository Configuration Assistant (RepCA) included with the OMS installer, or apply the schema patch steps from the OEM 13c patching documentation.

Check for invalid objects in the SYSMAN schema:

```sql
SELECT object_name, object_type, status
FROM   dba_objects
WHERE  owner = 'SYSMAN'
  AND  status != 'VALID'
ORDER BY object_type, object_name;
```

Key SYSMAN-owned tables implicated in performance or integrity issues:

- `SYSMAN.MGMT_METRICS` — raw metric data; subject to fragmentation over time
- `SYSMAN.MGMT_JOB_EXEC_SUMMARY` — job execution history
- `SYSMAN.MGMT_TARGETS` — target registration records

Check OEM component version in the repository:

```sql
SELECT component_name, version, status
FROM   sysman.mgmt_versions;
```

The component version here must match the installed OMS binary version. A mismatch after a failed upgrade is a common cause of console errors.

---

## Repository Database Performance Problems Affecting the OEM Console

A slow OMR database manifests directly as a slow or unresponsive OEM console. Symptoms: page load timeouts, metrics that are hours out of date, job queues building up.

### Quick diagnosis

```sql
-- Active sessions waiting in OMR
SELECT sid, event, wait_class, seconds_in_wait, state
FROM   v$session
WHERE  username IN ('SYSMAN', 'DBSNMP')
  AND  status = 'ACTIVE'
ORDER BY seconds_in_wait DESC;
```

```sql
-- Top SQL by elapsed time from SYSMAN
SELECT sql_id,
       ROUND(elapsed_time / 1000000, 1) elapsed_s,
       executions,
       sql_text
FROM   v$sql
WHERE  parsing_schema_name = 'SYSMAN'
  AND  executions > 0
ORDER BY elapsed_time DESC
FETCH FIRST 10 ROWS ONLY;
```

```sql
-- Check for lock contention on SYSMAN tables
SELECT l.sid, l.type, l.lmode, l.request, o.object_name
FROM   v$lock l
JOIN   dba_objects o ON l.id1 = o.object_id
WHERE  o.owner = 'SYSMAN'
  AND  l.request > 0;
```

### Common causes and fixes

| Problem | Resolution |
|---|---|
| Stale or missing SYSMAN statistics | `DBMS_STATS.GATHER_SCHEMA_STATS('SYSMAN', options => 'GATHER STALE')` |
| SYSMAN tablespace full | Extend `MGMT_TABLESPACE` or enable autoextend |
| Purge job not running | Verify `MGMT_PURGE` scheduler jobs are enabled (see below) |
| High `MGMT_METRICS` fragmentation | Gather stats on the table; consider shrink space after purge |

### Tablespace check

```sql
SELECT tablespace_name,
       ROUND(used_space * 8192 / 1073741824, 2) AS used_gb,
       ROUND(tablespace_size * 8192 / 1073741824, 2) AS total_gb
FROM   dba_tablespace_usage_metrics
WHERE  tablespace_name LIKE 'MGMT%';
```

### Purge job status

OEM's built-in purge jobs trim old metric and event data. If they are disabled, SYSMAN tablespaces grow unbounded.

```sql
SELECT job_name, enabled, state, last_run_duration
FROM   dba_scheduler_jobs
WHERE  owner = 'SYSMAN'
  AND  job_name LIKE '%PURGE%';
```

Re-enable a disabled purge job:

```sql
BEGIN
  dbms_scheduler.enable('SYSMAN.MGMT_PURGE');
END;
/
```

---

## Job and Metric Collection Failures

### Jobs stuck in Running or Pending state

From the console: **Enterprise > Job > Activity**, filter by status.

For bulk investigation via SQL:

```sql
SELECT job_name, status, start_time, last_updated_time
FROM   sysman.mgmt_job_exec_summary
WHERE  status IN (1, 2)        -- 1 = Running, 2 = Pending
  AND  start_time < SYSDATE - 1
ORDER BY start_time;
```

A job stuck in `Running` for more than a day usually indicates a dead agent or an OMS disruption during execution. Use the console to suspend and reschedule.

If many jobs are stalled simultaneously, the OMS job dispatcher may be blocked. Restart the OMS:

```bash
$OMS_HOME/bin/emctl stop oms
$OMS_HOME/bin/emctl start oms
```

### Metric collection errors for a target

If a target shows **Metric Collection Error**, navigate to the target's monitoring configuration in the console:

**Target home > Monitoring > Metric Collection Errors**

Common causes:

- Agent lost connectivity to the monitored target (database listener down, host unreachable).
- Monitoring credentials missing or expired. Update via **Setup > Security > Monitoring Credentials**.
- DBSNMP account locked on the monitored Oracle Database:
  ```sql
  -- Connect as SYSDBA to the monitored database (not the OMR)
  ALTER USER dbsnmp ACCOUNT UNLOCK IDENTIFIED BY "<password>";
  ```
  Then update the credential in OEM.
- Plug-in version mismatch between OMS and agent. Check deployed plug-ins on the agent:
  ```bash
  $AGENT_HOME/bin/emctl listplugins agent -type all
  ```

### Triggering a manual collection

To force collection of a specific metric on an agent:

```bash
$AGENT_HOME/bin/emctl control agent runCollection \
  <target_name>:<target_type> <metric_name>
```

### Blackout interference

A metric collection gap may coincide with an active blackout. Check:

```bash
emcli get_blackouts -target_type=oracle_database -target_name=<db_name>
```

---

## OEM Console Access Issues (WebLogic and SSL)

### WebLogic domain problems

The OEM console runs inside the `GCDomain` WebLogic domain. The domain contains the AdminServer (`EMGC_ADMINSERVER`) and at least one OMS managed server (`EMGC_OMS1`). Both must be running for the console to be accessible.

Check server states:

```bash
$OMS_HOME/bin/emctl status oms -details
```

Look for `EMGC_ADMINSERVER` and `EMGC_OMS1` in `RUNNING` state.

Access the WebLogic Administration Console directly to inspect or restart individual servers:

```
https://<oms-host>:7101/console
```

Restart the full OMS stack cleanly:

```bash
$OMS_HOME/bin/emctl stop oms -all
$OMS_HOME/bin/emctl start oms
```

The `-all` flag stops the WebLogic AdminServer together with the OMS managed servers. Omitting it leaves the AdminServer running, which is sometimes intentional but can cause partial state issues on re-start.

If WebLogic servers are stuck in `STARTING`, check for port conflicts on 7001, 7099, 7101, and 7788 (default WebLogic ports for OEM GCDomain).

### SSL and certificate errors

OEM 13c uses HTTPS for all communication: browser to OMS (port 7803), agent to OMS (port 4889 upload), and emcli to OMS.

Browser `NET::ERR_CERT_AUTHORITY_INVALID` errors indicate the browser does not trust the OEM certificate authority. Import the OEM CA certificate into the browser's trusted CA store. The CA certificate is at:

```
$OMS_HOME/sysman/config/cacerts/
```

If agents or emcli cannot connect due to certificate trust failures:

```bash
# Re-secure OMS — regenerates OMS certificates
$OMS_HOME/bin/emctl secure oms

# After re-securing OMS, re-secure all agents
$AGENT_HOME/bin/emctl secure agent
```

For a Server Load Balancer (SLB) in front of OMS, include SLB port overrides when re-securing:

```bash
$OMS_HOME/bin/emctl secure oms -host <oms-host> \
  -slb_port 4903 \
  -slb_console_port 443 \
  -sysman_pwd <password> \
  -reg_pwd <reg_password>
```

For custom CA-signed certificates, the import procedure is documented in the OEM Security Guide (see Sources). Certificates must be imported into the WebLogic identity keystore (`$MIDDLEWARE_HOME/user_projects/domains/GCDomain/config/fmwconfig/`) and the OMS wallet.

---

## Log File Locations

### OMS logs

| Log | Path |
|---|---|
| OMS application log | `$OMS_HOME/sysman/log/emoms.log` |
| OMS trace log | `$OMS_HOME/sysman/log/emoms.trc` |
| OMS patcher/upgrade log | `$OMS_HOME/sysman/log/emoms_patcher.log` |
| OMS secure log | `$OMS_HOME/sysman/log/secure.log` |

### WebLogic logs

| Log | Path |
|---|---|
| WebLogic AdminServer log | `$MIDDLEWARE_HOME/user_projects/domains/GCDomain/servers/EMGC_ADMINSERVER/logs/EMGC_ADMINSERVER.log` |
| OMS managed server log | `$MIDDLEWARE_HOME/user_projects/domains/GCDomain/servers/EMGC_OMS1/logs/EMGC_OMS1.log` |
| WebLogic domain log | `$MIDDLEWARE_HOME/user_projects/domains/GCDomain/logs/GCDomain.log` |
| WebLogic HTTP access log | `$MIDDLEWARE_HOME/user_projects/domains/GCDomain/servers/EMGC_OMS1/logs/access.log` |

### Agent logs

| Log | Path |
|---|---|
| Agent runtime log | `$AGENT_HOME/agent_inst/sysman/log/emagent.log` |
| Agent trace log | `$AGENT_HOME/agent_inst/sysman/log/emagent.trc` |
| Upload log | `$AGENT_HOME/agent_inst/sysman/log/emagent_upload.log` |
| Agent secure log | `$AGENT_HOME/agent_inst/sysman/log/secure.log` |

### Config/upgrade logs

| Log | Path |
|---|---|
| OMS install/config | `$OMS_HOME/cfgtoollogs/oui/` |
| Repository upgrade (RepCA) | `$OMS_HOME/cfgtoollogs/repmanager/` |
| Plug-in lifecycle (OEM 13.5) | `$AGENT_HOME/sysman/log/plugins/` |

`$OMS_HOME` defaults to the OMS Oracle Home (e.g., `/u01/app/oracle/middleware/oms`).
`$MIDDLEWARE_HOME` is typically the same directory or a parent; check `$OMS_HOME/../../` if the paths differ in your installation.
`$AGENT_HOME` defaults to the agent Oracle Home (e.g., `/u01/app/oracle/agent13c/agent_13.5.0.0.0`).

---

## Key Diagnostic Commands

### emctl — OMS operations

All `emctl` OMS commands run as the OMS software owner from `$OMS_HOME/bin/`.

| Purpose | Command |
|---|---|
| OMS status (brief) | `emctl status oms` |
| OMS status with OMR and WebLogic details | `emctl status oms -details` |
| Start OMS (WebLogic AdminServer + managed servers) | `emctl start oms` |
| Stop OMS managed servers only | `emctl stop oms` |
| Stop OMS and WebLogic AdminServer | `emctl stop oms -all` |
| Check OMS binary version | `emctl version oms` |
| Re-generate OMS certificates | `emctl secure oms` |
| Display OMS agent registration password | `emctl get_reg_pwd oms` |
| Reconfigure OMS repository connection | `emctl config oms -store_repos_details -repos_conndesc "..." -repos_user sysman` |

### emctl — Agent operations

All agent `emctl` commands run as the agent software owner on the monitored host from `$AGENT_HOME/bin/`.

| Purpose | Command |
|---|---|
| Agent status | `emctl status agent` |
| Start agent | `emctl start agent` |
| Stop agent | `emctl stop agent` |
| Force immediate metric upload | `emctl upload agent` |
| Re-secure agent against current OMS | `emctl secure agent` |
| Re-secure agent with explicit password | `emctl secure agent -reg_passwd <password>` |
| Clear pending upload queue (data loss) | `emctl clearstate agent` |
| List deployed plug-ins | `emctl listplugins agent -type all` |
| Run a specific metric collection | `emctl control agent runCollection <target_name>:<target_type> <metric_name>` |
| Reload agent configuration | `emctl reload agent` |

### emcli — Enterprise Manager CLI

emcli connects to the OMS and requires a prior setup and login session. Run from any host that can reach the OMS on port 7803.

```bash
# Initial setup (run once per environment)
emcli setup -url=https://<oms-host>:7803/em -username=sysman

# Log in before running other verbs
emcli login -username=sysman

# Sync emcli client with OMS (downloads verb list)
emcli sync
```

Common diagnostic verbs:

| Purpose | Command |
|---|---|
| List all agents and status | `emcli get_agents` |
| List targets by type | `emcli get_targets -targets="oracle_database"` |
| Unblock an agent | `emcli unblock_agent -agent="<host>:<port>"` |
| List active blackouts | `emcli get_blackouts` |
| List plug-ins deployed on an agent | `emcli list_plugins_on_agent -agent_names="<host>:<port>"` |
| Resync agent | `emcli resync_agent -agent="<host>:<port>"` |

Full verb reference: `emcli help` or the OEM Command Line Interface guide (see Sources).

---

## Oracle Version Notes (OEM 13.4 vs 13.5)

### OEM 13.4

- Terminal release for OEM 13.4 is Bundle Patch 20 (13.4.0.20). Oracle recommends upgrading to 13.5 for continued support.
- Certified OMR database versions: Oracle Database 12.2 through 19c. Oracle Database 19c is the recommended OMR for 13.4 deployments.
- Ships with WebLogic 12.2.1.3 in the GCDomain. WebLogic log paths listed above apply.
- Agent version 13.4.x agents can operate against an OEM 13.5 OMS during a phased upgrade period. Mixed-version deployments are supported temporarily during migration only.
- The `emcli setup` command stores session files under `$ORACLE_HOME/bin/`. If upgrading from 13.4 to 13.5, re-run `emcli setup` after the upgrade.

### OEM 13.5

- Ships with WebLogic 12.2.1.4 in the GCDomain. Verify the WebLogic version before applying WebLogic-specific workarounds:
  ```bash
  $OMS_HOME/oracle_common/common/bin/wlst.sh -e "print weblogic.version"
  ```
- JDK 8 is the baseline certified JDK for OEM 13.5 OMS. JDK 11 support was introduced in OEM 13.5.0.15 RU. Do not change JVM flags without verifying the installed JDK version (`java -version`).
- Certified OMR database versions: Oracle Database 12.2 through 21c for initial 13.5.0 releases. Check My Oracle Support for updated certification as newer database releases are qualified.
- Agent 13.5 introduced enhanced plug-in lifecycle logging. The per-plug-in deployment log is at:
  ```
  $AGENT_HOME/sysman/log/plugins/
  ```
- The `emcli setup` session file location changed in OEM 13.5. If emcli commands return authorization errors after upgrading from 13.4, re-run:
  ```bash
  emcli setup -url=https://<oms-host>:7803/em -username=sysman
  ```
- When upgrading OMS from 13.4 to 13.5, the installer automatically runs the repository schema upgrade. If the upgrade fails midway, check `$OMS_HOME/cfgtoollogs/repmanager/` before attempting to restart the OMS with the new binary.

---

## Sources

- Oracle Enterprise Manager Cloud Control 13c Administrator's Guide:
  https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emadm/index.html

- Oracle Enterprise Manager Cloud Control 13c Security Guide:
  https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emsec/index.html

- Oracle Enterprise Manager Cloud Control 13c Basic Installation Guide:
  https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/embsc/index.html

- Oracle Enterprise Manager Cloud Control 13c Upgrade Guide:
  https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emupg/index.html

- Oracle Enterprise Manager Command Line Interface (emcli) Reference:
  https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emcli/index.html

- Oracle Enterprise Manager Cloud Control 13.5 Release Notes:
  https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emrns/index.html

- Oracle Enterprise Manager Cloud Control 13.4 Release Notes:
  https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.4/emrns/index.html

- My Oracle Support — Enterprise Manager 13c Master Note for OMS Startup Issues (Doc ID 1495929.1):
  https://support.oracle.com/rs?type=doc&id=1495929.1

- My Oracle Support — Enterprise Manager 13c Agent Troubleshooting Guide (Doc ID 1914184.1):
  https://support.oracle.com/rs?type=doc&id=1914184.1
