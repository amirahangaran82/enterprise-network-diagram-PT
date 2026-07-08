# Enterprise Campus Network - Cisco Packet Tracer Project

An enterprise network build in Cisco Packet Tracer, covering six departments, a server farm, VoIP, wireless, and a two-tier collapsed core with redundancy baked into almost every layer.

## Why I built this

I wanted a bigger project than isolated lab exercises, something where I could practice designing, configuring, troubleshooting, and documenting a network end to end instead of drilling one technology at a time.

Thank you for taking the time to look through my project!

## Table of Contents

- [Topology Overview](#topology-overview)
- [Is This Spine-Leaf?](#is-this-spine-leaf)
- [IP Addressing and VLAN Scheme](#ip-addressing-and-vlan-scheme)
- [Technologies Implemented](#technologies-implemented)
  - [Access Layer (2960 Switches)](#access-layer-2960-switches)
  - [Distribution/Core Layer (3650 Switches)](#distributioncore-layer-3650-switches)
  - [Router Layer (ISR 4331)](#router-layer-isr-4331)
  - [Server Room](#server-room)
- [High Availability and Reliability](#high-availability-and-reliability)
- [Access Control Lists](#access-control-lists)
- [Lessons Learned / Packet Tracer Quirks](#lessons-learned--packet-tracer-quirks)
- [Known Limitations / Roadmap](#known-limitations--roadmap)
- [Tools Used](#tools-used)
- [Opening This Project](#opening-this-project)

## Topology Overview

<img width="3691" height="2885" alt="PICTURE" src="https://github.com/user-attachments/assets/fff21852-2cf7-47a7-8037-b26f934a711a" />


The network is split into three functional layers plus a server room:

- **Access Layer:** Six 2960-24TT switches, one per department (Management, HR, Marketing, IT, Accounting, R&D). Each one carries end devices, PCs, IP phones, laptops, a wireless AP, and a printer.
<img width="2987" height="487" alt="Access Layer ONLY" src="https://github.com/user-attachments/assets/5bd5c706-0956-472f-b73d-6cf216aaa41d" />
- **Distribution/Core Layer:** Two 3650 switches (CORE-SW1 and CORE-SW2) doing double duty as both distribution and core. They handle inter-VLAN routing, OSPF, and HSRP, and they're connected to each other with a Layer 3 EtherChannel.
- **Router Layer:** An ISR4331 (R1) sitting between the core and the ISP router, handling PAT and advertising a default route into OSPF.
<img width="2128" height="495" alt="Distribution + Router" src="https://github.com/user-attachments/assets/f9588af3-11cd-412d-b630-1c05ce21a975" />
- **Server Room:** A dedicated switch hosting DHCP, DNS, FTP, Syslog, NTP, Web/Email, RADIUS/TACACS+ servers, a WLC, and a VoIP-PBX (2811 Router).
<img width="437" height="488" alt="Server room" src="https://github.com/user-attachments/assets/c6887884-41bd-4147-8a35-8a4e6bf379d5" />


Every access switch is dual-homed with one uplink to CORE-SW1 and one to CORE-SW2, so there's no single point of failure between an access switch and the rest of the network.

## Is This Spine-Leaf?

It's actually a two-tier collapsed core design: Layer 2 access, Layer 3 routing and HSRP at the core, tied together with OSPF. I looked at spine-leaf while planning this and picked the collapsed core model instead, since it fits a six-department network better than a design built for large data center fabrics.

## IP Addressing and VLAN Scheme

### Department VLANs

| VLAN | Purpose | Data Subnet | Voice VLAN | Voice Subnet |
|---|---|---|---|---|
| 10 | Management | 192.168.10.0/24 | 11 | 192.168.11.0/24 |
| 20 | HR | 192.168.20.0/24 | 21 | 192.168.21.0/24 |
| 30 | Marketing | 192.168.30.0/24 | 31 | 192.168.31.0/24 |
| 40 | IT | 192.168.40.0/24 | 41 | 192.168.41.0/24 |
| 50 | Accounting | 192.168.50.0/24 | 51 | 192.168.51.0/24 |
| 60 | R&D | 192.168.60.0/24 | 61 | 192.168.61.0/24 |
| 70 | Server Room | 192.168.70.0/24 | - | Statically assigned |
| 93 | Native (unused, security only) | - | - | - |
| 100 | Guest Wi-Fi | 192.168.100.0/24 | - | - |
| 500 | Management (SSH/mgmt access) | 172.16.50.0/25 | - | - |

### HSRP Virtual Gateways

| VLAN | HSRP Virtual IP |
|---|---|
| 10 | 192.168.10.3 |
| 20 | 192.168.20.3 |
| 30 | 192.168.30.3 |
| 40 | 192.168.40.3 |
| 50 | 192.168.50.3 |
| 60 | 192.168.60.3 |
| 70 | 192.168.70.3 |
| 100 | 192.168.100.3 |
| 500 | 172.16.50.3 |

HSRP groups are split so that each core switch is the active gateway for roughly half the VLANs. That way both cores are actually forwarding traffic under normal conditions instead of one of them sitting idle as a cold standby.

### Infrastructure Links

| Link | Subnet | Notes |
|---|---|---|
| R1 to ISP | 5.239.0.252/30 | Public-facing WAN link |
| R1 to CORE-SW1 | 192.168.255.0/30 | Point-to-point routed link |
| R1 to CORE-SW2 | 192.168.255.4/30 | Point-to-point routed link |
| CORE-SW1 <-> CORE-SW2 | 172.16.50.128/30 | L3 EtherChannel (Po1) |

### OSPF Router IDs

| Device | Router ID |
|---|---|
| R1 | 3.3.3.3 |
| CORE-SW1 | 1.1.1.1 |
| CORE-SW2 | 2.2.2.2 |

## Technologies Implemented

I tried to implement every technology an enterprise network might need (and what Packet Tracer could handle).

### Access Layer (2960 Switches)

| Technology | Why | Effect |
|---|---|---|
| RSTP + PortFast + BPDU Guard | Faster convergence than legacy STP, and access ports shouldn't be waiting through listening/learning states | Endpoints come up almost instantly, and a rogue switch plugged into an access port gets the port shut down instead of injecting BPDUs into the topology |
| VTP disabled | One misconfigured switch shouldn't be able to wipe out VLAN databases network-wide | Each switch keeps its own independent VLAN database |
| VLANs + Voice VLANs per department | Keep broadcast domains small and separate departmental traffic | Smaller failure domains, and phones get tagged into a separate VLAN from the PC sitting next to them |
| Native VLAN changed from 1 to 93 | VLAN 1 is the default target for VLAN hopping attacks | Removes the most commonly assumed native VLAN, closing off a well-known attack vector |
| DHCP Snooping + rate limiting, Option 82 disabled | Stop rogue DHCP servers and DHCP starvation attacks | Untrusted ports can't hand out leases, and excessive DHCP traffic gets rate-limited before it becomes a problem |
| Dynamic ARP Inspection + rate limiting | Stop ARP spoofing / man-in-the-middle attacks | Only ARP replies matching the DHCP snooping binding table get forwarded |
| SSH access + login/enable via TACACS+ | Telnet sends credentials in plaintext, and local-only accounts don't scale or audit well | Encrypted remote management with centralized authentication and accounting |
| Dedicated management VLAN (500) + interface IP | Management traffic shouldn't ride the same VLAN as user data | Out-of-band-style separation for admin access, even without true OOB hardware |
| Standard ACL on VTY lines, permitting only IT and Server Room subnets | SSH should be reachable, not open to every department | Only 192.168.40.0/24 and 192.168.70.0/24 can even attempt to SSH in, everyone else is denied before the login prompt |
| Syslog server assigned | Local switch buffers get wiped on reboot | Centralized, persistent logging for troubleshooting and audits |
| LLDP enabled | Vendor-neutral neighbor discovery works even in mixed-vendor environments | Easier topology verification and troubleshooting |
| NTP configured | Log timestamps are useless if every device's clock disagrees | Consistent timestamps across every log source |
| Trunk ports to core, with a Guest VLAN and BlackHole VLAN | Restrict which VLANs actually need to traverse each trunk, isolate guest traffic, and neutralize unused ports | Smaller attack surface and cleaner troubleshooting since trunks only carry the VLANs they need to |
| DTP disabled (switchport nonegotiate) | Dynamic trunk negotiation is a known VLAN hopping vector | Ports stick to their configured mode and can't be tricked into trunking |
| Unused ports shut down and set to access mode | An open, unused port is a free entry point for anyone with physical access | Reduces the physical attack surface significantly |

### Distribution/Core Layer (3650 Switches)

| Technology | Why | Effect |
|---|---|---|
| IP routing enabled | These switches need to route between VLANs, not just switch within them | Turns the 3650s into full Layer 3 devices |
| HSRP with split groups + interface tracking | A single active gateway per VLAN means one core switch does all the routing while the other sits idle, and priority alone doesn't help if the active switch's uplink dies while the switch itself stays up | Each HSRP group tracks the uplink toward R1 (and toward the server room switch for VLAN 500), so priority drops and failover triggers the moment the active switch loses its path out, not just when the switch itself goes down |
| L3 EtherChannel between the two 3650s | A single link between the cores is a single point of failure and a bandwidth bottleneck | Combines multiple physical links into one logical, higher-bandwidth, resilient connection |
| OSPF with passive interfaces on access-facing SVIs | Access-facing VLANs don't have OSPF neighbors, so there's no reason to send Hellos out of them | Reduces unnecessary control-plane traffic and shrinks the OSPF attack surface |
| OSPF reference bandwidth changed to 10 Gbps | The default reference bandwidth (100 Mbps) makes every link over 100 Mbps look identical in cost, which is meaningless on modern gigabit and 10G links | Path cost actually reflects real link speed differences again |
| IP helper on voice VLAN interfaces | IP phones are on a different subnet than the DHCP server | DHCP broadcasts get relayed correctly so phones can lease an address and register with the PBX |
| Trunk ports down to access switches | Multiple VLANs need to traverse a single physical link to each access switch | One physical link carries every department's traffic to its access switch |
| Standard ACL on VTY lines + Extended ACL on the Guest VLAN SVI | Same SSH restriction as the access layer, plus guests need to be walled off from internal subnets | Only IT/Server Room can SSH into the core switches, and guest traffic can only resolve DNS and reach the internet |

### Router Layer (ISR 4331)

| Technology | Why | Effect |
|---|---|---|
| OSPF, advertising a default route | Internal switches need a way out to the internet without a static route on every device | A single default-route advertisement handles internet-bound traffic for the entire internal network |
| OSPF reference bandwidth changed to 10 Gbps | Same reasoning as the core layer, keeps cost calculations meaningful | Consistent path selection across the whole OSPF domain |
| PAT (NAT overload) using the router's own interface IP | Only one public IP is available from the ISP | Every internal host shares that single public address for outbound traffic |
| Loopback interface for SSH | A management IP tied to a single physical interface goes down with that interface | Management access stays reachable as long as the router has any active path, since loopbacks never go down on their own |
| Standard ACL on VTY lines + Extended ACL on the ISP-facing interface | SSH shouldn't be open past IT/Server Room, and nothing on the internet should be able to initiate a connection inward | SSH is locked to internal admin subnets, and the only inbound internet traffic allowed is replies to connections the network itself started |

### Server Room

The server room isn't really a network layer, it's the reason the network exists in the first place. It hosts:

- **VoIP**, running as Cisco CME (Call Manager Express) on the PBX router. It handles registration and call control for 19 IP phones, with each department getting its own extension block (Management is 11xx, HR is 21xx, Marketing 31xx, and so on), and a dedicated sub-interface and DHCP pool per voice VLAN so phones get an address and their default gateway from the PBX itself.
- **RADIUS and TACACS+** for authentication (RADIUS was meant to drive 802.1X on the WLC, but Packet Tracer's 802.1X support didn't cooperate, so wireless auth falls back to standard RADIUS instead)
- **DHCP**
- **Web and Email**
- **NTP** (the actual time source the rest of the network syncs to)
- **DNS, FTP, and Syslog**

## High Availability and Reliability

Both High-availability and Reliability are important when designing a network since shut downs are costly for a company both money and time wise.

- **Dual core switches, both active.** HSRP groups are split so that CORE-SW1 owns VLANs 10-30,70,93,100 and CORE-SW2 owns VLANs 40-60 active gateway role. If either one fails, the other takes over every VLAN's gateway duties automatically, and under normal conditions neither one sits idle.
- **L3 EtherChannel between the cores.** Instead of a single link (and a single point of failure) between CORE-SW1 and CORE-SW2, multiple physical links are bundled into one logical connection. Lose one physical link and OSPF and HSRP don't even notice.
- **Every access switch is dual-homed.** Each department switch has an uplink to both core switches. Lose a link, or lose an entire core switch, and the department still has a path out.
- **OSPF handles the actual rerouting.** If a link or a device fails, OSPF recalculates and traffic reroutes without anyone touching a config.
- **Centralized NTP and Syslog.** Not redundancy in the failover sense, but it means that when something does fail, there's an accurate, time-synced record of what happened and when, which matters just as much during an actual incident.

## Access Control Lists

Three ACLs handle the actual traffic filtering in this network. Unfortunately I have no experince with real networks so I didn't know what exactly to block, but I applied 3 ACLS where each does one thing:
1- Only IT and Server room's subnets can SSH to the network equipment.
2- The Guest Wi-Fi can only access the internet and DNS server.
3- Unsolicited connections from outside are blocked.

**1. Management access (Standard ACL, applied to VTY lines on both core switches and R1)**

```
ip access-list standard 10
 remark Only IT and Server Room can SSH in
 10 permit 192.168.40.0 0.0.0.255
 20 permit 192.168.70.0 0.0.0.255
 30 deny any
!
line vty 0 15
 ip access-class 10 in
```

**2. Guest Wi-Fi isolation (Extended ACL, applied inbound on the VLAN 100 SVI, identical on both core switches)**

```
ip access-list extended guest_vlan
 remark Guests get DNS and the internet, nothing else
 10 permit tcp 192.168.100.0 0.0.0.255 host 192.168.70.8 eq 53
 11 permit udp 192.168.100.0 0.0.0.255 host 192.168.70.8 eq 53
 20 deny ip 192.168.100.0 0.0.0.255 192.168.0.0 0.0.255.255
 30 deny ip 192.168.100.0 0.0.0.255 172.16.50.0 0.0.0.127
 40 permit ip 192.168.100.0 0.0.0.255 any
!
interface vlan 100
 ip access-group guest_vlan in
```

**3. Outside-facing lockdown (Extended ACL, applied inbound on R1's ISP-facing interface)**

```
ip access-list extended block_outside
 remark Anti-spoofing, nobody outside should claim our own address space
 10 deny ip 192.168.0.0 0.0.255.255 any
 20 deny ip 172.16.0.0 0.15.255.255 any
 30 deny ip 10.0.0.0 0.255.255.255 any
 40 deny ip 127.0.0.0 0.255.255.255 any
 remark Allow only replies to connections started from inside
 50 permit icmp any any echo-reply
 60 permit icmp any any unreachable
 80 permit tcp any any established
 90 deny ip any any
!
interface g0/0/0
 ip access-group block_outside in
```

## Lessons Learned / Packet Tracer Quirks

A few things I only figured out by breaking them first, worth documenting so I don't relearn them the hard way next time:

- **AAA breaks on L3 switches.** L3 switches and the router authenticate fine right after configuration, but reopening the saved project shows a "Login Invalid" error on the AAA-authenticated lines. I had to switch from AAA to local login.
- **NTP needs patience and a real server.** Packet Tracer takes about 5 minutes to actually sync a client's clock after pointing it at an NTP source, and it only supports a single NTP server per device (no backup source, which is a real limitation for a production design). It also only accepts an actual Server object as the time source, a router or L3 switch acting as an NTP server doesn't work.
- **WLC and LAP ports have to be trunks, but DHCP for wireless clients only worked once I put the DHCP pool on the trunk's native VLAN.** Access ports broke wireless client traffic through Spanning Tree, and a straight trunk port left the APs unable to lease an address at all until the pool sat on the native VLAN instead of a tagged one.
- **BPDU Guard and IP phones don't mix.** Ports connecting to IP phones needed BPDU Guard disabled specifically, since the phones tag frames with dot1q and that alone is enough to trip Packet Tracer's BPDU Guard "port inconsistent" error.

## Known Limitations / Roadmap

Being upfront about what isn't finished, or isn't fully enforceable in Packet Tracer:

- A **help desk privilege level (level 11)** limited to interface status checks, interface shut/no shut, and ping was attempted but dropped. Packet Tracer's IOS emulation doesn't support the parser view commands needed to actually build a restricted CLI view, so this is a Packet Tracer limitation rather than a design choice.
- Loop Guard on the fiber connection between the two 3650s was considered and intentionally skipped, since that link is a Layer 3 EtherChannel and doesn't have the Layer 2 loop conditions Loop Guard protects against (and Packet Tracer doesn't support the command anyway).
- The outside-facing ACL's original design logged denied traffic and matched on ICMP time-exceeded, but Packet Tracer doesn't support the `log` or `time-exceeded` keywords, so those lines were dropped from the working config. On real hardware I'd add both back.
- No redundant NTP source, Packet Tracer only allows one NTP server per device, so the NTP server itself is a single point of failure for time sync network-wide. On real hardware this would get a second stratum source.

## What I Learned

- Tracing a VLAN communication issue back to a missing trunk-allowed VLAN, instead of assuming something bigger was broken.
- Fixing a DHCP relay pointed at the wrong subnet, which was why phones on one VLAN weren't getting addresses.
- Watching HSRP actually fail over, and learning to tell a real failover from a misconfigured priority value.
- Telling a genuine misconfiguration apart from a Packet Tracer limitation, which took a lot of forum reading and repeated testing.
- Realizing how much documentation matters.

## Tools Used

- Cisco Packet Tracer
- Microsoft Visio

