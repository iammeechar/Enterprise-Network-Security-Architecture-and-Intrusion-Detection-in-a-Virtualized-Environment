# Phase 5 — Firewall Failure Testing and Troubleshooting

## 1. Phase Information

- **Project:** Enterprise Network Security Architecture and Intrusion Detection in a Virtualized Environment
- **Phase:** 5 — Firewall Integration and Security Policy
- **Platform:** EVE-NG
- **Firewall:** VyOS 1.5-rolling-202406170021
- **Design:** Option B — Explicit Routing Boundary
- **Status:** Recorded failure and recovery tests passed
- **Configuration status:** Original health-check targets restored; configuration committed and saved

## 2. Purpose

This document records the failure scenarios, observed results, and troubleshooting decisions associated with the VyOS perimeter firewall integration.

The primary objective was to demonstrate that the firewall could detect the failure of its preferred default-route health check, select the backup route through EDGE-2, maintain connectivity to the backup test destination, and return to EDGE-1 after the original health-check target was restored.

The test deliberately changed the health-check target rather than shutting down an interface or changing the EDGE router configuration. This isolated the behavior of the VyOS failover mechanism.

## 3. Test Environment

### 3.1 Default-route design

| Role | Next hop | Interface | Metric | Normal health-check target |
|---|---|---|---:|---|
| Primary | 10.255.105.2 | eth2 | 10 | 198.51.100.1 |
| Backup | 10.255.106.2 | eth3 | 20 | 203.0.113.1 |

The primary route is preferred under normal conditions because its metric is lower. The health-check mechanism determines whether the tracked route remains eligible.

### 3.2 Test host

VPC7 was connected to VLAN 10 (USERS) and configured as follows:

```text
IP address:      10.10.10.10
Subnet mask:     255.255.255.0
Default gateway: 10.10.10.1
```

The tests used ICMP because the initial LAN-to-WAN firewall policy intentionally permits new ICMP traffic only to the two documented test prefixes.

## 4. Failure Test 1 — Controlled Primary Probe Failure

### Objective

Verify that VyOS removes the preferred default route when its configured primary health-check target stops responding.

### Procedure

The original primary target was:

```text
198.51.100.1
```

A temporary target was configured:

```text
198.51.100.254
```

The initial command was:

```text
set protocols failover route 0.0.0.0/0 next-hop 10.255.105.2 check target 198.51.100.254
```

The subsequent configuration comparison showed that this command had added a second target instead of replacing the original target.

This was identified before committing the change.

### Troubleshooting decision

The configuration was inspected:

```text
show protocols failover
```

Both targets were visible under the primary next hop:

```text
target 198.51.100.1
target 198.51.100.254
```

The original target was then explicitly removed:

```text
delete protocols failover route 0.0.0.0/0 next-hop 10.255.105.2 check target 198.51.100.1
```

The configuration comparison confirmed the intended change:

```text
- target "198.51.100.1"
+ target "198.51.100.254"
```

Only after confirming that the primary had a single temporary target was the change committed.

### Observed result

The default-route output showed:

```text
Routing entry for 0.0.0.0/0
  Known via "kernel", distance 0, metric 20, best
  * 10.255.106.2, via eth3
```

The primary route was no longer shown as the best route. The backup through EDGE-2 became preferred.

### Conclusion

**PASS.** The controlled health-check failure caused VyOS to prefer the backup default route.

The test exercised health-check-driven route selection without physically shutting down either WAN-facing interface.

## 5. Failure Test 2 — Connectivity During Backup Operation

### Objective

Verify that traffic could reach the backup test destination after EDGE-2 became the preferred default route.

### Test A — VyOS to backup destination

Command:

```text
run ping 203.0.113.1 count 5
```

Observed result:

```text
5 packets transmitted, 5 received, 0% packet loss
```

### Test B — VPC7 to backup destination

Command:

```text
ping 203.0.113.1
```

Observed result: five successful replies.

### Conclusion

**PASS.** The backup path provided connectivity from VyOS and VPC7 to the simulated ISP2 test destination while EDGE-2 was the best default route.

This confirms reachability for the tested ICMP flow. It does not prove that every application or protocol would remain available during failover.

## 6. Recovery Test — Restore the Primary Health Check

### Objective

Restore the original configuration and verify that EDGE-1 becomes the preferred default route again.

### Procedure

The temporary probe was removed:

```text
delete protocols failover route 0.0.0.0/0 next-hop 10.255.105.2 check target 198.51.100.254
```

The original target was restored:

```text
set protocols failover route 0.0.0.0/0 next-hop 10.255.105.2 check target 198.51.100.1
```

The configuration comparison showed:

```text
- target "198.51.100.254"
+ target "198.51.100.1"
```

The original target was confirmed in configuration mode before committing.

### Observed result

After the commit, the default-route output showed:

```text
Routing entry for 0.0.0.0/0
  Known via "kernel", distance 0, metric 20
  * 10.255.106.2, via eth3

Routing entry for 0.0.0.0/0
  Known via "kernel", distance 0, metric 10, best
  * 10.255.105.2, via eth2
```

EDGE-1 was again the best route, while EDGE-2 remained installed as the backup.

The final failover configuration showed:

- Primary target: `198.51.100.1`
- Backup target: `203.0.113.1`
- Primary metric: 10
- Backup metric: 20
- Health-check type: ICMP
- Health-check timeout: five seconds

The configuration was saved after restoration.

### Conclusion

**PASS.** The original health-check target was restored, the primary route recovered, and the expected route preference returned.

## 7. Firewall Policy Failure Test — Unpermitted Destination

### Objective

Check whether new ICMP traffic to a destination outside the permitted test prefixes is dropped by the LAN-to-WAN policy.

### Procedure

From VPC7:

```text
ping 192.0.2.1
```

### Observed result

The five ping attempts timed out.

The `PH5-LAN-WAN` firewall counters subsequently showed five packets and 420 bytes on the default-drop rule.

The configured new-ICMP permit rules covered `198.51.100.0/24` and `203.0.113.0/24`, not `192.0.2.1`.

### Conclusion

**PASS for the recorded test.** The failed ping attempts and matching default-drop counter support the conclusion that this test traffic was dropped by the LAN-to-WAN default policy.

This was a limited policy test. It did not test every rule, protocol, source subnet, or zone direction.

## 8. Troubleshooting Lessons

### 8.1 A configuration command can add instead of replace

The first temporary-target command added a second target. The `compare` and `show protocols failover` outputs exposed this before commit.

**Lesson:** Inspect the candidate configuration after changes to multi-valued settings. Do not assume that setting a new value automatically removes the old value.

### 8.2 Verify route selection, not just link state

The meaningful failover evidence was the change in the best default route:

- During failure: EDGE-2 via `10.255.106.2`, metric 20.
- After recovery: EDGE-1 via `10.255.105.2`, metric 10.

**Lesson:** An interface being up does not establish that its path is healthy. Route selection and end-to-end reachability provide complementary evidence.

### 8.3 Test from more than one point

VyOS successfully reached the backup destination, and VPC7 also reached it.

**Lesson:** A firewall-originated ping checks a different traffic path from a client-originated ping. Both are useful, but neither substitutes for the other.

### 8.4 Correlate connectivity failures with firewall counters

VPC7's five failed pings to `192.0.2.1` correlated with five packets on the LAN-to-WAN default-drop rule.

**Lesson:** A timeout alone does not identify the cause. Correlating test traffic with counters gives stronger evidence for a firewall-policy diagnosis.

### 8.5 Save only after restoring the intended state

The primary probe was restored, the candidate configuration was checked, the change was committed, and the default route was verified before the final save.

**Lesson:** Controlled failure tests should leave the lab in its intended baseline state. Temporary failure settings should not be saved as the final configuration.

### 8.6 Distinguish configured behavior from directly observed behavior

The source NAT rules were present and end-to-end connectivity succeeded, but the test did not independently capture the translated source address.

**Lesson:** Record NAT as configured until packet-level or session-level evidence directly confirms the translation.

## 9. Failure-Test Summary

| Test | Expected behavior | Observed evidence | Status |
|---|---|---|---|
| Primary probe failure | EDGE-2 becomes best route | Metric 20 via eth3 became best | PASS |
| Backup connectivity from VyOS | ISP2 test destination reachable | 5/5 successful replies | PASS |
| Backup connectivity from VPC7 | Client reaches ISP2 test destination | 5 successful replies | PASS |
| Unpermitted destination | New ICMP traffic dropped | Five timeouts and matching default-drop counter | PASS |
| Restore primary probe | EDGE-1 becomes best route | Metric 10 via eth2 became best | PASS |
| Restore final configuration | Original targets retained and saved | Configuration displayed correctly; save executed | PASS |

## 10. Scope Limitations and Deferred Testing

The following are not claimed as completed tests:

- Physical or logical shutdown of each edge link.
- Failure of the ISP routers themselves.
- Independent packet-level verification of NAT translation.
- Exhaustive TCP/UDP, application, and firewall-zone policy testing.
- SSH management access and service availability.
- Firewall high availability and state synchronization.
- Inter-VLAN enforcement through VyOS.

These topics are either separate verification tasks or outside the current phase scope. Further firewall mechanisms and expanded security policies will be practiced in a separate lab.

## 11. Final Assessment

Phase 5 successfully demonstrated the tested VyOS perimeter-routing behavior, restricted ICMP policy enforcement, backup-path reachability, and restoration of the primary default route.

The temporary health-check change was removed, the original targets were restored, and the final configuration was saved.

The phase is ready for documentation review and formal freeze, subject to confirming that the three documentation files accurately match the final live configuration.
