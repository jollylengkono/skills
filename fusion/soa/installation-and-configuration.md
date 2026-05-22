# Oracle SOA Suite 12c — Installation and Configuration

## Overview

Use this skill to plan and execute an Oracle SOA Suite 12.2.1.4 installation, including Repository Creation Utility (RCU) schema setup, WebLogic domain creation via the Configuration Wizard, MDS repository configuration, Oracle Service Bus (OSB) domain setup, adapter configuration, and post-install validation. The baseline is SOA Suite 12.2.1.4. Differences from SOA Suite 11g (11.1.1.x) are called out in the version notes section.

Key components installed together:

- **Oracle WebLogic Server** — the application server runtime hosting SOA managed servers.
- **Oracle SOA Suite** — BPEL Process Manager, Oracle Mediator, Oracle BPM Suite, Oracle B2B.
- **Oracle Service Bus (OSB)** — message routing and transformation layer (separate install option or collocated domain).
- **Oracle Metadata Services (MDS)** — shared metadata repository for composites and OSB artifacts.
- **SOAINFRA schema** — Oracle Database schema storing BPEL instance state, Mediator routing data, and B2B runtime data.

---

## Prerequisites

### Java Development Kit

SOA Suite 12.2.1.4 requires a certified JDK. Use Oracle JDK 8 (Update 191 or later) or Oracle JDK 11 for 12.2.1.4 only when the patch set explicitly certifies JDK 11. Do not use OpenJDK for production deployments unless Oracle certification confirms support.

Verify the JDK before running any installer:

```bash
java -version
# Expected: java version "1.8.0_xxx" (JDK 8) or "11.x.x" (JDK 11, if certified)
export JAVA_HOME=/u01/jdk/jdk1.8.0_xxx
export PATH=$JAVA_HOME/bin:$PATH
```

### Operating System

SOA Suite 12.2.1.4 is certified on:

- Oracle Linux 6.x / 7.x (x86-64)
- Red Hat Enterprise Linux 6.x / 7.x (x86-64)

Verify OS prerequisites and exact certified configurations using the Oracle Fusion Middleware Certification Matrix:
https://www.oracle.com/middleware/technologies/fusion-certification.html

Required OS packages (Oracle Linux / RHEL example):

```bash
yum install -y binutils compat-libcap1 compat-libstdc++-33 gcc gcc-c++ \
  glibc glibc-devel ksh libaio libaio-devel libgcc libstdc++ libstdc++-devel \
  libXext libXtst libX11 libXau libxcb libXi make sysstat unzip
```

### Hardware Sizing (Production Minimum)

| Resource | SOA Managed Server | Admin Server |
|---|---|---|
| CPU | 4 cores | 2 cores |
| Heap (JVM -Xmx) | 4–8 GB | 1–2 GB |
| Disk (domain) | 20 GB | 10 GB |
| /tmp | 4 GB | 4 GB |

Adjust heap based on BPEL composite count, message payload sizes, and MDS artifact volume.

### Database for SOAINFRA and MDS Schemas

SOA Suite 12.2.1.4 requires an Oracle Database to host the RCU-created schemas. Supported database versions for 12.2.1.4 include Oracle Database 12.1, 12.2, and 19c. Oracle Database 19c is the recommended baseline.

Required database parameters (set before running RCU):

```sql
-- Minimum values required by SOA Suite RCU
ALTER SYSTEM SET open_cursors = 500 SCOPE=SPFILE;
ALTER SYSTEM SET processes = 300 SCOPE=SPFILE;  -- increase for production
ALTER SYSTEM SET session_cached_cursors = 200 SCOPE=SPFILE;
```

Restart the database instance after changing `processes` and `open_cursors` if using SPFILE.

---

## Step 1 — Download Installers

Download the following components from Oracle Software Delivery Cloud (eDelivery) or Oracle Technology Network (OTN):

1. Oracle WebLogic Server 12.2.1.4 (generic installer: `fmw_12.2.1.4.0_wls.jar`)
2. Oracle SOA Suite and Business Process Management 12.2.1.4 (generic installer: `fmw_12.2.1.4.0_soa.jar`)
3. Oracle Service Bus 12.2.1.4 (generic installer: `fmw_12.2.1.4.0_osb.jar`) — required if configuring an OSB domain

Verify SHA-256 checksums against the values posted on Oracle's download page before running any installer.

