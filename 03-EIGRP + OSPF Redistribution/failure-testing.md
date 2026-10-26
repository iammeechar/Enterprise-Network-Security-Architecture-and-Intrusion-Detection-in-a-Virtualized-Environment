 # Phase 3 — EIGRP + OSPF Redistribution: Failure Testing

**Project:** Enterprise Network Security Architecture and Intrusion Detection in a Virtualized Environment  
**Phase:** 3 — EIGRP + OSPF Redistribution  
**Platform:** EVE-NG  
**Status:** Recorded tests completed; final evidence review pending.

## 1. Objectives

Failure testing validates that the redistribution boundary behaves predictably under normal operation, policy changes, and link failures.

The tests focus on:

- EIGRP and OSPF adjacency establishment.
- Prefix filtering in both redistribution directions.
- Redistribution withdrawal and restoration.
- EIGRP link failure and recovery.
- Bidirectional reachability using explicit source addresses.
- Preservation of the Phase 2 enterprise routing baseline.

## 2. Test results

| ID | Test | Expected result | Recorded result |
|---|---|---|---|
| FT-01 | EIGRP adjacency | LEGACY-1 and LEGACY-2 establish a neighbor relationship | **PASS** — neighbor `10.255.105.2` observed on LEGACY-1 Gi1 |
| FT-02 | OSPF adjacency | DIST-2 and LEGACY-1 reach FULL state | **PASS** — neighbor RID `5.5.5.5` at `10.255.104.2` reached FULL |
| FT-03 | EIGRP-to-OSPF permitted routes | `172.16.10.0/24` and `172.16.20.0/24` appear in OSPF | **PASS** — both installed as OSPF external type 2 routes with tag 100 |
| FT-04 | EIGRP-to-OSPF filtered route | `172.16.30.0/24` remains absent from DIST-2 | **PASS** — route lookup returned `% Subnet not in table` |
| FT-05 | OSPF-to-EIGRP permitted routes | `10.10.10.0/24` and `10.10.30.0/24` appear in EIGRP | **PASS** — both routes present |
| FT-06 | OSPF-to-EIGRP filtered routes | `10.10.20.0/24` and `10.10.40.0/24` remain absent from LEGACY-2 | **PASS** — both route lookups returned `% Subnet not in table` |
| FT-07 | Withdraw EIGRP redistribution | Legacy external routes disappear from OSPF | **PASS** — both permitted legacy routes disappeared from DIST-2 |
| FT-08 | Restore EIGRP redistribution | Permitted legacy routes return with tag 100 | **PASS** — both routes returned |
| FT-09 | Withdraw OSPF redistribution | Permitted enterprise routes disappear from EIGRP | **PASS** — `10.10.10.0/24` and `10.10.30.0/24` disappeared from LEGACY-2 |
| FT-10 | Restore OSPF redistribution | Permitted enterprise routes return to EIGRP | **PASS** — both routes returned |
| FT-11 | EIGRP link failure | Neighbor and learned legacy routes are withdrawn | **PASS** — adjacency and routes disappeared; DIST-2 lost the redistributed legacy routes |
| FT-12 | EIGRP link recovery | Neighbor and routes return | **PASS** — adjacency, three legacy routes, and OSPF external routes recovered |
| FT-13 | Source-specific connectivity | Approved destinations remain reachable in both directions | **PASS** — all four recorded source-specific ping tests returned 5/5 |
| FT-14 | Route-tag deny-sequence test | A route tagged 100 is demonstrably rejected by route-map sequence 5 | **NOT PROVEN** — configuration exists, but this isolated behavior was not demonstrated |

## 3. Bidirectional connectivity results

The following results were recorded during the phase:

| Source device | Destination | Source address | Result |
|---|---|---|---|
| DIST-2 | `172.16.10.1` | `10.10.10.3` | 5/5 replies |
| DIST-2 | `172.16.20.1` | `10.10.10.3` | 5/5 replies |
| LEGACY-2 | `10.10.10.3` | `172.16.10.1` | 5/5 replies |
| LEGACY-2 | `10.10.30.3` | `172.16.10.1` | 5/5 replies |

Earlier baseline tests also included successful pings to HSRP virtual IPs. The table above records the latest specific post-recovery host-address tests; it should not be interpreted as a fresh post-recovery test of every HSRP virtual IP.

## 4. Test procedure: EIGRP link failure

**Purpose:** Validate neighbor loss, route withdrawal, and recovery when the legacy routing link fails.

### Before the failure

Verify the following:

