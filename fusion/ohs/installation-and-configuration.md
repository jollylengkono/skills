# Oracle HTTP Server 12c — Installation and Configuration

## Overview

Oracle HTTP Server (OHS) 12.2.1.4 is the Oracle-supported web server component of Oracle Fusion Middleware 12c. It is based on Apache HTTP Server and is the standard front-end for WebLogic Server proxying, SSL/TLS termination, and virtual host routing in Oracle Fusion Middleware deployments.

OHS can be deployed in two topologies:

- **Standalone domain** — an OHS instance managed by its own Fusion Middleware domain without a WebLogic Server, suitable for pure reverse-proxy and static-content scenarios.
- **Collocated with WebLogic** — an OHS instance registered in a WebLogic domain, sharing domain infrastructure with managed servers.

The core management tool is `opmnctl` (Oracle Process Manager and Notification Server control), which starts, stops, and monitors OHS component instances. Lifecycle integration with a WebLogic domain uses Node Manager.

---

## Prerequisites

### Platform Support

OHS 12.2.1.4 is certified on 64-bit Linux platforms including:

- Oracle Linux 7.x, 8.x (x86-64)
- Red Hat Enterprise Linux 7.x, 8.x (x86-64)
- SUSE Linux Enterprise Server 12.x, 15.x (x86-64)

Confirm exact platform and OS-version certification in the Oracle Fusion Middleware 12c Certification Matrix before installation.

### Required OS Packages

The OHS installer requires the following packages on Oracle Linux / RHEL:

```bash
yum install -y binutils gcc glibc glibc-devel libaio \
  libstdc++ libstdc++-devel make sysstat unzip
```

### Java Development Kit

OHS 12.2.1.4 requires JDK 8 (1.8.0_191 or later). The Oracle JDK is available from the Oracle JDK download page. Set `JAVA_HOME` before running the installer:

```bash
export JAVA_HOME=/usr/java/jdk1.8.0_211
export PATH=$JAVA_HOME/bin:$PATH
```

### Required Ports

| Component | Default Port | Description |
|-----------|-------------|-------------|
| OHS HTTP listener | 7777 | Plain HTTP |
| OHS HTTPS listener | 4443 | SSL/TLS |
| OPMN request port | 6700 | opmnctl communication |
| OPMN local port | 6701 | OPMN local |

Adjust these in component configuration files after creation. Ensure host firewall rules allow traffic on the selected ports.

### Disk and Memory

| Resource | Minimum |
|----------|---------|
| Oracle Home | 2.5 GB |
| /tmp | 500 MB |
| RAM | 2 GB (4 GB recommended) |

---

## Installation: Installer Overview

OHS 12.2.1.4 is installed using the Fusion Middleware Infrastructure installer or the standalone OHS installer (`fmw_12.2.1.4.0_ohs_linux64.bin`). The standalone OHS installer is used when a full WebLogic/FMW Infrastructure install is not required.

### Step 1: Download the Installer

Download `fmw_12.2.1.4.0_ohs_linux64.bin` from Oracle Software Delivery Cloud (eDelivery) or My Oracle Support. Verify the posted SHA-256 checksum before proceeding.

### Step 2: Run the Installer

```bash
chmod +x fmw_12.2.1.4.0_ohs_linux64.bin
./fmw_12.2.1.4.0_ohs_linux64.bin
```

For a silent install, use a response file:

```bash
./fmw_12.2.1.4.0_ohs_linux64.bin -silent \
  -responseFile /tmp/ohs_install.rsp \
  -jreLoc $JAVA_HOME
```

A minimal silent install response file:

```ini
[ENGINE]
Response File Version=1.0.0.0.0

[GENERIC]
ORACLE_HOME=/u01/oracle/middleware/Oracle_HTTP_Server1
INSTALL_TYPE=Standalone HTTP Server (Managed independently of WebLogic server)
```

Valid `INSTALL_TYPE` values are:
- `Standalone HTTP Server (Managed independently of WebLogic server)`
- `Colocated HTTP Server (Managed through WebLogic server)`

Choosing the install type at installer time determines which configuration tooling is used afterward but does not prevent later collocated domain association.

### Step 3: Installer Screens (GUI)

