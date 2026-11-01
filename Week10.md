# NET-330: Week 10

## Build a Network Challenge 1: OSPF for Foster 202 Co.

In this lab, I built the Foster 202 Co. network in Packet Tracer. The company has a main office in Burlington and a smaller office in Montreal, connected by a T1 serial link. I configured the interfaces, the VLANs on the multilayer switch, and OSPF so every laptop could ping every other laptop. The ISP, partner, and customer sites were left for Challenge 2.

### Addressing

| Network | Address | Gateway |
|---|---|---|
| BTV-MTL link | 172.16.0.0/30 | BTV-Router 172.16.0.1, MTL-Router 172.16.0.2 |
| BTV Core (VLAN 10) | 172.16.10.0/24 | BTV-Router 172.16.10.1, 3560 switch 172.16.10.2 |
| BTV Users (VLAN 5) | 172.16.5.0/24 | 3560 switch 172.16.5.1 |
| BTV Data Center (VLAN 6) | 172.16.6.0/24 | 3560 switch 172.16.6.1 |
| MTL Net | 172.16.20.0/24 | MTL-Router 172.16.20.1 |

| Laptop | IP | Gateway |
|---|---|---|
| User-Laptop | 172.16.5.2 | 172.16.5.1 |
| DataCenter-Station | 172.16.6.2 | 172.16.6.1 |
| Montreal Laptop | 172.16.20.2 | 172.16.20.1 |

### Port Connections

| Link | Ports |
|---|---|
| BTV-Router to MTL-Router | Serial0/1/0 on both |
| BTV-Router to 3560 switch | BTV Fa0/0 to switch Fa0/1 |
| MTL-Router to Montreal switch | MTL Fa0/0 |
| User-Laptop | 3560 Fa0/2 (VLAN 5) |
| DataCenter-Station | 3560 Fa0/6 (VLAN 6) |

The Montreal switch is a Layer 2 switch and needs no configuration.

### Finding the Ports and the Clock Rate

The routers had no addresses, and all looked the same, so I ran `show ip interface brief` on each device first. To see which port led where, I used `no shutdown` on the ports and then `show cdp neighbors`. For the serial link, `show controllers serial 0/1/0` showed that BTV-Router is the DTE end and MTL-Router is the DCE end. The `clock rate` command only goes on the DCE end, so it went on MTL-Router.

### Configuration

**MTL-Router**
```
enable
config terminal
hostname MTL-Router
no ip domain-lookup
interface serial 0/1/0
 ip address 172.16.0.2 255.255.255.252
 clock rate 64000
 no shutdown
 exit
interface fastEthernet 0/0
 ip address 172.16.20.1 255.255.255.0
 no shutdown
 exit
router ospf 1
 network 172.16.0.0 0.0.0.3 area 0
 network 172.16.20.0 0.0.0.255 area 0
end
copy run start
```

**BTV-Router**
```
enable
config terminal
hostname BTV-Router
no ip domain-lookup
interface serial 0/1/0
 ip address 172.16.0.1 255.255.255.252
 no shutdown
 exit
interface fastEthernet 0/0
 ip address 172.16.10.1 255.255.255.0
 no shutdown
 exit
router ospf 1
 network 172.16.0.0 0.0.0.3 area 0
 network 172.16.10.0 0.0.0.255 area 0
end
copy run start
```

**BTV-Dist-SW (3560 multilayer switch)**
```
enable
config terminal
hostname BTV-Dist-SW
no ip domain-lookup
ip routing
vlan 5
 name Users
 exit
vlan 6
 name DC
 exit
vlan 10
 name Core
 exit
interface vlan 10
 ip address 172.16.10.2 255.255.255.0
 no shutdown
 exit
interface vlan 5
 ip address 172.16.5.1 255.255.255.0
 no shutdown
 exit
interface vlan 6
 ip address 172.16.6.1 255.255.255.0
 no shutdown
 exit
interface fastEthernet 0/1
 switchport mode access
 switchport access vlan 10
 exit
interface fastEthernet 0/2
 switchport mode access
 switchport access vlan 5
 exit
interface fastEthernet 0/6
 switchport mode access
 switchport access vlan 6
 exit
router ospf 1
 network 172.16.10.0 0.0.0.255 area 0
 network 172.16.5.0 0.0.0.255 area 0
 network 172.16.6.0 0.0.0.255 area 0
end
copy run start
```

