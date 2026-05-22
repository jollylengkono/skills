# Oracle Enterprise Manager 13c Upgrade

## Overview

This skill covers upgrading Oracle Enterprise Manager from OEM 13.4 to OEM 13.5. An in-place upgrade replaces the OMS binaries, upgrades the Oracle Management Repository (OMR) schema, and migrates configuration data in a single installer session. After the OMS upgrade completes, Management Agents are upgraded separately using mass upgrade tooling.

The three major phases are:

1. **Pre-upgrade** — backup, compatibility checks, and environment validation.
2. **OMS upgrade** — running the OEM 13.5 installer against the existing 13.4 OMS home and OMR.
3. **Post-upgrade** — agent mass upgrade, plug-in re-deployment, and health validation.

Upgrading an OEM deployment is a destructive, in-place operation. A tested OMR backup is the only reliable rollback path.

---

## Core Concepts

### Component Upgrade Order

Components must be upgraded in this order:

1. OMR database (if a database patch is required as part of the upgrade prerequisites)
2. OMS (the installer handles the OMR schema upgrade automatically)
3. Management Agents (after OMS is confirmed healthy)
4. Plug-ins (re-deployed to agents after OMS upgrade)

Agents can continue running against the old OMS during the OMS upgrade window; OEM 13.4 agents are compatible with an OEM 13.5 OMS until the agent upgrade is completed. Do not upgrade agents before the OMS upgrade is complete.

### OMR Schema Upgrade

The OEM 13.5 installer runs the OMR schema upgrade automatically as part of the upgrade session. The schema upgrade modifies the `SYSMAN` schema and several related schemas (`SYSMAN_MDS`, `SYSMAN_APM`, `SYSMAN_OPSS`). The installer cannot be run without modifying these schemas; there is no option to defer the schema upgrade.

### Plug-in Compatibility

Plug-ins deployed on OEM 13.4 must be at a version compatible with OEM 13.5 before the OMS upgrade begins. Incompatible plug-ins block the installer. The pre-upgrade console check and installer prerequisite check both report incompatible plug-in versions.

---

## Pre-Upgrade Checklist

Complete every item in this checklist before launching the OEM 13.5 installer.

### 1. Review the Upgrade Guide

Read the Oracle Enterprise Manager Cloud Control Upgrade Guide for 13.5 before proceeding. The guide lists certified source versions, known issues, and any upgrade-specific patches that must be applied before starting.

```
https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emupg/
```

### 2. Confirm the Source Release

OEM 13.5 supports direct upgrade from OEM 13.4. Verify your current release from the console (Help > About) or from the command line:

```bash
$OMS_HOME/bin/emctl status oms -details
```

The output includes the OMS version string. Upgrade directly from earlier releases (13.3 and below) is not supported to 13.5 without an intermediate step to 13.4 first.

### 3. Back Up the OMR Database

Take a full consistent RMAN backup of the OMR database. This is the only supported rollback mechanism if the upgrade fails after the schema upgrade begins.

```bash
# Connect to RMAN as a privileged user
rman target /

# Full backup with archive logs
BACKUP DATABASE PLUS ARCHIVELOG;
```

Record the backup set location and confirm the backup is restorable before proceeding. Also record the current SYSMAN schema version:

```sql
-- Run as SYSMAN or SYS against the OMR
SELECT param_value FROM mgmt_versions
WHERE param_name = 'REPOS_VERSION';
```

### 4. Export SYSMAN Schema (Optional Logical Backup)

For environments that require a logical backup in addition to RMAN:

```bash
expdp system/password@omr_service \
  schemas=SYSMAN,SYSMAN_MDS,SYSMAN_APM,SYSMAN_OPSS \
  directory=DATA_PUMP_DIR \
  dumpfile=oem_presupgrade_%U.dmp \
  logfile=oem_presupgrade_expdp.log
```

### 5. Check Plug-in Compatibility

From the OEM 13.5 Upgrade Guide, identify which plug-in versions are supported. Check currently deployed plug-in versions in the OEM console or via emcli:

```bash
emcli list_plugins_on_server
```

For any plug-in that is not supported on OEM 13.5, either upgrade the plug-in to a compatible version before the OMS upgrade, or undeploy it. The OEM 13.5 installer blocks the upgrade if an incompatible plug-in is detected and deployed.

### 6. Run the Pre-Upgrade Console Check

OEM provides a built-in pre-upgrade check accessible from the console:

**Navigation:** Setup > Manage Cloud Control > Upgrade

The pre-upgrade check reports:
- Incompatible plug-ins
- OMR database version compatibility
- Disk space on the OMS host
- Missing prerequisite patches

