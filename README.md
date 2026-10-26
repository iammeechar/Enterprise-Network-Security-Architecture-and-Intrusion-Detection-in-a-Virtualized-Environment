# Enterprise Network Security Architecture and Intrusion Detection in a Virtualized Environment

## Project Overview

This project focuses on designing, implementing, verifying, securing, attacking, monitoring, and troubleshooting a simulated enterprise network using **EVE-NG**, virtual network devices, Linux-based security tools, and controlled testing environments.

The project is being developed incrementally through structured implementation phases. Each phase establishes a verified baseline before the next phase introduces additional functionality.

The initial phases establish the network infrastructure, routing architecture, legacy network integration, Internet-edge routing, and perimeter firewall. The remaining security work will build upon this foundation by introducing protected services, intrusion detection, vulnerability assessment, security monitoring, and controlled attack-and-response exercises.

**Current status:** Phases 0–5 completed. The routing and addressing foundation is substantially complete. Advanced security implementation remains future work.

---

## Objectives

The main objectives are to:

- Design and implement a realistic enterprise campus network.
- Establish redundant Layer 2 and Layer 3 infrastructure.
- Implement VLANs, trunking, link aggregation, spanning tree, and first-hop redundancy.
- Configure OSPF and integrate a legacy EIGRP routing domain through controlled redistribution.
- Implement dual-ISP eBGP connectivity, routing policy, and default-route failover.
- Integrate a perimeter firewall using explicit routing boundaries, security zones, firewall policies, and source NAT.
- Verify normal operation through routing, connectivity, and policy tests.
- Introduce controlled failures and investigate their root causes.
- Integrate existing Python-based packet analysis and intrusion detection tooling.
- Develop a security testing environment for vulnerability assessment, attack simulation, detection, investigation, and validation.
- Maintain reproducible configurations, test evidence, troubleshooting records, and version-controlled documentation.

## Project Scope

The project is divided into two broad areas.

### Completed infrastructure foundation

- Layer 2 switching and redundancy.
- Inter-VLAN routing and HSRP.
- OSPF routing.
- EIGRP/OSPF redistribution.
- Dual-ISP eBGP.
- Routing policy and default-route failover.
- VyOS perimeter firewall integration.
- Static routing across the firewall boundary.
- Firewall zones, stateful policies, and source NAT.
- Verification and failure testing.

### Remaining security implementation

- Protected server and DMZ services.
- Service hardening and vulnerability assessment.
- Integration of the existing Python Network Packet Analyzer.
- Reliable packet capture and traffic observation.
- Intrusion detection and alert validation.
- SIEM/log aggregation and security investigation.
- Controlled attack simulation and defensive validation.
- Incident investigation, remediation, and retesting.

The remaining security components are planned stages, not claims of completed implementation.

---

## Architecture

The network follows a hierarchical enterprise design with a redundant distribution layer, multiple access switches, a legacy routing domain, a dual-edge Internet simulation, and a VyOS perimeter firewall.

### Logical topology

```text
                         ISP-1          ISP-2
                           | \          / |
                           |  \        /  |
                           |   EDGE-1--   |
                           |      |       |
                           |   EDGE-2     |
                           |      |       |
                           +------+-------+
                                  |
                         VyOS Perimeter Firewall
                           /              \
                          /                \
                     DIST-1 ------------- DIST-2
                       |   \             /   |
                       |    \           /    +---- LEGACY-1 ---- LEGACY-2
                       |     \         /          OSPF/EIGRP
                       |      \       /
                     ACC-1   ACC-2   ACC-3   ACC-4
                       |       |       |       |
                     VPC7    VPC8    VPC9    VPC10
```

*Logical representation only; actual interface connections and routing paths are documented in the phase-specific topology and configuration files.*

### Design principles

