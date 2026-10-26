 # Phase 4 — Verification Record

## 1. Purpose

This document records the checks performed after implementing dual-ISP eBGP, route filtering, provider preference, and conditional OSPF default origination.

**Result:** Core functionality verified. Failure behavior tested. All four router configurations saved to startup configuration.

## 2. BGP adjacency verification

### Expected state

All four enterprise-to-provider eBGP sessions should be established:

| Local router | Neighbor | Remote AS | Expected |
|---|---|---:|---|
| EDGE-1 | `10.255.110.2` | 65001 | Established |
| EDGE-1 | `10.255.112.2` | 65002 | Established |
| EDGE-2 | `10.255.111.2` | 65001 | Established |
| EDGE-2 | `10.255.113.2` | 65002 | Established |
| ISP-1 | `10.255.110.1` | 65010 | Established |
| ISP-1 | `10.255.111.1` | 65010 | Established |
| ISP-2 | `10.255.112.1` | 65010 | Established |
| ISP-2 | `10.255.113.1` | 65010 | Established |

Command:

```cisco
show ip bgp summary
```

**Observed:** All four eBGP sessions were established. Under the normal healthy state, each enterprise edge received two prefixes from each ISP: that ISP's test prefix and its default route.

The enterprise edges therefore had four received prefix entries across the two providers, subject to BGP best-path selection.

## 3. Enterprise prefix verification

### EDGE-1

Command:

```cisco
show ip route 192.0.2.0
```

**Observed:** `192.0.2.0/24` was connected through Loopback0, with the local endpoint `192.0.2.1/32`.

### EDGE-2

Command:

```cisco
show ip route 192.0.2.0
```

**Observed:** `192.0.2.0/24` was learned as an intra-area OSPF route through `10.255.103.1`, with metric 2.

This confirmed that EDGE-2 had the enterprise prefix through the existing OSPF domain and could originate it through its BGP `network` statement.

## 4. Prefix advertisement verification

### Enterprise advertisement to providers

Command on each enterprise edge:

```cisco
show ip bgp neighbors <peer-ip> advertised-routes
```

**Observed:** All four enterprise-to-provider advertisement checks showed only:

- `192.0.2.0/24`

No campus VLAN or legacy network prefixes were advertised to the providers.

### ISP-1 advertisement

Checked toward both EDGE-1 and EDGE-2.

**Observed advertised prefixes:**

- `0.0.0.0/0`
- `198.51.100.0/24`

### ISP-2 advertisement

Checked toward both EDGE-1 and EDGE-2.

**Observed advertised prefixes:**

- `0.0.0.0/0`
- `203.0.113.0/24`

The filters therefore limited each provider to its own test prefix plus the default route.

## 5. Provider isolation verification

The providers were checked for unintended route exchange.

**Observed:**

- ISP-1 did not learn `203.0.113.0/24`.
- ISP-2 did not learn `198.51.100.0/24`.
- Both providers learned `192.0.2.0/24` from the enterprise edges.

This is the expected result of the provider outbound prefix lists and the enterprise-only inbound filters.

## 6. Local-preference verification

The enterprise edges were checked for provider path preference.

**Expected:**

| Prefix learned from | Local preference |
|---|---:|
| ISP-1 | 200 |
| ISP-2 | 100 |

**Observed:** Both enterprise edges preferred ISP-1's default route and ISP-1's test-prefix path when the relevant routes were available. ISP-2 remained an eligible backup path.

This confirms the configured preference policy under the tested conditions. Local preference applies within the enterprise AS; it does not instruct the external providers how to route their own traffic.

## 7. OSPF verification

Commands:

```cisco
show ip ospf neighbor
show ip route 0.0.0.0
show running-config | section router ospf
```

### EDGE-1

Observed OSPF neighbors:

- EDGE-2, router ID `4.4.4.4`, FULL on Gi1 at `10.255.103.2`.
- DIST-1, router ID `1.1.1.1`, FULL on Gi2 at `10.255.101.1`.

EDGE-1's OSPF configuration retained the original enterprise network statements and included:

```cisco
default-information originate
```

The command was configured without `always`.

### EDGE-2

Observed OSPF neighbors:

- EDGE-1, router ID `3.3.3.3`, FULL on Gi1 at `10.255.103.1`.
- DIST-2, router ID `2.2.2.2`, FULL on Gi2 at `10.255.102.1`.

### Distribution switches

DIST-1 and DIST-2 learned the OSPF default from the enterprise edge domain. The existing legacy routes `172.16.10.0/24` and `172.16.20.0/24` remained present as OSPF external type 2 routes with tag 100.

No BGP-to-OSPF redistribution was introduced.

## 8. Data-plane reachability

Five-packet ping tests were performed from both enterprise edge routers.

| Source | Destination | Result |
|---|---|---|
| EDGE-1 | `198.51.100.1` | 5/5 successful |
| EDGE-1 | `203.0.113.1` | 5/5 successful |
| EDGE-2 | `198.51.100.1` | 5/5 successful |
| EDGE-2 | `203.0.113.1` | 5/5 successful |

These results confirm end-to-end reachability to both provider loopback test endpoints in the healthy state.

They do not prove access to the real Internet; the ISP routers are simulated.

## 9. Default-route withdrawal and recovery

The default route was removed and restored on the simulated providers.

### ISP-1 default removed

**Observed:**

- ISP-1 no longer had its default network available for BGP origination.
- Both enterprise edges selected ISP-2's default, with local preference 100.
- DIST-1 and DIST-2 retained the OSPF default while EDGE-1 still had a valid default through ISP-2.

### ISP-1 default restored

**Observed:**

- Both enterprise edges preferred ISP-1 again, with local preference 200.
- The internal OSPF default remained available.

### Both provider defaults removed

**Observed:**

- EDGE-1 had no available default path.
- EDGE-2 had no default route in its routing table.
- DIST-1 and DIST-2 withdrew their OSPF default route.

### Both provider defaults restored

**Observed:**

- Both enterprise edges relearned the provider defaults.
- ISP-1 again became the preferred default path.
- The distribution switches relearned the OSPF default.

This confirmed that the OSPF default advertisement is conditional and does not persist when EDGE-1 loses its default route.

## 10. Single-link failure verification

EDGE-1's Gi3 connection to ISP-1 was shut down and later restored.

### During the failure

**Observed:**

- Gi3 was administratively down/down.
- The eBGP peer `10.255.110.2` moved to Idle.
- The ISP-2 peer `10.255.112.2` remained established.
- EDGE-1 selected ISP-2's default route.
- `198.51.100.0/24` disappeared from EDGE-1's routing table.
- Ping to `198.51.100.1` failed, returning `U.U.U` with 0/5 successful packets.

This is an expected limitation of the synthetic design: ISP-2 does not advertise ISP-1's specific test prefix. A backup default does not automatically provide a specific route to a destination prefix that only exists behind the failed provider.

### After recovery

**Observed:**

- Gi3 returned to up/up.
- Both EDGE-1 eBGP sessions were established again.
- Both providers again supplied two prefixes.
- ISP-1's default route became preferred.
- `198.51.100.0/24` returned to the routing table.
- Ping to `198.51.100.1` succeeded with 5/5 packets.

ISP-1 logged a brief BGP connection-collision notification during recovery, followed by the neighbor returning to the Up state. The event was transient and the session recovered.

## 11. Configuration persistence

After testing, the running configuration was saved on:

- EDGE-1
- EDGE-2
- ISP-1
- ISP-2

Command used:

```cisco
copy running-config startup-config
```

The save operation was confirmed on all four routers.

## 12. Final assessment

| Verification area | Result |
|---|---|
| Four eBGP sessions | Passed |
| Enterprise prefix availability | Passed |
| Inbound/outbound prefix filtering | Passed |
| ISP prefix isolation | Passed |
| ISP-1 preferred over ISP-2 | Passed |
| Both provider loopback pings in healthy state | Passed |
| Conditional OSPF default origination | Passed |
| Default withdrawal and recovery | Passed |
| Single ISP-link failure and recovery | Passed |
| Existing OSPF and legacy routes preserved | Passed |
| Startup configuration saved on all four routers | Passed |

**Conclusion:** Phase 4 meets its defined lab objectives. The main remaining architectural consideration is that default-route failover and destination-specific reachability are different properties. The current design deliberately tests that distinction.
