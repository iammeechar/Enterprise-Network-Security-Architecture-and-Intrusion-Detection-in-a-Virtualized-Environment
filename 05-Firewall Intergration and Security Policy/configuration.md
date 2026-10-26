# Phase 5 — Firewall Integration and Security Policy

## 1. Phase Information

- **Project:** Enterprise Network Security Architecture and Intrusion Detection in a Virtualized Environment
- **Phase:** 5 — Firewall Integration and Security Policy
- **Platform:** EVE-NG
- **Firewall:** VyOS 1.5-rolling-202406170021
- **Design:** Option B — Explicit Routing Boundary
- **Status:** Implemented; initial connectivity and failover tests passed; documentation review pending
- **Baseline:** Frozen Phase 4 implementation
- **Configuration:** Committed and saved after restoration of the original primary health-check target

## 2. Objective

Integrate VyOS into the existing enterprise network as an explicit perimeter firewall and routing boundary between the enterprise distribution layer and the two simulated ISP/edge paths.

The objectives are to:

1. Establish routed connectivity between the enterprise distribution layer, VyOS, and both edge routers.
2. Use explicit static routes to control route ownership and forwarding behavior.
3. Provide a preferred default route through EDGE-1 and a backup through EDGE-2.
4. Apply zone-based firewall policies to traffic crossing the VyOS boundary.
5. Configure source NAT for enterprise IPv4 traffic exiting either WAN-facing interface.
6. Verify connectivity, policy behavior, and failover/recovery without redesigning the frozen enterprise core.

## 3. Design Decision

### Option B — Explicit Routing Boundary

VyOS operates as a distinct routed firewall boundary. It does not participate in the enterprise OSPF domain.

Enterprise routes are explicitly configured on VyOS using static routes through DIST-1 and DIST-2. The default route points toward EDGE-1 under normal conditions, with EDGE-2 available as a backup.

This approach was selected because it:

- Makes the firewall's routing responsibilities explicit.
- Separates the enterprise IGP from the perimeter firewall.
- Makes the intended packet path easier to audit and troubleshoot.
- Limits changes to the previously frozen routing architecture.

Extending OSPF through VyOS (Option A) remains a separate research option and is not part of this implementation.

## 4. Topology and Addressing

### 4.1 VyOS interfaces

| VyOS interface | Address | Connected device/interface | Purpose |
|---|---|---|---|
| eth0 | 10.255.101.2/30 | DIST-1 Et1/2 — 10.255.101.1 | Enterprise transit |
| eth1 | 10.255.102.2/30 | DIST-2 Et1/2 — 10.255.102.1 | Enterprise transit |
| eth2 | 10.255.105.1/30 | EDGE-1 Gi2 — 10.255.105.2 | Primary WAN/edge path |
| eth3 | 10.255.106.1/30 | EDGE-2 Gi2 — 10.255.106.2 | Backup WAN/edge path |

All four interfaces were observed operationally as up/up during implementation verification.

### 4.2 Enterprise VLANs

| VLAN | Name | Subnet |
|---:|---|---|
| 10 | USERS | 10.10.10.0/24 |
| 20 | VOICE | 10.10.20.0/24 |
| 30 | SERVERS | 10.10.30.0/24 |
| 40 | SECURITY | 10.10.40.0/24 |
| 50 | DMZ | 10.10.50.0/24 |
| 60 | GUEST | 10.10.60.0/24 |
| 99 | MANAGEMENT | 10.10.99.0/24 |

The enterprise VLAN gateways remain on the distribution layer using the existing HSRP configuration. VyOS does not replace those gateways.

VLAN 100, `10.255.100.0/30`, remains the DIST-1/DIST-2 transit network from the earlier design. It is not a direct VyOS interface network in this phase.

### 4.3 Legacy networks

| Network | Subnet |
|---|---|
| Legacy network 1 | 172.16.10.0/24 |
| Legacy network 2 | 172.16.20.0/24 |

These networks remain part of the earlier EIGRP/OSPF redistribution design. VyOS reaches them through the distribution layer.

