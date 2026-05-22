# Oracle Enterprise Manager 13c Patching

## Overview

Oracle Enterprise Manager (OEM) 13c patching covers four principal components: the Oracle Management Service (OMS), the Oracle Management Repository (OMR) database, Management Agents, and deployed plug-ins. Each component has its own patch type, tooling, and sequencing requirements. Patching all components in the correct order prevents version incompatibilities and service disruptions.

This skill covers prerequisites, per-component workflows, post-patch verification, rollback, and common errors.

---

## Patch Types

| Patch Type | Description |
|---|---|
| Bundle Patch (BP) | Cumulative set of fixes for OMS, Agent, and plug-ins released on a quarterly schedule |
| Release Update (RU) | Platform-level cumulative patch that supersedes previous BPs; introduced for OEM 13.5 |
| One-off patch | Single fix for a specific bug; applied on top of an installed BP or RU |
| Plug-in patch | Patch specific to a deployed plug-in; applied separately from the OMS patch |
| OMR patch | Standard Oracle Database patch (RU or one-off) applied to the repository database using OPatch |

---

## Patching Workflow

The recommended patching sequence is:

1. Back up the OMR database and OMS configuration.
2. Patch the OMR database (apply DB RU/one-off to the repository).
3. Patch the OMS using `omspatcher` / OPatch.
4. Patch deployed plug-ins.
5. Upgrade Management Agents using Self Update or Agent Gold Image.
6. Verify all components and restart services.

Do not patch Agents before the OMS patch is complete. Agents must not run a higher plug-in version than the OMS.

---

## Prerequisites Before Patching

### OPatch and OMSPatcher Versions

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

### Pre-Patch System Checks

Run the built-in pre-requisite check before applying any patch. OMSPatcher will refuse to proceed if pre-requisites fail.

```bash
# Analyze only — do not apply
$OMS_HOME/OMSPatcher/omspatcher prereq -patch_top <patch_staging_dir>
```

For OPatch-based patches on the OMR:

```bash
$ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -ph <patch_dir>
```

### Back Up the Oracle Management Repository

Take a full consistent backup of the OMR database before every patching session. At minimum:

1. Take an RMAN full backup of the OMR.
2. Export the `SYSMAN` schema if a logical backup is also required.
3. Record the current OMS version string from `emcli get_targets` or the EM console (About page).

```sql
-- Confirm OMR tablespace health before patching
SELECT tablespace_name, status FROM dba_tablespaces
WHERE tablespace_name IN ('MGMT_TABLESPACE','MGMT_AD4J_TS');
```

### Stop OMS Before Patching

```bash
# Stop all OMS instances (run on each OMS host)
$OMS_HOME/bin/emctl stop oms -all

# Verify status
$OMS_HOME/bin/emctl status oms
```

---

## OMS Patching with OMSPatcher

OMSPatcher is the primary tool for applying Bundle Patches and one-off patches to the OMS home.

### Download and Stage the Patch

1. Log in to My Oracle Support (support.oracle.com).
2. Search for the patch by number or use the "Recommended Patches" search for OEM 13c.
3. Download the patch zip to a staging directory on the OMS host.
4. Unzip: `unzip <patch_number>.zip -d <staging_dir>`

### Apply the Patch

```bash
# Run as the OMS software owner (e.g., oracle)
cd <staging_dir>/<patch_number>

# Apply OMS patch
$OMS_HOME/OMSPatcher/omspatcher apply -analyze        # dry run first
$OMS_HOME/OMSPatcher/omspatcher apply
```

OMSPatcher handles:
- Stopping and restarting the OMS automatically (in single-OMS setups).
- Updating the OMR schema (runs SQL scripts against `SYSMAN` where required).
- Patching the AdminServer WebLogic domain.

For multi-OMS deployments, apply the patch on the primary OMS first, then on each additional OMS. The OMR schema update runs only once; subsequent OMS hosts skip it automatically.

### Restart OMS After Patching

```bash
$OMS_HOME/bin/emctl start oms
$OMS_HOME/bin/emctl status oms
```

### Verify Patch Application

```bash
$OMS_HOME/OPatch/opatch lspatches
```

Confirm the patch number appears in the output.

---

## Agent Patching — Mass Upgrade via Self Update and Agent Gold Image

### Self Update

Self Update provides a catalog of agent patches, plug-in updates, and connectors available for download directly within the OEM console. It requires outbound internet access (or a proxy) from the OMS host, or an offline bundle if the environment is air-gapped.

**Navigation:** Setup > Extensibility > Self Update

Steps:
1. In the Self Update console, select **Management Agent** and check for available updates.
2. Download the patch to the Software Library.
3. Apply to selected agent targets or all agents of a given platform.

Self Update uses the Software Library as an intermediate store. Ensure the Software Library is healthy before starting:

```bash
emcli get_software_libraries
```

### Agent Gold Image

