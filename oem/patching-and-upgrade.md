# Oracle Enterprise Manager 13c Patching and Upgrade

## Overview

This skill covers two related lifecycle operations for Oracle Enterprise Manager 13c:

- **Patching** — applying Bundle Patches, Release Updates, one-off patches, plug-in patches, and agent mass patches to an existing OEM 13c deployment without changing the major release version.
- **Upgrading** — moving from OEM 13.4 to OEM 13.5 using the full OEM installer, which replaces OMS binaries and upgrades the Oracle Management Repository (OMR) schema.

Both operations follow a component sequencing rule: OMR database first, then OMS, then Management Agents, then plug-ins. Agents must never run at a higher OEM version than the OMS they report to.

---

## Part 1: Patching

### Patch Types

| Patch Type | Description |
|---|---|
| Bundle Patch (BP) | Cumulative set of fixes for OMS, Agent, and plug-ins released on a quarterly schedule |
| Release Update (RU) | Platform-level cumulative patch that supersedes previous BPs; introduced for OEM 13.5 |
| One-off patch | Single fix for a specific bug; applied on top of an installed BP or RU |
| Plug-in patch | Patch specific to a deployed plug-in; applied separately from the OMS patch |
| OMR patch | Standard Oracle Database patch (RU or one-off) applied to the repository database using OPatch |

### Patching Workflow

The recommended sequence is:

1. Back up the OMR database and OMS configuration.
2. Patch the OMR database (apply DB RU/one-off to the repository).
3. Patch the OMS using `omspatcher` / OPatch.
4. Patch deployed plug-ins.
5. Upgrade Management Agents using Self Update or Agent Gold Image.
6. Verify all components and restart services.

Do not patch Agents before the OMS patch is complete. Agents must not run a higher plug-in version than the OMS.

### Prerequisites Before Patching

#### OPatch and OMSPatcher Versions

OEM 13c patching uses two tools:

- **OPatch** — for OMR database patches and standalone OMS patches.
- **OMSPatcher** — for OMS-level patches; it wraps OPatch and handles multi-OMS environments.

Before applying any patch, verify that both tools meet the minimum version stated in the patch README. Upgrade OPatch by downloading the latest version from MOS (patch ID listed in the patch README) and replacing the existing `$ORACLE_HOME/OPatch` directory.

```bash
# Check OPatch version
$OMS_HOME/OPatch/opatch version

# Check OMSPatcher version
$OMS_HOME/OMSPatcher/omspatcher version
```

#### Pre-Patch System Checks

```bash
# Analyze only — do not apply
$OMS_HOME/OMSPatcher/omspatcher prereq -patch_top <patch_staging_dir>
```

For OPatch-based patches on the OMR:

```bash
$ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -ph <patch_dir>
```

#### Back Up the Oracle Management Repository

Take a full consistent backup of the OMR database before every patching session. At minimum:

1. Take an RMAN full backup of the OMR.
2. Export the `SYSMAN` schema if a logical backup is also required.
3. Record the current OMS version string from `emcli get_targets` or the EM console (About page).

```sql
-- Confirm OMR tablespace health before patching
SELECT tablespace_name, status FROM dba_tablespaces
WHERE tablespace_name IN ('MGMT_TABLESPACE','MGMT_AD4J_TS');
```

#### Stop OMS Before Patching

```bash
# Stop all OMS instances (run on each OMS host)
$OMS_HOME/bin/emctl stop oms -all

# Verify status
$OMS_HOME/bin/emctl status oms
```

### OMS Patching with OMSPatcher

OMSPatcher is the primary tool for applying Bundle Patches and one-off patches to the OMS home.

```bash
# Download and stage the patch from support.oracle.com, then:
cd <staging_dir>/<patch_number>

# Dry run first
$OMS_HOME/OMSPatcher/omspatcher apply -analyze

# Apply
$OMS_HOME/OMSPatcher/omspatcher apply
```

OMSPatcher handles stopping/restarting the OMS automatically (single-OMS), updating the OMR schema, and patching the WebLogic AdminServer domain.

