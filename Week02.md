# NET-330 — Week 02

## Simple Network — Hardware Lab

This week we rebuilt the same Laptop–Switch–Router–Switch–Server topology from Week 01's Packet Tracer lab, but on physical hardware

**Equipment used:**
- **1841 Router** 
- **Switches** — small Netgear switches 
- **Workstations** — classroom computers, booted off USB thumb drives into a live Kali Linux environment

---

### Step 1: Cabling
- Standard Ethernet cables connecting workstations to switches to the router.

### Step 2: Console Into the Router
1. Plugged the light-blue USB console cable into the workstation's USB port.
2. Checked Device Manager -> Ports (COM & LPT) to find the assigned COM number.
3. Connected the other end (also light blue) to the router's console port.
4. Powered everything on.
5. Opened PuTTY -> **Connection -> Serial**:

| Setting | Value |
|---|---|
| Serial line | COM port from Device Manager |
| Speed (baud) | 9600 |
| Data bits | 8 |
| Parity | None |
| Stop bits | 1 |
| Flow control | None |

6. Confirmed the **Session** tab connection type was set to Serial with the matching COM port.
7. Clicked **Open**, hit Enter -> dropped into the router CLI.

### Step 3: Boot Workstations to Kali
1. Plugged the USB thumb drive into the powered-off computer.
2. Pressed **F10** at the boot splash screen.
3. Boot Menu -> selected **UEFI Vendor Product**.
4. Selected **Live System (amd64)** -> first option.
5. Logged into the live Kali desktop (`kali` / `kali` if prompted).

Repeated for both workstations.

### Step 4: Router Configuration

```
enable
configure terminal

interface FastEthernet 0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
 exit

interface FastEthernet 0/1
 ip address 192.168.2.1 255.255.255.0
 no shutdown
 exit

end
```

### Step 5: IP Configuration on Kali Workstations

Kali's live-boot network manager can be inconsistent, so IPs were set directly via the terminal. First, checked the actual interface name with `ip a`.

**Workstation on the one side:**
```
sudo ip addr add 192.168.1.2/24 dev <interface>
sudo ip route add default via 192.168.1.1
```

**Workstation on the other side:**
```
sudo ip addr add 192.168.2.2/24 dev <interface>
sudo ip route add default via 192.168.2.1
```

### Step 6: Verify Connectivity

From the router:
```
show interfaces
```
Confirmed both FastEthernet interfaces showed as **up/up**.

From the workstations:
```
ping 192.168.1.2
ping 192.168.2.2
```
Both came back with successful replies.

### Step 7: Save Config & Power Cycle Test

From **privileged EXEC mode** (`Router#`, *not* `Router(config)#`):
```
copy running-config startup-config
```
Then physically powered the router off, then back on, and re-checked the interface IPs to confirm they survived the reboot.

### Step 8: Reset Router to Blank State

Once testing was confirmed, the config was rolled back so the next group could start clean:

```
enable
configure terminal

interface FastEthernet 0/0
 no ip address
 shutdown
 exit

interface FastEthernet 0/1
 no ip address
 shutdown
 exit

end
```

---

## Reference Charts

Quick-lookup material for subnetting and address classes — useful any week this comes back up.

<img width="415" height="739" alt="image" src="https://github.com/user-attachments/assets/d06627b3-3fe8-4b11-ab57-0aa0cbe6eead" />

<img width="387" height="516" alt="image" src="https://github.com/user-attachments/assets/6f0cee73-65d4-4880-89c4-364b763e56f5" />

---

## Assignments

## Lab 2-1: Subnet Design

Designing a fully VLAN-segmented network, then building it in Packet Tracer across a two-site (East/West) topology with distribution and edge layers.

**Assigned network:** `10.7.0.0/16` (the "7" is my birthday day)

### Step 1: Addressing Design

| VLAN | VLAN Name | Hosts Needed | Network | Netmask | Router Address |
|---|---|---|---|---|---|
| 1 | Management | 250 | 10.7.14.0/24 | 255.255.255.0 | 10.7.14.1 |
| 100 | FacStaff | 200 | 10.7.15.0/24 | 255.255.255.0 | 10.7.15.1 |
| 110 | Student | 450 | 10.7.12.0/23 | 255.255.254.0 | 10.7.12.1 |
| 130 | StuLab1 | 35 | 10.7.16.128/26 | 255.255.255.192 | 10.7.16.129 |
| 140 | StuLab2 | 65 | 10.7.16.0/25 | 255.255.255.128 | 10.7.16.1 |
| 200 | StuWireless | 1024 | 10.7.0.0/21 | 255.255.248.0 | 10.7.0.1 |
| 210 | FSWireless | 650 | 10.7.8.0/22 | 255.255.252.0 | 10.7.8.1 |

