 # Phase 3 — EIGRP + OSPF Redistribution: Configuration

**Project:** Enterprise Network Security Architecture and Intrusion Detection in a Virtualized Environment  
**Phase:** 3 — EIGRP + OSPF Redistribution  
**Platform:** EVE-NG  
**Baseline:** Phase 2 — Layer 3 / SVI / HSRP / OSPF / Edge Routing  
**Status:** Implementation completed; recorded test scenarios verified; formal freeze pending final evidence capture.

## 1. Objective

Integrate a legacy EIGRP routing domain into the existing enterprise OSPF core through controlled, bidirectional route redistribution.

The implementation must:

- Advertise approved legacy prefixes into the enterprise OSPF domain.
- Advertise approved enterprise prefixes into the legacy EIGRP domain.
- Filter routes that should not cross the redistribution boundary.
- Apply route tagging to support loop-prevention policy.
- Verify route installation, reachability, route withdrawal, and recovery.
- Preserve the frozen Phase 2 topology and routing configuration.

## 2. Topology

```text
                 Enterprise OSPF Core
                         |
                      DIST-2
                         |
                 10.255.104.0/30
                 D2: 10.255.104.1
                 L1: 10.255.104.2
                         |
                     LEGACY-1
                  Redistribution Point
                    OSPF / EIGRP
                         |
                 10.255.105.0/30
                 L1: 10.255.105.1
                 L2: 10.255.105.2
                         |
                     LEGACY-2
                      EIGRP AS 100
                         |
                  Legacy Loopbacks
                  Lo10: 172.16.10.1/24
                  Lo20: 172.16.20.1/24
                  Lo30: 172.16.30.1/24
```

LEGACY-1 is the redistribution boundary between OSPF process 100 and EIGRP autonomous system 100.

## 3. IP addressing

| Device | Interface | IP address | Purpose |
|---|---|---|---|
| DIST-2 | Et1/3 | 10.255.104.1/30 | OSPF transit |
| LEGACY-1 | Gi2 | 10.255.104.2/30 | OSPF-facing interface |
| LEGACY-1 | Gi1 | 10.255.105.1/30 | EIGRP-facing interface |
| LEGACY-2 | Gi1 | 10.255.105.2/30 | EIGRP-facing interface |
| LEGACY-2 | Lo10 | 172.16.10.1/24 | Legacy Users test network |
| LEGACY-2 | Lo20 | 172.16.20.1/24 | Legacy Servers test network |
| LEGACY-2 | Lo30 | 172.16.30.1/24 | Route-filtering test network |

### Routing parameters

| Parameter | Value |
|---|---|
| Enterprise routing protocol | OSPF |
| OSPF process | 100 |
| OSPF area | 0 |
| LEGACY-1 OSPF router ID | 5.5.5.5 |
| Legacy routing protocol | EIGRP |
| EIGRP AS | 100 |
| Redistribution boundary | LEGACY-1 |

## 4. EIGRP-to-OSPF redistribution

### Policy

The following prefixes are permitted from EIGRP into OSPF:

- `172.16.10.0/24`
- `172.16.20.0/24`

The prefix `172.16.30.0/24` is intentionally filtered to verify that redistribution does not automatically advertise every legacy network.

The permitted routes are tagged with route tag `100`.

### Configuration on LEGACY-1

```cisco
ip prefix-list EIGRP-to-OSPF seq 10 permit 172.16.10.0/24
ip prefix-list EIGRP-to-OSPF seq 20 permit 172.16.20.0/24

route-map EIGRP-to-OSPF permit 10
 match ip address prefix-list EIGRP-to-OSPF
 set tag 100

router ospf 100
 redistribute eigrp 100 route-map EIGRP-to-OSPF
```

### Expected outcome

- `172.16.10.0/24` is redistributed into OSPF.
- `172.16.20.0/24` is redistributed into OSPF.
- Both redistributed routes carry tag `100`.
- `172.16.30.0/24` remains absent from the enterprise OSPF routing table.