---

## Step 2 — Install Oracle WebLogic Server

WebLogic Server must be installed first into the Oracle Home (Middleware Home) before running the SOA Suite or OSB installers.

```bash
export JAVA_HOME=/u01/jdk/jdk1.8.0_xxx
java -jar fmw_12.2.1.4.0_wls.jar
```

In the graphical installer:

1. **Welcome** — click Next.
2. **Auto Updates** — skip or configure a local patch source.
3. **Installation Location** — set the Oracle Home (e.g., `/u01/oracle/middleware/`).
4. **Installation Type** — select **Fusion Middleware Infrastructure** (includes WebLogic Server and JRF). Do not select WebLogic Server only, as SOA Suite requires JRF support libraries.
5. **Prerequisite Checks** — resolve all failures before proceeding.
6. **Installation Summary** — review and click Install.

Silent install using a response file (`wls_install.rsp`):

```bash
java -jar fmw_12.2.1.4.0_wls.jar -silent \
  -responseFile /tmp/wls_install.rsp \
  -invPtrLoc /etc/oraInst.loc
```

Key response file parameters:

```ini
[ENGINE]
Response File Version=1.0.0.0.0

[GENERIC]
ORACLE_HOME=/u01/oracle/middleware
INSTALL_TYPE=Fusion Middleware Infrastructure
```

---

## Step 3 — Install Oracle SOA Suite

Run the SOA Suite installer against the same Oracle Home:

```bash
java -jar fmw_12.2.1.4.0_soa.jar
```

Installation type options:

| Type | Use |
|---|---|
| SOA Suite | Installs BPEL, Mediator, BPM, B2B, MFT components |
| Business Intelligence and SOA | Adds BPM Analytics |
| SOA Suite with BPM | BPEL + BPM without full BI |

For most deployments, select **SOA Suite**.

Set the Oracle Home to the same location used for WebLogic (`/u01/oracle/middleware/`). The installer adds SOA libraries and templates on top of the existing WebLogic installation.

Optionally install OSB into the same Oracle Home:

```bash
java -jar fmw_12.2.1.4.0_osb.jar
# Oracle Home: /u01/oracle/middleware/
```

---

## Step 4 — RCU Schema Creation

The Repository Creation Utility (RCU) creates and initializes the database schemas required by SOA Suite. RCU is located at `$ORACLE_HOME/oracle_common/bin/rcu`.

### Schemas Created for SOA Suite

| Schema Prefix + Suffix | Purpose |
|---|---|
| `<prefix>_STB` | Service Table — required for all domains |
| `<prefix>_OPSS` | Oracle Platform Security Services |
| `<prefix>_IAU` | Audit Services |
| `<prefix>_IAU_APPEND` | Audit (append) |
| `<prefix>_IAU_VIEWER` | Audit (viewer) |
| `<prefix>_MDS` | Oracle Metadata Services (shared artifacts) |
| `<prefix>_UMS` | User Messaging Service |
| `<prefix>_SOAINFRA` | SOA Infrastructure (BPEL, Mediator, B2B runtime) |

Add `<prefix>_ORASDPM` if deploying User Messaging Service.

### Running RCU

```bash
$ORACLE_HOME/oracle_common/bin/rcu
```

RCU wizard steps:

1. **Welcome** — click Next.
2. **Create Repository** — select **Create Repository > System Load and Product Load**.
3. **Database Connection Details** — provide:
   - Database Type: Oracle Database
   - Host Name: `<db_host>`
   - Port: `1521` (or custom listener port)
   - Service Name: `<db_service_name>`
   - Username: `SYS`
   - Role: SYSDBA
4. **Select Components** — choose a schema prefix (e.g., `DEV`). Select:
   - Oracle Platform Security Services
   - Audit Services, Audit Services Append, Audit Services Viewer
   - Oracle Metadata Services
   - SOA Infrastructure
   - User Messaging Service
   - Service Table (auto-selected)
5. **Schema Passwords** — set passwords for each schema or use a single password for all.
6. **Map Tablespaces** — accept defaults or map to pre-created tablespaces. Recommended: place `<prefix>_SOAINFRA` data on a dedicated tablespace with `AUTOEXTEND ON`.
7. **Summary** — review and click Create.
8. **Completion Summary** — verify all components show success before closing.

### Silent RCU