For multi-OMS deployments, apply on the primary OMS first, then each additional OMS. The OMR schema update runs only once.

```bash
# Restart and verify
$OMS_HOME/bin/emctl start oms
$OMS_HOME/OPatch/opatch lspatches
```

### Agent Patching — Self Update and Agent Gold Image

#### Self Update

**Navigation:** Setup > Extensibility > Self Update

Steps:
1. Select **Management Agent** and check for available updates.
2. Download the patch to the Software Library.
3. Apply to selected agent targets or all agents of a given platform.

Ensure the Software Library is healthy before starting:

```bash
emcli get_software_libraries
```

#### Agent Gold Image

**Navigation:** Setup > Manage Cloud Control > Gold Agent Images

Workflow:
1. Create a Gold Image from a reference agent with all desired patches applied.
2. Promote the image to Active status.
3. Subscribe target agents to the Gold Image.
4. Use the Mass Agent Upgrade job to push the image to subscribed agents.

```bash
emcli submit_job -job_type="MASS_AGENT_UPGRADE" \
  -input_file="agents.txt" \
  -gold_agent_image="<image_name>"
```

#### Manual Agent Patching (emcli)

```bash
emcli get_targets -targets="oracle_emd"
emcli submit_job -job_type="PATCH_AGENT" \
  -target_list="<agent_target_name>:oracle_emd" \
  -input_file="patch_params.properties"
```

### Plug-in Patching

Plug-in patches must be applied after the OMS patch.

```bash
# Check deployed plug-in versions
emcli list_plugins_on_server

# Stop OMS, apply plug-in patch
$OMS_HOME/bin/emctl stop oms -all
$OMS_HOME/OMSPatcher/omspatcher apply -patch_top <plug_in_patch_dir>
$OMS_HOME/bin/emctl start oms

# Redeploy updated plug-in to agents
emcli deploy_plugin_on_agent \
  -agent_names="<agent1>,<agent2>" \
  -plugin="<plugin_id>:<version>"
```

Plug-in versions on agents must not exceed the version deployed on the OMS.

### OMR Database Patch Application

Apply Database Release Updates or one-off patches to the OMR using OPatch.

```bash
# Stop OMS first, then as the DB software owner:
$ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -ph <patch_dir>
$ORACLE_HOME/OPatch/opatch apply <patch_dir>

# If the patch includes datapatch steps, run after the database is open:
cd $ORACLE_HOME/OPatch
./datapatch -verbose
```

The OMR database must remain at a version and patch level supported by OEM 13c. Always stop the OMS before patching the OMR.

### Self Update — Online and Offline

**Online mode**: Setup > Extensibility > Self Update > Check Updates

**Offline mode** (air-gapped):
1. Download the catalog bundle from MOS on an internet-connected host.
2. Transfer to the OMS host.
3. Import:

```bash
emcli import_update_catalog -file="<catalog_bundle>" -omslocal
```

---

## Part 2: Upgrade (OEM 13.4 to OEM 13.5)

Upgrading is an in-place operation that replaces OMS binaries and upgrades the OMR schema. A tested OMR backup is the only reliable rollback path.

### Component Upgrade Order

1. OMR database (if a prerequisite DB patch is required)
2. OMS (the installer handles OMR schema upgrade automatically)
3. Management Agents (after OMS is confirmed healthy)
4. Plug-ins (re-deployed to agents after OMS upgrade)

OEM 13.4 agents can continue reporting to a 13.5 OMS during the agent upgrade window. Do not upgrade agents before the OMS upgrade is complete.

### Pre-Upgrade Checklist

#### 1. Confirm the Source Release

OEM 13.5 supports direct upgrade from OEM 13.4 only. Verify current release:

```bash
$OMS_HOME/bin/emctl status oms -details
```

Upgrade from OEM 13.3 or earlier requires an intermediate step to 13.4 first.

#### 2. Back Up the OMR Database

```bash
rman target /
BACKUP DATABASE PLUS ARCHIVELOG;
```

Record current SYSMAN schema version:

```sql
SELECT param_value FROM mgmt_versions
WHERE param_name = 'REPOS_VERSION';
```

