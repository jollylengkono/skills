# Java Runtime Patching and Upgrade (Fusion Middleware Context)

## Overview

Use this skill to plan and execute Java runtime patching and version upgrades for Oracle Fusion Middleware estates (WebLogic, SOA Suite, GoldenGate). It covers Oracle JDK Critical Patch Update (CPU) application, in-place vs parallel-install patching strategies, LTS-to-LTS upgrade planning from Java 8 through Java 25, module system impact when crossing the Java 8/9 boundary, middleware stack re-validation, and rollback procedures.

For Java version timeline and LTS planning, load `references/java-version-matrix.md`. For GC-specific tuning after an upgrade, load `references/java-gc-reference.md`.

---

## Core Concepts

### Oracle JDK CPU Patching

Oracle releases Critical Patch Updates (CPUs) on a quarterly schedule in January, April, July, and October. Each CPU for the Java SE platform is a cumulative patch that includes all prior CPU fixes for that major release line. Oracle JDK CPU patches address security vulnerabilities and critical stability fixes.

Key facts:
- CPU patches are release-line-specific: a Java 17 CPU patch does not apply to a Java 21 installation.
- For LTS releases under an Oracle Java SE subscription, CPU patches are available for the full support window of the release. See the Oracle Java SE Support Roadmap for end-of-support dates.
- CPU patch releases for Java SE are identified by their version string (e.g., `17.0.11`, `21.0.3`). The minor and patch numbers encode the CPU quarter.

### In-Place JDK Patch vs Parallel Install

Two approaches exist for applying a CPU patch to a running middleware host:

**In-place patch** replaces the existing JDK binaries at `$JAVA_HOME` with the new CPU release.
- The middleware process must be stopped before the JDK directory is modified.
- The existing `$JAVA_HOME` path remains valid; no startup script changes are required.
- Rollback requires keeping a backup of the pre-patch JDK directory.

**Parallel install** installs the new JDK to a different directory and switches `$JAVA_HOME` in the middleware startup scripts to point to the new location.
- The old JDK installation remains untouched until validated.
- Rollback is a pointer change: revert `$JAVA_HOME` in the startup scripts.
- Required when the host runs multiple middleware components that cannot all be stopped simultaneously.
- Preferred approach for major version upgrades (e.g., Java 17 to Java 21) where re-validation is needed before committing.

### LTS-to-LTS Upgrade Path

The recommended upgrade sequence for enterprise middleware estates is:

```
Java 8 -> Java 11 -> Java 17 -> Java 21 -> Java 25
```

Non-LTS feature releases (9, 10, 12–16, 18–20, 22–24) are not targets for long-lived middleware deployments. The Java 8-to-11 hop is the highest-risk transition because it crosses the Java 9 module system boundary. Each subsequent LTS-to-LTS hop introduces incremental encapsulation enforcement but is less disruptive than the initial Java 9 boundary crossing.

---

## Practical Examples

### Example 1: Applying a Quarterly CPU Patch (In-Place)

**Scenario:** A WebLogic 14.1.2 host runs Java 17.0.8. Oracle has released 17.0.11 as the latest CPU. Apply the patch in-place.

**Pre-flight checks**

1. Identify the current JDK version and home:
   ```bash
   java -version
   echo $JAVA_HOME
   ```

2. Confirm the middleware version's certified Java version range in the Oracle Fusion Middleware Certification Matrix before patching. A CPU update within the same major release (e.g., 17.0.8 to 17.0.11) is generally safe; validate the certification matrix if the minor version changes.

3. Take a backup of the current JDK directory:
   ```bash
   cp -a $JAVA_HOME ${JAVA_HOME}_backup_17.0.8
   ```

**Apply the patch**

4. Download the Oracle JDK 17.0.11 CPU release from oracle.com/java/technologies/downloads/ for your platform.

5. Stop all middleware processes that use this JDK installation:
   ```bash
   # Example for WebLogic
   $DOMAIN_HOME/bin/stopManagedWebLogic.sh ManagedServer1
   $DOMAIN_HOME/bin/stopWebLogic.sh
   ```

