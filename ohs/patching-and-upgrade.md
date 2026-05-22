# Oracle HTTP Server Patching and Upgrade

## Overview

Oracle HTTP Server (OHS) 12.2.1.x is distributed as part of Oracle Fusion Middleware and patched using OPatch (for one-off patches) and the Oracle Universal Installer (OUI) framework. Understanding the patch taxonomy, correct tooling, and sequencing requirements prevents service outages and inventory corruption.

This skill covers:

- Patch types and the OPatch workflow for OHS 12.2.1.4
- Bundle Patch (BP) and Patch Set Update (PSU) application
- Pre-patch backup and validation
- Rolling patch procedures for HA deployments
- Upgrade paths from earlier OHS 12.2.1.x releases
- Post-patch validation

Baseline version: **OHS 12.2.1.4**. Behavior differences for earlier 12.2.1.x releases are noted where relevant.

---

## Patch Types

| Patch Type | Description |
|---|---|
| Bundle Patch (BP) | Cumulative quarterly patch that bundles security fixes and high-priority bug fixes for OHS and the Oracle Web Tier. Supersedes previous BPs for the same release. |
| Patch Set Update (PSU) | Security-focused cumulative patch; in the 12.2.1.x lifecycle the Bundle Patch serves as the primary cumulative vehicle. |
| One-off patch | Single-bug fix applied on top of an installed BP; available from My Oracle Support (MOS). |
| Interim patch | Synonym for one-off patch; the terms are used interchangeably in MOS and OPatch documentation. |

Oracle stopped releasing new Patch Sets (full installers that change the third or fourth digit of the version) in the 12.2 line. Moving between 12.2.1.x minor releases (e.g., 12.2.1.3 to 12.2.1.4) requires an out-of-place upgrade, not an in-place patch.

---

## Pre-Patch Requirements

### Verify OPatch Version

OHS 12.2.1.4 requires OPatch 13.9.x or later. The minimum OPatch version is stated in the patch README (`README.txt` or `README.html` inside the patch zip). Always update OPatch before applying any patch.

```bash
# Check the current OPatch version
$ORACLE_HOME/OPatch/opatch version
```

To update OPatch, download the latest 13.x OPatch archive (patch 6880880, platform-specific) from MOS and replace the existing `$ORACLE_HOME/OPatch` directory with the new one:

```bash
# Run as the OHS software owner
mv $ORACLE_HOME/OPatch $ORACLE_HOME/OPatch.bak
unzip p6880880_<version>_<platform>.zip -d $ORACLE_HOME
```

Verify after replacement:

```bash
$ORACLE_HOME/OPatch/opatch version
```

### Review the Patch README

Every patch zip contains a README. Read it before proceeding. It specifies:

- Required OPatch version
- Any prerequisite patches that must already be installed
- Whether a Node Manager or Administration Server restart is required
- Any post-apply SQL or configuration steps

### Check the Current Patch Inventory

```bash
# List all patches installed in the OHS Oracle Home
$ORACLE_HOME/OPatch/opatch lsinventory

# Short listing — patch IDs and descriptions only
$ORACLE_HOME/OPatch/opatch lspatches
```

Note the installed Bundle Patch number and any one-off patches. If a one-off patch is superseded by the new BP, OPatch will report a conflict unless the one-off is rolled back first.

### Check for Conflicts Before Applying

Run the OPatch prerequisite check in analysis mode before any apply operation:

```bash
# Single patch
$ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail \
  -ph /stage/<patch_number>

# If applying multiple patches at once (napply scenario)
$ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail \
  -ph /stage/<patch1>,/stage/<patch2>
```

A clean result means no conflicts. Resolve any reported conflicts (typically by rolling back superseded one-offs) before proceeding.

---

## Pre-Patch Backups

Take these backups before every patching session. A missing backup makes rollback significantly harder.

### 1. Back Up the Oracle Home

Copy the entire OHS Oracle Home to a staging location:

```bash
cp -a $ORACLE_HOME /backup/ORACLE_HOME_backup_$(date +%Y%m%d)
```

