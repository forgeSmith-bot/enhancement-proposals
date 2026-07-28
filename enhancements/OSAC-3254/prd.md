# Expose Inventory Metadata in BareMetalInstance Status

| Field | Value |
| :--- | :--- |
| Author(s) | OSAC Product Team |
| Jira | OSAC-2308 / OSAC-3254 |
| Milestone | v0.2 |
| Date | 2026-07-28 |

## Problem Statement

When a BareMetalInstance is provisioned, the bare metal fulfillment service assigns a physical host from the backend inventory. However, none of the host's identifying information (such as the boot MAC address or primary IP address) is surfaced back to the BareMetalInstance status.

This creates a critical disconnect for consumers like Cluster as a Service (CaaS). During cluster installation, the Assisted Installer agent registers from the newly booted physical host using its MAC address, but CaaS has no way of programmatically correlating this agent with the provisioned BareMetalInstance. As a result, CaaS cannot automatically map agents to instances, forcing manual intervention by administrators to correlate hosts, which significantly delays cluster deployment and increases operational overhead. Additionally, Tenant Users cannot self-service connect to their provisioned instances because their network identities (IP and MAC) are completely hidden, leading to additional support requests and a suboptimal user experience.

## In Scope

- **Automatic Metadata Propagation:** The BareMetalInstance status includes the boot MAC address and primary IP address once the instance is provisioned.
- **Status Field Exposure:** Surfacing the physical host's boot MAC address in the BareMetalInstance status.
- **IP Address Exposure:** Surfacing the physical host's primary IP address (when available in the inventory backend) in the BareMetalInstance status.
- **API Availability:** Exposing this metadata via both the Get and List endpoints of the OSAC BareMetalInstance API so that internal services can programmatically consume them.
- **CLI Support:** Displaying the propagated MAC address and IP address in the OSAC CLI when listing or describing a BareMetalInstance.
- **E2E Integration Testing:** Validating automatic metadata propagation and CaaS agent correlation workflows via automated E2E test suites with simulated inventory scenarios.
- **Technical Documentation:** Updating user and API guides to document the new BareMetalInstance status fields, API behaviors, and CLI output details.
- **Tenant Isolation and Security:** Strict namespace boundaries ensuring that a Tenant User can only view or list metadata (boot MAC address and primary IP) for BareMetalInstances belonging to their own tenant.

## Out of Scope

- **Full Hardware Specifications Exposure:** Exposing comprehensive hardware specifications such as CPU cores, RAM size, disk layout, or GPU details in the status field (this is managed via BareMetalInstanceType specs) [Jira: OSAC-2308].
- **Inventory Backend Modifications:** Modifying or extending the schemas, APIs, or database of the existing inventory backend (the operator must consume existing inventory data as-is) [Jira: OSAC-2308].
- **DHCP Configuration and IP AMON:** Provisioning or configuring DHCP reservations or managing active network IP assignments (the metadata is read-only).
- **Assisted Installer Agent Lifecycle:** Managing, deploying, or troubleshooting the Assisted Installer agents running on the bare metal hosts.
- **Automatic DNS Registration:** Registering DNS A/AAAA or PTR records for the propagated host IP addresses in any external or internal DNS provider.
- **UI Status Visualization:** Designing or implementing the frontend representation of these fields in the OSAC Web Console (deferred to a subsequent UI-specific epic).
- **Network Interface Reconfiguration:** Allowing tenant users or operators to modify the physical host's MAC address or assigned inventory IP address via the BareMetalInstance spec.
- **Advanced Multi-NIC Mapping:** Surfacing full network interface maps (all secondary and tertiary interfaces) in the status; only the primary boot MAC and primary IP address are in scope.

## Milestone Scoping

- **Target Milestone:** v0.2
- **Deferred Capabilities:**
  - Automated DNS registration for BareMetalInstance IPs (deferred to subsequent milestone).
  - Visualization of MAC and IP metadata in the OSAC Web Console (deferred to subsequent UI epic).
  - Advanced Multi-NIC mapping and full interface status schemas (deferred to future milestone).

## User Stories

### Cloud Provider Admin

- As a Cloud Provider Admin, I want host inventory metadata (such as the boot MAC address and primary IP address) to be propagated automatically from the backend inventory to the BareMetalInstance status, so that I do not need to perform manual lookups or maintain offline host spreadsheets.

### Cloud Infrastructure Admin

- As a Cloud Infrastructure Admin, I want CaaS to automatically read the boot MAC address of a provisioned BareMetalInstance so that CaaS can correlate the instance with the Assisted Installer agent registering from that host without manual pairing.

### Tenant Admin

Not affected by this feature.

### Tenant User

- As a Tenant User, I want to see the primary IP address and MAC address of my provisioned BareMetalInstance via the CLI or API, so that I can configure my local SSH clients and network routes to connect to my newly deployed instance.

## Acceptance Criteria

- [ ] **Metadata Propagation Latency:** The boot MAC address and primary IP address are populated in the `BareMetalInstance` status within 5 seconds of the instance reaching the `Ready` state.
- [ ] **Tenant Isolation Boundaries:** A Tenant User can only retrieve the MAC and IP metadata for `BareMetalInstances` within their authorized tenant namespace; requests to retrieve instances in other namespaces are blocked.
- [ ] **API Payload Integrity:** The `GET /v1/baremetalinstances` (List) and `GET /v1/baremetalinstances/{id}` (Get) endpoints return the correct `status.bootMacAddress` and `status.primaryIpAddress` fields in the response payload.
- [ ] **CLI Output Formatting:** The OSAC CLI command `osac baremetalinstance describe <name>` displays the host's boot MAC and primary IP in a dedicated network metadata table structure.
- [ ] **Negative Scenarios and Fail-Safe Behavior:** If the backend inventory lacks IP address metadata for an assigned host, the `BareMetalInstance` status successfully exposes the boot MAC address, while the primary IP address field remains empty without causing provisioning errors.

## Assumptions

- **Inventory Metadata Accuracy:** The physical host inventory backend contains valid, pre-populated, and accurate boot MAC address and IP address metadata for all available physical hosts. [Assumption]
- **Network Connectivity:** The host's primary IP address surfaced in the inventory is reachable over the tenant's VirtualNetwork or configured routing pathways once the instance is powered on. [Assumption]

## Dependencies

- **Bare Metal Inventory Service API:** The inventory service must expose the physical host's boot MAC address and IP address via its existing query APIs to enable automatic retrieval and propagation of metadata.
- **CaaS / Assisted Installer Integration:** The CaaS layer must support querying the BareMetalInstance status API to read the MAC address for agent correlation.