6. Extract the new JDK to the same `$JAVA_HOME` path (overwrite):
   ```bash
   # For a .tar.gz archive
   tar -xzf jdk-17.0.11_linux-x64_bin.tar.gz --strip-components=1 -C $JAVA_HOME
   ```

7. Verify the replacement succeeded:
   ```bash
   java -version
   # Expected: openjdk version "17.0.11" or "java version "17.0.11""
   ```

8. Start middleware and confirm normal operation. Review startup logs for JVM warnings before declaring success.

**Rollback (in-place)**

If startup or smoke testing fails:

```bash
# Remove the patched JDK and restore the backup
rm -rf $JAVA_HOME
mv ${JAVA_HOME}_backup_17.0.8 $JAVA_HOME
```

Restart middleware and confirm the previous version is active:
```bash
java -version
```

---

### Example 2: Applying a CPU Patch Using Parallel Install

**Scenario:** A host runs both a WebLogic Admin Server and a GoldenGate microservices deployment. They cannot be stopped simultaneously during a maintenance window. Use parallel install so each component can be migrated independently.

**Steps**

1. Install the new JDK CPU release to a new directory alongside the existing installation:
   ```bash
   tar -xzf jdk-21.0.3_linux-x64_bin.tar.gz -C /opt/oracle/jdk
   # New path: /opt/oracle/jdk/jdk-21.0.3
   ```

2. Verify the new installation:
   ```bash
   /opt/oracle/jdk/jdk-21.0.3/bin/java -version
   ```

3. For each middleware component in scope, update the startup script or environment file to point to the new JDK:
   - WebLogic: update `$DOMAIN_HOME/bin/setDomainEnv.sh` — change `JAVA_HOME` export.
   - GoldenGate microservices: update the `$OGG_HOME/bin/oggenv.sh` or the systemd unit file `Environment=JAVA_HOME=...`.

4. Stop, reconfigure, and restart each component one at a time within the maintenance window.

5. After all components are validated on the new JDK, decommission the old JDK directory on a subsequent maintenance cycle.

**Rollback (parallel install)**

Revert the `JAVA_HOME` reference in the affected startup script to the old path and restart the component. The old JDK directory is untouched.

---

### Example 3: LTS-to-LTS Upgrade — Java 8 to Java 11

The Java 8-to-11 upgrade is the highest-risk transition in an enterprise middleware estate because it crosses the Java 9 Platform Module System (JPMS) boundary. Treat this as a project, not a patch.

**Phase 1: Inventory and dependency scan**

1. Identify all JARs in the application and middleware classpath.
2. Run `jdeps` against application JARs to find internal JDK API usage:
   ```bash
   jdeps --jdk-internals --multi-release 11 myapp.jar
   ```
   Any `JDK internal API` lines in the output must be addressed before moving to Java 11 in production.

3. Note that the following Java EE and CORBA modules were removed in Java 11 (JEP 320). If your middleware deployment uses any of these, they must be sourced from a standalone library:
   - `java.corba` (RMI-IIOP, ORB)
   - `java.xml.bind` (JAXB)
   - `java.xml.ws` (JAX-WS)
   - `java.activation` (JAF)
   - `java.transaction` (JTA)
   - `java.xml.ws.annotation`

   WebLogic 12.2.1.4 and 14.x bundle standalone replacements for these APIs. Confirm the installed WebLogic version ships or supports the standalone versions before upgrading the JDK.

4. Check for any use of `sun.*` or `com.sun.*` internal APIs in application code. These are not guaranteed to be accessible under the Java 9+ strong encapsulation model.

**Phase 2: Pre-upgrade validation in a non-production environment**

5. Install Java 11 in parallel on a test host.
6. Start WebLogic or SOA with Java 11 and capture the full startup log. Look for:
   - `WARNING: An illegal reflective access operation has occurred` — permitted in Java 11 but blocked in Java 17+.
   - `Error: Could not find or load main class` — classloading or module path issue.
   - `IllegalAccessError` or `InaccessibleObjectException` — module boundary violation.

7. Run functional smoke tests against all critical SOA composites, JCA adapters, and WebLogic deployments.

**Phase 3: Production cutover using parallel install**

