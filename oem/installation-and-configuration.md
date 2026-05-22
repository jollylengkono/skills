# Oracle Enterprise Manager 13c — Installation and Configuration

## Overview

Oracle Enterprise Manager (OEM) 13c is Oracle's integrated cloud control platform for managing Oracle Database, WebLogic Server, Fusion Middleware, and host targets. This skill covers the full lifecycle from pre-installation planning through agent deployment and high availability setup.

The three main components are:

- **Oracle Management Service (OMS)** — the mid-tier Java EE application that serves the Cloud Control console and coordinates agent communication. Runs inside a WebLogic Server domain.
- **Oracle Management Repository (OMR)** — an Oracle Database that stores all monitoring data, configuration metadata, and job definitions.
- **Oracle Management Agent (OMA)** — a lightweight process deployed on each monitored host that collects metric data and forwards it to OMS.

---

## Installation Types

OEM 13c provides three installation types selectable from the installer wizard.

### Simple Install

Installs the OMS and creates the OMR database on the same host using a bundled database template. All components are configured automatically.

Use for evaluation and proof-of-concept only. Simple Install is not suitable for production because it collocates all components on one host with no HA path.

### Advanced Install

The recommended production installation type. The installer deploys the OMS on one host and connects it to a pre-existing, separately managed OMR database. Supports full port customization, SSL configuration, and multiple plug-in selection.

Use Advanced Install for all production and production-like environments.

### Software Only Install

Installs the OMS software binaries without configuring or starting any services. After a Software Only install, run the Configuration Assistant (`omsca`) separately to configure the OMS.

Use Software Only Install when:
- Building a multi-OMS HA topology by adding a second OMS to an existing site.
- The installation and configuration steps must be performed at different times or by different teams.
- Deploying to a shared or cloned file system image that will be configured later.

---

## Prerequisites

### Operating System

OEM 13c Release 5 (13.5) is certified on:

- Oracle Linux 7.x and 8.x (x86-64)
- Red Hat Enterprise Linux 7.x and 8.x (x86-64)
- SUSE Linux Enterprise Server 15.x (x86-64)

Check the OEM 13c certification matrix on My Oracle Support (MOS Doc ID 1105635.1) for exact platform support before proceeding. The OMS host must be a 64-bit operating system.

Required OS packages (Oracle Linux / RHEL example):

```bash
# Check and install required packages
yum install -y make binutils gcc libaio glibc glibc-devel \
  libstdc++ libstdc++-devel sysstat unzip
```

Set kernel parameters in `/etc/sysctl.conf` per the EM Installation guide. Key parameters include `fs.file-max`, `kernel.sem`, and `net.core` socket buffer settings. Run `sysctl -p` after editing.

### Hardware Sizing

**OMS Host (minimum for production)**

| Resource | Requirement |
|----------|-------------|
| CPU | 4 cores (8+ recommended) |
| RAM | 16 GB (24 GB+ recommended for 100+ targets) |
| /u01 (OMS home) | 50 GB |
| /tmp | 4 GB |
| Swap | Equal to RAM, minimum 16 GB |

**OMR Database Host (minimum for production)**

| Resource | Requirement |
|----------|-------------|
| CPU | 4 cores (8+ recommended) |
| RAM | 16 GB |
| Data filesystem | 100 GB (grows with monitoring retention) |
| REDO logs | 3 × 300 MB log groups recommended |

Consult the OEM 13c Sizing Guide (MOS Doc ID 1353073.1) for target-count-based sizing. The repository can grow rapidly; allocate separate ASM disk groups or dedicated filesystems for data, index, and redo.

### JDK

OEM 13c ships with Oracle JDK 8. Do not install or configure a separate JDK for OMS. The installer places its own JDK under the Middleware Home. Confirm that `JAVA_HOME` is not set in the oracle user's shell profile, or that it does not conflict with the installer-bundled JDK.

### Required Ports

| Component | Default Port | Description |
|-----------|-------------|-------------|
| OMS Console (HTTPS) | 7803 | Cloud Control UI |
| OMS Console (HTTP) | 7788 | Redirects to HTTPS |
| OMS Upload (HTTPS) | 4904 | Agent metric upload |
| OMS Upload (HTTP) | 4889 | Agent metric upload (non-SSL) |
| WebLogic Admin Server | 7102 | AdminServer console |
| Node Manager | 7403 | WebLogic Node Manager |
| OMR Listener | 1521 | Database listener (or custom) |

