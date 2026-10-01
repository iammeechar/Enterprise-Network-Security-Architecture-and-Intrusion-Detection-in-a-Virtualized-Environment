# Phase 1 — Layer 2 Failure Testing

**Project:** Enterprise Network Security Architecture and Intrusion Detection in a Virtualized Environment
**Phase:** 1A — Enterprise Layer 2 Foundation
**Failure Testing Status:** COMPLETE
**Baseline:** Phase-1A-L2-Baseline-v1

---

# 1. Purpose

Failure testing was performed to determine whether the Layer 2 architecture behaves correctly when expected faults are introduced.

Each test followed:

```text
BEFORE
  ↓
FAILURE
  ↓
SYMPTOM
  ↓
INVESTIGATION
  ↓
FIX / RECOVERY
  ↓
AFTER
```

The tests covered:

1. STP path failure
2. VLAN removal from a trunk
3. Incorrect access VLAN
4. LACP member failure
5. STP root change

---

# 2. Test 1 — Distribution-to-Access Uplink Failure

## Objective

Verify that Rapid PVST+ can move an access switch to its redundant uplink when the active path fails.

## Baseline

ACC-1:

```text
Et0/0  Root Forwarding
Et0/1  Alternate Blocking
```

## Failure

On DIST-1:

```cisco
configure terminal
interface ethernet 0/2
 shutdown
end
```

## Observed Result

ACC-1 changed its root path:

Before:

```text
Et0/0  Root Forwarding
Et0/1  Alternate Blocking
```

During failure:

```text
Et0/0  Designated Learning
Et0/1  Root Forwarding
```

The root path cost changed from:

```text
100
```

to:

```text
156
```

This reflects the longer path through DIST-2 and the inter-distribution Port-Channel.

VPCS-1 to VPCS-2 connectivity remained available through the alternate path after convergence.

## Recovery

```cisco
configure terminal
interface ethernet 0/2
 no shutdown
end
```

The topology returned to:

```text
Et0/0  Root Forwarding
Et0/1  Alternate Blocking
```

## Result

**PASS**

### Lesson

STP was not simply preventing loops; it was also providing a usable recovery path when the preferred path failed.

---

# 3. Test 2 — VLAN Removed from Trunk

## Objective

Determine whether VLAN 10 connectivity fails when it is removed from the active trunk and whether the redundant topology can provide an alternative path.

## Failure

On DIST-1:

```cisco
configure terminal
interface ethernet 0/2
 switchport trunk allowed vlan remove 10
end
```

## Observed State

The physical interface remained:

```text
up/up
```

However, VLAN 10 was no longer allowed on Et0/2.

`show interfaces trunk` showed:

```text
Et0/2 allowed:
20,30,40,50,60,99
```

while other paths continued to allow VLAN 10.

## Investigation

ACC-1 STP showed VLAN 10 using the alternate path.

The physical link remained operational, but VLAN 10 was no longer available across that specific trunk.

## Connectivity

VPCS-1 to VPCS-2:

```text
5/5 successful
```

Connectivity remained available through:

```text
ACC-1
   ↓
DIST-2
   ↓
Po1
   ↓
DIST-1
   ↓
ACC-2
```

## Recovery

VLAN 10 was restored to the DIST-1/ACC-1 trunk.

Final verification confirmed VLAN 10 was again allowed.

## Result

**PASS**

### Lesson

A link being `up/up` does not guarantee that a particular VLAN can traverse it.

Layer 2 troubleshooting must therefore distinguish:

```text
Physical state
      ↓
Trunk state
      ↓
Allowed VLANs
      ↓
STP state
      ↓
Forwarding
```

---

# 4. Test 3 — Incorrect Access VLAN

## Objective

Demonstrate the effect of assigning an endpoint access port to the wrong VLAN.

## Failure

On ACC-1:

```cisco
configure terminal
interface ethernet 0/2
 switchport access vlan 20
end
```

## Observed State

`show vlan brief` showed:

```text
20  VOICE  active  Et0/2
```

VLAN 10 no longer contained the endpoint port.

VPCS-1 remained configured as:

```text
10.10.10.101/24
```

while VPCS-2 remained:

```text
10.10.10.102/24
```

## Connectivity Test

Ping from VPCS-1 to VPCS-2 failed.

This was expected because VPCS-1 had effectively been moved into VLAN 20 while retaining an address from the VLAN 10 subnet.

## Recovery

```cisco
configure terminal
interface ethernet 0/2
 switchport access vlan 10
end
```

Connectivity was restored:

```text
5/5 successful
```

## Additional Command-Entry Error

During testing:

```text
VPCS> VPCS> ping 10.10.10.102
```

produced:

```text
Bad command
```

This was a command-entry mistake rather than a network failure.

## Result

**PASS**

### Lesson

An endpoint's IP address does not determine its Layer 2 VLAN.

The switchport configuration determines the endpoint's broadcast domain.

---

# 5. Test 4 — LACP Member Failure

## Objective