8. Install Java 11 to a new directory on production hosts.
9. Update `$JAVA_HOME` in the domain startup scripts.
10. Coordinate a maintenance window to restart all middleware servers in the domain.
11. Monitor GC logs and startup for the first 24 hours — see `references/java-gc-reference.md` for CMS-to-G1GC migration notes relevant to this upgrade.

**JVM flag cleanup mandatory at this transition**

Remove or replace these flags before starting under Java 11:

| Obsolete flag | Replacement or action |
|---|---|
| `-XX:MaxPermSize` / `-XX:PermSize` | Remove; use `-XX:MaxMetaspaceSize` |
| `-XX:+PrintGCDetails` | Replace with `-Xlog:gc*:file=<path>` |
| `-XX:+PrintGCDateStamps` | Covered by `-Xlog:gc*` unified logging |
| `-Djava.endorsed.dirs` | Removed; remove the flag |
| `-Djava.ext.dirs` | Removed; remove the flag |

Starting under Java 11 with obsolete flags produces `Unrecognized VM option` errors that prevent startup.

---

### Example 4: LTS-to-LTS Upgrade — Java 17 to Java 21 (or Java 21 to Java 25)

Upgrades within the Java 17+ LTS line are lower risk than the Java 8/9 boundary crossing but still require review.

**Steps**

1. Check the Oracle Fusion Middleware Certification Matrix to confirm the target Java version is certified for your middleware release.

2. Review the Oracle JDK migration guide for the target version. Key areas for Java 17-to-21:
   - Virtual threads (Project Loom, JEP 444): no impact unless application code uses platform thread assumptions.
   - Sequenced collections (JEP 431): additive API change, no breakage.
   - Strong encapsulation: `--illegal-access` was removed in Java 17; any remaining `--add-opens` flags should be audited for necessity.

3. Audit all `--add-opens` and `--add-exports` JVM flags in the middleware startup scripts. With each LTS upgrade, verify each one against the Oracle documentation for the specific middleware release. Remove those no longer needed; retaining unnecessary `--add-opens` flags increases attack surface.

4. Run startup validation in a non-production environment and check for `WARNING: Using obsolete option` or other JVM diagnostic messages.

5. Apply using parallel install and phase production cutover across the domain.

---

### Example 5: Middleware Stack Re-Validation After Java Upgrade

After any Java upgrade — whether a CPU patch or a major version upgrade — validate the middleware stack before considering the upgrade complete.

**Minimum re-validation checklist**

| Check | How |
|---|---|
| JVM starts without errors or warnings | Review full startup log of AdminServer and each Managed Server |
| GC log is being written | Confirm GC log file is being populated |
| WebLogic console accessible | Log in; check all deployed applications are in ACTIVE state |
| SOA composites are deployed | Log in to EM Fusion Middleware Control; confirm composite states |
| GoldenGate Extract/Replicat resume | Check trail file processing after restart |
| JDBC connectivity | Run a test query from the middleware data source; check datasource health in the console |
| JCA adapters operational | Trigger a test integration flow or confirm adapter connections are not in error state |
| Application smoke test | Run a representative business transaction end-to-end |

A Java CPU patch within the same major version is lower risk; a major version upgrade warrants a full re-validation cycle.

---

## Best Practices and Common Mistakes

**Best practices**

- Always back up the current JDK directory before an in-place patch, or retain the previous parallel install until re-validation is complete.
- Treat major version upgrades (crossing any LTS boundary) as a change project with a non-production validation phase, not as a routine patch.
- Keep the JDK upgrade and middleware configuration upgrade as separate changes; do not combine a Java upgrade with a WebLogic patch or SOA upgrade in the same maintenance window.
- Maintain a record of the JVM flag set before and after each upgrade. Review every flag against the JDK release notes at each major version boundary.
- Consult the Oracle Fusion Middleware Certification Matrix before selecting a Java version target. Not all Java releases are certified for all WebLogic and SOA releases.
- Use parallel install when multiple components share a host and cannot be stopped simultaneously, or when a longer validation window is needed before committing.

**Common mistakes**

