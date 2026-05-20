---
name: oci
description: Oracle Cloud Infrastructure domain routing for landing-zone architecture, IAM/security guardrails, and networking operations troubleshooting. Use when tasks involve OCI tenancy foundation design, compartment and policy strategy, security control rollout (Cloud Guard, Security Zones, KMS), or OCI networking triage in landing-zone deployments.
---

# Oracle Cloud Infrastructure Skills

This domain provides OCI guidance with emphasis on landing-zone foundations and day-2 operations.
Use the routing table to select the narrowest OCI topic reference.

## How to Use This Domain

1. Identify whether the task is architecture, IAM/security guardrails, or networking operations.
2. Open only the needed reference file.
3. Validate final design/support decisions against current OCI documentation and platform constraints.

## Directory Structure

```text
oci/
├── SKILL.md
└── references/
    ├── landing-zone-core.md
    ├── iam-security-guardrails.md
    └── networking-operations.md
```

## Category Routing

| Topic | Path |
|---|---|
| OCI Core Landing Zone structure, deployment flow, and customization points | `oci/references/landing-zone-core.md` |
| IAM model, compartment strategy, Cloud Guard/Security Zones/KMS guardrails | `oci/references/iam-security-guardrails.md` |
| Networking topology checks, routing/security control triage, and ops troubleshooting | `oci/references/networking-operations.md` |

## Key Starting Points

- `oci/references/landing-zone-core.md`
- `oci/references/iam-security-guardrails.md`
- `oci/references/networking-operations.md`

## Common Multi-Step Flows

| Task | Recommended Sequence |
|---|---|
| Build a new OCI tenancy foundation | `landing-zone-core` -> `iam-security-guardrails` -> `networking-operations` validation checklist |
| Harden an existing landing zone | `iam-security-guardrails` -> `landing-zone-core` customization points -> policy/guardrail verification |
| Diagnose connectivity incidents | `networking-operations` triage -> route/security control checks -> DRG and logging validation |

## Oracle Version Notes (19c vs 26ai)

When OCI tasks also involve Oracle Database targets:

- Use `19c` as baseline compatibility assumption unless the user provides a different baseline.
- Treat `26ai` adoption as an explicit compatibility/certification check across OCI service choices, client runtimes, and operational tooling.
- Keep cloud foundation design (IAM/network/security) and database-version decisions separate, then reconcile during integration planning.

## Sources

- https://github.com/oci-landing-zones/terraform-oci-core-landingzone
- https://docs.oracle.com/en-us/iaas/Content/cloud-adoption-framework/oci-core-landing-zone.htm
- https://docs.oracle.com/en-us/iaas/Content/cloud-adoption-framework/oci-landing-zones-overview.htm
