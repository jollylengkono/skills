---
name: weblogic-installation-and-configuration
description: Oracle WebLogic Server installation and configuration guidance for 12.2.1.4 and 14.1.1 — silent install, domain creation, JDBC data sources, JMS, Node Manager, and post-install validation.
---

# WebLogic Server — Installation and Configuration

## Overview

Use this skill for source-backed guidance on installing Oracle WebLogic Server and configuring domains, data sources, and supporting services. Covers WLS 12.2.1.4 (the current 12c release) and 14.1.1 (the current 14c release) with notes on differences between them.

Baseline version: **WebLogic Server 14.1.1**. Differences for 12.2.1.4 are called out explicitly.

## Core Concepts

- WebLogic uses an **Oracle Home** (software binaries) and a **Domain Home** (runtime configuration). These must be kept separate.
- Domain topology: one **Administration Server** + one or more **Managed Servers**, optionally grouped into **Clusters**.
- **Node Manager** is required for remote lifecycle management (start/stop/restart Managed Servers from Admin Console or WLST).
- **WLST (WebLogic Scripting Tool)** supports both online (connected) and offline (domain config only) modes.
- Post-install, always validate Admin Server startup before deploying any Managed Server or application.

## Prerequisites

### Java Requirements

| WLS Version | Required JDK |
|-------------|--------------|
| 14.1.1      | Oracle JDK 11 (minimum), JDK 17 recommended |
| 12.2.1.4    | Oracle JDK 8 (1.8.0_191+) or JDK 11 |

Set `JAVA_HOME` before running any installer or WLST script:

```bash
export JAVA_HOME=/opt/java/jdk-17
export PATH=$JAVA_HOME/bin:$PATH
java -version
```

### Supported Platforms

Verify the current certification matrix at https://www.oracle.com/middleware/technologies/fusion-certification.html before installation. Common certified platforms for WLS 14.1.1: Oracle Linux 7/8, RHEL 7/8, Windows Server 2016/2019.

### System Requirements

- Minimum 1 GB RAM for Admin Server (2+ GB recommended for production domains)
- Minimum 1.5 GB disk space for Oracle Home
- `/tmp` must have at least 500 MB available during install

## Installation

### GUI Mode

```bash
java -jar fmw_14.1.1.0.0_wls.jar
```

Follow the installer wizard. Select **WebLogic Server** install type. Use a dedicated OS user (e.g., `oracle`), not root.

### Silent Mode

Create a response file `wls_silent.rsp`:

```properties
[ENGINE]
Response File Version=1.0.0.0.0

[GENERIC]
ORACLE_HOME=/opt/oracle/middleware/wls14
INSTALL_TYPE=WebLogic Server
MYORACLESUPPORT_USERNAME=
MYORACLESUPPORT_PASSWORD=
DECLINE_SECURITY_UPDATES=true
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false
```

Run silent install:

```bash
java -jar fmw_14.1.1.0.0_wls.jar -silent -responseFile /path/to/wls_silent.rsp \
  -invPtrLoc /etc/oraInst.loc
```

Verify completion:

```bash
cat /opt/oracle/middleware/wls14/inventory/registry.xml | grep -i version
```

## Domain Creation

### GUI Mode (Configuration Wizard)

```bash
$ORACLE_HOME/oracle_common/common/bin/config.sh
```

1. Select **Create a new domain**.
2. Choose **Basic WebLogic Server Domain** template.
3. Set domain name, location, Admin Server listen address/port.
4. Set admin credentials.
5. Configure Node Manager (see below).
6. Review and create.

### Silent Mode (WLST Offline)

```python
# create_domain.py
readTemplate("/opt/oracle/middleware/wls14/wlserver/common/templates/wls/wls.jar")

cd('Servers/AdminServer')
set('ListenAddress', '')
set('ListenPort', 7001)

cd('/')
cd('Security/base_domain/User/weblogic')
cmo.setPassword('WelcomeWLS14!')

setOption('DomainName', 'base_domain')
writeDomain('/opt/oracle/user_projects/domains/base_domain')
closeTemplate()
```

Run:

```bash
$ORACLE_HOME/oracle_common/common/bin/wlst.sh create_domain.py
```

## JDBC Data Source Configuration

Configure via Admin Console (**Services > Data Sources > New**) or WLST online:

```python
# wlst_datasource.py — run connected to running Admin Server
connect('weblogic', 'WelcomeWLS14!', 't3://localhost:7001')
edit()
startEdit()

cd('/')
cmo.createJDBCSystemResource('MyDS')
cd('JDBCSystemResources/MyDS/JDBCResource/MyDS')
cmo.setName('MyDS')

cd('JDBCSystemResources/MyDS/JDBCResource/MyDS/JDBCDriverParams/MyDS')
set('DriverName', 'oracle.jdbc.OracleDriver')
set('URL', 'jdbc:oracle:thin:@//dbhost:1521/pdbname')
set('PasswordEncrypted', 'dbpassword')

cd('JDBCSystemResources/MyDS/JDBCResource/MyDS/JDBCConnectionPoolParams/MyDS')
set('MinCapacity', 5)
set('MaxCapacity', 30)
set('TestTableName', 'SQL SELECT 1 FROM DUAL')

activate()
```

