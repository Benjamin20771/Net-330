# NET-330: Week 06

## Lab 6-1: Static NAT

Simple two-router topology to learn the basic static NAT pattern before combining it with PAT in Lab 6-3. A web server sits behind R1 on a private address; the goal is to "masquerade" it behind a public address so a PC on R0's network can reach it.

### Addressing

| Device | Interface | IP | Mask |
|---|---|---|---|
| R1 | FastEthernet0/0 | 10.0.0.1 | 255.0.0.0 |
| R1 | Serial0/0/0 | 20.0.0.2 | 255.0.0.0 |
| R0 | FastEthernet0/0 | 30.0.0.1 | 255.0.0.0 |
| R0 | Serial0/0/0 | 20.0.0.1 | 255.0.0.0 |
| Web Server (behind R1) | — | 10.0.0.2 | 255.0.0.0 |
| Static NAT public address | — | 50.0.0.1 | — |

### Step 1: Configure Router Interfaces

**R1:**
```
Router>enable
Router#config terminal
Router(config)#hostname R1
R1(config)#interface fastethernet 0/0
R1(config-if)#ip address 10.0.0.1 255.0.0.0
R1(config-if)#no shutdown
R1(config-if)#exit
R1(config)#interface serial 0/0/0
R1(config-if)#ip address 20.0.0.2 255.0.0.0
R1(config-if)#no shutdown
R1(config-if)#exit
```

**R0:**
```
Router>enable
Router#config terminal
Router(config)#hostname R0
R0(config)#interface fastethernet 0/0
R0(config-if)#ip address 30.0.0.1 255.0.0.0
R0(config-if)#no shutdown
R0(config-if)#exit
R0(config)#interface serial 0/0/0
R0(config-if)#ip address 20.0.0.1 255.0.0.0
R0(config-if)#clock rate 64000
R0(config-if)#bandwidth 64
R0(config-if)#no shutdown
R0(config-if)#exit
```

**Info** `clock rate` only goes on whichever end of a serial link is acting as DCE (here, R0) so setting it on the wrong end, or on both ends, throws an error. Packet Tracer's serial cable icon shows which end is DCE if it's not obvious.

### Step 2: Configure Routing

```
! R1
R1(config)#ip route 30.0.0.0 255.0.0.0 20.0.0.1

! R0
R0(config)#ip route 50.0.0.0 255.0.0.0 20.0.0.2
```

Pinged 20.0.0.1 and 30.0.0.1 from the PCs to confirm basic reachability first. As expected, there's no route to 10.0.0.2 anywhere in R0's table. A ping straight to the web server's real private address fails, which is the whole reason NAT is needed here: the private 10.0.0.0/8 network was never meant to be reachable directly from outside.

### Step 3: Configure Static NAT on R1

Define inside/outside:
```
R1(config)#interface fastEthernet 0/0
R1(config-if)#ip nat inside
R1(config-if)#exit
R1(config)#interface serial 0/0/0
R1(config-if)#ip nat outside
R1(config-if)#exit
```

Create the static mapping:
```
R1(config)#ip nat inside source static 10.0.0.2 50.0.0.1
```

### Tech Journal — Lab 6-1 Static NAT

- **Setting NAT interfaces**: `ip nat inside` goes on the interface facing the private network (R1's Fa0/0, facing the web server); `ip nat outside` goes on the interface facing the public/untrusted network (R1's Serial0/0/0, facing R0). NAT only translates traffic that crosses from an inside-marked interface to an outside-marked one (or vice versa)
- **Static NAT command**: `ip nat inside source static <inside-local-ip> <inside-global-ip>` — e.g. `ip nat inside source static 10.0.0.2 50.0.0.1`. This is a permanent, one-to-one mapping: the private 10.0.0.2 always appears as 50.0.0.1 to anything outside, and the mapping exists in the NAT table whether or not there's active traffic (unlike dynamic/PAT entries, which age out).

---

## Lab 6-2: NAT PAT (Port Address Translation)

### Addressing

| Device | Interface | IP | Mask |
|---|---|---|---|
| R1 (NAT device) | FastEthernet0/0 | 192.168.0.1 | 255.255.255.0 |
| R1 | Serial0/0/0 | 30.0.0.1 | 255.0.0.0 |
| R2 | FastEthernet0/0 | 20.0.0.1 | 255.0.0.0 |
| R2 | Serial0/0/0 | 30.0.0.2 | 255.0.0.0 |
| PAT shared public address | — | 30.0.0.120 | 255.0.0.0 |

