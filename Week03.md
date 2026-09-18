# NET-330 — Week 03

## Reference: VLAN Subnet Table (carried over from Lab 2-1, 10.7.0.0/16)

| VLAN | Name | Network | Netmask | Router Address |
|---|---|---|---|---|
| 1 | Management | 10.7.14.0/24 | 255.255.255.0 | 10.7.14.1 |
| 100 | FacStaff | 10.7.15.0/24 | 255.255.255.0 | 10.7.15.1 |
| 110 | Student | 10.7.12.0/23 | 255.255.254.0 | 10.7.12.1 |
| 130 | StuLab1 | 10.7.16.128/26 | 255.255.255.192 | 10.7.16.129 |
| 140 | StuLab2 | 10.7.16.0/25 | 255.255.255.128 | 10.7.16.1 |

This week builds directly on top of the Lab 2-1 topology — East-Core-Switch-01 is still the only routing device; West-Core-Swith-01 still just trunks traffic through.

---

## Lab 3-2: Management VLAN Server

Added a Generic Server directly onto East-Core-Switch-01's Management VLAN (VLAN 1) — this server later becomes DHCP-01 in Lab 3-3.

### Addressing
| Device | IP | Mask | Gateway |
|---|---|---|---|
| East-Core-Switch-01 (VLAN 1 interface) | 10.7.14.1 | 255.255.255.0 | — |
| Server | 10.7.14.2 | 255.255.255.0 | 10.7.14.1 |

### Steps
1. Dragged a **Generic Server** onto the canvas near East-Core-Switch-01.
2. Server → Desktop → IP Configuration → set static IP `10.7.14.2` / `255.255.255.0`, gateway `10.7.14.1`.
3. Configured the switch port as access on VLAN 1:
```
enable
configure terminal
interface FastEthernet 0/3
 switchport mode access
 switchport access vlan 1
 exit
```
4. Assigned the router-side IP to the VLAN 1 interface:
```
interface vlan 1
 ip address 10.7.14.1 255.255.255.0
 no shutdown
 exit
```
5. Connected server Fa0 → East-Core Fa0/3 with straight-through copper.
6. Verified with `ping 10.7.14.1` from the server, then pinged devices on other VLANs (e.g. FacStaff-01, and a West-side device) to confirm full inter-VLAN routing — not just reachability to the local gateway.

> 📝 **Gotcha:** VLAN 1 is the only VLAN interface that comes up admin-down by default on these switches. Every other VLAN (100/110/130/140) didn't need this, so it's easy to forget the `no shutdown` here specifically.

---

## Lab 3-3: DHCP Server in Packet Tracer

Goal: stop hand-typing every client's IP (painful in Lab 2-1) by standing up a real DHCP server and having the router relay broadcast requests to it across VLANs.

### Step 1: Rename & Enable
- Renamed the Lab 3-2 server to **DHCP-01**.
- DHCP-01 → **Services** tab → DHCP → **On**.

### Step 2: Configure `serverPool` (VLAN 1 / Management)
Default pool, edited rather than recreated:
- Start Address: `10.7.14.100`
- Default Gateway: `10.7.14.1`
- Subnet Mask: `255.255.255.0`
- DNS/TFTP left at `0.0.0.0`

### Step 3: Created Pools for User VLANs

| Pool Name | Default Gateway | Start IP | Subnet Mask | Max Users |
|---|---|---|---|---|
| FACSTAFF | 10.7.15.1 | 10.7.15.20 | 255.255.255.0 | 200 |
| STUDENT | 10.7.12.1 | 10.7.12.20 | **255.255.254.0** | 450 |
| LAB1 | 10.7.16.129 | 10.7.16.148 | 255.255.255.192 | 35 |
| LAB2 | 10.7.16.1 | 10.7.16.20 | 255.255.255.128 | 65 |

> 📝 **Gotcha:** Packet Tracer defaults every new pool's mask to `255.255.255.0`. Easy to miss on Student, which is actually a **/23** (`255.255.254.0`) — if left at the default, the pool won't match the real subnet and clients get bad leases.

Start addresses offset from `.1` (e.g. `.20`, `.100`, `.148`) to leave room at the bottom of each subnet for static devices (routers, servers) without DHCP potentially handing out a conflicting address.