**Approach:** Biggest subnets placed first since they need alignment on larger address boundaries (StuWireless /21, FSWireless /22, Student /23), then smaller blocks (the /24s, then /25, then /26) fill in the remaining space with no waste.

### Physical Topology

| Layer | Device | Role |
|---|---|---|
| Distribution | East-Core-Switch-01 (3560-24PS) | **Router** — gets `ip routing` + VLAN interface IPs |
| Distribution | West-Core-Swith-01 (3560-24PS) | Trunk-only, no routing config |
| Edge | East-Edge-01 (2960-24TT) | FacStaff-01, FacStaff-02, Student-01, Student-02 |
| Edge | East-Edge-02 (2960-24TT) | FacStaff-03, FacStaff-04, Student-03, Student-04, Lab1-01, Lab1-02 |
| Edge | West-Edge-01 (2960-24TT) | FacStaff-05, Student-05 |
| Edge | West-Edge-02 (2960-24TT) | FacStaff-06, Student-06, Lab2-01 |

### Step 2: Edge Switch Configuration

**Create VLANs — all four edge switches:**
```
enable
configure terminal
vlan 100
 name FacStaff
 exit
vlan 110
 name Student
 exit
```
East-Edge-02 only, add: `vlan 130 / name StuLab1`
West-Edge-02 only, add: `vlan 140 / name StuLab2`

**Access ports**, using ranges to configure multiple ports at once, e.g., on East-Edge-01:
```
interface range FastEthernet 0/4-5
 switchport mode access
 switchport access vlan 100
 exit
interface range FastEthernet 0/13-14
 switchport mode access
 switchport access vlan 110
 exit
```
Same pattern repeated per switch, with Lab ports (`0/21-22` on East-Edge-02 → VLAN 130, `0/21` on West-Edge-02 → VLAN 140) added where applicable.

### End Device IP Assignments (static, entered by hand under Desktop → IP Configuration on each PC)

| Device | Switch | Port | IP Address | Mask | Gateway |
|---|---|---|---|---|---|
| FacStaff-01 | East-Edge-01 | Fa0/4 | 10.7.15.2 | 255.255.255.0 | 10.7.15.1 |
| FacStaff-02 | East-Edge-01 | Fa0/5 | 10.7.15.3 | 255.255.255.0 | 10.7.15.1 |
| Student-01 | East-Edge-01 | Fa0/13 | 10.7.12.2 | 255.255.254.0 | 10.7.12.1 |
| Student-02 | East-Edge-01 | Fa0/14 | 10.7.12.3 | 255.255.254.0 | 10.7.12.1 |
| FacStaff-03 | East-Edge-02 | Fa0/4 | 10.7.15.4 | 255.255.255.0 | 10.7.15.1 |
| FacStaff-04 | East-Edge-02 | Fa0/5 | 10.7.15.5 | 255.255.255.0 | 10.7.15.1 |
| Student-03 | East-Edge-02 | Fa0/13 | 10.7.12.4 | 255.255.254.0 | 10.7.12.1 |
| Student-04 | East-Edge-02 | Fa0/14 | 10.7.12.5 | 255.255.254.0 | 10.7.12.1 |
| Lab1-01 | East-Edge-02 | Fa0/21 | 10.7.16.130 | 255.255.255.192 | 10.7.16.129 |
| Lab1-02 | East-Edge-02 | Fa0/22 | 10.7.16.131 | 255.255.255.192 | 10.7.16.129 |
| FacStaff-05 | West-Edge-01 | Fa0/4 | 10.7.15.6 | 255.255.255.0 | 10.7.15.1 |
| Student-05 | West-Edge-01 | Fa0/13 | 10.7.12.6 | 255.255.254.0 | 10.7.12.1 |
| FacStaff-06 | West-Edge-02 | Fa0/4 | 10.7.15.7 | 255.255.255.0 | 10.7.15.1 |
| Student-06 | West-Edge-02 | Fa0/13 | 10.7.12.7 | 255.255.254.0 | 10.7.12.1 |
| Lab2-01 | West-Edge-02 | Fa0/21 | 10.7.16.2 | 255.255.255.128 | 10.7.16.1 |

Cabled PC -> switch with a straight-through copper cable.

**Verify:** same VLAN + same switch pings succeed; different VLAN/switch pings fail (expected).

### Step 3: Trunking

