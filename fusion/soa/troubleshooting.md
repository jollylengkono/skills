# Oracle SOA Suite Troubleshooting

## Overview

Use this skill to diagnose and resolve Oracle SOA Suite runtime issues across SOA 11g through 14.1.2. Load `references/soa-version-product-matrix.md` when troubleshooting requires version-specific product coverage.

## Fast Triage Workflow

1. Identify the affected component: BPEL, Mediator, Oracle Service Bus (OSB), BPM, B2B, or MFT.
2. Check Enterprise Manager Fusion Middleware Control for composite and service status.
3. Locate the relevant server log: `<domain>/servers/<ManagedServer>/logs/<ManagedServer>.log`.
4. Classify the failure: instance-level error, system-level infrastructure (JMS, JDBC), or configuration issue.

## BPEL Process Instance Failures

Common causes:
- Invoke activity timeout (remote service unavailable or slow to respond)
- Fault propagation not handled by fault handlers in the composite
- Correlation key failures (missing or mismatched key values)
- Database errors during dehydration to the SOAINFRA schema

Diagnostic steps:
- Enterprise Manager > SOA > SOA Infrastructure > Flow Instances — filter by fault state
- Check the audit trail for the failed instance to identify the faulting activity
- Query the dehydration store for recent faults: `SELECT * FROM CUBE_INSTANCE WHERE STATE = 4 AND MODIFIED_DATE > SYSDATE - 1/24` (SOAINFRA schema)
- Verify the invoke endpoint is reachable from the SOA Managed Server host

## Mediator Routing Failures

Common causes:
- Routing rule condition evaluation failure
- Assignment expression errors (XSLT or XQuery)
- Target endpoint connectivity failure
- Missing or misconfigured binding component

Diagnostic steps:
- Enterprise Manager > SOA > SOA Infrastructure > Flow Instances — select the Mediator component
- Check the Mediator trace to identify which routing rule was evaluated
- Verify WSDL and binding component definitions in the SOA composite
- Check SOA infrastructure JMS queues for stuck or undelivered messages

## Oracle Service Bus (OSB) Issues

Common causes:
- Business service endpoint unavailability
- Pipeline stage script or XQuery expression errors
- JCA adapter faults propagating from the transport layer
- Throttling policy activation limiting inbound request rate

Diagnostic steps:
- OSB Console > Operations > Service Health — review error counts and response times
- Enable OSB message tracing at the admin level for targeted diagnosis; disable after collection
- Check pipeline alert rules if alert notifications were triggered
- Review OSB runtime errors in the Managed Server log

## JMS and Adapter Issues

- Verify JMS connection factory is active: Admin Console > Services > Messaging > JMS Servers
- Check JMS queue depths: Admin Console > Services > Messaging > JMS Modules > Queues > Monitoring
- For JCA adapters (DB Adapter, File Adapter, AQ Adapter): check adapter endpoint state in EM and review adapter-specific log entries
- Adapter deployment failures: check `<domain>/servers/<name>/logs/` for adapter initialization errors at startup

## MDS Metadata Issues

Common causes:
- Missing shared artifacts (WSDL, schema) in the MDS repository
- Application-specific and shared MDS partition conflicts
- MDS consistency check failures after patching or redeployment

Diagnostic steps:
- Review `<domain>/servers/<name>/logs/` for `MDS-` prefixed error messages
- Use `exportMetadata` and `importMetadata` WLST commands to inspect and restore MDS artifacts
- Verify MDS schema (`<prefix>_MDS`) tablespace availability and JDBC connectivity

## Log File Locations

| Component | Default log path |
|---|---|
| SOA Managed Server | `<domain>/servers/<name>/logs/<name>.log` |
| Admin Server | `<domain>/servers/AdminServer/logs/AdminServer.log` |
| SOA infrastructure log | Enterprise Manager > SOA > Logs |
| OSB runtime log | SOA Managed Server log |

## Oracle Version Notes (19c vs 26ai)

- For estates on Oracle Database 19c: SOA infra schema queries and dehydration store connections use standard JDBC settings.
- For 26ai database targets: validate SOA infrastructure JDBC connection compatibility before migrating the database independently.
- Keep SOA infrastructure schema (SOAINFRA, MDS) on a validated and supported database version; consult Oracle certification before upgrading the database independently of SOA.

## Sources

- `references/soa-version-product-matrix.md`
- Oracle SOA Suite documentation: https://docs.oracle.com/en/middleware/soa-suite/
- SOA Suite 11g documentation: https://www.oracle.com/middleware/technologies/soa-suite-11gdocumentation.html
- Oracle Middleware certification: https://www.oracle.com/middleware/technologies/fusion-certification.html