Resolve all reported issues before continuing.

### 7. Verify OMR Database Prerequisites

The OMR database must meet the minimum Oracle Database version and patch level required by OEM 13.5. Confirm the OMR database version:

```sql
SELECT version FROM v$instance;
```

OEM 13.5 requires Oracle Database 12.2 or later for the OMR. Oracle Database 19c is the recommended OMR database version for new and upgraded deployments.

Check that the OMR tablespaces have sufficient free space for the schema upgrade:

```sql
SELECT tablespace_name,
       ROUND(SUM(bytes)/1048576,1) AS free_mb
FROM dba_free_space
WHERE tablespace_name IN (
  'MGMT_TABLESPACE','MGMT_ECM_DEPOT_TS','MGMT_AD4J_TS'
)
GROUP BY tablespace_name;
```

Add space to any tablespace that is near capacity before starting the upgrade.

### 8. Shut Down the OMS

Stop all OMS instances before running the installer. In a multi-OMS deployment, stop all secondary OMS instances first, then stop the primary OMS last.

```bash
# Run on each OMS host; use -all to stop AdminServer as well
$OMS_HOME/bin/emctl stop oms -all

# Verify OMS is stopped
$OMS_HOME/bin/emctl status oms
```

The OMR database must remain running throughout the upgrade. Do not shut down the OMR database.

### 9. Verify Disk Space on OMS Host

The OEM 13.5 installer requires free space in the following locations:

| Location | Minimum Free Space |
|----------|--------------------|
| OMS home parent directory | 12 GB |
| `/tmp` | 4 GB |
| Middleware home | 3.5 GB |

Check available space:

```bash
df -h $OMS_HOME /tmp
```

### 10. Download the OEM 13.5 Installer

Download the OEM 13.5 software from Oracle Software Delivery Cloud (eDelivery) or from My Oracle Support. The installer ships as a set of zip files. Unzip all parts into a single staging directory on the OMS host before running.

```bash
unzip em13500_linux64.bin -d /stage/em135
# Additional zip parts if present:
# unzip em13500_linux64-2.zip -d /stage/em135
```

---

## OMS Upgrade — Installer Modes

The OEM 13.5 installer supports two upgrade modes.

### Graphical (GUI) Mode

Launch the installer with the graphical wizard when a display is available:

```bash
/stage/em135/em13500_linux64.bin
```

The wizard guides through:
1. Installation type selection — choose **Upgrade an Existing Enterprise Manager System**.
2. Source OMS home detection (auto-detected from inventory).
3. OMR connection details.
4. Plug-in selection (add new plug-ins or carry forward existing compatible ones).
5. Review and confirmation.
6. Prerequisite check (runs automatically; blocks on failures).
7. Upgrade execution.

### Silent Mode

Silent mode is used for automated, scripted, or headless upgrades. Prepare a response file from the GUI run or from the template provided in the installer documentation, then run:

```bash
/stage/em135/em13500_linux64.bin -silent \
  -responseFile /stage/em135/upgrade.rsp \
  -invPtrLoc /etc/oraInst.loc
```

Key response file parameters for an upgrade:

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

Do not hardcode passwords in a response file stored in a shared location. Use a restricted-permission file (`chmod 600`) and remove it after the upgrade completes.

### Installer Behavior During Upgrade

The installer:
1. Validates prerequisites (aborts if any critical check fails).
2. Backs up the existing OMS configuration (stored under the installer scratch directory).
3. Installs 13.5 OMS binaries into the existing OMS home.
4. Runs the OMR schema upgrade SQL scripts (this step can take 30–90 minutes depending on OMR size).
5. Reconfigures the WebLogic domain.
6. Starts the OMS.

The installer writes detailed logs to:
```
$OMS_HOME/cfgtoollogs/oui/
$OMS_HOME/sysman/log/schemamanager/
```

Monitor these logs in a separate terminal during the upgrade session:

```bash
tail -f $OMS_HOME/sysman/log/schemamanager/m_*.log
```

---

## OMR Schema Upgrade

The OMR schema upgrade is performed automatically by the installer. It is not a separate manual step in a standard upgrade. However, understanding what the installer does helps with diagnosis if the upgrade stalls.

The installer runs the Schema Manager utility, which:
1. Connects to the OMR as `SYS` using the credentials provided in the installer.
2. Applies DDL and DML scripts to upgrade `SYSMAN` and associated schemas.
3. Records progress in `MGMT_COMPONENT_VERSIONS`.

If the installer exits abnormally during the schema upgrade, do not restart the OMS with the partially upgraded schema. Instead:

1. Review the Schema Manager log in `$OMS_HOME/sysman/log/schemamanager/`.
2. Identify the last completed step.
3. Contact Oracle Support before attempting to continue or roll back manually.
4. If the OMR schema is corrupt or inconsistent, restore from the RMAN backup taken in the pre-upgrade checklist.

---

## Agent Mass Upgrade Post-OMS Upgrade

After the OMS upgrade is confirmed healthy, upgrade all Management Agents from 13.4 to 13.5. OEM 13.4 agents continue to report to a 13.5 OMS during this window, but they cannot use features introduced in 13.5 until upgraded.

### Recommended Approach: Agent Gold Image

1. Identify or create a reference agent that has been upgraded to 13.5 and validated.
2. Create a Gold Image from the reference agent:

   **Navigation:** Setup > Manage Cloud Control > Gold Agent Images > Create

3. Promote the image to Active status.
4. Subscribe existing 13.4 agents to the new Gold Image.
5. Submit the Mass Agent Upgrade job:

   **Navigation:** Setup > Manage Cloud Control > Upgrade Agents

   Or via emcli:

   ```bash
   emcli submit_job \
     -job_type="MASS_AGENT_UPGRADE" \
     -input_file="agents_to_upgrade.txt"
   ```

   The input file lists one agent target name per line.

6. Monitor job progress:

   **Navigation:** Enterprise > Job Activity > select the Mass Agent Upgrade job

   Or:

   ```bash
   emcli get_jobs -status=RUNNING
   ```

### Alternative: Self Update

For environments that use Self Update for agent patches:

1. Open Self Update: **Setup > Extensibility > Self Update**
2. Select **Management Agent** and download the 13.5 agent software for each required platform.
3. Apply the update to selected agents or all agents of a platform type.

### Verify Agent Upgrade

After the mass upgrade job completes, confirm agents are at the expected version:

```bash
# Check agent version on a specific host
$AGENT_HOME/bin/emctl status agent

# Or query from OMS
emcli get_targets -targets="oracle_emd" -noheader
```

From the console: **Setup > Manage Cloud Control > Agents** — verify the Version column shows 13.5.x.

---

## Plug-in Re-Deployment After Upgrade

Plug-ins upgraded or added during the OMS upgrade are available on the OMS, but agents continue to run the previous plug-in version until the plug-in is explicitly re-deployed to them.

### Check Plug-in Deployment Status

```bash
emcli list_plugins_on_server
```

Compare OMS plug-in versions with agent-deployed versions:

**Navigation:** Setup > Extensibility > Plug-ins > Deployment Status tab

### Re-Deploy Plug-ins to Agents

```bash
emcli deploy_plugin_on_agent \
  -agent_names="<agent1>,<agent2>,..." \
  -plugin="<plugin_id>:<version>"
```

For a large estate, iterate over all agents for each plug-in using an emcli script, or use the console mass deployment wizard:

**Navigation:** Setup > Extensibility > Plug-ins > select plug-in > Deploy on Agent

Agents must be reachable and in UP status before plug-in re-deployment. Address any unreachable agents before attempting bulk plug-in re-deployment.

Plug-in versions on agents must not exceed the plug-in version deployed on the OMS.

---

## Post-Upgrade Validation

Run these checks after the OMS upgrade and after all agents have been upgraded.

### OMS Status

```bash
$OMS_HOME/bin/emctl status oms
$OMS_HOME/bin/emctl status oms -details
```

Expected output includes:
- Oracle Management Service is Up
- WebLogic AdminServer and the Managed Server (EMGC_OMS1) are listed as running
- Repository connection is shown as active

### Agent Status

On each upgraded agent host:

```bash
$AGENT_HOME/bin/emctl status agent
```

Expected output: `Oracle Management Agent is Running and Ready`

For spot-checking multiple agents from the OMS, query via emcli:

```bash
emcli get_targets -targets="oracle_emd" -noheader | awk '{print $1, $4}'
```

### Console Health Check

1. Log in to the OEM 13.5 console.
2. Verify the home page loads without JavaScript errors.
3. Navigate to **Setup > About Enterprise Manager** and confirm the OMS version shows 13.5.x.
4. Check that all previously monitored targets appear with current metric status.
5. Confirm no critical upgrade-related incidents are open in the Incident Manager.

### OMR Schema Version

```sql
-- Run against the OMR as SYSMAN or SYS
SELECT param_value FROM mgmt_versions
WHERE param_name = 'REPOS_VERSION';
```

The value should reflect the OEM 13.5 schema version. Compare to the expected version in the OEM 13.5 Upgrade Guide.

### Installed Patches and Versions

