# Oracle GoldenGate Installation and Configuration

## Overview

This skill covers Oracle GoldenGate installation and initial configuration for Oracle-to-Oracle replication using GoldenGate 21c as the current baseline. It addresses both Classic (non-microservices) and Microservices architecture deployments, GGSCI-based setup versus the Service Manager web UI, supplemental logging on the source Oracle Database, Extract and Replicat process creation, trail file configuration, initial load setup, and connectivity validation.

Use `references/goldengate-source-map.md` to route version-specific questions to official Oracle documentation. Verify the certification matrix before beginning any new installation: https://www.oracle.com/integration/goldengate/certifications/

---

## Architecture Choices: Classic vs. Microservices

### Classic Architecture

Classic GoldenGate is managed entirely through the GGSCI command-line utility. Processes (Extract, Pump, Replicat, Manager) run as OS-level processes under a single Oracle GoldenGate home. All configuration is stored in plain-text parameter files in the `dirprm/` subdirectory.

Classic architecture is the only option for GoldenGate 12c. It remains supported in GoldenGate 21c but is not the preferred deployment model for new installations.

Key characteristics:
- Single GoldenGate installation directory (`$OGG_HOME`)
- Manager process acts as the service controller
- Parameter files in `$OGG_HOME/dirprm/`, checkpoint files in `$OGG_HOME/dirchk/`, trail files in configurable paths
- Administration exclusively through GGSCI or OS file editing

### Microservices Architecture

Introduced in GoldenGate 18c and the standard for GoldenGate 19c and 21c deployments. The Microservices architecture replaces the single GoldenGate home with a set of REST-based services managed by a Service Manager.

Components:
- **Service Manager** — entry point; manages all other services within a deployment; bound to a specific network host and port
- **Admin Server** — hosts the Admin Client (web-based GGSCI equivalent) and REST API for process management
- **Distribution Server** — manages outbound trail delivery paths (replaces the standalone Pump Extract in classic)
- **Receiver Server** — accepts inbound trail delivery on the target side
- **Performance Metrics Server** — aggregates real-time throughput metrics across all components

Key characteristics:
- Configuration stored in a deployment directory separate from the GoldenGate software home
- Web UI available at `https://<host>:<ServiceManagerPort>` for process management
- REST API for automation
- Admin Client (web-based) replaces GGSCI for microservices-managed processes; GGSCI is also available in the GoldenGate home for compatibility

---

## Prerequisites

### System Requirements

Before installation, verify the following from the official certification sources. Do not assume support based on earlier releases.

- Source and target Oracle Database versions are in the GoldenGate certification matrix
- Operating system platform is certified for the selected GoldenGate release
- Required OS user (typically `oracle`) exists with appropriate directory permissions
- Network connectivity between source and target hosts on all required ports
- Time synchronization (NTP) between source, GoldenGate, and target hosts — unsynchronized clocks cause SCN and timestamp discrepancies

Certification matrix: https://www.oracle.com/integration/goldengate/certifications/

System requirements for GoldenGate 21c: https://docs.oracle.com/en/middleware/goldengate/core/21.3/coredoc/install-verify-certification-and-system-requirements.html

### Required Oracle Database Privileges

The GoldenGate database user on the source database requires a minimum set of privileges. For Oracle Database 19c with Integrated Extract, Oracle recommends using the `DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE` procedure to grant the necessary privileges rather than granting them individually.

```sql
-- Create the GoldenGate capture user on the source database
CREATE USER c##ggadmin IDENTIFIED BY "<password>"
  DEFAULT TABLESPACE users
  TEMPORARY TABLESPACE temp;

-- Use the provided procedure to grant required privileges (19c, CDB-aware)
EXEC DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE('c##ggadmin', CONTAINER => 'ALL');
```

For a non-CDB or PDB-specific setup, omit the `CONTAINER => 'ALL'` clause and connect to the target PDB.

For the Replicat user on the target database:

