 # Phase 3 — EIGRP + OSPF Redistribution: Verification

**Project:** Enterprise Network Security Architecture and Intrusion Detection in a Virtualized Environment  
**Phase:** 3 — EIGRP + OSPF Redistribution  
**Baseline:** Phase 2 — Layer 3 / SVI / HSRP / OSPF / Edge Routing  
**Platform:** EVE-NG  
**Status:** Implementation and recorded scenarios verified; formal freeze pending evidence review.

## 1. Purpose

This document records the verification of the Phase 3 implementation against the project objectives.

The verification covers:

- Routing adjacency.
- Bidirectional redistribution.
- Prefix filtering.
- Route tagging.
- End-to-end reachability.
- Redistribution withdrawal and restoration.
- Link failure and recovery.
- Running/startup configuration consistency.
- Remaining requirements before formal freeze.

## 2. Adjacency verification

### EIGRP

The EIGRP adjacency between LEGACY-1 and LEGACY-2 was established over `10.255.105.0/30`.

Recorded observations:

- LEGACY-1 identified neighbor `10.255.105.2` on Gi1.
- LEGACY-2 Gi1 used `10.255.105.2` and was up/up after recovery.
- The three legacy loopback routes were learned by LEGACY-1 when the adjacency was operational.

**Result: PASS**

### OSPF

The OSPF adjacency between DIST-2 and LEGACY-1 reached FULL state in OSPF process 100, Area 0.

Recorded observations:

- Neighbor router ID: `5.5.5.5`.
- Neighbor address: `10.255.104.2`.
- DIST-2 retained its existing Phase 2 OSPF adjacency to DIST-1, router ID `1.1.1.1`.

**Result: PASS**

## 3. EIGRP-to-OSPF route verification

The route policy permits `172.16.10.0/24` and `172.16.20.0/24`, while filtering `172.16.30.0/24`.

| Prefix | Expected behavior | Recorded verification |
|---|---|---|
| `172.16.10.0/24` | Redistributed into OSPF | Present on DIST-2 as an OSPF external type 2 route |
| `172.16.20.0/24` | Redistributed into OSPF | Present on DIST-2 as an OSPF external type 2 route |
| `172.16.30.0/24` | Filtered | Absent from DIST-2 routing table |

Recorded route attributes for the two permitted prefixes:

- OSPF process: 100.
- External route type: E2.
- Administrative distance: 110.
- Metric: 20.
- Route tag: 100.
- Next hop: `10.255.104.2`.
- Advertising ASBR router ID: `5.5.5.5`.

The lookup for `172.16.30.0/24` returned:

```text
% Subnet not in table
```

**Result: PASS**

## 4. OSPF-to-EIGRP route verification

The route policy permits `10.10.10.0/24` and `10.10.30.0/24`, while filtering `10.10.20.0/24` and `10.10.40.0/24`.

| Prefix | Expected behavior | Recorded verification |
|---|---|---|
| `10.10.10.0/24` | Redistributed into EIGRP | Present on LEGACY-2 |
| `10.10.30.0/24` | Redistributed into EIGRP | Present on LEGACY-2 |
| `10.10.20.0/24` | Filtered | Absent from LEGACY-2 routing table |
| `10.10.40.0/24` | Filtered | Absent from LEGACY-2 routing table |

The lookups for `10.10.20.0/24` and `10.10.40.0/24` returned:

```text
% Subnet not in table
```

The observed EIGRP route metric on LEGACY-2 was `321536`, with:

| Attribute | Recorded value |
|---|---|
| Administrative distance | 170 |
| Next hop | 10.255.105.1 |
| Bandwidth | 10000 Kbit/s |
| Delay | 2560 μs |
| Reliability | 100/255 |
| Load | 1/255 |
| MTU | 1500 |

**Result: PASS**

## 5. Reachability verification

The following source-specific tests were recorded:

| Source device | Destination | Source address | Result |
|---|---|---|---|
| DIST-2 | `172.16.10.1` | `10.10.10.3` | 5/5 replies |
| DIST-2 | `172.16.20.1` | `10.10.10.3` | 5/5 replies |
| LEGACY-2 | `10.10.10.3` | `172.16.10.1` | 5/5 replies |
| LEGACY-2 | `10.10.30.3` | `172.16.10.1` | 5/5 replies |

These results demonstrate reachability for the tested source/destination combinations after the relevant routes were restored.