```bash
$ORACLE_HOME/oracle_common/bin/rcu -silent \
  -createRepository \
  -connectString "<db_host>:1521/<service_name>" \
  -dbUser SYS -dbRole SYSDBA \
  -schemaPrefix DEV \
  -component STB \
  -component OPSS \
  -component IAU -component IAU_APPEND -component IAU_VIEWER \
  -component MDS \
  -component UMS \
  -component SOAINFRA \
  < /tmp/rcu_passwords.txt
```

The password input file (`rcu_passwords.txt`) must contain one password per line: first the SYS/SYSDBA password, then the schema passwords in the order listed by `-component` flags.

---

## Step 5 — Domain Creation with Configuration Wizard

The Configuration Wizard creates the WebLogic domain that hosts the SOA Managed Servers.

```bash
$ORACLE_HOME/oracle_common/common/bin/config.sh
```

### Wizard Steps

**1. Configuration Type** — Select **Create a new domain**. Set the domain location (e.g., `/u01/oracle/domains/soaDomain`).

**2. Templates** — Select the product templates to configure. For a full SOA + OSB domain:

| Template | Required For |
|---|---|
| Oracle SOA Suite | BPEL, Mediator, BPM |
| Oracle Service Bus | OSB proxy and business services |
| Oracle Enterprise Manager — Fusion Middleware Control | EM-based management |
| Oracle JRF (auto-selected) | JRF libraries |
| WebLogic Coherence Cluster Extension | Coherence-based SOA cluster |

**3. Application Location** — leave at default or set to a separate application directory.

**4. Administrator Account** — set WebLogic admin username and password.

**5. Domain Mode and JDK** — select **Production** for production environments. Confirm the JDK path.

**6. Database Configuration Type** — select **RCU Data**. Provide:
   - DBMS / Service: `<db_service_name>`
   - Host: `<db_host>`
   - Port: `1521`
   - Schema Owner: `<prefix>_STB` (Service Table schema)
   - Schema Password: password set in RCU

Click **Get RCU Configuration**. The wizard validates the schemas and auto-populates the remaining data source fields.

**7. JDBC Component Schema** — review the auto-populated data sources. Verify that `SOADataSource`, `OraSDPMDataSource`, and `mds-soa` data sources all point to the correct schemas and database host.

**8. JDBC Component Schema Test** — test all connections before proceeding.

**9. Keystore** — leave defaults unless a custom keystore is required.

**10. Advanced Configuration** — select the following for production domains:
   - Administration Server
   - Node Manager
   - Topology (Managed Servers, Clusters, Coherence)

**11. Administration Server** — set listen address and port (default: `7001`). Use the host FQDN or `0.0.0.0` for all interfaces.

**12. Managed Servers** — the wizard creates `soa_server1` (SOA) and `osb_server1` (OSB, if template was selected). Set:
   - Listen Address: host FQDN or `0.0.0.0`
   - Listen Port: `8001` (SOA), `9001` (OSB) — adjust to avoid conflicts

**13. Clusters** — add `soa_cluster` and `osb_cluster` for clustered deployments. Assign managed servers to clusters.

**14. Coherence Cluster** — accept default Coherence cluster settings unless a specific Coherence address is required.

**15. Machines** — add a machine entry for the host running the managed servers. Assign Node Manager connection type (SSL recommended) and listen port (`5556` default).

**16. Virtual Targets** — skip unless using WebLogic Multi-Tenancy (not applicable to standard SOA installs).

**17. Configuration Summary** — review and click Create.

---

## Step 6 — Start the Domain

After domain creation, start the infrastructure in order:

```bash
# 1. Start the Admin Server
$DOMAIN_HOME/startWebLogic.sh

# 2. Start Node Manager (enables managed server start/stop via Admin Console or WLST)
$DOMAIN_HOME/bin/startNodeManager.sh

# 3. Start Managed Servers via Admin Console or WLST
# From WLST:
java weblogic.WLST
connect('weblogic','<password>','t3://adminhost:7001')
start('soa_server1','Server')
start('osb_server1','Server')
exit()
```

Wait for managed servers to reach RUNNING state before proceeding.

---

## Step 7 — MDS Repository Configuration

Oracle Metadata Services (MDS) stores shared artifacts (WSDL files, XSD schemas, XSLT maps) used by SOA composites and OSB. The MDS database repository was created by RCU in the `<prefix>_MDS` schema.

