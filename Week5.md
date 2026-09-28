# NET-330: Week 05

## Reference: Healthcare Facility Subnet Table (Lab 5-1, 10.20.0.0/16)

| VLAN | Name | Network Address | Subnet Mask | Host Range | Default Gateway | DHCP Pool Range |
|---|---|---|---|---|---|---|
| 100 | Clinic | 10.20.0.0/23 | 255.255.254.0 | 10.20.0.1 to 10.20.1.254 | 10.20.0.1 | 10.20.0.20 to 10.20.1.254 |
| 110 | Visitor | 10.20.2.0/23 | 255.255.254.0 | 10.20.2.1 to 10.20.3.254 | 10.20.2.1 | 10.20.2.20 to 10.20.3.254 |
| 120 | Office | 10.20.4.0/23 | 255.255.254.0 | 10.20.4.1 to 10.20.5.254 | 10.20.4.1 | 10.20.4.20 to 10.20.5.254 |
| 1 | Default | 10.20.6.0/24 | 255.255.255.0 | 10.20.6.1 to 10.20.6.254 | 10.20.6.1 | 10.20.6.10 to 10.20.6.254 |
| 130 | Counseling | 10.20.7.0/24 | 255.255.255.0 | 10.20.7.1 to 10.20.7.254 | 10.20.7.1 | 10.20.7.20 to 10.20.7.254 |

Clinic, Visitor, and Office each need 300 hosts, which requires a /23 (510 usable). Default and Counseling only need 150 hosts each, so a /24 (254 usable) covers them.

**Static server addresses** (carved out of the Default VLAN):

| Device | IP | Mask | Gateway |
|---|---|---|---|
| DHCP-01 | 10.20.6.2 | 255.255.255.0 | 10.20.6.1 |
| DNS-01 | 10.20.6.4 | 255.255.255.0 | 10.20.6.1 |

---

## Lab 5-1: Small Enterprise Class Lab, Community Healthcare Facility

Single Distribution Area design. One multilayer switch acting as router at the Border layer, three plain switches at Core, three plain switches at Edge, two servers hanging off the Data Center Core.

### Devices Used

| Role | Devices | Model |
|---|---|---|
| Border | Hospital Router | 3650-24PS (multilayer switch used for routing) |
| Core | North Core, South Core, Data Center Core | 2960-24TT |
| Edge | North Wing Edge, South Wing Edge, Counseling Center Edge | 2960-24TT |
| Servers | DHCP-01, DNS-01 | Server-PT |
| Clients | PCs per VLAN on North Wing and South Wing | PC-PT |

### Pre Lab Setup

Set Options, Preferences, Miscellaneous, Auto File Backup Interval to 1 minute. Save the workspace immediately as `NET-330-Lab-5-1-name` before doing anything else, since auto backup only starts working after that first manual save.

### Interface Naming Gotcha

The 3650-24PS does not use FastEthernet port names like the 2960s do. It uses `GigabitEthernet1/0/X`. Ran into an "Invalid interface type and number" error trying `interface FastEthernet0/1` on the router before catching this. Running `show ip interface brief` on any new device before typing interface commands is the fastest way to confirm actual port names rather than assuming.

Also worth noting: the `switchport trunk encapsulation dot1q` command does not work on either the 3650-24PS or the 2960-24TT in this lab. Both throw `Invalid input detected`. Skipping that line entirely and going straight to `switchport mode trunk` works fine, same as some of the switches from earlier weeks.

### Hospital Router Configuration

```
enable
configure terminal
hostname Hospital-Router
ip routing

vlan 100
 name Clinic
 exit
vlan 110
 name Visitor
 exit
vlan 120
 name Office
 exit
vlan 130
 name Counseling
 exit

interface vlan 1
 ip address 10.20.6.1 255.255.255.0
 no shutdown
 exit

interface vlan 100
 ip address 10.20.0.1 255.255.254.0
 no shutdown
 exit

interface vlan 110
 ip address 10.20.2.1 255.255.254.0
 no shutdown
 exit

interface vlan 120
 ip address 10.20.4.1 255.255.254.0
 no shutdown
 exit

interface vlan 130
 ip address 10.20.7.1 255.255.255.0
 no shutdown
 exit

interface GigabitEthernet1/0/1
 switchport mode trunk
 exit

interface GigabitEthernet1/0/2
 switchport mode trunk
 exit

end
copy run start
```

VLAN 1 is administratively down by default on this device, so it needs its own `no shutdown` line even though the other VLANs do not. Gi1/0/1 faces North Core, Gi1/0/2 faces South Core. The port toward Data Center Core (Gi1/0/3) needs no trunk config at all since it only carries VLAN 1 traffic.

### North Core and South Core Configuration

Same command block for both, just swap the hostname.

```
enable
configure terminal
hostname North-Core

vlan 100
 name Clinic
 exit
vlan 110
 name Visitor
 exit
vlan 120
 name Office
 exit

interface FastEthernet0/1
 switchport mode trunk
 exit

interface FastEthernet0/2
 switchport mode trunk
 exit

end
copy run start
```

### North Wing Edge and South Wing Edge Configuration

Same block for both, just swap hostname.