- **Redundancy:** Dual distribution switches, redundant access uplinks, and dual enterprise edge routers.
- **Segmentation:** Separate VLANs for users, voice, servers, security, DMZ, guest, and management.
- **Controlled routing:** OSPF for the enterprise core, EIGRP for the legacy domain, and eBGP at the simulated Internet edge.
- **Policy enforcement:** Explicit routing and a deliberate firewall boundary at VyOS.
- **Resilience:** Test link failures, routing reconvergence, default-route failover, and recovery.
- **Reproducibility:** Maintain frozen phase baselines and version-controlled documentation.
- **Resource awareness:** Use a virtualized topology that fits the available hardware and expand only when a test requires it.

---

## Lab Environment

### Primary platform

| Component | Environment |
|---|---|
| Main host | Lenovo ThinkPad T14s Gen 1 |
| Host operating system | Windows 11 |
| Main hypervisor | VMware |
| Network emulation | EVE-NG 6.2.0-4 |
| Additional virtualization | Oracle VirtualBox 7.2.4 |
| Existing application-security VM | Debian-lab2 |
| Security testing workstation | HP ProBook G450 running Kali Linux through dual boot |

The EVE-NG environment has been observed to consume approximately 12 GB of RAM at baseline, before heavy traffic generation. Additional devices and services must therefore be selected with resource consumption in mind.

### Network device images

The lab has used or has access to images including:

- Cisco CSR1000v, including `csr1000vng-universalk9.17.03.08a-serial`.
- Cisco IOL Layer 2 IOS 17.12.1.
- VyOS 1.5 rolling.
- FortiGate 7.2.4.
- ArubaCX and Cisco NX-OSv images for separate experimentation.

Not every available image is part of the implemented capstone. Additional platforms may be used in separate architecture or advanced technology labs.

### Existing security resources

- Python Network Packet Analyzer using Scapy.
- Existing DVWA/Burp application-security environment on Debian-lab2.
- GitHub repositories for networking labs and Python security tooling.
- Kali Linux for controlled testing and packet analysis.

The security tools and application environment will be audited before integration into the completed enterprise topology.

---

## Network Addressing and Segmentation

### VLAN allocation

| VLAN | Name | Purpose |
|---:|---|---|
| 10 | USERS | Enterprise user endpoints |
| 20 | VOICE | Voice network |
| 30 | SERVERS | Enterprise server network |
| 40 | SECURITY | Security-related network |
| 50 | DMZ | Intended demilitarized-zone network |
| 60 | GUEST | Guest access network |
| 99 | MANAGEMENT | Network device management |
| 100 | DIST-TRANSIT | Distribution interconnection and Layer 3 transit |

The presence of a VLAN named DMZ does not, by itself, establish firewall-enforced isolation. The effective traffic path and enforcement point must be verified for each protected service.

### Management addressing

| Device | Management IP |
|---|---|
| DIST-1 | `10.10.99.11` |
| DIST-2 | `10.10.99.12` |
| ACC-1 | `10.10.99.13` |
| ACC-2 | `10.10.99.14` |
| ACC-3 | `10.10.99.15` |
| ACC-4 | `10.10.99.16` |

### Phase 2 transit addressing

| Connection | Network | Address allocation |
|---|---|---|
| DIST-1–DIST-2 transit | `10.255.100.0/30` | DIST-1 `.1`, DIST-2 `.2` |
| DIST-1–EDGE-1 | `10.255.101.0/30` | DIST-1 `.1`, EDGE-1 `.2` |
| DIST-2–EDGE-2 | `10.255.102.0/30` | DIST-2 `.1`, EDGE-2 `.2` |
| EDGE-1–EDGE-2 | `10.255.103.0/30` | EDGE-1 `.1`, EDGE-2 `.2` |
| DIST-2–LEGACY-1 | `10.255.104.0/30` | DIST-2 `.1`, LEGACY-1 `.2` |
| LEGACY-1–LEGACY-2 | `10.255.105.0/30` | LEGACY-1 `.1`, LEGACY-2 `.2` |