Two MDS partitions are relevant for SOA Suite:

| Partition | Purpose |
|---|---|
| `soa-infra` | SOA composite application artifacts |
| `oracle` | Shared Oracle-provided artifacts (seeded at domain creation) |

Verify the MDS JDBC data source `mds-soa` is active in Enterprise Manager (EM Fusion Middleware Control) or the WebLogic Admin Console under **Services > Data Sources**.

### Import Shared Artifacts into MDS

Use WLST to import shared WSDL or XSD artifacts into MDS:

```python
# WLST offline — run from the SOA Oracle Home
from oracle.mds.lcm.client import MetadataManager

connect('weblogic','<password>','t3://adminhost:7001')
importMetadata(
    application='soa-infra',
    server='soa_server1',
    fromLocation='/tmp/mds_export.zip',
    docs='/**'
)
```

---

## Step 8 — Oracle Service Bus Domain Configuration

OSB is configured within the same WebLogic domain if the OSB template was selected in the Configuration Wizard. The OSB Managed Server (`osb_server1`) hosts the OSB runtime.

After starting `osb_server1`, access the OSB Service Bus Console:

```
http://<host>:9001/servicebus
```

### Creating an OSB Project via Service Bus Console

1. Log in to the Service Bus Console with WebLogic admin credentials.
2. Navigate to **Project Explorer > Create Project**.
3. Provide a project name (e.g., `MyIntegration`).
4. Add resources (Proxy Services, Business Services, Pipeline, WSDL) under the project.
5. Activate the session to commit changes to the runtime.

### OSB Configuration via Oracle JDeveloper / OEPE

For development use, configure OSB resources in Oracle JDeveloper 12c with the Oracle Enterprise Pack for Eclipse (OEPE) and deploy via the WebLogic deployment mechanism or the OSB sbconfig.jar utility.

---

## Step 9 — Adapter Configuration

SOA Suite includes JCA-based adapters for database, file system, JMS, and AQ integration. Adapters are deployed as part of the SOA infrastructure and are configured via the WebLogic Admin Console.

### Database Adapter

The Database Adapter uses a JDBC data source to connect to external databases. Create a dedicated data source in the Admin Console:

1. **Admin Console > Services > Data Sources > New > Generic Data Source**.
2. Set JNDI name: `eis/DB/<MyDataSource>` — the JNDI prefix `eis/DB/` is required for the Database Adapter to discover the connection.
3. Set the database driver, connect string, and credentials.
4. Target the data source to the SOA Managed Server(s).

In Oracle JDeveloper, create a Database Adapter interaction using the **SCA Component Palette > Database Adapter** and reference the JNDI name configured above.

### File Adapter

The File Adapter reads from and writes to file system directories on the SOA Managed Server host. Configure the inbound directory path in the adapter configuration within the composite:

- `PhysicalDirectory` property: absolute path on the managed server host (e.g., `/u01/soa/inbound/`).
- The OS user running the SOA Managed Server process must have read/write access to configured directories.

No JDBC data source is required for the File Adapter.

### JMS Adapter

The JMS Adapter integrates with WebLogic JMS or external JMS providers.

For WebLogic JMS:

1. Create a JMS Module and JMS Queue in the Admin Console.
2. Create a Connection Factory with JNDI name `jms/MyConnectionFactory`.
3. In the JMS Adapter configuration within the composite, reference the connection factory JNDI name and queue JNDI name.

For external JMS providers (e.g., Oracle AQ, IBM MQ), configure a Foreign JMS Server in WebLogic to proxy the remote JMS resources into the local WebLogic JNDI namespace.

---

## Step 10 — Composite Deployment

SOA composites (.SAR files) are deployed to the SOA Infrastructure.

### Deploy via Enterprise Manager

1. Navigate to **Enterprise Manager Fusion Middleware Control > SOA > soa-infra > Deploy**.
2. Upload the composite SAR file.
3. Select the target partition (default: `default`).
4. Click Deploy.

### Deploy via WLST

```python
connect('weblogic','<password>','t3://adminhost:7001')
sca_deployComposite(
    'http://adminhost:7001',
    '/tmp/MyComposite_rev1.0.jar',
    overwrite=True,
    forceDefault=True
)
```

### Deploy via Ant (build pipeline integration)

