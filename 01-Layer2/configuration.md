# Phase 1 — Enterprise Layer 2 Configuration

**Project:** Enterprise Network Security Architecture and Intrusion Detection in a Virtualized Environment
**Phase:** 1A — Enterprise Layer 2 Foundation
**Status:** Implementation Complete
**Freeze Status:** Pending Final Freeze Review
**Baseline:** Phase-1A-L2-Baseline-v1

---

## 1. Objective

The objective of Phase 1A was to establish the Layer 2 switching foundation of the enterprise network.

The phase introduced and validated:

* VLAN segmentation
* 802.1Q trunking
* Access ports
* Management SVIs
* LACP EtherChannel
* Rapid PVST+
* STP root and secondary-root placement
* Redundant uplinks
* Layer 2 failure recovery

No inter-VLAN routing was implemented during this phase.

---

## 2. Phase Boundary

### Included

* VLAN creation
* VLAN propagation
* Access-port configuration
* Trunk configuration
* Management VLAN
* Management SVIs
* LACP
* EtherChannel
* RSTP / Rapid PVST+
* STP root placement
* Redundant Layer 2 paths
* Layer 2 failure testing

### Excluded

* Inter-VLAN routing
* OSPF
* EIGRP
* BGP
* MPLS
* VRF
* Firewall policy
* NAT
* IDS
* Security monitoring
* Application services

The Layer 3 gateway addresses were planned in Phase 0, but routing between VLANs is intentionally deferred to Phase 2.

---

# 3. Topology

```text
                         DIST-1
                    STP ROOT
                  10.10.99.11
                  /          \
             Po1 /            \ 
                /              \
               /                \
          DIST-2                 |
       10.10.99.12              |
          /    \                 |
         /      \                |
      ACC-1    ACC-2             |
   .99.13      .99.14            |
      |          |               |
    VPCS-1     VPCS-2            |
```

### Physical connections

```text
DIST-1 Et0/0 <----> DIST-2 Et0/0
DIST-1 Et0/1 <----> DIST-2 Et0/1

DIST-1 Et0/2 <----> ACC-1 Et0/0
DIST-1 Et0/3 <----> ACC-2 Et0/0

DIST-2 Et0/2 <----> ACC-1 Et0/1
DIST-2 Et0/3 <----> ACC-2 Et0/1

ACC-1 Et0/2 <----> VPCS-1
ACC-2 Et0/2 <----> VPCS-2
```

DIST-1 and DIST-2 form an LACP EtherChannel.

The access switches have redundant individual uplinks toward the distribution switches. STP prevents the resulting Layer 2 loop.

---

# 4. Management Addressing

| Device | Management VLAN | IP Address     |
| ------ | --------------: | -------------- |
| DIST-1 |              99 | 10.10.99.11/24 |
| DIST-2 |              99 | 10.10.99.12/24 |
| ACC-1  |              99 | 10.10.99.13/24 |
| ACC-2  |              99 | 10.10.99.14/24 |

These are the actual addresses used during Phase 1A implementation.

They should not be changed to the earlier `.21/.22` values from the preliminary lab addressing scheme.

---

# 5. VLAN Plan

| VLAN | Name       | Subnet        | Gateway    | Phase 1A Use       |
| ---: | ---------- | ------------- | ---------- | ------------------ |
|   10 | USERS      | 10.10.10.0/24 | 10.10.10.1 | Active             |
|   20 | VOICE      | 10.10.20.0/24 | 10.10.20.1 | Created/propagated |
|   30 | SERVERS    | 10.10.30.0/24 | 10.10.30.1 | Created/propagated |
|   40 | SECURITY   | 10.10.40.0/24 | 10.10.40.1 | Created/propagated |
|   50 | DMZ        | 10.10.50.0/24 | 10.10.50.1 | Created/propagated |
|   60 | GUEST      | 10.10.60.0/24 | 10.10.60.1 | Created/propagated |
|   99 | MANAGEMENT | 10.10.99.0/24 | 10.10.99.1 | Active             |

The gateway addresses are architectural allocations only at this stage. No Layer 3 gateway routing was configured.

---

# 6. VLAN Configuration

The following VLANs were created on all four switches:

```cisco
enable
configure terminal

vlan 10
 name USERS

vlan 20
 name VOICE

vlan 30
 name SERVERS

vlan 40
 name SECURITY

vlan 50
 name DMZ

vlan 60
 name GUEST

vlan 99
 name MANAGEMENT

end
```

Verification:

```cisco
show vlan brief
```

---

# 7. Management SVI Configuration

## DIST-1

```cisco
interface vlan 99
 ip address 10.10.99.11 255.255.255.0
 no shutdown
```

## DIST-2

```cisco
interface vlan 99
 ip address 10.10.99.12 255.255.255.0
 no shutdown
```

## ACC-1

```cisco
interface vlan 99
 ip address 10.10.99.13 255.255.255.0
 no shutdown
```

## ACC-2

```cisco
interface vlan 99
 ip address 10.10.99.14 255.255.255.0
 no shutdown
```

Verification:

```cisco
show ip interface brief
```

The SVIs initially showed `down/down` until VLAN 99 had an active Layer 2 path.

