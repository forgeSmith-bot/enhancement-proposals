# Expose Inventory Metadata in BareMetalInstance Status

| Field | Value |
| :--- | :--- |
| Author(s) | OSAC Product Team |
| Jira | OSAC-2308 (Parent Epic) / OSAC-3254 (Child Status Tracking Issue) |
| Milestone | v0.2 |
| Date | 2026-07-28 |

## Problem Statement

When a BareMetalInstance is provisioned, the bare metal fulfillment service assigns a physical host from the backend inventory. However, none of the host's identifying information (such as the boot MAC address or primary IP address) is surfaced back to the BareMetalInstance status.

This creates a critical disconnect for consumers like Cluster as a Service (CaaS). During cluster installation, the Assisted Installer agent registers from the newly booted physical host using its MAC address, but CaaS has no way of programmatically correlating this agent with the provisioned BareMetalInstance. As a result, CaaS cannot automatically map agents to instances, forcing manual intervention by administrators to correlate hosts, which significantly delays cluster deployment and increases operational overhead. Additionally, Tenant Users cannot self-service connect to their provisioned instances because their network identities (IP and MAC) are completely hidden, leading to additional support requests and a suboptimal user experience.

## In Scope

- **Network Metadata Exposure:** Surfacing the physical host's boot MAC address and primary IP address (when available in the inventory backend) in the BareMetalInstance status once the instance is provisioned. The primary IP address is expected to be reachable on the tenant's VirtualNetwork, rather than only on an infrastructure-level network, enabling self-service Tenant User SSH connections.
- **API Availability:** Exposing this metadata via both the Get and List endpoints of the OSAC BareMetalInstance API so that downstream services and automation can programmatically consume them.
- **CLI Support:** Displaying the propagated MAC address and IP address in the OSAC CLI when listing or describing a BareMetalInstance.
- **Seamless Platform Integration:** CaaS can automatically correlate Assisted Installer agents with provisioned BareMetalInstances using the exposed boot MAC address, eliminating manual host pairing.
- **Technical Documentation:** Updating the user-facing API reference and CLI guides to cover the newly exposed boot MAC address and primary IP address fields in the BareMetalInstance status.
- **Tenant Isolation and Security:** Strict namespace boundaries ensuring that a Tenant User can only view or list metadata (boot MAC address and primary IP) for BareMetalInstances belonging to their own tenant.

## Out of Scope

- **Full Hardware Specifications Exposure:** Exposing comprehensive hardware specifications such as CPU cores, RAM size, disk layout, or GPU details in the status field (this is managed via BareMetalInstanceType specs) [Jira: OSAC-2308].
- **Inventory Backend Modifications:** Modifying or extending the schemas, APIs, or database of the existing inventory backend is out of scope; the feature consumes existing inventory data as-is [Jira: OSAC-2308].
- **DHCP Configuration and IPAM (IP Address Management):** Provisioning or configuring DHCP reservations or managing active network IP assignments (the metadata is read-only).
- **Assisted Installer Agent Lifecycle:** Managing, deploying, or troubleshooting the Assisted Installer agents running on the bare metal hosts.
- **Automatic DNS Registration:** Registering DNS A/AAAA or PTR records for the propagated host IP addresses in any external or internal DNS provider.
- **UI Status Visualization:** Designing or implementing the frontend representation of these fields in the OSAC Web Console (deferred to a subsequent UI-specific epic).
- **Network Interface Reconfiguration:** Allowing tenant users or administrators to modify the physical host's MAC address or assigned inventory IP address via the BareMetalInstance spec.
- **Advanced Multi-NIC Mapping:** Surfacing full network interface maps (all secondary and tertiary interfaces) in the status; only the primary boot MAC and primary IP address are in scope.

## Milestone Scoping

