# Phase 2 — Layer 3, SVI, HSRP and OSPF Configuration

## Enterprise Network Security Architecture and Intrusion Detection in a Virtualized Environment

**Platform:** EVE-NG
**Phase:** 2 — Layer 3 / SVI / HSRP / OSPF
**Status:** Complete
**Previous Phase:** Phase 1 — Enterprise Layer 2
**Next Phase:** EIGRP + Redistribution

---

# 1. Phase Objective

Phase 2 extends the completed Layer 2 enterprise campus into a routed Layer 3 architecture.

The objective was to introduce:

* Layer 3 routing at the distribution layer
* Switched Virtual Interfaces (SVIs)
* First Hop Redundancy Protocol (HSRP)
* A dedicated Layer 3 transit network
* OSPF Area 0
* Routed connections between the campus and WAN/edge routers
* OSPF route exchange across the enterprise topology
* Redundant routing paths
* Controlled failure and convergence testing

The phase follows the project lifecycle:

```text
BUILD
  ↓
VERIFY
  ↓
FAILURE TESTING
  ↓
TROUBLESHOOT
  ↓
DOCUMENT
  ↓
FREEZE
```

The Layer 2 foundation from Phase 1 was retained.

---

# 2. Phase 1 Starting State

Phase 1 completed the enterprise Layer 2 foundation.

The campus consisted of:

* DIST-1
* DIST-2
* ACC-1
* ACC-2
* ACC-3
* ACC-4

The Layer 2 design included:

* VLAN segmentation
* 802.1Q trunks
* LACP between DIST-1 and DIST-2
* Rapid PVST+
* Redundant access uplinks
* VLAN 99 management
* DIST-1 as the primary STP root
* DIST-2 as the secondary STP root

Phase 2 does not redesign this Layer 2 architecture.

Instead, Layer 3 functionality is added at the distribution layer.

---

# 3. Phase 2 Architecture

The Layer 3 architecture introduces two additional Cisco CSR1000v routers:

* EDGE-1
* EDGE-2

Both use:

```text
csr1000vng-universalk9.17.03.08a-serial
```

The resulting routed topology is:

```text
                    EDGE-1
                   3.3.3.3
                  /       \
                 /         \
            DIST-1         EDGE-2
           1.1.1.1        4.4.4.4
               \             /
                \           /
                 \         /
                  DIST-2
                 2.2.2.2
```

OSPF operates across the routed portion of this topology.

The four-router ring provides an alternate routing path when a routed link fails.

---

# 4. Additional Equipment

## EDGE-1

| Attribute      | Value                      |
| -------------- | -------------------------- |
| Device         | Cisco CSR1000v             |
| Image          | 17.03.08a                  |
| Role           | Enterprise WAN/edge router |
| OSPF Router ID | 3.3.3.3                    |

## EDGE-2

| Attribute      | Value                      |
| -------------- | -------------------------- |
| Device         | Cisco CSR1000v             |
| Image          | 17.03.08a                  |
| Role           | Enterprise WAN/edge router |
| OSPF Router ID | 4.4.4.4                    |

The edge routers were introduced because the final enterprise architecture requires a routed WAN/edge domain and future support for technologies including:

* EIGRP redistribution
* BGP
* MPLS
* VRF
* MP-BGP / L3VPN
* WAN redundancy

They are therefore part of the capstone architecture rather than temporary devices added only to demonstrate OSPF.

---

# 5. Layer 3 Design Improvements

The following improvements were introduced during Phase 2.

## 5.1 Distribution Layer Routing

DIST-1 and DIST-2 were converted from Layer 2-only distribution switches into Layer 3 routing devices.

```cisco
ip routing
```

This establishes the distribution layer as the campus Layer 3 boundary.

---

## 5.2 Switched Virtual Interfaces

SVIs were configured for the production VLANs.

The addressing model uses:

```text
.1 = HSRP virtual gateway
.2 = DIST-1
.3 = DIST-2
```

for VLANs 10–60.

---

# 6. SVI and Gateway Addressing

