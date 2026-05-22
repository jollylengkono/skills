# Oracle GoldenGate Patching and Upgrade

## Overview

Use this skill to plan and execute Oracle GoldenGate patching and upgrade operations safely. Coverage includes OPatch-based patching for Classic GoldenGate installations, upgrading from 12c release lines to 21c, rolling upgrade procedures for near-zero downtime, pre-upgrade checklist steps, GoldenGate Microservices deployment upgrade, and post-upgrade validation.

Load `references/goldengate-source-map.md` whenever version support or certification must be verified before selecting a patch or target release.

## Core Concepts

### Patch Types

Oracle GoldenGate patches are delivered as Oracle patch bundles applied with OPatch (for installations that include an Oracle Home) or as full-installer replacements for standalone Classic installations.

- **Bundle Patch (BP)**: periodic cumulative patch sets released on a quarterly cadence, aligned with Oracle's Critical Patch Update (CPU) schedule.
- **Patch Set Update (PSU)**: security-focused patch bundles for specific release lines; replaced by Bundle Patches for most GoldenGate lines.
- **One-Off Patch**: single-issue fix applied on top of a Bundle Patch; always apply the current Bundle Patch first, then the one-off.

### Upgrade vs. Patch

- A **patch** applies fixes within the same release line (e.g., 21.3.0.0.0 to 21.3.0.0.1).
- An **upgrade** installs a new Oracle GoldenGate release (e.g., 12.3.0.1 to 21.3).
- For upgrades across major release lines, a new Oracle Home is created alongside the existing installation; processes are migrated, not in-place overwritten.

### Classic vs. Microservices Architecture

- **Classic GoldenGate**: file-system based installation; patched with OPatch or by installing a new GoldenGate home and re-pointing processes.
- **GoldenGate Microservices Architecture (MA)**: introduced as default from 19c; managed via the Service Manager, Admin Server, and REST API. Upgrades involve deploying a new service and migrating deployments.

## Pre-Upgrade Checklist

Complete all checklist items before stopping processes or replacing binaries.

### 1. Identify Current Release

```text
# From the GoldenGate home directory (Classic):
./ggsci
GGSCI> INFO ALL
GGSCI> ! ./ggsci --version   (or)
$ ./ggsci <<< "VERSIONS"
```

For Microservices: check the Admin Server home page or use:
```text
GET /services/v2/version   (Admin Server REST endpoint)
```

Record the exact release string (e.g., `12.3.0.1.4`). Match it against the certification matrix before selecting a target release.

### 2. Verify Certification

- Confirm source database version, OS platform, and JDK version are certified for the target GoldenGate release.
- Reference: https://www.oracle.com/integration/goldengate/certifications/
- Reference: https://docs.oracle.com/en/middleware/goldengate/core/21.3/coredoc/install-verify-certification-and-system-requirements.html

### 3. Drain Trail Files

Allow Extract to write all captured transactions to the trail before stopping processes.

```text
GGSCI> SEND EXTRACT <name>, LOGEND
```

Wait for lag to reach near zero. Confirm:

```text
GGSCI> LAG EXTRACT <name>
GGSCI> LAG REPLICAT <name>
```

Do not proceed until Replicat lag is zero or within the acceptable window defined by the maintenance plan.

### 4. Stop Processes in the Correct Sequence

Stop in this order to prevent trail file gaps:

1. Stop Extract (capture stops; no new records written to trail):
   ```text
   GGSCI> STOP EXTRACT <extract_name>
   ```
2. Stop Pump Extract (if a dedicated Pump is used):
   ```text
   GGSCI> STOP EXTRACT <pump_name>
   ```
3. Stop Replicat (apply stops; checkpoint is written):
   ```text
   GGSCI> STOP REPLICAT <replicat_name>
   ```

Confirm all processes are stopped:

```text
GGSCI> INFO ALL
```

All statuses must show `STOPPED` before proceeding.

### 5. Back Up the GoldenGate Home and Parameter Files

- Copy the entire GoldenGate home directory to a backup location.
- Back up the `dirprm/` directory separately; it contains all parameter files.
- Record current checkpoint positions:
  ```text
  GGSCI> INFO EXTRACT <name>, DETAIL
  GGSCI> INFO REPLICAT <name>, DETAIL
  ```

### 6. Confirm Disk Space

Verify adequate disk space for the new GoldenGate home and trail retention during the upgrade window.

---

