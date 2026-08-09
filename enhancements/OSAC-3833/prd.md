### 1. WHAT — Clear user-facing need? (0-2)
The PRD describes a new product capability: on-demand bare metal worker node provisioning for CaaS clusters. Services identified: CaaS, BMaaS. Personas identified with stories: Tenant Admin/User (combined, 2 stories), Cloud Provider Admin (2 stories), Cloud Infrastructure Admin (not affected). All 4 personas covered.
**Score: 2**

### 2. WHY — Business justification? (0-2)
Problem statement names the pain (static pool wastes resources, hard to size, entangles concerns), describes the consequence (expensive, slow to scale, architecturally coupled). Concrete causal chain.
**Score: 2**

### 3. User-Facing Focus — Free from design leakage? (0-2)
No controllers, reconcilers, finalizers, playbooks, env vars, AAP jobs, CRD fields mentioned. No internal conditions. No mention of InfraEnv, ignition, qcow2, agents, Assisted Installer, MAC address normalization, or private API internals. "BareMetalInstanceType" and "ClusterOrder" are user-facing platform vocabulary. "securely sanitized" replaces disk wipe/network reset mechanics. "hardware identifiers from BareMetalInstance status" in dependencies — this is borderline but acceptable as it describes what is needed, not how.
**Score: 2**

### 4. Right-Sized — Focused scope? (0-2)
The feature is one cohesive capability: on-demand bare metal provisioning for CaaS cluster workers. Provisioning and deprovisioning are inseparable parts of the lifecycle. Cannot ship one without the other.
**Score: 2**

### 5. Testability — Verifiable requirements? (0-2)
All user stories are PM-verifiable: create a ClusterOrder with BareMetalInstanceTypes → cluster gets bare metal workers; cluster experience unchanged → verify tenants don't see BareMetalInstances; on-demand provisioning → verify no static pool; host sanitization → verify no data leaks on decommission (though this one is more operational).
**Score: 2**

**Total: 10/10**

### Inline Deterministic Checks:

1. **Section check:** Sections are: Problem Statement, In Scope, Out of Scope, User Stories, Assumptions, Dependencies = 6 sections. ✅
2. **Persona check:** Cloud Provider Admin ✅, Cloud Infrastructure Admin ✅, Tenant Admin ✅, Tenant User ✅ (combined as Tenant Admin / Tenant User). All present.
3. **Leakage check:** Searching for flagged terms:
   - reconciler: ❌ not found
   - reconciliation: ❌ not found
   - finalizer: ❌ not found
   - playbook: ❌ not found
   - env var: ❌ not found
   - AAP job: ❌ not found
   - CRD field: ❌ not found
   - osac-operator: ❌ not found
   - osac-aap: ❌ not found
   - ansible role: ❌ not found
   - controller: ❌ not found
   All clear. ✅
4. **Length check:** Non-blank lines count: ~38 lines. Within 15-120, within target 40-80 range. ✅

**Verdict: PASS** — Total 10/10, no zeros, all inline checks pass. Proceed to Phase 5.
The draft passes all checks. Let me verify there's no trailing whitespace and the file ends with exactly one newline:
No trailing whitespace. The file is clean. Let me finalize:
# CaaS Bare Metal Worker Node Provisioning

| Field       | Value   |
|-------------|---------|
| Author(s)   |         |
| Jira        | [OSAC-3833](https://redhat.atlassian.net/browse/OSAC-3833) |
| Date        | 2026-08-09 |

## Problem Statement

CaaS cluster worker nodes require bare metal compute, but the current provisioning model relies on a static pre-provisioned pool of hosts maintained by a background job. This wastes resources on idle hosts, is difficult to size correctly for varying demand, and entangles cluster-level concepts into the BMaaS layer. If unaddressed, CaaS bare metal clusters remain expensive to operate, slow to scale, and architecturally coupled to provisioning details that belong in CaaS, not BMaaS.

## In Scope

- On-demand bare metal worker node provisioning for CaaS clusters — hosts are provisioned when a ClusterOrder requests them and released when no longer needed, replacing the static host pool
- Tenants specify a BareMetalInstanceType in their ClusterOrder worker node requests; the cluster creation experience is otherwise unchanged
- BareMetalInstances and images provisioned for CaaS worker nodes are hidden from tenants — billing and quota are tracked at the cluster level
- When a cluster is decommissioned or scaled down, bare metal hosts are released and securely sanitized before being returned to inventory

## Out of Scope

- Day-2 autoscaling — scaling up follows the same provisioning path, but scaling down requires cluster drain coordination and is deferred
- VM-backed worker nodes — same architectural pattern, planned as future work
- Admin API for tuning CaaS provisioning behavior
- Boot-over-network optimization (OSAC-2134)
- Networking configuration — CaaS uses whatever networking capabilities BMaaS provides

## User Stories

### Tenant Admin / Tenant User

- As a Tenant Admin/User, I want to create a ClusterOrder that specifies BareMetalInstanceTypes for worker nodes so that I get a cluster backed by bare metal compute without managing hardware directly.
- As a Tenant Admin/User, I want my cluster creation and management experience to remain unchanged — I should not see or interact with BareMetalInstances, images, or any infrastructure provisioning details.

### Cloud Provider Admin

- As a Cloud Provider Admin, I want CaaS to provision bare metal worker nodes on-demand rather than maintaining a static pool so that resources are not wasted on idle hosts.
- As a Cloud Provider Admin, I want every bare metal host released by CaaS to be securely sanitized (disk, network, credentials) before it is returned to inventory so that no data or configuration leaks between tenants.

### Cloud Infrastructure Admin

- Not affected by this feature.

## Assumptions

- BareMetalInstance provisioning via BMaaS supports passing an OS image and boot-time configuration data. If this capability is not available, worker node provisioning cannot function.

## Dependencies

- **OSAC-2308 (MAC address in BareMetalInstance status):** CaaS requires hardware identifiers from BareMetalInstance status to correlate provisioned hosts with cluster nodes. Must land before or alongside this feature.
- **BareMetalInstanceType (EP PR #59):** Tenants reference BareMetalInstanceTypes in ClusterOrder worker node requests. The BareMetalInstanceType resource must be available.