| VLAN | Name     | Network       | DIST-1     | DIST-2     | HSRP VIP   |
| ---: | -------- | ------------- | ---------- | ---------- | ---------- |
|   10 | USERS    | 10.10.10.0/24 | 10.10.10.2 | 10.10.10.3 | 10.10.10.1 |
|   20 | VOICE    | 10.10.20.0/24 | 10.10.20.2 | 10.10.20.3 | 10.10.20.1 |
|   30 | SERVERS  | 10.10.30.0/24 | 10.10.30.2 | 10.10.30.3 | 10.10.30.1 |
|   40 | SECURITY | 10.10.40.0/24 | 10.10.40.2 | 10.10.40.3 | 10.10.40.1 |
|   50 | DMZ      | 10.10.50.0/24 | 10.10.50.2 | 10.10.50.3 | 10.10.50.1 |
|   60 | GUEST    | 10.10.60.0/24 | 10.10.60.2 | 10.10.60.3 | 10.10.60.1 |

Management VLAN:

| VLAN | Name       | Network       | DIST-1      | DIST-2      |
| ---: | ---------- | ------------- | ----------- | ----------- |
|   99 | MANAGEMENT | 10.10.99.0/24 | 10.10.99.11 | 10.10.99.12 |

VLAN 99 does not use an HSRP `.1` address in the current design.

---

# 7. HSRP Design

HSRP was introduced to provide a redundant default gateway for the production VLANs.

The distribution switches share gateway responsibility.

### DIST-1 Active

* VLAN 10
* VLAN 30
* VLAN 50

Priority:

```text
110
```

Preemption is enabled.

### DIST-2 Active

* VLAN 20
* VLAN 40
* VLAN 60

Priority:

```text
110
```

Preemption is enabled.

This provides gateway load sharing while maintaining redundancy.

The design avoids relying on a single distribution switch as the gateway for every VLAN.

---

# 8. Dedicated OSPF Transit Network

A dedicated transit VLAN was introduced between DIST-1 and DIST-2.

```text
VLAN 100
Name: DIST-TRANSIT
Network: 10.255.100.0/30
```

Addressing:

| Device | Interface | Address         |
| ------ | --------- | --------------- |
| DIST-1 | VLAN 100  | 10.255.100.1/30 |
| DIST-2 | VLAN 100  | 10.255.100.2/30 |

VLAN 100 is carried over the existing DIST-1 ↔ DIST-2 Port-Channel.

The Port-Channel was extended to permit VLAN 100:

```text
10,20,30,40,50,60,99-100
```

---

# 9. Why a Dedicated Transit VLAN Was Used

Production VLANs already provide Layer 2 connectivity for users, servers, security systems, DMZ services and guests.

Using those VLANs as OSPF adjacency networks would create unnecessary OSPF neighbor relationships.

The dedicated transit VLAN separates:

```text
Production VLANs
      ↓
HSRP / Gateway Functions

VLAN 100
      ↓
OSPF Transit / Routing Functions
```

Production VLAN interfaces were therefore configured as passive OSPF interfaces.

VLAN 100 remains an active OSPF interface.

This provides a cleaner separation between:

* Layer 2 segmentation
* Gateway redundancy
* Dynamic routing

---

# 10. Routed Distribution-to-Edge Links

Three /30 routed networks were introduced.

## DIST-1 ↔ EDGE-1

```text
10.255.101.0/30
```

| Device | Interface        | Address      |
| ------ | ---------------- | ------------ |
| DIST-1 | Ethernet1/2      | 10.255.101.1 |
| EDGE-1 | GigabitEthernet2 | 10.255.101.2 |

DIST-1 interface:

```cisco
interface Ethernet1/2
 description L3-TRANSIT-TO-EDGE-1
 no switchport
 ip address 10.255.101.1 255.255.255.252
 no shutdown
```

---

## DIST-2 ↔ EDGE-2

```text
10.255.102.0/30
```

| Device | Interface        | Address      |
| ------ | ---------------- | ------------ |
| DIST-2 | Ethernet1/2      | 10.255.102.1 |
| EDGE-2 | GigabitEthernet2 | 10.255.102.2 |