Ensure these ports are open in host firewalls (`firewalld` or `iptables`) and any network ACLs between OMS, OMR, and agent hosts.

---

## OMR Database Preparation

The OMR must be a dedicated Oracle Database. Do not share it with application workloads. Supported versions for the OMR in OEM 13.5:

- Oracle Database 19c (recommended)
- Oracle Database 12.2

### Required Initialization Parameters

Set these in the OMR `spfile` before running the OEM installer:

```sql
-- Minimum parameter values required by OEM installer
ALTER SYSTEM SET db_block_size = 8192 SCOPE=SPFILE;          -- Must be 8 KB; set at creation
ALTER SYSTEM SET session_cached_cursors = 200 SCOPE=SPFILE;
ALTER SYSTEM SET open_cursors = 300 SCOPE=SPFILE;
ALTER SYSTEM SET sga_target = 2G SCOPE=SPFILE;               -- Adjust to available RAM
ALTER SYSTEM SET pga_aggregate_target = 1G SCOPE=SPFILE;
ALTER SYSTEM SET undo_retention = 900 SCOPE=SPFILE;
ALTER SYSTEM SET log_buffer = 10485760 SCOPE=SPFILE;
ALTER SYSTEM SET shared_pool_size = 600M SCOPE=SPFILE;
ALTER SYSTEM SET statistics_level = TYPICAL SCOPE=SPFILE;
ALTER SYSTEM SET timed_statistics = TRUE SCOPE=SPFILE;
ALTER SYSTEM SET job_queue_processes = 20 SCOPE=SPFILE;
ALTER SYSTEM SET recyclebin = OFF SCOPE=SPFILE;
ALTER SYSTEM SET audit_trail = NONE SCOPE=SPFILE;            -- OEM manages its own audit
ALTER SYSTEM SET enable_ddl_logging = FALSE SCOPE=SPFILE;
```

Restart the OMR database after applying parameter changes.

### Required Tablespaces

The OEM installer creates its schema objects in the following tablespaces. Pre-create them with autoextend enabled, or allow the installer to create them (Advanced Install prompts for datafile paths):

| Tablespace | Initial Size | Purpose |
|------------|-------------|---------|
| MGMT_TABLESPACE | 500 MB | Core repository schema |
| MGMT_ECM_DEPOT_TS | 500 MB | Configuration data |
| MGMT_AD4J_TS | 200 MB | Application diagnostics |

If pre-creating:

```sql
CREATE TABLESPACE MGMT_TABLESPACE
  DATAFILE '/u02/oradata/orcl/mgmt01.dbf' SIZE 500M
  AUTOEXTEND ON NEXT 100M MAXSIZE UNLIMITED
  EXTENT MANAGEMENT LOCAL SEGMENT SPACE MANAGEMENT AUTO;

CREATE TABLESPACE MGMT_ECM_DEPOT_TS
  DATAFILE '/u02/oradata/orcl/mgmt_ecm01.dbf' SIZE 500M
  AUTOEXTEND ON NEXT 100M MAXSIZE UNLIMITED
  EXTENT MANAGEMENT LOCAL SEGMENT SPACE MANAGEMENT AUTO;

CREATE TABLESPACE MGMT_AD4J_TS
  DATAFILE '/u02/oradata/orcl/mgmt_ad4j01.dbf' SIZE 200M
  AUTOEXTEND ON NEXT 50M MAXSIZE UNLIMITED
  EXTENT MANAGEMENT LOCAL SEGMENT SPACE MANAGEMENT AUTO;
```

### SYSMAN Account

The OEM installer creates the `SYSMAN` schema user in the OMR. Do not create this user manually before running the installer. Ensure the SYS and SYSTEM passwords are available at install time because the installer configures grants and synonyms using privileged access.

---

## Installation Workflow

### 1. Download and Extract

Download `em13500_linux64.bin` from Oracle Software Delivery Cloud (eDelivery) or My Oracle Support. Verify the checksum against the posted SHA-256 digest.

```bash
chmod +x em13500_linux64.bin
./em13500_linux64.bin
```

### 2. Installer Screens — Advanced Install

The graphical wizard or silent install response file covers these stages:

1. **Software Updates** — Skip or configure a local software library patch source.
2. **Prerequisite Checks** — The installer validates OS packages, swap, /tmp, and OS parameters. Resolve all failures before proceeding. Warnings can be acknowledged.
3. **Installation Type** — Select **Advanced**.
4. **Installation Details** — Provide:
   - **Middleware Home** (e.g., `/u01/app/oracle/middleware`) — OMS and WebLogic binaries.
   - **Agent Base Directory** (e.g., `/u01/app/oracle/agent`) — local OMS-host agent.
   - **Host Name** — fully qualified domain name (FQDN) resolvable by all managed hosts.
5. **Configuration Details** — Admin Server, Node Manager, and WebLogic domain passwords.
6. **Database Connection Details** — OMR connect string, SYS password, and sysdba mode.
7. **Management Tablespace** — confirm or override datafile locations.
8. **Deployment Size** — Small (up to 99 targets), Medium (100–999), Large (1000+). This sets OMS heap and thread pool sizes.
9. **Review and Install**.

### 3. Silent Install

Generate a response file during a GUI run or use a pre-filled template:

```bash
./em13500_linux64.bin -silent -responseFile /tmp/em_install.rsp \
  -J-Djava.io.tmpdir=/tmp
```

Key response file parameters (documented in the EM Installation Guide, Appendix B):

```ini
INSTALL_UPDATES_SELECTION=skip
ORACLE_MIDDLEWARE_HOME_LOCATION=/u01/app/oracle/middleware
ORACLE_AGENT_BASE_DIR=/u01/app/oracle/agent
ORACLE_HOSTNAME=omshost.example.com
DEPLOYMENT_SIZE=MEDIUM
DATABASE_HOSTNAME=omrhost.example.com
LISTENER_PORT=1521
SERVICENAME_OR_SID=emrep
SYS_PASSWORD=<SYS_password>
SYSMAN_PASSWORD=<SYSMAN_password>
SYSMAN_CONFIRM_PASSWORD=<SYSMAN_password>
MGMT_TABLESPACE_LOCATION=/u02/oradata/orcl/mgmt01.dbf
ECM_TABLESPACE_LOCATION=/u02/oradata/orcl/mgmt_ecm01.dbf
JVM_DIAGNOSTICS_TABLESPACE_LOCATION=/u02/oradata/orcl/mgmt_ad4j01.dbf
```

Do not store response files containing passwords in shared or version-controlled locations.

### 4. Post-Install Verification

```bash
# Check OMS status
$MIDDLEWARE_HOME/bin/emctl status oms

# Check repository connectivity
$MIDDLEWARE_HOME/bin/emctl status oms -details

# Verify OMS processes (OMS, WebLogic Admin Server, Node Manager)
$MIDDLEWARE_HOME/bin/emctl status oms -bip_only
```

Access the Cloud Control UI at `https://<OMS_FQDN>:7803/em` and log in as `sysman`.

---

## Post-Installation Configuration

### WebLogic Domain and Admin Server

The OEM installer creates a WebLogic domain named `GCDomain` under `$MIDDLEWARE_HOME/gc_inst/`. The Admin Server listens on port 7102 by default.

Access the WebLogic Administration Console at `https://<OMS_FQDN>:7102/console`.

The `weblogic` password is set during installation. The Admin Server manages two managed servers: `EMGC_OMS1` (OMS application) and `EMGC_BIP1` (BI Publisher).

Node Manager runs as a separate process and is required to start/stop managed servers remotely. It is configured at `$MIDDLEWARE_HOME/gc_inst/NodeManager/`.

Start and stop the full OMS stack in order:

```bash
# Start: Node Manager first, then OMS
$MIDDLEWARE_HOME/bin/emctl start oms

# Stop: OMS first, then Node Manager
$MIDDLEWARE_HOME/bin/emctl stop oms
```

### SSL Configuration

OEM 13c uses self-signed certificates by default. Replace with CA-signed certificates for production.

To upload a new OMS certificate:

```bash
# Stop OMS before modifying keystores
$MIDDLEWARE_HOME/bin/emctl stop oms

# Use orapki to manage the OMS keystore
$ORACLE_HOME/bin/orapki wallet add \
  -wallet $MIDDLEWARE_HOME/gc_inst/WebTierIH1/config/OHS/ohs1/keystores/default \
  -trusted_cert -cert /path/to/ca-bundle.crt -auto_login_only

# Import the signed OMS certificate
$ORACLE_HOME/bin/orapki wallet add \
  -wallet $MIDDLEWARE_HOME/gc_inst/WebTierIH1/config/OHS/ohs1/keystores/default \
  -user_cert -cert /path/to/oms-signed.crt -auto_login_only

$MIDDLEWARE_HOME/bin/emctl start oms
```