1. **Installation Inventory Setup** — Accept or change the central inventory location.
2. **Welcome** — Review supported configurations link.
3. **Auto Updates** — Skip or configure a patch source.
4. **Installation Location** — Set the Oracle Home (e.g., `/u01/oracle/middleware/Oracle_HTTP_Server1`).
5. **Installation Type** — Select Standalone or Colocated.
6. **Prerequisite Checks** — Resolve any failures before proceeding.
7. **Installation Summary** — Review and click Install.
8. **Installation Progress** — Wait for completion.
9. **Installation Complete** — Note the Oracle Home path for subsequent configuration.

---

## Domain Creation and OHS Component Creation

After installing the OHS Oracle Home, create a domain and an OHS component instance using the `config.sh` Configuration Wizard or `wlst` offline scripts.

### Standalone Domain via Configuration Wizard (GUI)

```bash
$ORACLE_HOME/oracle_common/common/bin/config.sh
```

Wizard steps for a new standalone domain:

1. **Create a new domain** — provide the domain home location (e.g., `/u01/oracle/config/domains/ohs_domain`).
2. **Templates** — select **Oracle HTTP Server (Standalone)**.
3. **System Components** — add an OHS component, specify a name (e.g., `ohs1`), and confirm the listen address and ports.
4. **OHS Server** — set HTTP and HTTPS listen ports.
5. **Configuration Summary** — review and click Create.

### Standalone Domain Silent Configuration

Use a WLST offline script to create a standalone domain silently:

```python
# create_ohs_domain.py — run with wlst.sh offline
selectTemplate('Oracle HTTP Server (Standalone)')
loadTemplates()

cd('/SystemComponent/ohs1')
set('ListenPort', 7777)
set('ListenPortEnabled', True)
set('SSLListenPort', 4443)

setOption('DomainName', 'ohs_domain')
writeDomain('/u01/oracle/config/domains/ohs_domain')
closeTemplate()
```

Run it:

```bash
$ORACLE_HOME/oracle_common/common/bin/wlst.sh create_ohs_domain.py
```

### Collocated Domain Configuration

When OHS is collocated in a WebLogic domain, run the Configuration Wizard from the existing WebLogic domain home and add the **Oracle HTTP Server** template. The OHS instance is then registered with Node Manager and managed via the WebLogic Admin Console or `wlst`.

---

## Starting and Stopping OHS with opmnctl

After domain creation, OHS instances are managed with `opmnctl` from the domain's `bin` directory.

```bash
# Start all OPMN-managed components in the domain
$DOMAIN_HOME/bin/startComponent.sh ohs1

# Stop a component
$DOMAIN_HOME/bin/stopComponent.sh ohs1

# Check component status via opmnctl directly
$ORACLE_HOME/opmn/bin/opmnctl @$DOMAIN_HOME/config/fmwconfig/components/OPMN \
  status -l

# Start all components using opmnctl
$ORACLE_HOME/opmn/bin/opmnctl @$DOMAIN_HOME/config/fmwconfig/components/OPMN \
  startall

# Stop all components
$ORACLE_HOME/opmn/bin/opmnctl @$DOMAIN_HOME/config/fmwconfig/components/OPMN \
  stopall
```

The `startComponent.sh` / `stopComponent.sh` scripts in `$DOMAIN_HOME/bin` wrap `opmnctl` and set the required environment automatically. Prefer them over calling `opmnctl` directly unless troubleshooting OPMN itself.

---

## SSL/TLS Certificate Configuration

