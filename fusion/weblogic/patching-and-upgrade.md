# WebLogic Server Patching and Upgrade

## Overview

Use this skill to plan and execute WebLogic Server patching (Bundle Patches, PSUs) and version upgrades (12.2.1.4 to 14.1.1, and subsequent lines). Load `references/weblogic-version-matrix.md` to confirm version labels before committing to a patch or upgrade path.

This skill covers:
- OPatch vs Smart Update (BSU) — when each applies
- Applying Bundle Patches and PSUs for 12.2.1.4 and 14.1.1
- Rolling patch for clustered domains
- In-place vs out-of-place upgrade
- Upgrade path from 12.2.1.4 to 14.1.1
- Pre-patch backup checklist
- Post-patch Admin Console and WLST validation

---

## Patching Tool Selection: OPatch vs Smart Update (BSU)

### Smart Update (BSU)

BSU (`bsu.sh` / `bsu.cmd`) was the patch tool for WebLogic 10.3.x (11g) and 12.1.x. It operated on the MW_HOME Patch Catalog and applied Maintenance Packs and individual patches as ZIP bundles.

BSU is **not available** for WebLogic 12.2.1.x, 14.1.x, or 15.1.x. Do not attempt to use BSU on any release from 12.2.1.0 onward.

### OPatch

OPatch (`opatch`) replaced BSU starting with WebLogic 12.2.1.0 and is required for all patching on 12.2.1.x, 14.1.x, and 15.1.x. OPatch ships inside the Oracle Home (`<ORACLE_HOME>/OPatch/opatch`) and must be updated before applying new patches when Oracle My Support indicates a minimum OPatch version requirement.

Check the installed OPatch version before any patch session:

```bash
<ORACLE_HOME>/OPatch/opatch version
```

Upgrade OPatch by replacing the `OPatch` directory with the downloaded version from My Oracle Support patch 6880880. Do not overwrite other Oracle Home contents.

---

## Pre-Patch Backup Checklist

Complete this checklist before applying any patch or starting an upgrade. Skipping these steps can make rollback impossible.

1. **Stop all servers in the domain** — Admin Server, Managed Servers, and Node Manager.
2. **Back up the Oracle Home** (`$ORACLE_HOME`) with a full file-system copy or OS snapshot. OPatch applies binary changes to JARs inside the Oracle Home; without this backup, rollback requires re-running the patch with `opatch rollback`.
3. **Back up the domain directory** (`$DOMAIN_HOME`) including `config/`, `applications/`, `servers/`, and `security/`.
4. **Back up the OPatch inventory** (`$ORACLE_HOME/inventory/` and the central inventory pointer at `/etc/oraInst.loc` or equivalent on Windows).
5. **Record the current patch level**: run `opatch lsinventory` and save the output.
6. **Verify disk space**: OPatch stages extracted patch content during apply. Allow at least 2x the patch archive size as free space under `$ORACLE_HOME`.
7. **Confirm JDK version compatibility** for the target patch: PSU and BP release notes on My Oracle Support list minimum JDK requirements.
8. **Test rollback procedure in a non-production environment** before applying to production.

---

## Applying a Bundle Patch or PSU — WebLogic 12.2.1.4

WebLogic 12.2.1.4 patches are distributed as ZIP archives available from My Oracle Support. The standard process uses OPatch.

### Step-by-step: applying a Bundle Patch on 12.2.1.4

```bash
# 1. Confirm OPatch minimum version (see patch README for requirement)
$ORACLE_HOME/OPatch/opatch version

# 2. Extract the patch archive (example patch number 12345678)
unzip p12345678_122140_Generic.zip -d /tmp/patches/

# 3. Stop all servers in the domain
# (use Admin Console, WLST, or Node Manager stop scripts)

# 4. Apply the patch
cd /tmp/patches/12345678
$ORACLE_HOME/OPatch/opatch apply

# 5. Confirm successful application
$ORACLE_HOME/OPatch/opatch lsinventory
```