#### 3. Optional Logical Backup

```bash
expdp system/password@omr_service \
  schemas=SYSMAN,SYSMAN_MDS,SYSMAN_APM,SYSMAN_OPSS \
  directory=DATA_PUMP_DIR \
  dumpfile=oem_presupgrade_%U.dmp \
  logfile=oem_presupgrade_expdp.log
```

#### 4. Check Plug-in Compatibility

```bash
emcli list_plugins_on_server
```

Any plug-in not certified for OEM 13.5 must be upgraded or undeployed before the OMS upgrade begins. The 13.5 installer blocks on incompatible plug-in versions.

#### 5. Run the Pre-Upgrade Console Check

**Navigation:** Setup > Manage Cloud Control > Upgrade

Reports: incompatible plug-ins, OMR database version, disk space, missing prerequisite patches. Resolve all issues before continuing.

#### 6. Verify OMR Database Prerequisites

```sql
SELECT version FROM v$instance;
```

OEM 13.5 requires Oracle Database 12.2 or later for the OMR. Oracle Database 19c is the recommended OMR version.

Check tablespace free space:

```sql
SELECT tablespace_name,
       ROUND(SUM(bytes)/1048576,1) AS free_mb
FROM dba_free_space
WHERE tablespace_name IN (
  'MGMT_TABLESPACE','MGMT_ECM_DEPOT_TS','MGMT_AD4J_TS'
)
GROUP BY tablespace_name;
```

#### 7. Stop All OMS Instances

```bash
$OMS_HOME/bin/emctl stop oms -all
$OMS_HOME/bin/emctl status oms
```

Stop secondary OMS instances first, then the primary. The OMR database must remain running throughout.

#### 8. Verify Disk Space

| Location | Minimum Free Space |
|----------|--------------------|
| OMS home parent directory | 12 GB |
| `/tmp` | 4 GB |
| Middleware home | 3.5 GB |

```bash
df -h $OMS_HOME /tmp
```

#### 9. Download and Stage the OEM 13.5 Installer

Download from Oracle Software Delivery Cloud (eDelivery) or MOS. Unzip all parts into a single staging directory:

```bash
unzip em13500_linux64.bin -d /stage/em135
```

### OMS Upgrade — Installer Modes

#### GUI Mode

```bash
/stage/em135/em13500_linux64.bin
```

Select **Upgrade an Existing Enterprise Manager System**. The wizard auto-detects the source OMS home from inventory.

#### Silent Mode

```bash
/stage/em135/em13500_linux64.bin -silent \
  -responseFile /stage/em135/upgrade.rsp \
  -invPtrLoc /etc/oraInst.loc
```

Key response file parameters:

```ini
INSTALL_UPDATES_SELECTION=skip
UPGRADE_EXISTING_OMS=true
OMS_HOME=/u01/app/oracle/oms
DATABASE_HOSTNAME=omr-host.example.com
LISTENER_PORT=1521
SERVICENAME_OR_SID=EMREP
SYS_PASSWORD=<sys_password>
SYSMAN_PASSWORD=<sysman_password>
```

Use `chmod 600` on the response file and delete it after the upgrade.

The installer logs to:
```
$OMS_HOME/cfgtoollogs/oui/
$OMS_HOME/sysman/log/schemamanager/
```

Monitor during the upgrade:
```bash
tail -f $OMS_HOME/sysman/log/schemamanager/m_*.log
```

### OMR Schema Upgrade

The installer runs the Schema Manager automatically. It connects to the OMR as `SYS`, applies DDL and DML scripts to upgrade `SYSMAN`, `SYSMAN_MDS`, `SYSMAN_APM`, and `SYSMAN_OPSS`, and records progress in `MGMT_COMPONENT_VERSIONS`.

If the installer exits abnormally during the schema upgrade, do not start the OMS. Review the Schema Manager log, contact Oracle Support, and if necessary restore the OMR from the RMAN backup.

### Agent Mass Upgrade Post-OMS Upgrade

After the OMS is confirmed at 13.5, upgrade all Management Agents.

**Recommended: Agent Gold Image**