After replacing the OMS certificate, resecure all agents to pick up the new certificate hash:

```bash
$AGENT_HOME/bin/emctl secure agent -reg_passwd <sysman_password>
```

### Securing the OMS

The OMS upload port uses Oracle Secure Sockets Layer (OSSL). Secure mode is enabled by default after installation. Verify:

```bash
$MIDDLEWARE_HOME/bin/emctl secure status oms
```

To enable HTTPS for the upload port explicitly:

```bash
$MIDDLEWARE_HOME/bin/emctl secure oms -sysman_pwd <password> \
  -reg_pwd <agent_registration_password>
```

---

## Agent Deployment

Agents must be deployed to every host to be monitored. Two primary methods are supported.

### Push-Based Deployment (AgentDeploy)

The OMS pushes the agent to remote hosts using SSH. Prerequisites:

- SSH connectivity from the OMS host to the target host as the oracle OS user (password or key-based).
- The oracle user exists on the target host with a writable home directory.
- The target host satisfies platform prerequisites (architecture, OS packages).

From the Cloud Control UI:

1. Navigate to **Setup > Add Target > Add Targets Manually**.
2. Select **Install Agent on Host**.
3. Provide the target hostname, OS platform, and installation base directory.
4. Supply SSH credentials (Named Credential or one-time entry).
5. The OMS stages the agent bundle, runs `agentDeploy.sh` on the remote host, and registers the new agent.

From the command line using `agentDeploy.sh` directly (after staging):

```bash
# On the target host, after staging the agent bundle from OMS
/tmp/agent_staging/agentDeploy.sh \
  AGENT_BASE_DIR=/u01/app/oracle/agent \
  RESPONSE_FILE=/tmp/agent.rsp
```

Minimum `agent.rsp` contents:

```ini
OMS_HOST=omshost.example.com
EM_UPLOAD_PORT=4904
AGENT_REGISTRATION_PASSWORD=<registration_password>
```

### Pull-Based Deployment (Agent Gold Image)

Gold Image deployment allows agents to self-install from a versioned image stored in OMS. This is recommended for large-scale rollouts and ensures all agents run identical software versions.

**Create a Gold Image:**

1. Promote an existing, patched agent to Gold Image status:
   - Navigate to **Setup > Manage Cloud Control > Gold Agent Images**.
   - Select an existing agent, click **Create Gold Image**.
   - Provide an image name and version.

2. Set the Gold Image as the Active Version.

**Deploy from Gold Image (Cloud Control UI):**

1. Navigate to **Setup > Add Target > Add Targets Manually > Install Agent on Host**.
2. Select the Gold Image version in the deployment wizard.

**Deploy from Gold Image (command line — agent pull):**

On the target host, download the agent bootstrap script from OMS and run it:

```bash
# Download bootstrap script (adjust port and OMS FQDN)
curl -k https://omshost.example.com:4904/em/install/getAgentImage \
  -o agentimage.zip

unzip agentimage.zip

./agentDeploy.sh \
  AGENT_BASE_DIR=/u01/app/oracle/agent \
  OMS_HOST=omshost.example.com \
  EM_UPLOAD_PORT=4904 \
  AGENT_REGISTRATION_PASSWORD=<registration_password>
```

After deployment, verify the agent is up and communicating:

```bash
# On the managed host
$AGENT_HOME/bin/emctl status agent
$AGENT_HOME/bin/emctl upload agent
```

---

## Adding Targets

### Database Target

1. Navigate to **Setup > Add Target > Add Targets Manually**.
2. Select **Add Using Guided Process > Oracle Database, Listener, and Automatic Storage Management**.
3. Specify the monitoring host (where the agent is running), then search for discovered databases.
4. Provide a monitoring credential (a database user with at least the `SELECT_CATALOG_ROLE` and `CREATE SESSION` privileges). The recommended dedicated monitoring user:

```sql
CREATE USER dbsnmp IDENTIFIED BY "<password>"
  DEFAULT TABLESPACE USERS TEMPORARY TABLESPACE TEMP;

GRANT CREATE SESSION TO dbsnmp;
GRANT SELECT_CATALOG_ROLE TO dbsnmp;
GRANT EXECUTE ON sys.lc_replication_session_api TO dbsnmp; -- if applicable
```

The `DBSNMP` account is created by default in Oracle Database and is the conventional monitoring user for OEM.

### Host Target

Host targets are added automatically when an agent is deployed. If a host is not showing:

```bash
# On the managed host
$AGENT_HOME/bin/emctl config agent listtargets
```

Confirm the host and `oracle_emd` target types are present.

### WebLogic Domain Target

1. Navigate to **Setup > Add Target > Add Targets Manually**.
2. Select **Add Using Guided Process > Oracle WebLogic Domain**.
3. Provide the Admin Server host, port, and WebLogic credentials.
4. OEM uses the WebLogic JMX MBean API to discover servers, data sources, and applications within the domain.

The agent on the Admin Server host must be running and must be able to reach the Admin Server port.

### Middleware Auto-Discovery

OEM can discover Oracle Fusion Middleware components (WebLogic domains, SOA composites, OHS instances) automatically by scanning a host:

1. Navigate to **Setup > Add Target > Configure Auto Discovery**.
2. Select the hosts to scan.
3. OEM schedules a discovery job via the host agent. Results appear under **Setup > Add Target > Auto Discovery Results**.

Review discovered targets and promote them to monitored targets individually or in bulk.

---

## Enterprise User Security (SSO / LDAP Integration)

OEM supports LDAP-based authentication (Oracle Internet Directory, Microsoft Active Directory, or any LDAP v3 provider) for single sign-on and centralized user management.

### Configuring LDAP Authentication

1. Navigate to **Setup > Security > Enterprise User Security**.
2. Select **LDAP Directory**.
3. Provide:
   - LDAP host and port.
   - Base DN for user and group search.
   - A bind DN with search privileges.
   - SSL setting (strongly recommended for production).
4. Map LDAP groups to OEM roles (e.g., map an AD group to the `EM_ALL_OPERATOR` OEM role).

After configuration, users authenticate against the external LDAP directory. OEM does not synchronize passwords; it performs a real-time LDAP bind per login.

### Oracle Access Manager (OAM) SSO

OEM 13c supports SSO via Oracle Access Manager. Configuration requires:

- OAM 11g or 12c deployment.
- Registration of OEM as a protected resource in OAM.
- Configuration of the OEM WebLogic domain to use OAM WebGate or SAML assertions.

Refer to MOS Doc ID 1569211.1 for the OAM-OEM integration procedure.

---

## High Availability

A production OEM deployment uses at least two OMS instances behind a load balancer and a highly available OMR database (Oracle RAC or Data Guard).

### Architecture

```
Clients
  |
[Software Load Balancer]  (e.g., Oracle Traffic Director, F5, nginx)
  |               |
[OMS1]          [OMS2]        -- Active/Active OMS pair
  |               |
[Shared Filesystem]           -- /gc_inst shared via NFS or ACFS
  |               |
         [OMR — Oracle RAC or Active Data Guard]
```

### Shared Filesystem Requirements

Both OMS instances must share:

- `$MIDDLEWARE_HOME/gc_inst/` — configuration, certificate keystores, and WebLogic domain.
- The OEM software library (configured under **Setup > Provisioning and Patching > Software Library**).

Use NFS, Oracle ACFS, or another shared POSIX filesystem. Mount with `sync,noac` options on NFS to prevent stale reads between OMS nodes.

### Software Load Balancer Configuration

Set the SLB virtual hostname and ports in OEM after installing the first OMS:

```bash
# Configure OMS to use the SLB hostname
$MIDDLEWARE_HOME/bin/emctl config oms -store_repos_details \
  -repos_host omrhost.example.com -repos_port 1521 \
  -repos_sid emrep -repos_user sysman -repos_pwd <password>

# Set the SLB virtual server name
$MIDDLEWARE_HOME/bin/emctl config oms -set_slb \
  -slb_console_host slb.example.com -slb_console_port 443 \
  -slb_upload_host slb.example.com -slb_upload_port 4904 \
  -sysman_pwd <password>
```

After setting the SLB, agents and browsers connect to `slb.example.com`. Reconfigure agents to use the SLB upload address:

```bash
$AGENT_HOME/bin/emctl secure agent -emdWalletSrcUrl \
  https://slb.example.com:4904/em
```

### Adding a Second OMS (Software Only Install)

1. Run the installer in **Software Only** mode on the second OMS host, pointing to the same shared Middleware Home or a replicated home.
2. Run the OMS Configuration Assistant (`omsca`) to attach the new OMS to the existing OMR:

```bash
$MIDDLEWARE_HOME/oui/bin/runConfig.sh \
  ORACLE_HOME=$MIDDLEWARE_HOME \
  ACTION=Configure MODE=perform \
  COMPONENT_XML={encap_oms.1_0_0_0_0.xml}
```

The full documented procedure is in the OEM Advanced Installation Guide, Chapter 5 (Installing Additional OMS).

### OMR High Availability

- **Oracle RAC**: Point the OEM installer at the RAC service name (not an individual instance SID).
- **Active Data Guard**: Use the primary database for the OMR and configure OEM to reconnect automatically after a failover using an Oracle Net TNS alias that resolves to both primary and standby via SCAN.

---

## Common Installation Errors

### Prerequisite Check: Insufficient /tmp Space

**Symptom**: Installer fails with `PRVG-1620` or a /tmp space warning.

**Resolution**: Ensure at least 4 GB free in /tmp, or redirect the temporary directory:

```bash
./em13500_linux64.bin -J-Djava.io.tmpdir=/u01/tmp
```

### OMR Connection Failure During Install

**Symptom**: `INS-32025: The chosen installation conflicts with software already installed` or `ORA-12541: TNS:no listener`.

**Resolution**: Verify the OMR listener is running and the connect string resolves from the OMS host:

```bash
tnsping <OMR_SERVICE_OR_SID>
sqlplus sys/<password>@<OMR_CONNECT_STRING> as sysdba
```

### OMS Start Failure: Admin Server Port Conflict

**Symptom**: `emctl start oms` reports the Admin Server failed to start; port 7102 already in use.

**Resolution**: Check for a stale WebLogic process or port conflict:

```bash
ss -tlnp | grep 7102
# Kill stale process if confirmed, then retry emctl start oms
```

### Agent Upload Failures After Installation

**Symptom**: Agent shows `Agent unreachable` or `blocked` status in Cloud Control.

**Resolution**:

1. Verify the agent can reach the OMS upload port:
   ```bash
   curl -k https://omshost.example.com:4904/empbs/upload
   # Expected: HTTP 200 or a redirect, not a connection timeout
   ```
2. Resecure the agent:
   ```bash
   $AGENT_HOME/bin/emctl secure agent -reg_passwd <password>
   $AGENT_HOME/bin/emctl upload agent
   ```

### Repository Configuration Fails: Tablespace Size

**Symptom**: Installer exits with `ORA-01659: unable to allocate MINEXTENTS` during schema creation.

**Resolution**: Expand the existing tablespace datafile or pre-create tablespaces with a larger initial allocation (at least 500 MB each) and `AUTOEXTEND ON`.

### Installer Hangs on "Configuring BI Publisher"

**Symptom**: Progress bar stalls at the BI Publisher configuration step for more than 30 minutes.

**Resolution**: Check for `Caused by: java.net.UnknownHostException` in `$MIDDLEWARE_HOME/cfgtoollogs/bip/`. The OMS host FQDN must resolve to itself via forward and reverse DNS. Add an entry to `/etc/hosts` if DNS is not configured:

```
192.0.2.10   omshost.example.com omshost
```

---

## Sources

- Oracle Enterprise Manager Cloud Control Basic Installation Guide 13c Release 5 — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emadv/index.html
- Oracle Enterprise Manager Cloud Control Advanced Installation and Configuration Guide 13c Release 5 — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/embsc/index.html
- Oracle Enterprise Manager Cloud Control Administrator's Guide 13c Release 5 — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emadm/index.html
- Oracle Enterprise Manager Cloud Control Security Guide 13c Release 5 — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emsec/index.html
- Oracle Enterprise Manager Cloud Control Upgrade Guide 13c Release 5 — https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/emupg/index.html
- MOS Doc ID 1105635.1 — Enterprise Manager Cloud Control Certification Checker
- MOS Doc ID 1353073.1 — Enterprise Manager 13c Sizing Guide
- MOS Doc ID 1569211.1 — How to Configure OAM SSO with Enterprise Manager 13c