This demonstrated an important SVI dependency:

```text
Physical/Layer 2 connectivity
        ↓
VLAN exists
        ↓
VLAN is active on the path
        ↓
SVI can become operational
```

No `ip routing` was enabled.

---

# 8. Distribution Interconnect — LACP EtherChannel

DIST-1 and DIST-2 use two physical links as members of Port-Channel 1.

## DIST-1

```cisco
interface range ethernet 0/0 -1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50,60,99
 channel-group 1 mode active
 no shutdown
```

Port-Channel:

```cisco
interface port-channel 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50,60,99
 no shutdown
```

## DIST-2

The same LACP configuration was applied to Ethernet 0/0 and 0/1 and Port-Channel 1.

---

# 9. LACP Troubleshooting Event

When DIST-1 was configured before DIST-2, the member interfaces became suspended:

```text
Et0/0 suspended: LACP currently not enabled on the remote port.
Et0/1 suspended: LACP currently not enabled on the remote port.
```

The EtherChannel showed:

```text
Po1(SD) LACP Et0/0(s) Et0/1(s)
```

This was expected because the remote switch had not yet negotiated LACP.

After configuring DIST-2, the bundle became:

```text
Po1(SU) LACP Et0/0(P) Et0/1(P)
```

Meaning:

* `S` = Layer 2 EtherChannel
* `U` = Port-Channel in use
* `P` = member successfully bundled
* `LACP` = LACP negotiation active

Verification:

```cisco
show etherchannel summary
show interfaces port-channel 1
show interfaces trunk
```

---

# 10. Access-Switch Uplinks

The four distribution-to-access links were configured as 802.1Q trunks.

Example:

```cisco
interface ethernet 0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50,60,99
 no shutdown
```

Equivalent configuration was applied to the redundant uplinks.

The access switches therefore have:

```text
ACC-1
  ├── DIST-1
  └── DIST-2

ACC-2
  ├── DIST-1
  └── DIST-2
```

The redundant paths create a potential Layer 2 loop, which is handled by STP.

---

# 11. Endpoint Access Ports

## ACC-1

```cisco
interface ethernet 0/2
 switchport mode access
 switchport access vlan 10
 no shutdown
```

## ACC-2

```cisco
interface ethernet 0/2
 switchport mode access
 switchport access vlan 10
 no shutdown
```

Verification:

```cisco
show vlan brief
```

The endpoint ports appeared under VLAN 10.

---

# 12. VPCS Addressing

## VPCS-1

```text
10.10.10.101/24
```

## VPCS-2

```text
10.10.10.102/24
```

No default gateway was configured because Phase 1A is Layer 2 only.

The endpoints were therefore used to test same-subnet Layer 2 connectivity.

---

# 13. Rapid PVST+ Configuration

Rapid PVST+ was enabled on all switches:

```cisco
configure terminal
spanning-tree mode rapid-pvst
end
```

DIST-1 was configured as the preferred root:

```cisco
spanning-tree vlan 10,20,30,40,50,60,99 root primary
```

DIST-2 was configured as the secondary root:

```cisco
spanning-tree vlan 10,20,30,40,50,60,99 root secondary
```

---

# 14. Final Layer 2 Design

The resulting Layer 2 hierarchy is:

```text
                    DIST-1
                  STP ROOT
                     |
                  Port-Channel
                     |
                    DIST-2
                  STP SECONDARY
                   /       \
                  /         \
              ACC-1       ACC-2
                |            |
             VLAN 10      VLAN 10
                |            |
             VPCS-1       VPCS-2
```

The redundant access uplinks provide an alternate path.

STP blocks one path during normal operation and can activate it when the forwarding path fails.

---

# 15. Errors Encountered During Configuration

### Incorrect hostname verification command

Attempted:

```cisco
show running config | include host name.
```

Result:

```text
% Invalid input detected
```

Correct command:

```cisco
show running-config | include hostname
```

---

### IOL trunk encapsulation error

Attempted:

```cisco
switchport mode trunk
```

Result:

```text
Command rejected: An interface whose trunk encapsulation is "Auto" can not be configured to "trunk" mode.
```

Fix:

```cisco
switchport trunk encapsulation dot1q
switchport mode trunk
```

This is an IOL-specific practical consideration encountered during the lab.

---

### LACP suspended members

Before the remote side was configured:

```text
Et0/0 suspended: LACP currently not enabled on the remote port.
Et0/1 suspended: LACP currently not enabled on the remote port.
```

Resolution:

Configure matching LACP parameters on DIST-2.

---

# 16. Configuration Persistence

The final configurations were saved using:

```cisco
copy running-config startup-config
```

Result:

```text
[OK]
```

This was performed after final verification.

---

# 17. Phase 1A Configuration Result

The Layer 2 switching foundation was successfully implemented:

* VLANs created
* Management VLAN established
* Management SVIs operational
* 802.1Q trunks operational
* LACP EtherChannel operational
* Redundant access uplinks operational
* Rapid PVST+ operational
* DIST-1 established as STP root
* DIST-2 established as STP secondary
* VPCS endpoints placed in VLAN 10
* Same-subnet connectivity verified
* Configuration saved

**Configuration Gate: COMPLETE**
