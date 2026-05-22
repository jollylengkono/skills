# Oracle Enterprise Manager 13c Certification Matrix

## Overview

This reference covers the certification matrix for Oracle Enterprise Manager (OEM) Cloud Control 13c, specifically release 13.5. It helps administrators quickly identify supported target versions, operating systems, JDK versions, Oracle Management Repository (OMR) database versions, browsers, and plug-ins before deploying or upgrading OEM environments.

The authoritative and always-current certification data lives in the My Oracle Support (MOS) certification matrix tool. Treat the values in this file as a starting-point reference and always confirm against the live matrix before a production deployment or upgrade.

---

## Supported Oracle Database Target Versions

OEM 13.5 can monitor and manage the following Oracle Database target versions. Management Agent must be deployed to each target host.

| Database Version | Target Support Status |
|---|---|
| Oracle Database 11g R2 (11.2) | Supported (verify specific patch requirements in MOS certification matrix) |
| Oracle Database 12c R1 (12.1) | Supported |
| Oracle Database 12c R2 (12.2) | Supported |
| Oracle Database 18c (18.x) | Supported |
| Oracle Database 19c (19.x) | Supported (Long-Term Release, recommended baseline) |
| Oracle Database 21c (21.x) | Supported (Innovation Release; verify in certification matrix) |

Notes:
- Extended Support for 11g R2 ended in December 2020. Continued monitoring may work but Oracle does not guarantee full feature parity on out-of-support targets.
- Always deploy the latest Management Agent release to ensure plug-in compatibility with newer database targets.
- RAC, Data Guard, and Multitenant targets follow the same version matrix but may require specific configuration steps documented in the OEM Administration Guide.

---

## Supported Operating Systems

### Oracle Management Service (OMS) Host

OMS 13.5 is supported on 64-bit platforms only.

| Operating System | Versions | Notes |
|---|---|---|
| Oracle Linux | 7.x, 8.x | Preferred platform; fully tested |
| Red Hat Enterprise Linux | 7.x, 8.x | Supported |
| SUSE Linux Enterprise Server | 12.x, 15.x | Supported; verify in certification matrix |
| Windows Server | 2016, 2019 | Supported for OMS; verify in certification matrix |

Note: Solaris and AIX are not supported as OMS hosts in 13.5. OMS must run on a supported Linux or Windows Server platform.

### Management Agent Host

The Management Agent supports a broader set of platforms because it must deploy to monitored target hosts.

| Operating System | Versions | Notes |
|---|---|---|
| Oracle Linux / RHEL | 6.x, 7.x, 8.x | Supported |
| SUSE Linux Enterprise Server | 12.x, 15.x | Supported |
| Oracle Solaris (SPARC) | 11.x | Supported; verify exact release in certification matrix |
| Oracle Solaris (x86-64) | 11.x | Supported; verify exact release in certification matrix |
| IBM AIX (Power) | 7.1, 7.2 | Supported; verify in certification matrix |
| Windows Server | 2012 R2, 2016, 2019 | Supported |
| HP-UX Itanium | 11.31 | Verify in certification matrix |

For the definitive and up-to-date platform list, use the MOS certification matrix (see Sources below).

---

## Supported JDK Versions

OEM 13.5 ships with and requires a specific JDK. Oracle bundles Oracle JDK with the OEM installer; do not replace it with an external JDK without guidance from Oracle Support.

| OEM Release | Bundled JDK | Notes |
|---|---|---|
| OEM 13.5 (13.5.0.x) | Oracle JDK 1.8 (JDK 8) | Bundled; no separate JDK installation required |

- OEM does not support OpenJDK or third-party JDK distributions unless explicitly certified by Oracle.
- JDK upgrades within a major OEM release (e.g., 13.5.0.0 to 13.5.0.x) may update the bundled JDK as part of the OEM patch. Verify with each Bundle Patch or Release Update applied.
- Verify the exact JDK patch level required for a given OEM Bundle Patch in the patch readme on MOS.

---

## Supported Oracle Management Repository (OMR) Database Versions

The Oracle Management Repository stores all OEM data and runs inside a supported Oracle Database instance. This is separate from any monitored target databases.

| Database Version | OMR Support Status |
|---|---|
| Oracle Database 12c R2 (12.2) | Supported; verify patch baseline in MOS |
| Oracle Database 19c (19.x) | Supported and recommended for new deployments |
| Oracle Database 21c (21.x) | Verify in certification matrix before use |

Notes:
- Oracle Database 11g is no longer a supported OMR platform for OEM 13.5.
- Oracle recommends 19c as the OMR database for new OEM 13.5 deployments due to its Long-Term Support status.
- The OMR database must have specific initialization parameters configured. Refer to the OEM Installation Guide for required parameter values (e.g., `db_block_size`, `open_cursors`, `session_cached_cursors`).
- Multitenant (CDB/PDB) is supported for the OMR as of OEM 13.5 with specific configuration requirements; verify the exact patch requirements in MOS Note 2170830.1.