OPatch will prompt for confirmation before modifying the Oracle Home. Review the list of files to be replaced and confirm. If the patch includes a prerequisite check (`prereq CheckConflictAgainstOHWithDetail`), run it before `apply`:

```bash
$ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -ph /tmp/patches/12345678
```

### Confirming the PSU level

After apply, verify the installed patch and its version in the inventory:

```bash
$ORACLE_HOME/OPatch/opatch lsinventory -detail | grep -i "Bundle\|PSU\|Patch"
```

---

## Applying a Bundle Patch or PSU — WebLogic 14.1.1

The process for 14.1.1 is identical to 12.2.1.4: OPatch is used, patches are downloaded from My Oracle Support, and servers must be down during apply. The key differences are:

- The Oracle Home path reflects the 14.1.1 installation directory, not the 12.2.1.4 path.
- JDK requirements differ: 14.1.1 supports JDK 8 and JDK 11 (verify the patch README for the specific BP or PSU applied).
- OPatch minimum version requirements may be higher for 14.1.1 patch trains; always check the patch README.

---

## Rolling Patch for Clustered Domains

Oracle does not support applying OPatch while servers are running (online patching is not available for WebLogic binary patches). For clustered domains, use a rolling restart approach after patching:

1. Patch the Oracle Home on each machine in sequence (all machines share the same Oracle Home or each has its own copy).
2. On each machine: stop the Managed Servers on that machine, apply the patch to that machine's Oracle Home, then restart those Managed Servers before moving to the next machine.
3. Patch and restart the Admin Server last.

**Important**: the Admin Server and all Managed Servers must ultimately run from Oracle Homes at the same patch level. Running mixed patch levels within a cluster during the rolling window is a transitional state; complete the rollout before returning to steady state.

For clusters using a shared Oracle Home (NFS-mounted or copied), all servers on that shared home go down together; a true rolling patch requires separate Oracle Home installations per host or the use of out-of-place patching.

---

## In-Place vs Out-of-Place Upgrade

### In-place upgrade

An in-place upgrade uses the existing Oracle Home location and runs the Fusion Middleware Upgrade Assistant (UA) to update configuration artifacts and the domain to the new release schema. The Oracle Home is replaced or updated in position.

Advantages: simpler if infrastructure does not allow new Oracle Home paths.
Risks: if the upgrade fails mid-run, the original Oracle Home may be partially modified; rollback depends on the filesystem backup taken in the pre-patch checklist.

### Out-of-place upgrade

An out-of-place upgrade installs the target WebLogic release into a new Oracle Home directory alongside the existing installation. The Upgrade Assistant then reconfigures the domain to reference the new Oracle Home.

Advantages: the original Oracle Home is untouched during the upgrade, allowing rollback without a restore. This is the Oracle-recommended approach for major version upgrades.

Oracle's Fusion Middleware documentation explicitly recommends out-of-place upgrade for major-version migrations (e.g., 12.2.1.4 to 14.1.1).

---

## Upgrade Path: WebLogic 12.2.1.4 to 14.1.1

Oracle supports a direct upgrade from 12.2.1.4 to 14.1.1. A phased intermediate hop through 12.2.1.3 or earlier 12.2.1.x releases is not required if the source is already at 12.2.1.4.

### Pre-upgrade prerequisites

- Apply the latest available Bundle Patch to the 12.2.1.4 Oracle Home before starting the upgrade. Upgrading from an unpatched 12.2.1.4 base may encounter known issues that a current BP resolves.
- Verify JDK compatibility: WebLogic 14.1.1 requires JDK 8 (minimum update per the certification matrix) or JDK 11. Confirm the JDK version in use on the source environment and whether it meets 14.1.1 requirements.
- Run the Upgrade Readiness Check with the Upgrade Assistant before modifying any installation:

