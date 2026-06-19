---
name: oci
description: Oracle Cloud Infrastructure domain routing for landing-zone architecture, IAM/security guardrails, networking operations, OCI Kubernetes Engine (OKE), and Enterprise AI workflows. Use for OCI tenancy foundation design, compartment and policy strategy, security control rollout (Cloud Guard, Security Zones, KMS), OCI networking triage, OKE cluster design/Terraform/Resource Manager planning, OKE incident troubleshooting (Generic VNIC Attachment, Multus, pod networking, node pools, add-ons, ingress, load balancers, OCIR, Workload Identity), and OCI Generative AI / Enterprise AI models, Responses API agents, RAG, cost estimation, governance, and private endpoints.
---

# Oracle Cloud Infrastructure Skills

This domain provides OCI guidance spanning landing-zone foundations and day-2
operations, OCI Kubernetes Engine (OKE), and Enterprise AI built on OCI
Generative AI. Use the routing table to select the narrowest OCI topic reference
or nested skill.

## How to Use This Domain

1. Identify whether the task is landing-zone architecture, IAM/security
   guardrails, networking operations, OKE, or Enterprise AI.
2. Start with the routing table below and open only the smallest reference file
   or nested skill that matches the task.
3. Prefer official Oracle documentation and live read-only discovery commands
   before making design or remediation recommendations.
4. Ask before running commands that create, update, delete, restart, scale,
   drain, or otherwise mutate OCI or Kubernetes resources.
5. Keep OCI-owned design decisions here, and route database-owned SQL, vector,
   and Select AI implementation details to `db/features/`.

## Directory Structure

```text
oci/
├── SKILL.md
├── enterprise-ai/
│   ├── SKILL.md
│   ├── models/
│   ├── agent-workflows/
│   ├── governance/
│   ├── data/
│   ├── cost/
│   └── integrations/
├── oke/
│   ├── cluster-design.md
│   ├── troubleshooting.md
│   ├── gva-node-pools.md
│   ├── multus-multihome.md
│   ├── skills/
│   ├── scripts/
│   ├── agents/
│   ├── shared/
│   ├── examples/
│   └── tests/
└── references/
    ├── landing-zone-core.md
    ├── iam-security-guardrails.md
    └── networking-operations.md
```

## Category Routing

| Topic | Start With |
|---|---|
| OCI Core Landing Zone structure, deployment flow, and customization points | `oci/references/landing-zone-core.md` |
| IAM model, compartment strategy, Cloud Guard/Security Zones/KMS guardrails | `oci/references/iam-security-guardrails.md` |
| Networking topology checks, routing/security control triage, and ops troubleshooting | `oci/references/networking-operations.md` |
| Design or scaffold an OKE cluster, Terraform stack, or OCI Resource Manager stack | Start with `oci/oke/cluster-design.md`, then load `oci/oke/skills/oke-cluster-generator/SKILL.md` |
| Troubleshoot OKE workloads, pods, services, DNS, add-ons, ingress, load balancers, image pulls, storage, Workload Identity, or cluster access | Start with `oci/oke/troubleshooting.md`, then load `oci/oke/skills/oke-troubleshooter/SKILL.md` |
| Configure OKE managed node pools with Generic VNIC Attachment secondary VNIC profiles and Application Resources | Start with `oci/oke/gva-node-pools.md`, then load `oci/oke/skills/oke-gva-deployer/SKILL.md` |
| Deploy or validate Multus NetworkAttachmentDefinitions and multi-interface pods on OKE | Start with `oci/oke/multus-multihome.md`, then load `oci/oke/skills/oke-multihome-deployer/SKILL.md` |
| OCI Generative AI models, custom/imported models, endpoints, or private endpoints | `oci/enterprise-ai/SKILL.md` |
| OCI Responses API agents, tools, memory, File Search, Code Interpreter, MCP, or SQL Search | `oci/enterprise-ai/SKILL.md` |
| OCI Generative AI and OCI Generative AI Agents cost estimation | `oci/enterprise-ai/cost/cost-estimation.md` |
| OCI Enterprise AI governance, IAM, API keys, OAuth, guardrails, or ZPR | `oci/enterprise-ai/governance/private-endpoints-and-governance.md` |

## Key Starting Points