## Patching Classic GoldenGate with OPatch

Applies when the GoldenGate installation is part of an Oracle Middleware home that includes an OPatch installation.

### Step 1: Verify OPatch Version

The OPatch version must meet the minimum required by the patch. Check the patch README for the required version.

```bash
$ORACLE_HOME/OPatch/opatch version
```

Update OPatch if required: download the latest OPatch from Oracle Support (patch 6880880) and replace the existing OPatch directory.

### Step 2: Verify No Active Processes

All GoldenGate processes must be stopped (see Pre-Upgrade Checklist, Step 4).

### Step 3: Apply the Patch

```bash
cd <patch_directory>
$ORACLE_HOME/OPatch/opatch apply
```

OPatch will validate prerequisites, apply the patch, and update the Oracle Home inventory.

### Step 4: Confirm Patch Application

```bash
$ORACLE_HOME/OPatch/opatch lspatches
```

Confirm the patch ID appears in the output.

### Step 5: Restart Processes

```text
GGSCI> START EXTRACT <extract_name>
GGSCI> START EXTRACT <pump_name>
GGSCI> START REPLICAT <replicat_name>
```

Verify process states and lag after restart (see Post-Upgrade Validation).

---

## Upgrading Classic GoldenGate: 12c to 21c

This procedure installs GoldenGate 21c in a new home directory while the 12c installation remains intact for rollback.

### Step 1: Download GoldenGate 21c

Obtain the GoldenGate 21c installation archive from Oracle Software Delivery Cloud or My Oracle Support. Verify the checksum against the download page.

### Step 2: Install GoldenGate 21c in a New Home

```bash
# Create a new home directory separate from the 12c home
mkdir -p /u01/app/oracle/goldengate21c

# Extract the archive
unzip gg21c_<platform>.zip -d /tmp/gg21c_install

# Run the installer (silent mode example)
/tmp/gg21c_install/fbo_ggs_Linux_x64_shiphome/Disk1/runInstaller \
  -silent \
  -responseFile /tmp/gg21c_install/response/oggcore.rsp
```

Customize `oggcore.rsp` with the correct `INSTALL_OPTION`, `SOFTWARE_LOCATION`, and `MANAGER_PORT` values before running the installer. Refer to the installation guide for all required response file parameters:
https://docs.oracle.com/en/middleware/goldengate/core/21.3/installing/

### Step 3: Run the Upgrade Assistant (if applicable)

For installations under an Oracle Fusion Middleware umbrella, run the Upgrade Assistant to migrate configuration metadata. Refer to:
https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/oamig/

For standalone GoldenGate installations, parameter files and checkpoint files are manually migrated (see Step 4).

### Step 4: Migrate Parameter Files

Copy the 12c `dirprm/` contents to the 21c GoldenGate home:

```bash
cp /u01/app/oracle/goldengate12c/dirprm/*.prm \
   /u01/app/oracle/goldengate21c/dirprm/
```

Review each parameter file for deprecated parameters before starting processes. The 21c release notes document removed or changed parameters.

### Step 5: Update the Manager Parameter File

Ensure the Manager parameter file (`mgr.prm`) in the 21c home has the correct `PORT`, `ACCESSRULE`, and `PURGEOLDEXTRACTS` settings for the new environment.

### Step 6: Start the 21c Manager

```bash
cd /u01/app/oracle/goldengate21c
./ggsci
GGSCI> START MANAGER
GGSCI> INFO MANAGER
```

### Step 7: Register Extracts and Replicats in the New Home

```text
GGSCI> ADD EXTRACT <name>, INTEGRATED TRANLOG, BEGIN NOW
GGSCI> ADD EXTTRAIL <path>, EXTRACT <name>
GGSCI> ADD REPLICAT <name>, INTEGRATED, EXTTRAIL <path>
```

Restore checkpoint positions from the backed-up `DETAIL` output if recovering an in-progress migration.

---

## Rolling Upgrade for Near-Zero Downtime

A rolling upgrade runs the new GoldenGate release on the target side while the source continues with the existing release. This approach minimizes apply-side downtime.

### Applicable Scenarios

- Source Extract remains on the existing release (12c or 19c) during the upgrade window.
- A new Replicat on the upgraded release applies trail files produced by the older Extract.
- Trail file format compatibility between source and target GoldenGate releases must be confirmed before proceeding. Refer to the release notes for the target GoldenGate version.

### Procedure

