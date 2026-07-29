# Expose Inventory Metadata in BareMetalInstance Status

| Field | Value |
| :--- | :--- |
| Author(s) | OSAC Product Team |
| Jira | OSAC-2308 (Parent Epic) / OSAC-3254 (Child Status Tracking Issue) |
| Milestone | v0.2 |
| Date | 2026-07-29 |

## Terminology

- **BMaaS (Bare Metal as a Service):** An OSAC core service responsible for provisioning and managing the lifecycle of physical bare metal host instances.
- **CaaS (Cluster as a Service):** An OSAC core service that provisions and manages Kubernetes clusters using Hosted Control Planes (HCP).
- **Backend Inventory:** The system of record containing physical host hardware specifications, network interfaces, and assigned MAC addresses.
- **Assisted Installer:** An installer agent that registers from a newly booted physical host during cluster deployment to facilitate automated Kubernetes installation.

## Problem Statement

When a BareMetalInstance is provisioned, the Bare Metal as a Service (BMaaS) fulfillment service assigns a physical host from the backend inventory. However, none of the host's identifying information (such as the boot MAC address) is surfaced back to the BareMetalInstance status.

This creates a critical disconnect for consumers like Cluster as a Service (CaaS). During cluster installation, the Assisted Installer agent registers from the newly booted physical host using its MAC address, but CaaS has no way of programmatically correlating this agent with the provisioned BareMetalInstance. As a result, CaaS cannot automatically map agents to instances, forcing manual intervention by administrators to correlate hosts, which significantly delays cluster deployment and increases operational overhead. Additionally, Tenant Users cannot view assigned physical interfaces or correlate host hardware information, leading to additional support requests and a suboptimal user experience.

## In Scope

- **Network Interface Metadata Exposure:** Surfacing the physical host's boot MAC address and MAC addresses of other physical network interfaces (when available in the BMaaS inventory backend) in the BareMetalInstance status once the instance is provisioned, enabling hardware correlation and downstream usage.
- **API Availability:** Exposing this metadata via both the Get and List endpoints of the OSAC BareMetalInstance API so that downstream services and automation can programmatically consume them.
- **CLI Support:** Displaying the propagated boot MAC address and physical interface MACs in the OSAC CLI when listing or describing a BareMetalInstance.
- **Seamless Platform Integration:** Facilitating subsequent downstream consumer correlation by exposing the reliable boot MAC address in the status. The actual CaaS workflow integration and end-to-end installer agent correlation are deferred to the owning EP/design.
- **Tenant Isolation and Security:** Enforcing BMaaS tenant authorization as the authoritative boundary where Tenant Users are restricted to viewing or listing physical network interface MAC address metadata only for BareMetalInstances belonging to their own tenant, independently of Kubernetes namespaces. The designated Cloud Provider Admin and Cloud Infrastructure Admin roles are explicitly granted elevated authorization to view or list this metadata across all tenants for fleet-wide visibility and administration.

## Out of Scope

- **Full Hardware Specifications Exposure:** Exposing comprehensive hardware specifications such as CPU cores, RAM size, disk layout, or GPU details in the status field (this is managed via BareMetalInstanceType specs) [Jira: OSAC-2308].
- **Inventory Backend Modifications:** Modifying or extending the schemas, APIs, or database of the existing inventory backend is out of scope; the feature consumes existing inventory data as-is [Jira: OSAC-2308].
- **IP Address Tracking and IPAM (IP Address Management):** Tracking, configuring, or exposing IP addresses, DHCP configurations, active network IP assignments, or IP address mappings; this is completely out of scope and left to the details of the networking design/EP.
- **Assisted Installer Agent Lifecycle:** Managing, deploying, or troubleshooting the Assisted Installer agents running on the bare metal hosts.
- **Automatic DNS Registration:** Registering DNS A/AAAA or PTR records for the bare metal hosts in any external or internal DNS provider.
- **UI Status Visualization:** Designing or implementing the frontend representation of these fields in the OSAC Web Console (deferred to a subsequent UI-specific epic).
- **Network Interface Reconfiguration:** Allowing tenant users or administrators to modify the physical host's MAC addresses via the BareMetalInstance spec.
- **Advanced Multi-NIC Mapping:** Surfacing full complex network interface maps (all secondary and tertiary interfaces mapping to logical networks) in the status; only the physical interface MAC addresses (boot MAC and available physical MACs) are in scope.
- **CaaS Integration and End-to-End Installation Completion:** The actual end-to-end orchestration of CaaS agent-to-instance correlation, integration testing of cluster installations, and network reachability verification of provisioned physical interfaces are out of scope and deferred to the owning EP/design.

## Milestone Scoping

- **Target Milestone:** v0.2
- **Deferred Capabilities:**
  - IP address tracking and logical IP status mapping (deferred to networking EP/design).
  - Visualization of MAC metadata in the OSAC Web Console (deferred to subsequent UI epic).
  - Exposing full detailed physical interface mapping schemas beyond MAC addresses (deferred to future milestone).
