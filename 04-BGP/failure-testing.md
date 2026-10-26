 # Phase 4 — Failure Testing and Recovery

## 1. Purpose

This document records deliberate failures used to test the dual-ISP eBGP design. Each test includes the failure introduced, the expected behavior, the observed behavior, and the recovery procedure.

**Phase status:** Failure tests completed; configurations saved after testing.

## 2. Test 1 — Withdraw ISP-1's default route

### Objective

Verify that the enterprise can select ISP-2's default route when ISP-1 stops originating a default.

### Initial condition

- Both providers have static default routes to Null0.
- Both provider BGP sessions are established.
- ISP-1 has local preference 200.
- ISP-2 has local preference 100.

### Procedure

On ISP-1:

```cisco
configure terminal
no ip route 0.0.0.0 0.0.0.0 Null0
end
```

Check the result on ISP-1 and both enterprise edge routers:

```cisco
show ip route 0.0.0.0
show ip bgp
show ip bgp summary
```

### Expected behavior

- ISP-1 no longer has a matching default route to originate.
- The ISP-1 default is withdrawn from its BGP advertisements.
- The enterprise edges select ISP-2's default route.
- The OSPF default remains available while EDGE-1 still has a default route through ISP-2.

### Observed behavior

- ISP-1 reported `% Network not in table` for the withdrawn default network.
- EDGE-1 and EDGE-2 selected ISP-2's default with local preference 100.
- DIST-1 and DIST-2 retained the OSPF default.

**Result: Passed.**

### Recovery

On ISP-1:

```cisco
configure terminal
ip route 0.0.0.0 0.0.0.0 Null0
end
```

Verify that the default returns and ISP-1 becomes preferred again.

**Observed recovery:** Both edges preferred ISP-1 at local preference 200; the internal OSPF default remained present.

---

## 3. Test 2 — Remove both provider defaults

### Objective

Verify that the enterprise does not retain a false default route when neither provider supplies one.

### Procedure

Remove the ISP-2 default first:

```cisco
configure terminal
no ip route 0.0.0.0 0.0.0.0 Null0
end
```

Then remove the ISP-1 default:

```cisco
configure terminal
no ip route 0.0.0.0 0.0.0.0 Null0
end
```

Check the enterprise edges and distribution switches:

```cisco
show ip route 0.0.0.0
show ip bgp
show ip ospf neighbor
```

### Expected behavior

- Neither edge has a usable BGP default path.
- EDGE-1 has no default route and stops originating the OSPF default.
- The distribution switches withdraw their OSPF default route.

### Observed behavior

- EDGE-1's BGP output showed no available default path.
- EDGE-2 had no default route in its routing table.
- DIST-1 and DIST-2 removed their OSPF default routes.

**Result: Passed.**

### Recovery

Restore the static default on both ISP routers:

```cisco
configure terminal
ip route 0.0.0.0 0.0.0.0 Null0
end
```

Run the command on ISP-1 and ISP-2.

### Recovery result

- Both edge routers relearned the provider defaults.
- ISP-1 became the preferred default route.
- The distribution switches relearned the OSPF default.

**Key lesson:** `default-information originate` without `always` prevents EDGE-1 from continuing to advertise a default when no default route is installed.

---

## 4. Test 3 — Shut down EDGE-1's ISP-1 interface

### Objective

Verify BGP adjacency failure, default-route failover, destination-specific reachability, and session recovery when one enterprise-to-provider link fails.

### Failure injection

On EDGE-1:

```cisco
configure terminal
interface GigabitEthernet3
 shutdown
end
```

### Verification commands

```cisco
show ip interface brief
show ip bgp summary
show ip route 0.0.0.0
show ip route 198.51.100.0
ping 198.51.100.1
```

### Expected behavior

- Gi3 becomes administratively down.
- The ISP-1 eBGP session drops.
- The ISP-2 eBGP session remains established.
- EDGE-1 selects ISP-2's default route.
- The ISP-1-specific prefix is withdrawn because the ISP-1 path is unavailable.

### Observed behavior