```bash
# OMS home patch inventory
$OMS_HOME/OPatch/opatch lspatches

# Confirm OMS binary version
$OMS_HOME/bin/emctl status oms -details | grep "OMS Version"
```

---

## Best Practices and Common Mistakes

### Do

- Take and verify an RMAN backup of the OMR before starting. This is the only reliable rollback path.
- Stop all OMS instances before running the installer.
- Resolve all plug-in compatibility issues before starting the upgrade — incompatible plug-ins cannot be resolved mid-upgrade.
- Upgrade agents after the OMS is confirmed healthy, not simultaneously.
- Monitor the Schema Manager log throughout the upgrade to detect stalls early.
- Use a response file with `chmod 600` permissions for silent upgrades and delete it after the upgrade.

### Do Not

- Do not shut down the OMR database during the upgrade. The installer requires the OMR to be running and accessible throughout.
- Do not attempt to start the OMS with a partially upgraded OMR schema if the installer aborted mid-schema upgrade. Restore from backup instead.
- Do not upgrade agents before the OMS upgrade is complete. An agent cannot run a higher OEM version than its OMS.
- Do not apply standard OMS Bundle Patches as a substitute for an upgrade. Patching 13.4 does not produce a 13.5 OMS; the full upgrade installer is required.
- Do not run the upgrade installer as `root`. Run as the OMS software owner OS user.

### Common Mistakes

**Insufficient OMR tablespace free space**: The schema upgrade requires free space in `MGMT_TABLESPACE` and related tablespaces. Check free space before starting and add space if any tablespace is under 20% free.

**Forgetting to stop secondary OMS instances in a multi-OMS setup**: The installer will warn if it detects another OMS connected to the OMR. Stop all OMS instances before upgrading.

**Plug-in version mismatch not caught until installer pre-check**: Run `emcli list_plugins_on_server` and cross-reference with the OEM 13.5 certification matrix days before the upgrade window to allow time to update incompatible plug-ins.

**Agent credentials not set for mass upgrade job**: The mass agent upgrade job requires valid preferred credentials on each target host. Pre-validate host credentials before the upgrade window.

---

## Oracle Version Notes

### Upgrading to OEM 13.4

OEM 13.4 supported direct upgrade from OEM 13.3.x. The OEM 13.4 installer introduced automated pre-upgrade advisor checks accessible from within the existing 13.3 console before initiating the upgrade. Plug-in compatibility requirements for 13.4 were less strict than for 13.5; several plug-ins that required manual attention for a 13.5 upgrade were automatically carried forward by the 13.4 installer.

OEM 13.4 also introduced the ability to perform an upgrade in a Software Only + Configuration Assistant mode, separating binary installation from OMS configuration across maintenance windows.

### Upgrading to OEM 13.5

OEM 13.5 requires that all deployed plug-ins be at versions explicitly certified for 13.5. The installer prerequisite check is stricter than in 13.4 and will block the upgrade for any incompatible plug-in version, including plug-in versions that were acceptable for 13.4.

OEM 13.5 moved from the Bundle Patch (BP) naming convention to the Release Update (RU) naming convention for post-upgrade patches, aligning with Oracle Database patching terminology. Post-upgrade patches applied to a 13.5 OMS use `omspatcher` with Release Update patch IDs rather than the Bundle Patch IDs used in 13.4.

The OMR database version requirement was tightened for OEM 13.5: Oracle Database 12.2 is the minimum supported OMR version, and Oracle Database 19c is the recommended baseline. OMR databases running Oracle Database 12.1 must be upgraded before the OEM 13.5 OMS upgrade can proceed.

OEM 13.5 also introduced enhanced upgrade pre-check tooling, including an offline pre-upgrade report that can be generated and reviewed before the maintenance window begins.

---

## Sources

- Oracle Enterprise Manager Cloud Control 13c Upgrade Guide (13.5): https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emupg/
- Oracle Enterprise Manager Cloud Control 13c: Upgrading Oracle Management Agent — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emupg/upgrading-oracle-management-agent.html
- Oracle Enterprise Manager Cloud Control 13c: Overview of the Upgrade Process — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emupg/overview-upgrade-process-enterprise-manager.html
- Oracle Enterprise Manager Cloud Control 13c: Plug-ins and the Upgrade Process — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emupg/plug-ins-and-upgrade-process.html
- Oracle Enterprise Manager Cloud Control 13c: Managing Gold Agent Images — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emadm/managing-gold-agent-images.html
- Oracle Enterprise Manager Cloud Control 13c documentation hub: https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/
- Oracle Enterprise Manager 13c: All Releases — https://docs.oracle.com/en/enterprise-manager/
