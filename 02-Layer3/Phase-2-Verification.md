# Phase 2 — Verification and Validation

## Enterprise Network Security Architecture and Intrusion Detection in a Virtualized Environment

**Phase:** 2 — Layer 3 / SVI / HSRP / OSPF
**Status:** Complete

---

# 1. Verification Objective

The purpose of this document is to demonstrate that the Phase 2 Layer 3 implementation operates correctly under normal conditions.

Verification covers:

* Layer 3 interfaces
* SVIs
* HSRP
* OSPF neighbors
* OSPF route exchange
* End-to-end connectivity
* Management network reachability
* Normal-state traceroutes

Failure and convergence testing is documented separately in:

```text
Phase-2-Failure-Testing.md
```

---

# 2. Layer 3 Verification

`ip routing` was enabled on:

* DIST-1
* DIST-2

The routing tables showed connected Layer 3 networks, confirming that the distribution switches were operating as Layer 3 routing devices.

---

# 3. SVI Verification

All production SVIs were verified as operational.

Configured networks:

```text
10.10.10.0/24
10.10.20.0/24
10.10.30.0/24
10.10.40.0/24
10.10.50.0/24
10.10.60.0/24
10.10.99.0/24
```

The SVIs were operational and the HSRP gateway addresses were reachable.

---

# 4. HSRP Verification

HSRP was verified with the following active distribution roles.

### DIST-1

Active:

```text
VLAN 10
VLAN 30
VLAN 50
```

Priority:

```text
110
```

Preemption:

```text
Enabled
```

### DIST-2

Active:

```text
VLAN 20
VLAN 40
VLAN 60
```

Priority:

```text
110
```

Preemption:

```text
Enabled
```

This confirms that gateway responsibility is distributed across the two distribution switches.

---

# 5. Transit VLAN Verification

VLAN 100:

```text
Name: DIST-TRANSIT
Network: 10.255.100.0/30
```

Addressing:

```text
DIST-1 = 10.255.100.1
DIST-2 = 10.255.100.2
```

The interface was verified as operational and the two addresses successfully communicated.

VLAN 100 was also permitted across the existing DIST-1 ↔ DIST-2 Port-Channel.

---

# 6. Routed Link Verification

The following routed links were operational.

| Link            | Network         |
| --------------- | --------------- |
| DIST-1 ↔ DIST-2 | 10.255.100.0/30 |
| DIST-1 ↔ EDGE-1 | 10.255.101.0/30 |
| DIST-2 ↔ EDGE-2 | 10.255.102.0/30 |
| EDGE-1 ↔ EDGE-2 | 10.255.103.0/30 |

All interfaces reached an operational Layer 3 state.

---

# 7. OSPF Neighbor Verification

The final OSPF neighbor state was:

### DIST-1

```text
Neighbor ID     State       Address          Interface
3.3.3.3         FULL/DR     10.255.101.2    Ethernet1/2
2.2.2.2         FULL/DR     10.255.100.2    Vlan100
```

### EDGE-1

```text
Neighbor ID     State       Address          Interface
4.4.4.4         FULL/DR     10.255.103.2    GigabitEthernet1
1.1.1.1         FULL/BDR    10.255.101.1    GigabitEthernet2
```

This confirms that the intended OSPF adjacencies formed.

---

# 8. OSPF Route Verification — DIST-1

`show ip route ospf` produced:

```text
O 10.255.102.0/30 [110/11] via 10.255.100.2
O 10.255.103.0/30 [110/11] via 10.255.101.2
```

DIST-1 therefore learned remote edge networks through OSPF.

---

# 9. OSPF Route Verification — DIST-2

DIST-2 learned:

```text
O 10.255.101.0/30 [110/11] via 10.255.100.1
O 10.255.103.0/30 [110/11] via 10.255.102.2
```

This confirms bidirectional route exchange.

---

# 10. OSPF Route Verification — EDGE-1

EDGE-1 learned the enterprise campus prefixes:

```text
O 10.10.10.0/24 via 10.255.101.1
O 10.10.20.0/24 via 10.255.101.1
O 10.10.30.0/24 via 10.255.101.1
O 10.10.40.0/24 via 10.255.101.1
O 10.10.50.0/24 via 10.255.101.1
O 10.10.60.0/24 via 10.255.101.1
O 10.10.99.0/24 via 10.255.101.1
```