1. Create a Gold Image from a reference 13.5 agent.
2. Promote the image to Active status.
3. Subscribe 13.4 agents to the new Gold Image.
4. Submit the Mass Agent Upgrade job:

```bash
emcli submit_job \
  -job_type="MASS_AGENT_UPGRADE" \
  -input_file="agents_to_upgrade.txt"
```

Monitor: Enterprise > Job Activity

**Alternative: Self Update**

Setup > Extensibility > Self Update — download the 13.5 agent software for each platform, then apply to agents.

Verify after upgrade:

```bash
$AGENT_HOME/bin/emctl status agent
emcli get_targets -targets="oracle_emd" -noheader
```

### Plug-in Re-Deployment After Upgrade

Agents continue running the previous plug-in version until explicitly re-deployed.

```bash
emcli list_plugins_on_server
emcli deploy_plugin_on_agent \
  -agent_names="<agent1>,<agent2>,..." \
  -plugin="<plugin_id>:<version>"
```

**Navigation:** Setup > Extensibility > Plug-ins > Deployment Status tab

---

## Post-Patch / Post-Upgrade Verification

### OMS Health

```bash
$OMS_HOME/bin/emctl status oms
$OMS_HOME/bin/emctl status oms -details
```

### Patch and Version Inventory

```bash
# OMS home patches
$OMS_HOME/OPatch/opatch lspatches

# OMSPatcher composite patch list
$OMS_HOME/OMSPatcher/omspatcher lspatches
```

### OMR Schema Version

```sql
SELECT param_value FROM mgmt_versions
WHERE param_name = 'REPOS_VERSION';
```

### Agent Status

```bash
emcli get_targets -targets="oracle_emd" -noheader | grep "STATUS"
```

### Console Accessibility

Log in to the OEM console and confirm: home page loads, all targets appear with current status, version shown under Help > About matches expected version.

### Plug-in Versions

```bash
emcli list_plugins_on_server
```

---

## Rollback

### Rolling Back an OMS Patch

```bash
$OMS_HOME/bin/emctl stop oms -all
$OMS_HOME/OMSPatcher/omspatcher rollback -id <patch_number>
$OMS_HOME/bin/emctl start oms
```

If OMSPatcher rollback fails due to schema changes, restore from the OMR backup.

### Rolling Back an OMR Database Patch

```bash
$ORACLE_HOME/OPatch/opatch rollback -id <patch_number>
```

If datapatch steps were run, restore from RMAN backup — datapatch rollback is not always possible after the database has been open with the patched schema.

### Restoring from OMR Backup

1. Shut down the OMS completely.
2. Restore the OMR from the RMAN backup.
3. Open the database and verify `SYSMAN` is intact.
4. Start the OMS and confirm normal operation.

### Rolling Back Agent Patches

Agents patched via Gold Image can be reverted by reassigning the subscription to the previous Gold Image version and rerunning the mass upgrade job. For manually patched agents:

```bash
$AGENT_HOME/OPatch/opatch rollback -id <patch_number>
$AGENT_HOME/bin/emctl reload agent
```

There is no in-place rollback for an OEM version upgrade. Restore the OMR from the pre-upgrade RMAN backup, reinstall the 13.4 OMS binaries, and reconnect to the restored OMR.

---

## Common Errors and Resolutions

### OMSPatcher Fails: "OPatch version is not supported"

Download the required OPatch version from MOS and replace `$OMS_HOME/OPatch` before retrying.

### Pre-requisite Check Reports Conflicts

Run `$OMS_HOME/OPatch/opatch lsinventory`. Apply any prerequisite patches listed in the README. One-off patches superseded by the bundle can be rolled back first.

### "SYSMAN Schema Version Mismatch" After OMS Restart

Re-run OMSPatcher apply; it detects the schema is behind and re-executes the SQL steps.

### Agent Self Update Fails: "Software Library Not Configured"

**Navigation:** Setup > Provisioning and Patching > Software Library — verify the storage location is writable.

### Agent Upgrade Job Fails: Credential or SSH Error

**Navigation:** Setup > Security > Preferred Credentials > Host — verify host credentials and test connectivity.