```sql
CREATE USER ggapply IDENTIFIED BY "<password>";
GRANT CREATE SESSION TO ggapply;
GRANT ALTER ANY TABLE TO ggapply;      -- for Integrated Replicat apply
GRANT INSERT ANY TABLE TO ggapply;
GRANT UPDATE ANY TABLE TO ggapply;
GRANT DELETE ANY TABLE TO ggapply;
-- Grant schema-specific privileges when a broader ANY TABLE grant is not acceptable
```

Verify minimum privilege lists against the installation guide for the specific GoldenGate release in use. Privilege requirements differ between Classic and Integrated Extract modes.

---

## Supplemental Logging on the Source Database

Supplemental logging must be enabled on the Oracle source database before Extract can capture change data reliably. There are two levels: database-level minimal supplemental logging and table-level supplemental logging.

### Step 1: Enable Minimal Supplemental Logging at the Database Level

```sql
-- Connect as SYSDBA to the source database
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;

-- Verify
SELECT SUPPLEMENTAL_LOG_DATA_MIN FROM V$DATABASE;
-- Expected: YES
```

Minimal supplemental logging records enough information for Oracle to reconstruct the primary key of each modified row. It must be enabled before adding table-level supplemental logging.

For a CDB source, enable minimal supplemental logging at the CDB root level:

```sql
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
```

### Step 2: Enable Table-Level Supplemental Logging

Use GGSCI `ADD TRANDATA` or the SQL `DBMS_STREAMS_ADM.ADD_TABLE_RULES` approach. The GGSCI approach is preferred for GoldenGate-managed replication.

From GGSCI on the source system:

```
DBLOGIN USERID c##ggadmin, PASSWORD "<password>"
ADD TRANDATA hr.employees
ADD TRANDATA hr.departments
```

To add supplemental logging for all tables in a schema:

```
ADD SCHEMATRANDATA hr
```

Verify supplemental logging is active for a table:

```
INFO TRANDATA hr.employees
```

From SQL*Plus, you can also verify:

```sql
SELECT LOG_GROUP_NAME, TABLE_NAME, LOG_GROUP_TYPE
FROM ALL_LOG_GROUPS
WHERE OWNER = 'HR';
```

Tables without a primary key require ALL COLUMNS supplemental logging, which increases redo volume. Identify such tables before enabling replication:

```sql
SELECT t.table_name
FROM all_tables t
WHERE t.owner = 'HR'
  AND t.table_name NOT IN (
    SELECT table_name
    FROM all_constraints
    WHERE owner = 'HR' AND constraint_type = 'P'
  );
```

---

## Installing Oracle GoldenGate 21c

### Download and Preparation

Download the GoldenGate 21c installer from Oracle Software Delivery Cloud or My Oracle Support. The installer is a ZIP archive for Linux x86-64. Verify the SHA-256 checksum before proceeding.

```bash
# Create the GoldenGate software home
mkdir -p /u01/app/oracle/goldengate/21c
chown oracle:oinstall /u01/app/oracle/goldengate/21c

# Unzip the installer (as the oracle OS user)
unzip V<nnnnn>-01.zip -d /tmp/ogg_installer
```

### Classic Architecture Installation

The Classic installer uses an OUI-based (Oracle Universal Installer) wizard or silent mode.

```bash
# Run the GUI installer
/tmp/ogg_installer/fbo_ggs_Linux_x64_Oracle_shiphome/Disk1/runInstaller

# Silent install example (adjust response file path)
/tmp/ogg_installer/fbo_ggs_Linux_x64_Oracle_shiphome/Disk1/runInstaller \
  -silent -responseFile /tmp/ogg_classic.rsp
```

Minimum response file parameters for a silent Classic install:

```ini
INSTALL_OPTION=ORA21c
SOFTWARE_LOCATION=/u01/app/oracle/goldengate/21c
START_MANAGER=false
DATABASE_LOCATION=/u01/app/oracle/product/21c/dbhome_1
MANAGER_PORT=7809
```

After installation, create the GoldenGate subdirectories:

```bash
cd /u01/app/oracle/goldengate/21c
./ggsci
```

Inside GGSCI:

```
GGSCI> CREATE SUBDIRS
```

This creates the required `dirprm/`, `dirchk/`, `dirdat/`, `dirrpt/`, `dirtmp/`, and `dirpcs/` subdirectories.

### Microservices Architecture Installation

The Microservices installer is also OUI-based. After the software is installed, a separate step creates the deployment using the Oracle GoldenGate Configuration Assistant (`oggca.sh`).

```bash
# Install the software (GUI or silent)
/tmp/ogg_installer/fbo_ggs_Linux_x64_Oracle_shiphome/Disk1/runInstaller

# After installation, create the deployment
$OGG_HOME/bin/oggca.sh
```

The `oggca.sh` wizard prompts for:
- Deployment name (e.g., `LocalOGG`)
- Deployment home directory (separate from the software home; e.g., `/u01/app/oracle/goldengate/deployments/LocalOGG`)
- Service Manager port
- Admin Server, Distribution Server, Receiver Server, and Performance Metrics Server ports
- Administrator username and password
- Oracle wallet location for credential storage

For a silent Microservices deployment creation:

```bash
$OGG_HOME/bin/oggca.sh -silent \
  -responseFile /tmp/ogg_microservices.rsp
```

After the deployment is created, start the Service Manager:

```bash
# The Service Manager startup script is placed in the deployment home
$OGG_DEPLOY_HOME/bin/startSM.sh
```

Access the Service Manager web console at:

```
https://<host>:<ServiceManagerPort>
```

---

## Manager Configuration (Classic Architecture)

The Manager process is required in Classic GoldenGate. It monitors processes, manages trail files, and listens on a configured port for remote connections.

Create or edit `$OGG_HOME/dirprm/mgr.prm`:

```
-- Manager parameter file
PORT 7809
PURGEOLDEXTRACTS ./dirdat/*, USECHECKPOINTS, MINKEEPDAYS 3
LAGCRITICALMINUTES 30
LAGREPORTMINUTES 10
AUTORESTART EXTRACT *, WAITMINUTES 5, RESETMINUTES 60
AUTORESTART REPLICAT *, WAITMINUTES 5, RESETMINUTES 60
```

Start Manager from GGSCI:

```
GGSCI> START MANAGER
GGSCI> INFO MANAGER
```

The `PURGEOLDEXTRACTS` parameter is required to prevent trail files from exhausting disk space. `USECHECKPOINTS` ensures trails are not purged before all downstream processes have consumed them.

---

## Extract Process Configuration

### Integrated Extract (Oracle 11.2.0.3 and later — recommended)

Integrated Extract uses Oracle LogMiner infrastructure and is the preferred capture mode for Oracle source databases from 11.2.0.3 onward.

Create the Extract parameter file at `$OGG_HOME/dirprm/ext1.prm`:

```
-- Extract parameter file: ext1.prm
EXTRACT ext1
USERIDALIAS oggcred DOMAIN OracleGoldenGate   -- use a wallet-based credential alias
EXTTRAIL ./dirdat/lt                          -- local trail path and two-character prefix
LOGALLSUPCOLS                                 -- write supplemental columns to trail
TABLE hr.employees;
TABLE hr.departments;
```

Register and add the Integrated Extract from GGSCI:

```
GGSCI> DBLOGIN USERID c##ggadmin, PASSWORD "<password>"
GGSCI> REGISTER EXTRACT ext1 DATABASE
GGSCI> ADD EXTRACT ext1, INTEGRATED TRANLOG, BEGIN NOW
GGSCI> ADD EXTTRAIL ./dirdat/lt, EXTRACT ext1, MEGABYTES 500
GGSCI> START EXTRACT ext1
GGSCI> INFO EXTRACT ext1
```

The `REGISTER EXTRACT` command registers the Extract with the Oracle Database LogMiner infrastructure. This step is required for Integrated Extract and must be done before adding the trail.

### Classic Extract (pre-11.2.0.3 or non-Oracle source)

```
-- Classic Extract parameter file: ext1.prm
EXTRACT ext1
USERID c##ggadmin, PASSWORD "<password>"
EXTTRAIL ./dirdat/lt
TABLE hr.employees;
TABLE hr.departments;
```