OHS uses an Oracle Wallet (PKCS#12 format managed by `orapki`) to store SSL/TLS certificates and keys. The wallet is located at:

```
$DOMAIN_HOME/config/fmwconfig/components/OHS/<instance_name>/keystores/default/
```

### Creating a Self-Signed Wallet (Development/Testing)

```bash
# Create a new auto-login wallet
orapki wallet create \
  -wallet $DOMAIN_HOME/config/fmwconfig/components/OHS/ohs1/keystores/default \
  -auto_login -pwd WalletPasswd123

# Add a self-signed certificate
orapki wallet add \
  -wallet $DOMAIN_HOME/config/fmwconfig/components/OHS/ohs1/keystores/default \
  -dn "CN=ohs1.example.com,OU=IT,O=ExampleOrg,L=Austin,ST=Texas,C=US" \
  -keysize 2048 \
  -self_signed \
  -validity 365 \
  -pwd WalletPasswd123

# Display wallet contents
orapki wallet display \
  -wallet $DOMAIN_HOME/config/fmwconfig/components/OHS/ohs1/keystores/default \
  -pwd WalletPasswd123
```

### Importing a CA-Signed Certificate (Production)

```bash
# Step 1: Generate a certificate signing request (CSR)
orapki wallet add \
  -wallet $DOMAIN_HOME/config/fmwconfig/components/OHS/ohs1/keystores/default \
  -dn "CN=ohs1.example.com,OU=IT,O=ExampleOrg,C=US" \
  -keysize 2048 \
  -pwd WalletPasswd123

orapki wallet export \
  -wallet $DOMAIN_HOME/config/fmwconfig/components/OHS/ohs1/keystores/default \
  -dn "CN=ohs1.example.com,OU=IT,O=ExampleOrg,C=US" \
  -request /tmp/ohs1.csr \
  -pwd WalletPasswd123

# Step 2: After receiving the signed certificate from your CA,
# import the CA root and any intermediate certificates first
orapki wallet add \
  -wallet $DOMAIN_HOME/config/fmwconfig/components/OHS/ohs1/keystores/default \
  -trusted_cert -cert /tmp/ca-root.crt \
  -pwd WalletPasswd123

orapki wallet add \
  -wallet $DOMAIN_HOME/config/fmwconfig/components/OHS/ohs1/keystores/default \
  -trusted_cert -cert /tmp/ca-intermediate.crt \
  -pwd WalletPasswd123

# Step 3: Import the signed server certificate
orapki wallet add \
  -wallet $DOMAIN_HOME/config/fmwconfig/components/OHS/ohs1/keystores/default \
  -user_cert -cert /tmp/ohs1-signed.crt \
  -pwd WalletPasswd123
```

### Configuring SSL in ssl.conf

The OHS SSL virtual host configuration is in:

```
$DOMAIN_HOME/config/fmwconfig/components/OHS/<instance_name>/ssl.conf
```

Key directives for SSL configuration:

```apache
<VirtualHost *:4443>
    ServerName ohs1.example.com:4443

    SSLEngine on
    SSLProtocol TLSv1.2 TLSv1.3
    SSLCipherSuite HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4

    # Oracle Wallet path — must match the wallet directory
    SSLWallet "$DOMAIN_HOME/config/fmwconfig/components/OHS/ohs1/keystores/default"

    DocumentRoot "/u01/oracle/ohs_static"
    <Directory "/u01/oracle/ohs_static">
        Require all granted
    </Directory>
</VirtualHost>
```

Restart the OHS instance after any SSL configuration change:

```bash
$DOMAIN_HOME/bin/stopComponent.sh ohs1
$DOMAIN_HOME/bin/startComponent.sh ohs1
```

---

## mod_wl_ohs: Proxying to WebLogic Server

`mod_wl_ohs` is the Oracle WebLogic proxy module included with OHS. It forwards HTTP and HTTPS requests from OHS to one or more WebLogic Managed Servers or clusters.

The module configuration file is:

```
$DOMAIN_HOME/config/fmwconfig/components/OHS/<instance_name>/mod_wl_ohs.conf
```

### Basic Proxy Configuration

```apache
LoadModule weblogic_module "$ORACLE_HOME/ohs/modules/mod_wl_ohs.so"

<IfModule mod_weblogic.c>
    # WebLogicHost targets a single Managed Server
    # WebLogicCluster targets a cluster (round-robin load balancing)

    <Location /myapp>
        SetHandler weblogic-handler
        WebLogicCluster wls1.example.com:7003,wls2.example.com:7003
        WLProxySSL OFF
        WLProxySSLPassThrough OFF
    </Location>
</IfModule>
```

### Proxying with SSL to WebLogic (WLProxySSL)

When WebLogic Managed Servers use HTTPS listen ports:

```apache
<Location /secureapp>
    SetHandler weblogic-handler
    WebLogicCluster wls1.example.com:7004,wls2.example.com:7004
    WLProxySSL ON
    WLProxySSLPassThrough ON
</Location>
```

### Important mod_wl_ohs Parameters

| Parameter | Description |
|-----------|-------------|
| `WebLogicHost` | Single WebLogic host and port |
| `WebLogicCluster` | Comma-separated cluster member list `host:port,...` |
| `WLProxySSL` | Use SSL between OHS and WebLogic (`ON`/`OFF`) |
| `ConnectTimeoutSecs` | Seconds to wait for WebLogic connection (default: 10) |
| `ConnectRetrySecs` | Seconds between retry attempts |
| `WLIOTimeoutSecs` | Seconds to wait for WebLogic response (default: 300) |
| `KeepAliveEnabled` | Enable HTTP keep-alive to WebLogic (`true`/`false`) |
| `KeepAliveSecs` | Keep-alive idle timeout in seconds |
| `FileCaching` | Enable plugin-level response caching (`ON`/`OFF`) |
| `ErrorPage` | Custom error page URL on backend failure |

Refer to Oracle documentation for the complete parameter reference; undocumented parameters should not be assumed.

---

## Virtual Hosts

OHS virtual host configuration follows Apache HTTP Server conventions. The main configuration file is `httpd.conf`, with included component files for SSL and module-specific settings.

Main config location:

```
$DOMAIN_HOME/config/fmwconfig/components/OHS/<instance_name>/httpd.conf
```

### Name-Based Virtual Host Example

```apache
# In httpd.conf or an included conf file

<VirtualHost *:7777>
    ServerName app1.example.com
    DocumentRoot "/u01/oracle/ohs_static/app1"

    <Location /api>
        SetHandler weblogic-handler
        WebLogicCluster wls-app1a.example.com:7003,wls-app1b.example.com:7003
    </Location>

    ErrorLog  "$DOMAIN_HOME/servers/ohs1/logs/app1_error.log"
    CustomLog "$DOMAIN_HOME/servers/ohs1/logs/app1_access.log" combined
</VirtualHost>

<VirtualHost *:7777>
    ServerName app2.example.com
    DocumentRoot "/u01/oracle/ohs_static/app2"

    <Location /ws>
        SetHandler weblogic-handler
        WebLogicCluster wls-app2a.example.com:7003,wls-app2b.example.com:7003
    </Location>
</VirtualHost>
```

### IP-Based Virtual Hosts

Bind OHS to specific IP addresses by replacing the wildcard in the `VirtualHost` directive:

```apache
Listen 10.0.1.10:7777
Listen 10.0.1.11:7777

<VirtualHost 10.0.1.10:7777>
    ServerName internal.example.com
    ...
</VirtualHost>

<VirtualHost 10.0.1.11:7777>
    ServerName external.example.com
    ...
</VirtualHost>
```

After editing any configuration file, validate the syntax before restarting:

```bash
$ORACLE_HOME/oracle_ohs/bin/apachectl -f \
  $DOMAIN_HOME/config/fmwconfig/components/OHS/ohs1/httpd.conf -t
```

---

## Post-Install Validation

### 1. Verify OPMN and OHS Process Status

```bash
$ORACLE_HOME/opmn/bin/opmnctl @$DOMAIN_HOME/config/fmwconfig/components/OPMN \
  status -l
```

Expected output shows `ohs1` with status `Alive`.

### 2. Confirm HTTP Listener Response

```bash
curl -I http://localhost:7777/
# Expected: HTTP/1.1 200 OK or 403 Forbidden (if no DocumentRoot index file)
# A connection refused indicates OHS is not listening.
```

### 3. Confirm HTTPS Listener (Skip Certificate Verification for Self-Signed)

```bash
curl -Ik https://localhost:4443/
```

### 4. Validate mod_wl_ohs Proxy

```bash
curl -I http://localhost:7777/myapp/
# Expected: response proxied from WebLogic (2xx, 3xx, or application error page)
# Connection refused or 503 indicates WebLogic backend is unreachable.
```

### 5. Review OHS Log Files

Log files are under:

```
$DOMAIN_HOME/servers/<instance_name>/logs/
```

Key log files:

| File | Contents |
|------|----------|
| `access_log` | Per-request access log (Apache combined format) |
| `error_log` | OHS errors, startup messages, module errors |
| `mod_wl_ohs.log` | WebLogic proxy module connection and routing messages |
| `opmn/logs/opmn.log` | OPMN daemon lifecycle and health messages |

Increase mod_wl_ohs logging verbosity for troubleshooting:

```apache
# In mod_wl_ohs.conf — add inside the IfModule block
<IfModule mod_weblogic.c>
    Debug ALL
    WLLogFile /tmp/mod_wl_ohs_debug.log
</IfModule>
```

Remove `Debug ALL` after troubleshooting; it generates high log volume.

---

## Best Practices and Common Mistakes

### Best Practices

- **Use an auto-login wallet** (`-auto_login`) for production OHS deployments so the server starts without requiring a wallet password at boot time.
- **Restrict SSLProtocol** to `TLSv1.2 TLSv1.3` and exclude SSLv2, SSLv3, and TLSv1.0/1.1 in `ssl.conf`.
- **Pin the WebLogicCluster list** to all Managed Server addresses (or use a cluster address) so mod_wl_ohs can route around individual server failures.
- **Set `WLIOTimeoutSecs`** to a value consistent with the longest expected application response time; the default (300 s) can be too long for interactive users and too short for heavy batch endpoints.
- **Separate access logs per virtual host** to simplify log analysis and archiving.
- **Validate `httpd.conf` syntax** with `apachectl -t` before every restart to catch typos before they cause a startup failure.
- **Use `KeepAliveEnabled true`** on the mod_wl_ohs connection to WebLogic to reduce TCP connection overhead on high-traffic sites.

### Common Mistakes

- **Running `opmnctl` without specifying the OPMN config path** — `opmnctl` run without the `@<domain-opmn-config>` argument defaults to the Oracle Home-level OPMN, not the domain-level OPMN, and will not find domain-managed components.
- **Editing configuration files under `$ORACLE_HOME` instead of `$DOMAIN_HOME`** — OHS instance configuration lives under the domain home, not the Oracle Home. Changes to Oracle Home templates have no effect on running instances.
- **Forgetting to import CA intermediates before the server certificate** — `orapki wallet add -user_cert` fails if the signing CA chain is not already trusted in the wallet.
- **Using `WLProxySSL ON` without confirming the WebLogic SSL listen port** — the default Managed Server SSL port is 7002, not 7003. Verify the actual SSL port in the WebLogic console before configuring the proxy.
- **Not restarting OHS after `ssl.conf` or `mod_wl_ohs.conf` changes** — OHS reads configuration files only at startup; `graceful` reload is not available via OPMN in standalone deployments. A full stop and start is required.
- **Using `Debug ALL` in mod_wl_ohs.conf in production** — this generates one log line per request header, consuming disk rapidly on busy sites.

---

## Oracle Version Notes

OHS versioning tracks Oracle Fusion Middleware 12c releases. The baseline for this skill is **OHS 12.2.1.4.0**, the long-term release in the 12c family.

- **12.2.1.3 vs 12.2.1.4** — Oracle Fusion Middleware 12.2.1.4 is the recommended production release; 12.2.1.3 reached end of Premier Support. Avoid new deployments on 12.2.1.3.
- **11g OHS (11.1.1.x)** — Used `webcache` and `opmn` differently; `mod_ohs.conf` and wallet paths differ. Do not apply 12c procedures to an 11g OHS instance.
- **TLS 1.3 support** — TLS 1.3 is available from OHS 12.2.1.4 with the applicable bundle patches applied. Confirm the installed patch level before configuring `SSLProtocol TLSv1.3` in `ssl.conf`.
- **No Oracle Database version dependency** — OHS is a web server component and has no direct dependency on an Oracle Database version. Version notes in this file do not follow the 19c/26ai database baseline convention.

---

## Sources

- Oracle Fusion Middleware Installing and Configuring Oracle HTTP Server 12c (12.2.1.4) — https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/wtins/index.html
- Oracle Fusion Middleware Administering Oracle HTTP Server 12c (12.2.1.4) — https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/wtadm/index.html
- Oracle Fusion Middleware Configuring the Web Tier 12c (12.2.1.4) — https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/wtcfg/index.html
- Oracle Fusion Middleware mod_wl_ohs Reference — https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/wtadm/mod_wl_ohs-module.html
- Oracle Fusion Middleware SSL Configuration in Oracle Fusion Middleware 12c — https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/asadm/configuring-ssl.html
- Oracle Database Security Guide: orapki Utility — https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/orapki-utility.html
- Oracle Fusion Middleware 12c Certification Matrix — https://www.oracle.com/middleware/technologies/fusion-certification.html