### Step 4: Tested Locally (VLAN 1 — no relay needed)
Added a test workstation on East-Core Fa0/4, set to DHCP, confirmed it pulled an address starting at `10.7.14.100`. Worked immediately since the DHCP server is on the same broadcast domain — no relay required for VLAN 1 traffic.

### Step 5: Configured DHCP Relay (`ip helper-address`)

DHCP requests are broadcasts, and broadcasts don't cross router boundaries — so every **other** VLAN needs to be told to forward those broadcasts to DHCP-01 as a unicast:

```
enable
configure terminal

interface vlan 100
 ip helper-address 10.7.14.2
 exit

interface vlan 110
 ip helper-address 10.7.14.2
 exit

interface vlan 130
 ip helper-address 10.7.14.2
 exit

interface vlan 140
 ip helper-address 10.7.14.2
 exit
```
VLAN 1 doesn't get a helper-address — the server is already local to it.

### Step 6: Fixed a Missing East–West Link

While testing, discovered the East-Core ↔ West-Core trunk from Lab 2-1 wasn't actually present in this file (diagram had no line between the two core switches at all) — meaning West-side devices had no path to DHCP-01. Added it:

```
enable
configure terminal
interface GigabitEthernet0/5
 switchport mode trunk
 switchport trunk allowed vlan 100,110,130,140
 exit
```
Applied on **both** East-Core-Switch-01 and West-Core-Swith-01, cabled Gi0/5 to Gi0/5 with copper. Confirmed with `show interfaces trunk` on both switches before retesting.

> 📝 **Takeaway:** always double-check inherited topology pieces (like an East-West link from a prior lab) actually made it into the working file — don't assume a previous lab's config carried over just because the devices are still there.

### Step 7: Converted Clients to DHCP
For every FacStaff, Student, Lab1, and Lab2 device: Desktop → IP Configuration → **DHCP**. Used `ipconfig /renew` on any device that didn't pick up a lease automatically.

### Step 8: Final Verification
Re-ran cross-VLAN ping tests from Lab 2-1 — same results, just DHCP-assigned addresses instead of static ones.

---

## Tech Journal Entry: DHCP in Packet Tracer

Standalone reference for setting this up again in future labs, since DHCP will keep coming up.

### 1. How to Create Pools
On the DHCP server, go to **Services → DHCP**. Each pool needs:
- **Pool Name** — anything descriptive (e.g. `FACSTAFF`)
- **Default Gateway** — the router address for that VLAN
- **DNS Server** — leave at `0.0.0.0` if not specified
- **Start IP Address** — first address to hand out (offset from `.1`, e.g. `.20` or `.100`, to leave room for statically-assigned devices)
- **Subnet Mask** — **must match the actual VLAN subnet** — Packet Tracer defaults every new pool to `/24` (`255.255.255.0`), which is wrong for any VLAN that isn't actually a /24
- **Max Users** — set close to the real host requirement from the subnet table
- **TFTP Server** — leave at `0.0.0.0` unless specified

Click **Save** after each pool — it's easy to lose an edit by navigating away without saving.

### 2. Working with `serverPool`
`serverPool` is the **default pool** and **cannot be deleted or renamed** — it's built into the DHCP service. Rather than fighting it, just repurpose it as the pool for whichever VLAN the server itself lives on (Management/VLAN 1 in this case). Edit its Start Address, Gateway, and Subnet Mask directly instead of creating a separate pool for that VLAN.

### 3. Assigning `ip helper-address`
DHCP traffic is broadcast, and broadcasts don't cross router boundaries — so any VLAN the DHCP server isn't physically sitting on needs to be told to relay those broadcasts to the server as a unicast. Done from the **router's** VLAN interface config (on East-Core-Switch, since it's the routing device):
```
enable
configure terminal
interface vlan <VLAN_NUMBER>
 ip helper-address <DHCP_SERVER_IP>
 exit
```
Repeat once per user VLAN. The VLAN the server physically lives on does **not** need this — it already reaches the server without any relay.

