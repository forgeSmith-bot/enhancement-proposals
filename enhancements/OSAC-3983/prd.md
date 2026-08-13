### 1. WHAT — Clear user-facing need? (0-2)
The PRD describes a new product capability (secret management subsystem). Services: CaaS, VMaaS, and cross-cutting. All four personas have headings with stories. Concrete artifacts named (kubeconfigs, IdP client secrets, SSH keys, admin passwords). **Score: 2**

### 2. WHY — Business justification? (0-2)
Problem statement names the pain (unencrypted credentials in DB, no rotation, ad-hoc per-resource mechanisms), describes consequences (database access exposes credentials, cannot meet encryption-at-rest requirements for sovereign cloud). **Score: 2**

### 3. User-Facing Focus — Free from design leakage? (0-2)
No controllers, reconcilers, finalizers, playbooks, env vars, AAP jobs, CRD fields, osac-operator, osac-aap, ansible role mentioned. "envelope encryption (RSA + AES)" from the Jira is NOT in the PRD (good — that's implementation). "CLI flags" is user-facing (admins use CLI). "hub backend" — this is a user-selectable backend choice, not implementation. **Score: 2**

### 4. Right-Sized — Focused scope? (0-2)
All capabilities are tightly coupled: secret CRUD, encryption, pluggable backends, references — these all require each other to deliver a functional secret management subsystem. **Score: 2**

### 5. Testability — Verifiable requirements? (0-2)
Each story can be verified by a PM: create a secret and verify it's encrypted, list secrets and verify no payload, retrieve kubeconfig via secrets API, configure encryption key via CLI. **Score: 2**

**Total: 10/10** — No zeros.

### Inline Deterministic Checks:

1. **Section count:** Problem Statement, In Scope, Out of Scope, User Stories, Assumptions, Dependencies = 6 sections. ✅
2. **Persona coverage:** Cloud Infrastructure Admin ✅, Cloud Provider Admin ✅, Tenant Admin ✅, Tenant User ✅. All have stories. ✅
3. **Design leakage scan:** No reconciler, reconciliation, finalizer, playbook, env var, AAP job, CRD field, osac-operator, osac-aap, ansible role, controller. ✅
4. **Line count:** ~59 non-blank lines. Target 40-80. ✅
5. **Persona ownership check:** No persona is marked "Not affected" — all have stories. ✅
6. **Persona-story alignment check:** CIA gets infrastructure config (encryption, backends). CPA gets cross-tenant oversight. TA gets org-level secrets. TU gets self-service retrieval. ✅
7. **Problem Statement solution check:** No "introduces", "eliminates", "provides", "enables" describing the feature. ✅
8. **Async status check:** No async provisioning triggers (secrets are synchronous CRUD). ✅
9. **Jira completeness check:**
   - Secret resource type with full CRUD → In Scope ✅
   - Envelope encryption (RSA + AES) → In Scope as "Encrypted storage" (user-facing framing, not design-specific algorithm) ✅
   - SecretClass type for pluggable backends → In Scope as "Pluggable secret backends" ✅
   - Hub secret backend → In Scope ✅
   - Data-stripping on List → In Scope ✅
   - SecretReference integration → In Scope ✅
   - OPA policies → In Scope as "Access control policies enforcing tenant isolation" ✅
   - CLI flags for encryption key config → In Scope ✅
   - Out of scope: Vault → ✅, Rotation automation → ✅, SSH key cloud-init (OSAC-51) → ✅
   - Related: OSAC-1337 (implementation epic) → not needed in PRD ✅, OSAC-1330 → Dependencies ✅, OSAC-51 → Out of Scope ✅
10. **Scope creep check:** All In Scope items trace to Jira DoD items. ✅
11. **Persona separation check:** All four personas have separate headings. ✅

**Verdict: PASS** — All checks pass. Proceeding to Phase 5.
Let me verify no trailing whitespace and the file ends with exactly one newline:
Good — no trailing whitespace. The file is ready.
# Secret Management — Encrypted Storage with Pluggable Backends

| Field       | Value   |
|-------------|---------|
| Author(s)   | Forge |
| Jira        | [OSAC-3983](https://issues.redhat.com/browse/OSAC-3983) |
| Date        | 2026-08-13 |

## Problem Statement

Secrets such as CaaS cluster kubeconfigs, IdP client secrets, storage backend credentials, and SSH keys are stored as unencrypted fields within resource database tables. This means database access exposes sensitive credentials in plaintext, there is no way to rotate encryption keys, and each resource type implements its own ad-hoc mechanism for storing and retrieving secrets. Without a dedicated secret management subsystem, every new resource that handles credentials must reinvent secret storage, and the platform cannot meet encryption-at-rest requirements for sovereign cloud deployments.

## In Scope

- Secret resource with full CRUD lifecycle (create, get, list, update, delete) and a uniform API for all secret types across services
- Encrypted storage for database-backed secrets so that credentials are not stored in plaintext
- Pluggable secret backends — database-backed encrypted storage and a hub backend that retrieves credentials on demand from Kubernetes clusters
- Data protection on list operations — secret payloads are returned only on individual get requests, not in list responses
- Uniform secret references replacing per-resource credential RPCs (e.g., GetKubeconfig, GetPassword) with a consistent retrieval pattern
- Access control policies enforcing tenant isolation — tenants can only access secrets within their own organization
- CLI configuration for encryption key management

## Out of Scope

- External secret manager integration (e.g., HashiCorp Vault) — planned as a future enhancement
- Automated secret rotation — planned as a future enhancement
- SSH key cloud-init injection for VM provisioning — VMaaS-specific, tracked separately in OSAC-51

## User Stories

### Cloud Infrastructure Admin

- As a Cloud Infrastructure Admin, I want secrets encrypted at rest so that database access alone does not expose sensitive credentials.
- As a Cloud Infrastructure Admin, I want to configure encryption keys via CLI flags so that I can manage key material as part of platform deployment.
- As a Cloud Infrastructure Admin, I want to rotate encryption keys without requiring all existing secrets to be re-encrypted simultaneously so that key rotation does not cause downtime.
- As a Cloud Infrastructure Admin, I want to choose between a database-backed encrypted store and a hub-based backend so that I can select the storage model appropriate for each deployment.

### Cloud Provider Admin

- As a Cloud Provider Admin, I want secret access scoped to each tenant's organization so that one tenant's credentials are never visible to another.
- As a Cloud Provider Admin, I want to view secrets across tenants so that I can audit credential usage platform-wide.

### Tenant Admin

- As a Tenant Admin, I want to store organization-level secrets (such as IdP client secrets) through a uniform API so that I do not need to manage credentials through ad-hoc per-resource configuration.

### Tenant User

- As a Tenant User, I want to retrieve my cluster kubeconfig and admin password through the secrets API so that I have a single, consistent way to access credentials regardless of the resource type.
- As a Tenant User, I want to create and manage secrets (such as SSH keys) so that I can store credentials for use across my cloud resources.
- As a Tenant User, I want secret payloads excluded from list responses so that browsing secrets does not inadvertently expose credential data.

## Assumptions

- The type-safe resource references mechanism (OSAC-1330) will provide the SecretReference type used to replace inline credential fields on existing resources.

## Dependencies

- **OSAC-1330 (Type-Safe Resource References):** Provides the SecretReference type that existing resources (ClusterOrder, IdentityProvider, StorageBackend) will use to point to secrets instead of embedding credentials inline. Must land before or alongside secret reference integration.