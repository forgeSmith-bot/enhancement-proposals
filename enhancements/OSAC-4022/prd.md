# Secret Management — Encrypted Storage with Pluggable Backends

| Field       | Value   |
|-------------|---------|
| Author(s)   | Forge   |
| Jira        | [OSAC-4022](https://issues.redhat.com/browse/OSAC-4022) |
| Date        | 2026-08-13 |

## Problem Statement

Secrets such as CaaS cluster kubeconfigs, IdP client secrets, storage credentials, and SSH keys are stored as unencrypted fields within resource database tables. Database access exposes sensitive credentials in plaintext, there is no separation of secret lifecycle from resource lifecycle, no way to rotate encryption keys, and no uniform method for retrieving secrets — each resource type requires ad-hoc retrieval (e.g., GetKubeconfig, GetPassword). If unaddressed, the platform cannot enforce encryption at rest, cannot provide consistent access control over sensitive data, and cannot support alternative storage backends for secrets.

## In Scope

- A dedicated Secret resource with full CRUD operations and a uniform API for storing and retrieving secrets across all services (CaaS kubeconfigs, IdP client secrets, storage credentials, SSH keys)
- Encryption at rest for database-backed secrets, with support for encryption key rotation without requiring simultaneous re-encryption of all stored secrets
- Pluggable secret backends via a SecretClass abstraction, supporting database storage and hub (Kubernetes) backends; hub backend retrieves secret data on demand from Kubernetes clusters
- Secret payload is excluded from list responses — only individual secret retrieval returns the decrypted payload
- SecretReference integration so that existing per-resource secret retrieval (GetKubeconfig, GetPassword) is replaced by a uniform pattern
- Access control policies for secret access, enforcing tenant isolation
- CLI support for encryption key configuration

## Out of Scope

- External secret manager integration (e.g., HashiCorp Vault) — planned as a future enhancement
- Automated secret rotation — planned as a future enhancement
- SSH key cloud-init injection for VMs — VMaaS-specific, tracked separately in OSAC-51

## User Stories

### Cloud Infrastructure Admin

- As a Cloud Infrastructure Admin, I want secrets encrypted at rest so that database access does not expose sensitive credentials in plaintext.
- As a Cloud Infrastructure Admin, I want to rotate encryption keys without re-encrypting all secrets simultaneously so that key rotation does not require downtime or a bulk migration operation.
- As a Cloud Infrastructure Admin, I want to configure encryption keys via CLI flags so that I can manage key material as part of platform deployment.
- As a Cloud Infrastructure Admin, I want to configure pluggable secret backends so that I can choose between database storage and external secret stores based on infrastructure requirements.

### Cloud Provider Admin

- As a Cloud Provider Admin, I want access control policies to enforce tenant isolation on secrets so that tenants cannot access each other's sensitive data.
- As a Cloud Provider Admin, I want to see secrets scoped to each tenant so that I can verify tenant isolation and manage cross-tenant secret governance.

### Tenant Admin

- As a Tenant Admin, I want to store and retrieve IdP client secrets through the secrets API so that I can manage identity provider configuration without relying on ad-hoc retrieval methods.

### Tenant User

- As a Tenant User, I want to retrieve my CaaS cluster kubeconfig through the secrets API so that I have a uniform, predictable way to access cluster credentials.
- As a Tenant User, I want to retrieve passwords and other credentials through the same secrets API so that I do not need to learn different retrieval methods for each resource type.
- As a Tenant User, I want secret payloads excluded from list responses so that browsing secrets does not inadvertently expose sensitive data.

## Assumptions

- The existing ad-hoc secret retrieval RPCs (GetKubeconfig, GetPassword) can be replaced by SecretReference-based lookups without breaking existing client workflows, since OSAC does not support in-place upgrades.

## Dependencies

- **OSAC-1330 (Type-safe resource references):** Provides the SecretReference type used to link resources to their associated secrets. Must land before SecretReference integration can replace ad-hoc RPCs.