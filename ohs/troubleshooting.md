# Oracle HTTP Server Troubleshooting

## Overview

Use this skill to diagnose and resolve Oracle HTTP Server (OHS) 12.2.1.4 operational issues, including process failures managed by OPMN, startup errors, SSL handshake failures, mod_weblogic proxy connectivity problems, and HTTP 502/503 errors from a WebLogic backend. All diagnostic commands and log paths follow Oracle Fusion Middleware 12.2.1.4 documentation.

## Fast Triage Workflow

1. Confirm OHS process state with `opmnctl status`.
2. Identify the failing component from the status output (state column: `Alive`, `Down`, `Init`, `Stop`).
3. Locate the relevant log: OPMN log for process-level failures; `error_log` for HTTP/SSL errors.
4. Classify the issue: process startup, SSL/TLS, mod_weblogic proxy, or HTTP error from WebLogic.

## OHS Process Management and OPMN Diagnostics

OHS 12.2.1.4 runs under Oracle Process Manager and Notification Server (OPMN). Use `opmnctl` from `$ORACLE_INSTANCE/bin/` or `$ORACLE_HOME/opmn/bin/` to inspect and control OHS processes.

### Check Process Status

```bash
# List all components and their states
$ORACLE_INSTANCE/bin/opmnctl status

# Show extended detail including pid and port
$ORACLE_INSTANCE/bin/opmnctl status -l
```

Key state values in the output:

| State  | Meaning |
|--------|---------|
| `Alive`  | Process is running normally |
| `Down`   | Process has exited or failed to start |
| `Init`   | Process is starting; persisting Init indicates a stuck start |
| `Stop`   | Process was stopped intentionally |

### Start, Stop, and Restart OHS

```bash
# Start all OPMN-managed components
$ORACLE_INSTANCE/bin/opmnctl startall

# Start only the OHS component
$ORACLE_INSTANCE/bin/opmnctl startproc ias-component=<component_name>

# Stop a specific OHS component
$ORACLE_INSTANCE/bin/opmnctl stopproc ias-component=<component_name>

# Restart a specific OHS component
$ORACLE_INSTANCE/bin/opmnctl restartproc ias-component=<component_name>
```

Replace `<component_name>` with the name shown in `opmnctl status` output (for example, `ohs1`).

### OPMN Diagnostic Commands

```bash
# Dump OPMN state to standard output for offline review
$ORACLE_INSTANCE/bin/opmnctl status -fmt "%cmp %prc %sta %pid %prt"

# Check OPMN daemon itself
$ORACLE_INSTANCE/bin/opmnctl ping
```

If `opmnctl ping` fails, the OPMN daemon is not running. Start it with:

```bash
$ORACLE_INSTANCE/bin/opmnctl start
```

## Log File Locations

All paths use `$ORACLE_INSTANCE` as the Oracle instance home (for example, `/u01/app/oracle/instances/instance1`).

| Log | Default Path |
|-----|-------------|
| OPMN daemon log | `$ORACLE_INSTANCE/diagnostics/logs/OPMN/opmn/opmn.log` |
| OHS component log (per-component) | `$ORACLE_INSTANCE/diagnostics/logs/OHS/<component_name>/` |
| OHS error log | `$ORACLE_INSTANCE/diagnostics/logs/OHS/<component_name>/error_log` |
| OHS access log | `$ORACLE_INSTANCE/diagnostics/logs/OHS/<component_name>/access_log` |
| OHS console output | `$ORACLE_INSTANCE/diagnostics/logs/OHS/<component_name>/console~<component_name>.log` |
| mod_weblogic log | Configured via `WLLogFile` directive in `mod_wl_ohs.conf`; no default path |

Always check both the OPMN log and the OHS component error log together — OPMN records the process exit code, and `error_log` records the Apache-level failure reason.

## Startup Errors

Common causes of OHS failing to reach the `Alive` state:

- **Port conflict**: Another process holds the Listen port. Check `error_log` for `Address already in use`.
- **Invalid configuration syntax**: Apache-format syntax errors prevent `httpd` from parsing `httpd.conf`. Check `error_log` for `Syntax error`.
- **Missing SSL wallet**: If SSL is configured, a missing or inaccessible wallet causes an immediate exit. Check `error_log` for `SSL wallet` or `unable to open wallet`.
- **Permissions**: OHS cannot read its own configuration or log directory. Check file ownership and mode.

### Validate Configuration Before Starting

OHS supports offline configuration syntax checking using the `configtest` option:

```bash
$ORACLE_INSTANCE/config/OHS/<component_name>/httpd.conf

# Run syntax check using the OHS binary (adjust path to your ORACLE_HOME)
$ORACLE_HOME/ohs/bin/httpd -f $ORACLE_INSTANCE/config/OHS/<component_name>/httpd.conf -t
```

A clean output reads `Syntax OK`. Errors include the offending file name and line number.

## SSL Handshake Failures

OHS 12.2.1.4 uses Oracle Wallet Manager (OWM) or ORAPKI for certificate management. SSL is implemented via `mod_ossl`.

### Symptoms

- Client receives `SSL_ERROR_HANDSHAKE_FAILURE_ALERT` or `ERR_SSL_PROTOCOL_ERROR`.
- `error_log` contains messages such as `SSL handshake failed` or `SSL library error`.
- `error_log` contains `certificate verify failed`.

### Diagnostic Steps

1. Confirm the wallet path in `ssl.conf` or `httpd.conf` (`SSLWallet` directive).
2. Verify the wallet exists and is readable by the OHS OS user.
3. List wallet contents with ORAPKI:

```bash
orapki wallet display -wallet <wallet_directory>
```

4. Confirm the server certificate is not expired:

```bash
orapki wallet display -wallet <wallet_directory> | grep -A5 "Certificate"
```

5. Check that the issuing CA certificate is present in the wallet as a trusted certificate. A missing chain causes `certificate verify failed` on the client.

6. Confirm the configured `SSLProtocol` and `SSLCipherSuite` directives are compatible with the connecting client. OHS 12.2.1.4 ships with TLS 1.0, 1.1, and 1.2 enabled by default; TLS 1.3 is not supported on this version.

### Common SSL Errors in error_log

| Error text | Likely cause |
|------------|-------------|
| `SSL0119E: Initialization error, Could not find the certificate` | `SSLWallet` path is wrong or the certificate alias does not exist |
| `SSL0109E: Process interrupted or timed out` | Client aborted the handshake |
| `SSL0117E: SSL Handshake Failed` | Protocol or cipher suite mismatch between client and server |
| `NZ error` followed by error code | Oracle Security Services layer error; consult Oracle Support with the NZ code |

## mod_weblogic Connectivity and 502/503 Errors

OHS proxies requests to WebLogic Server using `mod_weblogic` (`mod_wl_ohs`). The configuration file is typically `$ORACLE_INSTANCE/config/OHS/<component_name>/moduleconf/mod_wl_ohs.conf`.

### Symptoms

- HTTP 502 Bad Gateway: OHS received an invalid or no response from WebLogic.
- HTTP 503 Service Unavailable: No WebLogic backend is available (all hosts in the cluster are down or overloaded).
- `error_log` contains `Connect to <host>:<port> failed` or `Recoverable error`.

### Diagnostic Steps

1. Confirm WebLogic Managed Servers are running (`opmnctl status` if co-managed; otherwise check WebLogic Admin Console).
2. Verify the `WebLogicHost` and `WebLogicPort` (or `WebLogicCluster`) values in `mod_wl_ohs.conf` match the actual Managed Server listen addresses.
3. Check network reachability from the OHS host to the WebLogic host and port.
4. Review `WLLogFile` output for connection-level errors. Set `WLLogLevel INFO` or `WLLogLevel DEBUG` temporarily to increase detail — change back to `WARNING` after diagnosis to avoid log volume growth.
5. Inspect the WebLogic Managed Server logs at `$DOMAIN_HOME/servers/<name>/logs/<name>.log` for application-level errors that cause the backend to return a bad response.

### mod_wl_ohs.conf Key Directives

| Directive | Purpose |
|-----------|---------|
| `WebLogicHost` | Single Managed Server hostname |
| `WebLogicPort` | Single Managed Server port |
| `WebLogicCluster` | Comma-separated `host:port` list for cluster load balancing |
| `ConnectTimeoutSecs` | Timeout before OHS gives up connecting to WebLogic |
| `WLIOTimeoutSecs` | Timeout waiting for a response once connected |
| `WLLogFile` | Path to the mod_weblogic log file |
| `WLLogLevel` | Log verbosity: `WARNING`, `INFO`, `DEBUG` |

A 502 caused by `WLIOTimeoutSecs` being too short for a slow backend will show `Recoverable error, responseCode=502` in the mod_weblogic log. Increase `WLIOTimeoutSecs` only after confirming the backend is legitimately slow rather than hung.

### Cluster Failover Behavior

