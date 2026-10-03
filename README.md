# Cisco Enterprise Network Implementation & Security Lab

A full enterprise network built and configured in Cisco Packet Tracer, covering switching, routing, network services, security hardening, and centralized management.

---

## Table of Contents

- [Overview](#overview)
- [Network Topology](#network-topology)
- [IP Addressing Scheme](#ip-addressing-scheme)
- [Implementation Breakdown](#implementation-breakdown) 
  - [1. Switching](#switching)
  - [2. Routing](#routing)
  - [3. Network Services](#network-services)
  - [4. Security](#security)
  - [5. Management & Monitoring](#management--monitoring)
- [Design Decisions & Rationale](#design-decisions--rationale)
- [Challenges & Troubleshooting](#challenges--troubleshooting)
- [Verification & Testing](#verification--testing)
- [Known Limitations](#known-limitations)
- [Future Improvements](#future-improvements)

---

## Overview

This project simulates a corporate network consisting of a headquarters (HQ) and two remote branches. It was built as a hands-on consolidation of CCNA topics, with every configuration decision documented and justified rather than applied by rote.

**Tools used:** Cisco Packet Tracer
**Devices:** 4x Cisco 2911 Routers, 1x Cisco 3650 Multilayer Switch, 4x Cisco 2960 Switches, 1x Server (DHCP + Syslog), 14x End devices

**Topics covered:** VLANs, VTP, STP, EtherChannel, Inter-VLAN Routing, Router-on-a-Stick, OSPF, Static Routing, DHCP Relay, NAT/PAT, ACLs, Port Security, DHCP Snooping, SSH, NTP, Syslog, SNMP

---

## Network Topology

```
                            [ISP-Router]
                          203.0.113.2/30
                                 |
                          203.0.113.1/30
                            [Router0]  ← HQ Edge / NAT Gateway
                          /      |      \
                 192.168.99.5    |    192.168.99.9
                       /         |         \
              [Router1]          |          [Router2]
             Branch B            |           Branch A
                  |         192.168.99.1          |
             [Switch1]     [Core-Switch 3650]  [Switch2]
            VLANs 210-240    (L3 / SVIs)      VLANs 110-140
                                 |
                    +------------+------------+
                    |            |            |
               [SW-IT]    [SW-Sales-HR]   [DHCP Server]
              VLAN 20      VLAN 30, 40      VLAN 50
                    |            |
                    +===LACP=====+   ← EtherChannel (Po1)

```

### Physical Link Summary

| Link Interfaces Network  |                         |                     |
| ------------------------ | ----------------------- | ------------------- |
| Router0 ⇔ Core-Switch    | Gi0/2 ⇔ Gi1/0/1         | 192.168.99.0/30     |
| Router0 ⇔ Router1        | Gi0/0 ⇔ Gi0/0           | 192.168.99.4/30     |
| Router0 ⇔ Router2        | Gi0/1 ⇔ Gi0/0           | 192.168.99.8/30     |
| Router0 ⇔ ISP-Router     | Fa0/3/0 (Vlan1) ⇔ Gi0/0 | 203.0.113.0/30      |
| Core ⇔ SW-IT             | Gi1/0/2 ⇔ Gi0/1         | Trunk (802.1Q)      |
| Core ⇔ SW-Sales-HR       | Gi1/0/3 ⇔ Gi0/1         | Trunk (802.1Q)      |
| SW-IT ⇔ SW-Sales-HR      | Fa0/1-2 ⇔ Fa0/1-2       | EtherChannel (LACP) |

---

## IP Addressing Scheme

### Headquarters

| VLAN Name Network Gateway (SVI on Core)  |         |                 |              |
| ---------------------------------------- | ------- | --------------- | ------------ |
| 10                                       | MGMT    | 192.168.10.0/24 | 192.168.10.1 |
| 20                                       | IT      | 192.168.20.0/24 | 192.168.20.1 |
| 30                                       | HR      | 192.168.30.0/24 | 192.168.30.1 |
| 40                                       | Sales   | 192.168.40.0/24 | 192.168.40.1 |
| 50                                       | Servers | 192.168.50.0/24 | 192.168.50.1 |

### Branch A (behind Router2)

| VLAN Name Network Gateway (Sub-interface)  |               |                  |               |
| ------------------------------------------ | ------------- | ---------------- | ------------- |
| 110                                        | MGMT-BranchA  | 192.168.110.0/24 | 192.168.110.1 |
| 120                                        | IT-BranchA    | 192.168.120.0/24 | 192.168.120.1 |
| 130                                        | HR-BranchA    | 192.168.130.0/24 | 192.168.130.1 |
| 140                                        | Sales-BranchA | 192.168.140.0/24 | 192.168.140.1 |

### Branch B (behind Router1)

| VLAN Name Network Gateway (Sub-interface)  |               |                  |               |
| ------------------------------------------ | ------------- | ---------------- | ------------- |
| 210                                        | MGMT-BranchB  | 192.168.210.0/24 | 192.168.210.1 |
| 220                                        | IT-BranchB    | 192.168.220.0/24 | 192.168.220.1 |
| 230                                        | HR-BranchB    | 192.168.230.0/24 | 192.168.230.1 |
| 240                                        | Sales-BranchB | 192.168.240.0/24 | 192.168.240.1 |

### Point-to-Point Links (/30)

All router-to-router links use `/30` subnets, providing exactly two usable addresses per link — no address waste.

| Link Network      |                 |
| ----------------- | --------------- |
| Router0 ⇔ Core    | 192.168.99.0/30 |
| Router0 ⇔ Router1 | 192.168.99.4/30 |
| Router0 ⇔ Router2 | 192.168.99.8/30 |
| Router0 ⇔ ISP     | 203.0.113.0/30  |

> **Note on 203.0.113.0/24:** This range is reserved by RFC 5737 for documentation and examples. It was chosen deliberately to make the public-facing link visually distinct from the private internal ranges.

---

## Implementation Breakdown

### 1. Switching

#### VLANs & VTP

The Core Switch acts as the VTP Server; all access switches are VTP Clients, so VLANs are defined once and propagated automatically.

```cisco
! Core Switch (VTP Server)
vtp domain mycompany
vtp mode server

vlan 10
 name MGMT
vlan 20
 name IT
vlan 30
 name HR
vlan 40
 name Sales
vlan 50
 name Servers

```

```cisco
! Access Switches (VTP Clients)
vtp domain mycompany
vtp mode client

```

#### Trunking

```cisco
interface gi1/0/2
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50

```

#### STP & EtherChannel

Two parallel links were intentionally added between SW-IT and SW-Sales-HR to create a physical loop, demonstrating STP in action before consolidating them with EtherChannel.

**Before EtherChannel** — STP blocked the redundant links:

```
Fa0/1    Altn   BLK   19
Fa0/2    Altn   BLK   19
Gi0/1    Root   FWD   4

```

**After EtherChannel (LACP)** — both links bundled and forwarding:

```cisco
interface range fa0/1 - 2
 channel-group 1 mode active

interface port-channel 1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50

```

```
Group  Port-channel  Protocol   Ports
1      Po1(SU)       LACP       Fa0/1(P) Fa0/2(P)

```

### 2. Routing

#### Inter-VLAN Routing — Two Approaches

| Location Method Reason  |                                    |                                                                           |
| ----------------------- | ---------------------------------- | ------------------------------------------------------------------------- |
| HQ                      | SVIs on Layer 3 Core Switch        | High inter-VLAN traffic volume; routing happens in hardware on the switch |
| Branches                | Router-on-a-Stick (sub-interfaces) | Only 3 hosts per branch; a dedicated L3 switch would be over-engineering  |

```cisco
! HQ — SVI on Core Switch
ip routing
interface vlan 20
 ip address 192.168.20.1 255.255.255.0
 ip helper-address 192.168.50.10

```

```cisco
! Branch — Router-on-a-Stick on Router2
interface gi0/1.120
 encapsulation dot1Q 120
 ip address 192.168.120.1 255.255.255.0
 ip helper-address 192.168.50.10

```

#### OSPF (Single Area 0)

```cisco
router ospf 1
 router-id 1.1.1.1
 network 192.168.99.0 0.0.0.3 area 0
 network 192.168.99.4 0.0.0.3 area 0
 network 192.168.99.8 0.0.0.3 area 0
 default-information originate

```

| Device Router ID  |         |
| ----------------- | ------- |
| Router0           | 1.1.1.1 |
| Router1           | 2.2.2.2 |
| Router2           | 3.3.3.3 |
| Core Switch       | 4.4.4.4 |

#### Default Route Propagation

A single static default route on Router0 is advertised to the entire network via OSPF, avoiding the need to configure static routes on every device.

```cisco
ip route 0.0.0.0 0.0.0.0 203.0.113.2

router ospf 1
 default-information originate

```

Verified on a remote branch router:

```
O*E2 0.0.0.0/0 [110/1] via 192.168.99.9, GigabitEthernet0/0

```

### 3. Network Services

#### Centralized DHCP

A single DHCP server in the HQ serves all 10 user VLANs across all three sites — the standard enterprise approach (centralized management, single point of policy change).

Since DHCP requests are broadcasts that don't cross VLAN boundaries, `ip helper-address` was configured on every user-facing SVI and sub-interface:

```cisco
interface vlan 20
 ip helper-address 192.168.50.10

```

**Configured Pools:**

| Pool Gateway Start IP  |               |                |
| ---------------------- | ------------- | -------------- |
| MGMT-Pool              | 192.168.10.1  | 192.168.10.10  |
| IT-Pool                | 192.168.20.1  | 192.168.20.10  |
| HR-Pool                | 192.168.30.1  | 192.168.30.10  |
| Sales-Pool             | 192.168.40.1  | 192.168.40.10  |
| BranchA-IT-Pool        | 192.168.120.1 | 192.168.120.10 |
| BranchA-HR-Pool        | 192.168.130.1 | 192.168.130.10 |
| BranchA-Sales-Pool     | 192.168.140.1 | 192.168.140.10 |
| BranchB-IT-Pool        | 192.168.220.1 | 192.168.220.10 |
| BranchB-HR-Pool        | 192.168.230.1 | 192.168.230.10 |
| BranchB-Sales-Pool     | 192.168.240.1 | 192.168.240.10 |

> **Note:** VLAN 50 (Servers) has no DHCP pool by design — servers use static addressing so their IPs never change, which matters for `ip helper-address` references and NAT rules.

#### NAT (PAT)

All internal hosts share the single public-facing address on Router0's outside interface.

```cisco
access-list 1 permit 192.168.0.0 0.0.255.255

interface vlan 1
 ip nat outside
interface gi0/0
 ip nat inside
interface gi0/1
 ip nat inside
interface gi0/2
 ip nat inside

ip nat pool ISP_POOL 203.0.113.1 203.0.113.1 netmask 255.255.255.252
ip nat inside source list 1 pool ISP_POOL overload

```

Verified translation table:

```
Pro   Inside global    Inside local       Outside local   Outside global
icmp  203.0.113.1:1    192.168.20.11:1    203.0.113.2:1   203.0.113.2:1

```

### 4. Security

#### ACLs — Branch Isolation

**Security goal:** If one branch is compromised, the attacker should not be able to pivot laterally into the other branch. Both branches retain full access to HQ resources.

```
Branch A  ←✗ BLOCKED ✗→  Branch B
    ↓ allowed                ↓ allowed
         HQ (DHCP, shared services)

```

Nine explicit `deny` statements cover every source/destination network pair between the two branches — not just matching departments:

```cisco
ip access-list extended block-branch-to-branch
 deny ip 192.168.120.0 0.0.0.255 192.168.220.0 0.0.0.255
 deny ip 192.168.120.0 0.0.0.255 192.168.230.0 0.0.0.255
 deny ip 192.168.120.0 0.0.0.255 192.168.240.0 0.0.0.255
 deny ip 192.168.130.0 0.0.0.255 192.168.220.0 0.0.0.255
 deny ip 192.168.130.0 0.0.0.255 192.168.230.0 0.0.0.255
 deny ip 192.168.130.0 0.0.0.255 192.168.240.0 0.0.0.255
 deny ip 192.168.140.0 0.0.0.255 192.168.220.0 0.0.0.255
 deny ip 192.168.140.0 0.0.0.255 192.168.230.0 0.0.0.255
 deny ip 192.168.140.0 0.0.0.255 192.168.240.0 0.0.0.255
 permit ip any any

interface gi0/1.120
 ip access-group block-branch-to-branch in

```

A mirrored ACL is applied on Router1 for the reverse direction, so traffic is dropped at the first hop regardless of which side initiates.

#### Port Security

```cisco
interface range fa0/3 - 5
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict

```

`restrict` was chosen over `shutdown` so an unauthorized device is rejected while the legitimate device stays online.

#### DHCP Snooping

Protects against rogue DHCP servers. Only uplink ports facing the legitimate server are marked trusted; all access ports are untrusted by default.

```cisco
ip dhcp snooping
ip dhcp snooping vlan 20

interface gi0/1
 ip dhcp snooping trust

```

#### SSH & Device Hardening

Telnet is disabled entirely. Each device has its own unique credentials so a single compromised password doesn't unlock the whole network.

```cisco
service password-encryption
ip domain-name mynetwork.local
username R0Admin privilege 15 secret R0Admin@123
crypto key generate rsa      ! 1024-bit
ip ssh version 2

line vty 0 4
 login local
 transport input ssh

line console 0
 login local
 logging synchronous

```

| Device              | Username      | Password          |
| ------------------- | ------------- | ----------------- |
| Router0             | R0Admin       | R0Admin@123       |
| Router1             | R1Admin       | R1Admin@123       |
| Router2             | R2Admin       | R2Admin@123       |
| Core Switch         | CoreAdmin     | CoreAdmin@123     |
| SW-IT               | ITAdmin       | ITAdmin@123       |
| SW-Sales-HR         | SalesHRAdmin  | SalesHRAdmin@123  |
| Switch1             | Sw1Admin      | Sw1Admin@123      |
| Switch2             | Sw2Admin      | Sw2Admin@123      |

> **Note:** These credentials are for this Packet Tracer lab environment only and are not intended for production use.

Unused physical ports on Router0's expansion module were administratively shut down.

### 5. Management & Monitoring

#### NTP

The Core Switch acts as the authoritative time source; all other devices sync to it. Accurate, consistent timestamps are essential for correlating Syslog events across devices during troubleshooting.

```cisco
! Core Switch
clock set 12:51:00 14 September 2026
clock timezone EET 2
ntp master 3

```

```cisco
! All other devices
ntp server 192.168.10.1

```

Verified sync:

```
Clock is synchronized, stratum 4, reference is 192.168.99.1

```

#### Syslog

All devices forward logs to a central collector, so events from the entire network can be reviewed in one place.

```cisco
service timestamps log datetime msec
logging host 192.168.50.10

```

Sample collected entries:

```
03.01.2026 12:01:51 PM   192.168.99.6   %LINEPROTO-5-UPDOWN: ...
*Sep 14, 13:06:33        192.168.99.2   %OSPF-5-ADJCHG: ...

```

#### SNMP

```cisco
snmp-server community MyReadOnly RO

```

Read-only access was chosen deliberately — a monitoring system needs to read device state, not modify it.

---

## Design Decisions & Rationale

### Why different VLAN IDs and subnets per site?

Reusing VLAN IDs or subnets across physically separate sites would cause overlapping networks, breaking OSPF's ability to route correctly. Each site gets a unique range even though the department names (IT, HR, Sales) repeat.

### Why SVIs at HQ but Router-on-a-Stick at branches?

Router-on-a-Stick funnels all inter-VLAN traffic through a single physical link, creating a potential bottleneck. That's a real concern at HQ (high traffic, many hosts) but negligible at a 3-host branch over a Gigabit link. Adding a Layer 3 switch at each branch would be over-engineering for the traffic volume involved.

### Why centralized DHCP instead of per-branch servers?

Single point of configuration, lower cost, and consistent policy across sites. This is the standard enterprise pattern. Per-branch DHCP is typically reserved for sites with unreliable WAN links where local resilience outweighs central control.

### Why is the MGMT VLAN separate with no user devices?

Management traffic (SSH, SNMP, NTP) is isolated from user traffic. Switches receive an IP on the MGMT VLAN so they are individually reachable as devices, while no user host is ever placed in that VLAN.

### Why are branches allowed to reach HQ but not each other?

This follows a hub-and-spoke security model. Branches need HQ resources (DHCP, shared services), but direct branch-to-branch communication provides little business value while widening the blast radius of a compromise. Cross-branch collaboration should route through HQ-hosted shared resources instead.

---

## Challenges & Troubleshooting

### 1. ACL applied in the wrong direction

**Symptom:** After configuring the branch-isolation ACL with `ip access-group ... out`, pings between branches still succeeded. The `permit ip any any` counter was incrementing while the `deny` statements showed zero matches.

**Root cause:** `out` filters traffic leaving the router *toward* the sub-interface's own VLAN. Traffic originating from branch hosts enters the router on that sub-interface and exits elsewhere, so the ACL never evaluated it.

**Fix:** Changed to `in`, which inspects traffic as it enters the router from the branch network — before any routing decision is made.

```
Before:  Reply from 192.168.230.10 ...          (traffic passed)
After:   Reply from 192.168.130.1: Destination host unreachable

```

### 2. Partial isolation instead of full isolation

**Symptom:** Only three `deny` statements were initially written (IT↔IT, HR↔HR, Sales↔Sales), which allowed cross-department traffic between branches (e.g., Branch A HR could still reach Branch B IT).

**Root cause:** Traffic that didn't match a specific pair fell through to `permit ip any any`.

**Fix:** Expanded to all 3 × 3 = 9 source/destination combinations for complete isolation.

### 3. Layer 2 module couldn't accept an IP address

**Symptom:** `ip address` and `ip nat outside` were both rejected with `Invalid input` on `Fa0/3/0`.

**Root cause:** The HWIC-4ESW expansion module presents Layer 2 switchports, not routed interfaces — the same reason a plain switchport can't take an IP without `no switchport`.

**Fix:** Configured the IP and NAT role on `interface vlan 1` instead, since all module ports belong to VLAN 1 by default.

### 4. NAT wouldn't accept an interface reference

**Symptom:** `ip nat inside source list 1 interface vlan 1 overload` was rejected across every naming variant tried (`vlan 1`, `vlan1`, `Vlan1`).

**Fix:** Used an explicit NAT pool containing the single public address instead of referencing the interface by name.

```cisco
ip nat pool ISP_POOL 203.0.113.1 203.0.113.1 netmask 255.255.255.252
ip nat inside source list 1 pool ISP_POOL overload

```

### 5. VTP client created a local VLAN unexpectedly

**Symptom:** `switchport access vlan 20` returned `% Access VLAN does not exist. Creating vlan 20` on SW-IT.

**Root cause:** `show vtp status` revealed the switch was still in **Server** mode (the default) with an empty domain name, so it wasn't receiving VLANs from the Core Switch.

**Fix:** Set the correct domain name and client mode. Once corrected, all five VLANs propagated automatically.

### 6. Console line had conflicting authentication methods

**Symptom:** Both `password ConsolePass` and `login local` were configured on the same line.

**Root cause:** These are mutually exclusive. `login local` takes precedence, making the standalone password silently inert.

**Fix:** Removed the redundant password line, leaving `login local` as the single authentication method.

### 7. NTP took an unusually long time to converge

**Symptom:** Clients reported `Clock is unsynchronized, stratum 16, no reference clock` for several minutes despite correct configuration and successful pings to the NTP master.

**Resolution:** Synchronization eventually completed on its own. Packet Tracer's NTP implementation converges considerably slower than real hardware. Configuration was verified correct via `show run | include ntp` and reachability testing before waiting it out.

### 8. DHCP failed across all VLANs simultaneously

**Symptom:** `DHCP request failed` on every client, even though pings to the DHCP server succeeded.

**Diagnostic reasoning:** A successful ping ruled out routing, trunking, and VLAN assignment as causes, isolating the problem to the DHCP service itself.

**Root cause:** The DHCP service had been configured with all pools but never switched **On** in the server's Services tab.

---

## Verification & Testing

| Test Command / Method Expected Result  |                                  |                                        |   |
| -------------------------------------- | -------------------------------- | -------------------------------------- | - |
| OSPF adjacency                         | `show ip route ospf`             | All remote networks learned            | ✅ |
| Default route propagation              | `show ip route` on branch router | `O*E2 0.0.0.0/0` present               | ✅ |
| EtherChannel bundling                  | `show etherchannel summary`      | `Po1(SU)` with both ports `(P)`        | ✅ |
| STP loop prevention                    | `show spanning-tree`             | Blocked port before EtherChannel       | ✅ |
| VTP propagation                        | `show vlan brief` on client      | All VLANs present without local config | ✅ |
| DHCP across sites                      | Client set to DHCP               | Correct IP from matching pool          | ✅ |
| Branch isolation (A→B)                 | `ping 192.168.230.10` from PC6   | 100% loss, host unreachable            | ✅ |
| Branch isolation (B→A)                 | `ping 192.168.130.10` from PC0   | 100% loss, host unreachable            | ✅ |
| HQ reachability from branch            | `ping 192.168.50.10` from PC6    | 0% loss                                | ✅ |
| NAT translation                        | `show ip nat translations`       | Inside local → inside global mapping   | ✅ |
| Internet simulation                    | `ping 203.0.113.2` from HQ host  | Replies received                       | ✅ |
| NTP sync                               | `show ntp status`                | `Clock is synchronized`                | ✅ |
| Syslog collection                      | Server Services → SYSLOG         | Entries from multiple devices          | ✅ |
| SNMP                                   | `show run \| include snmp`       | Community string present               | ✅ |

---

## Known Limitations

These are constraints of the simulation environment, not the design:

- **Object-groups unsupported.** In production IOS, the nine ACL `deny` statements could be collapsed using object-groups. Packet Tracer doesn't support them, so each source/destination pair is written explicitly.
- **`snmp-server location`** **/** **`contact`** **unsupported.** The IOS build in Packet Tracer only accepts `snmp-server community`. These are descriptive fields and their absence doesn't affect SNMP functionality.
- **`logging trap`** **unsupported.** The default logging level (`informational`) applied automatically and was confirmed via `show logging`.
- **NTP convergence is slow.** Real hardware syncs in seconds to a couple of minutes; the simulator took noticeably longer.
- **No IPv6.** Deliberately scoped out of this build.

---

## Future Improvements

- **FHRP (HSRP/VRRP)** with a redundant distribution switch for gateway redundancy — currently each VLAN has a single point of failure at its gateway.
- **Multi-area OSPF** — a single area is appropriate at this scale, but multi-area would demonstrate route summarization and LSA filtering as the network grows.
- **Centralized AAA (RADIUS/TACACS+)** to replace per-device local accounts, giving single-point credential management and per-admin audit trails.
- **Static NAT + DMZ** for publicly reachable services (web server), demonstrating inbound translation alongside the existing outbound PAT.
- **Per-device or role-based SNMP credentials** and SNMPv3 for authenticated, encrypted monitoring traffic.
- **Management-plane ACL** restricting VLAN 10 access to a specific admin workstation.

---

## Repository Contents

```
├── README.md
├── CCNA-Project.pkt          # Packet Tracer project file

```

---

*Built as a hands-on CCNA capstone. Every configuration choice in this project was reasoned through rather than copied — the troubleshooting section documents where the initial approach was wrong and what the actual root cause turned out to be.*