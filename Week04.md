# NET-330: Week 04

## Reference: Hospital VLAN Table (Lab 4-1, Alternative Assignment)

| VLAN Name | VLAN # | Net/Mask | Default Gateway |
|---|---|---|---|
| West Clinic | 100 | 192.168.10.0/24 | 192.168.10.1 |
| West Admin | 110 | 192.168.11.0/24 | 192.168.11.1 |
| Central Clinic | 200 | 192.168.20.0/24 | 192.168.20.1 |
| Central Admin | 210 | 192.168.21.0/24 | 192.168.21.1 |
| East Clinic | 300 | 192.168.30.0/24 | 192.168.30.1 |
| East Admin | 310 | 192.168.31.0/24 | 192.168.31.1 |
| Backbone (future) | 50 | 192.168.50.0/24 | -- |

VLAN 50 has no gateway and no interface built yet. 

---

## Lab 4-1: Small Enterprise Class Lab (Alternative Assignment)

Make-up lab for a missed class session, built solo instead of in a group. All three hospital sites (West, Central, East) were built independently using the same repeating pattern, rather than just one wing as a group.

### Equipment Per Site
- 1x 3560-24PS (MLS / distribution layer): same model used for the core switches in Lab 2-1
- 2x 2960-24TT (edge layer: North-Wing and South-Wing)
- 2x PC (Clinic PC on North switch, Admin PC on South switch)

### Cabling Per Site
| Connection | Cable Type |
|---|---|
| MLS <-> North-Wing switch | Crossover |
| MLS <-> South-Wing switch | Crossover |
| North-Wing switch <-> Clinic PC | Straight-through |
| South-Wing switch <-> Admin PC | Straight-through |

No cabling exists between the three sites yet. 

### West Site Configuration

**West-MLS:**
```
enable
configure terminal
hostname West-MLS
ip routing

vlan 100
 name West-Clinic
 exit
vlan 110
 name West-Admin
 exit

interface vlan 100
 ip address 192.168.10.1 255.255.255.0
 no shutdown
 exit

interface vlan 110
 ip address 192.168.11.1 255.255.255.0
 no shutdown
 exit

interface FastEthernet 0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 exit

interface FastEthernet 0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 exit

end
copy run start
```

**North-West-Wing-SW:**
```
enable
configure terminal
hostname North-West-Wing-SW

vlan 100
 name West-Clinic
 exit

interface FastEthernet 0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 exit

interface FastEthernet 0/2
 switchport mode access
 switchport access vlan 100
 exit

end
copy run start
```

**South-West-Wing-SW:**
```
enable
configure terminal
hostname South-West-Wing-SW

vlan 110
 name West-Admin
 exit

interface FastEthernet 0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 exit

interface FastEthernet 0/2
 switchport mode access
 switchport access vlan 110
 exit

end
copy run start
```

**PCs (Kali workstations):**
| PC | IP | Mask | Gateway |
|---|---|---|---|
| West-Clinic PC | 192.168.10.2 | 255.255.255.0 | 192.168.10.1 |
| West-Admin PC | 192.168.11.2 | 255.255.255.0 | 192.168.11.1 |

**Verify:** `ping 192.168.11.2` from the West-Clinic PC.

### Central and East Sites

Same pattern: hostname, `ip routing`, VLAN creation, VLAN interfaces, trunk config on the MLS; hostname, single VLAN, access port, trunk uplink on each edge switch. Only the site name, VLAN numbers, and subnet octet change.

| Site | MLS Hostname | Clinic VLAN | Admin VLAN | Clinic Subnet | Admin Subnet |
|---|---|---|---|---|---|
| Central | Central-MLS | 200 | 210 | 192.168.20.0/24 | 192.168.21.0/24 |
| East | East-MLS | 300 | 310 | 192.168.30.0/24 | 192.168.31.0/24 |

Edge switches follow the same naming pattern: North-Central-Wing-SW / South-Central-Wing-SW, North-East-Wing-SW / South-East-Wing-SW.

### Tech Journal Notes (per lab's required deliverable)

**New Cisco configuration commands learned:**
- `hostname <name>`: renames the device; the new name shows in every prompt from that point on.
- `ip routing`: required on a multilayer switch (MLS) before it will route between VLANs at all. Without it, VLAN interfaces can have IP addresses but traffic won't actually route between them.
- `vlan <#>` / `name <name>`: creates VLANs directly from the CLI. These switches have no GUI VLAN database, so every VLAN has to be defined this way.
- `interface vlan <#>` / `ip address <ip> <mask>` / `no shutdown`: creates the routed interface for a VLAN, giving it an IP that acts as the default gateway for devices on that VLAN.
- `switchport trunk encapsulation dot1q`: has to be set before `switchport mode trunk`. This defines how VLAN tags are added to frames crossing the trunk link.
- `switchport mode access` / `switchport access vlan <#>`: assigns a specific port to a single VLAN for an end device.
- `copy running-config startup-config` (or the shortcut `copy run start`): saves the active configuration to memory. Configuration changes take effect immediately but are not saved automatically; a reload without this command loses everything.

**Troubleshooting/issues encountered:**
- Ran `copy running-config startup-config` while still in global config mode (`(config)#` prompt) and got `% Invalid input detected`. The save command only works from privileged EXEC mode (`#`), not from inside `configure terminal`. Fixed by typing `end` first to back out to the `#` prompt, then running the save command.
- The save command prompts for a destination filename (`Destination filename [startup-config]?`). Pressing Enter accepts the default, which is the correct destination, so no need to type anything there.
- Had to be careful with the order of `switchport trunk encapsulation dot1q` and `switchport mode trunk`. Encapsulation has to be configured first, or the trunk port can end up misconfigured.

---

## Assignments

### DNS Quiz

**DNS server hierarchy, highest to lowest:**
- Highest: Root Servers
- Middle: Top-Level Domain (TLD) Servers
- Lowest: Authoritative DNS Server

**The 2 types of DNS queries:**
- Iterative
- Recursive

**Fill in the blank:** Once a DNS server learns a mapping, it **caches** it in its local memory.

---

### DNS Security Assignment

Full paper submitted separately (APA format, with references). Summary of key points:

- **Recursive vs. iterative queries:** recursive means the DNS server does the full lookup chain on the client's behalf; iterative means the server just refers the client to the next server to try.
- **DNS amplification attack:** enabling recursion for any client on the internet (an "open resolver") lets an attacker spoof a victim's IP in a query, causing the server to blast a large response at the victim instead of the real requester. This is a DDoS technique documented by CISA (Alert TA13-088A, 2013).
- **Proper recursion config:** an organization's DNS server should only allow recursive queries from its own internal client ranges, while still answering non-recursive queries about its own domain from anyone on the internet.
- **DNS Views (split-horizon DNS):** lets one server give different answers to the same query depending on where the request came from, typically internal vs external clients, improving both efficiency (avoids routing traffic out and back in) and security (hides internal hostnames/addresses from outsiders).
- **IPAM:** centralizes IP address tracking with tight DNS/DHCP integration (often called DDI), replacing manual spreadsheets. Organizations are moving toward it to cut down on address conflicts and keep DNS records automatically in sync as networks grow.

**Key sources used:** CISA Alert TA13-088A (2013) for the amplification attack; ManageEngine for IPAM.
