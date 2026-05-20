# OCI IAM and Security Guardrails for Landing Zones

## Overview

Use this reference to design and validate IAM and security controls for OCI landing zones.
It is anchored on:

- OCI Core Landing Zone architecture and Terraform implementation patterns.
- OCI IAM, Cloud Guard, Security Zones, and Vault documentation.

The OCI Core Landing Zone model uses compartments, groups, and IAM policies to separate duties by function (network, security, app, database) and applies CIS-focused controls such as Cloud Guard, encryption keys, logging, and Security Zones.

## IAM/Compartment Strategy

1. Start with a functional compartment model, then apply RBAC.
   Typical pattern from OCI Core Landing Zone:
   - Enclosing compartment (optional parent)
   - Network compartment
   - Security compartment
   - Application development compartment
   - Database compartment
   - Exadata infrastructure compartment (optional)
2. Keep security tooling centralized in the Security compartment (logging, key management, scanning, notifications) as the landing zone pattern recommends.
3. Scope administrator access by compartment ownership, not by tenancy-wide rights.
4. Use compartment boundaries as the first governance boundary, then enforce least privilege with IAM policies.

## Policies and Groups

1. Write policies with OCI policy syntax:
   `Allow <subject> to <verb> <resource-type> in <location> [where <conditions>]`
2. Remember OCI policies allow access; they do not provide explicit deny statements.
3. Use least privilege verbs intentionally:
   - `inspect` for discovery
   - `read` for viewing details
   - `use` for operating existing resources
   - `manage` for full admin actions
4. Model identities in two layers:
   - Human admin/user groups (network-admins, security-admins, app-admins, db-admins)
   - Dynamic groups for workload principals (compute/functions) that need OCI API access
5. Prefer compartment-scoped policies first; use tenancy-scoped policies only when required.
6. In identity domains, ensure group naming is consistent and policy scope is explicit (tenancy vs compartment path).

Example policy patterns:

```text
Allow group <domain>/security-admins to manage cloud-guard-family in tenancy
Allow group <domain>/security-admins to manage security-zone-family in tenancy
Allow group <domain>/key-admins to manage vaults in compartment Security
Allow dynamic-group <domain>/app-workloads to use keys in compartment Security
```

## Security Guardrails (Cloud Guard/Security Zones/KMS)

### Cloud Guard

- Define target scope deliberately. A target can monitor a compartment and its subcompartments.
- Do not overlap targets accidentally: a compartment can be associated with only one target.
- Use detector recipes and responder recipes that match your operating model, then tune for false positives.

### Security Zones

- Use Security Zones for high assurance compartments.
- OCI validates create/update operations against security zone policies and denies operations that violate policy.
- Treat security zone violations as deployment blockers, not warnings.

### KMS (Vault and Keys)

- Use OCI Vault for customer-managed encryption keys where stronger key governance is required.
- For services like Block Volume, customer-managed keys are optional; Oracle-managed keys are used by default if you do not specify your own key.
- Define key lifecycle controls (rotation, disable, deletion approvals). Vault/key deletion is scheduled with a waiting period (7 to 30 days), so include this in change planning.

## Validation Checklist

- [ ] Compartment hierarchy matches ownership model (Network, Security, App, DB, optional Exadata/enclosing).
- [ ] Admin groups exist for each compartment owner function.
- [ ] No unnecessary tenancy-wide `manage all-resources` grants.
- [ ] Dynamic groups are defined for workload identities that call OCI APIs.
- [ ] Human and workload policies are separated and scoped to minimum required compartments.
- [ ] Cloud Guard is enabled with intended target scope and non-overlapping target design.
- [ ] Security Zones are applied to designated high assurance compartments.
- [ ] No unresolved Security Zone policy violations in planned deployment scope.
- [ ] Vault and key policies allow only required services/principals to use keys.
- [ ] CMK requirements are verified for sensitive workloads; default Oracle-managed key usage is explicitly accepted where CMK is not required.
- [ ] Logging/notifications/security services are deployed in the Security compartment per landing zone pattern.
- [ ] Terraform plan/apply runbooks include eventual consistency retry handling for IAM/policy propagation issues.

## Sources

- OCI Core Landing Zone repository (architecture and guardrail scope): https://github.com/oci-landing-zones/terraform-oci-core-landingzone
- OCI Core Landing Zone documentation: https://docs.oracle.com/en-us/iaas/Content/cloud-adoption-framework/oci-core-landing-zone.htm
- IAM Policies Overview (syntax and allow model): https://docs.oracle.com/iaas/Content/Identity/policieshow/Policy_Basics.htm
- IAM Policy Reference (verbs/resources): https://docs.public.content.oci.oraclecloud.com/en-us/iaas/Content/Identity/Reference/policyreference.htm
- Managing Groups: https://docs.oracle.com/iaas/Content/Identity/groups/managinggroups.htm
- Managing Dynamic Groups: https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/managingdynamicgroups.htm
- IAM Tenancy and Compartments (security guidance): https://docs.oracle.com/en-us/iaas/Content/Security/Reference/iam_security_topic-IAM_Tenancy_and_Compartments.htm
- About OCI Targets (Cloud Guard scope rules): https://docs.oracle.com/en-us/iaas/Content/cloud-guard/using/targets-about.htm
- Cloud Guard Concepts: https://docs.oracle.com/iaas/cloud-guard/using/cg-concepts.htm
- Overview of Security Zones: https://docs.oracle.com/iaas/Content/security-zone/using/security-zones.htm
- Security Zone Policies (deny violating operations): https://docs.oracle.com/en-us/iaas/Content/security-zone/using/security-zone-policies.htm
- Overview of Vaults and Key Management: https://docs.oracle.com/iaas/Content/KeyManagement/Concepts/keyoverview.htm
- Managing Vault Encryption Keys for Block Volume: https://docs.public.oneportal.content.oci.oraclecloud.com/iaas/Content/Block/Concepts/managingblockencryptionkeys.htm
- Deleting a Vault (7-30 day schedule window): https://docs.oracle.com/en-us/iaas/Content/KeyManagement/Tasks/managingvaults_topic-To_delete_a_vault.htm