```xml
<!-- Example Ant target using SOA Ant tasks -->
<target name="deploy-composite">
    <ant antfile="${oracle.home}/bin/ant-sca-deploy.xml"
         target="deploy"
         inheritAll="false">
        <property name="serverURL" value="http://adminhost:7001"/>
        <property name="user" value="weblogic"/>
        <property name="password" value="${wls.password}"/>
        <property name="sarLocation" value="/tmp/MyComposite_rev1.0.jar"/>
        <property name="overwrite" value="true"/>
        <property name="forceDefault" value="true"/>
    </ant>
</target>
```

The `ant-sca-deploy.xml` build file is located at `$ORACLE_HOME/bin/ant-sca-deploy.xml`.

---

## Post-Install Validation

### Enterprise Manager Fusion Middleware Control

Access EM Fusion Middleware Control:

```
http://<adminhost>:7001/em
```

Validate the following after installation:

1. **SOA Infrastructure status**: Navigate to **SOA > soa-infra** — verify the target shows as Up (green).
2. **Managed Server health**: Navigate to **WebLogic Domain > <domain_name>** — all servers should show Running.
3. **Data source state**: Admin Console > Services > Data Sources — all SOA data sources (SOADataSource, mds-soa, OraSDPMDataSource) should show State = Running.
4. **Adapter deployment**: Admin Console > Deployments — `DbAdapter`, `FileAdapter`, `JmsAdapter` should show Active.

### SOA Composer

SOA Composer is a browser-based editor for Oracle Business Rules and Human Task forms included with SOA Suite. Access at:

```
http://<soa_managed_server_host>:8001/soa/composer
```

Log in with WebLogic admin credentials. Verify that the connection to MDS succeeds and that the default composite partitions are visible.

### Verify SOAINFRA Schema Connectivity

Run a basic query against the SOAINFRA schema to confirm schema initialization:

```sql
-- Connect as the SOAINFRA schema user or a DBA
SELECT COUNT(*) FROM <prefix>_soainfra.component_def;
-- Should return 0 or more rows without error
```

### Smoke Test: Deploy a Hello World Composite

Deploy a minimal SOA composite (a BPEL process or Mediator with a single assign/echo) and trigger a test instance via EM > SOA > soa-infra > Flow Instances. Verify the instance completes without fault.

---

## Best Practices and Common Mistakes

### Installation

- Always install into the same Oracle Home in sequence: WebLogic (Fusion Middleware Infrastructure) first, then SOA Suite, then OSB. Running installers in a different order or into separate homes causes deployment failures.
- Do not run the Configuration Wizard until all product installers have completed successfully.
- Avoid spaces and non-ASCII characters in Oracle Home and domain paths.
- Run RCU as a user with SYSDBA privilege to the target database. Do not run RCU as the application schema owner.

### Schema and RCU

- Use a dedicated schema prefix per environment (e.g., `DEV_`, `UAT_`, `PROD_`). Sharing a prefix across environments causes data corruption.
- Pre-create the `SOAINFRA` tablespace with `AUTOEXTEND ON` and place it on fast storage. BPEL dehydration writes every instance state transition to this tablespace.
- Run RCU in a non-production test against a copy of the target database before running it in production.

### Domain Configuration

- Use the FQDN (fully qualified domain name) for all listen addresses. Short hostnames cause SSL certificate validation failures and agent communication problems.
- For production, always configure Node Manager and set managed server startup scripts to use Node Manager. Starting managed servers directly via `startManagedWebLogic.sh` does not support auto-restart.
- Set `PRODUCTION` domain mode. Development mode disables certain security checks and allows auto-deployment from the `autodeploy` directory, which is inappropriate for production.

### MDS

- Shared schemas and WSDL files used across composites must be stored in MDS, not deployed inline per composite. Inline copies create version management problems and increase deployment sizes.
- Back up the MDS repository using `exportMetadata` before patching, redeployment, or domain migration operations.

### Adapters

- Always use the `eis/DB/` JNDI prefix for Database Adapter connection factories. Using a non-standard JNDI path causes adapter endpoint lookup failures at runtime.
- Do not reuse the `SOADataSource` JDBC data source for application adapter connections. Create a separate data source for each external system connected via the Database Adapter.

### Composites

- Deploy composites to a named partition other than `default` to separate environments or application domains within a single SOA infrastructure.
- Always set `forceDefault=true` on initial deployment in a new environment to avoid stale composite version routing.

---

