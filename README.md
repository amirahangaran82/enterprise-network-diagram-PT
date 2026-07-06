# Enterprise Campus Network - Cisco Packet Tracer Project

A full enterprise network build in Cisco Packet Tracer, covering six departments, a server farm, VoIP, wireless, and a two-tier collapsed core with redundancy baked into almost every layer.

## Why I built this

I'm finishing up a CCNA and wanted something bigger than lab exercises to show for it, a project that actually looks like a network an SMB or branch office would run day to day, not just a topology with three routers and a "hello world" OSPF config. This is my attempt at that. It's not perfect (see the [Known Limitations](#known-limitations--roadmap) section, I'm not going to pretend otherwise), but every piece in here was configured, tested, and broken at least once before it worked.

If you're reviewing this as part of a job application: thanks for taking the time to look through it. I tried to document my reasoning, not just dump configs, so you can see how I think about network design, not just whether I can copy commands from a guide.

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
- [Known Limitations / Roadmap](#known-limitations--roadmap)


## Topology Overview

![Network Topology](topology.png)

The network is split into three functional layers plus a server room:

- **Access Layer:** Six 2960-24TT switches, one per department (Management, HR, Marketing, IT, Accounting, R&D). Each one carries end devices, PCs, IP phones, laptops, a wireless AP, and in some cases a printer.
- **Distribution/Core Layer:** Two 3650 switches (CORE-SW1 and CORE-SW2) doing double duty as both distribution and core. They handle inter-VLAN routing, OSPF, and HSRP, and they're connected to each other with a Layer 3 EtherChannel.
- **Router Layer:** An ISR4331 (R1) sitting between the core and the ISP router, handling PAT and advertising a default route into OSPF.
- **Server Room:** A dedicated switch hosting DHCP, DNS, FTP, Syslog, NTP, Web/Email, RADIUS/TACACS+, a WLC, and a VoIP-PBX.

Every access switch is dual-homed with one uplink to CORE-SW1 and one to CORE-SW2, so there's no single point of failure between an access switch and the rest of the network.

## Is This Spine-Leaf?

Short answer: not really, though it's easy to mistake it for one at first glance.

The physical layout does resemble spine-leaf. Every access switch connects to both core switches, so you get that same "fan out" pattern where a leaf never has to hop through another leaf to reach the fabric. That's where the similarity ends, though. A real spine-leaf fabric is Layer 3 all the way down, every leaf-to-spine link is a routed point-to-point link, and redundancy comes from ECMP across BGP or OSPF, with no Spanning Tree involved because there's no Layer 2 loop to protect against.

What I actually built here is a **two-tier collapsed core**: the access-to-core links are Layer 2 trunks, RSTP is running underneath them, and gateway redundancy comes from HSRP rather than ECMP. The two 3650s are effectively doing the job that a separate distribution layer and core layer would normally split between them, which is a pretty standard pattern for a network of this size where a dedicated core layer would be overkill.

So: spine-leaf shaped, but built on classic Cisco hierarchical principles rather than a true L3 fabric. I think that's the right call for a network this size, spine-leaf really earns its keep in data centers with dozens of leaf switches and heavy east-west traffic, not a six-department campus network.

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

Every item below has a reason behind it. I tried to avoid configuring something just because a checklist somewhere said "enterprise networks do this."

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
| HSRP with split groups | A single active gateway per VLAN means one core switch does all the routing while the other sits idle | Both core switches actively forward traffic, and gateway failure is transparent to end devices |
| L3 EtherChannel between the two 3650s | A single link between the cores is a single point of failure and a bandwidth bottleneck | Combines multiple physical links into one logical, higher-bandwidth, resilient connection |
| OSPF with passive interfaces on access-facing SVIs | Access-facing VLANs don't have OSPF neighbors, so there's no reason to send Hellos out of them | Reduces unnecessary control-plane traffic and shrinks the OSPF attack surface |
| OSPF reference bandwidth changed to 10 Gbps | The default reference bandwidth (100 Mbps) makes every link over 100 Mbps look identical in cost, which is meaningless on modern gigabit and 10G links | Path cost actually reflects real link speed differences again |
| IP helper on voice VLAN interfaces | IP phones are on a different subnet than the DHCP server | DHCP broadcasts get relayed correctly so phones can lease an address and register with the PBX |
| Trunk ports down to access switches | Multiple VLANs need to traverse a single physical link to each access switch | One physical link carries every department's traffic to its access switch |

### Router Layer (ISR 4331)

| Technology | Why | Effect |
|---|---|---|
| OSPF, advertising a default route | Internal switches need a way out to the internet without a static route on every device | A single default-route advertisement handles internet-bound traffic for the entire internal network |
| OSPF reference bandwidth changed to 10 Gbps | Same reasoning as the core layer, keeps cost calculations meaningful | Consistent path selection across the whole OSPF domain |
| PAT (NAT overload) using the router's own interface IP | Only one public IP is available from the ISP | Every internal host shares that single public address for outbound traffic |
| Loopback interface for SSH | A management IP tied to a single physical interface goes down with that interface | Management access stays reachable as long as the router has any active path, since loopbacks never go down on their own |

### Server Room

The server room isn't really a network layer, it's the reason the network exists in the first place. It hosts:

- **VoIP** (PBX + phone registration)
- **RADIUS and TACACS+** for authentication (RADIUS was meant to drive 802.1X on the WLC, but Packet Tracer's 802.1X support didn't cooperate, so wireless auth falls back to standard RADIUS instead)
- **DHCP**
- **Web and Email**
- **NTP** (the actual time source the rest of the network syncs to)
- **DNS, FTP, and Syslog**

## High Availability and Reliability

This was one of the main things I wanted to prove out with this project, that redundancy shows up in the design itself, not just as a single fallback link somewhere.

- **Dual core switches, both active.** HSRP groups are split so that CORE-SW1 and CORE-SW2 each own the active gateway role for roughly half the VLANs. If either one fails, the other takes over every VLAN's gateway duties automatically, and under normal conditions neither one sits idle.
- **L3 EtherChannel between the cores.** Instead of a single link (and a single point of failure) between CORE-SW1 and CORE-SW2, multiple physical links are bundled into one logical connection. Lose one physical link and OSPF and HSRP don't even notice.
- **Every access switch is dual-homed.** Each department switch has an uplink to both core switches. Lose a link, or lose an entire core switch, and the department still has a path out.
- **OSPF handles the actual rerouting.** If a link or a device fails, OSPF recalculates and traffic reroutes without anyone touching a config.
- **Centralized NTP and Syslog.** Not redundancy in the failover sense, but it means that when something does fail, there's an accurate, time-synced record of what happened and when, which matters just as much during an actual incident.

## Known Limitations / Roadmap

Being upfront about what isn't finished yet:

- **Standard ACLs restricting SSH access** are planned for the access, distribution, and router layers, but not yet implemented.
- **Extended ACLs** at the distribution and router layers are still on the list.
- A **help desk privilege level (level 11)** limited to interface status checks, interface shut/no shut, and ping was attempted but dropped. Packet Tracer's IOS emulation doesn't support the parser view commands needed to actually build a restricted CLI view, so this is a Packet Tracer limitation rather than a design choice.
- Loop Guard on the fiber connection between the two 3650s was considered and intentionally skipped, since that link is a Layer 3 EtherChannel and doesn't have the Layer 2 loop conditions Loop Guard protects against (and Packet Tracer doesn't support the command anyway).