These addresses document the Phase 2/3 routing foundation. Phase 5 introduced VyOS as an explicit routing boundary and established the corresponding firewall-facing links described below.

### Phase 3 legacy loopbacks

| Prefix | Purpose |
|---|---|
| `172.16.10.0/24` | Legacy user network |
| `172.16.20.0/24` | Legacy server network |
| `172.16.30.0/24` | Route-filtering test prefix |

The `172.16.30.0/24` prefix is used to verify that redistribution policy prevents unauthorized route propagation.

### Phase 4 Internet-edge test prefixes

| Prefix | Purpose |
|---|---|
| `192.0.2.0/24` | Enterprise synthetic test prefix originated by EDGE-1 |
| `198.51.100.0/24` | ISP-1 synthetic test prefix |
| `203.0.113.0/24` | ISP-2 synthetic test prefix |

These are documentation/test address ranges. The enterprise prefix is a routing test prefix, not an automatically implemented or protected DMZ service.

---

# Implementation Phases

## Phase 0 — Architecture Baseline

**Status:** Complete and frozen.

Phase 0 established the authoritative design baseline for the capstone.

### Scope

- Enterprise VLAN and IP addressing plan.
- Hierarchical switching architecture.
- OSPF enterprise routing.
- Legacy EIGRP integration through controlled redistribution.
- Dual-ISP BGP edge.
- Perimeter firewall and NAT.
- Security, DMZ, IDS and SIEM objectives.
- Failure testing and documentation requirements.

The baseline is maintained as `Phase-0-Baseline-v1`.

MPLS/VRF/MP-BGP and VXLAN/EVPN were considered in the broader learning roadmap, but they are not implemented in the frozen Phases 0–5. They are reserved for separate advanced labs.

## Phase 1 — Layer 2 Switching

**Status:** Complete and frozen.

### Devices

- DIST-1 and DIST-2.
- ACC-1, ACC-2, ACC-3 and ACC-4.
- VPC7, VPC8, VPC9 and VPC10.

### Implemented functionality

- VLAN creation and assignment.
- 802.1Q trunking.
- Redundant access-to-distribution links.
- LACP EtherChannel between DIST-1 and DIST-2.
- Rapid PVST+.
- Primary and secondary spanning-tree root placement.
- Switch management addressing.
- End-device VLAN connectivity.

### Key design decisions

- DIST-1 is the primary spanning-tree root.
- DIST-2 is the secondary root.
- Distribution interconnection uses LACP Port-channel 1.
- The trunk carries VLANs 10, 20, 30, 40, 50, 60, 99 and 100.

### Failure tests completed

1. Access uplink failure.
2. VLAN removal from a trunk.
3. Incorrect endpoint VLAN assignment.
4. LACP member failure.
5. Spanning-tree root change.

The tests validated expected behavior, recovery, and the impact of Layer 2 failures. Configurations were saved and the phase was frozen.

## Phase 2 — Layer 3 Switching, HSRP and OSPF

**Status:** Complete and frozen.

### Implemented functionality

- Routed interfaces between distribution and edge devices.
- VLAN SVIs.
- HSRP default-gateway redundancy.
- OSPF process 100, Area 0.
- Passive production SVIs.
- Routed transit links and OSPF adjacencies.
- Connectivity and route-convergence testing.

### HSRP design

The virtual gateway uses `.1`, with DIST-1 using `.2` and DIST-2 using `.3` for the relevant SVI addresses.

| Active distribution switch | VLANs |
|---|---|
| DIST-1 | 10, 30, 50 |
| DIST-2 | 20, 40, 60 |

DIST-1 uses a higher HSRP priority for VLANs 10, 30 and 50; DIST-2 uses a higher priority for VLANs 20, 40 and 60. Preemption is configured.

### Troubleshooting highlight

