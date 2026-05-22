# Oracle HTTP Server 12.2.1.4 Certification Matrix

## Overview

This reference covers the certification matrix for Oracle HTTP Server (OHS) 12.2.1.4, the current long-term supported release in the Oracle Fusion Middleware 12c family. It helps administrators quickly identify supported operating systems, JDK requirements, WebLogic Server integration versions, and Oracle product co-deployment combinations before installing, upgrading, or patching OHS environments.

The authoritative and always-current certification data lives in the My Oracle Support (MOS) certification matrix tool. Treat the values in this file as a starting-point reference and always confirm against the live matrix before a production deployment or upgrade.

---

## Supported Operating Systems

OHS 12.2.1.4 is supported on 64-bit platforms only. 32-bit platforms are not supported.

| Operating System | Versions | Architecture | Notes |
|---|---|---|---|
| Oracle Linux | 7.x, 8.x | x86-64 | Preferred platform; fully tested with Oracle Unbreakable Enterprise Kernel |
| Red Hat Enterprise Linux | 7.x, 8.x | x86-64 | Supported; binary compatible with Oracle Linux packages |
| SUSE Linux Enterprise Server | 12.x, 15.x | x86-64 | Supported; verify exact service pack in MOS certification matrix |
| Windows Server | 2016, 2019 | x86-64 | Supported; verify edition (Standard/Datacenter) in certification matrix |

Notes:
- Oracle Linux 6.x reached end of Premier Support in March 2021; it is no longer a supported OHS 12.2.1.4 platform. Do not deploy new OHS instances on OL6.
- Solaris and AIX are not supported OHS 12.2.1.4 server platforms. These platforms were available in earlier OHS 11g releases but were not carried forward to the 12c line.
- Always apply current OS kernel errata before installing OHS. The OHS installer performs prerequisite kernel parameter checks; refer to the Installation Guide for required values (`fs.file-max`, `net.core.rmem_max`, and related parameters).
- For the definitive and up-to-date platform list, use the MOS certification matrix (see Sources below).

---

## JDK Requirements

OHS 12.2.1.4 is a C-based web server component and does not itself run on the JVM. However, a supported JDK is required on the Oracle Home host because the Oracle Universal Installer (OUI) and Fusion Middleware Infrastructure installer use Java.

| OHS Release | Required JDK | Notes |
|---|---|---|
| OHS 12.2.1.4.0 | Oracle JDK 8 (1.8.0_191 or later) | Required for OUI and FMW Infrastructure installation tooling |
| OHS 12.2.1.4.0 | Oracle JDK 11 (11.0.x) | Supported for installer tooling as of later 12.2.1.4 Bundle Patches; verify exact BP level in patch readme |

Notes:
- OpenJDK is not supported for Fusion Middleware installation tooling. Use Oracle JDK from Oracle Technology Network or an Oracle Linux yum channel.
- If OHS is co-located with WebLogic Server in a Collocated Installation (managed server proxy topology), the same JDK used by WebLogic applies to the shared Oracle Home environment. Refer to the WebLogic JDK certification guidance.
- Verify the exact minimum JDK patch level required for each OHS Bundle Patch or PSU in the patch readme available on MOS.

---

## Supported Oracle Product Integrations

### WebLogic Server Integration

OHS 12.2.1.4 is most commonly deployed as a reverse proxy in front of WebLogic Server managed servers, using the WebLogic Plugin (mod_wl_ohs). The following WebLogic Server versions are certified with OHS 12.2.1.4.

| WebLogic Server Version | OHS Integration Support | Notes |
|---|---|---|
| WebLogic Server 12.2.1.3 | Supported | Earlier 12.2.1.x release; verify current patch requirements in MOS |
| WebLogic Server 12.2.1.4 | Supported (recommended pairing) | Same FMW 12.2.1.4 release family; recommended for new deployments |
| WebLogic Server 14.1.1.0 | Verify in MOS certification matrix | OHS 12.2.1.4 with WLS 14.1.1 requires validation of mod_wl_ohs plugin version |

Notes:
- The mod_wl_ohs plugin (WebLogic Proxy Plugin) must be version-compatible with the target WebLogic Server. Plugin updates are delivered via OHS Bundle Patches and should not be mixed across incompatible release levels.
- WebLogic Server 10.3.x (11g) co-deployment with OHS 12.2.1.4 is not supported. OHS 11g (11.1.1.x) was the corresponding version for that WebLogic release family.
- For Fusion Middleware collocated deployments, both OHS and WebLogic must share the same supported 12.2.1.4 patch baseline. Mismatched patch levels may cause proxy plugin failures.
- Always verify the specific OHS-to-WebLogic combination in the MOS certification matrix before upgrading either component independently.

### Oracle HTTP Server — Standalone vs. Collocated Topologies

OHS 12.2.1.4 supports two primary installation topologies:

| Topology | Description | Notes |
|---|---|---|
| Standalone Domain | OHS runs in its own WebLogic domain without a full WLS server | Recommended for front-end web tier; lightweight; managed via WLST or Node Manager |
| Collocated Installation | OHS and WLS Managed Server share the same Oracle Home | Used when OHS proxies to collocated WLS application components; requires a Fusion Middleware Infrastructure home |

### Oracle Fusion Middleware Components

OHS 12.2.1.4 is part of the Oracle Fusion Middleware 12c (12.2.1.4) product family and is certified for co-deployment with the following components at the same release level:

| FMW Component | Version | Notes |
|---|---|---|
| Oracle Fusion Middleware Infrastructure | 12.2.1.4 | Required for collocated OHS topologies |
| Oracle SOA Suite | 12.2.1.4 | OHS commonly serves as the HTTP front-end for SOA composite endpoints |
| Oracle Service Bus | 12.2.1.4 | OHS proxies OSB proxy service URIs |
| Oracle Identity and Access Management | 12.2.1.4 | OHS front-ends OAM and OIM; WebGate integration is a separate certification (see OAM documentation) |
| Oracle Access Manager WebGate | 12.2.1.4 | WebGate (mod_oblix or mod_wg) is installed into the OHS Oracle Home; must match the OAM server version |

Note: Oracle Access Manager WebGate has its own certification matrix entry in MOS separate from the base OHS entry. Verify the WebGate version, OHS version, and OAM server version as a three-way combination.

---

## Browser and Client Compatibility

OHS itself is an HTTP server infrastructure component and does not expose a browser-based management UI equivalent to OEM or APEX. Browser compatibility is relevant in two areas:

### Oracle HTTP Server Administration via Fusion Middleware Control (Enterprise Manager)

If OHS is managed through Oracle Fusion Middleware Control (the FMW EM console running in a WebLogic Admin Server), access to that console requires a supported browser.

| Browser | Notes |
|---|---|
| Google Chrome (current stable) | Recommended |
| Mozilla Firefox (current ESR) | Recommended |
| Microsoft Edge (Chromium, current stable) | Supported |
| Internet Explorer 11 | Verify in MOS certification matrix; generally deprecated |

### Client HTTP/TLS Compatibility

OHS 12.2.1.4 uses OpenSSL-based Oracle Wallet for SSL/TLS. Supported protocol and cipher configuration depends on the Oracle Database and FMW patch level applied.

| Protocol | Default Status | Notes |
|---|---|---|
| TLS 1.2 | Enabled | Minimum recommended for production deployments |
| TLS 1.3 | Enabled (later Bundle Patches) | Requires a Bundle Patch that includes the relevant OpenSSL update; verify BP level |
| TLS 1.0 / 1.1 | Disabled or restricted | Oracle recommends disabling TLS 1.0 and 1.1; refer to OHS security hardening documentation |
| SSL 3.0 | Not supported | Disabled; do not enable |

---

## How to Check the Certification Matrix (My Oracle Support)

The live certification matrix at My Oracle Support is the only authoritative source for OHS certification data. Version support changes with each Bundle Patch and Release Update.

Steps to access:

1. Log in to My Oracle Support at https://support.oracle.com.
2. Navigate to the Certifications tab.
3. In the Product field, enter "Oracle HTTP Server".
4. Select the OHS release (e.g., 12.2.1.4.0).
5. Select the operating system or integration product you want to verify.
6. Review the certified combinations and any associated notes or required patches.

Useful MOS Notes for OHS 12.2.1.4 certification:
- MOS Note 1492194.1 — "Oracle Fusion Middleware 12c Certification Matrix" (index note for all FMW components including OHS)
- MOS Note 2539143.1 — "Oracle HTTP Server 12c (12.2.1.x) Proactive Patch Information" (patch and bundle patch guidance)
- MOS Note 1905622.1 — "Interoperability Notes: Oracle HTTP Server 12c with Oracle WebLogic Server"

Note: MOS Note numbers are provided as research starting points. Always confirm their current content and applicability, as Oracle updates Notes over time.

---

## Oracle Version Notes

### OHS 12.2.1.4 vs. Earlier 12.2.1.x Releases

- OHS 12.2.1.4 is the terminal and long-term supported release of the 12.2.1.x line. Oracle has ended standard support for 12.2.1.3 and earlier; new deployments should use 12.2.1.4.
- TLS 1.3 support was not available in the initial 12.2.1.4 release. It was introduced through later Bundle Patches. Verify the specific Bundle Patch level required for TLS 1.3 in the patch readme on MOS.
- Oracle Linux 8 and Windows Server 2019 were added as certified platforms in later 12.2.1.4 Bundle Patches. Earlier 12.2.1.x releases did not certify these OS versions. Confirm the exact BP level required for newer OS support in MOS.

### OHS 12.2.1.4 vs. OHS 11g (11.1.1.x)

- OHS 11g is end-of-life and no longer receives patches. Do not plan new deployments on OHS 11g.
- OHS 12c uses a different module architecture from OHS 11g. Apache module compatibility is not assumed; custom modules compiled for OHS 11g must be recompiled and recertified for OHS 12c.
- mod_osso (Oracle SSO) is not available in OHS 12.2.1.4. Oracle Access Manager WebGate (mod_wg) is the replacement for SSO integration.

---

## Sources

- Oracle HTTP Server 12c Administrator's Guide (12.2.1.4):
  https://docs.oracle.com/en/middleware/fusion-middleware/web-tier/12.2.1.4/admhs/

- Oracle HTTP Server 12c Installation Guide (as part of Oracle Fusion Middleware Installation Guide for Oracle Web Tier):
  https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/wtins/

- Oracle Fusion Middleware 12c (12.2.1.4) Release Notes:
  https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/infra-relnotes/

- Oracle Fusion Middleware 12c Supported System Configurations (docs.oracle.com overview):
  https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/

- Oracle WebLogic Server Proxy Plugin documentation (mod_wl_ohs):
  https://docs.oracle.com/en/middleware/standalone/weblogic-server/14.1.1.0/wlprx/

- My Oracle Support Certification Matrix — Oracle HTTP Server (login required):
  https://support.oracle.com/certify

- My Oracle Support — Fusion Middleware 12c Certification Matrix Index Note 1492194.1 (login required):
  https://support.oracle.com/epmos/faces/DocumentDisplay?id=1492194.1

- My Oracle Support — OHS 12c Proactive Patch Information Note 2539143.1 (login required):
  https://support.oracle.com/epmos/faces/DocumentDisplay?id=2539143.1