---

## Browser Compatibility Matrix

The OEM 13c Cloud Control console is a web application accessed through a supported browser.

| Browser | Versions | Notes |
|---|---|---|
| Google Chrome | Current stable release | Recommended |
| Mozilla Firefox | Current ESR (Extended Support Release) | Recommended |
| Microsoft Edge (Chromium) | Current stable release | Supported; verify in certification matrix |
| Internet Explorer | 11 | Support status: verify in certification matrix; generally deprecated in favor of Chromium-based browsers |
| Apple Safari | Current stable release (macOS) | Verify in certification matrix |

Notes:
- Oracle regularly updates browser support as browsers release new major versions. Always check the MOS certification matrix for the specific OEM release in use.
- Pop-up blockers must be disabled for the OEM console domain.
- TLS 1.2 or later is required for secure connections to the OEM console.

---

## Supported Plug-in Versions

OEM uses plug-ins to extend monitoring and management capabilities. The plug-in version must be certified against both the OEM release and the target version being managed.

### Oracle Database Plug-in

| OEM Release | Plug-in Version | Supported DB Targets |
|---|---|---|
| OEM 13.5 | 13.5.x.x.x | 11.2, 12.1, 12.2, 18c, 19c, 21c (verify exact sub-version in MOS) |

### Oracle WebLogic Server Plug-in

| OEM Release | Plug-in Version | Supported WLS Targets |
|---|---|---|
| OEM 13.5 | 13.9.x.x.x (verify in MOS) | WebLogic 10.3.x, 12.1.x, 12.2.x, 14.x (verify in certification matrix) |

### Oracle Fusion Middleware Plug-in

| OEM Release | Plug-in Version | Notes |
|---|---|---|
| OEM 13.5 | 13.9.x.x.x (verify in MOS) | Covers SOA Suite, Service Bus, Identity Management, and other FMW components; verify exact target version support in certification matrix |

### Oracle Host Plug-in

| OEM Release | Plug-in Version | Notes |
|---|---|---|
| OEM 13.5 | Ships with Management Agent | The host plug-in monitors OS-level metrics; it is bundled with the Agent and does not require a separate download in most cases |

General plug-in guidance:
- Plug-ins are versioned independently of OEM and of each other.
- Not all plug-in versions work with all OEM releases; use the MOS certification matrix to confirm a specific plug-in version against your OEM release.
- After an OEM upgrade, run the plug-in prerequisite check before applying or upgrading plug-ins.
- Use the Self Update feature in OEM (Setup > Extensibility > Self Update) to obtain the latest certified plug-in versions.

---

## How to Check the Certification Matrix (My Oracle Support)

The live certification matrix at My Oracle Support is the only authoritative source for OEM certification data. Version support changes with each Bundle Patch and Release Update.

Steps to access:

1. Log in to My Oracle Support at https://support.oracle.com.
2. Navigate to the Certifications tab.
3. In the Product field, enter "Enterprise Manager Base Platform - OMS".
4. Select the OEM release (e.g., 13.5.0.0.0).
5. Select the operating system or database version you want to verify.
6. Review the certified combinations and any associated notes or required patches.

Useful MOS Notes for OEM 13.5 certification:
- MOS Note 2073163.1 — "Master Note: Enterprise Manager Cloud Control 13c (13.5) Upgrade / Install"
- MOS Note 2170830.1 — "Enterprise Manager Cloud Control 13c: Repository on Multitenant (CDB) Database"
- MOS Note 1079123.1 — "Enterprise Manager: Supported Configurations" (index note; links to platform-specific details)

Note: MOS Note numbers are provided as research starting points. Always confirm their current content and applicability, as Oracle updates Notes over time.

---

## Oracle Version Notes

OEM 13.5 introduced or changed certification for several components relative to 13.4:

- Oracle Database 21c target support was added; 11g targets remain reachable but require verifying agent compatibility given 11g's end-of-standard-support status.
- CDB/PDB (Multitenant) as the OMR database became more formally documented in 13.5; earlier 13.x releases had restrictions.
- Internet Explorer support has progressively diminished across 13.x releases; Chromium-based browsers are the current recommendation.

---

## Sources

- Oracle Enterprise Manager Cloud Control 13c Installation Guide:
  https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emins/

- Oracle Enterprise Manager Cloud Control 13c Administrator's Guide:
  https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emadm/

- Oracle Enterprise Manager Cloud Control 13c Upgrade Guide:
  https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emupg/

- Oracle Enterprise Manager System Monitoring Plug-in documentation (index):
  https://docs.oracle.com/en/enterprise-manager/

- My Oracle Support Certification Matrix (login required):
  https://support.oracle.com/certify

- My Oracle Support — Enterprise Manager Master Note 2073163.1 (login required):
  https://support.oracle.com/epmos/faces/DocumentDisplay?id=2073163.1
