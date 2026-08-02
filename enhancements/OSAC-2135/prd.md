# CaaS Bare-Metal Worker Node Provisioning

| Field       | Value |
|-------------|-------|
| Author(s)   | CaaS and BMaaS Product Teams |
| Jira        | OSAC-2135 |
| Date        | 2026-08-02 |

## Problem Statement

Cluster as a Service (CaaS) requires bare-metal compute hosts to back OpenShift cluster worker nodes. Currently, this process relies on a background cron job that maintains a static pool of agents (pre-booted hosts running the Assisted Installer ISO). 

This approach introduces severe operational and resource inefficiencies:
- **Idle Resource Waste:** Physical hosts remain booted and idle in the static pool, consuming power and hardware resources without active workloads.
- **Sizing Inaccuracies:** The static pool size is difficult to predict and scale, leading to either resource exhaustion or excessive waste.
- **Architectural Coupling:** Cluster-specific concepts (such as agents and InfraEnvs) are tightly coupled into the Bare-Metal-as-a-Service (BMaaS) layer, violating the principle of separation of concerns.

If this is not addressed, OSAC will suffer from high operational costs due to underutilized hardware, slow or failing cluster scale-up requests, and maintainability issues stemming from coupled service domains.

## In Scope

- **On-Demand Provisioning:** Direct, automated creation of `BareMetalInstances` via the BMaaS private API for cluster worker nodes.
- **Standardized Boot Payload:** Booting of worker nodes using a standard `ComputeImage` (qcow2 format) combined with cluster-specific discovery ignition passed as user data.
- **MAC Address Correlation:** Retrieval of physical MAC addresses from `BareMetalInstance` status to uniquely map physical hosts to registered Assisted Installer agents. The complete matching contract is defined as follows:
  - **Exposed Values:** BMaaS must expose all physical MAC addresses of the host's network interfaces as a structured list within the `BareMetalInstance` status, clearly identifying the primary/boot interface MAC address as the canonical reference.
  - **Normalization and Comparison:** CaaS retrieves this list of MAC addresses, normalizes all values to lowercase, colon-separated format (e.g., `aa:bb:cc:dd:ee:ff`), and compares them against the list of interface MAC addresses reported by the Assisted Installer agent's system discovery reports.
  - **Deterministic Edge-Case Behavior:**
    - *Missing MACs:* If the `BareMetalInstance` status lacks MAC address information, CaaS will pause provisioning, mark the node's reconcile status as `AwaitingHardwareDiscovery`, and retry with an exponential backoff.
    - *Duplicate MACs:* If a MAC address matches multiple `BareMetalInstance` resources or multiple registered agents, CaaS will trigger a high-severity alert, isolate the conflicting hosts/agents in a quarantined state, and fail the affected `ClusterOrder` with a clear validation error.
    - *Changed MACs:* The MAC-to-agent mapping contract is immutable once successfully established. Any subsequent changes in the reported MAC addresses of an active instance will trigger a node reconciliation failure, prompting a replacement of the node.
- **Tenant Isolation & Security:** Complete exclusion of CaaS-managed `BareMetalInstances` and underlying `ComputeImages` from tenant-facing APIs and UI consoles.
- **Automatic Lifecycle Cleanup:** Deletion of `BareMetalInstance` resources upon explicit, administrator-initiated cluster decommissioning or manual node pool scale-down operations. This triggers a mandatory, automated, blocking host cleanup (including deep disk wipe, network interface reset, and credentials removal) performed by BMaaS before the host can be returned to the general active inventory pool. This lifecycle cleanup applies exclusively to these manual, administrator-driven operations and does not cover automated workload-driven scaling.
- **Race Prevention:** Allocation of exactly one isolated `InfraEnv` per cluster, uniquely scoped and keyed by the Cluster UUID as its identity key, to prevent cross-tenant agent registration races. Node pools within the same cluster share this single `InfraEnv`. The `InfraEnv` is created immediately during initial cluster bootstrap (prior to any worker node provisioning) and is deleted only during the final cluster decommissioning phase, after all associated nodes have been successfully deprovisioned and cleaned up. This guarantees that exactly one `InfraEnv` instance exists per cluster lifecycle scope.
- **Resource Definition:** Integration of `ClusterOrder` specifications with BMaaS resource definitions, allowing tenants to request specific worker node hardware via `ClusterOrder.spec.nodeRequests[].resourceClass`.

## Out of Scope

- **Day-2 Autoscaling:** Automated dynamic, workload-driven scaling based on real-time resource utilization, CPU/memory pressure, or pod scheduling states. This remains out of scope for the current phase, as scaling down requires complex orchestration around cluster node draining and agent unbinding. Therefore, `BareMetalInstance` deletion and cleanup are restricted solely to explicit, manual administrator-initiated decommission or scale-down operations.
- **Virtual Machine Worker Nodes:** Provisioning VM-based worker nodes using this on-demand pattern (deferred to future VMaaS integrations).
- **Admin Tuning APIs:** Dedicated administrator-facing APIs for tweaking CaaS provisioning heuristics or retry thresholds.
- **Boot Optimization:** Network boot acceleration or advanced bare-metal caching strategies `[Jira: OSAC-2134]`.
- **Custom Networking Configuration:** Direct management of tenant-specific VLANs or advanced network routing by CaaS (CaaS will consume default BMaaS-provided network interfaces).