1. Install the new GoldenGate release on the target system (new home directory).
2. Leave the source Extract running; do not stop it.
3. On the target, stop the existing Replicat:
   ```text
   GGSCI> STOP REPLICAT <name>
   ```
4. Record the Replicat checkpoint (trail file sequence and RBA):
   ```text
   GGSCI> INFO REPLICAT <name>, DETAIL
   ```
5. Add the Replicat in the new 21c home using the recorded checkpoint position:
   ```text
   GGSCI> ADD REPLICAT <name>, EXTTRAIL <path>, BEGIN <seqno> <rba>
   ```
6. Start the new Replicat and monitor lag:
   ```text
   GGSCI> START REPLICAT <name>
   GGSCI> LAG REPLICAT <name>
   ```
7. Once the Replicat is confirmed stable, upgrade the source Extract in the next maintenance window.

---

## GoldenGate Microservices Deployment Upgrade

Microservices Architecture (MA) deployments are upgraded by deploying a new version of the Service Manager and migrating existing deployments.

### Step 1: Review the MA Upgrade Documentation

Before proceeding, read the upgrade chapter for the target release:
- GoldenGate 21.3 MA upgrade: https://docs.oracle.com/en/middleware/goldengate/core/21.3/ggcab/upgrading-oracle-goldengate-microservices.html

### Step 2: Stop Active Processes

Use the Admin Server UI or Admin Client to stop all Extracts and Replicats cleanly. Drain trail files before stopping (see Pre-Upgrade Checklist, Step 3).

### Step 3: Install the New GoldenGate Microservices Software

Install the new release into a separate Oracle Home. Do not overwrite the existing installation.

```bash
unzip gg21c_microservices_<platform>.zip -d /tmp/gg21c_ma
/tmp/gg21c_ma/fbo_ggs_Linux_x64_shiphome/Disk1/runInstaller \
  -silent \
  -responseFile /tmp/gg21c_ma/response/oggcore.rsp
```

### Step 4: Run the Oracle GoldenGate Configuration Assistant (OGGCA)

OGGCA handles Service Manager deployment and migrates existing deployment configurations to the new home. Run in silent mode for automated environments:

```bash
$NEW_GG_HOME/bin/oggca.sh \
  -silent \
  -responseFile /path/to/oggca.rsp
```

Refer to the OGGCA documentation for required response file fields:
https://docs.oracle.com/en/middleware/goldengate/core/21.3/ggcab/

### Step 5: Verify the Service Manager

Access the Service Manager at `https://<host>:<port>` and confirm all deployments are listed and their processes show `Running` status.

### Step 6: Restart Processes via Admin Server

Use the Admin Server UI or REST API to restart Extract and Replicat processes in the correct order (Extract first, then Replicat).

---

## Post-Upgrade Validation

Complete all validation steps before declaring the upgrade successful.

### 1. Verify Process States

```text
GGSCI> INFO ALL
```

All Extract and Replicat processes must show `RUNNING`. Any `ABENDED` process requires immediate investigation via `VIEW REPORT <name>`.

### 2. Check Lag

```text
GGSCI> LAG EXTRACT <name>
GGSCI> LAG REPLICAT <name>
```

Lag should return to the pre-upgrade baseline within the expected catch-up window. Persistent lag increase after catch-up warrants investigation.

### 3. Verify Trail File Continuity

```text
GGSCI> INFO EXTRACT <name>, DETAIL
GGSCI> INFO REPLICAT <name>, DETAIL
```

Confirm checkpoint positions are advancing. A checkpoint that does not advance indicates the process is not making progress.

### 4. Review the Process Report Files

```text
GGSCI> VIEW REPORT <extract_name>
GGSCI> VIEW REPORT <replicat_name>
```

Look for any warning or error messages written since restart.

### 5. Check the Discard File

If `DISCARDFILE` is configured, review it for any records rejected during the first post-upgrade apply cycle:

```text
GGSCI> VIEW DISCARD <replicat_name>
```

### 6. Validate Row Counts

Run source and target row-count queries on a representative set of high-volume tables to confirm data has applied correctly. Use `STATS REPLICAT <name>` to confirm operation counts are consistent with expectations.

---

## Best Practices and Common Mistakes

### Best Practices