### 4. Issues Encountered
- **Wrong subnet mask on a new pool.** Packet Tracer pre-fills `255.255.255.0` for every pool regardless of the VLAN's real size. The Student VLAN (a /23) needed `255.255.254.0` set manually, or its pool wouldn't match the actual subnet and clients would get broken leases.
- **VLAN 1 interface not passing traffic.** VLAN 1 is the only VLAN interface that's admin-down by default on these switches (`no shutdown` is required after setting its IP) — every other VLAN interface didn't need this extra step, so it's easy to forget on VLAN 1 specifically.
- **Missing East-West trunk link.** Discovered mid-lab that the Gi0/1 trunk between the two core switches from Lab 2-1 wasn't actually present in this file — West-side devices had no path to the DHCP server until that link was added and trunked. Good reminder to verify inherited topology pieces are actually there rather than assuming they carried over.

---

### Assignment 3-1 — DHCP Security Paper

Researched and wrote a formal paper (submitted separately as a Word doc, APA format) identifying three DHCP security issues, each with a vulnerability description and defensive countermeasures:

1. **DHCP Starvation Attacks** — attacker floods the server with DHCPDISCOVER messages using spoofed MAC addresses, exhausting the address pool and causing a denial-of-service for legitimate clients. Defended primarily through DHCP snooping rate-limiting and port security (limiting MAC addresses per port).

2. **Rogue DHCP Servers / DHCP Spoofing** — attacker's unauthorized DHCP server races the legitimate one to respond to client DISCOVER broadcasts, then hands out a malicious gateway/DNS to enable man-in-the-middle attacks. Defended through DHCP snooping's trusted/untrusted port model, dynamic ARP inspection, and active scanning for unauthorized servers.

3. **TunnelVision (CVE-2024-3661)** — a rogue DHCP server uses Option 121 (Classless Static Route) to inject routes more specific than a VPN's catch-all route, silently pulling victim traffic outside the encrypted tunnel while the VPN client still shows "connected." Defended by preventing the rogue server in the first place (DHCP snooping) plus VPN-side hardening that ignores DHCP-supplied routes.

**Key sources used:** Dilworth (2025, arXiv) for the taxonomic overview and TunnelVision explanation; Moratti & Cronce (2024, Leviathan Security) as the original TunnelVision disclosure; Cisco documentation for DHCP snooping configuration; Pentera, PivIT Global, and ProSec GmbH for starvation/spoofing mechanics and mitigation.

---

### Assignment 3-2 — DHCP Wireshark Capture (Windows)

Captured a full release/renew cycle in Wireshark by running `ipconfig /release` then `ipconfig /renew` while capturing, then filtering on `bootp`.

**Capture results:**

| # | Time | Source | Destination | Operation |
|---|---|---|---|---|
| 7 | 4.22s | 10.0.17.27 | 10.0.17.2 | DHCP Release |
| 8 | 8.70s | 0.0.0.0 | 255.255.255.255 | DHCP Discover |
| 9 | 8.70s | 10.0.17.2 | 10.0.17.27 | DHCP Offer |
| 10 | 8.70s | 0.0.0.0 | 255.255.255.255 | DHCP Request |
| 11 | 8.74s | 10.0.17.2 | 10.0.17.27 | DHCP ACK |

**Total DHCP packets exchanged:** 5

**DHCP Server IP:** 10.0.17.2 (source of both the Offer and the ACK)

**Source IP of the Offer packet:** 10.0.17.2

**Why Discover/Request show source `0.0.0.0`:** the client doesn't have a confirmed IP at that point in the exchange, so it can't list a real source address — it broadcasts to `255.255.255.255` instead so any DHCP server on the segment can hear it.

**Additional configuration info in the Offer/ACK (beyond the IP address itself), from expanding the Bootstrap Protocol options:**
- Subnet Mask: 255.255.255.0
- Router (Default Gateway): 10.0.17.2
- Domain Name Server: 10.0.17.2
- Domain Name: cyber.local
- IP Address Lease Time: 1 day (86400 sec)
- DHCP Server Identifier: 10.0.17.2

> 📝 **Observation:** DHCP server, default gateway, and DNS server are all the same device (10.0.17.2) on this network — a common setup where a single router/firewall appliance handles all three roles.