Agent Gold Image provides a versioned, tested agent baseline that can be promoted across your estate.

**Navigation:** Setup > Manage Cloud Control > Gold Agent Images

Workflow:
1. Create a Gold Image from a reference agent that has all desired patches applied.
2. Promote the image to Active status.
3. Subscribe target agents to the Gold Image.
4. Use the Mass Agent Upgrade job to push the image to subscribed agents.

```bash
# Alternatively, use emcli for scripted mass upgrades
emcli submit_job -job_type="MASS_AGENT_UPGRADE" \
  -input_file="agents.txt" \
  -gold_agent_image="<image_name>"
```

Agents that fail to upgrade are left at their current version; the job report details the failure reason per agent.

### Manual Agent Patching (emcli)

For targeted single-agent patching outside Self Update:

```bash
emcli get_targets -targets="oracle_emd"
emcli submit_job -job_type="PATCH_AGENT" \
  -target_list="<agent_target_name>:oracle_emd" \
  -input_file="patch_params.properties"
```

---

## Plug-in Patching

Plug-in patches are released independently of the main OMS bundle patch. They must be applied after the OMS patch.

### Check Deployed Plug-in Versions

```bash
emcli list_plugins_on_server
```

### Apply a Plug-in Patch

1. Download the plug-in patch from MOS or via Self Update.
2. Stop the OMS.
3. Apply using OMSPatcher:

```bash
$OMS_HOME/OMSPatcher/omspatcher apply -patch_top <plug_in_patch_dir>
```

4. Restart the OMS.
5. Redeploy the updated plug-in to agents as needed using Self Update or emcli:

```bash
emcli deploy_plugin_on_agent \
  -agent_names="<agent1>,<agent2>" \
  -plugin="<plugin_id>:<version>"
```

Plug-in versions on agents must not exceed the version deployed on the OMS.

---

## OMR Database Patch Application

The Oracle Management Repository runs on a standard Oracle Database. Apply Database Release Updates (RUs) or one-off patches to the OMR using OPatch, following standard database patching procedures.

### Recommended Steps

1. Stop the OMS before patching the OMR database.
2. Apply the DB patch using OPatch:

```bash
# As the DB software owner
$ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -ph <patch_dir>
$ORACLE_HOME/OPatch/opatch apply <patch_dir>
```

3. If the patch includes datapatch steps, run datapatch after the database is open:

```bash
cd $ORACLE_HOME/OPatch
./datapatch -verbose
```

4. Start the OMS after the OMR is fully patched and open.

### OMR-Specific Considerations

- The OMR database must remain at a version and patch level supported by OEM 13c. Consult the OEM 13c certification matrix before applying a major DB patch.
- Applying a DB RU to the OMR while the OMS is running will cause errors; always stop the OMS first.
- Record the output of `datapatch -verbose` for verification.

---

## Self Update Feature

Self Update is the built-in mechanism for discovering and downloading patches, plug-ins, and connectors from Oracle's patch catalog.

### Online Mode

When the OMS host has internet access, Self Update connects directly to the Oracle patch catalog. Schedule periodic refreshes:

**Navigation:** Setup > Extensibility > Self Update > Check Updates

### Offline Mode

For air-gapped environments:

1. On an internet-connected host, download the Self Update catalog bundle from MOS.
2. Transfer the bundle to the OMS host.
3. Import using emcli:

```bash
emcli import_update_catalog -file="<catalog_bundle>" -omslocal
```

4. Apply updates from the imported catalog as usual.

---

## Post-Patch Verification

Run these checks after patching all components.

### OMS Health

```bash
$OMS_HOME/bin/emctl status oms
$OMS_HOME/bin/emctl status oms -details
```

### Patch Inventory

```bash
# OMS home patches
$OMS_HOME/OPatch/opatch lspatches

# OMSPatcher composite patch list
$OMS_HOME/OMSPatcher/omspatcher lspatches
```

### Repository Schema Version

```sql
-- Confirm SYSMAN schema version matches expected post-patch version
SELECT param_value FROM mgmt_versions
WHERE param_name = 'REPOS_VERSION';
```

### Agent Status

```bash
emcli get_targets -targets="oracle_emd" -noheader | grep "STATUS"
# Or check via console: Setup > Manage Cloud Control > Agents
```

### Console Accessibility

Log in to the OEM console after OMS restart and confirm:
- Home page loads without errors.
- All previously monitored targets appear with current status.
- No critical incidents related to the patching activity remain open.

### Plug-in Versions

```bash
emcli list_plugins_on_server
```

Confirm all plug-ins show expected post-patch versions.

---

## Rollback

### Rolling Back an OMS Patch

OMSPatcher supports patch rollback. Stop the OMS before rolling back.

```bash
$OMS_HOME/bin/emctl stop oms -all

# Roll back a specific patch
$OMS_HOME/OMSPatcher/omspatcher rollback -id <patch_number>

$OMS_HOME/bin/emctl start oms
```

