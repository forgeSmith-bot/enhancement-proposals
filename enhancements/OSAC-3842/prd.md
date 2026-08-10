# CaaS Bare Metal Worker Node Provisioning

| Field       | Value   |
|-------------|---------|
| Author(s)   | [Open Question: Who is the feature owner? — meeting participants include Avishay, Adrien, Nick] |
| Jira        | https://redhat.atlassian.net/browse/OSAC-3842 |
| Date        | 2026-08-10 |

## 1. Problem Statement

CaaS needs bare metal compute to back cluster worker nodes. Today, worker node hosts are provisioned through a cron job that maintains a static pool of pre-booted agents. This approach wastes resources on idle hosts that may never be assigned to a cluster, is difficult to size correctly because demand is unpredictable, and couples cluster-specific concepts (agents, InfraEnvs) into the BMaaS layer where they do not belong. If unaddressed, CaaS continues to over-provision hardware, and BMaaS accumulates domain knowledge about cluster internals that makes both systems harder to evolve independently.

## 2. Goals and Non-Goals

### 2.1 Goals

- Tenants can create clusters backed by bare metal worker nodes by specifying a BareMetalInstanceType in their ClusterOrder — without managing hardware, images, or agents directly.
- CaaS provisions bare metal worker nodes on-demand when a cluster is created or scaled up, eliminating idle pre-provisioned host pools.
- CaaS deprovisions bare metal worker nodes when a cluster is deleted or scaled down, returning hosts to BMaaS inventory.
- BMaaS has no knowledge of clusters, agents, or Assisted Installer — it provisions a host with an image and user data.

### 2.2 Non-Goals

- Day-2 autoscaling — automated scale-up is straightforward; scale-down requires careful timing around workload drain and is deferred.
- VM worker nodes — the same provisioning pattern applies but is future work.
- Admin API for tuning CaaS provisioning behavior (e.g., retry policies, timeouts).
- Boot-over-network optimization (OSAC-2134).
- Networking configuration — CaaS uses whatever networking API BMaaS provides.
- Per-BareMetalInstance billing — billing and quota are tracked at the cluster level, not individual bare metal instances.

## 3. Requirements

### 3.1 Functional Requirements

#### Cluster Creation with Bare Metal Workers

- **FR-1:** Tenant Users can create a ClusterOrder specifying BareMetalInstanceTypes for worker nodes (via the resource class field in node requests). The cluster is provisioned with bare metal worker nodes matching the requested types.

- **FR-2:** Tenant Users' cluster creation experience is unchanged — BareMetalInstances, images, agents, and other infrastructure details are not visible to tenants in any API response, UI view, or CLI output.

#### Worker Node Provisioning

- **FR-3:** When a ClusterOrder requests bare metal worker nodes, each requested worker node is automatically provisioned and joined to the cluster without tenant intervention. The tenant observes workers transitioning from requested to ready.

#### Worker Node Deprovisioning

- **FR-4:** When a tenant deletes a cluster or scales down a node pool, the affected worker nodes are removed from the cluster and their underlying bare metal hosts are returned to inventory. No manual cleanup is required.

#### Status Visibility

- **FR-5:** Tenants can see the provisioning progress of their cluster's worker nodes at the cluster level (e.g., number of workers requested, number ready) without seeing BareMetalInstance details. [Open Question: What specific status fields and failure messages are surfaced at the cluster level? — using best guess: existing ClusterOrder status mechanism surfaces node provisioning progress]

#### Tenant Isolation

- **FR-6:** BareMetalInstances and images created by CaaS for worker node provisioning are hidden from tenants — they do not appear in tenant-scoped list, get, or search operations.

### 3.2 Non-Functional Requirements

- **NFR-1:** No Assisted Installer, agent, or InfraEnv terminology appears in BMaaS APIs or tenant-facing APIs. BMaaS treats the image and user data as opaque payloads.

- **NFR-2:** Every bare metal host that is deprovisioned must be fully cleaned by BMaaS (disk wipe, network reset, credential removal) before it is returned to inventory. This is a security requirement — hosts cycle between tenants and must not carry residual data or configuration from previous use.

- **NFR-3:** Worker nodes provisioned for different clusters must not interfere with each other during concurrent provisioning or scale operations. Tenant isolation is maintained throughout the provisioning lifecycle.

## 4. Acceptance Criteria

- [ ] A Tenant User can create a ClusterOrder specifying BareMetalInstanceTypes for worker nodes, and the cluster is provisioned with bare metal worker nodes matching the requested types.
- [ ] BareMetalInstances and images created for worker node provisioning are not visible to tenants in any API, UI, or CLI surface.
- [ ] When a cluster is deleted or a node pool is scaled down, the corresponding BareMetalInstances are deleted and BMaaS cleans the hosts (disk wipe, network reset) before returning them to inventory.
- [ ] No Assisted Installer, agent, or InfraEnv terminology appears in BMaaS APIs or tenant-facing APIs.
- [ ] Tenants can see worker node provisioning progress at the cluster level without seeing BareMetalInstance details.
- [ ] Worker nodes provisioned via different clusters cannot interfere with each other's agent registration.

## 5. Assumptions