---

## EDGE-1 ↔ EDGE-2

```text
10.255.103.0/30
```

| Device | Interface        | Address      |
| ------ | ---------------- | ------------ |
| EDGE-1 | GigabitEthernet1 | 10.255.103.1 |
| EDGE-2 | GigabitEthernet1 | 10.255.103.2 |

This link provides the alternate path required for routing resilience.

---

# 11. OSPF Design

OSPF process:

```text
100
```

Area:

```text
Area 0
```

Router IDs:

| Device | Router ID |
| ------ | --------- |
| DIST-1 | 1.1.1.1   |
| DIST-2 | 2.2.2.2   |
| EDGE-1 | 3.3.3.3   |
| EDGE-2 | 4.4.4.4   |

The initial OSPF implementation uses a single Area 0.

Multi-area OSPF is not required at this stage and will only be introduced if justified by the later architecture.

---

# 12. OSPF Passive Interfaces

The production VLANs were initially included in OSPF.

This produced unnecessary OSPF adjacencies over the production VLANs.

The configuration was corrected by making the production SVIs passive:

```cisco
router ospf 100
 passive-interface vlan 10
 passive-interface vlan 20
 passive-interface vlan 30
 passive-interface vlan 40
 passive-interface vlan 50
 passive-interface vlan 60
 passive-interface vlan 99
```

The result is:

```text
Production VLANs
    ↓
Advertised by OSPF
    ↓
No OSPF neighbor formation

VLAN 100
    ↓
Active OSPF transit
    ↓
OSPF adjacency
```

---

# 13. OSPF Networks

The distribution switches advertise:

```text
10.10.10.0/24
10.10.20.0/24
10.10.30.0/24
10.10.40.0/24
10.10.50.0/24
10.10.60.0/24
10.10.99.0/24
10.255.100.0/30
```

The routed edge links advertise:

```text
10.255.101.0/30
10.255.102.0/30
10.255.103.0/30
```

---

# 14. Troubleshooting During Implementation

## OSPF Not Enabled on DIST-1 Ethernet1/2

The first OSPF verification showed:

```text
%OSPF: OSPF not enabled on Ethernet1/2
```

Although the OSPF process contained:

```text
network 10.255.101.0 0.0.0.3 area 0
```

the interface itself had no configuration.

Verification showed:

```text
interface Ethernet1/2
end
```

The root cause was therefore not the OSPF network statement.

The interface had not been converted into a routed Layer 3 interface.

The correction was:

```cisco
interface Ethernet1/2
 description L3-TRANSIT-TO-EDGE-1
 no switchport
 ip address 10.255.101.1 255.255.255.252
 no shutdown
```

After this correction, the DIST-1 ↔ EDGE-1 OSPF adjacency formed successfully.

This was an important troubleshooting example because the OSPF configuration itself appeared correct while the underlying interface configuration was missing.

---

# 15. Final OSPF Adjacency Design

The final OSPF topology contains four adjacencies:

```text
DIST-1 ↔ DIST-2
DIST-1 ↔ EDGE-1
EDGE-1 ↔ EDGE-2
EDGE-2 ↔ DIST-2
```

All reached:

```text
FULL
```

The topology therefore forms a routing ring.

---

# 16. Configuration Persistence

After successful implementation and testing, configurations were saved to startup configuration.

The known-good state should be preserved before proceeding to the next phase.

---

# 17. Phase 2 Configuration Result

The following Layer 3 functions were successfully implemented:

* Layer 3 routing on distribution switches
* Production VLAN SVIs
* HSRP gateway redundancy
* Dedicated OSPF transit VLAN
* Routed distribution-to-edge links
* EDGE-1 ↔ EDGE-2 routed link
* OSPF Area 0
* Explicit OSPF router IDs
* Passive production VLAN interfaces
* OSPF route advertisement
* OSPF route learning
* Redundant routed topology
* End-to-end Layer 3 connectivity
* OSPF failure recovery

**Configuration Gate: COMPLETE**

The Layer 3 configuration is now frozen pending future architectural phases.
