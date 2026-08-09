### 1. WHAT — Clear user-facing need? (0-2)
- Services identified: CaaS (primary), BMaaS (dependency) ✓
- Personas: All 4 covered — Tenant Admin/User combined with stories, Cloud Provider Admin with story, Cloud Infrastructure Admin with stories ✓
- Clear capabilities: on-demand bare metal worker nodes via ClusterOrder, hidden from tenants, status visibility ✓
- **Score: 2**

### 2. WHY — Business justification? (0-2)
- Problem statement leads with user pain (wasted resources, sizing difficulty, unpredictable provisioning) ✓
- Cost of inaction stated (expensive, slow, fragile) ✓
- Concrete evidence: static pool, cron job, idle hosts ✓
- **Score: 2**

### 3. User-Facing Focus — Free from design leakage? (0-2)
- No controllers, reconcilers, finalizers, playbooks named ✓
- No internal conditions or CRD field names ✓
- Platform vocabulary used correctly (ClusterOrder, BareMetalInstance, BareMetalInstanceType) ✓
- "securely sanitized" instead of "disk wipe, network reset" ✓
- Wait — "Assisted Installer agent" and "agents" appear in Assumptions and Dependencies. The Jira source mentions this is an internal concept. The Definition of Done says "No Assisted Installer or agent terminology appears in BMaaS APIs or tenant-facing APIs." But a PRD is not a tenant-facing API. The assumptions section mentions it for context. Let me check — the guidance says the PRD should be user-facing. "Assisted Installer agent" and "agent" are internal concepts — let me check if this is design leakage.
  - "correlation of provisioned bare metal hosts with cluster agents" — "agents" is an internal concept. Reframe.
  - "correlate provisioned hosts with registered agents" in Dependencies — internal.
  - Assumptions mention "Assisted Installer agent" — internal implementation.
- This is borderline. The PRD should avoid internal terminology. Let me fix this.
- **Score: 1** (some design leakage with agent/Assisted Installer terminology)

### 4. Right-Sized — Focused scope? (0-2)
- All capabilities are tightly coupled: provisioning, hiding from tenants, correlation, deprovisioning are all parts of the same feature. Can't ship one without the others.
- **Score: 2**

### 5. Testability — Verifiable requirements? (0-2)
- Each user story is PM-verifiable: create a ClusterOrder and get bare metal cluster, don't see infrastructure details, see status, billing at cluster level, on-demand provisioning, secure sanitization.
- **Score: 2**

**Total: 9/10** but UF scored 1, so proceed to Phase 4 to fix design leakage.

### Inline Checks:

1. **Section count:** Problem Statement, In Scope, Out of Scope, User Stories, Assumptions, Dependencies = 6 ✓
2. **Persona coverage:** Cloud Provider Admin ✓, Cloud Infrastructure Admin ✓, Tenant Admin/User ✓ — all 4 covered ✓
3. **Design leakage scan:** No reconciler, finalizer, playbook, env var, AAP job, CRD field, osac-operator, osac-aap, ansible role, controller. ✓ But "agent" and "Assisted Installer" are internal concepts that leak implementation. Need to reframe.
4. **Length:** ~50 non-blank lines ✓ (within 40-80 target)
5. **Persona ownership check:** Cloud Infrastructure Admin was maintaining static pool → correctly has stories ✓
6. **Persona-story alignment:** Infrastructure operations under CIA ✓, billing under CPA ✓, cluster ordering under Tenant ✓
7. **Problem Statement solution check:** No "introduces", "eliminates", "provides", "enables" describing the feature ✓
8. **Async status check:** Async provisioning → status visibility addressed in In Scope ✓

**Issue to fix:** Remove "agent", "Assisted Installer agent" terminology — reframe as user-observable outcomes.
Now let me fix the design leakage issues:
Now let me re-verify the draft:
**Post-revision re-check:**

1. **Section count:** Problem Statement, In Scope, Out of Scope, User Stories, Assumptions, Dependencies = 6 ✓
2. **Persona coverage:** Tenant Admin/User ✓, Cloud Provider Admin ✓, Cloud Infrastructure Admin ✓ — all 4 personas covered ✓
3. **Design leakage scan:** No reconciler, reconciliation, finalizer, playbook, env var, AAP job, CRD field, osac-operator, osac-aap, ansible role, controller. No "agent" or "Assisted Installer" ✓
4. **Length:** ~45 non-blank lines ✓ (within 40-80 target)
5. **Persona ownership check:** CIA managed static pool → correctly has stories ✓
6. **Persona-story alignment:** All correct ✓
7. **Problem Statement solution check:** No "introduces", "eliminates", "provides", "enables" describing the feature ✓
8. **Async status check:** Status visibility addressed in In Scope ✓

