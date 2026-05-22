# Oracle HTTP Server Performance Tuning

## Overview

Use this skill to tune Oracle HTTP Server (OHS) 12.2.1.4 for throughput, connection efficiency, and backend proxy performance. OHS is built on Apache HTTP Server 2.4 and ships with Oracle-specific modules including `mod_weblogic` for proxying requests to WebLogic Server. Tuning centers on four areas: MPM worker model, connection management, backend proxy configuration, and SSL/TLS session handling.

Baseline version: OHS 12.2.1.4 (Oracle Fusion Middleware 12c).

## MPM Configuration: mpm_worker vs mpm_event

OHS 12.2.1.4 supports the Apache 2.4 prefork, worker, and event MPMs. For production deployments, `mpm_worker` or `mpm_event` are strongly preferred over `mpm_prefork` because they use threads rather than processes, dramatically reducing memory overhead per connection.

**mpm_worker** uses a fixed pool of threads per process. Each thread handles one connection at a time.

**mpm_event** (recommended for keep-alive workloads) decouples keep-alive idle connections from worker threads, allowing threads to handle active requests while a separate listener thread manages idle keep-alive connections. This improves concurrency when many clients hold open keep-alive connections but are not actively sending requests.

MPM selection is made in `$ORACLE_INSTANCE/config/OHS/<component_name>/httpd.conf` (or the included `mpm.conf`) by loading the appropriate module:

```apacheconf
# Load only one MPM
LoadModule mpm_event_module   "${ORACLE_HOME}/ohs/modules/mod_mpm_event.so"
# LoadModule mpm_worker_module  "${ORACLE_HOME}/ohs/modules/mod_mpm_worker.so"
```

### MaxRequestWorkers and ThreadsPerChild

These two directives control the total thread concurrency ceiling.

```apacheconf
<IfModule mpm_event_module>
    StartServers            2
    MinSpareThreads        25
    MaxSpareThreads        75
    ThreadsPerChild        25
    MaxRequestWorkers     150
    MaxConnectionsPerChild  0
</IfModule>
```

| Directive | Meaning | Guidance |
|---|---|---|
| `ThreadsPerChild` | Threads launched per worker process | Start at 25; increase to 50 for high-concurrency deployments. Higher values reduce process overhead but increase per-process memory. |
| `MaxRequestWorkers` | Total concurrent request ceiling across all processes | Must be a multiple of `ThreadsPerChild`. Set to the number of threads that can be sustained by available RAM and CPU. The default of 150 is conservative for production loads. |
| `MaxConnectionsPerChild` | Requests a single child process handles before recycling | Set to 0 (unlimited) for stable workloads. A nonzero value (e.g., 10000) mitigates slow memory leaks in any loaded modules. |

Calculate the process count: `MaxRequestWorkers / ThreadsPerChild`. With `MaxRequestWorkers=300` and `ThreadsPerChild=25`, OHS spawns up to 12 child processes.

Avoid setting `MaxRequestWorkers` so high that the system exhausts RAM. Each thread consumes memory for stack space and open connection buffers. Measure resident set size under load before increasing this value.

## KeepAlive Settings

HTTP keep-alive allows a single TCP connection to serve multiple requests, reducing connection setup overhead. For most OHS deployments fronting WebLogic, enabling keep-alive toward downstream clients is beneficial, but the timeout must be set conservatively to avoid holding threads idle.

```apacheconf
KeepAlive             On
MaxKeepAliveRequests  100
KeepAliveTimeout      5
```

| Directive | Default | Guidance |
|---|---|---|
| `KeepAlive` | On | Keep enabled for browser-facing traffic and REST API clients that reuse connections. |
| `MaxKeepAliveRequests` | 100 | Limit requests per connection to protect against any single client monopolizing a thread. 0 means unlimited. |
| `KeepAliveTimeout` | 5 | Seconds to wait for a subsequent request on a keep-alive connection. 2–5 seconds is appropriate for most deployments. Values above 15 waste threads when `mpm_worker` is used; with `mpm_event`, idle keep-alive connections are cheaper but a long timeout still wastes file descriptors. |

When `mpm_event` is used, `KeepAliveTimeout` governs how long the async listener holds an idle connection. Lower values free descriptors faster. For internal API proxies with predictable clients, 2 seconds is often sufficient.

## mod_weblogic Connection Pool Tuning

OHS proxies requests to WebLogic Server using `mod_weblogic` (the WebLogic Web Server Plugin). The plugin maintains its own internal connection pool to backend WebLogic Managed Server instances. Timeouts and pool sizing in `mod_weblogic` are independent of the MPM thread count.