```
GGSCI> ADD EXTRACT ext1, TRANLOG, BEGIN NOW
GGSCI> ADD EXTTRAIL ./dirdat/lt, EXTRACT ext1, MEGABYTES 500
GGSCI> START EXTRACT ext1
```

### Pump Extract (Data Pump)

A Pump Extract reads from the local trail and writes to a remote trail on the target system. Using a Pump isolates the primary Extract from network interruptions and allows buffering.

Create `$OGG_HOME/dirprm/pmp1.prm`:

```
-- Pump Extract parameter file: pmp1.prm
EXTRACT pmp1
PASSTHRU                                      -- pass trail records without transformation
RMTHOST target-host.example.com, MGRPORT 7809
RMTTRAIL ./dirdat/rt                          -- remote trail path and prefix
TABLE hr.*;
```

```
GGSCI> ADD EXTRACT pmp1, EXTTRAILSOURCE ./dirdat/lt
GGSCI> ADD RMTTRAIL ./dirdat/rt, EXTRACT pmp1, MEGABYTES 500
GGSCI> START EXTRACT pmp1
GGSCI> INFO EXTRACT pmp1
```

---

## Replicat Process Configuration

### Integrated Replicat (Oracle target 12.1.2 and later — recommended)

Integrated Replicat applies transactions using the Oracle Database inbound server, which provides transaction dependency resolution for parallelism.

Create `$OGG_HOME/dirprm/rep1.prm` on the target system:

```
-- Replicat parameter file: rep1.prm
REPLICAT rep1
USERIDALIAS ggapplycred DOMAIN OracleGoldenGate
ASSUMETARGETDEFS                              -- source and target have the same table structure
DISCARDFILE ./dirrpt/rep1.dsc, APPEND, MEGABYTES 50
MAP hr.employees, TARGET hr.employees;
MAP hr.departments, TARGET hr.departments;
```

```
GGSCI> DBLOGIN USERID ggapply, PASSWORD "<password>"
GGSCI> ADD REPLICAT rep1, INTEGRATED, EXTTRAIL ./dirdat/rt
GGSCI> START REPLICAT rep1
GGSCI> INFO REPLICAT rep1
```

The `DISCARDFILE` parameter specifies where records that cannot be applied are written. Monitor this file during and after the initial load phase.

### Classic Replicat

```
-- Classic Replicat parameter file
REPLICAT rep1
USERID ggapply, PASSWORD "<password>"
ASSUMETARGETDEFS
DISCARDFILE ./dirrpt/rep1.dsc, APPEND, MEGABYTES 50
MAP hr.employees, TARGET hr.employees;
MAP hr.departments, TARGET hr.departments;
```

```
GGSCI> ADD REPLICAT rep1, EXTTRAIL ./dirdat/rt
GGSCI> START REPLICAT rep1
```

---

## Initial Load Setup

An initial load populates the target tables with a consistent copy of the source data before Change Data Capture (CDC) begins. GoldenGate provides two primary initial load methods for Oracle-to-Oracle replication.

### Method 1: GoldenGate Direct Load (Extract/Replicat)

This method uses a dedicated initial-load Extract and Replicat pair. The initial-load Extract performs a full table scan and writes records to a file-based trail or directly to the Replicat.

**On the source system**, capture the start SCN for CDC before the initial load begins:

```sql
SELECT CURRENT_SCN FROM V$DATABASE;
-- Record this SCN value; it is the starting point for the CDC Extract after load completes
```

Create the initial-load Extract parameter file `$OGG_HOME/dirprm/iext.prm`:

```
-- Initial Load Extract parameter file
EXTRACT iext
USERID c##ggadmin, PASSWORD "<password>"
RMTHOST target-host.example.com, MGRPORT 7809
RMTTASK REPLICAT, GROUP irep           -- direct task delivery (no trail file)
TABLE hr.employees;
TABLE hr.departments;
```

