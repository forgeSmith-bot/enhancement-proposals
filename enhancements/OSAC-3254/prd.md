# Expose Inventory Metadata in BareMetalInstance Status

| Field | Value |
| :--- | :--- |
| Author(s) | OSAC Product Team |
| Jira | OSAC-3254 / OSAC-2308 |
| Milestone | v0.2 |
| Date | 2026-07-28 |

## Problem Statement

When a BareMetalInstance is provisioned, the bare metal fulfillment service assigns a physical host from the backend inventory. However, none of the host's identifying information (such as the boot MAC address or primary IP address) is surfaced back to the BareMetalInstance status.

This creates a critical disconnect for consumers like Cluster as a Service (CaaS). During cluster installation, the Assisted Installer agent registers from the newly booted physical host using its MAC address, but CaaS has no way of programmatically correlating this agent with the provisioned BareMetalInstance. As a result, CaaS cannot automatically map agents to instances, forcing manual intervention by administrators to correlate hosts, which significantly delays cluster deployment and increases operational overhead. Additionally, Tenant Users cannot self-service connect to their provisioned instances because their network identities (IP and MAC) are completely hidden, leading to additional support requests and a suboptimal user experience.

## In Scope

- **Automatic Metadata Propagation:** The BareMetalInstance status automatically includes the physical host's boot MAC address and primary IP address once the instance is provisioned. The primary IP address is resolved as the runtime DHCP-discovered IP (queried dynamically via the dispatcher's `query_dhcp_lease` role from the fabric manager's DHCP lease API after provisioning), rather than any static, pre-allocated inventory-assigned IP. This aligns with the existing downstream networking contract and ensures that consumers (like CaaS / Assisted Installer) receive actual runtime network details.
- **Status Field Exposure:** Surfacing the physical host's boot MAC address in the BareMetalInstance status immediately upon successful host allocation.
- **IP Address Exposure:** Surfacing the physical host's primary IP address in the BareMetalInstance status. The authoritative source for this IP address is the runtime DHCP lease dynamically queried from the fabric manager's DHCP lease API once the host completes provisioning. It is propagated to the status within 5 seconds of the instance reaching the `READY` state (after the provisioned OS boots and successfully acquires its DHCP lease), preventing consumers from receiving conflicting or stale network identities.
- **API Availability:** Exposing this metadata via both the Get and List endpoints of the OSAC BareMetalInstance API so that internal services can programmatically consume them. Both endpoints must enforce explicit authorization checks for viewing BareMetalInstance MAC/IP metadata. The List endpoint must strictly enforce tenant namespace isolation so that one tenant's network identities (MAC and IP addresses) are never returned to unauthorized callers or other tenants.
- **CLI Support:** Displaying the propagated MAC address and IP address in the OSAC CLI when listing or describing a BareMetalInstance. The CLI list (wide table format) and describe outputs must respect the explicit authorization checks, ensuring that these sensitive fields are only visible to authorized callers, with tenant-level namespace isolation fully preserved.
- **E2E Integration Testing:** Validating automatic metadata propagation and CaaS agent correlation workflows via automated E2E test suites with simulated inventory scenarios.
- **Technical Documentation:** Updating user and API guides to document the new BareMetalInstance status fields, API behaviors, and CLI output details.

### Milestone Scoping

- **Target Milestone:** v0.2
- **Deferred Capabilities:**
  - Designing or implementing the frontend representation of these fields in the OSAC Web Console (deferred to subsequent UI-specific epic).
  - Exposing comprehensive hardware specifications such as CPU cores, RAM size, disk layout, or GPU details in the status (deferred to future hardware inventory roadmap).

## Out of Scope