Verify that Port-Channel 1 remains operational when one physical LACP member fails.

## Baseline

```text
Po1(SU) LACP
Et0/0(P)
Et0/1(P)
```

Both physical links were bundled.

## Failure

On DIST-1:

```cisco
configure terminal
interface ethernet 0/0
 shutdown
end
```

## Observed State

The EtherChannel changed to:

```text
Po1(SU) LACP
Et0/0(D)
Et0/1(P)
```

Interpretation:

```text
Et0/0 = Down
Et0/1 = Still bundled
Po1 = Still operational
```

## Connectivity

VPCS-1 to VPCS-2 remained operational.

No STP failover was required because the logical Port-Channel itself remained available.

## Recovery

```cisco
configure terminal
interface ethernet 0/0
 no shutdown
end
```

Final state:

```text
Po1(SU) LACP
Et0/0(P)
Et0/1(P)
```

## Result

**PASS**

### Lesson

EtherChannel provides link-level redundancy inside a logical Layer 2 interface.

This is different from STP redundancy.

```text
EtherChannel:
Multiple physical links
        ↓
One logical interface
```

versus:

```text
STP:
Multiple logical paths
        ↓
One forwarding path
        +
One standby path
```

---

# 6. Test 5 — STP Root Change

## Objective

Deliberately change the STP root to confirm that the topology responds to a change in bridge priority.

## Baseline

DIST-1:

```text
Root ID priority 24586
This bridge is the root
```

DIST-2:

```text
Root ID priority 24586
Root = DIST-1
```

---

## Failure / Change

On DIST-2:

```cisco
configure terminal
spanning-tree vlan 10 root primary
end
```

DIST-2 then became the root.

Observed:

```text
Root ID priority 20490
Address aabb.cc00.2000
This bridge is the root
```

The priority calculation reflected the VLAN system ID extension:

```text
20480 + VLAN 10
= 20490
```

The previous root priority was:

```text
24576 + VLAN 10
= 24586
```

Lower bridge priority wins the STP root election.

---

# 7. Resulting Topology During Test

DIST-2 became the root.

DIST-1 changed to:

```text
Po1 Root Forwarding
```

ACC-1 changed to:

```text
Et0/0 Alternate Blocking
Et0/1 Root Forwarding
Et0/2 Designated Forwarding
```

ACC-2 followed the same logical change.

This demonstrated that the access switches selected the new root path through DIST-2.

---

# 8. Connectivity During Root Change

VPCS connectivity remained operational.

Observed test:

```text
5/5 successful
```

Example successful latency values included approximately:

```text
1.882 ms
4.238 ms
5.559 ms
10.317 ms
11.152 ms
```

The exact latency is not treated as a performance benchmark; the relevant result is successful end-to-end connectivity.

---

# 9. Root Restoration

DIST-2 was returned to its intended secondary role:

```cisco
configure terminal
spanning-tree vlan 10 root secondary
end
```

Final topology returned to:

```text
DIST-1
  STP ROOT

DIST-2
  STP SECONDARY

ACC-1
  Et0/0 Root Forwarding
  Et0/1 Alternate Blocking

ACC-2
  Et0/0 Root Forwarding
  Et0/1 Alternate Blocking
```

DIST-1 again showed:

```text
This bridge is the root
```

---

# 10. Failure-Test Summary

| Test | Failure                        | Expected Behavior                    | Result |
| ---- | ------------------------------ | ------------------------------------ | ------ |
| 1    | DIST-1 → ACC-1 uplink shutdown | STP uses alternate path              | PASS   |
| 2    | VLAN 10 removed from trunk     | VLAN uses available alternate path   | PASS   |
| 3    | Endpoint moved to VLAN 20      | Endpoint loses VLAN 10 connectivity  | PASS   |
| 4    | LACP member shutdown           | Port-Channel remains operational     | PASS   |
| 5    | STP root changed               | Topology reconverges around new root | PASS   |

---

# 11. Troubleshooting Hierarchy Demonstrated

The tests reinforced the intended troubleshooting sequence:

```text
Interface / Physical
        ↓
VLAN / Layer 2
        ↓
STP / Path Selection
        ↓
EtherChannel / LACP
        ↓
Endpoint Connectivity
```

Examples:

### Physical/link problem

Test 1 and Test 4 required checking interface and bundle state.

### VLAN problem

Test 2 required checking allowed VLANs.

### Access VLAN problem

Test 3 required checking the endpoint switchport VLAN assignment.

### STP problem

Tests 1 and 5 required checking:

```cisco
show spanning-tree vlan 10
```

---

# 12. Failure Testing Conclusion

All five planned Phase 1A failure tests passed.

The Layer 2 architecture demonstrated:

* Redundant forwarding paths
* STP loop prevention
* STP failover
* LACP member resilience
* VLAN isolation
* Trunk/VLAN troubleshooting
* Root-election behavior
* End-to-end connectivity preservation where expected

**Failure Testing Gate: COMPLETE**