Internal network: 192.168.0.0/24, with multiple PCs that will all share the single public address 30.0.0.120.

### Step 1: Configure Router Interfaces

Addresses assigned per the table above on R1 and R2 (same pattern as Lab 6-1 — `interface`, `ip address`, `no shutdown`).

### Step 2: Configure Routing

R1 just needs a default route (gateway of last resort) pointing at R2, rather than specific routes, since everything beyond R2 is "the rest of the internet" from R1's perspective:

```
R1(config)#ip route 0.0.0.0 0.0.0.0 30.0.0.2
```

At this point, pinging 20.0.0.2 from a 192.168.0.0/24 PC fails as it has no NAT yet, and R2 has no idea how to route back to a private 192.168.0.0/24 address even if the ping did get through.

### Step 3: Configure PAT on R1

Define inside/outside interfaces (same concept as Lab 6-1):
```
R1(config)#interface fastEthernet 0/0
R1(config-if)#ip nat inside
R1(config-if)#exit
R1(config)#interface serial 0/0/0
R1(config-if)#ip nat outside
R1(config-if)#exit
```

Create a NAT pool holding the single public address available:
```
R1(config)#ip nat pool test 30.0.0.120 30.0.0.120 netmask 255.0.0.0
```

Create an access-list defining which internal addresses are allowed to use that pool:
```
R1(config)#access-list 1 permit 192.168.0.0 0.0.0.255
```

Tie the ACL and the pool together with `overload`, which is what turns this into PAT instead of plain one-to-one dynamic NAT:
```
R1(config)#ip nat inside source list 1 pool test overload
```

### Step 4: Verification

With PAT running, multiple PCs on the 192.168.0.0/24 network were all able to load the web page hosted on the server at 20.0.0.2 at the same time, despite only one public address existing.

```
R1#show ip nat translations
Pro  Inside global     Inside local       Outside local      Outside global
icmp 30.0.0.120:1        192.168.0.7:1      20.0.0.2:1         20.0.0.2:1
icmp 30.0.0.120:2        192.168.0.7:2      20.0.0.2:2         20.0.0.2:2
icmp 30.0.0.120:3        192.168.0.7:3      20.0.0.2:3         20.0.0.2:3
icmp 30.0.0.120:4        192.168.0.7:4      20.0.0.2:4         20.0.0.2:4
```

Every row shares the same inside global address (30.0.0.120)

### Tech Journal — Lab 6-2 NAT PAT

- `ip nat pool <name> <start-ip> <end-ip> netmask <mask>` — defines the pool of public addresses available for translation. Here the pool is artificially restricted to exactly one address (30.0.0.120–30.0.0.120) to force overload/PAT behavior, since a pool with only one address can't do one-to-one dynamic NAT for more than one host at a time.
- `access-list 1 permit 192.168.0.0 0.0.0.255` — defines which inside source addresses are eligible for translation through the pool. Without this, R1 has no way to know which traffic should be NATed.
- `ip nat inside source list 1 pool test overload` — the command that ties it all together: traffic sourced from anything matching access-list 1, going from an inside interface to an outside interface, gets translated to an address from pool `test`, and `overload` allows many inside hosts to share that single address simultaneously by distinguishing sessions via port number (theoretically up to ~64,000 concurrent translations per public address).
- `show ip nat translations` is the verification command — with real PAT running, every row shares the same inside global address but a different port, which is the visual giveaway that overload/PAT is active rather than static or plain dynamic NAT.

---

## Lab 6-3: PAT + Static NAT on the Champlain/CNCS Topology

Combines both of the previous two labs' techniques onto one router in a larger, realistic topology: **PAT (overload)**, same concept as Lab 6-2, so Foster and Skiff PCs can reach the Burlington Telecom server out on the "internet," and **static NAT**, same concept as Lab 6-1, so the Pub Web Server can be reached from outside. The NAT boundary is the **CC Border Router** 

### Reference: Subnet Table