For large homes, a tar archive reduces disk usage:

```bash
tar -czf /backup/ohs_home_$(date +%Y%m%d).tar.gz -C $(dirname $ORACLE_HOME) \
  $(basename $ORACLE_HOME)
```

### 2. Back Up the Domain Directory

The WebLogic domain that hosts OHS component instances is stored separately from the Oracle Home.

```bash
cp -a $DOMAIN_HOME /backup/DOMAIN_HOME_backup_$(date +%Y%m%d)
```

### 3. Record the Current Patch Level

Save the current inventory listing for reference:

```bash
$ORACLE_HOME/OPatch/opatch lsinventory > /backup/opatch_lsinventory_pre_$(date +%Y%m%d).txt
$ORACLE_HOME/OPatch/opatch lspatches  > /backup/opatch_lspatches_pre_$(date +%Y%m%d).txt
```

---

## Applying a Bundle Patch or One-Off Patch

### Download and Stage the Patch

1. Log in to My Oracle Support (support.oracle.com).
2. Search for the patch number or navigate to **Patches & Updates > Product or Family (Advanced)** and search for "Oracle HTTP Server 12.2.1.4".
3. Download the patch zip to a staging directory on the OHS host.
4. Unzip the patch:

```bash
unzip p<patch_number>_122140_<platform>.zip -d /stage
```

This creates a directory `/stage/<patch_number>` containing the patch metadata and files.

### Stop OHS Instances

OPatch requires the Oracle Home to be inactive before applying patches to avoid file-locking errors.

```bash
# Stop all OHS component instances in the domain
$DOMAIN_HOME/bin/stopComponent.sh ohs1

# If managed through Node Manager
$ORACLE_HOME/ohs/bin/apachectl -f $DOMAIN_HOME/config/fmwconfig/components/OHS/ohs1/httpd.conf -k stop
```

If the OHS is part of an HA pair, stopping it removes it from the load balancer rotation. Ensure the remaining node is healthy before stopping this one (see Rolling Patch section below).

### Apply the Patch

```bash
cd /stage/<patch_number>

# Apply the patch
$ORACLE_HOME/OPatch/opatch apply
```

OPatch prompts for confirmation before making changes. Review the listed files and confirm.

For non-interactive execution (automated pipelines), use:

```bash
$ORACLE_HOME/OPatch/opatch apply -silent
```

OPatch writes a detailed log to `$ORACLE_HOME/cfgtoollogs/opatch/opatch<timestamp>.log`.

### Apply Multiple One-Off Patches in a Single Operation

When applying several one-offs, use `napply` to apply them atomically and reduce the number of restarts:

```bash
$ORACLE_HOME/OPatch/opatch napply /stage \
  -id <patch1>,<patch2>,<patch3>
```

### Restart OHS After Patching

```bash
$DOMAIN_HOME/bin/startComponent.sh ohs1
```

Verify the process started:

```bash
$ORACLE_HOME/ohs/bin/apachectl -f \
  $DOMAIN_HOME/config/fmwconfig/components/OHS/ohs1/httpd.conf -t
```

---

## Rolling Patch for HA Deployments

In a high-availability topology, two or more OHS nodes sit behind a hardware or software load balancer. Patch them one at a time to maintain service continuity.

### Procedure

1. **Remove node 1 from load balancer rotation.** Use your load balancer console or CLI to disable the node 1 pool member. Confirm active sessions have drained or set a drain timeout per your SLA.

2. **Stop OHS on node 1.**

   ```bash
   $DOMAIN_HOME/bin/stopComponent.sh ohs1
   ```

3. **Apply the patch on node 1** following the standard `opatch apply` workflow above.

4. **Restart OHS on node 1** and validate (see Post-Patch Validation section).

5. **Re-enable node 1 in the load balancer rotation.** Confirm it is receiving traffic.

6. **Remove node 2 from load balancer rotation** and repeat steps 2–5 on node 2.

Key constraint: Both Oracle Home directories must be patched before the patch is considered fully applied across the cluster. Do not run mixed-patch versions in production for longer than the maintenance window requires.

