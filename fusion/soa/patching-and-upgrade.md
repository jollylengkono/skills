# Oracle SOA Suite Patching and Upgrade

## Overview

Use this skill to apply Bundle Patches to an Oracle SOA Suite installation, upgrade the SOAINFRA schema after patching, execute the supported 11g-to-12c upgrade path, and validate the environment post-patch. Coverage spans SOA Suite 11g (11.1.1.x) through 12c (12.2.1.x) and the current 14.1.2 line.

---

## Core Concepts

### Patch Delivery for SOA Suite

Oracle SOA Suite patches are delivered as:

| Patch type | Description |
|---|---|
| Bundle Patch (BP) | Cumulative set of bug fixes for SOA Suite, released quarterly. Applied to the SOA Oracle Home using OPatch. |
| Patch Set Update (PSU) | Platform-wide cumulative patch used in the 11g era; superseded by Bundle Patches in 12c. |
| One-off patch | Single fix for a specific bug; applied on top of a Bundle Patch. |
| Merge patch | Merged set of one-offs consolidated for easier application. |

Patches are downloaded from My Oracle Support (MOS) at support.oracle.com. Each patch ships with a README that lists prerequisite patch levels, minimum OPatch version, and required pre/post steps. **Always read the README before applying.**

### OPatch

OPatch is the standard Oracle binary patching utility. For SOA Suite, OPatch operates against the Fusion Middleware Oracle Home (the directory containing the SOA product binaries, not the domain directory). It does not modify running processes; the server must be stopped before applying patches that touch binary files.

### RCU and the SOAINFRA Schema

The Repository Creation Utility (RCU) creates the SOAINFRA, MDS, and related schemas in the database. After applying a Bundle Patch that includes schema changes, the Upgrade Assistant (UA) or an inline SQL script (documented in the patch README) must be run to bring the schema in line with the new binary level. Skipping the schema upgrade step after a patch that requires it will cause startup failures.

### Upgrade Assistant

The Upgrade Assistant (UA) is the standard tool for upgrading Fusion Middleware domain configurations and database schemas from one release line to another. It is used both for major release upgrades (11g to 12c) and for schema-level updates after some Bundle Patches.

---

## Pre-Patch Checklist

Complete every item on this checklist before applying any Bundle Patch or starting a major upgrade.

### 1. Verify OPatch Version

Each patch README specifies a minimum OPatch version. Check the current version and update if required.

```bash
# Check installed OPatch version
$ORACLE_HOME/OPatch/opatch version

# Update OPatch by unzipping the new OPatch zip over the existing directory
# (download latest from MOS; patch 6880880 is the standard OPatch patch number)
```

### 2. Quiesce SOA Composites

Before stopping the server, quiesce all deployed SOA composites to allow in-flight instances to complete or to prevent new instances from starting.

Via Enterprise Manager Fusion Middleware Control:
- Navigate to SOA > soa-infra > Deployed Composites.
- Select all composites and choose **Quiesce**.
- Wait for the quiesce state to be confirmed before proceeding.

Via WLST (offline bulk quiesce):

```python
# Connect to AdminServer
connect('<admin_user>', '<admin_password>', 't3://<host>:<admin_port>')
# Navigate to SOA runtime MBean
domainRuntime()
cd('SoaInfrastructure')
# Individual composite quiesce via MBean — see SOA Admin Guide for MBean path
```

Allow in-flight long-running BPEL instances to reach a safe dehydration point before stopping. Check for active instances:

```sql
-- Run in SOAINFRA schema
SELECT COUNT(*) FROM CUBE_INSTANCE WHERE STATE NOT IN (4, 5, 9, 32);
-- States 4=closed/completed, 5=closed/faulted, 9=stale, 32=suspended
-- Wait until count reaches zero or acceptable threshold before proceeding
```

### 3. Back Up the SOAINFRA and MDS Schemas

Take a full backup of all SOA-related schemas before patching.

