# CaaS Bare Metal Worker Node Provisioning

| Field       | Value   |
|-------------|---------|
| Author(s)   |         |
| Jira        | [OSAC-3835](https://redhat.atlassian.net/browse/OSAC-3835) |
| Date        | 2026-08-09 |

## Problem Statement

CaaS tenants need OpenShift clusters backed by bare metal worker nodes, but today bare metal capacity is pre-provisioned in a static pool that wastes resources on idle hosts and is difficult to size correctly. Tenants have no way to request bare metal workers as part of their cluster — and the static pool approach leaks cluster infrastructure concepts into layers that should not be aware of them. Without on-demand bare metal provisioning, CaaS cannot efficiently scale bare metal capacity to match actual cluster demand.

## In Scope

- On-demand bare metal worker node provisioning for CaaS clusters via BMaaS, replacing the static pre-provisioned pool
- BareMetalInstances and images created for CaaS worker nodes are hidden from tenants — tenants interact only with ClusterOrders
- Billing and quota tracked at the cluster level, not per individual bare metal instance
- Secure host cleanup by BMaaS when bare metal workers are deprovisioned — hosts are fully sanitized before returning to inventory to prevent data or configuration leakage between tenants
- Services affected: CaaS, BMaaS

## Out of Scope

- Day-2 autoscaling — scaling down requires careful coordination with cluster drain and is deferred to a future feature
- VM-backed worker nodes — same provisioning pattern, future work
- Admin API for tuning CaaS provisioning behavior
- Boot-over-network optimization (OSAC-2134)
- Networking configuration — CaaS uses whatever networking BMaaS provides

## User Stories

### Tenant Admin / Tenant User

- As a Tenant Admin or Tenant User, I want to create a ClusterOrder specifying BareMetalInstanceTypes for my worker nodes so that I get a cluster backed by bare metal without managing hardware directly.
- As a Tenant Admin or Tenant User, I want my cluster creation experience to remain unchanged — I should not see or interact with BareMetalInstances, images, or any infrastructure details provisioned on my behalf.

### Cloud Provider Admin

- As a Cloud Provider Admin, I want every deprovisioned bare metal host to be securely sanitized by BMaaS before it is returned to inventory so that no data or configuration leaks between tenants during scale-down or cluster decommission.

### Cloud Infrastructure Admin

- Not affected by this feature.

## Assumptions

- The Assisted Installer agent can be fully bootstrapped from a standard OS image with user data — no dedicated boot ISO is required.
- BMaaS has no awareness of clusters, agents, or the Assisted Installer — it provisions a host with an image and user data, and its responsibility ends when the host boots successfully.

## Dependencies

- **OSAC-2308 (MAC address in BareMetalInstance status):** CaaS requires the MAC address from BareMetalInstance status to correlate provisioned hosts with registered cluster agents. Must land before this feature.
- **BareMetalInstanceType EP (PR #59):** Defines the instance type model that ClusterOrder worker node requests reference. Must land before this feature.
- **User data pass-through in BareMetalInstance spec:** CaaS requires the ability to supply user data when creating a BareMetalInstance so that the host boots with the correct bootstrap configuration.