- **Full Hardware Specifications Exposure:** Exposing comprehensive hardware specifications such as CPU cores, RAM size, disk layout, or GPU details in the status field (this is managed via BareMetalInstanceType specs) [Jira: OSAC-2308].
- **Inventory Backend Modifications:** Modifying or extending the schemas, APIs, or database of the existing inventory backend (the operator must consume existing inventory data as-is) [Jira: OSAC-2308].
- **DHCP Configuration and IP AMON:** Provisioning or configuring DHCP reservations or managing active network IP assignments (the metadata is read-only).
- **Assisted Installer Agent Lifecycle:** Managing, deploying, or troubleshooting the Assisted Installer agents running on the bare metal hosts.
- **Automatic DNS Registration:** Registering DNS A/AAAA or PTR records for the propagated host IP addresses in any external or internal DNS provider.
- **UI Status Visualization:** Designing or implementing the frontend representation of these fields in the OSAC Web Console (deferred to a subsequent UI-specific epic).
- **Network Interface Reconfiguration:** Allowing tenant users or operators to modify the physical host's MAC address or assigned inventory IP address via the BareMetalInstance spec.
- **Advanced Multi-NIC Mapping:** Surfacing full network interface maps (all secondary and tertiary interfaces) in the status; only the primary boot MAC and primary IP address are in scope.

## User Stories

### Cloud Provider Admin

- As a Cloud Provider Admin, I want host inventory metadata (such as the boot MAC address and primary IP address) to be propagated automatically from the backend inventory to the BareMetalInstance status, so that I do not need to perform manual lookups or maintain offline host spreadsheets.

### Cloud Infrastructure Admin

- As a Cloud Infrastructure Admin, I want CaaS to automatically read the boot MAC address of a provisioned BareMetalInstance so that CaaS can correlate the instance with the Assisted Installer agent registering from that host without manual pairing.

### Tenant Admin

Not affected by this feature.

### Tenant User

- As a Tenant User, I want to see the primary IP address and MAC address of my own provisioned BareMetalInstance via the CLI or API (with strict namespace isolation preventing access to other tenants' instances), so that I can configure my local SSH clients and network routes to connect to my newly deployed instance.

## Acceptance Criteria

- [ ] When a BareMetalInstance is successfully provisioned, its status contains the correct host boot MAC address and primary IP address.
- [ ] The boot MAC address and primary IP address metadata appear in the BareMetalInstance status within 5 seconds of the instance reaching the `READY` state.
- [ ] Get, List, and CLI list/describe operations require explicit authorization checks and expose the boot MAC address and primary IP address only to authorized callers.
- [ ] The List API endpoint and CLI list output enforce strict tenant isolation, ensuring that one tenant's network identities (MAC and IP addresses) are never returned to another tenant.
- [ ] A Tenant User can retrieve the boot MAC address and primary IP address of their own BareMetalInstances via the Get and List API endpoints.
- [ ] A Tenant User cannot retrieve or view the status metadata of BareMetalInstances belonging to other tenants (strict tenant namespace isolation).
- [ ] The OSAC CLI displays the boot MAC address and primary IP address in both list (wide table format) and describe outputs for BareMetalInstances only to authorized callers.
- [ ] Automated E2E test suites validate the agent correlation workflow by verifying that a simulated Assisted Installer agent can successfully pair with a BareMetalInstance using the exposed boot MAC address.

## Assumptions

- **Inventory Metadata Accuracy:** The physical host inventory backend contains valid, pre-populated, and accurate boot MAC address metadata for all available physical hosts. [Assumption]
- **DHCP Leases API Reliability:** The fabric's DHCP service or DHCP lease API is operational and returns correct, timely leases matching the host's boot MAC address. [Assumption]
- **Network Connectivity:** The host's runtime DHCP-discovered primary IP address is reachable over the tenant's VirtualNetwork or configured routing pathways once the instance is powered on. [Assumption]

## Dependencies

- **Bare Metal Inventory Service API:** The inventory service must expose the physical host's boot MAC address via its existing query APIs so the fulfillment layer can retrieve them.
- **CaaS / Assisted Installer Integration:** The CaaS layer must support querying the BareMetalInstance status API to read the MAC address for agent correlation.