### Key Connection Pool Parameters

| Parameter | Recommended Default | Notes |
|-----------|---------------------|-------|
| `MinCapacity` | 5–10 | Pre-warmed connections |
| `MaxCapacity` | 20–50 | Match DB session limit / number of managed servers |
| `ConnectionCreationRetryFrequencySeconds` | 10 | Recovery interval |
| `TestTableName` | `SQL SELECT 1 FROM DUAL` | Oracle-specific health check |
| `TestConnectionsOnReserve` | true | Validates connections before app use |
| `SecondsToTrustAnIdlePoolConnection` | 10 | Skip test for recently-used connections |

## Node Manager Setup

Node Manager is required for remote Managed Server lifecycle management.

Configure during domain creation (recommended) or manually:

```bash
# Start Node Manager (Unix domain sockets or plain sockets)
$ORACLE_HOME/oracle_common/common/bin/wlst.sh

wls:/offline> nmGenBootStartupProps('/opt/oracle/user_projects/domains/base_domain')
wls:/offline> exit()

nohup $DOMAIN_HOME/bin/startNodeManager.sh &
```

Enroll the domain with Node Manager:

```python
# enroll_nm.py
connect('weblogic', 'WelcomeWLS14!', 't3://adminhost:7001')
nmEnroll('/opt/oracle/user_projects/domains/base_domain',
         '/opt/oracle/middleware/wls14/oracle_common/common/nodemanager')
```

## Starting and Stopping

### Admin Server

```bash
# Start
$DOMAIN_HOME/bin/startWebLogic.sh

# Stop (graceful)
$DOMAIN_HOME/bin/stopWebLogic.sh
```

### Managed Servers (via Node Manager)

```python
# via WLST
connect('weblogic', 'password', 't3://adminhost:7001')
nmConnect('weblogic', 'password', 'adminhost', '5556',
          'base_domain', '/opt/oracle/user_projects/domains/base_domain')
nmStart('ManagedServer1')
```

## Post-Install Validation

1. **Admin Console** — `http://adminhost:7001/console` — log in with admin credentials.
2. **Server state** — Admin Console > **Environment > Servers**; all servers should show `RUNNING`.
3. **WLST connectivity**:
   ```python
   connect('weblogic', 'password', 't3://adminhost:7001')
   state('base_domain', 'Domain')
   ```
4. **JDBC test** — Admin Console > **Services > Data Sources > MyDS > Monitoring > Testing** > click Test Data Source.
5. **Server log** — review `$DOMAIN_HOME/servers/AdminServer/logs/AdminServer.log` for `<BEA-000365>` (Server state changed to RUNNING).

## Best Practices and Common Mistakes

- **Do not install as root.** Use a dedicated OS user with no `sudo` access to system directories.
- **Separate Oracle Home from Domain Home.** Never place domain directories inside `$ORACLE_HOME`.
- **Use Node Manager.** Direct `startManagedWebLogic.sh` is acceptable for development only; production requires Node Manager for crash recovery.
- **Set `JAVA_HOME` before any WLS script.** WLS picks up the wrong JDK silently if `JAVA_HOME` is unset.
- **Do not share Admin Server listen ports** with Managed Servers. Use distinct ports (e.g., 7001 for Admin, 8001+ for Managed).
- **Enable SSL on Admin Server** before production use. Plain HTTP admin port should be disabled or firewalled.
- **Validate data source health-check SQL.** `SQL SELECT 1 FROM DUAL` is Oracle-specific — not portable but correct for Oracle DB targets.
- **Back up the domain config directory** (`$DOMAIN_HOME/config/`) before any patch or schema change.

## Oracle Version Notes (19c vs 26ai)

This section covers WLS version differences, not Oracle Database versions:

| Capability | WLS 12.2.1.4 | WLS 14.1.1 |
|------------|-------------|------------|
| Certified Java | JDK 8, JDK 11 | JDK 11, JDK 17 (JDK 8 dropped) |
| Jakarta EE | Java EE 8 | Jakarta EE 8 (namespace migration required for Jakarta EE 9+) |
| Minimum Linux | OL 6 (EOL), OL 7 | OL 7, OL 8 |
| Deployment model | Same | Same |
| Config Wizard | `config.sh` | `config.sh` (same binary path) |

For Oracle Database target version compatibility (e.g., which JDBC driver to bundle):
- Use Oracle JDBC Thin driver from the matching database release (`ojdbc11.jar` for JDK 11+, `ojdbc8.jar` for JDK 8).
- JDBC driver is backward-compatible: an Oracle DB 19c JDBC driver can connect to Oracle DB 19c targets from both WLS 12c and 14c.

## Sources

- WLS 14.1.1 Installation Guide: https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/14.1.1/install/
- WLS 12.2.1.4 Installation Guide: https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/12.2.1.4/install/
- WLS 14.1.1 Domain Configuration: https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/14.1.1/domcf/
- WLS JDBC Configuration: https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/14.1.1/jdbca/
- Node Manager Administration: https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/14.1.1/nodem/
- WLST Reference: https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/14.1.1/wlstg/
- Fusion Middleware Certification Matrix: https://www.oracle.com/middleware/technologies/fusion-certification.html
