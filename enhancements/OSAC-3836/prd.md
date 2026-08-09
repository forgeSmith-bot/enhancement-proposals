# CaaS Bare Metal Worker Node Provisioning

| Field       | Value   |
|-------------|---------|
| Author(s)   | To be determined |
| Jira        | [OSAC-3836](https://redhat.atlassian.net/browse/OSAC-3836) |
| Date        | 2026-08-09 |

## Problem Statement

CaaS tenants need OpenShift clusters backed by bare metal compute, but there is no on-demand provisioning path today. The current approach maintains a static pool of pre-booted hosts via a cron job, which wastes resources on idle hosts, is difficult to size to actual demand, and couples cluster-specific concepts into the BMaaS layer. Without on-demand provisioning, bare metal capacity is either over-provisioned (wasting hardware) or under-provisioned (blocking cluster creation), and the coupling between layers creates an operational burden that does not scale.

## In Scope

- Tenants can create OpenShift clusters with bare metal worker nodes by selecting a BareMetalInstanceType in their ClusterOrder. Bare metal hosts are provisioned on-demand — no static host pooling.
- BareMetalInstances and images created to back cluster worker nodes are hidden from tenants. Tenants interact only with ClusterOrder. Billing and quota are tracked at the cluster level.
- When a cluster is decommissioned or scaled down, bare metal hosts are released. Every released host is fully cleaned before reassignment — no residual data, credentials, or network configuration from the previous tenant's workload.
- BMaaS has no knowledge of clusters, agents, or Assisted Installer. No cluster or agent terminology appears in BMaaS APIs or tenant-facing APIs.

## Out of Scope

- Day-2 autoscaling — automatic scale-up and scale-down based on cluster load. Scale-down requires coordination with cluster drain and agent unbind.
- VM worker nodes — the same on-demand provisioning pattern applied to virtual machines. Planned as future work.
- Admin API for tuning CaaS provisioning behavior (timeouts, retry policies, provisioning parameters).
- Boot-over-network optimization (OSAC-2134).
- Networking configuration — CaaS uses whatever networking API BMaaS provides. No new networking capabilities are introduced.

## User Stories

### Tenant User

- As a Tenant User, I want to create a ClusterOrder specifying BareMetalInstanceTypes for my worker nodes so that I get an OpenShift cluster backed by bare metal without managing hardware, images, or agents directly.

- As a Tenant User, I want my cluster creation experience to be unchanged — I should not see or interact with BareMetalInstances, images, or any infrastructure details. I see only my ClusterOrder and its status.

- As a Tenant User, I want billing and quota for bare metal worker nodes tracked at the cluster level so that I manage costs through my cluster, not through individual bare metal hosts I cannot see.

### Cloud Provider Admin

- As a Cloud Provider Admin, I want every bare metal host released from a tenant's cluster to be fully cleaned (disk wipe, network reset, credential removal) before it is returned to inventory, so that no data or configuration leaks between tenants.

- As a Cloud Provider Admin, I want BareMetalInstances created for CaaS cluster workers to be hidden from tenants so that tenants cannot interfere with infrastructure backing their clusters.

## Assumptions

- BareMetalInstanceType names are usable as values in ClusterOrder's resourceClass field. The existing resource class selection mechanism in the UI, CLI, and API is sufficient — no new interaction model is needed.
- BMaaS already supports host cleanup (disk wipe, network reset) on BareMetalInstance deletion. This feature relies on that cleanup being comprehensive and reliable.

## Dependencies

- **OSAC-2308 (MAC address in BareMetalInstance status):** The platform requires MAC addresses from provisioned bare metal hosts to correlate them with cluster worker nodes.
- **BareMetalInstanceType (EP PR #59):** Defines the bare metal configuration types that tenants reference when creating clusters with bare metal workers.
- **User data pass-through in BareMetalInstance:** Bare metal hosts must support user-provided boot configuration data to enable on-demand worker provisioning.
- **BareMetalInstance references ComputeImage:** Existing capability for bare metal hosts to boot from a specified disk image.

## Acceptance Criteria

- [ ] A tenant can create a ClusterOrder specifying BareMetalInstanceType names for worker nodes, and bare metal hosts are provisioned on-demand as worker nodes for the cluster.
- [ ] BareMetalInstances and images created for CaaS cluster workers are not visible to tenants — tenants see only their ClusterOrder and its worker node status.
- [ ] When a tenant deletes a cluster or scales down a node pool, bare metal hosts are released and fully cleaned before being returned to inventory — no residual data, credentials, or network configuration from the previous use.
- [ ] No Assisted Installer or agent terminology appears in BMaaS APIs or tenant-facing APIs.
- [ ] Billing and quota for bare metal worker nodes are tracked at the cluster level, not per individual bare metal host.

## Risks

### Agent registration failure after host boot

Bare metal provisioning completes when the host boots, but the cluster worker node is functional only after the boot agent registers and joins the cluster. If registration fails (network issues, image corruption, boot configuration errors), the tenant observes a cluster that never reaches the expected worker count.

- **Owner:** CaaS team
- **Mitigation:** To be determined — error handling and retry strategy are design-level concerns.

### Bare metal inventory exhaustion

Bare metal hosts are finite physical resources. If no hosts matching the requested BareMetalInstanceType are available, cluster provisioning cannot complete.

- **Owner:** To be determined
- **Mitigation:** To be determined — capacity planning and user-facing error reporting are design-level concerns.

## Open Questions

### Does the ClusterOrder UI need changes to surface BareMetalInstanceType options?

- **Owner:** To be determined
- **Impact:** If BareMetalInstanceTypes require new UI treatment beyond the existing resourceClass selector, UI work may be in scope. The source material states the tenant experience is "unchanged," suggesting existing UI patterns suffice.

### What error does the tenant see when bare metal capacity is exhausted?

- **Owner:** To be determined
- **Impact:** Affects cluster provisioning — the tenant needs a meaningful error when provisioning cannot complete due to bare metal inventory exhaustion.

---

## Unattended PRD Run Complete

Phases: ingest ✓ → exemplars ✓ → dimensions ✓ → self-clarify ✓ → draft ✓ → review ✓ → context ✓
Artifacts: .artifacts/prd/OSAC-3836/
Score: 10/10 (WHAT:2 WHY:2 UF:2 RS:2 T:2)
Open questions: 2 items need human resolution
Revision rounds: 1
Result: PRD generated following the flat exemplar structure (OSAC-2872, OSAC-1270, OSAC-2917) with user stories grouped by persona. All design details (MAC correlation, ignition payloads, InfraEnv mechanics, controller logic) kept out of PRD per OSAC review patterns.

Next steps:
- Review open questions in session-context.md
- Set the Author(s) field to the human feature owner
- Run /revise to refine the PRD interactively
- Run /publish when ready for external review