# Java Runtime Installation and Configuration (Fusion Middleware Context)

## Overview

This skill covers Java runtime installation and configuration for Oracle Fusion Middleware environments — WebLogic Server, SOA Suite, and Oracle GoldenGate. It addresses Oracle JDK versus OpenJDK selection for middleware use, silent install procedures, `JAVA_HOME` configuration, baseline JVM flag setup per product, minimum Java version requirements per middleware product, and pre-install compatibility verification.

Use this skill before provisioning or upgrading a middleware environment to validate that the Java runtime meets product requirements.

---

## Oracle JDK vs OpenJDK for Middleware

Oracle distributes two Java binaries:

- **Oracle JDK** — the Oracle-branded distribution. Requires an Oracle Java SE subscription for production use beyond the free tier. Binaries are available from https://www.oracle.com/java/technologies/downloads/.
- **Oracle OpenJDK** — the open-source reference implementation from https://jdk.java.net/. Free for all uses including production. No long-term patch stream is provided by Oracle for non-current releases; each new release supersedes the prior one.

For Oracle Fusion Middleware, Oracle certifies specific JDK distributions per product and version. The Oracle Fusion Middleware certification matrix (available on My Oracle Support) distinguishes between Oracle JDK and Oracle OpenJDK certifications. Always consult that matrix before selecting a distribution.

Key rules:
- Oracle Fusion Middleware products primarily certify against **Oracle JDK**. Use Oracle JDK for production middleware unless the certification matrix explicitly lists Oracle OpenJDK.
- Oracle JDK 17 and Oracle JDK 21 are both LTS releases. Use one of these as the baseline unless your product version requires Java 8 or 11 (older middleware) or restricts to Java 25 (future).
- Never use a non-LTS Java version (e.g., Java 18, 19, 20, 22, 23, 24) for production middleware. These receive no long-term patch support.

---

## Minimum Java Version Requirements by Middleware Product

The following table summarizes the minimum certified Java version per Oracle Fusion Middleware product line. Consult the Oracle Fusion Middleware Supported System Configurations page for the exact current matrix before installation.

| Middleware Product | Minimum Java Version | Notes |
|---|---|---|
| WebLogic Server 12.2.1.4 | Java 8 (update 211+) | Java 11 also certified. Java 17 not supported on 12.2.1.4. |
| WebLogic Server 14.1.1 | Java 8 | Java 11 and Java 17 also certified. |
| WebLogic Server 14.1.2 | Java 17 | Java 21 also certified. Java 8/11 not supported. |
| WebLogic Server 15c | Java 21 | Java 25 planned. Java 17 also supported. |
| Oracle SOA Suite 12.2.1.4 | Java 8 | Java 11 also certified on current patch sets. |
| Oracle SOA Suite 14.1.2 | Java 17 | Aligns with WebLogic 14.1.2 requirement. |
| Oracle GoldenGate 21c | Java 8 or 11 | Required for Microservices Architecture (MA) server; verify the exact patch set. |
| Oracle GoldenGate 23ai | Java 17 | Java 21 also certified. |

Always verify against the current Certification Matrix at: https://www.oracle.com/middleware/technologies/fusion-certification.html

---

## Verifying Java Version Compatibility Before Middleware Install

Before running a middleware installer, confirm the following:

### 1. Check installed Java version and vendor

```bash
java -version
# Expected output example (Oracle JDK 17):
# java version "17.0.x" 2024-xx-xx LTS
# Java(TM) SE Runtime Environment (build 17.0.x+...)
# Java HotSpot(TM) 64-Bit Server VM (build 17.0.x+..., mixed mode, sharing)
```

The vendor string (`Java(TM) SE Runtime Environment` vs `OpenJDK Runtime Environment`) confirms whether you are running Oracle JDK or OpenJDK.

### 2. Confirm JAVA_HOME resolves to the correct JDK

```bash
echo $JAVA_HOME
$JAVA_HOME/bin/java -version
```

Both commands must point to the same JDK that the middleware installer will use. Mismatches between `$JAVA_HOME` and the system `java` command in `PATH` are a common source of install failures.

### 3. Confirm the JDK is 64-bit

