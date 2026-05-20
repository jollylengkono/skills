# OCI Core Landing Zone Architecture Reference

## Overview

The OCI Core Landing Zone is the centralized OCI landing zone blueprint that unifies earlier CIS Landing Zone and Oracle Enterprise Landing Zone approaches. It is delivered as Terraform code and aligns tenancy foundations with CIS OCI benchmark guidance plus OCI architecture practices.

Operationally, treat it as a foundation layer:
- It sets up compartments, IAM groups/policies, networking patterns, governance, and security services.
- It does not directly provision workload runtime services (for example, application compute or database workloads).
- It supports both greenfield and brownfield tenancy adoption patterns.

## Repository Structure Mapping

Use this map to navigate the repository quickly:

- `DEPLOYMENT-GUIDE.md`: end-to-end operating guide, including considerations, scenarios, and deployment methods (Terraform CLI, Resource Manager UI, Resource Manager CLI).
- `ARCH-MAPPING-CIS.md`: mapping between architecture elements and CIS benchmark intent.
- `VARIABLES.md`, `SPEC.md`, `schema.yml`: input model and configuration contract.
- `templates/`: ready scenario presets (for example `cis-basic`, `new-identity-domain`, `custom-identity-domain`, `standalone-three-tier-vcn-*`, `hub-spoke-*`, `externally-managed-vcns`).
- `extensions/`: extension patterns used to separate foundational prerequisites from later workload-oriented additions.
- `images/`: architecture diagrams used by docs.

Main Terraform composition in repo root is grouped by prefix:
- `iam_*.tf`: compartments, groups, dynamic groups, identity domain, policies.
- `net_*.tf`: VCN topologies, DRG/hub, VPN/FastConnect, additional networks, network firewall.
- `sec_*.tf`: Cloud Guard, Security Zones, Vault, Vulnerability Scanning, Bastion.
- `mon_*.tf`: tags, topics, notifications, service connector, alarms, logs.
- `zpr_*.tf`: Zero Trust Packet Routing policy resources.
- `variables_*.tf`, `providers.tf`, `data_sources.tf`, `outputs.tf`: interface and execution wiring.

## Deployment Flow

Use this operational flow:

1. Select execution path:
- OCI Resource Manager (single-click stack import from GitHub), then run `Plan` and `Apply`.
- Terraform CLI with local clone for custom changes.

2. Validate prerequisites and constraints:
- Confirm required permissions (default deployment expects root-level tenancy admin ability to create foundational compartments and policies).
- Decide greenfield vs brownfield onboarding model.
- Choose target network topology before first apply to reduce disruptive later refactors.

3. Choose a baseline template scenario:
- Start with a `templates/` scenario closest to your target (identity model, standalone or hub-spoke network, firewall model, bastion/jump-host, ZPR option).

4. Set key configuration inputs:
- Use `service_label` to avoid naming collisions in existing tenancies.
- Provide `existing_drg_id` when integrating with pre-existing DRG connectivity.
- Adjust variables and, when needed, targeted `net_*.tf` topology files for route/security rule specifics.

5. Run staged deployment:
- `Plan` first, review drift and resource actions.
- `Apply` only after plan validation.
- Post-deployment, verify IAM segregation, compartment placement, network connectivity intent, and enabled security controls.

6. Deploy workloads on top of the foundation:
- Treat core landing zone as baseline controls and attachment points.
- Add workload-specific resources in separate layers or extensions.

## Common Customization Points

- Identity model:
  - New identity domain or custom identity domain templates.
  - Reuse existing groups/dynamic groups with landing-zone policy model.

- Network model:
  - No-network option.
  - Standalone three-tier VCN (defaults, custom, or ZPR-enabled).
  - Hub-spoke variants with DRG, OCI Network Firewall, third-party appliance, FastConnect virtual circuit, or IPSec VPN.
  - Externally managed VCN integration path.

- Existing tenancy integration:
  - Brownfield naming isolation via `service_label`.
  - Existing DRG attachment via `existing_drg_id`.
  - Migration of pre-existing resources into landing-zone compartments with policy review.

- Security posture tuning:
  - ZPR is optional and requires explicit configuration.
  - Cloud Guard, Security Zones, Vault, Vulnerability Scanning, and Bastion are part of baseline security capability and can be tailored by variable/config choices.

## Sources

- OCI Core Landing Zone repository (primary source): https://github.com/oci-landing-zones/terraform-oci-core-landingzone
- Repository templates directory: https://github.com/oci-landing-zones/terraform-oci-core-landingzone/tree/main/templates
- Repository deployment guide: https://github.com/oci-landing-zones/terraform-oci-core-landingzone/blob/main/DEPLOYMENT-GUIDE.md
- OCI Landing Zones overview: https://docs.oracle.com/en-us/iaas/Content/cloud-adoption-framework/oci-landing-zones-overview.htm
- OCI Core Landing Zone documentation: https://docs.oracle.com/en-us/iaas/Content/cloud-adoption-framework/oci-core-landing-zone.htm