```bash
# Example: Readiness check only (no changes applied)
<14.1.1_ORACLE_HOME>/oracle_common/upgrade/bin/ua -readiness
```

- Review the Readiness Report output and resolve all warnings and errors before proceeding.

### Step-by-step: out-of-place upgrade from 12.2.1.4 to 14.1.1

```
Step 1: Install WebLogic 14.1.1 into a new Oracle Home (do not select
        "Upgrade" during the installer; install fresh).

Step 2: Confirm the 12.2.1.4 servers are stopped.

Step 3: Back up the domain directory (see Pre-Patch Backup Checklist).

Step 4: Run the Upgrade Assistant from the 14.1.1 Oracle Home:
        <14.1.1_ORACLE_HOME>/oracle_common/upgrade/bin/ua

Step 5: In the Upgrade Assistant UI or silent mode:
        - Select "All Configurations Used By a Domain".
        - Point to the existing 12.2.1.4 domain home.
        - Complete the schema and config upgrade steps.

Step 6: Run the Reconfiguration Wizard to update the domain to reference
        the new Oracle Home:
        <14.1.1_ORACLE_HOME>/oracle_common/bin/reconfig.sh -log=<log_path> \
          -log_priority=ALL

Step 7: Start the Admin Server from the 14.1.1 Oracle Home and validate
        (see Post-Patch Validation below).

Step 8: Start Managed Servers and verify cluster health.
```

The Reconfiguration Wizard (`reconfig.sh` / `reconfig.cmd`) updates `config.xml` and server start scripts to reference the new Oracle Home; it must be run after the Upgrade Assistant when changing Oracle Home paths.

---

## Post-Patch Validation

### Admin Console validation

1. Start the Admin Server.
2. Log in to the Admin Console at `http://<host>:7001/console`.
3. Navigate to **Environment > Servers** and confirm all server states are RUNNING.
4. Navigate to **Deployments** and confirm all applications are in the ACTIVE state.
5. Navigate to **Services > Data Sources** and check the **Monitoring** tab for each DataSource; verify the connection test passes.
6. Navigate to **Domain > Configuration > General** and confirm the domain version reflects the target release.

### WLST validation

Connect to the domain with WLST and confirm server and deployment state:

```python
# Connect to the Admin Server
connect('weblogic', '<password>', 't3://localhost:7001')

# Check all server states
domainRuntime()
serverRuntimes = cmo.getServerRuntimes()
for sr in serverRuntimes:
    print(sr.getName(), sr.getState())

# Check deployed applications
appRuntimes = cmo.getApplicationRuntimes()
for ar in appRuntimes:
    print(ar.getName(), ar.getHealthState())

disconnect()
```

### Confirming the patch level with WLST

```python
connect('weblogic', '<password>', 't3://localhost:7001')
serverRuntime()
print(cmo.getWeblogicVersion())
disconnect()
```

Compare the printed version string against the expected patch level recorded in the pre-patch checklist.

### OPatch inventory confirmation

```bash
<ORACLE_HOME>/OPatch/opatch lsinventory
```

Confirm the target Bundle Patch or PSU appears in the installed patches list.

---

## Best Practices and Common Mistakes

**Always stop all servers before running OPatch apply.** OPatch does not enforce a server-down requirement by default; applying with servers running risks inconsistent class loading and undefined behavior at runtime.

**Do not apply patches across different Oracle Home versions.** A patch extracted for 12.2.1.4 cannot be applied to a 14.1.1 Oracle Home. Verify the patch platform and version from the My Oracle Support README before running apply.

**Upgrade OPatch before applying patches that require a newer OPatch version.** Attempting to apply a patch with an incompatible OPatch version produces an error during the prerequisite check. Update OPatch from MOS patch 6880880 first.

**Keep a separate Oracle Home backup per host.** In clustered deployments where each host has its own Oracle Home copy, back up each host's Oracle Home independently before starting the rolling patch.

