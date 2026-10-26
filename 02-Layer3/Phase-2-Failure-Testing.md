# Phase 2 — Failure Testing and OSPF Convergence

## Enterprise Network Security Architecture and Intrusion Detection in a Virtualized Environment

**Phase:** 2 — Layer 3 / SVI / HSRP / OSPF
**Status:** Complete

---

# 1. Failure Testing Objective

Failure testing was performed to determine whether the new Layer 3/OSPF architecture could maintain connectivity after the loss of a routed link.

The test followed the project methodology:

```text
Normal Operation
      ↓
Baseline
      ↓
Controlled Failure
      ↓
Observe
      ↓
OSPF Convergence
      ↓
Traffic Verification
      ↓
Restore
      ↓
Recovery Verification
```

The objective was not simply to confirm that an interface could be shut down.

The objective was to demonstrate that the redundant enterprise topology provides an alternate forwarding path.

---

# 2. Pre-Failure Topology

Before the failure:

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

The direct DIST-1 ↔ EDGE-1 path was operational.

OSPF adjacency:

```text
DIST-1 ↔ EDGE-1
```

State:

```text
FULL
```

The normal traffic path from EDGE-1 into the campus was therefore:

```text
EDGE-1
   ↓
DIST-1
   ↓
Campus
```

---

# 3. Pre-Failure OSPF Baseline

## DIST-1

```text
Neighbor ID     Pri   State       Address          Interface
3.3.3.3         1     FULL/DR     10.255.101.2    Ethernet1/2
2.2.2.2         1     FULL/DR     10.255.100.2    Vlan100
```

## EDGE-1

```text
Neighbor ID     Pri   State       Address          Interface
4.4.4.4         1     FULL/DR     10.255.103.2    GigabitEthernet1
1.1.1.1         1     FULL/BDR    10.255.101.1    GigabitEthernet2
```

This confirmed that the direct DIST-1 ↔ EDGE-1 OSPF adjacency was healthy before introducing the failure.

---

# 4. Failure Introduced

The DIST-1 ↔ EDGE-1 routed link was deliberately disabled.

On DIST-1:

```cisco
conf t
interface Ethernet1/2
 shutdown
end
```

This removed the direct routed path:

```text
DIST-1 X EDGE-1
```

---

# 5. OSPF Neighbor Behavior After Failure

## DIST-1

After the failure, the EDGE-1 neighbor disappeared.

Remaining adjacency:

```text
2.2.2.2  FULL/DR
10.255.100.2
Vlan100
```

This demonstrated that DIST-1 correctly detected the failed OSPF adjacency while retaining connectivity to DIST-2.

---

## EDGE-1

EDGE-1 lost its DIST-1 adjacency.

Remaining neighbor:

```text
4.4.4.4  FULL/DR
10.255.103.2
GigabitEthernet1
```

EDGE-1 therefore retained its connection to EDGE-2.

---

# 6. OSPF Convergence

OSPF recalculated the available paths after the direct link failure.

The original path:

```text
EDGE-1 → DIST-1
```

was no longer available.

The alternate path became:

```text
EDGE-1
   ↓
EDGE-2
   ↓
DIST-2
   ↓
DIST-1 / Campus
```

---

# 7. DIST-1 Routing Table After Failure

DIST-1 learned the remote networks through DIST-2:

```text
O 10.255.101.0/30 [110/13] via 10.255.100.2
O 10.255.102.0/30 [110/11] via 10.255.100.2
O 10.255.103.0/30 [110/12] via 10.255.100.2
```

This demonstrates that DIST-1 selected VLAN 100 toward DIST-2 as the alternate route.

---

# 8. EDGE-1 Routing Table After Failure

Before failure, campus routes were reached through:

```text
10.255.101.1
```

After failure, the campus routes were learned through EDGE-2:

```text
O 10.10.10.0/24 via 10.255.103.2
O 10.10.20.0/24 via 10.255.103.2
O 10.10.30.0/24 via 10.255.103.2
O 10.10.40.0/24 via 10.255.103.2
O 10.10.50.0/24 via 10.255.103.2
O 10.10.60.0/24 via 10.255.103.2
O 10.10.99.0/24 via 10.255.103.2
```

The next hop changed from DIST-1 to EDGE-2.

This is direct evidence of OSPF convergence.

---

# 9. Traffic Verification During Failure

Connectivity was tested while the DIST-1 ↔ EDGE-1 link remained shutdown.

## EDGE-1 → VLAN 30

```text
ping 10.10.30.1
```

Result:

```text
Success rate is 100 percent (5/5)
```

## EDGE-1 → VLAN 20

```text
ping 10.10.20.1
```

Result:

```text
Success rate is 100 percent (5/5)
```

Therefore, the link failure did not cause loss of end-to-end campus reachability.

---

# 10. Traceroute During Failure