## 5. OSPF-to-EIGRP redistribution

### Policy

The following enterprise prefixes are permitted into EIGRP:

- `10.10.10.0/24` — Users
- `10.10.30.0/24` — Servers

The following networks are excluded from the tested redistribution policy:

- `10.10.20.0/24` — Voice
- `10.10.40.0/24` — Security

An EIGRP seed metric is specified because OSPF metrics cannot be directly used as EIGRP composite metrics.

### Configuration on LEGACY-1

```cisco
ip prefix-list OSPF-to-EIGRP seq 10 permit 10.10.10.0/24
ip prefix-list OSPF-to-EIGRP seq 30 permit 10.10.30.0/24

route-map OSPF-to-EIGRP deny 5
 match tag 100

route-map OSPF-to-EIGRP permit 10
 match ip address prefix-list OSPF-to-EIGRP
 set metric 10000 100 255 1 1500

router eigrp 100
 redistribute ospf 100 route-map OSPF-to-EIGRP
```

The configured seed metric is:

| Metric component | Value |
|---|---:|
| Bandwidth | 10000 Kbit/s |
| Delay | 100 tens of microseconds |
| Reliability | 255 |
| Load | 1 |
| MTU | 1500 bytes |

**Route-tag limitation:** route-map sequence 5 is configured to deny routes tagged `100`. However, an isolated live test proving that a route tagged `100` is rejected by this specific sequence has not yet been recorded. The configuration is present, but this particular behavior must not be described as independently verified.

## 6. Phase 2 baseline protection

Phase 3 extends the existing topology. The following Phase 2 links must remain unchanged:

| Link | Network |
|---|---|
| DIST-1 ↔ DIST-2 | 10.255.100.0/30 |
| DIST-1 ↔ EDGE-1 | 10.255.101.0/30 |
| DIST-2 ↔ EDGE-2 | 10.255.102.0/30 |
| EDGE-1 ↔ EDGE-2 | 10.255.103.0/30 |

The new Phase 3 links are:

- DIST-2 Et1/3 ↔ LEGACY-1 Gi2: `10.255.104.0/30`
- LEGACY-1 Gi1 ↔ LEGACY-2 Gi1: `10.255.105.0/30`

Do not modify DIST-2 Et1/2, which belongs to the existing Phase 2 connection to EDGE-2.

## 7. Configuration and verification commands

Capture actual device outputs before the formal phase freeze.

### LEGACY-1

```cisco
show ip interface brief
show ip ospf neighbor
show ip eigrp neighbors
show ip route eigrp
show ip route ospf
show ip prefix-list
show route-map
show running-config | section router ospf
show running-config | section router eigrp
show startup-config | section router ospf
show startup-config | section router eigrp
```

### DIST-2

```cisco
show ip ospf neighbor
show ip route 172.16.10.0
show ip route 172.16.20.0
show ip route 172.16.30.0
show ip ospf database external
show running-config | section router ospf
show startup-config | section router ospf
```

### LEGACY-2

```cisco
show ip eigrp neighbors
show ip route eigrp
show ip route 10.10.10.0
show ip route 10.10.20.0
show ip route 10.10.30.0
show ip route 10.10.40.0
show ip route 172.16.30.0
show running-config | section router eigrp
show startup-config | section router eigrp
```

## 8. Operational lessons

- Restore redistribution using the complete route-map-based statement.
- During testing, the command form `no redistribute eigrp 100 route-map ...` did not safely produce the intended withdrawal on the IOS image in use and risked leaving a redistribution statement without its route-map.
- The tested withdrawal method was to remove the redistribution statement under the relevant routing process, then restore the exact intended statement.
- Verify running and startup configurations after changes.
- Preserve actual terminal output, errors, and screenshots as evidence rather than reconstructing output from memory.

## 9. Full configuration evidence

Full running/startup configurations for DIST-2, LEGACY-1, and LEGACY-2 will be captured and added separately. The snippets in this document describe the relevant redistribution policy and are not a substitute for the complete device configurations.