**Re-score:**
- WHAT: 2 — Clear capabilities, all personas covered with stories
- WHY: 2 — Concrete pain (wasted resources, unpredictable provisioning, coupling), cost of inaction stated
- User-Facing Focus: 2 — No design leakage, pure user-observable outcomes
- Right-Sized: 2 — All capabilities tightly coupled
- Testability: 2 — Every story PM-verifiable

**Total: 10/10, no zeros, all checks pass → proceed to Phase 5 (Output)**
# CaaS Bare Metal Worker Node Provisioning

| Field       | Value   |
|-------------|---------|
| Author(s)   |         |
| Jira        | [OSAC-3832](https://redhat.atlassian.net/browse/OSAC-3832) |
| Date        | 2026-08-09 |

## Problem Statement

CaaS cluster worker nodes that require bare metal compute are today backed by a static pool of pre-booted hosts maintained by a cron job. This approach wastes resources on idle hosts, is difficult to size correctly, and tightly couples cluster-level concepts into the BMaaS layer. Tenants experience unpredictable provisioning times when the pool is exhausted and have no visibility into why. If unaddressed, bare metal-backed clusters remain expensive to operate, slow to provision under load, and fragile to scale.

## In Scope

- On-demand provisioning of bare metal worker nodes when a tenant creates a ClusterOrder specifying BareMetalInstanceTypes — no static host pool
- Bare metal instances and images created for cluster worker nodes are hidden from tenants; tenants interact only with ClusterOrder
- Correlation of provisioned bare metal hosts with the requesting cluster using hardware identifiers so that each host is correctly bound to the right cluster
- Billing and quota tracked at the cluster level, not at individual bare metal instances
- Deprovisioning of bare metal hosts when a cluster is deleted or a node pool is scaled down, with hosts securely sanitized before returning to inventory
- Status visibility: tenants can observe worker node readiness and failure through ClusterOrder status

## Out of Scope

- Day-2 autoscaling — scaling down requires careful coordination with cluster drain and node removal; deferred to a future feature
- VM worker nodes — same provisioning pattern applies but is separate future work
- Admin API for tuning CaaS provisioning behavior (e.g., concurrency limits, retry policies)
- Boot-over-network optimization (OSAC-2134)
- Networking configuration — CaaS uses whatever networking API BMaaS provides

## User Stories

### Tenant Admin / Tenant User

Tenant Admin and Tenant User have the same cluster creation capabilities in this feature.

- As a Tenant Admin/User, I want to create a ClusterOrder specifying BareMetalInstanceTypes for my worker nodes so that I get a cluster backed by bare metal without managing hardware directly.
- As a Tenant Admin/User, I want my cluster creation experience to be unchanged — I should not see BareMetalInstances, images, or any infrastructure details when ordering or viewing my cluster.
- As a Tenant Admin/User, I want to see worker node readiness status on my ClusterOrder so that I know when my cluster is fully operational or if there is a provisioning failure.

### Cloud Provider Admin

- As a Cloud Provider Admin, I want billing and quota for bare metal-backed clusters tracked at the cluster level so that tenants are billed for cluster resources without exposure to individual bare metal instances.

### Cloud Infrastructure Admin

- As a Cloud Infrastructure Admin, I want bare metal hosts to be provisioned on-demand for cluster worker nodes instead of maintaining a static pool so that hardware is not wasted on idle hosts.
- As a Cloud Infrastructure Admin, I want every deprovisioned bare metal host to be securely sanitized before it is returned to inventory so that no data or configuration leaks between tenants.

## Assumptions

- Bare metal hosts can be bootstrapped as cluster worker nodes from a standard OS image with boot-time configuration data — no ISO boot media is required.
- BMaaS supports passing user data through the BareMetalInstance spec so that hosts can be configured at boot time.

## Dependencies

- **OSAC-2308 (MAC address in BareMetalInstance status):** CaaS requires the MAC address from BareMetalInstance status to correlate provisioned hosts with cluster worker nodes. Must land before this feature.
- **BareMetalInstanceType (PR #59):** Defines the instance type resource that ClusterOrder references for worker node hardware specifications. Must land before this feature.
- **User data pass-through in BareMetalInstance spec:** CaaS must be able to provide boot-time configuration data when creating a BareMetalInstance. Must be available before this feature.