```
enable
configure terminal
hostname North-Wing-Edge

vlan 100
 name Clinic
 exit
vlan 110
 name Visitor
 exit
vlan 120
 name Office
 exit

interface range FastEthernet0/1-6
 switchport mode access
 switchport access vlan 100
 exit

interface range FastEthernet0/7-10
 switchport mode access
 switchport access vlan 110
 exit

interface range FastEthernet0/11-16
 switchport mode access
 switchport access vlan 120
 exit

interface FastEthernet0/24
 switchport mode trunk
 exit

end
copy run start
```

6 ports Clinic, 4 ports Visitor, 6 ports Office, matching the lab requirement exactly. The uplink trunk port needs to be a dedicated port outside the 1 through 16 access range. Do not reuse a port that is already assigned to an access VLAN as the trunk uplink.

### Data Center Core Configuration

```
enable
configure terminal
hostname Data-Center-Core
end
copy run start
```

Only VLAN 1 runs here, which is the default on every port, so nothing else is needed. Router and both servers plug straight in.

### DHCP Server Configuration

Rename to DHCP-01. Static IP 10.20.6.2, mask 255.255.255.0, gateway 10.20.6.1. Turn DHCP service on under Services.

Edit the default `serverPool` (cannot be deleted) for VLAN 1: gateway 10.20.6.1, start IP 10.20.6.10, mask 255.255.255.0.

Create four additional pools:

| Pool Name | Default Gateway | Start IP | Subnet Mask | Max Users |
|---|---|---|---|---|
| CLINIC | 10.20.0.1 | 10.20.0.20 | 255.255.254.0 | 300 |
| VISITOR | 10.20.2.1 | 10.20.2.20 | 255.255.254.0 | 300 |
| OFFICE | 10.20.4.1 | 10.20.4.20 | 255.255.254.0 | 300 |
| COUNSELING | 10.20.7.1 | 10.20.7.20 | 255.255.255.0 | 150 |

Packet Tracer defaults every new pool to 255.255.255.0. Easy to forget the /23 mask on Clinic, Visitor, and Office since that default is wrong for those three.

### DHCP Relay on Hospital Router

Every user VLAN interface needs `ip helper-address` pointing at the DHCP server, since DHCP requests are broadcasts and cannot cross the router without help.

```
enable
configure terminal

interface vlan 100
 ip helper-address 10.20.6.2
 exit

interface vlan 110
 ip helper-address 10.20.6.2
 exit

interface vlan 120
 ip helper-address 10.20.6.2
 exit

interface vlan 130
 ip helper-address 10.20.6.2
 exit

end
copy run start
```

VLAN 1 does not need a helper address since the DHCP server already lives on that VLAN directly.

### DNS Server Configuration

Rename to DNS-01. Static IP 10.20.6.4, mask 255.255.255.0, gateway 10.20.6.1. Turn DNS service on under Services.

**A Records** added through Services, DNS, entering each hostname, setting Type to A Record, and the target address, then clicking Add:

| Hostname | Type | IP Address |
|---|---|---|
| ns.word.com | A | 10.20.6.4 |
| dhcp.word.com | A | 10.20.6.2 |

Updated every DHCP pool (serverPool, CLINIC, VISITOR, OFFICE, COUNSELING) to hand out 10.20.6.4 as the DNS server instead of 0.0.0.0. Clients needed a fresh DHCP release and renew to actually pick up the new DNS server address after this change; a stale lease does not update on its own.

Verified from a client:

```
nslookup ns.word.com
nslookup dhcp.word.com
```

### Major Troubleshooting Issue: Native VLAN Mismatch

After building South Core and South Wing Edge, a test PC on South Wing pulled a DHCP address from the VLAN 1 pool (10.20.6.x) instead of the Clinic pool (10.20.0.x), even though the port it was plugged into was correctly assigned to VLAN 100 in the switch configuration.

The switch log revealed the actual cause:

```
%CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered on FastEthernet0/1 (100), with South-Core FastEthernet0/2 (1)
```

The trunk uplink port on South Wing Edge and one of the VLAN 100 access ports had been assigned to the same physical port number. That port was serving two conflicting roles at once: an access port for a client, and the trunk link to South Core. This caused traffic to be treated inconsistently between the two switches.

The fix was physical, not just configuration. Moving the actual cable so the trunk uplink to South Core landed on a dedicated port (Fa0/24, matching the pattern already used on North Wing Edge), separate from the access port range, cleared the mismatch. The PC correctly pulled a Clinic VLAN address on its next DHCP renewal.

**Lesson:** a switchport configuration can be completely correct in the running config and still fail if the physical cabling does not actually match what that configuration expects. When a device is doing something a configuration review says it should not be doing, check what is physically plugged into which port before assuming the command syntax is wrong.

---

## Deliverables Completed

| # | Deliverable | Status |
|---|---|---|
| 1 | Subnet Table | Done |
| 2 | Topology screenshot | Done |
| 3 | DHCP working (PC IP Configuration page) | Done |
| 4 | Cross subnet ping | Done |
| 5 | North Wing PC pinging South Wing PC | Done |
| 6 | DNS resolution (nslookup) | Done |

**Bonus (not completed):** Counseling Center Edge and its clients, additional data center servers with their own DNS records.