When `WebLogicCluster` is configured, mod_weblogic retries a failed request on another cluster member before returning 503 to the client. If all members are unavailable, the error log records `All WebLogic servers are busy handling requests` or a connection refused message for each member.

## Crash and Core Dump Analysis

If OHS exits abnormally, the OPMN log records the exit signal. A core dump may be written to the working directory configured in `opmn.xml` (`working-dir` attribute of the `process-set` element) or to the OS core dump location.

### Steps

1. Check `$ORACLE_INSTANCE/diagnostics/logs/OPMN/opmn/opmn.log` for the exit signal (for example, `signal 11` indicates a segmentation fault).
2. Locate the core file. The default working directory for OHS processes set in `opmn.xml` is typically `$ORACLE_INSTANCE/diagnostics/logs/OHS/<component_name>/`.
3. Confirm the `ulimit -c` value for the OHS OS user is not `0`; a zero limit suppresses core dumps.
4. Analyze the core with `gdb` or provide it to Oracle Support under an SR. Oracle Support requires the core file, the OHS binary, and all shared libraries from the installation.
5. Check `error_log` immediately before the crash for the last Apache worker log entry, which can narrow the failing module.

Repeated crashes on the same URI path or after processing specific content types suggest a module-level bug; engage Oracle Support with the relevant patch set exception (PSE) search before attempting a workaround.

## Best Practices and Common Mistakes

- **Always validate httpd.conf syntax** after any configuration change before restarting OHS. A syntax error prevents all worker processes from starting.
- **Do not edit files under `$ORACLE_HOME` directly** for instance-level configuration. Instance-specific configuration lives under `$ORACLE_INSTANCE/config/OHS/`. Changes to `$ORACLE_HOME` are overwritten by patching.
- **Set `WLLogFile` explicitly** in `mod_wl_ohs.conf`. Without it, mod_weblogic messages go to `error_log` mixed with Apache messages, making triage harder.
- **Restart OHS through OPMN**, not by sending signals directly to the `httpd` parent process. OPMN tracks the PID; bypassing it leaves OPMN unaware of the state change.
- **Rotate logs proactively**. OHS access and error logs are not rotated automatically unless `CustomLog` or `ErrorLog` is configured with `rotatelogs`. Large log files degrade `grep` performance during incidents.
- **SSL wallet auto-login**: Use auto-login wallets (`-auto_login` flag with ORAPKI) in production so OHS does not require a wallet password at startup. Requiring a password prevents OPMN from restarting OHS automatically after a crash.
- **TLS version enforcement**: Restrict `SSLProtocol` to `TLSv1.2` in environments with compliance requirements. The default in 12.2.1.4 includes TLS 1.0 and 1.1.

## Oracle Version Notes (OHS 12.2.1.4)

This skill targets OHS 12.2.1.4, which ships as part of Oracle Fusion Middleware 12c Release 2 (12.2.1.4.0). Key version context:

- OHS 12.2.1.4 is based on Apache HTTP Server 2.4.x. Configuration directives follow Apache 2.4 syntax, not the Apache 2.2 syntax used in OHS 11g.
- OPMN in 12.2.1.4 uses the `opmnctl` command only. The older `opmnctl startall` / `opmnctl stopall` pattern from 11g is preserved, but the underlying daemon and XML schema differ.
- OHS 11g (11.1.1.x) used a different wallet integration (Oracle HTTP Server 11g mod_ossl) and stored instance configuration under `$ORACLE_INSTANCE/config/OHS/ohs<n>/`. Path structures are similar but not identical.
- TLS 1.3 is not available in OHS 12.2.1.4. TLS 1.3 support requires a later Fusion Middleware release.
- The `mod_weblogic` (mod_wl_ohs) version bundled in 12.2.1.4 supports WLS 12.2.1.x cluster addressing. Proxy plugin updates are delivered through Fusion Middleware patch bundles, not standalone patches.

## Sources

- Oracle Fusion Middleware Administering Oracle HTTP Server 12c (12.2.1.4): https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/administer-ohs/index.html
- Oracle Fusion Middleware Installing and Configuring Oracle HTTP Server 12c (12.2.1.4): https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/wtins/index.html
- Oracle Fusion Middleware Configuring the Web Server Plug-In for Oracle WebLogic Server 12c (12.2.1.4): https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/clpwn/index.html
- Oracle Fusion Middleware OPMN Administrator's Guide 12c: https://docs.oracle.com/middleware/1213/opmn/OPMAN/index.htm
- Oracle Platform Security Services (OPSS) / ORAPKI Command Reference: https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.4/asadm/index.html
