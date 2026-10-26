# Phase 5 — Verification and Test Evidence

## 1. Purpose

This document records the verification performed after integrating VyOS as the explicit perimeter firewall and routing boundary.

The tests focus on interface state, routing, firewall behavior, end-to-end ICMP connectivity, and the final routing state after failover and recovery.

## 2. Verification Summary

| Test | Expected result | Observed result | Status |
|---|---|---|---|
| VyOS interface state | All four interfaces operational | All four interfaces previously observed up/up | PASS |
| Primary ISP test destination | Reachable from VyOS | 3/3 replies from 198.51.100.1 | PASS |
| Backup ISP test destination | Reachable from VyOS | 5/5 replies from 203.0.113.1 during backup testing | PASS |
| VPC7 default gateway | Reachable | 5/5 replies from 10.10.10.1 | PASS |
| VPC7 to primary test destination | Permitted ICMP succeeds | 5/5 replies from 198.51.100.1 | PASS |
| VPC7 to backup test destination | Permitted ICMP succeeds | 5/5 replies from 203.0.113.1 | PASS |
| Non-permitted test destination | New ICMP traffic is dropped | Five timeouts to 192.0.2.1; LAN-to-WAN default-drop counter increased by five | PASS |
| Primary route failure | Backup becomes best route | Route via 10.255.106.2, eth3, metric 20 became best | PASS |
| Primary route recovery | EDGE-1 becomes best route again | Route via 10.255.105.2, eth2, metric 10 became best | PASS |
| Final probe configuration | Original targets restored | 198.51.100.1 and 203.0.113.1 displayed in configuration mode | PASS |
| Configuration save | Final restored configuration saved | `save` executed without a reported error | PASS |

**Interpretation:** The recorded tests demonstrate initial connectivity, the observed effect of the LAN-to-WAN default-drop rule, and health-check-driven route failover and recovery. They do not constitute exhaustive validation of every rule, protocol, NAT translation, or management service.

## 3. Interface Verification

The following interfaces were observed up/up:

| Interface | Address |
|---|---|
| eth0 | 10.255.101.2/30 |
| eth1 | 10.255.102.2/30 |
| eth2 | 10.255.105.1/30 |
| eth3 | 10.255.106.1/30 |

Operational command used:

```text
run show interfaces
```

All four interfaces were operational during the recorded verification. This establishes link/interface state at the time of observation; it does not independently establish the health of every downstream route or application.

## 4. Primary and Backup Connectivity

### 4.1 Primary test destination

Command:

```text
run ping 198.51.100.1 count 3
```

Observed result:

```text
3 packets transmitted, 3 received, 0% packet loss
```

**Conclusion:** The primary test destination was reachable from VyOS during normal operation.

### 4.2 Backup test destination

Command:

```text
run ping 203.0.113.1 count 5
```

Observed during backup-route testing:

```text
5 packets transmitted, 5 received, 0% packet loss
```

**Conclusion:** VyOS could reach the backup test destination while the backup route was selected.

## 5. End-to-End Verification from VPC7

VPC7 was connected to ACC-1 on an access port in VLAN 10 (USERS).

Its addressing was:

```text
IP address:      10.10.10.10
Subnet mask:     255.255.255.0
Default gateway: 10.10.10.1
```

Equivalent VPCS command:

```text
ip 10.10.10.10/24 10.10.10.1
```

The gateway is the existing HSRP virtual IP on the distribution layer.

### 5.1 Gateway reachability

Command:

```text
ping 10.10.10.1
```

Observed result: five successful replies.

**Conclusion:** VPC7 could reach its configured default gateway.

### 5.2 Primary destination

Command:

```text
ping 198.51.100.1
```

Observed result: five successful replies.

**Conclusion:** VPC7 could reach the primary test destination through the implemented forwarding path.

### 5.3 Backup destination

Command:

```text
ping 203.0.113.1
```

Observed during the backup-route test: five successful replies.

**Conclusion:** VPC7 could reach the backup test destination while EDGE-2 was selected as the best default route.

## 6. Firewall Policy Verification

### 6.1 Permitted ICMP

The `PH5-LAN-WAN` policy permits new ICMP traffic to:

- `198.51.100.0/24`
- `203.0.113.0/24`

The successful VPC7 ping tests to `198.51.100.1` and `203.0.113.1` are consistent with this policy and the configured forwarding/NAT path.