OSPF initially failed to operate on DIST-1's Ethernet1/2 because the interface remained in Layer 2 switchport mode.

The root cause was identified from the OSPF error indicating that OSPF was not enabled on the interface. The interface was corrected using `no switchport`, after which the routed adjacency and connectivity could be validated.

Failure testing included shutting a routed link, verifying the resulting behavior, and restoring the link.

## Phase 3 — EIGRP and OSPF Redistribution

**Status:** Complete and frozen.

### Objective

Integrate the legacy EIGRP domain into the enterprise OSPF network without uncontrolled route propagation or unnecessary routing-loop risk.

### Implemented functionality

- EIGRP autonomous system 100.
- OSPF process 100, Area 0.
- Redistribution between EIGRP and OSPF.
- Route filtering using prefix-based policy.
- Route tagging.
- EIGRP seed metric configuration.
- Route withdrawal and recovery testing.

### Legacy topology

LEGACY-1 connects to DIST-2 and LEGACY-2. LEGACY-1 participates in both OSPF and EIGRP, while LEGACY-2 provides legacy test prefixes.

### Redistribution policy

| Direction | Permitted prefixes | Filtered prefixes |
|---|---|---|
| EIGRP → OSPF | `172.16.10.0/24`, `172.16.20.0/24` | `172.16.30.0/24` |
| OSPF → EIGRP | `10.10.10.0/24`, `10.10.30.0/24` | `10.10.20.0/24`, `10.10.40.0/24` |

Routes redistributed from EIGRP into OSPF use tag 100. The EIGRP seed metric is:

`10000 100 255 1 1500`

### Verification and failure testing

- OSPF and EIGRP adjacency checks.
- Route-table inspection.
- Source-based reachability testing.
- Verification of permitted and filtered routes.
- Legacy link shutdown and recovery.
- Route withdrawal and re-advertisement.

The tests demonstrated controlled route exchange and the expected filtering behavior.

## Phase 4 — Dual-ISP eBGP and Internet-Edge Routing

**Status:** Complete and frozen.

### Autonomous systems

| Network | AS number |
|---|---:|
| Enterprise | 65010 |
| ISP-1 | 65001 |
| ISP-2 | 65002 |

### Implemented functionality

- Four eBGP sessions across the dual-edge, dual-ISP topology.
- Enterprise prefix advertisement.
- ISP test-prefix advertisement.
- Prefix filtering.
- Default-route advertisement and propagation.
- Local preference for primary and backup ISP selection.
- Default-route withdrawal and recovery testing.
- Integration with the existing OSPF routing design.

### Interconnection networks

| Connection | Network |
|---|---|
| EDGE-1–ISP-1 | `10.255.110.0/30` |
| EDGE-2–ISP-1 | `10.255.111.0/30` |
| EDGE-1–ISP-2 | `10.255.112.0/30` |
| EDGE-2–ISP-2 | `10.255.113.0/30` |

EDGE-1 also uses loopback address `192.0.2.1` for the enterprise synthetic test prefix.

### Routing policy

- ISP-1 is preferred using local preference 200.
- ISP-2 is the lower-preference backup using local preference 100.
- The enterprise advertises only the intended enterprise test prefix.
- Each ISP advertises its own test prefix and a synthetic default route.
- ISP prefix filters restrict accepted enterprise advertisements.
- EDGE-1 conditionally originates the OSPF default using `default-information originate`, without `always`.
- BGP routes are not indiscriminately redistributed into OSPF.

### Important observed limitation

When the EDGE-1-to-ISP-1 path failed, the ISP-1-specific prefix `198.51.100.0/24` became unreachable because ISP-2 did not advertise that prefix.

This illustrates an important distinction: **default-route failover does not guarantee reachability to every destination previously reachable through the primary ISP.**

The BGP sessions, route advertisements, prefix filters, default failover and recovery were verified.

## Phase 5 — VyOS Perimeter Firewall Integration