| Network | CIDR | Role |
|---|---|---|
| BT Server Net | 153.104.18.0/24 | Outside — Burlington Telecom's server segment |
| CC-BT Net | 219.93.144.0/24 | Outside — transit link between Champlain and Burlington Telecom |
| CC Backbone | 192.168.100.0/24 | Inside — transit link connecting CC Border Router to the three distribution routers |
| Skiff | 192.168.1.0/24 | Inside — Skiff LAN |
| Foster | 192.168.3.0/24 | Inside — Foster LAN |
| IDC | 192.168.7.0/24 | Inside — Ireland Data Center LAN (Internal SRV + Pub Web SRV) |

**Device/Interface Addressing:**

| Device | Interface | IP | Facing |
|---|---|---|---|
| Burlington Telecom Router | Fa0/0 | 219.93.144.2 | CC-BT Net |
| Burlington Telecom Router | Fa0/1 | 153.104.18.1 | BT Server Net |
| CC Border Router | Fa0/0 | 192.168.100.1 | CC Backbone (inside) |
| CC Border Router | Fa0/1 | 219.93.144.1 | CC-BT Net (outside) |
| Foster Distribution Router | Gi0/0 | 192.168.100.2 | CC Backbone |
| Foster Distribution Router | Gi0/1 | 192.168.3.1 | Foster LAN |
| Ireland Data Center Router | Gi0/0 | 192.168.100.3 | CC Backbone |
| Ireland Data Center Router | Gi0/1 | 192.168.7.1 | IDC LAN |
| Skiff Distribution Router | Gi0/0 | 192.168.100.4 | CC Backbone |
| Skiff Distribution Router | Gi0/1 | 192.168.1.1 | Skiff LAN |

**Host Addressing (assigned, starting file had none set):**

| Device | IP | Mask | Gateway |
|---|---|---|---|
| Foster PC1 | 192.168.3.2 | 255.255.255.0 | 192.168.3.1 |
| Foster PC2 | 192.168.3.3 | 255.255.255.0 | 192.168.3.1 |
| Skiff PC1 | 192.168.1.2 | 255.255.255.0 | 192.168.1.1 |
| Skiff PC2 | 192.168.1.3 | 255.255.255.0 | 192.168.1.1 |
| Internal SRV | 192.168.7.2 | 255.255.255.0 | 192.168.7.1 |
| Pub Web SRV | 192.168.7.3 | 255.255.255.0 | 192.168.7.1 |
| Burlington Telecom Server | 153.104.18.2 | 255.255.255.0 | 153.104.18.1 |

### Step 1: Routing

Static NAT and PAT are both useless if packets can't find their way to the NAT router in the first place. Each distribution router just needs a default route pointing at CC Border:

```
enable
config terminal
ip route 0.0.0.0 0.0.0.0 192.168.100.1
end
copy run start
```

CC Border Router needs specific routes back to each inside subnet, plus a route out to the BT Server Net:

```
enable
config terminal
ip route 192.168.1.0 255.255.255.0 192.168.100.4
ip route 192.168.3.0 255.255.255.0 192.168.100.2
ip route 192.168.7.0 255.255.255.0 192.168.100.3
ip route 153.104.18.0 255.255.255.0 219.93.144.2
end
copy run start
```

The Burlington Telecom Router needs **no routes at all** back toward the 192.168.x.x networks it only ever sees post-NAT traffic sourced from its own directly-connected 219.93.144.0/24. That's the actual point of PAT: it hides the entire inside addressing scheme from the outside world, the same lesson as Lab 6-2's R2 never needing to know about 192.168.0.0/24.

### Step 2: NAT Configuration (CC Border Router)

```
enable
config terminal

interface FastEthernet0/0
 ip nat inside
 exit

interface FastEthernet0/1
 ip nat outside
 exit

access-list 1 permit 192.168.3.0 0.0.0.255
access-list 1 permit 192.168.1.0 0.0.0.255

ip nat inside source list 1 interface FastEthernet0/1 overload

ip nat inside source static 192.168.7.3 219.93.144.10

end
copy run start
```