**Both core switches** — add VLANs 100/110/130/140 to the database, then set Fa0/1 and Fa0/2 as trunk ports restricted to those VLANs:
```
interface FastEthernet 0/1
 switchport mode trunk
 switchport trunk allowed vlan 100,110,130,140
 exit
interface FastEthernet 0/2
 switchport mode trunk
 switchport trunk allowed vlan 100,110,130,140
 exit
```

**All four edge switches** — Fa0/1 set to trunk:
```
interface FastEthernet 0/1
 switchport mode trunk
 exit
```

### Step 4: Connect Edge -> Core
Crossover cable from each edge switch's Fa0/1 into a trunk port on its core switch (East edges -> East-Core, West edges -> West-Core).

**Verify:** same VLAN, different switch -> ping succeeds.

### Step 5: Enable Routing — East-Core-Switch-01 only
```
enable
configure terminal
ip routing

interface vlan 100
 ip address 10.7.15.1 255.255.255.0
 no shutdown
 exit
interface vlan 110
 ip address 10.7.12.1 255.255.254.0
 no shutdown
 exit
interface vlan 130
 ip address 10.7.16.129 255.255.255.192
 no shutdown
 exit
interface vlan 140
 ip address 10.7.16.1 255.255.255.128
 no shutdown
 exit
```
West-Core-Switch-01 gets none of this as East-Core is the only routing device; West just trunks traffic through.

**Verify:** different VLANs within East -> ping succeeds.

### Step 6: East–West Trunk
Both cores' Gi0/1 are set to trunk (allowed VLANs 100,110,130,140), connected with a crossover cable.

**Verify:** Every device on the network can reach every other device.

### CLI Mode References

| Mode | Prompt | How to get there |
|---|---|---|
| User EXEC | `Switch>` | 
| Privileged EXEC | `Switch#` | `enable` |
| Global Config | `Switch(config)#` | `configure terminal` |
| Interface Config | `Switch(config-if)#` | `interface <type> <slot/port>` |
| Interface Range Config | `Switch(config-if-range)#` | `interface range <type> <slot/port>-<port>` |

### Assignment 1 — DHCP Reading Summary

Core concepts from the assigned reading on DHCP (RFC 2131 / RFC 3315 for v6):

- **Purpose:** lets a host automatically get an IP address (plus subnet mask, default gateway, DNS server) from a DHCP server instead of manual config. The lease only lasts while connected/in use.
- **One DHCP server can serve multiple subnets** — routers don't need a DHCP server on every subnet. The router (or a **BOOTP relay agent**, per RFC 1542) forwards DHCP traffic across subnet boundaries to reach the server.
- **DORA process** (new lease):
  1. **DISCOVER** — client broadcasts (src port 68 -> dst port 67), looking for any DHCP server
  2. **OFFER** — server responds with a candidate IP (the `YIADDR` — "your IP address")
  3. **REQUEST** — client formally requests that offered IP (needed because there can be multiple offers from multiple servers — client picks one)
  4. **ACK** — server confirms; client can now use the address
- **Key identifiers in the DHCP packet:**
  - `CHADDR` — Client Hardware (MAC) Address, used as the client ID
  - `Transaction ID` — random number client picks, ties requests to responses
  - `GIADDR` — gateway/relay agent IP address (how the server knows which relay to reply through)
