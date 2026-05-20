# OCI Networking and Operations Troubleshooting for Landing Zone Deployments

## Overview

Use this reference for day-2 networking checks and incident triage in OCI landing-zone environments, especially hub-and-spoke designs built from OCI Core Landing Zone patterns.

Landing-zone context from `terraform-oci-core-landingzone`:
- Centralized deployment model with compartments for networking, security, app, and database resources.
- Optional topologies such as three-tier VCNs, Exadata VCNs, OKE VCNs, and a Hub VCN.
- Hub VCN patterns with DRG, firewall insertion, and optional bastion/jump host.

Actionable baseline checks:
1. Confirm what was actually deployed (many networking options are conditional in landing-zone templates).
2. Build an inventory of VCNs, subnets, route tables, gateways, DRG attachments, NSGs, and security lists.
3. Validate the traffic path per flow: source, destination, protocol, port, route target, and security policy at each hop.

## Network Topology Basics (VCN/Subnets/DRG)

### VCN and subnet checks

- Verify VCN CIDRs do not overlap where peering or DRG transit is required.
- Verify each subnet role (public or private) and do not assume it can be changed later.
- Verify each subnet has the intended route table, security lists, and DHCP options.
- Prefer regional subnets unless there is a strict AD-specific requirement.

Quick validation checklist:
- `VCN`: CIDRs, compartment, attached gateways.
- `Subnet`: public/private type, CIDR, route table, NSGs/security lists.
- `VNIC`: actual NSG membership and private/public IPs for affected workloads.

### DRG checks

- Verify DRG attachment inventory (VCNs, VPN, FastConnect, RPC) and intended ingress and egress path.
- Verify route table assignment for each attachment.
- Verify import route distributions are aligned with segmentation intent.
- If segmentation is required, do not rely only on DRG defaults; use dedicated DRG route tables and distribution policies.

Landing-zone specific checks:
- In hub-and-spoke designs, verify spokes are attached as expected and that inter-spoke routes are either allowed or intentionally blocked.
- If firewall insertion is enabled, verify next-hop behavior does not bypass firewall pathing.

## Routing and Security Controls

### Routing checks

1. Confirm subnet route-table selection first.
- Each subnet uses one route table.
- If not explicitly set, subnet uses VCN default route table.

2. Validate gateway route intent.
- Public subnet internet egress: `0.0.0.0/0 -> Internet Gateway`.
- Private subnet internet egress: `0.0.0.0/0 -> NAT Gateway`.
- OCI service private access: route to Service Gateway using OSN service CIDR label.

3. Validate DRG transit intent.
- Confirm ingress attachment route table resolves destination to the intended egress attachment.
- Check route distribution statements when routes are unexpectedly missing or over-shared.

4. Validate route precedence.
- Longest prefix match applies.
- Custom intra-VCN routes can override normal local routing behavior.

### Security control checks

1. Check all policy layers, not only one.
- Subnet security list rules.
- NSG rules on each VNIC/resource.
- Instance OS firewall rules.

2. Use NSGs for app-tier segmentation where possible.
- NSGs let you decouple subnet layout from app security posture.
- Security lists still apply at subnet scope and may allow or block unexpectedly.

3. Verify stateful versus stateless expectations.
- For stateless patterns, return traffic rules must be explicit.
- For VPN troubleshooting, stateless rules may require explicit ICMP type 3 code 4 handling.

## Observability and Logging

### Minimum logging baseline

1. Enable VCN Flow Logs in Logging service.
- Choose enablement scope intentionally: VCN, subnet, or resource.
- Create and use capture filters to control volume and focus.

2. Enable service logs for network edges in scope.
- Load balancer logs.
- Site-to-Site VPN logs when hybrid connectivity is used.

3. Keep log-group hygiene.
- Separate operational logs and security logs.
- Standardize retention per environment.

### Practical flow-log checks

- If expected traffic is missing, check for `QualityEvent.NoData` or `QualityEvent.SkipData` status in flow logs.
- For managed VNICs (for example, load balancer managed VNICs), remember flow logs identify VNIC ID but not always the parent service directly.
- When enabling flow logs by resource, confirm compartment, VCN, and subnet selection to avoid false negatives.

### Topology verification tool

- Use Network Path Analyzer (NPA) to validate reachability from configuration state without generating traffic.
- Use NPA output to pinpoint the exact route or security control that denies traffic.

## Troubleshooting Playbook

### Triage order (use for every incident)