- **Target Milestone:** v0.2
- **Deferred Capabilities:**
  - Automated DNS registration for BareMetalInstance IPs (deferred to subsequent milestone).
  - Visualization of MAC and IP metadata in the OSAC Web Console (deferred to subsequent UI epic).
  - Advanced Multi-NIC mapping and full interface status schemas (deferred to future milestone).
- **Upgrades and Backward Compatibility:**
  - *API Compatibility:* Since this feature introduces new optional status fields for the boot MAC address and primary IP address without modifying existing API response contracts, it is fully backward-compatible. Existing API clients can continue to interact with the BareMetalInstance endpoints without modification.
  - *Existing Resources:* Existing, already-provisioned BareMetalInstances are covered by this feature. Their status section will be automatically populated with the boot MAC address and primary IP address metadata via the automated synchronization mechanism (such as the background reconciliation loop) upon upgrade, without requiring instance reprovisioning or any manual user intervention.
- **E2E Testing Expectations:**
  - *Automated E2E Tests:* The metadata propagation workflow from provisioning completion to API/CLI verification, tenant isolation boundaries, and the negative scenarios (empty IP address/N/A CLI output) must be verified through automated end-to-end integration test suites.
  - *Manual Verification / Integration Testing:* The end-to-end integration flow of CaaS agent correlation using the boot MAC address will be verified through integrated staging environment runs prior to milestone completion.

## User Stories

### Cloud Provider Admin

- As a Cloud Provider Admin, I want host inventory metadata (such as the boot MAC address and primary IP address) to be propagated automatically from the backend inventory to the BareMetalInstance status, so that I do not need to perform manual lookups or maintain offline host spreadsheets.

### Cloud Infrastructure Admin

- As a Cloud Infrastructure Admin, I want to list all BareMetalInstances and their network metadata via the API or CLI, so that I can verify agent-to-instance mapping across the fleet and identify any unmapped hosts to troubleshoot cluster deployment failures.
- As a Cloud Infrastructure Admin, I want the CLI and API to gracefully display `N/A` or empty fields when backend inventory network metadata is missing, so that I can easily identify incomplete inventory records and troubleshoot configuration issues without encountering system errors or failing the provisioning process.
- **Conditional / Fallback Story:** As a Cloud Infrastructure Admin, if the existing platform status refresh capability is found to be missing or insufficient, I want an automated synchronization mechanism to keep the BareMetalInstance network metadata updated without manual user intervention, ensuring metadata remains current.

### Tenant Admin

Not affected by this feature, as Tenant Admins manage tenant-level policies and access controls rather than individual instance-level network metadata.

### Tenant User

- As a Tenant User, I want to see the primary IP address and MAC address of my provisioned BareMetalInstance via the CLI or API, so that I can configure my local SSH clients and network routes to connect to my newly deployed instance.

## Acceptance Criteria

- [ ] **Metadata Availability:** The boot MAC address and primary IP address are populated and available in the status within 60 seconds of the `BareMetalInstance` entering the `Ready` state. *Rationale: Guaranteeing that network metadata is available in a timely manner when the instance is marked Ready ensures downstream automation services can immediately consume it.*
- [ ] **Tenant Isolation Boundaries:** A Tenant User can only retrieve the MAC and IP metadata for `BareMetalInstances` within their authorized tenant namespace; requests to retrieve instances in other namespaces are blocked.
- [ ] **API Payload Integrity:** The `GET /api/fulfillment/v1/baremetal_instances` (List) and `GET /api/fulfillment/v1/baremetal_instances/{id}` (Get) endpoints return the boot MAC address and primary IP address in the status section of the response payload.
- [ ] **CLI Output Formatting:** The OSAC CLI command `osac baremetalinstance describe <name>` displays the host's boot MAC and primary IP in a dedicated network metadata table structure.
- [ ] **Negative Scenarios and Fail-Safe Behavior:** If the backend inventory lacks IP address metadata for an assigned host, the `BareMetalInstance` status successfully exposes the boot MAC address, while the primary IP address field remains empty without causing provisioning errors. In this scenario, the OSAC CLI command `osac baremetalinstance describe <name>` displays `N/A` in the primary IP address field rather than omitting the field, displaying empty columns, or failing.
- [ ] **Agent Correlation Workflow:** A CaaS cluster installation automatically correlates an Assisted Installer agent with the correct BareMetalInstance using the exposed boot MAC address, and the cluster installation proceeds to completion without manual host pairing by an administrator.
- [ ] **Conditional Metadata Synchronization Fallback:** If the platform's existing event-driven status refresh capability is validated as missing or insufficient during initial testing, the automated fallback background synchronization mechanism (reconciliation loop) must be enabled to automatically update the BareMetalInstance status with current backend inventory metadata within 10 minutes of any change, without requiring manual user or administrator intervention.