### OMS Fails to Start After Patching: AdminServer Wallet Error

```bash
$OMS_HOME/bin/emctl secure oms -sysman_pwd <password>
$OMS_HOME/bin/emctl start oms
```

### "Inventory Lock Exists" Error in OPatch

Verify no OPatch process is running, then:

```bash
rm $ORACLE_HOME/.patch_storage/temp_lock
```

### datapatch Reports "Patch Already Applied"

Informational. No action required if all patches show `APPLIED` state:

```sql
SELECT patch_id, status, action_time FROM dba_registry_sqlpatch
ORDER BY action_time DESC;
```

### Insufficient OMR Tablespace Free Space During Upgrade

Add space to `MGMT_TABLESPACE` and related tablespaces before the upgrade window. Any tablespace below 20% free is at risk.

### Plug-in Version Mismatch Blocks Upgrade Installer

Run `emcli list_plugins_on_server` and cross-reference with the OEM 13.5 certification matrix before the maintenance window. Upgrade or undeploy incompatible plug-ins.

---

## Best Practices and Common Mistakes

- Take and verify an RMAN backup of the OMR before any patching or upgrade session — this is the only reliable rollback path for schema-modifying operations.
- Stop all OMS instances before patching or upgrading. For multi-OMS, stop secondary instances before the primary.
- Resolve plug-in compatibility issues before starting an upgrade — incompatible plug-ins cannot be resolved mid-upgrade.
- Never upgrade agents before the OMS patch or upgrade is complete and confirmed healthy.
- Monitor the Schema Manager log (`$OMS_HOME/sysman/log/schemamanager/`) throughout upgrade sessions to detect stalls early.
- Use `chmod 600` on silent-mode response files and delete them after the session.
- Do not shut down the OMR database during an OMS upgrade; the installer requires it running throughout.
- Do not run the installer or patching tools as `root`; use the OMS software owner OS user.

---

## Oracle Version Notes

- OEM 13.3 and earlier use Bundle Patch naming; OEM 13.5 introduced Release Update (RU) naming aligned with Oracle Database patching terminology.
- OMSPatcher replaced `opatchauto` for OMS patching starting with OEM 13.2. Agent Gold Image was also introduced in OEM 13.2.
- OEM 13.5 requires Oracle Database 12.2 or later for the OMR; Oracle Database 12.1 OMR databases must be upgraded before the OEM 13.5 upgrade can proceed. Oracle Database 19c is the recommended OMR baseline.
- OEM 13.5 introduced stricter plug-in compatibility enforcement in the upgrade installer compared to 13.4; plug-in versions that were carried forward automatically in a 13.4 upgrade may require manual attention in a 13.5 upgrade.
- OEM 13.4 introduced a Software Only + Configuration Assistant upgrade mode, separating binary installation from OMS configuration across maintenance windows.

---

## Sources

- Oracle Enterprise Manager Cloud Control 13c: Patching Oracle Management Service — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emupg/patching-oracle-management-service.html
- Oracle Enterprise Manager Cloud Control 13c: Patching Oracle Management Agents — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emupg/patching-oracle-management-agent.html
- Oracle Enterprise Manager Cloud Control 13c: Self Update — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emadm/self-update.html
- Oracle Enterprise Manager Cloud Control 13c: Agent Gold Images — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emadm/managing-gold-agent-images.html
- Oracle Enterprise Manager Cloud Control 13c Upgrade Guide (13.5) — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emupg/
- Oracle Enterprise Manager Cloud Control 13c: Overview of the Upgrade Process — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emupg/overview-upgrade-process-enterprise-manager.html
- Oracle Enterprise Manager Cloud Control 13c: Plug-ins and the Upgrade Process — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emupg/plug-ins-and-upgrade-process.html
- Oracle OPatch User's Guide for Oracle Database — https://docs.oracle.com/en/database/oracle/oracle-database/19/opunk/index.html
- MOS Note: Enterprise Manager Cloud Control Recommended Patches — Doc ID 1357376.1
- MOS Note: OMSPatcher User Guide — Doc ID 1982664.1