- Always install the new GoldenGate release in a separate home directory. Never overwrite an active installation.
- Always drain trail files and verify zero lag before stopping any process.
- Maintain the 12c home intact until the upgrade is fully validated; it provides a fast rollback path.
- Test parameter file compatibility in a non-production environment before applying changes to production.
- Align GoldenGate patch cycles with the Oracle CPU schedule to minimize exposure to known vulnerabilities.
- Confirm OPatch version requirements before applying any patch; an outdated OPatch silently fails validation on some patch bundles.

### Common Mistakes

- **Stopping Extract before draining the trail**: if Extract is stopped while the Pump or Replicat has not caught up, trail files accumulate and apply lag increases.
- **Overwriting the existing GoldenGate home**: in-place overwrite removes the rollback path and can corrupt active checkpoint files.
- **Skipping parameter file review after upgrade**: deprecated parameters from 12c are not always fatal on startup in 21c but can produce unexpected behavior; always review the release notes parameter changes section.
- **Applying a one-off patch without the current Bundle Patch**: Oracle Support requires the current Bundle Patch as the baseline before one-off patches are applied.
- **Not confirming trail file format compatibility in a rolling upgrade**: trail files written by an older Extract release may not be readable by a significantly newer Replicat without a compatibility setting; confirm in the release notes before proceeding.
- **Forgetting to update ACCESSRULE in the Manager parameter file**: a new GoldenGate home requires the Manager `ACCESSRULE` entries to be carried forward; without them, remote GGSCI connections are rejected.

---

## Oracle Version Notes (19c vs 26ai)

### Classic GoldenGate (pre-19c)

- GoldenGate 12c release lines (12.1.2, 12.2.0.1, 12.3.0.1) use Classic Architecture exclusively: file-system based installation, OPatch or full-installer patching, GGSCI for all process management.
- Upgrading from 12c to 21c requires a new home installation; there is no in-place upgrade path between major release lines.
- 12c parameter files may contain parameters that are deprecated or removed in 21c. Review the 21c release notes before migrating parameter files.

### GoldenGate Microservices Architecture (19c+)

- GoldenGate 19c introduced Microservices Architecture as the default deployment model. Patching and upgrading MA deployments uses OGGCA and the Service Manager, not direct OPatch commands on process binaries.
- Rolling upgrades between 19c and 21c MA deployments are supported; verify trail file format compatibility in the 21c release notes before starting a rolling upgrade.
- For 26ai targets: GoldenGate release certification for Oracle Database 26ai must be confirmed at https://www.oracle.com/integration/goldengate/certifications/ before selecting the GoldenGate version to deploy. Do not assume 21c or 23c certification for 26ai without explicit verification.
- Post-upgrade, re-validate all Replicat parameter files for 26ai-specific syntax or behavior changes documented in the release notes.

### Upgrade Path Summary

| From | To | Architecture | Notes |
|---|---|---|---|
| 12.1.2 | 21.3 | Classic → Classic or MA | Staged upgrade through 12.3.0.1 or 19.1 recommended; verify certification |
| 12.3.0.1 | 21.3 | Classic → Classic or MA | Direct upgrade path; verify certification before proceeding |
| 19.1 MA | 21.3 MA | MA → MA | OGGCA-driven; rolling upgrade supported |
| 21.3 MA | 23 MA | MA → MA | OGGCA-driven; verify certification |

---

## Sources

- GoldenGate 21.3 installation guide: https://docs.oracle.com/en/middleware/goldengate/core/21.3/installing/
- GoldenGate 21.3 Microservices Architecture upgrade guide: https://docs.oracle.com/en/middleware/goldengate/core/21.3/ggcab/upgrading-oracle-goldengate-microservices.html
- GoldenGate 21.3 core documentation: https://docs.oracle.com/en/middleware/goldengate/core/21.3/ggcab/overview-oracle-goldengate.html
- GoldenGate 23 core documentation: https://docs.oracle.com/en/middleware/goldengate/core/23/coredoc/toc.htm
- GoldenGate 19.1 documentation: https://docs.oracle.com/en/middleware/goldengate/core/19.1/
- GoldenGate 12.3.0.1 documentation: https://docs.oracle.com/en/middleware/goldengate/core/12.3.0.1/books.html
- GoldenGate documentation hub: https://docs.oracle.com/en/database/goldengate/core/
- GoldenGate certifications and system requirements: https://www.oracle.com/integration/goldengate/certifications/
- GoldenGate 21.3 certification and system requirements guide: https://docs.oracle.com/en/middleware/goldengate/core/21.3/coredoc/install-verify-certification-and-system-requirements.html