- **Upgrades and Backward Compatibility:**
  - *API Compatibility:* Since this feature introduces new optional status fields for the boot MAC address, other physical interface MAC addresses, and last successful synchronization timestamp without modifying existing API response contracts, it is fully backward-compatible. Existing API clients can continue to interact with the BareMetalInstance endpoints without modification.
  - *Existing Resources:* Existing BareMetalInstances will have their status automatically populated with boot MAC and physical interface MAC metadata, and the last successful synchronization timestamp upon upgrade, without requiring reprovisioning or manual intervention.
- **E2E Testing Expectations:**
  - *Automated E2E Tests:* The metadata propagation workflow from provisioning completion to API/CLI verification, tenant isolation boundaries, and the negative scenarios (empty interface lists / N/A CLI output) must be verified through automated end-to-end integration test suites.
  - *Manual Verification / Integration Testing:* The propagation of boot MAC and physical interface MAC addresses from the backend inventory to the BareMetalInstance status must be verified in a staging environment. The actual end-to-end CaaS integration and network reachability validation are deferred to the owning EP/design.

## User Stories

### Cloud Provider Admin

- As a Cloud Provider Admin, I want host inventory metadata (such as the boot MAC address and physical interface MAC addresses) to be propagated automatically from the backend inventory to the BareMetalInstance status, so that I do not need to perform manual lookups or maintain offline host spreadsheets.
- As a Cloud Provider Admin, I want to see which BareMetalInstances are missing physical interface MAC metadata across all tenants, so that I have fleet-wide visibility into metadata completeness and can proactively resolve backend inventory issues.

### Cloud Infrastructure Admin

- As a Cloud Infrastructure Admin, I want to list all BareMetalInstances and their physical interface MAC metadata via the API or CLI, so that I can verify agent-to-instance mapping across the fleet and identify any unmapped hosts to troubleshoot cluster deployment failures.
- As a Cloud Infrastructure Admin, I want the CLI and API to gracefully display `N/A` or empty fields when backend inventory MAC metadata is missing, so that I can easily identify incomplete inventory records and troubleshoot configuration issues without encountering system errors or failing the provisioning process.
- As a Cloud Infrastructure Admin, I want the BareMetalInstance MAC metadata to remain current automatically when changes occur in the backend inventory, so that I can trust the displayed values without performing manual refreshes.

### Tenant Admin

- As a Tenant Admin, I want to view tenant-scoped aggregate visibility of all BareMetalInstances within my authorized tenant, including identifying which instances are missing interface MAC metadata, so that I can monitor provisioning completeness and assist Tenant Users with hardware correlation troubleshooting.

### Tenant User

- As a Tenant User, I want to see the boot MAC address and available physical interface MAC addresses of my provisioned BareMetalInstance via the CLI or API, so that I can identify and correlate physical host interfaces assigned to my instance.
- As a Tenant User, I want the CLI and API to gracefully display `N/A` or empty fields when my instance's MAC address is unavailable in the backend inventory, so that I am clearly informed of the missing network identity without encountering client formatting errors or experiencing system crashes.

## Acceptance Criteria

- [ ] **Metadata Availability:** The boot MAC address and physical interface MAC addresses are populated and available in the status within 60 seconds of the `BareMetalInstance` entering the `Ready` state. *Rationale: Guaranteeing that MAC address metadata is available in a timely manner when the instance is marked Ready ensures downstream automation services can immediately consume it.*
  - *SLA Fallback Behavior:* If MAC metadata is not available within the 60-second window, downstream consumers continue operating and tolerate delayed metadata without failing provisioning, while administrators are alerted via an observable warning event. Instance provisioning remains in the `Ready` state and is not failed or restarted.
- [ ] **Tenant Isolation Boundaries:** BMaaS tenant authorization serves as the authoritative boundary for Get/List MAC metadata access, independently of Kubernetes namespaces. A Tenant User is restricted to retrieving and listing MAC address metadata only for BareMetalInstances belonging to their own tenant; requests to retrieve instances belonging to other tenants are blocked. Cloud Provider Admins and Cloud Infrastructure Admins are authorized to view and list metadata across all tenants.
- [ ] **API Payload Integrity:** The API Get and List endpoints for BareMetalInstances return the boot MAC address, physical interface MAC addresses, and last successful synchronization timestamp in the status section of the response payload.
- [ ] **CLI Output Formatting:** The OSAC CLI command `osac baremetalinstance describe <name>` displays the host's boot MAC and physical interface MACs in a dedicated physical interfaces metadata table structure. Additionally, the list command `osac baremetalinstance list` (and the cross-tenant list for authorized administrators) includes dedicated columns for both the boot MAC address and physical-interface MAC metadata in its tabular output, maintaining clean formatting.
- [ ] **Negative Scenarios and Fail-Safe Behavior:**
  - If the backend inventory lacks secondary MAC address metadata for an assigned host, the `BareMetalInstance` status successfully exposes the boot MAC address, while other interface fields remain empty without causing provisioning errors.
  - If the boot MAC address or the entire inventory record itself is unavailable, the missing values are exposed as `N/A` or empty status fields with an appropriate warning or synchronization state (such as a status condition or warning event indicating metadata unavailability).
  - In any missing metadata scenario, the OSAC CLI commands `osac baremetalinstance describe <name>` and `osac baremetalinstance list` must continue to display `N/A` in missing fields or columns rather than omitting the field, displaying empty columns, or failing.
  - The instance's overall provisioning status must explicitly be preserved as `Ready` without encountering errors, failing, or triggering reprovisions.