EDGE-1 also learned:

```text
O 10.255.100.0/30 via 10.255.101.1
O 10.255.102.0/30 via 10.255.103.2
```

This confirms that the edge router successfully learned both the campus networks and the remote routed networks.

---

# 11. OSPF Route Verification — EDGE-2

EDGE-2 learned the same enterprise campus prefixes through DIST-2:

```text
O 10.10.10.0/24 via 10.255.102.1
O 10.10.20.0/24 via 10.255.102.1
O 10.10.30.0/24 via 10.255.102.1
O 10.10.40.0/24 via 10.255.102.1
O 10.10.50.0/24 via 10.255.102.1
O 10.10.60.0/24 via 10.255.102.1
O 10.10.99.0/24 via 10.255.102.1
```

It also learned:

```text
O 10.255.100.0/30 via 10.255.102.1
O 10.255.101.0/30 via 10.255.103.1
```

---

# 12. End-to-End Connectivity

Connectivity was tested from both edge routers toward the campus HSRP gateway addresses.

## EDGE-1

| Destination | Result |
| ----------- | ------ |
| 10.10.10.1  | 100%   |
| 10.10.20.1  | 80%    |
| 10.10.30.1  | 100%   |
| 10.10.40.1  | 80%    |
| 10.10.50.1  | 100%   |
| 10.10.60.1  | 80%    |

The 80% results occurred because the first ICMP request was lost during initial resolution; subsequent requests succeeded.

This was not a persistent routing failure.

Management addresses were also successfully reached:

```text
10.10.99.11
10.10.99.12
```

---

# 13. EDGE-2 Connectivity

EDGE-2 successfully reached the campus gateway addresses.

| Destination | Result |
| ----------- | ------ |
| 10.10.10.1  | 80%    |
| 10.10.20.1  | 100%   |
| 10.10.30.1  | 80%    |
| 10.10.40.1  | 100%   |
| 10.10.50.1  | 80%    |
| 10.10.60.1  | 100%   |

Again, the occasional first-packet loss was associated with initial resolution rather than a persistent connectivity problem.

Management SVI addresses were also successfully reachable.

---

# 14. Normal-State Traceroute

Traceroute was performed before failure testing to establish a baseline.

## EDGE-1 → VLAN 20

```text
1  10.255.101.1
2  10.10.20.3
```

The path entered the campus through DIST-1 and reached DIST-2, which was the active HSRP router for VLAN 20.

---

## EDGE-1 → VLAN 30

```text
1  10.255.101.1
```

DIST-1 was the active HSRP router for VLAN 30, so the destination was reached through DIST-1.

---

## EDGE-2 → VLAN 30

```text
1  10.255.102.1
2  10.10.30.2
```

EDGE-2 entered the campus through DIST-2 and reached DIST-1, which was the active HSRP router for VLAN 30.

---

## EDGE-2 → VLAN 20

```text
1  10.255.102.1
```

DIST-2 was the active HSRP router for VLAN 20.

---

# 15. Verification Interpretation

The verification results demonstrate that:

1. The distribution switches operate as Layer 3 devices.
2. Production VLANs have operational SVIs.
3. HSRP provides redundant gateway functionality.
4. VLAN 100 provides dedicated routing transit.
5. OSPF adjacencies form only on the intended routed/transit links.
6. OSPF successfully advertises campus networks.
7. EDGE-1 and EDGE-2 learn the enterprise campus routes.
8. The campus can be reached from both edge routers.
9. HSRP determines the active gateway for each production VLAN.
10. The routing topology supports an alternate path for failure recovery.

---

# 16. Verification Result

```text
Layer 3 Routing             PASS
SVIs                        PASS
HSRP                        PASS
VLAN 100 Transit            PASS
Routed Links                PASS
OSPF Adjacencies            PASS
OSPF Route Exchange         PASS
Campus Route Learning       PASS
EDGE-1 Reachability         PASS
EDGE-2 Reachability         PASS
Normal Traceroute           PASS
```

**Verification Gate: COMPLETE**

The network was ready for controlled failure testing.