## Assumptions

- **Inventory Metadata Accuracy:** The physical host inventory backend contains valid, pre-populated, and accurate boot MAC address and IP address metadata for all available physical hosts. [Assumption]
- **Network Connectivity:** The host's primary IP address surfaced in the inventory is reachable over the tenant's VirtualNetwork (rather than only on an infrastructure-level network) once the instance is powered on, enabling self-service Tenant User SSH connections. [Assumption]
- **Existing Synchronization Capability:** The OSAC platform possesses an existing periodic or event-driven synchronization capability that automatically keeps BareMetalInstance status updated with the backend inventory. This capability will be validated early in the cycle, and if it is found to be insufficient, the automated fallback background synchronization mechanism (reconciliation loop) will be used to ensure metadata remains current without requiring manual user or administrator intervention. [Assumption]

## Dependencies

- **Bare Metal Inventory Service API:** The inventory system must provide access to the physical host's boot MAC address and primary IP address metadata to enable propagation to the instance status.
- **CaaS / Assisted Installer Integration:** The cluster installer must be capable of consuming the exposed status metadata of the `BareMetalInstance` to correlate registered installer agents.
- **BareMetalInstance Status Refresh Validation:** Validate that the existing platform capability can keep BareMetalInstance metadata current. Detailed commitments, SLA metrics, fallback timelines, and delivery obligations have been relocated to the owning epic or design document.

## Risks

- **Stale or Incorrect Inventory Metadata:**
  - *Risk:* The physical host inventory backend contains stale or incorrect MAC or IP address metadata, causing CaaS to fail agent correlation or preventing Tenant Users from connecting.
  - *Impact / Mitigation:* This is mitigated by ensuring that the status propagation occurs dynamically, and by displaying `N/A` or empty fields when the backend lacks reliable data rather than failing provisioning. Downstream services must handle metadata mismatches gracefully. Since the automated synchronization model automatically updates the BareMetalInstance status with current backend inventory metadata, any corrected or updated metadata in the backend will automatically propagate to the status without requiring manual user or administrator intervention.
- **Metadata Drift Post-Propagation:**
  - *Risk:* If a physical host's primary IP address is reassigned or modified in the backend inventory after successful propagation, the BareMetalInstance status could become out-of-sync.
  - *Impact / Mitigation:* This is mitigated by the automated synchronization model. The platform automatically detects changes to host network metadata in the backend inventory (via backend events or periodic background reconciliation) and updates the BareMetalInstance status accordingly. No manual user or admin intervention, such as manual re-synchronization, reboot, or reprovisioning, is required to keep the metadata current.
- **Security Exposure of Host Network Metadata:**
  - *Risk:* Exposing physical host MAC and primary IP addresses could potentially allow unauthorized reconnaissance or targeted network scanning if accessed by malicious actors.
  - *Impact / Mitigation:* This is mitigated by enforcing strict tenant namespace isolation boundaries. Only authorized tenant users can view metadata for instances within their respective namespaces. Additionally, the metadata is read-only and does not grant configuration access to the underlying network infrastructure.