- Gi3 was administratively down/down.
- Neighbor `10.255.110.2` moved to Idle.
- Neighbor `10.255.112.2` remained established and supplied two prefixes.
- EDGE-1's default route moved to ISP-2 via `10.255.112.2`, with tag 65002.
- `198.51.100.0/24` was no longer in the routing table.
- Ping to `198.51.100.1` returned `U.U.U`, with 0/5 successful packets.

**Result: Passed for the intended failure behavior.**

The failed ping is not a failure of default-route failover. It demonstrates that a default route cannot replace a missing specific route when the backup provider does not advertise or provide reachability to that destination.

### Recovery

On EDGE-1:

```cisco
configure terminal
interface GigabitEthernet3
 no shutdown
end
```

Verify:

```cisco
show ip interface brief
show ip bgp summary
show ip route 0.0.0.0
show ip route 198.51.100.0
ping 198.51.100.1
```

### Observed recovery

- Gi3 returned to up/up.
- The ISP-1 eBGP session re-established.
- Both ISP peers again supplied two prefixes.
- ISP-1's default route became preferred.
- `198.51.100.0/24` returned.
- Ping to `198.51.100.1` succeeded with 5/5 packets.

ISP-1 logged a brief BGP connection-collision notification during reconvergence, then the session returned to the Up state. No persistent fault remained.

**Result: Passed.**

---

## 5. Test 4 — Verify the unaffected edge

### Objective

Ensure that the EDGE-1 link test did not leave EDGE-2 in an abnormal state.

### Checks

On EDGE-2:

```cisco
show ip interface brief
show ip bgp summary
show ip route 0.0.0.0
ping 198.51.100.1
ping 203.0.113.1
```

### Observed behavior

- Gi1, Gi2, Gi3, and Gi4 were up/up.
- Both eBGP sessions were established and each supplied two prefixes.
- The preferred default route remained via ISP-1.
- Both provider loopback pings succeeded with 5/5 packets.

**Result: Passed.**

---

## 6. What these failure tests prove

The tests demonstrate:

1. **BGP withdrawal:** Removing a provider's default route withdraws the corresponding advertised default.
2. **Backup selection:** The enterprise can select ISP-2 when ISP-1's default becomes unavailable.
3. **Conditional internal default:** The OSPF default is withdrawn when EDGE-1 has no default route.
4. **Session recovery:** Restoring a failed interface allows the eBGP session and routes to recover.
5. **Specific-route dependency:** Default-route failover does not guarantee reachability to every destination, especially a prefix advertised only by the failed provider.
6. **Policy consistency:** The backup path follows the configured local-preference policy when both provider defaults are available.

## 7. Troubleshooting checklist

If a test behaves differently in a future run, troubleshoot in this order:

1. **Physical/interface state**
   - `show ip interface brief`
   - Check addressing, `shutdown`, and interface status.

2. **Direct IP connectivity**
   - Ping the directly connected peer address.
   - Check subnet masks and point-to-point addressing.

3. **BGP session**
   - `show ip bgp summary`
   - Confirm remote AS numbers and identify peers that are not Established.

4. **Route existence**
   - `show ip route 0.0.0.0`
   - `show ip route <prefix>`
   - Confirm that a BGP `network` statement has a matching route.

5. **Policy**
   - `show ip prefix-list`
   - `show route-map`
   - `show ip bgp neighbors <peer-ip> advertised-routes`
   - Check prefix-list spelling, neighbor attachments, and implicit deny behavior.

6. **Path selection**
   - `show ip bgp`
   - Check local preference and whether a path is eligible and selected.

7. **OSPF propagation**
   - `show ip ospf neighbor`
   - `show ip route 0.0.0.0`
   - Confirm that EDGE-1 has a default route and that the OSPF default is being originated.

8. **Data plane**
   - Ping the relevant loopback endpoint.
   - Distinguish a missing specific route from loss of the default route.

9. **Recovery and persistence**
   - Restore the failed interface or static route.
   - Verify sessions and routing tables again.
   - Save the final configuration after tests.

## 8. Final state

All deliberate failures were recovered. Both provider defaults were restored, the EDGE-1 ISP-1 link was brought back up, both enterprise edge routers were healthy, and the final configurations were saved to startup configuration on all four routers.