```bash
# Export with Data Pump — run as DBA or schema owner
expdp system/password@<db_service> \
  schemas=<PREFIX>_SOAINFRA,<PREFIX>_MDS,<PREFIX>_STB,<PREFIX>_WLS \
  directory=DATA_PUMP_DIR \
  dumpfile=soa_schemas_prepatch_%U.dmp \
  logfile=soa_schemas_prepatch.log
```

Also take an RMAN backup of the database hosting the SOA schemas, or a consistent snapshot if running on a supported platform.

### 4. Back Up the Oracle Home and Domain

```bash
# Copy the SOA Oracle Home (stop servers first)
tar -czf /backup/soa_oracle_home_$(date +%Y%m%d).tar.gz $ORACLE_HOME

# Copy the WebLogic domain directory
tar -czf /backup/soa_domain_$(date +%Y%m%d).tar.gz $DOMAIN_HOME
```

### 5. Check for Conflicting Patches

```bash
$ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -ph <patch_staging_dir>
```

Review the output before proceeding. Conflicts must be resolved — typically by rolling back the conflicting one-off or confirming the bundle supersedes it — before applying the new patch.

### 6. Verify Disk Space

Ensure the Oracle Home mount point has sufficient free space (check the patch README for the specific patch's size requirements). OPatch retains the original files in a patch storage area for rollback.

```bash
df -h $ORACLE_HOME
```

---

## Applying a Bundle Patch with OPatch

### 1. Stop All Servers in the SOA Domain

Stop Managed Servers before stopping the AdminServer. The Node Manager can remain running.

```bash
# Stop SOA Managed Server (use WLST or WebLogic Admin Console)
# Stop AdminServer last
$DOMAIN_HOME/bin/stopManagedWebLogic.sh <managed_server_name> t3://<host>:<admin_port>
$DOMAIN_HOME/bin/stopWebLogic.sh
```

### 2. Stage and Apply the Patch

```bash
# Unzip the downloaded patch
unzip <patch_number>.zip -d /tmp/soa_patch_staging

# Navigate to the patch directory
cd /tmp/soa_patch_staging/<patch_number>

# Dry run: analyze without applying
$ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -ph .

# Apply the patch
$ORACLE_HOME/OPatch/opatch apply
```

OPatch prompts for confirmation before modifying the Oracle Home. Review the prompt output and enter `y` to proceed.

### 3. Verify the Patch Was Applied

```bash
$ORACLE_HOME/OPatch/opatch lspatches
```

Confirm the new patch number appears in the inventory listing.

### 4. Upgrade the SOAINFRA Schema (if required by the patch)

Check the patch README for the heading **"Post-Installation Steps"** or **"Schema Upgrade Steps"**. If schema changes are included:

**Option A — Upgrade Assistant (recommended for full schema upgrade):**

```bash
$ORACLE_HOME/oracle_common/upgrade/bin/ua
```

Launch the Upgrade Assistant, select **Upgrade Domain Schemas**, point it at the database connection, and follow the wizard. The UA validates the schema version and applies the required SQL.

**Option B — Manual SQL script (only when the README explicitly directs this):**

```bash
# Run the SQL script shipped in the patch under the /sql directory
sqlplus <soainfra_schema>/<password>@<db_service> @<patch_dir>/sql/upgrade.sql
```

Never run SQL scripts from a patch unless the patch README explicitly lists them as the required post-install step.

### 5. Start the Servers

```bash
$DOMAIN_HOME/bin/startWebLogic.sh
$DOMAIN_HOME/bin/startManagedWebLogic.sh <managed_server_name> t3://<host>:<admin_port>
```

---

## Rolling Patch for Clustered SOA Domains

For clustered SOA deployments where zero-downtime patching is required, Oracle supports a rolling patch approach for patches that do not include schema changes.

**Important:** Rolling patches are only supported when the patch README explicitly states compatibility with rolling application. Patches that include SOAINFRA schema changes require all Managed Servers to be down during the schema upgrade step and cannot be applied in a rolling fashion across a cluster that continues to process instances.

### Procedure

1. Confirm the patch README states the patch is rolling-patch compatible.
2. Apply OPatch to the Oracle Home on each cluster node sequentially. The Oracle Home is typically shared storage in clustered environments; if it is local per node, apply to each node separately.
3. If Oracle Home is on shared storage: stop one Managed Server at a time, apply the patch to the shared home, start that server, and proceed to the next node.

```bash
# On the node being patched:
# Stop the Managed Server for this node only
# Apply OPatch to the Oracle Home
$ORACLE_HOME/OPatch/opatch apply

# Restart this Managed Server before moving to the next node
```

4. After all nodes are patched, perform any required post-patch steps (schema upgrade, configuration refresh).

For clustered environments on shared storage where the patch is NOT rolling-compatible:
- Stop all Managed Servers across the cluster.
- Apply OPatch once to the shared Oracle Home.
- Run schema upgrade steps.
- Start all Managed Servers.

---

## Upgrading from SOA Suite 11g to 12c

The supported upgrade path from SOA 11g (11.1.1.7 or later) to SOA 12c (12.2.1.x) is:

```
SOA 11.1.1.7 → SOA 12.2.1.x (using Upgrade Assistant)
```

SOA releases earlier than 11.1.1.7 require a stepping-stone upgrade to 11.1.1.7 first. There is no direct in-place binary upgrade; the process installs a new 12c Oracle Home and migrates the domain and schemas.

### High-Level Upgrade Sequence

1. Confirm source is at a supported upgrade baseline (11.1.1.7 minimum). Patch to 11.1.1.7 if needed.
2. Install Oracle SOA Suite 12c into a new Oracle Home (do not overwrite the 11g home).
3. Run the Reconfiguration Wizard to update the domain to use the 12c Oracle Home.
4. Run the Upgrade Assistant to upgrade all relevant schemas (MDS, SOAINFRA, STB, WLS).
5. Apply current Bundle Patch to the new 12c Oracle Home.
6. Start and validate the upgraded domain.

### Step-by-Step

#### Step 1: Prepare the 11g Environment

```bash
# On 11g — apply the latest 11.1.1.7.x Bundle Patch before upgrading
# Confirm the SOAINFRA and MDS schema versions
SELECT comp_name, version, status FROM schema_version_registry
WHERE comp_name IN ('AS COMMON SCHEMAS', 'SOAINFRA', 'MDS');
```

Back up the 11g domain, Oracle Home, and database schemas as described in the pre-patch checklist.

#### Step 2: Install SOA 12c Oracle Home

Run the SOA 12c installer into a new, separate Oracle Home:

```bash
./fmw_12.2.1.x.0_soa_Disk1_1of1.bin
```

Select a different `ORACLE_HOME` path than the existing 11g installation. Do not modify the 11g home during this step.

#### Step 3: Run the Reconfiguration Wizard

The Reconfiguration Wizard updates the existing WebLogic domain to use the new 12c Oracle Home.

```bash
$NEW_ORACLE_HOME/oracle_common/common/bin/reconfig.sh -log=<log_dir>/reconfig.log
```

Point the wizard at the existing 11g domain directory. The wizard updates domain configuration files in place.

#### Step 4: Upgrade Schemas with the Upgrade Assistant

Launch the Upgrade Assistant from the new 12c Oracle Home:

```bash
$NEW_ORACLE_HOME/oracle_common/upgrade/bin/ua
```

Select **Upgrade Domain Schemas**. The Upgrade Assistant will detect all schemas associated with the domain and upgrade them in the required order: WLS metadata, MDS, SOAINFRA, B2B (if present), BPM (if present).

Provide the database connection details when prompted. Review the pre-upgrade validation report before allowing schema changes.

#### Step 5: Apply Current Bundle Patch to the 12c Oracle Home

After the Upgrade Assistant completes, apply the current SOA 12c Bundle Patch to the new Oracle Home before starting servers:

```bash
$NEW_ORACLE_HOME/OPatch/opatch apply
```

#### Step 6: Start the Domain and Validate

```bash
$DOMAIN_HOME/bin/startWebLogic.sh
$DOMAIN_HOME/bin/startManagedWebLogic.sh <soa_server_name> t3://<host>:<admin_port>
```

---

## Post-Patch Validation

Run these checks after applying any Bundle Patch or completing a major upgrade.

### OPatch Inventory

```bash
$ORACLE_HOME/OPatch/opatch lspatches
```

Confirm the expected patch number is listed.

### Schema Version Registry

```sql
-- Run as DBA or schema owner
SELECT comp_name, version, status, upgraded
FROM schema_version_registry
WHERE comp_name IN ('SOAINFRA', 'MDS', 'AS COMMON SCHEMAS', 'STB', 'WLS')
ORDER BY comp_name;
```

All components should show `VALID` status. If any show `INVALID` or `UPGRADING`, the schema upgrade did not complete. Re-run the Upgrade Assistant or consult the patch README for recovery steps.

### SOA Infrastructure Health Check via EM

1. Log in to Enterprise Manager Fusion Middleware Control.
2. Navigate to SOA > soa-infra.
3. Confirm the soa-infra application shows **Running** status.
4. Check Dashboard for any reported faults or failed composites.
5. Deploy a test composite (or re-enable a previously quiesced composite) and submit a test instance to verify end-to-end functionality.

### Server Log Review

Review the SOA Managed Server log for errors after startup:

```
$DOMAIN_HOME/servers/<soa_managed_server>/logs/<soa_managed_server>.log
```

Look for startup errors related to schema version mismatches (`SOAINFRA schema version mismatch`), MDS initialization failures, or JCA adapter startup errors.

### Verify Composite Deployment and Resume

```bash
# Via WLST — list deployed composites
wls:/offline> connect('<admin_user>','<admin_password>','t3://<host>:<admin_port>')
wls:/<domain>> domainRuntime()
# Navigate SOA runtime MBean to list and activate composites
```

Via EM: navigate to SOA > soa-infra > Deployed Composites, confirm all expected composites are present and in **Active** state. Resume any composites that were quiesced before patching.

---

## Best Practices and Common Mistakes

### Best Practices

- Always read the full patch README before starting. OPatch failures caused by skipped prerequisites are common and time-consuming to recover from.
- Keep OPatch current. Each Bundle Patch specifies a minimum OPatch version; running an older OPatch may silently skip validations.
- Quiesce composites before stopping servers. Abruptly stopping servers with active BPEL instances causes instance recovery on restart, which adds load and can reveal dehydration inconsistencies.
- Take a schema export before every patching session, even when a database-level backup exists. Schema-level restores are faster for targeted recovery.
- Apply the schema upgrade immediately after the binary patch, in the same maintenance window. Running a patched 12c binary against an unpatched schema is an unsupported mixed-version state.
- Test in a non-production environment first. Bundle Patches occasionally include behavior changes that affect adapter configurations or composite routing.

### Common Mistakes

- **Patching the binary but not upgrading the schema.** The server may start but fail when a composite processes an instance that touches upgraded code paths. Always check `schema_version_registry` after patching.
- **Applying a patch while servers are running.** OPatch may succeed (it locks files) but the running server continues using the old code until restart. For patches that modify shared libraries, this can produce inconsistent behavior.
- **Not verifying free space before OPatch.** OPatch stores a backup of replaced files in `$ORACLE_HOME/.patch_storage`. On large Bundle Patches, this requires gigabytes of space. A mid-patch failure due to disk exhaustion leaves the Oracle Home in a partially patched state.
- **Skipping the rollback test in non-production.** OPatch rollback is available (`opatch rollback -id <patch_number>`) but should be verified in a test environment before relying on it in production.
- **Running UA from the wrong Oracle Home.** During an 11g-to-12c upgrade, the Upgrade Assistant must be run from the new 12c Oracle Home, not from the 11g home.
- **Missing a schema that needs upgrading.** If the SOA domain uses B2B or BPM, those schemas must be upgraded by UA in the same session as SOAINFRA. Skipping a schema because it appears optional often causes startup failures in those product components.

### OPatch Rollback

If a Bundle Patch must be rolled back:

```bash
# Stop all servers first
$ORACLE_HOME/OPatch/opatch rollback -id <patch_number>

# Verify rollback
$ORACLE_HOME/OPatch/opatch lspatches
```

If schema changes were applied, the database must be restored from backup — OPatch rollback reverts only the Oracle Home binaries, not the database schema.

---

## Oracle Version Notes (SOA 11g vs SOA 12c Patching)

This section covers differences in patching approach between SOA 11g and SOA 12c/14c.

### Patching Tooling

| Aspect | SOA 11g | SOA 12c / 14c |
|---|---|---|
| Primary patch tool | OPatch | OPatch (same tool, newer versions) |
| Patch format | Bundle Patch or PSU | Bundle Patch |
| Schema upgrade tool | RCU re-run or manual SQL (patch-specific) | Upgrade Assistant (preferred) or inline SQL |
| OPatch minimum version | Specified per patch README; typically 11.1.0.x | Specified per patch README; typically 13.9.x or later |

### Schema Upgrade Behavior

In SOA 11g, bundle patches often included SQL scripts that were applied manually or via a custom `ant` target shipped in the patch. In SOA 12c, Oracle standardized on the Upgrade Assistant for schema-level changes, which provides pre-upgrade validation reports and rollback awareness.

If working with a legacy SOA 11g Bundle Patch, follow the patch README carefully — the 11g README may reference `ant` targets or specific WLST scripts rather than UA.

### In-Place Upgrade Support

There is no supported in-place binary upgrade path that overwrites an existing 11g Oracle Home with 12c binaries. The 12c installer must target a new Oracle Home; the domain is then reconfigured and schemas are upgraded while the domain directory remains in place.

For SOA 12c minor version updates (e.g., 12.2.1.3 to 12.2.1.4), patching via Bundle Patch is the supported path; a full reinstall is not required.

### Database Compatibility

| SOA Release | Minimum Supported DB | Recommended DB Baseline |
|---|---|---|
| SOA 11.1.1.7 | Oracle Database 11g R2 | 11.2.0.4 or 12c |
| SOA 12.2.1.x | Oracle Database 12c | 19c (long-term support) |
| SOA 14.1.2 | Oracle Database 19c or later | 19c |

Always consult the Oracle Fusion Middleware certification matrix for the exact database version supported for your specific SOA release and patch level before upgrading either component independently.

---

## Sources

- Oracle SOA Suite 12c: Patching Oracle Fusion Middleware — https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/opupg/patching-oracle-fusion-middleware.html
- Oracle Fusion Middleware Upgrade Guide for Oracle SOA Suite — https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/opupg/upgrading-oracle-soa-suite.html
- Oracle Fusion Middleware Infrastructure and Application Upgrade — https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/opupg/index.html
- Oracle OPatch User's Guide for Oracle Fusion Middleware — https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/opatc/index.html
- Oracle Fusion Middleware Repository Creation Utility User's Guide — https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/rcuug/index.html
- Oracle SOA Suite documentation hub — https://docs.oracle.com/en/middleware/soa-suite/
- Oracle Fusion Middleware Certification Matrix — https://www.oracle.com/middleware/technologies/fusion-certification.html
- Oracle SOA Suite 11g documentation — https://www.oracle.com/middleware/technologies/soa-suite-11gdocumentation.html