Create the initial-load Replicat parameter file `$OGG_HOME/dirprm/irep.prm` on the target:

```
-- Initial Load Replicat (task mode) parameter file
REPLICAT irep
ASSUMETARGETDEFS
DISCARDFILE ./dirrpt/irep.dsc, APPEND, MEGABYTES 50
MAP hr.employees, TARGET hr.employees;
MAP hr.departments, TARGET hr.departments;
```

```
-- On source GGSCI
GGSCI> ADD EXTRACT iext, SOURCEISTABLE
-- On target GGSCI
GGSCI> ADD REPLICAT irep, SPECIALRUN

-- Start the initial load from source
GGSCI> START EXTRACT iext
-- Monitor completion
GGSCI> VIEW REPORT iext
```

### Method 2: Oracle Data Pump Export/Import with GoldenGate SCN Coordination

For large tables where a full GoldenGate scan is slower than a Data Pump export, use Data Pump with SCN-consistent export:

```bash
# On the source host — consistent export at a known SCN
expdp c##ggadmin/password DIRECTORY=dp_dir DUMPFILE=hr_exp.dmp \
  SCHEMAS=hr FLASHBACK_SCN=<recorded_SCN> LOGFILE=hr_exp.log
```

After the import completes on the target, start the CDC Extract from the recorded SCN:

```
GGSCI> ADD EXTRACT ext1, INTEGRATED TRANLOG, SCN <recorded_SCN>
```

### Enabling HANDLECOLLISIONS During Initial Load Catchup

After the initial load, the CDC Replicat may encounter rows already applied by the load. Enable `HANDLECOLLISIONS` temporarily to suppress unique key conflicts during this window:

```
-- Add HANDLECOLLISIONS to the Replicat parameter file during catchup only
REPLICAT rep1
...
HANDLECOLLISIONS
MAP hr.employees, TARGET hr.employees;
```

Remove `HANDLECOLLISIONS` from the parameter file after the Replicat has caught up to the current SCN and lag approaches zero. Do not leave it enabled in steady-state operation; it suppresses errors that may indicate real data quality issues.

---

## Trail File Configuration

Trail files are the primary transport mechanism in GoldenGate. Each trail consists of a series of fixed-size segment files written by Extract and read by Pump or Replicat.

### Trail Naming Convention

Trail prefixes use two characters. Local trails (written by Extract on the source) and remote trails (written by Pump on the target) use separate prefix characters to avoid conflicts:

```
Local trail:  ./dirdat/lt  (files: lt000000, lt000001, ...)
Remote trail: ./dirdat/rt  (files: rt000000, rt000001, ...)
```

### Trail File Size

The `MEGABYTES` parameter on `ADD EXTTRAIL` or `ADD RMTTRAIL` sets the segment size. The default is 500 MB. Tune based on available disk space and purge frequency requirements.

```
GGSCI> ADD EXTTRAIL ./dirdat/lt, EXTRACT ext1, MEGABYTES 500
```

### Trail Purge

Configure `PURGEOLDEXTRACTS` in the Manager parameter file to automatically remove trail segments no longer needed by any downstream process:

```
PURGEOLDEXTRACTS ./dirdat/*, USECHECKPOINTS, MINKEEPDAYS 3
```

`USECHECKPOINTS` prevents purge of any trail segment that has not been fully consumed by all registered processes. `MINKEEPDAYS 3` retains segments for at least three days regardless of checkpoint status, providing a recovery window.

---

## Microservices Architecture: Service Manager and Admin Client

### Service Manager Setup

After creating the deployment with `oggca.sh`, verify services are running:

```bash
# Check Service Manager and deployment status
$OGG_DEPLOY_HOME/bin/ServiceManager status
```

In the Service Manager web UI (`https://<host>:<port>`):

1. Log in with the administrator credentials set during deployment creation.
2. Navigate to the deployment to see the Admin Server, Distribution Server, Receiver Server, and Performance Metrics Server status.
3. Use the **Administration Server** link to access process management.

### Creating Extract and Replicat via Admin Client (Microservices)