- LEGACY-1 and LEGACY-2 have an EIGRP adjacency.
- LEGACY-1 learns the three legacy loopback routes.
- DIST-2 has the redistributed routes `172.16.10.0/24` and `172.16.20.0/24`.

### Failure injection

On LEGACY-1:

```cisco
configure terminal
interface GigabitEthernet1
 shutdown
end
```

### Expected observations

- The EIGRP adjacency between LEGACY-1 and LEGACY-2 is lost.
- LEGACY-1 no longer learns the three loopback routes from LEGACY-2.
- DIST-2 withdraws the redistributed routes `172.16.10.0/24` and `172.16.20.0/24`.

**Recorded outcome:** These withdrawals were observed.

### Recovery

On LEGACY-1:

```cisco
configure terminal
interface GigabitEthernet1
 no shutdown
end
```

### Recovery verification

Confirm that:

- The EIGRP adjacency is re-established.
- All three legacy loopback routes return to LEGACY-1.
- The permitted OSPF external routes return to DIST-2.
- Source-specific connectivity is restored.

**Recorded outcome:** The neighbor, three legacy routes, and OSPF external routes returned. The recorded source-specific ping tests succeeded.

## 5. Test procedure: EIGRP-to-OSPF redistribution withdrawal

**Purpose:** Confirm that removing redistribution withdraws the affected routes from the enterprise routing table.

On LEGACY-1, under OSPF process 100:

```cisco
configure terminal
router ospf 100
 no redistribute eigrp 100
end
```

Verify on DIST-2:

```cisco
show ip route 172.16.10.0
show ip route 172.16.20.0
```

**Expected:** Both routes are absent.

Restore the intended policy on LEGACY-1:

```cisco
configure terminal
router ospf 100
 redistribute eigrp 100 route-map EIGRP-to-OSPF
end
```

Repeat the route lookups and confirm that both routes return with tag 100.

**Recorded outcome:** Withdrawal and restoration succeeded.

## 6. Test procedure: OSPF-to-EIGRP redistribution withdrawal

**Purpose:** Confirm that removing OSPF redistribution withdraws the enterprise routes from the legacy routing domain.

On LEGACY-1, under EIGRP AS 100:

```cisco
configure terminal
router eigrp 100
 no redistribute ospf 100
end
```

Verify on LEGACY-2:

```cisco
show ip route 10.10.10.0
show ip route 10.10.30.0
```

**Expected:** Both routes are absent.

Restore the policy:

```cisco
configure terminal
router eigrp 100
 redistribute ospf 100 route-map OSPF-to-EIGRP
end
```

Repeat the route lookups and confirm that both permitted enterprise routes return.

**Recorded outcome:** Withdrawal and restoration succeeded.

## 7. Troubleshooting observations

### Default-source ping failures

Initial ping attempts using the default source address failed, while tests using explicit source addresses succeeded.

The actual default source address was not recorded. Consequently, the precise cause remains unconfirmed.

A source-dependent return path or interaction with prefix filtering is a possible explanation, but it must not be documented as a proven root cause without further evidence.

Recommended follow-up:

1. Record the source address selected by the router.
2. Check the route to the destination.
3. Check the destination-side route back to the source subnet.
4. Verify the prefix-list and route-map policy.
5. Repeat the test with explicit source addresses.
6. Compare the results and preserve the relevant command outputs.

### Route-map tag deny sequence

The following policy is configured:

```cisco
route-map OSPF-to-EIGRP deny 5
 match tag 100
```

However, the recorded tests did not independently demonstrate that a route carrying tag 100 was rejected by this exact sequence.

If claiming proven loop-prevention behavior, perform and document an isolated test showing the tagged route, the policy match, and the resulting absence of that route from EIGRP.

## 8. Lessons learned

- Test route propagation in both directions; successful adjacency alone does not prove redistribution is correct.
- Validate permitted and filtered prefixes independently.
- Confirm withdrawal as well as restoration.
- Use explicit source addresses for meaningful reachability tests across routing domains.
- Record the exact IOS commands used during testing. Different command forms can have unintended effects on redistribution policy.
- Separate observed results from suspected explanations.
- Preserve raw command outputs and screenshots as evidence.

## 9. Final test status

The recorded adjacency, prefix filtering, redistribution withdrawal/restoration, EIGRP link failure/recovery, and source-specific reachability tests passed.

The tag-based deny sequence still requires isolated live proof if it is to be claimed as a demonstrated loop-prevention test. Formal phase freeze also requires final evidence capture and review.