Traceroute from EDGE-1 to VLAN 30 after the failure:

```text
traceroute 10.10.30.1
```

Result:

```text
1  10.255.103.2
2  10.255.102.1
3  10.10.30.2
```

The path was therefore:

```text
EDGE-1
   ↓
10.255.103.2
EDGE-2
   ↓
10.255.102.1
DIST-2
   ↓
10.10.30.2
DIST-1
   ↓
10.10.30.1 / HSRP
```

This proves that traffic was actually traversing the alternate path.

---

# 11. Failure Test Result

| Test                             | Result |
| -------------------------------- | ------ |
| Direct routed link failure       | PASS   |
| OSPF adjacency loss detected     | PASS   |
| Alternate route calculated       | PASS   |
| DIST-1 rerouted through DIST-2   | PASS   |
| EDGE-1 rerouted through EDGE-2   | PASS   |
| Campus routes remained available | PASS   |
| VLAN 30 connectivity             | PASS   |
| VLAN 20 connectivity             | PASS   |
| Alternate traceroute confirmed   | PASS   |

---

# 12. Link Restoration

After the failure test, the routed interface was restored.

On DIST-1:

```cisco
conf t
interface Ethernet1/2
 no shutdown
end
```

The interface returned to operation.

---

# 13. OSPF Recovery

After restoration, the original OSPF adjacency returned.

## DIST-1

```text
3.3.3.3  FULL/DR
10.255.101.2
Ethernet1/2
```

## EDGE-1

```text
1.1.1.1  FULL/BDR
10.255.101.1
GigabitEthernet2
```

The remaining adjacencies also remained operational:

```text
DIST-1 ↔ DIST-2
EDGE-1 ↔ EDGE-2
EDGE-2 ↔ DIST-2
```

The network therefore returned to its normal four-neighbor OSPF topology.

---

# 14. Failure Recovery Summary

The complete sequence was:

```text
NORMAL STATE

EDGE-1 → DIST-1 → Campus

        ↓

DIST-1 ↔ EDGE-1 LINK FAILURE

        ↓

OSPF detects adjacency loss

        ↓

OSPF recalculates paths

        ↓

EDGE-1 → EDGE-2 → DIST-2 → Campus

        ↓

Traffic remains reachable

        ↓

DIST-1 ↔ EDGE-1 LINK RESTORED

        ↓

OSPF adjacency returns FULL

        ↓

NORMAL STATE RESTORED
```

---

# 15. Engineering Interpretation

The failure test demonstrated the value of the four-router ring.

The additional EDGE-1 ↔ EDGE-2 connection is not simply additional connectivity.

It provides a second routing path when one of the direct campus-to-edge connections fails.

The test therefore validates an architectural principle:

> Redundancy is useful only when the routing protocol can detect failure, select an alternate path, and maintain service.

OSPF successfully provided that behavior in the virtualized enterprise environment.

---

# 16. Lessons Learned

### 16.1 OSPF adjacency does not guarantee correct interface configuration

During initial implementation, the OSPF network statement existed but DIST-1 Ethernet1/2 was not configured as a routed interface.

The interface had to be explicitly configured with:

```cisco
no switchport
ip address ...
```

This reinforced the troubleshooting hierarchy:

```text
Interface
   ↓
Layer 2
   ↓
Layer 3
   ↓
Routing
```

---

### 16.2 Route tables are more useful than adjacency status alone

The OSPF neighbors initially showed healthy `FULL` states.

However, route verification was required to prove that actual networks were being exchanged.

The phase therefore verified both:

```text
Neighbor formation
```

and:

```text
Route learning
```

---

### 16.3 Failure testing must include traffic verification

It is not sufficient to demonstrate that an OSPF neighbor disappeared.

The phase also verified:

* route changes
* ping connectivity
* traceroute path changes
* restoration of the original adjacency

This provided evidence that the network remained operational rather than merely demonstrating protocol behavior.

---

# 17. Phase 2 Failure Testing Result

```text
Normal-state baseline       PASS
Controlled failure          PASS
OSPF convergence             PASS
Alternate path selection     PASS
Traffic continuity           PASS
Traceroute verification      PASS
Link restoration             PASS
OSPF recovery                PASS
Final state verification     PASS
```

**Failure Testing Gate: COMPLETE**

---

# 18. Phase 2 Freeze

Phase 2 has successfully completed:

```text
BUILD
  ↓
VERIFY
  ↓
FAILURE TESTING
  ↓
TROUBLESHOOT
  ↓
RECOVERY
  ↓
DOCUMENT
  ↓
FREEZE
```

The Layer 3/HSRP/OSPF implementation is considered a known-good baseline.

No additional Layer 3 configuration is required before proceeding to the next architectural phase.

**Phase 2 — Layer 3 / SVI / HSRP / OSPF: FROZEN**