- **Options bundled in OFFER/ACK:** subnet mask, default gateway, lease time, DNS server (and historically WINS/NetBIOS info).
- **Lease reuse (renewal):** client can skip DISCOVER/OFFER and go straight to broadcasting a **REQUEST** with its previously-used IP in the "requested IP address" field (`ciaddr` stays `0.0.0.0` since it's not confirmed yet) -> server replies with **ACK** if that address/lease is still valid. Reduces broadcast traffic compared to a full new lease.
- **DHCPRELEASE:** client can voluntarily give up its lease early, identified by its CHADDR + address.
- **Unicast vs broadcast:** typically broadcast-heavy protocol, but some setups (e.g. SOHO routers) use unicast via the client's known MAC.

---

### Assignment 2 — IP Addressing & Subnetting Review

**Q1: Why is subnetting important/valuable?**
- **Reduces broadcast traffic and improves performance.** A broadcast packet reaches every device on a network, so as a flat network grows, broadcast traffic increases and can bog down switching performance. Subnetting keeps broadcasts contained within a subnet, letting other subnets maximize speed and effectiveness (Source: NetworkComputing, *"5 Subnetting Benefits"*).
- **Improves network security through segmentation.** Isolating departments/functions into separate subnets makes it harder for unauthorized access or malicious activity to spread network-wide, and lets firewalls/security controls apply more precisely per subnet (Source: telecomtrainer.com, *"Explain the purpose of subnetting"*).
- **More efficient use of IP address space.** Instead of wasting a large address block on one flat network, subnetting distributes addresses based on actual need (Source: JumpCloud, *"What Is Subnetting?"*) — this matters most with limited pools like the private RFC 1918 ranges.

**Q2: Hospital VLSM Table (10.16.0.0/16)**

| VLAN | Hosts Needed | Network | Subnet Mask | Host Range |
|---|---|---|---|---|
| Admin | 200 | 10.16.0.0/24 | 255.255.255.0 | 10.16.0.1 – 10.16.0.254 |
| Care | 400 | 10.16.2.0/23 | 255.255.254.0 | 10.16.2.1 – 10.16.3.254 |
| Psych | 200 | 10.16.4.0/24 | 255.255.255.0 | 10.16.4.1 – 10.16.4.254 |
| Guest | 600 | 10.16.8.0/22 | 255.255.252.0 | 10.16.8.1 – 10.16.11.254 |

Each block has to start on a boundary matching its own size — a /23 needs to start at an even third-octet value, a /22 at a multiple of 4. That's why Care's /23 skips ahead to octet `2` (leaving `10.16.1.0/24` unused) and Guest's /22 has to jump to octet `8` since `4` was already partly claimed by Psych. This wasted space is a normal side effect of VLSM alignment, not an error.

**Q3: Is 153.104.7.255/24 valid?**
No — with a /24 mask, network = `153.104.7.0`, broadcast = `153.104.7.255` (all host bits = 1). An all-1s host address is reserved to mean "every host on this subnet" and can't be assigned to a device.

**Q4: Router needed for 153.104.21.200/22 and 153.104.23.100/22?**
No. A /22 mask groups the third octet into blocks of 4 (20–23, 24–27, ...). Both `21` and `23` fall in the same block (20–23) -> shared network `153.104.20.0/22` -> same broadcast domain, no router required.

**Q5: Is 153.104.27.1/23 valid?**
Yes. A /23 groups the third octet into blocks of 2 (26–27). Network = `153.104.26.0/23`, broadcast = `153.104.27.255`. `153.104.27.1` is neither the network address nor the broadcast address — it's a valid usable host.

**Q6: Router needed for 153.104.21.200/27 and 153.104.33.100/27?**
Yes. A /27 mask only affects the last octet — it doesn't touch the third octet at all. Since one address is in the third-octet `21` and the other in `33`, they're on completely different networks and need a Layer 3 device to communicate.

---

### Assignment 3 — Subnet Design Practice

**Branch Office Table**

| Location | Network Address | Subnet Mask | CIDR | First Usable IP | Last Usable IP |
|---|---|---|---|---|---|
| Main Office (750 hosts) | 192.168.0.0 | 255.255.252.0 | /22 | 192.168.0.1 | 192.168.3.254 |
| Research Lab (400) | 192.168.4.0 | 255.255.254.0 | /23 | 192.168.4.1 | 192.168.5.254 |
| Manufacturing (400) | 192.168.6.0 | 255.255.254.0 | /23 | 192.168.6.1 | 192.168.7.254 |
| Sales Office (200) | 192.168.8.0 | 255.255.255.0 | /24 | 192.168.8.1 | 192.168.8.254 |
| Telecommuter VPN (200) | 192.168.9.0 | 255.255.255.0 | /24 | 192.168.9.1 | 192.168.9.254 |

Sizes given were already ordered largest -> smallest, so assigning in the listed order used zero wasted address space.

**Building/Dorm Table**

| Location | Network Address | CIDR | First Usable IP | Last Usable IP |
|---|---|---|---|---|
| Foster Hall (225 hosts) | 216.93.144.0 | /24 | 216.93.144.1 | 216.93.144.254 |
| Skiff 100 (25 hosts) | 216.93.145.0 | /27 | 216.93.145.1 | 216.93.145.30 |
| Joyce 310 (25 hosts) | 216.93.145.32 | /27 | 216.93.145.33 | 216.93.145.62 |

Foster needed a full /24 (254 usable); Skiff/Joyce only needed /27s (30 usable each), so both got carved as back-to-back /27 blocks inside the next available /24.

**162.241.216.0/21 — Quick Math**
- Usable hosts: 2^11 − 2 = **2046**
- First usable IP: **162.241.216.1**
- Last usable IP: **162.241.223.254**
- Broadcast address: **162.241.223.255**

A /21 groups the third octet into blocks of 8 (216–223), so the network spans `.216.0` through `.223.255`.