Earlier baseline testing also included successful pings to HSRP virtual IPs. The results above describe the recorded host-address tests and should not be used to imply that every destination or HSRP VIP was retested after every failure event.

**Result: PASS for the listed tests**

## 6. Redistribution withdrawal and recovery

### EIGRP-to-OSPF

Removing the EIGRP redistribution statement under OSPF caused the permitted legacy routes to disappear from DIST-2.

Restoring the statement with the `EIGRP-to-OSPF` route-map caused the permitted routes to return with tag 100.

**Result: PASS**

### OSPF-to-EIGRP

Removing OSPF redistribution under EIGRP caused `10.10.10.0/24` and `10.10.30.0/24` to disappear from LEGACY-2.

Restoring the `OSPF-to-EIGRP` route-map statement caused both routes to return.

**Result: PASS**

### EIGRP link failure and recovery

Shutting down LEGACY-1 Gi1 caused the EIGRP neighbor and learned legacy routes to disappear. DIST-2 also lost the redistributed legacy routes.

Restoring the interface recovered the EIGRP adjacency, the three legacy loopback routes, and the permitted OSPF external routes. The recorded source-specific connectivity tests succeeded after recovery.

**Result: PASS**

## 7. Running/startup configuration verification

The relevant routing sections of the running and startup configurations were compared.

### LEGACY-1

The relevant EIGRP sections matched and included:

```cisco
network 10.255.105.0 0.0.0.3
redistribute ospf 100 route-map OSPF-to-EIGRP
```

The relevant OSPF sections matched and included:

```cisco
router-id 5.5.5.5
redistribute eigrp 100 route-map EIGRP-to-OSPF
network 10.255.104.0 0.0.0.3 area 0
```

### DIST-2

The relevant OSPF running and startup sections matched, including:

```cisco
network 10.255.104.0 0.0.0.3 area 0
```

The frozen Phase 2 OSPF networks were also present.

### LEGACY-2

The relevant EIGRP running and startup sections matched, including the `10.255.105.0/30` transit and the three legacy loopback networks.

Recorded interface state after recovery:

- Gi1 — `10.255.105.2`, up/up.
- Gi2–Gi4 — administratively down.
- Lo10 — `172.16.10.1`, up/up.
- Lo20 — `172.16.20.1`, up/up.
- Lo30 — `172.16.30.1`, up/up.

**Result: PASS for the reviewed routing sections**

These statements summarize the recorded comparisons. Attach the full configuration outputs to the repository before formal freeze.

## 8. Known limitations

### Default-source ping failures

Initial default-source pings failed while source-specific pings succeeded. The selected default source address was not recorded, so the exact cause remains unconfirmed.

Do not describe a return-route or prefix-filter issue as the proven root cause until the source address and forward/return routing decisions have been captured.

### Tag-based loop-prevention test

The `OSPF-to-EIGRP` route-map contains a deny sequence matching tag 100. The configuration is present, but the recorded tests did not isolate and prove that a tagged route was rejected by that sequence.

This behavior remains **configured but not independently proven**.

## 9. Remaining work before formal freeze

- [ ] Capture complete running configurations for DIST-2, LEGACY-1, and LEGACY-2.
- [ ] Capture complete startup configurations for DIST-2, LEGACY-1, and LEGACY-2.
- [ ] Capture raw outputs for OSPF/EIGRP neighbors, routing tables, prefix lists, route maps, and reachability tests.
- [ ] Add screenshots or raw terminal evidence to the Phase 3 GitHub folder.
- [ ] If claiming demonstrated tag-based loop prevention, perform and document an isolated route-tag rejection test.
- [ ] Review all three Phase 3 documents against the captured outputs.
- [ ] Confirm no Phase 2 baseline configuration was changed unintentionally.
- [ ] Commit the reviewed documents and evidence to GitHub.
- [ ] Declare the phase frozen after the evidence review is complete.

## 10. Final verification decision

**Current status: IMPLEMENTATION AND RECORDED TESTS VERIFIED — FORMAL FREEZE PENDING.**

The recorded adjacency, route filtering, redistribution withdrawal/restoration, link failure/recovery, source-specific reachability, and relevant running/startup configuration comparisons succeeded.

Formal freeze remains pending final configuration and terminal evidence capture, review, and resolution or explicit acceptance of the route-tag test limitation.
