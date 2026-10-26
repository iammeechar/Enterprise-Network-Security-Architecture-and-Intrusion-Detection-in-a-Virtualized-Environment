 # Phase 4 — Internet Edge and eBGP Configuration

## 1. Overview

**Project:** Enterprise Network Security Architecture and Intrusion Detection in a Virtualized Environment  
**Phase:** 4 — Internet Edge and BGP  
**Platform:** EVE-NG, Cisco CSR1000v IOS XE 17.03.08a  
**Status:** Implemented, verified, failure-tested, and saved to startup configuration

### Objective

Introduce a simulated dual-ISP Internet edge using external Border Gateway Protocol (eBGP), controlled prefix advertisements, inbound route filtering, primary/backup default-route selection, and controlled default-route injection into the existing enterprise OSPF domain.

### Scope and guardrails

- Preserve the frozen Phase 0–3 topology and configurations.
- Use two enterprise edge routers and two simulated ISP routers.
- Establish four eBGP sessions.
- Advertise a synthetic enterprise test prefix rather than internal campus networks.
- Filter inbound and outbound BGP routes using prefix lists and route maps.
- Prefer ISP-1 over ISP-2 for the default route.
- Advertise an OSPF default only while EDGE-1 has a default route.
- Do not run OSPF on ISP routers.
- Do not redistribute BGP routes into OSPF.
- Do not introduce the FortiGate in this phase.

> **Lab limitation:** These are simulated provider networks and test prefixes. A default route in this lab does not represent real Internet connectivity.

---

## 2. Topology and addressing

### Autonomous systems

| Router | Role | BGP AS |
|---|---|---:|
| EDGE-1 | Enterprise edge 1 | 65010 |
| EDGE-2 | Enterprise edge 2 | 65010 |
| ISP-1 | Primary simulated provider | 65001 |
| ISP-2 | Backup simulated provider | 65002 |

### eBGP point-to-point links

| Connection | Enterprise interface/IP | ISP interface/IP | Subnet |
|---|---|---|---|
| EDGE-1 — ISP-1 | Gi3 — `10.255.110.1/30` | Gi1 — `10.255.110.2/30` | `10.255.110.0/30` |
| EDGE-2 — ISP-1 | Gi3 — `10.255.111.1/30` | Gi2 — `10.255.111.2/30` | `10.255.111.0/30` |
| EDGE-1 — ISP-2 | Gi4 — `10.255.112.1/30` | Gi1 — `10.255.112.2/30` | `10.255.112.0/30` |
| EDGE-2 — ISP-2 | Gi4 — `10.255.113.1/30` | Gi2 — `10.255.113.2/30` | `10.255.113.0/30` |

### Synthetic test prefixes

| Prefix | Origin | Purpose |
|---|---|---|
| `192.0.2.0/24` | Enterprise EDGE-1 Loopback0 | Enterprise test prefix advertised to providers |
| `198.51.100.0/24` | ISP-1 Loopback0 | ISP-1-specific reachability test |
| `203.0.113.0/24` | ISP-2 Loopback0 | ISP-2-specific reachability test |
| `0.0.0.0/0` | ISP-1 and ISP-2 static Null0 routes | Simulated provider default route |

Loopback test endpoints:

- EDGE-1: `192.0.2.1/24`
- ISP-1: `198.51.100.1/24`
- ISP-2: `203.0.113.1/24`

The enterprise prefix is learned by EDGE-2 through the existing OSPF adjacency before EDGE-2 advertises it through BGP.

---

## 3. Existing enterprise routing preserved

The following pre-existing design was retained:

- OSPF process 100, Area 0.
- EDGE-1 OSPF router ID `3.3.3.3`.
- EDGE-2 OSPF router ID `4.4.4.4`.
- EDGE-1 — DIST-1 link: `10.255.101.0/30`.
- EDGE-1 — EDGE-2 link: `10.255.103.0/30`.
- EDGE-2 — DIST-2 link: `10.255.102.0/30`.
- Phase 3 EIGRP-to-OSPF redistribution and its legacy routes remain in place.

The ISP-facing links are not included in OSPF. BGP is used between the enterprise edge and providers, while OSPF remains the internal enterprise routing protocol.

---

## 4. BGP policy design

### Enterprise inbound policy

Both enterprise edge routers accept only:

- `198.51.100.0/24` from ISP-1.
- `203.0.113.0/24` from ISP-2.
- `0.0.0.0/0` from either provider.

Local preference controls outbound provider selection inside AS 65010:

| Provider | Local preference | Intended result |
|---|---:|---|
| ISP-1 | 200 | Preferred |
| ISP-2 | 100 | Backup |

Local preference is an internal BGP path-selection attribute. The higher value makes the ISP-1 path preferable when competing eligible routes exist.

### Enterprise outbound policy

The enterprise advertises only:

- `192.0.2.0/24`

Internal VLANs, legacy networks, and other enterprise routes are not advertised to the simulated providers.

### ISP inbound policy

Both ISP routers accept only the enterprise prefix:

- `192.0.2.0/24`

### ISP outbound policy

ISP-1 advertises only:

- `198.51.100.0/24`
- `0.0.0.0/0`

ISP-2 advertises only:

- `203.0.113.0/24`
- `0.0.0.0/0`

This prevents the two simulated providers from advertising their test prefixes to one another through the enterprise AS.

---

## 5. EDGE-1 configuration elements

The following represents the Phase 4 configuration elements. It is not a replacement for the router's complete running configuration.

### Test loopback and ISP-facing interfaces

```cisco
interface Loopback0
 ip address 192.0.2.1 255.255.255.0

interface GigabitEthernet3
 description EBGP-TO-ISP-1
 ip address 10.255.110.1 255.255.255.252
 no shutdown

interface GigabitEthernet4
 description EBGP-TO-ISP-2
 ip address 10.255.112.1 255.255.255.252
 no shutdown
```

### Prefix lists and route maps

```cisco
ip prefix-list FROM-ISP1 seq 10 permit 198.51.100.0/24
ip prefix-list FROM-ISP1 seq 20 permit 0.0.0.0/0

ip prefix-list FROM-ISP2 seq 10 permit 203.0.113.0/24
ip prefix-list FROM-ISP2 seq 20 permit 0.0.0.0/0

ip prefix-list ENTERPRISE-TEST seq 10 permit 192.0.2.0/24

route-map ISP1-IN permit 10
 match ip address prefix-list FROM-ISP1
 set local-preference 200

route-map ISP2-IN permit 10
 match ip address prefix-list FROM-ISP2
 set local-preference 100

route-map ENTERPRISE-OUT permit 10
 match ip address prefix-list ENTERPRISE-TEST
```

### BGP configuration

```cisco
router bgp 65010
 bgp log-neighbor-changes
 network 192.0.2.0
 neighbor 10.255.110.2 remote-as 65001
 neighbor 10.255.110.2 route-map ISP1-IN in
 neighbor 10.255.110.2 route-map ENTERPRISE-OUT out
 neighbor 10.255.112.2 remote-as 65002
 neighbor 10.255.112.2 route-map ISP2-IN in
 neighbor 10.255.112.2 route-map ENTERPRISE-OUT out
```

### Controlled OSPF default origination

```cisco
router ospf 100
 router-id 3.3.3.3
 network 10.255.101.0 0.0.0.3 area 0
 network 10.255.103.0 0.0.0.3 area 0
 default-information originate
```

The `default-information originate` command does not include `always`. EDGE-1 therefore originates an OSPF default only when a default route exists in its routing table.

The existing OSPF statements shown above were retained. No ISP-facing interface was added to OSPF.

---

## 6. EDGE-2 configuration elements

### Test and ISP-facing interfaces

```cisco
interface GigabitEthernet3
 description EBGP-TO-ISP-1
 ip address 10.255.111.1 255.255.255.252
 no shutdown

interface GigabitEthernet4
 description EBGP-TO-ISP-2
 ip address 10.255.113.1 255.255.255.252
 no shutdown
```

EDGE-2 learns `192.0.2.0/24` through the existing enterprise OSPF network. Its BGP `network` statement originates that prefix only while a matching route is present in its routing table.

### Prefix lists and route maps

The inbound and outbound policies mirror EDGE-1, using the same prefix lists and local preferences:

```cisco
ip prefix-list FROM-ISP1 seq 10 permit 198.51.100.0/24
ip prefix-list FROM-ISP1 seq 20 permit 0.0.0.0/0

ip prefix-list FROM-ISP2 seq 10 permit 203.0.113.0/24
ip prefix-list FROM-ISP2 seq 20 permit 0.0.0.0/0

ip prefix-list ENTERPRISE-TEST seq 10 permit 192.0.2.0/24

route-map ISP1-IN permit 10
 match ip address prefix-list FROM-ISP1
 set local-preference 200

route-map ISP2-IN permit 10
 match ip address prefix-list FROM-ISP2
 set local-preference 100

route-map ENTERPRISE-OUT permit 10
 match ip address prefix-list ENTERPRISE-TEST
```

### BGP configuration

```cisco
router bgp 65010
 bgp log-neighbor-changes
 network 192.0.2.0
 neighbor 10.255.111.2 remote-as 65001
 neighbor 10.255.111.2 route-map ISP1-IN in
 neighbor 10.255.111.2 route-map ENTERPRISE-OUT out
 neighbor 10.255.113.2 remote-as 65002
 neighbor 10.255.113.2 route-map ISP2-IN in
 neighbor 10.255.113.2 route-map ENTERPRISE-OUT out
```

### Existing OSPF configuration

```cisco
router ospf 100
 router-id 4.4.4.4
 network 10.255.102.0 0.0.0.3 area 0
 network 10.255.103.0 0.0.0.3 area 0
```

EDGE-2 does not originate an OSPF default in this phase.

---

## 7. ISP-1 configuration elements

### Interfaces and test loopback

