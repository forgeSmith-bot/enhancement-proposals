# BareMetalInstancePool API

| Field       | Value   |
|-------------|---------|
| Author(s)   |         |
| Jira        | [OSAC-1565](https://issues.redhat.com/browse/OSAC-1565) |
| Date        | 2026-08-31 |

## Problem Statement

Tenants who need many identical bare metal hosts — for batch computing, cluster node pools, or similar workloads — must create and manage each BareMetalInstance individually. There is no way to declare a desired host count and have the platform maintain it, no single operation to scale capacity up or down, and no aggregate view of provisioning health across a group of hosts. Without a pool abstraction, tenants bear the full burden of tracking individual instance status, replacing failed hosts, and coordinating cleanup when a group is decommissioned.

## In Scope

- A new tenant-facing BareMetalInstancePool resource with full CRUD operations through the fulfillment API, allowing tenants to manage a group of identical BareMetalInstances as a single unit
- Declarative replica count: the pool provisions or deprovisions member BareMetalInstances to match the requested count when it is created or updated
- Network attachment: all pool members are provisioned on the network specified in the pool definition
- Aggregate pool status reflecting how many members are ready, progressing, or failed
- Pool deletion cascades to all member instances

## Out of Scope

- Heterogeneous pools — mixed hardware profiles or catalog items within a single pool
- Auto-scaling based on workload metrics — pools are manually scaled by updating the replica count

## User Stories

### Cloud Provider Admin

- Not affected by this feature.

### Cloud Infrastructure Admin

- Not affected by this feature.

### Tenant Admin

- As a Tenant Admin, I want to create a BareMetalInstancePool with a desired replica count, catalog item, and network so that my organization's bare metal hosts are provisioned as a group without per-instance management.
- As a Tenant Admin, I want to scale a pool up or down by updating its replica count so that I can adjust capacity with a single operation.
- As a Tenant Admin, I want to see aggregate pool status showing how many members are ready, progressing, or failed so that I can monitor provisioning health at a glance.
- As a Tenant Admin, I want deleting a pool to remove all its member instances so that no orphaned resources remain.

### Tenant User

- As a Tenant User, I want to create a BareMetalInstancePool with a desired replica count, catalog item, and network so that I can provision a group of identical bare metal hosts without managing each one individually.
- As a Tenant User, I want to scale a pool up or down by updating its replica count so that I can react to changing capacity needs without creating or deleting individual instances.
- As a Tenant User, I want to see the aggregate status of my pool — ready, progressing, and failed member counts — so that I can track provisioning progress and identify issues.

## Assumptions

- The pool abstraction provides meaningful operational value over managing individual BareMetalInstance resources directly. The Jira ticket notes this as an open question requiring a deliberate decision before implementation.
- The BareMetalInstance resource and its provisioning lifecycle are functional and available as the foundation for pool membership.

## Dependencies

- **BareMetalInstance API (OSAC-1118):** Pool members are BareMetalInstances. The existing bare metal provisioning lifecycle must be in place before pools can create and manage member instances.