---

## Rollback

OPatch supports rollback of individual patches. Rollback restores the original binary files but does not undo configuration changes made by post-patch scripts (if any were required).

### Roll Back a Single Patch

```bash
# Stop OHS first
$DOMAIN_HOME/bin/stopComponent.sh ohs1

# Roll back the patch
$ORACLE_HOME/OPatch/opatch rollback -id <patch_number>

# Restart OHS
$DOMAIN_HOME/bin/startComponent.sh ohs1
```

### Verify After Rollback

```bash
$ORACLE_HOME/OPatch/opatch lspatches
```

Confirm the patch ID no longer appears in the listing.

### Full Oracle Home Restore

If OPatch rollback fails or the Oracle Home is in an inconsistent state, restore from the pre-patch backup:

1. Stop all OHS instances.
2. Remove the corrupted Oracle Home.
3. Restore from backup:

   ```bash
   cp -a /backup/ORACLE_HOME_backup_<date> $ORACLE_HOME
   ```

4. Start OHS and verify.

---

## Upgrade Path from Earlier OHS 12.2.1.x Releases

Upgrading OHS from 12.2.1.3 (or earlier) to 12.2.1.4 is an out-of-place upgrade: a new Oracle Home is installed alongside the existing one, and existing component instances are reassociated or recreated.

### High-Level Steps

1. **Install OHS 12.2.1.4** into a new Oracle Home using the Oracle Fusion Middleware Infrastructure installer or the standalone Web Tier installer.

   ```bash
   java -jar fmw_12.2.1.4.0_ohs_linux64.jar
   ```

   Select **Installation Type: Collocated HTTP Server (Managed through WebLogic Server)** or **Standalone HTTP Server**, as appropriate.

2. **Apply the latest Bundle Patch** to the new 12.2.1.4 Oracle Home before configuring any domain (patching a fresh home is simpler than patching after domain creation).

3. **Create or re-associate the domain.** Use the Fusion Middleware Configuration Wizard (`config.sh`) to create a new WebLogic domain targeting the 12.2.1.4 home, or extend an existing domain.

   ```bash
   $NEW_ORACLE_HOME/oracle_common/common/bin/config.sh
   ```

4. **Migrate configuration files.** Copy and adapt `httpd.conf`, `mod_wl_ohs.conf`, virtual host definitions, and SSL wallet contents from the old domain to the new one. Do not copy binary files from the old Oracle Home.

5. **Validate the new configuration** (see Post-Patch Validation).

6. **Decommission the old Oracle Home** after a stabilization period.

Oracle does not support in-place binary replacement across minor-release boundaries (e.g., 12.2.1.3 to 12.2.1.4). Only in-release patch application (e.g., applying a BP on top of 12.2.1.4) is in-place.

---

## Post-Patch Validation

Run these checks after every patch application or upgrade.

### 1. Confirm Patch Is Listed in Inventory

```bash
$ORACLE_HOME/OPatch/opatch lspatches
```

The newly applied patch ID must appear in the output.

### 2. Verify the OHS Binary Version

```bash
$ORACLE_HOME/ohs/bin/httpd -version
```

The version string should reflect the patched release level.

### 3. Test the Configuration Syntax

```bash
$ORACLE_HOME/ohs/bin/apachectl \
  -f $DOMAIN_HOME/config/fmwconfig/components/OHS/ohs1/httpd.conf -t
```

Expected output: `Syntax OK`. Any configuration error must be resolved before starting OHS.

### 4. Start OHS and Confirm Process Health

```bash
$DOMAIN_HOME/bin/startComponent.sh ohs1

# Check the process is running
ps -ef | grep httpd | grep -v grep
```

### 5. Check OHS Log Files

Review the error log for startup warnings or errors:

```bash
# Default log location; path may differ if customized in httpd.conf
tail -100 $DOMAIN_HOME/servers/ohs1/logs/ohs1-error.log
```

### 6. Send a Test Request