If the OMSPatcher rollback fails due to schema changes, restore from the OMR backup taken before patching. Do not attempt a partial schema rollback manually.

### Rolling Back an OMR Database Patch

Use OPatch rollback for database-level patches:

```bash
$ORACLE_HOME/OPatch/opatch rollback -id <patch_number>
```

If datapatch steps were run, restore the OMR from RMAN backup — datapatch rollback is not always possible after the database has been open with the patched schema.

### Restoring from OMR Backup

If rollback tooling cannot restore a consistent state:

1. Shut down the OMS completely.
2. Restore the OMR database from the RMAN backup taken before patching.
3. Open the database and verify `SYSMAN` is intact.
4. Start the OMS and confirm normal operation.

### Rolling Back Agent Patches

Agents patched via Gold Image can be reverted by reassigning the agent subscription to the previous Gold Image version and rerunning the mass upgrade job. For manually patched agents:

```bash
# On the agent host
$AGENT_HOME/OPatch/opatch rollback -id <patch_number>
$AGENT_HOME/bin/emctl reload agent
```

---

## Common Errors and Resolutions

### OMSPatcher Fails: "OPatch version is not supported"

The patch README specifies a minimum OPatch version. Download the required OPatch version from MOS and replace `$OMS_HOME/OPatch` before retrying.

### Pre-requisite Check Reports Conflicts

A conflicting patch is already applied, or a required patch is missing.

- Run `$OMS_HOME/OPatch/opatch lsinventory` to see what is currently installed.
- Apply any prerequisite patches listed in the patch README before applying the target patch.
- If a conflict is reported with an already-installed one-off patch and the bundle supersedes it, the one-off can usually be rolled back first.

### "SYSMAN Schema Version Mismatch" After OMS Restart

This occurs when the OMS binary was patched but the OMR schema update did not complete. Re-run OMSPatcher apply; it will detect the schema is behind and re-execute the SQL steps.

### Agent Self Update Fails: "Software Library Not Configured"

The Software Library must be configured and accessible before Self Update can download patches.

**Navigation:** Setup > Provisioning and Patching > Software Library

Verify the storage location is writable by the OMS OS user.

### Agent Upgrade Job Fails: Credential or SSH Error

Mass agent upgrade jobs require valid preferred credentials for each target host. Verify host credentials are set:

**Navigation:** Setup > Security > Preferred Credentials > Host

Test connectivity from the OMS to the agent host before resubmitting the job.

### OMS Fails to Start After Patching: AdminServer Wallet Error

The WebLogic AdminServer wallet may need to be resynchronized after certain patches.

```bash
$OMS_HOME/bin/emctl secure oms -sysman_pwd <password>
$OMS_HOME/bin/emctl start oms
```

### "Inventory Lock Exists" Error in OPatch

A previous OPatch run may have left a lock file.

```bash
# Verify no OPatch process is running, then remove the lock
rm $ORACLE_HOME/.patch_storage/temp_lock
```

Only remove the lock file after confirming no patch operation is in progress.

### datapatch Reports "Patch Already Applied"

This is informational. No action required if all patches listed are in `APPLIED` state:

```sql
SELECT patch_id, status, action_time FROM dba_registry_sqlpatch
ORDER BY action_time DESC;
```

---

## Oracle Version Notes

- OEM 13.3 and earlier use Bundle Patch naming; OEM 13.5 introduced Release Update naming aligned with the Oracle Database patching model.
- OMSPatcher replaced the older `opatchauto` workflow for OMS patching starting with OEM 13.2.
- Agent Gold Image was introduced in OEM 13.2.
- Self Update offline catalog import via `emcli import_update_catalog` is available in OEM 13.2 and later.

---

## Sources

- Oracle Enterprise Manager Cloud Control 13c: Patching Oracle Management Service — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emupg/patching-oracle-management-service.html
- Oracle Enterprise Manager Cloud Control 13c: Patching Oracle Management Agents — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emupg/patching-oracle-management-agent.html
- Oracle Enterprise Manager Cloud Control 13c: Self Update — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emadm/self-update.html
- Oracle Enterprise Manager Cloud Control 13c: Agent Gold Images — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emadm/managing-gold-agent-images.html
- Oracle Enterprise Manager Cloud Control 13c: Upgrading and Updating Oracle Management Agent Using Mass Agent Upgrade — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emupg/upgrading-oracle-management-agent.html
- Oracle OPatch User's Guide for Oracle Database — https://docs.oracle.com/en/database/oracle/oracle-database/19/opunk/index.html
- MOS Note: Enterprise Manager Cloud Control: Recommended Patches — search My Oracle Support for Doc ID 1357376.1
- MOS Note: OMSPatcher User Guide — search My Oracle Support for Doc ID 1982664.1