- `ip nat inside` / `ip nat outside` mark which side of the router is private vs. public — NAT only translates traffic crossing between an inside-marked and outside-marked interface.
- Access-list 1 only matches Foster and Skiff's subnets — that's deliberate. IDC isn't included because its only outbound-facing device (Pub Web SRV) gets its own static mapping instead.
- `overload` is what makes this PAT instead of plain dynamic NAT, same as Lab 6-2 — many inside hosts share the single outside interface address (219.93.144.1) by using different port numbers. The only difference from Lab 6-2 is this uses `interface FastEthernet0/1` directly as the shared address instead of a separate one-address pool — both achieve the same overload behavior.
- The static entry maps Pub Web SRV's private 192.168.7.3 to public 219.93.144.10, the same `ip nat inside source static` pattern as Lab 6-1. That address was picked from within the already-connected 219.93.144.0/24 segment specifically so Cisco's automatic proxy-ARP handles it with no extra routing needed on the Burlington Telecom Router side.
- Internal SRV (192.168.7.2) intentionally has no NAT entry at all — it's meant to stay unreachable from outside, which is the whole reason a data center has both a public-facing and an internal-only server.

### Step 3: Verification

```
show ip nat translations
```

Confirmed both translation types appearing simultaneously:

```
Pro  Inside global      Inside local       Outside local      Outside global
icmp 219.93.144.1:12    192.168.3.2:12     153.104.18.2:12    153.104.18.2:12
icmp 219.93.144.1:13    192.168.3.2:13     153.104.18.2:13    153.104.18.2:13
icmp 219.93.144.1:14    192.168.3.2:14     153.104.18.2:14    153.104.18.2:14
icmp 219.93.144.1:15    192.168.3.2:15     153.104.18.2:15    153.104.18.2:15
---  219.93.144.10      192.168.7.3        ---                ---
```

The four ICMP entries are Foster PC1 pinging the Burlington Telecom server — each ping gets its own port number under the same outside global address (219.93.144.1), which is exactly what overloaded PAT looks like, same as the four rows in Lab 6-2's table. The bottom entry is the static NAT for Pub Web SRV — no ports involved, and it's permanent (stays in the table whether or not traffic is currently flowing, unlike the dynamic PAT entries which age out).

### Troubleshooting Note

**Lesson:** When a translation table shows nothing at all (not even a dropped/incomplete entry), check whether traffic is actually originating correctly before troubleshooting the NAT/ACL/routing config itself — an unconfigured end device looks identical to a routing failure from the router's point of view.

---

## Quiz 1 Study Guide

### Parsing IP Headers

An IPv4 header is 20 bytes minimum. The fields most likely to be asked about:

- **Version** (4 bits) — always `4` for IPv4.
- **IHL (Internet Header Length)** — header length in 32-bit words; minimum value `5` (= 20 bytes) when no options are present.
- **Total Length** — entire packet size (header + data) in bytes.
- **TTL (Time to Live)** — decremented by 1 at every router hop; packet is discarded when it hits 0. This is what traceroute exploits.
- **Protocol** — identifies the next-layer protocol (1 = ICMP, 6 = TCP, 17 = UDP).
- **Header Checksum** — error-checks the header only, not the payload; recalculated at every hop since TTL changes at each hop.
- **Source/Destination Address** — the two 32-bit IP addresses.

Be able to walk through a hex dump or Wireshark capture and identify these fields by their byte offsets, same skill used in the DHCP Wireshark capture assignment (Week 03).

### Subnet Masking and Valid Host Ranges

Core method, same as Week 01's binary review:

1. Convert the mask's "interesting octet" to binary to find the /x prefix (count the 1-bits).
2. **AND** the IP against the mask to get the Network ID.
3. The broadcast address is the network ID with all host bits set to 1.
4. Valid host range = (Network ID + 1) through (Broadcast − 1).

Quick mask-to-CIDR reference:
`128=/1  192=/2  224=/3  240=/4  248=/5  252=/6  254=/7  255=/8`

Example: `/23` network 10.7.12.0 → mask 255.255.254.0 → broadcast 10.7.13.255 → valid hosts 10.7.12.1–10.7.13.254.

### Subnet Design

Given a list of required host counts per department/VLAN, pick the smallest mask that covers the requirement with the standard powers-of-two table (2^n − 2 usable hosts for a given number of host bits n). Practiced this directly in Lab 5-1: 300-host requirements forced a /23 (510 usable) since a /24 (254 usable) wasn't enough, while 150-host requirements fit comfortably in a /24. Also expect to justify *why* a given mask was chosen, not just produce one.

### DHCP — Protocol and Packet Exchange

The four-packet exchange, abbreviated **DORA**:

1. **Discover** — client broadcasts (src 0.0.0.0, dst 255.255.255.255) looking for any DHCP server.
2. **Offer** — server responds with a proposed IP and lease info.
3. **Request** — client broadcasts back, confirming it's accepting that specific offer (broadcast again because there could be multiple competing servers).
4. **Acknowledge (ACK)** — server finalizes the lease.

On a renewal (not a fresh lease), the exchange can shrink to just Request/ACK, sent as unicast directly to the known server rather than broadcast — captured directly in the Week 03 Wireshark assignment.

### DHCP Service Design Considerations

- **Scope/pool sizing** — must match the actual subnet mask; a classic mistake (hit repeatedly across Labs 3-3, 5-1) is Packet Tracer defaulting every new pool to /24 regardless of the VLAN's real size.
- **Start address offset** — pools typically start a few addresses in (e.g., `.20` or `.100`) to leave room at the bottom of the subnet for statically-addressed infrastructure (routers, servers, printers) without risking a DHCP/static conflict.
- **Lease time** — how long a client holds an address before having to renew; short leases suit high-churn environments (guest/visitor networks), long leases reduce renewal traffic for stable environments.
- **Redundancy** — a single DHCP server is a single point of failure for address assignment network-wide.

### DHCP Server — General Configuration

- DHCP broadcasts don't cross router boundaries on their own — any VLAN/subnet that doesn't physically host the DHCP server needs `ip helper-address <DHCP-server-IP>` configured on its router interface, turning the relayed broadcast into a unicast.
- The VLAN the DHCP server physically sits on needs no helper address — it already hears the broadcast directly.
- Each pool configuration needs: default gateway, start address, subnet mask (matching the real subnet), and optionally DNS server and max users.

### Hierarchical Internetworking Model

| Layer | Function | Typical Devices |
|---|---|---|
| **Access (Edge)** | Where end devices (PCs, servers, phones) physically connect | Layer 2 switches (e.g. 2960-series) |
| **Distribution** | Aggregates edge switches, enforces policy/VLAN boundaries, often the inter-VLAN routing point | Multilayer switches or routers (e.g. 3560/3650-series) |
| **Core** | High-speed backbone that moves traffic between distribution blocks; not a routing policy boundary | Fast Layer 2 or Layer 3 switches, minimal config — just aggregation |
| **Border** | The edge of the organization's network, facing an outside/untrusted network (ISP, partner network) | Routers performing NAT, ACLs, and outside routing |

Lab 6-3 maps onto this directly: Foster/Skiff/IDC Distribution Routers sit at Distribution, the CCol Core Switch is plain Core (no routing, just aggregation), and CC Border Router is the Border layer device — it's specifically where NAT was configured, since NAT's job is translating between the organization's private addressing and the outside world.

### Network Address Translation

- **Definition**: a router-level technique that rewrites the source and/or destination IP address (and often port) of a packet as it crosses between two networks — most commonly between a private inside network and a public outside one.
- **Why use NAT**:
  - Conserves public IPv4 address space — a whole private network can share one or a few public addresses.
  - Hides internal addressing structure from outside networks (security-by-obscurity, not a substitute for a firewall).
  - Lets internal addressing change without renumbering anything the outside world depends on.
- **IP Masquerading**: another name for PAT/NAT overload — many inside hosts "masquerade" behind one outside IP, distinguished from each other by port number rather than address. This is the default mode most home/office routers use to share one ISP-assigned address.
- **Types of NAT**:
  - **Static NAT** — one-to-one, permanent mapping between a specific inside address and a specific outside address (Lab 6-1: 10.0.0.2 ↔ 50.0.0.1; Lab 6-3: 192.168.7.3 ↔ 219.93.144.10). Used when outside hosts need a consistent, predictable address to reach a specific internal host.
  - **Dynamic NAT** — maps inside addresses to outside addresses from a pool, one-to-one but assigned on demand rather than fixed. Requires enough pool addresses for however many simultaneous translations are needed.
  - **PAT / NAT Overload** — many-to-one, using port numbers to distinguish multiple inside hosts sharing a single outside address (Lab 6-2: all of 192.168.0.0/24 sharing 30.0.0.120; Lab 6-3: Foster/Skiff sharing 219.93.144.1). The practical answer to not having enough public addresses for every inside host.