**Run the Readiness Check before any upgrade.** The Upgrade Assistant's readiness-only mode (`-readiness` flag) does not modify any configuration; it identifies schema and configuration incompatibilities that would cause a live upgrade to fail.

**Do not skip the Reconfiguration Wizard after an out-of-place upgrade.** Without reconfig, `config.xml` still references the old Oracle Home path; the domain will fail to start from the new Oracle Home.

**Test rollback in a non-production environment.** OPatch rollback (`opatch rollback -id <patch_id>`) restores the original JARs from the staging area. Verify the rollback procedure works before applying to production.

**Mixed patch levels in a cluster are a transient state only.** Complete the rolling patch within a maintenance window; do not leave the cluster in a mixed-patch state across normal operation.

---

## Oracle Version Notes (12c vs 14c vs 15c Patching)

| Version | Patch tool | Notes |
|---|---|---|
| 11g (10.3.6) | Smart Update (BSU) | BSU-only; OPatch does not apply. Final BSU patches via My Oracle Support. |
| 12.1.3 | OPatch | OPatch replaced BSU at 12.2.1.0; 12.1.3 still uses BSU for WebLogic-specific patches. Verify the patch README. |
| 12.2.1.4 | OPatch | Last 12c patch-set line. Active Bundle Patch train on My Oracle Support. Direct upgrade to 14.1.1 supported. |
| 14.1.1.0.0 | OPatch | Standalone release line. Bundle Patches delivered quarterly (Patch Set Updates). JDK 8 and JDK 11 supported. |
| 14.1.2.0.0 | OPatch | Separate release from 14.1.1; do not conflate patch IDs between these two lines. |
| 15.1.1.0.0 | OPatch | Latest major line (as of May 2026). JDK 17 and JDK 21 supported; JDK 8 support dropped. |

Key differences to call out:

- **12.2.1.4 to 14.1.1**: direct upgrade supported via Upgrade Assistant. This is the most common in-production upgrade path.
- **14.1.1 to 14.1.2**: out-of-place recommended; these are distinct release lines with separate patch trains.
- **15.1.1 JDK requirement**: CMS GC is removed in JDK 14+; any GC flags using `-XX:+UseConcMarkSweepGC` must be removed before running 15c on JDK 17 or 21. See `troubleshooting.md` for JVM flag migration notes.
- **PSU naming**: Oracle uses "Bundle Patch" (BP) as the primary cumulative patch vehicle for WebLogic 12.2.1.4 and later; older terminology "PSU" (Patch Set Update) is used in some MOS notes interchangeably but the delivery mechanism is the same OPatch-applied cumulative archive.

---

## Sources

- WebLogic 12.2.1.4 patching (OPatch): https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/12.2.1.4/index.html
- WebLogic 14.1.1 patching and upgrade: https://docs.oracle.com/en/middleware/standalone/weblogic-server/14.1.1.0/index.html
- Fusion Middleware Upgrade Guide (12.2.1.4 to 14.1.1): https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/asmas/upgrading-oracle-fusion-middleware-12c.html
- Fusion Middleware Upgrade Guide for WebLogic 14.1.1: https://docs.oracle.com/en/middleware/standalone/weblogic-server/14.1.1.0/notes/whatsnew.html
- OPatch documentation (Oracle Universal Installer and OPatch): https://docs.oracle.com/en/database/oracle/oracle-database/19/opiad/index.html
- Upgrade Assistant reference (Fusion Middleware): https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/asmas/preparing-upgrade-oracle-fusion-middleware.html
- WebLogic 15.1.1 documentation: https://docs.oracle.com/en/middleware/standalone/weblogic-server/15.1.1/wlsig/index.html
- WebLogic 14.1.2 documentation: https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/14.1.2/index.html
- My Oracle Support (patch downloads, patch READMEs, OPatch 6880880): https://support.oracle.com
- `references/weblogic-version-matrix.md`