1. Define the failed flow exactly: source, destination, protocol, port, direction.
2. Run NPA for the same flow to identify route or policy denial points.
3. Verify route tables and route targets on both source and destination subnets.
4. Verify DRG attachment route table and route distribution (if DRG path is involved).
5. Verify NSG rules, security list rules, and host firewall rules together.
6. Correlate VCN Flow Logs and service logs (LB/VPN) for timing and drop signals.

### Common issues and focused checks

#### 1) Private subnet cannot reach internet

Checks:
- Private subnet has `0.0.0.0/0 -> NAT Gateway`.
- NAT Gateway exists in same VCN.
- Egress security rules and host firewall allow destination.

Notes:
- NAT Gateway supports outbound initiation and response only, not inbound internet-initiated access.
- NAT Gateway is not transitive for peered VCNs or on-premises networks.

#### 2) Cannot reach OCI service endpoints privately

Checks:
- Service Gateway exists and route rule points to correct OSN service CIDR label.
- Egress rules allow service destination.
- No conflicting broader route sends traffic to wrong next hop.

#### 3) Spoke-to-spoke or spoke-to-on-prem traffic fails via DRG

Checks:
- Confirm ingress attachment route table contains destination route.
- Confirm route distribution imports expected routes.
- Confirm intended segmentation is not intentionally blocking this path.
- Confirm firewall insertion next hops are present when required by design.

#### 4) Intermittent VPN or FastConnect behavior

Checks:
- Review Site-to-Site VPN logs for tunnel-down reason and proposal/subnet mismatches.
- For FastConnect, verify BGP session stability and on-premises device health during incident window.
- Confirm no recent route-policy change altered imported or exported prefixes.

#### 5) Terraform apply or destroy anomalies in core landing zone

Checks:
- If apply fails with transient `404-NotAuthorizedOrNotFound`, rerun plan/apply after propagation delay.
- For OCI Network Firewall teardown, be aware destroy may require a second pass due to attachment state.
- If using Hub VCN jump host with OCI native firewall, validate expected logging coverage (known limitation in repo notes).

## Sources

- OCI Core Landing Zone repository (architecture, scenarios, known issues):
  - https://github.com/oci-landing-zones/terraform-oci-core-landingzone
- Overview of VCNs and Subnets:
  - https://docs.oracle.com/en-us/iaas/Content/Network/Tasks/Overview_of_VCNs_and_Subnets.htm
- Networking Overview (DRG and transit concepts):
  - https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/overview.htm
- VCN Route Tables:
  - https://docs.oracle.com/en-us/iaas/Content/Network/Tasks/managingroutetables.htm
- NAT Gateway:
  - https://docs.oracle.com/en-us/iaas/Content/Network/Tasks/NATgateway.htm
- Security Rules:
  - https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/securityrules.htm
- Network Security Groups:
  - https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/networksecuritygroups.htm
- DRG routing management:
  - https://docs.oracle.com/en-us/iaas/Content/Network/Tasks/drg-routing-manage.htm
- Creating a DRG Route Table (default table behavior):
  - https://docs.oracle.com/en-us/iaas/Content/Network/Tasks/drg-rt-create.htm
- DRG route distribution behavior (CLI reference):
  - https://docs.oracle.com/en-us/iaas/tools/oci-cli/latest/oci_cli_docs/cmdref/network/drg-route-distribution.html
- Learn About Dynamic Routing Gateway Solutions (segmentation examples):
  - https://docs.oracle.com/en/solutions/learn-about-drg-solutions/index.html
- Flow Logs and enablement:
  - https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/vcn-flow-logs.htm
  - https://docs.oracle.com/en-us/iaas/Content/Network/Tasks/vcn-flow-logs-enable.htm
- VCN Flow Logs schema, limitations, and quality events:
  - https://docs.oracle.com/en-us/iaas/Content/Logging/Reference/details_for_vcn_flow_logs.htm
- Logging Overview:
  - https://docs.oracle.com/en-us/iaas/Content/Logging/Concepts/loggingoverview.htm
- Network Path Analyzer:
  - https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/path_analyzer.htm
- Troubleshooting VCNs and Connectivity:
  - https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/troubleshooting.htm
- Site-to-Site VPN Troubleshooting:
  - https://docs.oracle.com/en-us/iaas/Content/Network/Troubleshoot/ipsectroubleshoot2.htm
- Logging for Load Balancers:
  - https://docs.oracle.com/en-us/iaas/Content/Balance/Tasks/managingloadbalancer_topic-Logging.htm