## Oracle Version Notes — SOA Suite 11g vs 12c

This section documents the most significant differences between SOA Suite 11g (11.1.1.x) and SOA Suite 12c (12.2.1.4) that affect installation and configuration.

### Installation Model

| Area | 11g | 12c (12.2.1.4) |
|---|---|---|
| Installer format | `.bin` / GUI-only | `.jar` (generic Java installer) |
| WebLogic version | WebLogic 10.3.x | WebLogic 12.2.1.4 |
| JDK requirement | JDK 6 or 7 | JDK 8 (JDK 11 for specific patch levels) |
| RCU tool | Separate download | Bundled in Oracle Home (`oracle_common/bin/rcu`) |
| Oracle Home layout | Separate MW Home + SOA Home under it | Unified Oracle Home (Fusion Middleware Infrastructure + SOA in one directory) |

### Domain Configuration

- In 11g, the Configuration Wizard creates an `soa_server1` managed server directly associated with the domain. In 12c, templates determine which managed servers and clusters are created, and SOA and OSB servers are separated by default.
- 11g used the `weblogic.management.username` and `weblogic.management.password` Java system properties for start scripts. In 12c, Node Manager handles credential management using a per-domain boot.properties file.

### OSB (Oracle Service Bus)

- In 11g, OSB was named Oracle Service Bus (OSB) and was distributed as a separate product called **Oracle Service Bus** with its own installer. The domain template was applied separately.
- In 12c, OSB is included in the same Oracle Home and is configured via a single Configuration Wizard session, allowing SOA and OSB to coexist in one WebLogic domain.
- The OSB console URL changed: 11g used `http://<host>:<port>/sbconsole`; 12c uses `http://<host>:<port>/servicebus`.

### MDS

- In 11g, MDS could be backed by a file-based store or a database. In 12c, the database-backed MDS repository is required for SOA Suite production deployments.
- The `importMetadata` and `exportMetadata` WLST commands are available in both versions but the MDS partition names differ (`soa-infra` is consistent across versions).

### Schema Names

- 11g used the schema suffix `_SOAINFRA` for the main SOA schema (e.g., `DEV_SOAINFRA`) with a separate `_MDS` schema. The same convention applies in 12c, but 12c adds `_STB` (Service Table) and `_OPSS` schemas that do not exist in 11g.
- 11g RCU was a separate download from OTN. In 12c, RCU is located at `$ORACLE_HOME/oracle_common/bin/rcu`.

### Composite Deployment

- In 11g, composites were deployed using the `ant-sca-deploy.xml` Ant script and `sca_deployComposite` WLST command, both of which remain available in 12c.
- In 12c, composite deployment can also be performed from EM Fusion Middleware Control using an updated deployment wizard that supports partition selection and composite revisions.

---

## Sources

- Oracle SOA Suite 12c (12.2.1.4) Installation Guide: https://docs.oracle.com/en/middleware/soa-suite/soa/12.2.1.4/install-config/index.html
- Oracle Fusion Middleware Installing and Configuring Oracle SOA Suite and Business Process Management: https://docs.oracle.com/en/middleware/soa-suite/soa/12.2.1.4/soaig/index.html
- Oracle Fusion Middleware Repository Creation Utility User's Guide: https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/rcuug/index.html
- Oracle Fusion Middleware Creating WebLogic Domains Using the Configuration Wizard: https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/wldcw/index.html
- Oracle Fusion Middleware Administering Oracle SOA Suite and Oracle BPM Suite: https://docs.oracle.com/en/middleware/soa-suite/soa/12.2.1.4/administer/index.html
- Oracle Service Bus 12c (12.2.1.4) Installation Guide: https://docs.oracle.com/en/middleware/soa-suite/service-bus/12.2.1.4/install-config/index.html
- Oracle Fusion Middleware Installing and Configuring Oracle Service Bus: https://docs.oracle.com/en/middleware/soa-suite/service-bus/12.2.1.4/osbig/index.html
- Oracle Fusion Middleware Developing SOA Applications with Oracle SOA Suite: https://docs.oracle.com/en/middleware/soa-suite/soa/12.2.1.4/develop/index.html
- Oracle Fusion Middleware System Requirements and Specifications: https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/sysrs/index.html
- Oracle Fusion Middleware Certification Matrix: https://www.oracle.com/middleware/technologies/fusion-certification.html
