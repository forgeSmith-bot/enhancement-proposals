# CaaS Bare Metal Worker Node Provisioning

| Field       | Value   |
|-------------|---------|
| Author(s)   |         |
| Jira        | [OSAC-3831](https://issues.redhat.com/browse/OSAC-3831) |
| Date        | 2026-08-09 |

## Problem Statement

CaaS cluster worker nodes backed by bare metal are provisioned through a static pool of pre-booted hosts. This wastes resources on idle hosts that may never be assigned, is difficult to size correctly for fluctuating demand, and couples cluster-level concepts into the bare metal layer. Tenants experience this as unpredictable cluster provisioning times and capacity constraints that the platform cannot resolve dynamically.

The on-demand provisioning model eliminates the static pool: CaaS requests bare metal hosts directly from BMaaS when a cluster needs them and returns them when it does not. BMaaS has no awareness of clusters — it boots a host with an image and user data. This separation reduces waste, simplifies operations, and keeps bare metal infrastructure details invisible to tenants.

## In Scope

- On-demand bare metal worker node provisioning for CaaS clusters — hosts are requested from BMaaS when needed and returned when no longer needed, replacing the static pre-booted host pool
- Tenant-transparent provisioning — BareMetalInstances, images, and infrastructure details created for CaaS are not visible to tenants; billing and quota remain at the cluster level
- Bare metal worker node deprovisioning on cluster deletion or node pool scale-down, with hosts securely sanitized by BMaaS before returning to inventory
- CaaS and BMaaS service boundary — BMaaS provisions hosts with an image and user data; it has no knowledge of clusters or OpenShift concepts

## Out of Scope

- Day-2 autoscaling — scaling down requires coordination with cluster drain and workload evacuation; deferred to a future feature
- VM-based worker nodes — same architectural pattern, separate future work
- Admin API for tuning CaaS provisioning behavior (e.g., host selection preferences)
- Boot-over-network optimization (OSAC-2134)
- Networking configuration — CaaS uses whatever networking BMaaS provides

## User Stories

### Tenant Admin / Tenant User

- As a Tenant Admin/User, I want to create a ClusterOrder specifying BareMetalInstanceTypes for my worker nodes so that my cluster is backed by bare metal without managing hardware directly.
- As a Tenant Admin/User, I want my cluster creation experience to remain unchanged — I should not see or interact with BareMetalInstances, images, or any infrastructure details used to provision my worker nodes.

### Cloud Provider Admin

- As a Cloud Provider Admin, I want bare metal hosts to be securely sanitized by BMaaS before returning to inventory after a cluster is decommissioned or scaled down, so that no data or configuration from one tenant leaks to another.
- As a Cloud Provider Admin, I want bare metal hosts provisioned for CaaS to be requested on demand and returned when no longer needed, so that hosts are not wasted on idle pre-booted pools.

### Cloud Infrastructure Admin

- Not affected by this feature. Bare metal inventory management and network infrastructure are pre-existing capabilities.

## Assumptions

- The BMaaS private API supports creating BareMetalInstances with an image reference and user data pass-through. User data pass-through is not yet a confirmed capability.
- BareMetalInstanceType definitions exist and are referenced by name in ClusterOrder node requests.

## Dependencies

- **OSAC-2308 (MAC address in BareMetalInstance status):** CaaS requires the MAC address from BareMetalInstance status to identify which provisioned host joined the cluster. Must land before this feature.
- **BareMetalInstanceType (EP PR #59):** Defines the instance type resource that tenants reference when specifying bare metal worker node configurations in ClusterOrders.
- **User data pass-through in BareMetalInstance:** CaaS passes boot-time configuration as user data when creating a BareMetalInstance. This capability must be available in the BMaaS API.