```bash
java -d64 -version 2>&1 | grep -i "64-bit"
# If the JVM is 64-bit, the output should include "64-Bit Server VM"
# On Java 11+, the -d64 flag is removed; look for "64-Bit" in java -version output directly
```

Oracle Fusion Middleware requires a 64-bit JVM. 32-bit JVMs are not supported.

### 4. Cross-reference with the middleware certification matrix

Use the `java -version` output to confirm:
- The major version matches the certified range for your middleware product (see the table above).
- The update/patch level meets the minimum required (e.g., Java 8 update 211 or later for WebLogic 12.2.1.4).
- The JDK vendor matches the certified distribution (Oracle JDK vs OpenJDK).

---

## Silent Install of Oracle JDK (Linux)

Oracle JDK for Linux is distributed as a `.tar.gz` archive for manual installation, or as an RPM or DEB package. For middleware environments, the `.tar.gz` method is most portable.

### Step 1: Download the JDK archive

Download the appropriate JDK release from https://www.oracle.com/java/technologies/downloads/. For example, Oracle JDK 17 LTS:

```bash
# Example filename — actual filename varies by release and update number
# Verify the SHA-256 checksum after download
sha256sum jdk-17_linux-x64_bin.tar.gz
```

Compare the checksum against the value published on the Oracle download page before proceeding.

### Step 2: Extract and place the JDK

```bash
# Extract to a stable, permanent location — not /tmp
sudo mkdir -p /opt/oracle/java
sudo tar -xzf jdk-17_linux-x64_bin.tar.gz -C /opt/oracle/java

# Confirm extracted directory name
ls /opt/oracle/java
# e.g., jdk-17.0.x
```

Do not place the JDK under a directory that is managed by a package manager unless the JDK was installed via that package manager.

### Step 3: Set JAVA_HOME and update PATH

Set `JAVA_HOME` in the OS user profile that will run the middleware installation and runtime processes. For the `oracle` OS user:

```bash
# Add to ~/.bash_profile or the middleware service user's profile
export JAVA_HOME=/opt/oracle/java/jdk-17.0.x
export PATH=$JAVA_HOME/bin:$PATH
```

After editing, source the profile and verify:

```bash
source ~/.bash_profile
java -version
echo $JAVA_HOME
```

For middleware services managed by systemd, set `JAVA_HOME` in the unit environment file or the middleware domain environment script — not only in the user shell profile.

### Step 4: Verify no conflicting Java in PATH

```bash
which java
# Must resolve to $JAVA_HOME/bin/java, not a system-managed alternative
```

If the output does not match `$JAVA_HOME/bin/java`, a system-level `java` alternative (e.g., from `update-alternatives` on RPM-based Linux) is taking precedence. Either update the alternative or ensure `$JAVA_HOME/bin` appears first in `PATH`.

---

## JAVA_HOME Configuration for Middleware Domains

### WebLogic Server

WebLogic reads `JAVA_HOME` from the domain environment script. Explicitly set it rather than relying on shell inheritance:

```bash
# In $DOMAIN_HOME/bin/setDomainEnv.sh
# Find and override the JAVA_HOME variable before the existing assignment block
# Use the exact path to the certified JDK — do not use a symlink that could resolve differently after a JDK update

export JAVA_HOME=/opt/oracle/java/jdk-17.0.x
export JAVA_VENDOR=Oracle
```

The `JAVA_VENDOR` variable controls which JVM options WebLogic enables by default. Set it to `Oracle` when using Oracle JDK. Valid values include `Oracle`, `BEA`, and `Sun` for Oracle/Sun JVMs.

Do not modify `setDomainEnv.sh` directly in the WebLogic installation home (`$WL_HOME`). Always modify the copy in `$DOMAIN_HOME/bin/`.

### SOA Suite

SOA Suite runs within a WebLogic domain. Java configuration follows the WebLogic domain pattern above. Ensure the `JAVA_HOME` set in `$DOMAIN_HOME/bin/setDomainEnv.sh` matches the Java version certified for the installed SOA Suite version.

### GoldenGate Microservices Architecture

Oracle GoldenGate 21c and later use Java for the Microservices Architecture server components. Set `JAVA_HOME` before starting GoldenGate services:

```bash
# In the GoldenGate service manager environment
export JAVA_HOME=/opt/oracle/java/jdk-17.0.x
export PATH=$JAVA_HOME/bin:$PATH
```

GoldenGate reads `JAVA_HOME` at service startup. An incorrect or missing `JAVA_HOME` causes the Service Manager to fail to start the distribution, replication, and monitoring services.

---

## JVM Flag Baseline for Middleware Contexts

### General Principles

- Use the minimal set of JVM flags needed. Avoid copying tuning flags from older installations without reviewing their effect on the current JDK version.
- After each JDK major version upgrade, validate all existing JVM flags. Removed or renamed flags cause `Unrecognized option` errors at startup.
- Flags that control JVM behavior must be set in the middleware startup script or domain environment — not in a user shell profile.

### WebLogic Server JVM Baseline (Java 17)

A safe starting set for WebLogic 14.1.2 Managed Servers on Java 17:

```bash
# Heap sizing — set -Xms equal to -Xmx in production to prevent resize pauses
-Xms2048m -Xmx2048m

# Metaspace — set an explicit cap to prevent unbounded native memory growth
-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m

# Garbage collector — G1GC is the default on Java 9+; explicit declaration aids documentation
-XX:+UseG1GC
-XX:MaxGCPauseMillis=500

# GC logging (Java 9+ unified logging syntax — do not use -XX:+PrintGCDetails on Java 9+)
-Xlog:gc*:file=$LOG_DIR/gc.log:time,uptime:filecount=5,filesize=20m

# Module accessibility — add only as required after auditing with jdeps
# --add-opens java.base/java.lang=ALL-UNNAMED  # Example — include only if needed
```

Do not include `--add-opens` broadly without first identifying the specific module boundary violation using `jdeps --jdk-internals`. See `troubleshooting.md` for diagnostics.

### SOA Suite JVM Baseline (Java 17)

SOA Suite processes large XML payloads. Additions to the WebLogic baseline:

```bash
# Larger heap for XML-heavy BPEL and Mediator workloads
-Xms4096m -Xmx4096m

# Increase Metaspace cap — SOA hot-deploys can cause classloader accumulation
-XX:MetaspaceSize=512m -XX:MaxMetaspaceSize=1024m
```

Tune from measured GC log data; the values above are starting points only.

### GoldenGate JVM Baseline (Java 17)

GoldenGate Microservices server components typically require less heap than WebLogic:

```bash
-Xms512m -Xmx1024m
-XX:+UseG1GC
-Xlog:gc*:file=$GG_LOG_DIR/gc.log:time,uptime:filecount=3,filesize=10m
```

Refer to the GoldenGate Installation and Configuration Guide for the current release for product-specific JVM options.

### Flags Removed at Key JDK Boundaries

Do not carry these forward when upgrading:

| Removed Flag | Removed In | Replacement |
|---|---|---|
| `-XX:MaxPermSize` / `-XX:PermSize` | Java 8 | `-XX:MaxMetaspaceSize` |
| `-XX:+UseConcMarkSweepGC` | Java 14 | `-XX:+UseG1GC` or `-XX:+UseZGC` |
| `-XX:+PrintGCDetails` | Java 9 (replaced) | `-Xlog:gc*` |
| `-XX:+PrintGCDateStamps` | Java 9 (replaced) | `-Xlog:gc*:time` |
| `--illegal-access` | Java 17 (removed in 17) | `--add-opens` (targeted) |

---

## Best Practices and Common Mistakes

**Do:**
- Always verify the JDK version, vendor, and bitness before running a middleware installer.
- Set `JAVA_HOME` explicitly in the domain startup script, not only in the user shell.
- Use a permanent, non-symlinked JDK path in `JAVA_HOME` to avoid unintended runtime changes after a JDK update.
- Review all JVM flags after each JDK major version upgrade for removed or deprecated options.
- Use LTS Java releases (8, 11, 17, 21, 25) for production middleware. Non-LTS releases have short support windows.
- Leave at least 25% of host RAM unallocated to JVM heap to allow for OS, off-heap buffers, and other processes.