Use `curl` or a browser to send a request through OHS and confirm it returns the expected response code:

```bash
curl -I http://localhost:<listen_port>/
```

For SSL-enabled configurations:

```bash
curl -kI https://localhost:<ssl_port>/
```

### 7. Verify Back-End Proxy (mod_weblogic)

If OHS proxies requests to a WebLogic cluster via `mod_wl_ohs`, send a request to a proxied URI and confirm the WebLogic application responds:

```bash
curl -I http://localhost:<listen_port>/<proxied_context_root>/
```

---

## Common Errors and Resolutions

### "OPatch failed: opatch is not a recognized command"

The `$ORACLE_HOME/OPatch` directory is missing or OPatch was not unzipped correctly. Confirm the directory exists and `opatch` is executable:

```bash
ls $ORACLE_HOME/OPatch/opatch
chmod +x $ORACLE_HOME/OPatch/opatch
```

### "Prerequisite check failed: OPatch version not supported"

Update OPatch to the version specified in the patch README (see Pre-Patch Requirements above).

### "Patch conflicts with installed patch"

A previously installed one-off patch conflicts with the new Bundle Patch.

```bash
# Identify the conflicting patch
$ORACLE_HOME/OPatch/opatch lspatches

# Roll back the conflicting one-off
$ORACLE_HOME/OPatch/opatch rollback -id <conflicting_patch_id>

# Re-run the apply
$ORACLE_HOME/OPatch/opatch apply
```

Only roll back a one-off if the incoming BP explicitly supersedes it.

### "ApplySession failed: central inventory locked"

A prior OPatch session left a lock file. Confirm no OPatch process is running before removing the lock:

```bash
ps -ef | grep opatch

# If no process is running, remove the lock
rm $ORACLE_HOME/.patch_storage/temp_lock
```

### "Syntax error" in httpd.conf After Patch

A new module introduced by the patch may not be enabled, or a directive syntax changed. Compare the patched `httpd.conf.orig` (saved by OPatch) against your running configuration and merge carefully.

### OHS Fails to Start: "SSL wallet not found"

The wallet path configured in `ssl.conf` may point to a location not present in the new Oracle Home (relevant after an out-of-place upgrade). Update the `SSLWallet` directive to the correct wallet path and restart.

---

## Oracle Version Notes

**OHS 12.2.1.4 vs earlier 12.2.1.x releases:**

- OHS 12.2.1.4 is the terminal long-term support (LTS) release of the 12.2.1 line. Oracle recommends all 12.2.1.x customers upgrade to 12.2.1.4 before applying further BPs, as Oracle reduced patch coverage for 12.2.1.3 and earlier.
- The Bundle Patch model (cumulative quarterly patches) was standardized across Fusion Middleware 12.2.1.4. Earlier releases used a mix of PSUs and BPs; confirm the correct patch type for your exact version in MOS.
- OPatch 13.9.x is required for OHS 12.2.1.4 patching. OHS 12.2.1.3 patching required OPatch 13.9.x as well, but the minimum build number differs — always check the specific patch README.
- The standalone Web Tier installer for OHS was discontinued for releases after 12.2.1.4. Future major versions of OHS are distributed only as part of the Oracle Fusion Middleware Infrastructure.

---

## Sources

- Oracle Fusion Middleware Patching with OPatch — https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/opatc/index.html
- Oracle Fusion Middleware Installing and Configuring Oracle HTTP Server — https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/htins/index.html
- Oracle Fusion Middleware Administrator's Guide for Oracle HTTP Server — https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/hsadm/index.html
- Oracle Fusion Middleware Upgrading Oracle HTTP Server — https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/htupg/index.html
- Oracle OPatch User's Guide for Oracle Fusion Middleware — https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/opatc/opatch-commands-oracle-fusion-middleware.html
- Oracle Fusion Middleware Infrastructure Release Notes 12.2.1.4 — https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/infra-release-notes/index.html
- My Oracle Support — Patch 6880880 (OPatch utility): https://support.oracle.com (search patch 6880880, select platform and OPatch 13.x version)