The Admin Client is a command-line interface for Microservices that accepts the same commands as classic GGSCI with some additions.

Connect to the Admin Client:

```bash
$OGG_HOME/bin/adminclient
```

Inside Admin Client:

```
OGG> CONNECT https://<host>:<AdminServerPort> DEPLOYMENT LocalOGG AS ggadmin PASSWORD "<password>"
OGG> DBLOGIN USERIDALIAS oggcred DOMAIN OracleGoldenGate
OGG> ADD EXTRACT ext1, INTEGRATED TRANLOG, BEGIN NOW
OGG> ADD EXTTRAIL ./dirdat/lt, EXTRACT ext1, MEGABYTES 500
OGG> CREATE AND START EXTRACT ext1 USING PARAMS "<param_text>"
```

Parameters can also be created in the Admin Server web UI under **Administration Server > Extracts > Add Extract**.

### Creating a Distribution Path (Microservices)

In the Microservices architecture, the Distribution Server replaces the Pump Extract for trail delivery between systems.

From the Admin Server web UI:

1. Navigate to **Distribution Server**.
2. Click **Add Path**.
3. Specify:
   - Source trail path
   - Target Receiver Server host and port
   - Target trail path on the receiving deployment
   - Credentials for the target Receiver Server

Distribution paths can also be created via the REST API or Admin Client.

---

## Connectivity Validation

### Classic Architecture

Verify Manager is reachable on the target from the source:

```bash
# Test network connectivity to the remote Manager port
telnet target-host.example.com 7809
# or
nc -zv target-host.example.com 7809
```

From source GGSCI, verify the remote connection:

```
GGSCI> INFO EXTRACT pmp1
GGSCI> LAG EXTRACT pmp1
```

Check the Pump report file for connection errors:

```
GGSCI> VIEW REPORT pmp1
```

### Microservices Architecture

Verify the Service Manager is reachable:

```bash
curl -k https://target-host.example.com:<ServiceManagerPort>/services/v2
# Expected: HTTP 200 with a JSON response listing deployment metadata
```

Verify Distribution Server to Receiver Server connectivity from the source Admin Client:

```
OGG> INFO DISTPATH dp1
```

### Database Connectivity

Test the GoldenGate database credential from GGSCI before starting any process:

```
-- Classic GGSCI
GGSCI> DBLOGIN USERID c##ggadmin, PASSWORD "<password>"
GGSCI> INFO TRANDATA hr.employees

-- Admin Client (Microservices)
OGG> DBLOGIN USERIDALIAS oggcred DOMAIN OracleGoldenGate
OGG> INFO TRANDATA hr.employees
```

A successful `INFO TRANDATA` response that shows supplemental logging enabled confirms both database connectivity and supplemental logging status in one step.

---

## Best Practices and Common Mistakes

### Best Practices

- Enable minimal supplemental logging at the database level and use `ADD TRANDATA` or `ADD SCHEMATRANDATA` for targeted table-level logging. Do not enable ALL COLUMNS logging globally without evaluating redo volume impact.
- Use `DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE` to assign the capture user's privileges. Do not grant `DBA` to the GoldenGate user.
- Use Oracle wallet-based credential aliases (`USERIDALIAS`) instead of plaintext passwords in parameter files.
- Include a `DISCARDFILE` in every Replicat parameter file. An absent discard file means rejected records are lost silently.
- Include `PURGEOLDEXTRACTS` in the Manager parameter file for every environment. Trail disk exhaustion will stop Extract processes.
- Remove `HANDLECOLLISIONS` from the Replicat parameter file after the initial load catchup window closes.
- Validate connectivity (Manager port reachability, DBLOGIN success) before creating Extract or Replicat processes.

### Common Mistakes