### 6.2 Default-drop behavior

VPC7 attempted to ping:

```text
192.0.2.1
```

Observed result: five timeouts.

The LAN-to-WAN firewall counter showed five packets and 420 bytes on the default-drop rule after this test. The configured allowed new-ICMP destinations did not include `192.0.2.1`.

**Conclusion:** The timeout and matching firewall counter support the conclusion that this test traffic was dropped by the LAN-to-WAN default policy.

The timeout alone would not prove a firewall drop; the counter correlation is the additional evidence.

### 6.3 Relevant firewall counters

At the time of the recorded counter inspection, `PH5-LAN-WAN` showed:

| Rule | Observed packets | Interpretation |
|---|---:|---|
| Established accept | 0 | No matching packets recorded at that inspection |
| Related accept | 0 | No matching packets recorded at that inspection |
| Invalid drop | 0 | No matching packets recorded at that inspection |
| ICMP to 198.51.100.0/24 | 5 | Matching permitted ICMP traffic observed |
| ICMP to 203.0.113.0/24 | 0 at that earlier inspection | Counter was captured before the VPC7 backup-destination test |
| Default drop | 5 | Matches the five attempted pings to 192.0.2.1 |

Firewall counters are time-dependent and can change as traffic flows. The figures above represent the recorded inspection, not necessarily the current live counters.

The WAN-to-LAN established rule later showed five packets and 420 bytes, consistent with return traffic from the earlier successful ping exchange.

### 6.4 NAT verification boundary

The following source NAT rules were displayed:

```text
Rule 100: source 10.10.0.0/16, outbound eth2, masquerade
Rule 110: source 10.10.0.0/16, outbound eth3, masquerade
```

The end-to-end tests are consistent with successful forwarding and return traffic. However, the recorded evidence does not include a packet capture or NAT session/counter output that directly demonstrates the translated source address.

**Status:** NAT configured; direct packet-level translation evidence not recorded.

## 7. Final Routing Verification

After the original primary probe target was restored and the configuration was committed, the following command was executed:

```text
run show ip route 0.0.0.0/0
```

Final observed output:

```text
Routing entry for 0.0.0.0/0
  Known via "kernel", distance 0, metric 20
  * 10.255.106.2, via eth3

Routing entry for 0.0.0.0/0
  Known via "kernel", distance 0, metric 10, best
  * 10.255.105.2, via eth2
```

**Conclusion:** Both default routes were installed, and EDGE-1 was marked as the best route with metric 10. EDGE-2 remained installed with metric 20.

The final configuration was inspected in configuration mode using:

```text
show protocols failover
```

The output confirmed:

- Primary next hop `10.255.105.2`, interface `eth2`, metric 10, target `198.51.100.1`.
- Backup next hop `10.255.106.2`, interface `eth3`, metric 20, target `203.0.113.1`.
- Both health checks use ICMP with a five-second timeout.

The `save` command was then executed after restoration of the original probe target.

## 8. Verification Commands for Future Audits

Run the relevant commands from the appropriate VyOS mode:

```text
# Operational mode
show interfaces
show ip route 0.0.0.0/0
show nat source rules

# Configuration mode
configure
show protocols failover
show
```

For firewall inspection, use the supported operational firewall-show commands for the installed VyOS release. The recorded policy counter inspection used:

```text
run show firewall ipv4 name PH5-LAN-WAN
```

If a command is rejected by the installed image, verify its supported syntax rather than assuming the feature or configuration is absent.

## 9. Outstanding Verification Items

The following were not demonstrated by the recorded tests:

- Direct packet-level proof of source NAT translation.
- Exhaustive testing of every firewall rule and zone direction.
- SSH service availability and management access from VLAN 99.
- TCP/UDP application connectivity, which is intentionally not permitted by the current new-session LAN-to-WAN policy.
- Full end-to-end failure of each physical/logical link independently.
- Firewall high availability or state synchronization.

These items must not be marked as passed based solely on the successful ICMP and route-selection tests.

## 10. Conclusion

The recorded evidence supports successful implementation of the initial VyOS perimeter boundary, permitted ICMP connectivity to the two test destinations, observed default-drop behavior, and health-check-driven default-route failover and recovery.

Phase 5 verification is complete for the tests listed above. Broader policy testing and additional firewall mechanisms remain outside the scope of this phase and are reserved for a separate lab.