**Status:** Complete. Configuration, verification, failure testing and documentation are complete.

### Design decision

The selected approach is **Option B: Explicit Routing Boundary using VyOS**.

VyOS participates in neither the enterprise OSPF process nor the legacy EIGRP domain. Static routes provide explicit control over the firewall's routing decisions.

This approach was selected to:

- Keep the firewall boundary auditable.
- Make route ownership and packet paths explicit.
- Avoid unnecessary modifications to the completed routing design.
- Allow deliberate primary and backup path selection.
- Provide a foundation for subsequent security policy testing.

Extending OSPF Area 0 through VyOS was deferred to a separate design/research lab.

### VyOS interface addressing

| Interface | Connection | Address |
|---|---|---|
| eth0 | DIST-1 | `10.255.101.2/30` |
| eth1 | DIST-2 | `10.255.102.2/30` |
| eth2 | EDGE-1 | `10.255.105.1/30` |
| eth3 | EDGE-2 | `10.255.106.1/30` |

EDGE-1 uses `10.255.105.2` on the VyOS-facing link, and EDGE-2 uses `10.255.106.2`.

These firewall-facing networks supersede the direct distribution-to-edge path for the Phase 5 integrated topology. The phase-specific configuration files are authoritative for the final implementation.

### Static routing

VyOS has static routes for the enterprise VLAN prefixes and the permitted legacy prefixes:

- `172.16.10.0/24`
- `172.16.20.0/24`

The route design prefers DIST-1 for VLANs 10, 30, 50 and 99, and DIST-2 for VLANs 20, 40 and 60. The legacy prefixes prefer DIST-2. Primary routes use administrative distance 1, with backups at administrative distance 10.

### Default-route failover

- Primary default route: EDGE-1 via `10.255.105.2`, eth2, metric 10.
- Backup default route: EDGE-2 via `10.255.106.2`, eth3, metric 20.
- Health-check targets: `198.51.100.1` for the primary and `203.0.113.1` for the backup.

A controlled health-check failure was introduced by changing the primary target to `198.51.100.254`. The backup became the preferred route, and reachability to the ISP-2 test destination was verified. The original health-check target was restored, and EDGE-1 again became the preferred route.

### Firewall zones and policies

Three zones were established:

- **LAN:** eth0 and eth1.
- **LOCAL:** traffic destined for VyOS itself.
- **WAN:** eth2 and eth3.

The configuration uses named policies:

- `PH5-LAN-WAN`
- `PH5-WAN-LAN`
- `PH5-LAN-LOCAL`
- `PH5-LOCAL-LAN`
- `PH5-LOCAL-WAN`
- `PH5-WAN-LOCAL`

The policies implement default-drop behavior, established/related connection handling, invalid-state drops and selected ICMP/SSH management exceptions.

LAN-to-WAN ICMP is permitted to the two synthetic ISP test prefixes. LAN-to-LOCAL management permits ICMP and TCP/22 from `10.10.99.0/24`.

### Source NAT

Masquerade rules were configured for enterprise source addresses in `10.10.0.0/16`:

| Rule | Source | Outgoing interface |
|---:|---|---|
| 100 | `10.10.0.0/16` | eth2 |
| 110 | `10.10.0.0/16` | eth3 |

The legacy prefixes are excluded from this NAT scope.

The rules were configured and verified, but source translation was not independently proven through packet capture.

### Phase 5 verification

The following results were observed:

- VyOS to `198.51.100.1`: 3/3 successful.
- VyOS to `203.0.113.1`: 5/5 successful.
- VPC7 to its default gateway: 5/5 successful.
- VPC7 to both synthetic external destinations: 5/5 successful.
- VPC7 to `192.0.2.1`: five timeouts under the tested policy.
- Firewall default-drop counters increased by five packets and 420 bytes during the denied test.
- The expected established/related traffic counter was observed.
- Default-route failover and restoration were verified.