## User Stories

### Tenant Admin

- As a Tenant Admin, I want to create a `ClusterOrder` specifying supported `BareMetalInstanceTypes` (via `nodeRequests[].resourceClass`) for my worker nodes, so that my Kubernetes/OpenShift clusters are backed by high-performance physical hardware without me having to manage raw infrastructure directly.

- As a Tenant Admin, I want my resource usage, quotas, and billing to be tracked at the cluster level rather than at the individual bare-metal instance level, so that I can easily budget and monitor my organization's cloud spend.

### Tenant User

- As a Tenant User, I want my cluster creation and self-service management experience to remain entirely unchanged, so that I am never exposed to underlying `BareMetalInstances`, images, installers, or physical MAC addresses.

### Cloud Provider Admin

- As a Cloud Provider Admin, I want all CaaS-provisioned `BareMetalInstances` and standard `ComputeImages` to be hidden from tenant-facing views and catalogs, so that tenants cannot accidentally modify or delete underlying infrastructure nodes.

### Cloud Infrastructure Admin

- As a Cloud Infrastructure Admin, I want every deprovisioned bare-metal host to undergo a guaranteed, blocking cleanup (including deep disk wipe and network reset) by the BMaaS layer before being returned to the general inventory pool, so that I can prevent security leaks and configuration drift between different tenants.

- As a Cloud Infrastructure Admin, I want to ensure that no Assisted Installer, agent, or cluster-specific terminology is exposed within the BMaaS private APIs, so that the BMaaS service remains a clean, generic bare-metal-as-a-service provider.

## Assumptions

- **Ignition Support:** The BMaaS private API can ingest and reliably pass through standard discovery ignition payload as user data to the target physical host.
- **Agent Initialization:** The standard qcow2 `ComputeImage` provided is pre-configured to boot, process the ignition payload, and start the Assisted Installer agent without manual intervention.
- **Boundary of Failure:** BMaaS provisioning success is marked by an explicit, observable completion signal. BMaaS must emit a `BareMetalInstanceReady` status condition with `status=True` and reason `ProvisioningCompleted` on the `BareMetalInstance` resource.
  - This signal is only set after verifying that the standard qcow2 image has been successfully written to the physical disk, the custom ignition payload has been delivered, the host has completed its boot sequence, and active network readiness and IP address assignment on the boot interface are verified.
  - **Ownership of Failures:**
    - Any failure to write the image, deliver ignition, boot the host, or establish network readiness prior to emitting this `BareMetalInstanceReady` signal is classified as a **BMaaS-owned** infrastructure failure.
    - If the `BareMetalInstanceReady` signal is successfully emitted but the Assisted Installer agent fails to register with the control plane, the issue is classified and debugged as a **CaaS-level** software failure.

## Dependencies

- **MAC Address Status Exposure:** BMaaS must expose all physical MAC addresses of the host's interfaces as a structured list in the `BareMetalInstance` status subresource, clearly identifying the primary/boot interface MAC address as the canonical reference, to support the CaaS MAC normalization and matching contract `[Jira: OSAC-2308]`.
- **BareMetalInstanceType Definition:** The `BareMetalInstanceType` specifications and schema definitions must be finalized and available `[PR #59]`.
- **User Data Pass-through:** BMaaS private API must support the ingestion and pass-through of ignition configurations in the `BareMetalInstance` spec.

## Risks & Mitigations

- **Risk: Provisioning Latency and Timeouts**
  - *Description:* Writing standard qcow2 images to physical disks and performing complete boot cycles can take significant time, potentially causing CaaS timeouts.
  - *Mitigation:* Establish clear boundary lines. CaaS will separate "Bare-Metal Instance Ready" (marked by the BMaaS-emitted `BareMetalInstanceReady` status condition) from "Agent Registered" (CaaS complete), utilizing long-running, asynchronous reconciliation with generous timeout thresholds.

- **Risk: Cross-Tenant Data Contamination**
  - *Description:* During scale-down, a physical host previously used by Tenant A could be assigned to Tenant B with residual sensitive data on local storage.
  - *Mitigation:* BMaaS must mandate a synchronous, non-bypassable host sanitization flow (disk wipe and network state reset) immediately upon `BareMetalInstance` deletion, blocking the host's return to the active inventory until complete.

- **Risk: Cross-Tenant Agent Registration Races**
  - *Description:* Multiple agents registering from different hosts simultaneously might register to the wrong cluster control planes.
  - *Mitigation:* CaaS will provision exactly one unique `InfraEnv` per cluster (with node pools sharing this single `InfraEnv`), ensuring that registration endpoints and credentials are strictly isolated on a per-cluster basis using the Cluster UUID as the identity key.