- **Starting Extract before enabling supplemental logging.** Extract will start but will ABEND when it encounters a row change with no supplemental logging columns.
- **Forgetting `REGISTER EXTRACT` for Integrated Extract.** Without registration, the Oracle Database LogMiner server has no record of the Extract, and Extract cannot capture changes.
- **Using identical trail prefixes for local and remote trails.** Trail prefix collisions cause Extract or Pump to overwrite trail segments.
- **Running NTP-unsynchronized source and target.** Timestamp drift causes incorrect lag reporting and can cause Replicat conflicts on bi-directional setups.
- **Not specifying `MEGABYTES` on `ADD EXTTRAIL`.** The default trail segment size may not match operational requirements; always set it explicitly.
- **Leaving `HANDLECOLLISIONS` enabled after initial load catchup.** This suppresses PK and UK violations in steady-state replication, masking real data quality issues.

---

## Oracle Version Notes (19c vs 26ai)

This section refers to Oracle Database and GoldenGate release differences relevant to installation and configuration.

### GoldenGate 12c (Classic Architecture)

- GoldenGate 12c does not include the Microservices architecture. All management is through GGSCI and OS-level file editing.
- Integrated Extract for Oracle source requires Oracle Database 11.2.0.3 or later; it is not available for earlier database versions.
- Integrated Replicat is introduced in GoldenGate 12.1.2 and requires Oracle Database 12.1 or later as the target.
- `REGISTER EXTRACT` is required for Integrated Extract in GoldenGate 12c, as it is in all later releases.
- The `DBMS_GOLDENGATE_AUTH` package is available from Oracle Database 12c. For Oracle Database 11g source, grant privileges individually per the 12c installation guide.

### GoldenGate 21c (Current Baseline)

- Microservices architecture is the recommended deployment model. Classic architecture is supported but not preferred for new installations.
- `oggca.sh` replaces the older `ogg_setup.sh` and `oggcore.rsp` deployment creation workflow.
- Parallel Replicat (introduced in GoldenGate 19.1) is available and is the recommended apply mode for Oracle 19c and later targets where parallelism is required.
- CDB/PDB-aware Extract configuration (`c##` common users, `CONTAINER => 'ALL'` privilege grants) is standard for Oracle 19c CDB sources.

### Oracle Database 19c Source

- Enable minimal supplemental logging at the CDB root level before configuring PDB-level Extract.
- Use `c##ggadmin` (common user) for CDB-wide capture; use a local PDB user only for single-PDB capture.
- Verify that the LogMiner infrastructure is available in the source CDB root: `SELECT * FROM V$GOLDENGATE_CAPABILITIES` (available in Oracle 12c and later; column availability varies by release — consult release-specific documentation).

### Oracle Database 26ai Target

- Before configuring GoldenGate 21c Replicat against an Oracle 26ai target, verify certification status explicitly using the GoldenGate certification matrix. Do not assume compatibility based on 19c certifications.
- Review the 26ai-specific parameter compatibility notes in the Oracle GoldenGate 23c or latest documentation before reusing existing Replicat parameter files from 19c environments.
- If certification is unclear or unconfirmed, do not proceed to production configuration without a verified source.

---

## Sources

- Oracle GoldenGate 21c Installation and Upgrade Guide: https://docs.oracle.com/en/middleware/goldengate/core/21.3/coredoc/install-oracle-goldengate.html
- Oracle GoldenGate 21c Certification and System Requirements: https://docs.oracle.com/en/middleware/goldengate/core/21.3/coredoc/install-verify-certification-and-system-requirements.html
- Oracle GoldenGate 21c Core Documentation (overview and navigation): https://docs.oracle.com/en/middleware/goldengate/core/21.3/ggcab/overview-oracle-goldengate.html
- Oracle GoldenGate 23c Core Documentation: https://docs.oracle.com/en/middleware/goldengate/core/23/coredoc/toc.htm
- Oracle GoldenGate 12.3.0.1 Documentation Library: https://docs.oracle.com/en/middleware/goldengate/core/12.3.0.1/books.html
- Oracle GoldenGate 19.1 Documentation: https://docs.oracle.com/en/middleware/goldengate/core/19.1/
- Oracle GoldenGate Documentation Hub: https://docs.oracle.com/en/database/goldengate/core/
- Oracle GoldenGate Certification Matrix: https://www.oracle.com/integration/goldengate/certifications/