### Known Phase 5 limitations

- VyOS is a single point of failure.
- Inter-VLAN routing on the distribution switches can bypass VyOS, so comprehensive internal segmentation is not implemented.
- NAT translation was not independently verified with packet capture.
- SSH service availability and authentication were not independently verified.
- Testing covered selected scenarios, not every possible link, routing, firewall or state-tracking failure.
- Legacy prefixes are not included in the configured enterprise NAT scope.
- The failover test validates health-check-target failure and recovery, not every possible upstream failure mode.

These limitations are retained as engineering findings rather than being represented as completed functionality.

---

# Topology and Packet-Path Considerations

The completed topology supports several categories of future security experiments.

## Enterprise client traffic

VPC7–VPC10 provide endpoint traffic from the access layer. VPC7 has been used for perimeter connectivity and denied-traffic tests.

## Legacy traffic

The LEGACY-1/LEGACY-2 path supports testing reachability, redistribution policy and interactions between the legacy domain and the enterprise network.

## Simulated Internet traffic

ISP-1 and ISP-2 advertise synthetic prefixes and defaults. They are sufficient for routing, failover, firewall-policy and selected traffic tests without requiring a connection to a real ISP.

## Security service traffic

Future server and DMZ services must have an explicitly documented IP address, subnet, gateway, routing path and enforcement point.

A service placed on a VLAN is not automatically protected by VyOS. Before claiming that a firewall rule protects a service, the relevant packet path must be demonstrated.

## Traffic capture

Endpoint reachability does not imply visibility into every packet exchanged in the network. IDS integration requires an appropriate capture source, such as a mirrored switch port, a suitable capture interface or a recorded packet capture.

---

# Verification and Failure Testing

The project uses configuration inspection, routing-table analysis, protocol-neighbor checks, end-to-end connectivity tests, deliberate fault injection and recovery validation.

Representative commands used across the Cisco routing and switching phases include:

```text
show vlan brief
show interfaces trunk
show etherchannel summary
show spanning-tree
show standby brief
show ip interface brief
show ip route
show ip ospf neighbor
show ip protocols
show ip eigrp neighbors
show ip bgp summary
show ip bgp
ping
traceroute
```

The exact command set depends on the device image and phase. VyOS uses its own operational and configuration commands, recorded in the relevant phase documentation.

Failure tests are intended to establish more than simple reachability. They should also confirm the expected route selection, policy behavior, failure detection, recovery, and absence of unintended side effects.

---

# Engineering Decisions and Lessons Learned

The completed phases have established several important engineering principles:

1. **Redundancy must be tested.** A backup link or route is only useful when failure detection, selection and recovery work as intended.
2. **Routing and forwarding are different.** An adjacency or route in a routing table does not, by itself, prove end-to-end service reachability.
3. **Redistribution requires policy.** Prefix filtering, route tagging and deliberate seed metrics reduce uncontrolled route propagation.
4. **Default-route failover is not universal destination failover.** A backup ISP cannot reach prefixes it does not advertise or otherwise route.
5. **A firewall must sit on the actual packet path.** Inter-VLAN routing performed at distribution switches can bypass a perimeter firewall.
6. **Configured NAT is not the same as proven translation.** Packet-level evidence is required to demonstrate source-address translation independently.
7. **Control-plane health and service health differ.** Healthy protocol neighbors do not guarantee that every service is reachable.
8. **Frozen baselines protect reproducibility.** Advanced experiments should use separate lab copies rather than silently changing completed phases.
9. **Evidence matters.** Successful tests, failed tests, root causes, recovery behavior and known limitations should all be recorded.
10. **Simulation has limits.** EVE-NG can demonstrate protocol behavior and architecture concepts, but some results depend on image implementation and do not automatically prove production hardware behavior.

---

# Future Work