Plugin parameters are set in `$ORACLE_INSTANCE/config/OHS/<component_name>/moduleconf/weblogic.conf` or inline in `httpd.conf` within a `<Location>` or `<VirtualHost>` block.

### Key mod_weblogic Parameters

```apacheconf
<Location /myapp>
    SetHandler   weblogic-handler
    WebLogicHost  wls-managed1.example.com
    WebLogicPort  7002

    # Connection and I/O timeouts
    WLIOTimeoutSecs        300
    WLSocketTimeoutSecs     10

    # Connection pool
    WLMaxSkipTime           10
    WLConnectTimeoutSecs    10
    WLServerDown             10
    Debug                  OFF
</Location>
```

| Parameter | Description | Guidance |
|---|---|---|
| `WLIOTimeoutSecs` | Time (seconds) the plugin waits for a response from WebLogic after sending a request | Set to match or slightly exceed the longest expected backend response time. Default is 300. Reducing this below your application SLA causes premature 503 errors. |
| `WLSocketTimeoutSecs` | Time (seconds) the plugin waits to establish a TCP connection to WebLogic | Default is 2. Increase to 10–15 in environments with network latency or slow WebLogic startup. A value that is too low causes spurious connection failures during GC pauses on the backend. |
| `WLConnectTimeoutSecs` | Timeout for initial plugin-to-WebLogic handshake | Set higher than `WLSocketTimeoutSecs` when SSL is in use between OHS and WebLogic, as the TLS handshake adds latency. |
| `WLMaxSkipTime` | Seconds a failed server stays marked "down" before retry | Default 10. Tune to balance fast recovery against retry storm risk. |
| `Debug` | Plugin debug logging level | Set to `OFF` in production. `DEBUG` and `ERR` modes write per-request log entries and measurably reduce throughput. |

### Connection Pool Sizing

The plugin does not expose a simple pool-size directive. The effective connection concurrency to a backend is bounded by `MaxRequestWorkers` on the OHS side and the WebLogic thread pool on the backend side. Ensure WebLogic's execute thread count is not a bottleneck when increasing OHS `MaxRequestWorkers`.

For clustered deployments, list all Managed Servers in the `WebLogicCluster` directive rather than a single `WebLogicHost`:

```apacheconf
WebLogicCluster  wls1.example.com:7002,wls2.example.com:7002,wls3.example.com:7002
```

The plugin load-balances across the cluster using round-robin with failure detection. Stale entries are retired based on `WLServerDown`.

## SSL Session Cache

SSL/TLS handshakes are expensive. Enabling the SSL session cache allows OHS to resume sessions without a full handshake, reducing CPU load and client connection latency.

OHS 12.2.1.4 uses `mod_ssl` with the following cache directives (in `ssl.conf` or `httpd.conf`):

```apacheconf
SSLSessionCache         shmcb:/tmp/ssl_scache(512000)
SSLSessionCacheTimeout  300
```

| Directive | Description | Guidance |
|---|---|---|
| `SSLSessionCache` | Shared memory session cache location and size | `shmcb` is the standard shared-memory ring buffer. Size (bytes) should accommodate the expected concurrent TLS session count. A 512 KB cache holds approximately 100–200 sessions depending on cipher suite. Increase to 1–4 MB for high-traffic HTTPS deployments. |
| `SSLSessionCacheTimeout` | Seconds a cached session remains valid | 300 seconds (5 minutes) is the Apache default. Clients that reconnect within this window skip the full handshake. Reduce to 60–120 for security-sensitive deployments; increase to 600 for long-lived API clients. |

Verify that `SSLSessionCache` uses `shmcb` rather than `dbm` in production. The `dbm` backend relies on file I/O and performs poorly under load.

## Access Log Buffering

By default, OHS writes an access log entry synchronously on every request. Under high request rates this becomes a measurable I/O bottleneck, particularly on shared or network-attached storage.

OHS 12.2.1.4 uses `mod_log_config`. Buffered logging is available via `BufferedLogs`:

```apacheconf
BufferedLogs On
```

When `BufferedLogs On` is set, log writes are batched in memory and flushed periodically rather than on each request. This reduces the number of write syscalls significantly at the cost of losing the last buffer on an ungraceful crash.

For high-volume deployments where the log flush latency is acceptable, enabling `BufferedLogs` is a straightforward throughput improvement. If exact per-request log timestamps are required for compliance or debugging, leave it disabled.

An alternative for extreme log rates is to log to a named pipe and have a consumer process write asynchronously:

```apacheconf
CustomLog "|/usr/bin/tee -a /var/log/ohs/access.log" combined
```

This approach decouples the OHS request thread from disk I/O entirely but adds process management complexity.

## Capacity Planning Guidance

### Estimating MaxRequestWorkers