- `oci/references/landing-zone-core.md`
- `oci/references/iam-security-guardrails.md`
- `oci/references/networking-operations.md`
- `oci/oke/cluster-design.md`
- `oci/oke/troubleshooting.md`
- `oci/oke/gva-node-pools.md`
- `oci/oke/multus-multihome.md`
- `oci/enterprise-ai/SKILL.md`
- `oci/enterprise-ai/models/enterprise-ai-models.md`
- `oci/enterprise-ai/agent-workflows/responses-api-agents.md`
- `oci/enterprise-ai/cost/cost-estimation.md`
- `oci/enterprise-ai/governance/private-endpoints-and-governance.md`

## Operational Tools

The OKE operational skills include deterministic helper tools under
`oci/oke/scripts/` and skill-specific helper scripts under
`oci/oke/skills/*/scripts/`.

- Read-only discovery and evidence tools may be used to collect context.
- Generate-only tools may produce manifests, commands, Terraform snippets, or reports.
- Any tool or command that creates, updates, deletes, patches, restarts, scales, drains, debugs, assigns IPs, applies manifests, or otherwise mutates OCI or Kubernetes resources requires explicit user approval first.
- `oci/oke/scripts/gva-menu.sh` is allowed to create an OKE node pool for the GVA workflow only after the user approves execution and completes its final `CREATE` confirmation.
- `oci/oke/scripts/node-doctor-run.sh` requires approval before execution because it creates a temporary debug pod and may delete that pod during cleanup.

## Common Multi-Step Flows

| Task | Recommended Sequence |
|---|---|
| Build a new OCI tenancy foundation | `landing-zone-core` -> `iam-security-guardrails` -> `networking-operations` validation checklist |
| Harden an existing landing zone | `iam-security-guardrails` -> `landing-zone-core` customization points -> policy/guardrail verification |
| Diagnose connectivity incidents | `networking-operations` triage -> route/security control checks -> DRG and logging validation |
| Plan a production OKE cluster | `oke/cluster-design.md` |
| Diagnose an OKE service with no load balancer IP | `oke/troubleshooting.md` |
| Build a node pool with workload-specific secondary VNIC profiles | `oke/gva-node-pools.md` -> `oke/multus-multihome.md` if pods need multiple interfaces |
| Validate Multus pod networking on GVA-enabled nodes | `oke/multus-multihome.md` -> `oke/troubleshooting.md` if symptoms remain |
| Investigate OKE workload access to OCI APIs | `oke/troubleshooting.md` |
| Build a governed enterprise assistant | `enterprise-ai/SKILL.md` -> `enterprise-ai/agent-workflows/agent-tools.md` -> `enterprise-ai/data/rag-and-search.md` -> `enterprise-ai/governance/private-endpoints-and-governance.md` |

## Oracle Version Notes (19c vs 26ai)

When OCI tasks also involve Oracle Database targets:

- Use `19c` as baseline compatibility assumption unless the user provides a different baseline.
- Treat `26ai` adoption as an explicit compatibility/certification check across OCI service choices, client runtimes, and operational tooling.
- Keep cloud foundation design (IAM/network/security) and database-version decisions separate, then reconcile during integration planning.

## Scope Boundaries

- Keep OCI service, networking, IAM, agent hosting, and cost-estimation guidance in this domain.
- Route Oracle Database-owned implementation details to `db/features/`.
- Route APEX artifact generation to `apex/apexlang/`.
- Prefer official Oracle documentation for OCI service limits, IAM verbs, endpoint formats, regions, and pricing inputs because these change frequently.

## Sources

- https://github.com/oci-landing-zones/terraform-oci-core-landingzone
- https://docs.oracle.com/en-us/iaas/Content/cloud-adoption-framework/oci-core-landing-zone.htm
- https://docs.oracle.com/en-us/iaas/Content/cloud-adoption-framework/oci-landing-zones-overview.htm
- https://docs.oracle.com/en-us/iaas/Content/ContEng/home.htm
- https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengAttaching_Multiple_VNICs.htm
- https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contenggrantingworkloadaccesstoresources.htm
- https://github.com/oracle-terraform-modules/terraform-oci-oke
- https://docs.oracle.com/en-us/iaas/Content/generative-ai/overview.htm
- https://docs.oracle.com/en-us/iaas/Content/generative-ai/home.htm