```cisco
interface Loopback0
 ip address 198.51.100.1 255.255.255.0

interface GigabitEthernet1
 description EBGP-TO-EDGE-1
 ip address 10.255.110.2 255.255.255.252
 no shutdown

interface GigabitEthernet2
 description EBGP-TO-EDGE-2
 ip address 10.255.111.2 255.255.255.252
 no shutdown
```

### Default route and policy

```cisco
ip route 0.0.0.0 0.0.0.0 Null0

ip prefix-list ENTERPRISE-ONLY seq 10 permit 192.0.2.0/24

ip prefix-list ISP1-OUT seq 10 permit 198.51.100.0/24
ip prefix-list ISP1-OUT seq 20 permit 0.0.0.0/0

route-map ENTERPRISE-IN permit 10
 match ip address prefix-list ENTERPRISE-ONLY

route-map ISP1-OUT permit 10
 match ip address prefix-list ISP1-OUT
```

### BGP configuration

```cisco
router bgp 65001
 bgp log-neighbor-changes
 network 198.51.100.0 mask 255.255.255.0
 network 0.0.0.0
 neighbor 10.255.110.1 remote-as 65010
 neighbor 10.255.110.1 route-map ENTERPRISE-IN in
 neighbor 10.255.110.1 route-map ISP1-OUT out
 neighbor 10.255.111.1 remote-as 65010
 neighbor 10.255.111.1 route-map ENTERPRISE-IN in
 neighbor 10.255.111.1 route-map ISP1-OUT out
```

The default route is synthetic and terminates at Null0. It exists to test BGP default propagation and route-selection behavior.

---

## 8. ISP-2 configuration elements

### Interfaces and test loopback

```cisco
interface Loopback0
 ip address 203.0.113.1 255.255.255.0

interface GigabitEthernet1
 description EBGP-TO-EDGE-1
 ip address 10.255.112.2 255.255.255.252
 no shutdown

interface GigabitEthernet2
 description EBGP-TO-EDGE-2
 ip address 10.255.113.2 255.255.255.252
 no shutdown
```

### Default route and policy

```cisco
ip route 0.0.0.0 0.0.0.0 Null0

ip prefix-list ENTERPRISE-ONLY seq 10 permit 192.0.2.0/24

ip prefix-list ISP2-OUT seq 10 permit 203.0.113.0/24
ip prefix-list ISP2-OUT seq 20 permit 0.0.0.0/0

route-map ENTERPRISE-IN permit 10
 match ip address prefix-list ENTERPRISE-ONLY

route-map ISP2-OUT permit 10
 match ip address prefix-list ISP2-OUT
```

### BGP configuration

```cisco
router bgp 65002
 bgp log-neighbor-changes
 network 203.0.113.0 mask 255.255.255.0
 network 0.0.0.0
 neighbor 10.255.112.1 remote-as 65010
 neighbor 10.255.112.1 route-map ENTERPRISE-IN in
 neighbor 10.255.112.1 route-map ISP2-OUT out
 neighbor 10.255.113.1 remote-as 65010
 neighbor 10.255.113.1 route-map ENTERPRISE-IN in
 neighbor 10.255.113.1 route-map ISP2-OUT out
```

ISP-2 does not run OSPF and does not advertise ISP-1's test prefix.

---

## 9. Important implementation notes

1. **BGP network statements require a matching route.** The enterprise and ISP test prefixes are backed by loopback interfaces. Each provider's default is backed by a static Null0 route.
2. **Route maps have implicit deny behavior.** Only prefixes explicitly matched by the permitted entries are advertised or accepted.
3. **The two enterprise edge routers share AS 65010.** Sharing an AS number does not automatically make one edge's external routes available to the other. The internal OSPF path and the configured BGP policy determine which routes are available.
4. **The provider test prefixes are intentionally provider-specific.** If ISP-1's link fails, the enterprise can select ISP-2's default without gaining a specific route to ISP-1's loopback.
5. **The OSPF default is conditional.** Removing both provider defaults removes EDGE-1's default route, causing its OSPF default advertisement to withdraw.
6. **Configuration persistence:** The running configurations were saved to startup configuration on EDGE-1, EDGE-2, ISP-1, and ISP-2 after testing.

## 10. Verification commands

Useful commands for reviewing the implementation:

```cisco
show ip interface brief
show ip route
show ip route 192.0.2.0
show ip route 198.51.100.0
show ip route 203.0.113.0
show ip bgp summary
show ip bgp
show ip bgp neighbors
show ip bgp neighbors <peer-ip> advertised-routes
show ip protocols
show ip ospf neighbor
show ip prefix-list
show route-map
show running-config | section router bgp
show running-config | section router ospf
show running-config | include ^ip route
show startup-config
```

Replace `<peer-ip>` with the neighbor address being checked.

## 11. Final status

Phase 4 is implemented and saved. BGP peering, prefix filtering, provider preference, data-plane reachability, default-route withdrawal and recovery, and single-link failure behavior were tested. Detailed results belong in `verification.md` and `failure-testing.md`.