1. Measure average request duration in milliseconds under representative load.
2. Determine the target requests-per-second (RPS) for peak traffic.
3. Apply Little's Law: `MaxRequestWorkers >= RPS * (avg_response_time_seconds)`.
   - Example: 500 RPS, 200 ms average response → minimum 100 concurrent threads needed at steady state.
4. Add a headroom factor of 30–50% above the calculated minimum to absorb traffic spikes.
5. Verify that the resulting process count (`MaxRequestWorkers / ThreadsPerChild`) fits within available RAM. Each OHS child process consumes 30–80 MB resident depending on loaded modules and SSL configuration.

### Thread and Process Limits

OHS inherits Apache's hard process limits from the OS. Verify:
- `ulimit -u` (max user processes) is set high enough for the process count.
- `ulimit -n` (max open files) is at least `MaxRequestWorkers * 2` plus overhead for log files and SSL descriptors.

### Backend Saturation

OHS queues requests in its `MaxRequestWorkers` pool but the real concurrency limit is often the WebLogic execute thread pool. Profile both layers under load:
- If OHS threads are idle while clients wait: the bottleneck is WebLogic threads or database connections.
- If OHS threads are all busy and requests are queuing at the OS level: increase `MaxRequestWorkers` or add OHS instances.

### Horizontal Scaling

OHS is stateless. Multiple OHS instances can be placed behind a load balancer without session affinity concerns at the HTTP layer. WebLogic cluster session replication handles backend state. Scale OHS horizontally before increasing `MaxRequestWorkers` beyond the practical per-instance ceiling (~500 threads per OHS instance on modern hardware under typical workloads).

## Best Practices and Common Mistakes

**Do:**
- Use `mpm_event` with a conservative `KeepAliveTimeout` (2–5 s) for internet-facing deployments.
- Pre-test `MaxRequestWorkers` changes under load; do not rely on defaults.
- Set `WLIOTimeoutSecs` to reflect actual application SLAs rather than an arbitrary large value.
- Enable `SSLSessionCache` for any HTTPS deployment handling more than a few dozen concurrent clients.
- Rotate and archive access logs externally; do not allow log growth to consume the OHS host filesystem.

**Avoid:**
- Running `mpm_prefork` in production; it cannot share memory between processes and consumes far more RAM per connection.
- Setting `KeepAliveTimeout` above 15 seconds on worker-model MPMs; idle threads block new connections.
- Setting `WLSocketTimeoutSecs` below 5 seconds in environments with Java GC pauses on WebLogic; short socket timeouts cause false 503s during minor GC events.
- Enabling `Debug ON` in `weblogic.conf` in production; it logs per-request plugin detail and measurably degrades throughput.
- Ignoring `ulimit` settings when scaling `MaxRequestWorkers`; the OS process and file descriptor limits are the actual ceiling.

## Oracle Version Notes

OHS 12.2.1.4 is part of the Oracle Fusion Middleware 12c (12.2.1.4) stack and ships with Apache HTTP Server 2.4. All directives documented here are standard Apache 2.4 directives or Oracle Web Server Plugin 12.2.1.4 parameters.

OHS 11g (11.1.1.x) used Apache 2.2 and the prefork MPM by default. The `mpm_event` module was not available in 11g. If managing an 11g instance, use `mpm_worker` and note that `BufferedLogs` was not available in Apache 2.2 — buffered logging is a 12c capability.

The WebLogic Web Server Plugin parameter names (`WLIOTimeoutSecs`, `WLSocketTimeoutSecs`, `WebLogicCluster`, etc.) are consistent across plugin versions 12.1.x and 12.2.x. Verify plugin version against the Oracle Fusion Middleware Certification Matrix before applying 12.2.1.4-specific guidance to older plugin installations.

## Sources

- Oracle HTTP Server Administrator's Guide 12c (12.2.1.4): https://docs.oracle.com/en/middleware/fusion-middleware/web-tier/12.2.1.4/administer-ohs/index.html
- Oracle Fusion Middleware Using Oracle HTTP Server 12c (12.2.1.4): https://docs.oracle.com/en/middleware/fusion-middleware/web-tier/12.2.1.4/use-ohs/index.html
- Oracle WebLogic Server Web Server Plugin Reference 12c (12.2.1.4): https://docs.oracle.com/en/middleware/fusion-middleware/web-tier/12.2.1.4/plugin-reference/index.html
- Apache HTTP Server 2.4 documentation — MPM directives: https://httpd.apache.org/docs/2.4/mod/mpm_common.html
- Apache HTTP Server 2.4 documentation — mod_ssl: https://httpd.apache.org/docs/2.4/mod/mod_ssl.html
- Oracle Fusion Middleware Certification Matrix: https://www.oracle.com/middleware/technologies/fusion-certification.html
