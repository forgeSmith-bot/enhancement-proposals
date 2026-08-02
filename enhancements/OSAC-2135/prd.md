# CaaS Bare Metal Worker Node Provisioning

| Field       | Value |
|-------------|-------|
| Author(s)   | CaaS and BMaaS Product Teams |
| Jira        | OSAC-2135 |
| Date        | 2026-08-02 |

## Problem Statement

Cluster as a Service (CaaS) requires bare metal compute hosts to back OpenShift cluster worker nodes. Currently, this process relies on a background cron job that maintains a static pool of agents (pre-booted hosts running the Assisted Installer ISO). 

This approach introduces severe operational and resource inefficiencies:
- **Idle Resource Waste:** Physical hosts remain booted and idle in the static pool, consuming power and hardware resources without active workloads.
- **Sizing Inaccuracies:** The static pool size is difficult to predict and scale, leading to either resource exhaustion or excessive waste.
- **Architectural Coupling:** Cluster-specific concepts (such as agents and InfraEnvs) are tightly coupled into the Bare Metal as a Service (BMaaS) layer, violating the principle of separation of concerns.

If this is not addressed, OSAC will suffer from high operational costs due to underutilized hardware, slow or failing cluster scale-up requests, and maintainability issues stemming from coupled service domains.

## In Scope

- **On-Demand Provisioning:** Direct, automated creation of `BareMetalInstances` via the BMaaS private API for cluster worker nodes.
- **Standardized Boot Payload:** Booting of worker nodes using a standard `ComputeImage` (qcow2 format) combined with cluster-specific discovery ignition passed as user data.
- **MAC Address Correlation:** Retrieval of MAC addresses from `BareMetalInstance` status to uniquely map physical hosts to registered Assisted Installer agents.
- **Tenant Isolation & Security:** Complete exclusion of CaaS-managed `BareMetalInstances` and underlying `ComputeImages` from tenant-facing APIs and UI consoles.
- **Automatic Lifecycle Cleanup:** Deletion of `BareMetalInstance` resources upon cluster scale-down or decommission, triggering a mandatory, automated host cleanup (disk wipe, network reset, and credential removal) by BMaaS before returning the host to the general inventory.
- **Race Prevention:** Allocation of exactly one isolated `InfraEnv` per cluster (or per node pool) to prevent cross-tenant agent registration races.
- **Resource Definition:** Integration of `ClusterOrder` specifications with BMaaS resource definitions, allowing tenants to request specific worker node hardware via `ClusterOrder.spec.nodeRequests[].resourceClass`.

## Out of Scope

- **Day-2 Autoscaling:** Automated dynamic scaling based on real-time workload demands (scaling down requires complex orchestration around cluster draining and agent unbinding, which is deferred to a future phase).
- **Virtual Machine Worker Nodes:** Provisioning VM-based worker nodes using this on-demand pattern (deferred to future VMaaS integrations).
- **Admin Tuning APIs:** Dedicated administrator-facing APIs for tweaking CaaS provisioning heuristics or retry thresholds.
- **Boot Optimization:** Network boot acceleration or advanced bare metal caching strategies `[Jira: OSAC-2134]`.
- **Custom Networking Configuration:** Direct management of tenant-specific VLANs or advanced network routing by CaaS (CaaS will consume default BMaaS-provided network interfaces).

## User Stories

### Tenant Admin

- As a Tenant Admin, I want to create a `ClusterOrder` specifying supported `BareMetalInstanceTypes` (via `nodeRequests[].resourceClass`) for my worker nodes, so that my Kubernetes/OpenShift clusters are backed by high-performance physical hardware without me having to manage raw infrastructure directly.

- As a Tenant Admin, I want my resource usage, quotas, and billing to be tracked at the cluster level rather than at the individual bare metal instance level, so that I can easily budget and monitor my organization's cloud spend.

### Tenant User

- As a Tenant User, I want my cluster creation and self-service management experience to remain entirely unchanged, so that I am never exposed to underlying `BareMetalInstances`, images, installers, or physical MAC addresses.

### Cloud Provider Admin

- As a Cloud Provider Admin, I want all CaaS-provisioned `BareMetalInstances` and standard `ComputeImages` to be hidden from tenant-facing views and catalogs, so that tenants cannot accidentally modify or delete underlying infrastructure nodes.

### Cloud Infrastructure Admin

- As a Cloud Infrastructure Admin, I want every deprovisioned bare metal host to undergo a guaranteed, blocking cleanup (including deep disk wipe and network reset) by the BMaaS layer before being returned to the general inventory pool, so that I can prevent security leaks and configuration drift between different tenants.

- As a Cloud Infrastructure Admin, I want to ensure that no Assisted Installer, agent, or cluster-specific terminology is exposed within the BMaaS private APIs, so that the BMaaS service remains a clean, generic bare-metal-as-a-service provider.

## Assumptions

- **Ignition Support:** The BMaaS private API can ingest and reliably pass through standard discovery ignition payload as user data to the target physical host.
- **Agent Initialization:** The standard qcow2 `ComputeImage` provided is pre-configured to boot, process the ignition payload, and start the Assisted Installer agent without manual intervention.
- **Boundary of Failure:** BMaaS provisioning is considered successful once the image is written and the host boots. If the agent fails to register after boot, this is classified and debugged as a CaaS-level software failure rather than a BMaaS infrastructure failure.

## Dependencies

- **MAC Address Status Exposure:** BMaaS must expose the physical MAC address of the host in the `BareMetalInstance` status subresource `[Jira: OSAC-2308]`.
- **BareMetalInstanceType Definition:** The `BareMetalInstanceType` specifications and schema definitions must be finalized and available `[PR #59]`.
- **User Data Pass-through:** BMaaS private API must support the ingestion and pass-through of ignition configurations in the `BareMetalInstance` spec.

## Risks & Mitigations

- **Risk: Provisioning Latency and Timeouts**
  - *Description:* Writing standard qcow2 images to physical disks and performing complete boot cycles can take significant time, potentially causing CaaS timeouts.
  - *Mitigation:* Establish clear boundary lines. CaaS will separate "Instance Booted" (BMaaS complete) from "Agent Registered" (CaaS complete), utilizing long-running, asynchronous reconciliation with generous timeout thresholds.

- **Risk: Cross-Tenant Data Contamination**
  - *Description:* During scale-down, a physical host previously used by Tenant A could be assigned to Tenant B with residual sensitive data on local storage.
  - *Mitigation:* BMaaS must mandate a synchronous, non-bypassable host sanitization flow (disk wipe and network state reset) immediately upon `BareMetalInstance` deletion, blocking the host's return to the active inventory until complete.

- **Risk: Cross-Tenant Agent Registration Races**
  - *Description:* Multiple agents registering from different hosts simultaneously might register to the wrong cluster control planes.
  - *Mitigation:* CaaS will provision exactly one unique `InfraEnv` per cluster or node pool, ensuring that registration endpoints and credentials are strictly isolated to single tenants.