- Carrying obsolete JVM flags forward without review. Flags removed in Java 9 (`-XX:MaxPermSize`, `-Djava.ext.dirs`) and Java 14 (`-XX:+UseConcMarkSweepGC`) cause startup failures.
- Skipping `jdeps` analysis before a Java 8-to-11 upgrade. Internal JDK API usage that worked in Java 8 will fail silently or throw `IllegalAccessError` under Java 11+.
- Upgrading middleware components and JDK in the same window. If a problem occurs, there are two variables to diagnose. Change one at a time.
- Not checking the Oracle Fusion Middleware Certification Matrix first. Some WebLogic patch sets or SOA releases have a maximum certified Java version; installing a newer JDK may be unsupported until a corresponding middleware patch is applied.
- Neglecting to validate JDBC driver compatibility after a major Java version upgrade. The ojdbc driver version must be compatible with both the Java version and the Oracle Database version in use.
- Using broad `--add-opens` flags as a shortcut around module errors without identifying the root cause. This masks problems and should only be a temporary measure while the underlying code is fixed or a middleware patch is obtained.

---

## Oracle Version Notes (19c vs 26ai)

This section applies when Java upgrade planning intersects with Oracle Database version planning.

**Java LTS vs feature release in an enterprise estate**

- LTS releases (8, 11, 17, 21, 25) are the correct long-term targets for Oracle Fusion Middleware. Feature-line releases (e.g., 22, 24, 26) receive six months of support from Oracle; they are not suitable for middleware deployments that require multi-year stability.
- Java 25 is the latest LTS as of the date of this document. Java 26 is the current feature release. Plan upgrades toward Java 25, not Java 26, unless a specific feature in the release line is required and the estate can accept the shorter support window.

**Oracle Database baseline (19c)**

- Oracle Database 19c is the baseline compatibility target for most Oracle Fusion Middleware deployments. The ojdbc8 and ojdbc11 JDBC driver versions are certified against Oracle Database 19c and later.
- When upgrading the Java runtime, validate that the JDBC driver version in the middleware classpath is certified for both the new Java version and the database version in use. See the Oracle JDBC Driver Certification Matrix at https://www.oracle.com/database/technologies/appdev/jdbc-downloads.html.
- Java runtime upgrades and database upgrades are separate tracks. Do not combine them in the same maintenance window. Reconcile compatibility at integration and cutover testing.

**Oracle Database 26ai targets**

- For estates targeting Oracle Database 26ai, verify JDBC driver compatibility with the target database version separately from JVM upgrade planning.
- The JDBC driver must be certified for the target database release. Check https://www.oracle.com/database/technologies/appdev/jdbc-downloads.html for the current certification matrix.

---

## Sources

- Oracle Java SE Support Roadmap: https://www.oracle.com/java/technologies/java-se-support-roadmap.html
- Oracle Java SE Downloads (CPU releases): https://www.oracle.com/java/technologies/downloads/
- Oracle JDK 11 Migration Guide: https://docs.oracle.com/en/java/javase/11/migrate/index.html
- Oracle JDK 17 Migration Guide: https://docs.oracle.com/en/java/javase/17/migrate/index.html
- Oracle JDK 21 Migration Guide: https://docs.oracle.com/en/java/javase/21/migrate/index.html
- Oracle JDK 26 Migration Guide: https://docs.oracle.com/en/java/javase/26/migrate/index.html
- JEP 261 — Module System (Java 9): https://openjdk.org/jeps/261
- JEP 320 — Remove Java EE and CORBA Modules (Java 11): https://openjdk.org/jeps/320
- JEP 396 — Strongly Encapsulate JDK Internals by Default (Java 16): https://openjdk.org/jeps/396
- JEP 403 — Strongly Encapsulate JDK Internals (Java 17, final): https://openjdk.org/jeps/403
- Oracle Fusion Middleware Supported System Configurations (Certification Matrix): https://www.oracle.com/middleware/technologies/fusion-certification.html
- Oracle JDBC Driver Downloads and Certification Matrix: https://www.oracle.com/database/technologies/appdev/jdbc-downloads.html
- Oracle Java Release History: https://www.java.com/en/releases/
- jdeps tool documentation (JDK 21): https://docs.oracle.com/en/java/javase/21/docs/specs/man/jdeps.html