**Do not:**
- Do not set `JAVA_HOME` to a JRE directory. Middleware requires the full JDK (`javac`, `jcmd`, `jstack`, and other tools must be present).
- Do not use the system-default JVM from the OS package manager unless it matches the certified JDK distribution and version.
- Do not carry JVM flags from one major JDK version to the next without revalidation. Many flags have been removed since Java 8.
- Do not set `-Xss` (thread stack size) very low for WebLogic or SOA Suite; the default (512 KB or 1 MB) is appropriate for most workloads. Low `-Xss` values cause `StackOverflowError` in deep XML processing stacks.
- Do not mix JDK vendors (Oracle JDK and OpenJDK) across nodes in the same middleware cluster. Behavioral differences between vendors can create subtle runtime inconsistencies.

---

## Oracle Version Notes (19c vs 26ai)

This section covers Java version considerations when the middleware environment connects to Oracle Database.

- **Oracle Database 19c**: JDBC Thin driver from the 19c release (`ojdbc8.jar`) is certified for Java 8 through Java 17 connections. Use `ojdbc11.jar` for Java 11+ connections when available for your patch set. Middleware connecting to a 19c database must use a certified JDBC driver version — do not mix an old `ojdbc8.jar` with a Java 17 JVM without verifying compatibility.
- **Oracle Database 21c**: Requires JDBC driver `ojdbc11.jar` or later for Java 11+ connections. Java 8 connections can use `ojdbc8.jar`.
- **Oracle Database 23ai / 26ai**: JDBC driver updates are required. Validate the JDBC driver version separately from the JVM version. The database client-side JDBC driver must be certified for both the database server version and the JVM major version.
- **Java 8 end of Oracle public updates** has already passed for most licensees. Estates still running WebLogic 12.2.1.4 on Java 8 should plan an upgrade path. WebLogic 14.1.1 supports Java 17, providing an upgrade path that retains Oracle Database 19c compatibility.
- **Java 17 LTS** is the current recommended baseline for new Fusion Middleware installations targeting WebLogic 14.1.2, SOA Suite 14.1.2, and GoldenGate 23ai.
- **Java 21 LTS** is certified for WebLogic 14.1.2 and WebLogic 15c. It introduces virtual threads (`--enable-preview` required in Java 21; stable in Java 21 GA for server-side usage) — do not enable virtual thread frameworks in WebLogic unless explicitly supported and tested.
- **Java 25 LTS** (GA 2025-09-16) is the latest LTS. WebLogic 15c is the planned target for Java 25 support. Verify certification before adopting.

---

## Sources

- Oracle Java SE Downloads: https://www.oracle.com/java/technologies/downloads/
- Oracle Java SE Support Roadmap: https://www.oracle.com/java/technologies/java-se-support-roadmap.html
- Oracle Fusion Middleware Supported System Configurations: https://www.oracle.com/middleware/technologies/fusion-certification.html
- WebLogic Server 14.1.2 Installation Guide: https://docs.oracle.com/en/middleware/standalone/weblogic-server/14.1.2/wlsig/index.html
- WebLogic Server 14.1.2 System Requirements: https://docs.oracle.com/en/middleware/standalone/weblogic-server/14.1.2/sysrs/index.html
- Oracle SOA Suite 14.1.2 Installation Guide: https://docs.oracle.com/en/middleware/fusion-middleware/soa-suite/14.1.2/soaig/index.html
- Oracle GoldenGate 23ai Microservices Installation Guide: https://docs.oracle.com/en/middleware/goldengate/core/23/gimig/index.html
- JDK 17 Migration Guide: https://docs.oracle.com/en/java/javase/17/migrate/index.html
- JDK 21 Migration Guide: https://docs.oracle.com/en/java/javase/21/migrate/index.html
- Oracle Database JDBC Developer's Guide 19c: https://docs.oracle.com/en/database/oracle/oracle-database/19/jjdbc/index.html
- Oracle Database JDBC Developer's Guide 23ai: https://docs.oracle.com/en/database/oracle/oracle-database/23/jjdbc/index.html
- JEP 403 (Strong Encapsulation of JDK Internals): https://openjdk.org/jeps/403