### 4.4 Simulated ISP test destinations

| Destination | Purpose |
|---|---|
| 198.51.100.1 | Primary-path health-check and connectivity test |
| 203.0.113.1 | Backup-path health-check and connectivity test |
| 198.51.100.254 | Temporary nonresponsive probe target used during the controlled failover test |

The test destinations are documentation/example address ranges used in the virtual lab, not public Internet destinations.

## 5. Routing Configuration

### 5.1 Explicit enterprise routes

VyOS uses static routes to reach the enterprise VLAN and legacy prefixes. Each destination has a preferred next hop and a backup next hop.

| Destination | Preferred next hop | Backup next hop | Administrative distance |
|---|---|---|---|
| `10.10.10.0/24` | `10.255.101.1` | `10.255.102.1` | 1 preferred; 10 backup |
| `10.10.20.0/24` | `10.255.102.1` | `10.255.101.1` | 1 preferred; 10 backup |
| `10.10.30.0/24` | `10.255.101.1` | `10.255.102.1` | 1 preferred; 10 backup |
| `10.10.40.0/24` | `10.255.102.1` | `10.255.101.1` | 1 preferred; 10 backup |
| `10.10.50.0/24` | `10.255.101.1` | `10.255.102.1` | 1 preferred; 10 backup |
| `10.10.60.0/24` | `10.255.102.1` | `10.255.101.1` | 1 preferred; 10 backup |
| `10.10.99.0/24` | `10.255.101.1` | `10.255.102.1` | 1 preferred; 10 backup |
| `172.16.10.0/24` | `10.255.102.1` | `10.255.101.1` | 1 preferred; 10 backup |
| `172.16.20.0/24` | `10.255.102.1` | `10.255.101.1` | 1 preferred; 10 backup |

**Administrative distance and route preference**

The preferred static route uses the default administrative distance of **1**, while the backup static route uses an administrative distance of **10**. Since a lower administrative distance is preferred, VyOS selects the primary route when both routes are available.

The backup route provides an alternative path if the preferred route is withdrawn from the routing table. This is route preference based on administrative distance; it does not, by itself, guarantee detection of every downstream failure. The actual failover behavior depends on route availability and the mechanisms used to detect failures.

### 5.2 Default route and health checks

The default route uses VyOS failover tracking:

| Next hop | Interface | Metric | Probe target |
|---|---|---:|---|
| 10.255.105.2 (EDGE-1) | eth2 | 10 | 198.51.100.1 |
| 10.255.106.2 (EDGE-2) | eth3 | 20 | 203.0.113.1 |

Both next hops use ICMP health checks with a five-second timeout.

The intended behavior is:

- Prefer EDGE-1 while its tracked path is healthy.
- Withdraw the primary route when its probe fails.
- Prefer EDGE-2 while the primary is unavailable.
- Restore EDGE-1 as the best route after the original probe recovers.

### 5.3 Edge-router return routes

The simulated ISP routers have routes back to the VyOS transit networks:

| Router | Destination | Next hop |
|---|---|---|
| ISP-1 | 10.255.105.0/30 | 10.255.110.1 |
| ISP-1 | 10.255.106.0/30 | 10.255.111.1 |
| ISP-2 | 10.255.105.0/30 | 10.255.112.1 |
| ISP-2 | 10.255.106.0/30 | 10.255.113.1 |

These routes support return reachability to the simulated firewall transit networks.

## 6. Source NAT

Two source NAT masquerade rules are configured:

| Rule | Source | Outbound interface | Translation |
|---:|---|---|---|
| 100 | 10.10.0.0/16 | eth2 | Masquerade |
| 110 | 10.10.0.0/16 | eth3 | Masquerade |

The rules are intended to translate enterprise source addresses when traffic exits through either edge path.

The legacy networks `172.16.10.0/24` and `172.16.20.0/24` are not included in these NAT rules.

Successful end-to-end ICMP tests are consistent with the expected forwarding and return path. They do not, by themselves, independently prove the translated source address. NAT translation should be treated as configured, with packet-level verification remaining a separate task if required.