- [ ] **Agent Correlation Workflow:** The scope of this PRD is strictly confined to exposing reliable boot MAC metadata in the status of the `BareMetalInstance`. Downstream integration, such as the direct cluster installer mapping and CaaS workflow completion, is deferred to the owning EP/design. The status must expose the boot MAC address to make it programmatically queryable by downstream systems once available.
- [ ] **Metadata Synchronization:** The BareMetalInstance status metadata must automatically reflect current backend inventory data within 10 minutes of any change, without manual user or administrator intervention.
  - *SLA Fallback Behavior:* If synchronization exceeds the 10-minute SLA, downstream systems continue operating with the last-known cached metadata. The platform will alert administrators when the synchronization SLA is exceeded, ensuring active workloads and the instance status are not interrupted.
- [ ] **Synchronization Observability:** The `BareMetalInstance` status exposes an observable timestamp indicating the last successful metadata synchronization time, allowing users and administrators to verify that backend inventory synchronization is actively functioning.

## Assumptions

- **Inventory Metadata Accuracy:** The boot MAC address and physical interface MAC metadata from the physical host inventory backend is expected to be available and accurate, but may be missing or invalid for some physical hosts. This assumption supports the documented fallback and negative acceptance scenarios without guaranteeing metadata for every host.

## Dependencies

- **Bare Metal Inventory Service API:** The inventory system must provide access to the physical host's boot MAC address and physical interface MAC metadata to enable propagation to the instance status. This is a critical prerequisite to ensure reliable boot MAC metadata exposure.
- **Platform Core Synchronization Infrastructure:** The core platform synchronization framework is a prerequisite dependency required to trigger updates and propagate backend inventory changes dynamically to the BareMetalInstance status. Any required enhancements to this core infrastructure are tracked as a prerequisite dependency under [Jira: OSAC-2308].
- **BareMetalInstance Status Refresh:** The platform capability to automatically keep BareMetalInstance metadata synchronized with backend inventory updates. Detailed commitments, SLA metrics, and delivery obligations have been relocated to the owning epic or design document.
- **Deferred Network and Workflow Integration (Decoupled):** In accordance with scoping this feature strictly to metadata exposure, actual network reachability, tenant VirtualNetwork mapping, CaaS cluster installer integration, and end-to-end installation workflows are completely decoupled from this PRD, require no prerequisite integrations, and are deferred entirely to the owning EP/design.

## Risks

- **Stale or Incorrect Inventory Metadata:**
  - *Risk:* The physical host inventory backend contains stale or incorrect MAC metadata, causing CaaS to fail agent correlation.
  - *Impact / Mitigation:* This is mitigated by ensuring that the status propagation occurs dynamically, and by displaying `N/A` or empty fields when the backend lacks reliable data rather than failing provisioning. Downstream services must handle metadata mismatches gracefully. Since BareMetalInstance status metadata remains current automatically, any corrected or updated metadata in the backend will automatically propagate to the status without requiring manual user or administrator intervention.
- **Metadata Drift Post-Propagation:**
  - *Risk:* If a physical host's MAC addresses are reassigned or modified in the backend inventory after successful propagation, the BareMetalInstance status could become out-of-sync.
  - *Impact / Mitigation:* This is mitigated by ensuring the BareMetalInstance status metadata is kept up-to-date automatically. The platform automatically updates the BareMetalInstance status when host network interface metadata changes in the backend inventory. No manual user or admin intervention, such as manual re-synchronization, reboot, or reprovisioning, is required to keep the metadata current.
- **Security Exposure of Host Network Metadata:**
  - *Risk:* Exposing physical host MAC addresses could potentially allow unauthorized reconnaissance or targeted network spoofing if accessed by malicious actors.
  - *Impact / Mitigation:* This is mitigated by enforcing strict BMaaS tenant authorization boundaries independently of Kubernetes namespaces. Only authorized tenant users can view metadata for instances belonging to their respective tenants. Additionally, the metadata is read-only and does not grant configuration access to the underlying network infrastructure.
- **Missed Metadata SLA / Propagation Delay:**
  - *Risk:* Network metadata initial propagation (60 seconds) or synchronization (10 minutes) exceeds the targeted SLA due to backend inventory service delays or network latency.
  - *Impact / Mitigation:* This could temporarily delay downstream correlation. Mitigated by designing downstream consumers to gracefully poll and tolerate delayed metadata without failing immediately. Additionally, the BareMetalInstance provisioning state is decoupled from metadata retrieval, preventing transient inventory delays from marking otherwise healthy nodes as failed.