### Verification

The laptops could ping each other in both directions. `show ip route` on BTV-Router and MTL-Router showed the other side's networks as `O` routes.

### Notes

- On a multilayer switch, the IP address goes on the VLAN interface, not on the port. `ip routing` has to be turned on, or the VLANs will not talk to each other.
- The VLAN 10 link to the router is an access port in VLAN 10, and each laptop port is an access port in its own VLAN.
- Packet Tracer can auto-fill the wrong subnet mask on laptops, so I check the mask and gateway on every device.

---

## Lab 9-2: BGP to Connect a Partner and a Customer Network

This lab extended the same Packet Tracer file. The goal was to use BGP to connect the Foster 202 Co. network to the internet, a Partner network, and a Customer network. Each organization is its own autonomous system (AS).

### Topology and Addressing

| Organization | AS Number | Router |
|---|---|---|
| Foster 202 Co. | 3033 | BTV-Router |
| Vermont ISP | 1010 | Vermont-ISP |
| Pacific ISP | 2054 | Pacific-ISP |
| Partner Company | 34908 | Partner Co |
| Customer Company | 5132 | Customer Co |

| Link | Addresses | Ports |
|---|---|---|
| BTV-Router to Vermont-ISP | 192.168.2.2 and 192.168.2.1 (/30) | BTV Fa0/1, Vermont Fa0/1 |
| Vermont-ISP to Pacific-ISP | 192.168.1.1 and 192.168.1.2 (/30) | Fa0/0 on both |
| Pacific-ISP to Partner Co | 192.168.3.1 and 192.168.3.2 (/30) | Pacific Fa0/1, Partner Fa0/0 |
| Pacific-ISP to Customer Co | 192.168.4.1 and 192.168.4.2 (/30) | Serial0/1/0 on both |
| Partner LAN | 10.15.6.0/24 (router 10.15.6.1) | Partner Fa0/1 |
| Customer LAN | 10.200.24.0/24 (router 10.200.24.1) | Customer Fa0/0 |

Partner PC is 10.15.6.2, and Customer PC is 10.200.24.2. The starter file had these routers unconfigured, so I found the ports with `show ip interface brief` and `show cdp neighbors`.

### Changes on BTV-Router

BTV-Router is the link between the Foster OSPF network and the outside, so it needed extra configuration.

```
enable
config terminal
interface fastEthernet 0/1
 ip address 192.168.2.2 255.255.255.252
 no shutdown
 exit
ip route 0.0.0.0 0.0.0.0 192.168.2.1
router ospf 1
 default-information originate
 exit
router bgp 3033
 neighbor 192.168.2.1 remote-as 1010
 redistribute ospf 1
end
copy run start
```

- The static default route sends everything BTV-Router does not know about to the Vermont ISP. It is the only static route in the lab.
- `default-information originate` under OSPF passes that default route to MTL-Router and the 3560 switch, so the whole Foster network can reach the outside without their own static routes.
- `redistribute ospf 1` under BGP takes the Foster routes learned through OSPF and advertises them to the ISP.

### Configuration of the Other Routers

**Vermont-ISP**
```
enable
config terminal
hostname Vermont-ISP
interface fastEthernet 0/1
 ip address 192.168.2.1 255.255.255.252
 no shutdown
 exit
interface fastEthernet 0/0
 ip address 192.168.1.1 255.255.255.252
 no shutdown
 exit
router bgp 1010
 neighbor 192.168.1.2 remote-as 2054
 neighbor 192.168.2.2 remote-as 3033
 network 192.168.1.0 mask 255.255.255.252
 network 192.168.2.0 mask 255.255.255.252
end
copy run start
```