The next development stage is security-focused, but implementation will begin only after an audit of the existing security resources and a confirmed test design.

## Security implementation roadmap

### 1. Security asset audit

- Review the existing Python Network Packet Analyzer.
- Identify its current detection logic, packet capture method, alert format and dependencies.
- Review the planned automated vulnerability scanner.
- Audit the Debian-lab2 DVWA/Burp environment and its existing code.
- Confirm available Linux images and EVE-NG resource capacity.
- Determine the practical connectivity options between the Lenovo lab host and the Kali workstation on the HP.

### 2. Protected services and service hardening

- Select lightweight Linux service images where appropriate.
- Define the target addresses, VLANs and permitted traffic paths.
- Test service exposure, access controls and host-level hardening.
- Preserve the existing Debian/DVWA setup until its integration requirements are clear.

### 3. IDS integration

- Establish a reliable packet capture path.
- Reuse and improve the existing Python analyzer.
- Validate detection using controlled test traffic.
- Record alert quality, false positives, missed traffic and limitations.

### 4. Vulnerability assessment and controlled attack simulation

- Integrate the automated vulnerability scanner once its capabilities and dependencies have been audited.
- Test only against authorized lab targets.
- Compare scanner findings with actual service configuration.
- Validate preventive controls and document remediation.

### 5. SIEM, investigation and response

- Determine whether a separate SIEM is necessary after evaluating the existing alerting capabilities.
- Establish event and alert collection.
- Correlate relevant events with controlled test activity.
- Document investigation, remediation and retesting.

These stages must not be represented as completed until their implementations and tests have been documented.

---

# Separate Advanced Networking Labs

Some technologies from the broader learning roadmap are deliberately excluded from the frozen capstone phases.

Potential separate projects include:

- MPLS L3VPN and Multi-Tenant Service Provider Network Architecture.
- VXLAN/EVPN Fabric Architecture.
- Advanced OSPF and BGP policy laboratories.
- Three-Tier and Collapsed-Core Campus Architecture.
- Routed-Access Campus Architecture.
- Data Center Spine–Leaf Architecture.
- Multi-Site Enterprise WAN Architecture.
- Hybrid Cloud Network Architecture.

These projects will reuse the knowledge gained from the capstone without expanding or destabilizing its frozen baseline.

---

# Project Status

| Area | Status |
|---|---|
| Phase 0 architecture baseline | Complete and frozen |
| Phase 1 Layer 2 | Complete and frozen |
| Phase 2 Layer 3, HSRP and OSPF | Complete and frozen |
| Phase 3 EIGRP/OSPF redistribution | Complete and frozen |
| Phase 4 dual-ISP eBGP | Complete and frozen |
| Phase 5 VyOS perimeter firewall | Complete; documentation complete |
| Core routing and addressing | Substantially complete |
| Protected service / DMZ implementation | Future work |
| Python IDS integration | Future work |
| Automated vulnerability scanner integration | Future work |
| SIEM and investigation workflow | Future work |
| End-to-end attack/detect/respond validation | Future work |

## Conclusion

This capstone establishes a tested enterprise network foundation with Layer 2 redundancy, Layer 3 gateway redundancy, multiple routing protocols, controlled legacy route redistribution, dual-ISP eBGP, and a stateful VyOS perimeter firewall.

The completed phases provide a reproducible baseline for subsequent security implementation. Future work will focus on protecting services, generating controlled attack traffic, observing network activity, detecting suspicious behavior, investigating alerts, and validating remediation.

The central engineering objective is not merely to configure a network that works under normal conditions. It is to demonstrate how the network behaves under failure, how policy affects packet forwarding, how problems are diagnosed, and how security controls can be tested with repeatable evidence.

---

**Disclaimer:** This is a virtualized educational lab intended for authorized testing. Synthetic address ranges, simulated ISP networks and virtual device behavior do not represent a production Internet connection or guarantee identical behavior on physical equipment.