- BMaaS supports passing user data (ignition) through the BareMetalInstance spec and applying it at boot time. If this capability does not exist, it is a prerequisite that must be delivered before this feature.
- BMaaS host cleanup (disk wipe, network reset, credential removal) on BareMetalInstance deletion is an existing BMaaS responsibility. This feature relies on it but does not implement it.
- The qcow2 OS image (e.g., RHCOS) used for worker nodes is pre-registered as a ComputeImage in BMaaS. CaaS references it by identifier, not by URL.

## 6. Dependencies

- **OSAC-2308 — MAC address in BareMetalInstance status:** CaaS requires the MAC address from BareMetalInstance status to correlate provisioned hosts with registered agents. This is a prerequisite.
- **BareMetalInstanceType EP (PR #59):** Defines the BareMetalInstanceType resource that tenants reference in ClusterOrder node requests. Must be available before tenants can select bare metal hardware configurations.
- **User data pass-through in BareMetalInstance spec:** BMaaS must support an opaque user data field in the BareMetalInstance spec so CaaS can pass discovery ignition. If not yet implemented, this is a prerequisite.
- **BareMetalInstance references ComputeImage:** Existing capability. CaaS references a pre-registered qcow2 ComputeImage when creating BareMetalInstances.

## 7. Risks

### 7.1 Agent registration failure after successful host boot

BMaaS considers provisioning complete once the host boots. If the discovery agent fails to register (due to ignition errors, network issues, or image problems), the failure is invisible to BMaaS and surfaces only at the CaaS level. Debugging requires CaaS-level tooling.

- **Owner:** CaaS team
- **Mitigation:** CaaS must implement timeout and retry logic for agent registration and surface failures clearly at the cluster level.

### 7.2 Dependency sequencing risk

Multiple prerequisites (MAC address exposure, user data pass-through, BareMetalInstanceType) must land before this feature can be delivered end-to-end. Delays in any dependency block the full provisioning flow.

- **Owner:** To be determined
- **Mitigation:** Identify the critical path across dependencies and sequence work accordingly.

## 8. Open Questions

### 8.1 What specific cluster-level status fields and messages are surfaced to tenants during worker node provisioning?

- **Owner:** CaaS team
- **Impact:** Affects FR-5 and acceptance criteria. Tenants need to understand provisioning progress and failure reasons without seeing BareMetalInstance details.

### 8.2 Does the cluster creation UI need updates to display BareMetalInstanceTypes as selectable resource classes?

- **Owner:** To be determined
- **Impact:** Affects whether UI work is in scope. If BareMetalInstanceTypes are surfaced through existing catalog item mechanisms, no special UI work may be needed.

### 8.3 Who is the feature owner (PRD Author)?

- **Owner:** To be determined
- **Impact:** Affects the Author field in the metadata table.

## OSAC Dimensions

### Services

This feature applies to **CaaS** (primary) and **BMaaS** (dependency). CaaS owns the provisioning orchestration; BMaaS provides the bare metal host lifecycle (create, boot, clean, return to inventory). BMaaS is not modified by this feature — CaaS consumes its existing private API.

### Personas

| Persona | Interaction |
|---------|-------------|
| Tenant User | Creates ClusterOrders specifying BareMetalInstanceTypes for worker nodes. Sees cluster-level provisioning status. Does not see or interact with BareMetalInstances, images, or agents. |
| Tenant Admin | Same as Tenant User for cluster operations. |
| Cloud Provider Admin | Not directly affected by this feature. Manages BareMetalInstanceTypes and ComputeImages through existing BMaaS admin capabilities (prerequisite work, not this feature). |
| Cloud Infrastructure Admin | Not affected. Bare metal inventory management is an existing BMaaS responsibility. |

### Provisioning

CaaS orchestrates the full provisioning lifecycle: creates BareMetalInstances via the BMaaS private API, correlates agents via MAC address, approves and binds agents to the cluster. Deprovisioning deletes BareMetalInstances; BMaaS handles host cleanup. The provisioning boundary is clear: BMaaS is done when the host boots; CaaS is responsible for everything after (agent registration, correlation, binding).

### E2E Testing

The end-to-end provisioning flow (ClusterOrder → BareMetalInstance creation → agent registration → agent binding → cluster ready) and deprovisioning flow (cluster delete → BareMetalInstance delete → host cleanup) need E2E test coverage.

### Documentation

New provisioning pattern (bare metal worker nodes via ClusterOrder) needs documentation for tenants. The internal CaaS-BMaaS integration pattern needs operator documentation.

---

## Unattended PRD Run Complete

Phases: ingest ✓ → exemplars ✓ → dimensions ✓ → self-clarify ✓ → draft ✓ → review ✓ → context ✓
Artifacts: .artifacts/prd/OSAC-3842/
Score: 10/10 (WHAT:2 WHY:2 UF:2 RS:2 T:2)
Open questions: 3 items need human resolution
Revision rounds: 1
Result: PRD generated with 6 functional requirements, 3 non-functional requirements, 6 acceptance criteria, 3 assumptions, 4 dependencies, 2 risks, and 3 open questions.

Next steps:
- Review open questions in session-context.md
- Run /revise to refine the PRD interactively
- Run /publish when ready for external review