**Pacific-ISP**
```
enable
config terminal
hostname Pacific-ISP
interface fastEthernet 0/0
 ip address 192.168.1.2 255.255.255.252
 no shutdown
 exit
interface fastEthernet 0/1
 ip address 192.168.3.1 255.255.255.252
 no shutdown
 exit
interface serial 0/1/0
 ip address 192.168.4.1 255.255.255.252
 clock rate 64000
 no shutdown
 exit
router bgp 2054
 neighbor 192.168.1.1 remote-as 1010
 neighbor 192.168.3.2 remote-as 34908
 neighbor 192.168.4.2 remote-as 5132
 network 192.168.3.0 mask 255.255.255.252
 network 192.168.4.0 mask 255.255.255.252
end
copy run start
```

**Partner Co**
```
enable
config terminal
hostname Partner-Co
interface fastEthernet 0/0
 ip address 192.168.3.2 255.255.255.252
 no shutdown
 exit
interface fastEthernet 0/1
 ip address 10.15.6.1 255.255.255.0
 no shutdown
 exit
router bgp 34908
 neighbor 192.168.3.1 remote-as 2054
 network 10.15.6.0 mask 255.255.255.0
end
copy run start
```

**Customer Co**
```
enable
config terminal
hostname Customer-Co
interface serial 0/1/0
 ip address 192.168.4.2 255.255.255.252
 clock rate 64000
 no shutdown
 exit
interface fastEthernet 0/0
 ip address 10.200.24.1 255.255.255.0
 no shutdown
 exit
router bgp 5132
 neighbor 192.168.4.1 remote-as 2054
 network 10.200.24.0 mask 255.255.255.0
end
copy run start
```

The serial link between Pacific-ISP and Customer Co has a DCE end and a DTE end. The `clock rate` command only works on the DCE end and just prints an error on the DTE end, so I entered it on both.

### Tech Journal: How to Configure BGP

1. Get the basic connectivity working first. Give every router interface its address, turn it on with `no shutdown`, and make sure each router can ping its directly connected neighbor. BGP will not form a session over a link that does not work.
2. Start the BGP process with `router bgp <AS number>`. The number is the AS the router belongs to, such as 3033 for Foster 202 Co. Unlike OSPF's process ID, the AS number is meaningful and has to match what the neighbors expect.
3. Identify each peer with `neighbor <peer IP> remote-as <peer AS number>`. The IP is the address of the neighbor's interface on the shared link. When the remote AS is different from mine, this is an external BGP (eBGP) session, which is what every link in this lab uses.
4. Advertise the networks that belong to my AS with `network <address> mask <subnet mask>`. Unlike OSPF's `network` command, BGP uses a normal subnet mask, not a wildcard mask, and the network has to already be in the router's routing table.
5. To advertise routes learned from another protocol, use `redistribute`. On BTV-Router, `redistribute ospf 1` under BGP shares the Foster 202 OSPF routes with the ISP.
6. Give the edge router a way out. A static default route (`ip route 0.0.0.0 0.0.0.0 <ISP address>`) plus `default-information originate` under OSPF lets every router inside the company send outside traffic toward BTV-Router.
7. BGP sessions take a minute or two to come up. Check them with `show ip bgp summary` (a number instead of `Idle` or `Active` in the last column means the session is up) and look for `B` routes in `show ip route`.

### Key Ideas

- BGP is the routing protocol used between autonomous systems, while OSPF is used inside one. In this lab OSPF runs inside Foster 202 Co. and BGP connects the five organizations.
- BGP does not discover neighbors automatically like OSPF does. Each neighbor and its AS number has to be typed in on both sides.
- Other than the default route on BTV-Router, no static routes are used, so every other route is learned through OSPF or BGP.