## 7. Firewall Zones and Policy

The implementation uses four interfaces across three logical zones:

| Zone | Interfaces | Role |
|---|---|---|
| LAN | eth0, eth1 | Enterprise-facing traffic |
| WAN | eth2, eth3 | Edge/ISP-facing traffic |
| LOCAL | VyOS itself | Traffic destined to or originating from the firewall |

The configured zone policies use default-drop behavior, with explicit rules for permitted traffic.

### 7.1 LAN-to-WAN

Policy: `PH5-LAN-WAN`

- Accept established connections.
- Accept related traffic.
- Drop invalid traffic.
- Permit new ICMP traffic to `198.51.100.0/24`.
- Permit new ICMP traffic to `203.0.113.0/24`.
- Drop other traffic by default.

This is a deliberately restricted initial verification policy. It is not a general-purpose Internet access policy.

### 7.2 WAN-to-LAN

Policy: `PH5-WAN-LAN`

- Accept established connections.
- Accept related traffic.
- Drop invalid traffic.
- Drop other traffic by default.

### 7.3 LAN-to-LOCAL

Policy: `PH5-LAN-LOCAL`

- Accept established connections.
- Accept related traffic.
- Drop invalid traffic.
- Permit ICMP from the management subnet `10.10.99.0/24`.
- Permit TCP port 22 from the management subnet `10.10.99.0/24`.
- Drop other traffic by default.

SSH service availability and management access should be verified independently from the existence of the policy rules.

### 7.4 LOCAL-to-LAN

Policy: `PH5-LOCAL-LAN`

- Accept established connections.
- Accept related traffic.
- Drop invalid traffic.
- Permit ICMP.
- Drop other traffic by default.

### 7.5 LOCAL-to-WAN

Policy: `PH5-LOCAL-WAN`

- Accept established connections.
- Accept related traffic.
- Drop invalid traffic.
- Permit router-originated ICMP for health checks.
- Drop other traffic by default.

### 7.6 WAN-to-LOCAL

Policy: `PH5-WAN-LOCAL`

- Accept established connections.
- Accept related traffic.
- Drop invalid traffic.
- Drop other traffic by default.

## 8. Architectural Limitations

1. **Single firewall instance:** VyOS is a single point of failure in this lab. Dual firewalls, state synchronization, and high availability are not implemented.
2. **Inter-VLAN traffic:** Traffic routed directly between VLAN SVIs on DIST-1 and DIST-2 does not automatically traverse VyOS. The perimeter firewall therefore does not enforce all inter-VLAN security policies.
3. **Restricted outbound policy:** New LAN-to-WAN traffic is initially limited to ICMP toward the two documented test prefixes. General TCP/UDP application access is not enabled by this policy.
4. **Health-check scope:** A successful probe verifies reachability to its target, not the health of every service or every part of the Internet.
5. **NAT evidence:** NAT rules are configured, but a packet capture or equivalent source-address observation has not been recorded as direct proof of translation.
6. **Management services:** A firewall rule permitting SSH does not establish that the SSH service is enabled, listening, and reachable.

## 9. Configuration Management

The reconstructed VyOS configuration was committed and saved. The temporary failover-test target was subsequently removed, the original primary probe was restored, the change was committed, and `save` was executed again.

The final displayed configuration contains:

- Primary probe target `198.51.100.1`.
- Backup probe target `203.0.113.1`.
- Primary default route via EDGE-1 with metric 10.
- Backup default route via EDGE-2 with metric 20.

This file documents the intended final state. The live device configuration remains the authoritative source if a discrepancy is discovered during a later audit.

## 10. Phase 5 Scope Boundary

This phase covers perimeter firewall integration, explicit routing, source NAT configuration, basic security policy, and failover/recovery verification.

Detailed firewall-mechanism experiments—including expanded service policies, protocol-specific controls, deeper packet inspection, and additional attack scenarios—are deferred to a separate lab. They must not be treated as implemented merely because they are described as future work.
