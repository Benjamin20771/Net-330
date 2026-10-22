# NET-330: Week 09

## Lab 8-1B: Configuring OSPF in Packet Tracer

In this lab, I configured OSPF on a small enterprise network with four routers: Border, East, Data Center, and West. 
The goal was for the East PC to ping the West PC, with every route learned through OSPF instead of typed in by hand. 
My birthdate is 07/07/2006, so X = 7 in the addressing table.

### Addressing

| Network | Address | Mask | Router interfaces |
|---|---|---|---|
| Border to Data Center | 10.7.1.0 | 255.255.255.248 (/29) | Border 10.7.1.1, Data Center 10.7.1.2 |
| East-Router to Data Center | 10.7.1.20 | 255.255.255.252 (/30) | East 10.7.1.21, Data Center 10.7.1.22 |
| Data Center to West-Router | 10.7.1.32 | 255.255.255.252 (/30) | Data Center 10.7.1.33, West 10.7.1.34 |
| East LAN | 10.7.20.0 | 255.255.255.0 | East-Router 10.7.20.1 |
| West LAN | 10.7.40.0 | 255.255.255.0 | West-Router 10.7.40.1 |

| PC | IP | Mask | Gateway |
|---|---|---|---|
| East PC | 10.7.20.2 | 255.255.255.0 | 10.7.20.1 |
| West PC | 10.7.40.2 | 255.255.255.0 | 10.7.40.1 |

On/30 links, the router acting as the gateway gets the first usable address, and the other router gets the second.

### Port Connections

| Link | Ports |
|---|---|
| Border to Data Center | Border Gi0/0 to Data Center Gi0/2 |
| East to Data Center | East Gi0/0 to Data Center Gi0/0 |
| West to Data Center | West Gi0/0 to Data Center Gi0/1 |
| East LAN | East Gi0/1 |
| West LAN | West Gi0/1 |

### Configuration

I saved the starter file with my name first, which turns on auto-save. Each router hostname includes my name so the screenshots show it is my file.

**Border**
```
enable
config terminal
hostname Benjamin-Border
no ip domain-lookup
interface gigabitEthernet 0/0
 ip address 10.7.1.1 255.255.255.248
 no shutdown
 exit
router ospf 1
 network 10.7.1.0 0.0.0.7 area 0
end
copy run start
```

**Data Center**
```
enable
config terminal
hostname Benjamin-DataCenter
no ip domain-lookup
interface gigabitEthernet 0/2
 ip address 10.7.1.2 255.255.255.248
 no shutdown
 exit
interface gigabitEthernet 0/0
 ip address 10.7.1.22 255.255.255.252
 no shutdown
 exit
interface gigabitEthernet 0/1
 ip address 10.7.1.33 255.255.255.252
 no shutdown
 exit
router ospf 1
 network 10.7.1.0 0.0.0.7 area 0
 network 10.7.1.20 0.0.0.3 area 0
 network 10.7.1.32 0.0.0.3 area 0
end
copy run start
```

**East-Router**
```
enable
config terminal
hostname Benjamin-EastRouter
no ip domain-lookup
interface gigabitEthernet 0/0
 ip address 10.7.1.21 255.255.255.252
 no shutdown
 exit
interface gigabitEthernet 0/1
 ip address 10.7.20.1 255.255.255.0
 no shutdown
 exit
router ospf 1
 network 10.7.1.20 0.0.0.3 area 0
 network 10.7.20.0 0.0.0.255 area 0
end
copy run start
```

**West-Router**
```
enable
config terminal
hostname Benjamin-WestRouter
no ip domain-lookup
interface gigabitEthernet 0/0
 ip address 10.7.1.34 255.255.255.252
 no shutdown
 exit
interface gigabitEthernet 0/1
 ip address 10.7.40.1 255.255.255.0
 no shutdown
 exit
router ospf 1
 network 10.7.1.32 0.0.0.3 area 0
 network 10.7.40.0 0.0.0.255 area 0
end
copy run start
```

### Verification

- `show ip interface brief` showed every configured port as up/up on all four routers.
- `show ip ospf neighbor` on the Data Center router showed all three neighbors (Border, East, West) in the FULL state.
- `show ip route` on the East router listed 10.7.40.0/24 as an `O` route, and on the West router it listed 10.7.20.0/24 as an `O` route. The Data Center router had all of the networks in its table.
- The East PC successfully pinged the West PC.

### Tech Journal: How to Configure OSPF

1. Name each router, give it its interface IP addresses, and turn the interfaces on with `no shutdown`.
OSPF can advertise networks only on interfaces that are up and have an address.
3. Create the OSPF instance with `router ospf 1`. The number is a process ID that only matters on that router, so 1 is fine.
4. Tell OSPF which directly connected networks to advertise with `network <network address> <wildcard mask> area 0`.
Every network goes in area 0, the backbone, since this is a single area design.
6. Use a wildcard mask, which is the inverse of the subnet mask. A /29 is 0.0.0.7, a /30 is 0.0.0.3, and a /24 is 0.0.0.255.
7. Only advertise the networks directly connected to the router I am on.
OSPF shares them with the neighbors, and the neighbors pass along the rest, so every router learns every route on its own.
9. Verify with `show ip ospf neighbor` (neighbors should be FULL) and `show ip route` (look for `O` routes, shown with a cost like `[110/2]`), then test with a ping.

### Troubleshooting Notes

- **Placeholder text typed into the CLI.** My first Border config had `<interface to Data Center>` as a placeholder and I pasted it in literally.
The interface command failed, which left me in the wrong mode, so the rest of the commands failed too. I fixed it by finding the real port names and redoing the config.
Placeholders always need to be replaced before pasting.
- **Finding which port goes where.** All the routers looked the same in `show ip interface brief`, so I used `no shutdown` on the ports followed by `show cdp neighbors` to see which port connected to which device.
- **PCs could not ping but the routers were fine.** The routers showed FULL neighbors and OSPF routes, so the problem was on the PC side.
After I corrected the PC settings the ping worked. Packet Tracer can auto-fill a classful mask like 255.0.0.0 for 10.x addresses, so I now check the mask and the gateway on every PC.
- Adding `no ip domain-lookup` stops the long delay when I mistype a command and the router tries to look it up as a hostname.

---

## Assignment 8: OSPF Authentication

### Security Issues Without Authentication

Without authentication, an OSPF router trusts any OSPF packet it receives on a link. An attacker or a misconfigured device plugged into a segment can send hello packets, form an adjacency with a real router, and then inject false link-state advertisements. 
This lets them change the routing tables of the whole area. They could redirect traffic through their own machine to eavesdrop (a man-in-the-middle attack) or make traffic disappear completely (a black hole). 
Forged or constant updates can also overload routers and cause a denial of service. Because OSPF floods information to every router in the area, one bad advertisement can spread quickly.

### How OSPF Authentication Works

OSPF authentication makes a router prove that a packet came from a trusted neighbor before the packet is accepted. 
Each OSPF packet header has an authentication type field and an authentication data field. 
The routers on a link share a secret, and a router drops any packet that fails the check. 
There are three types: null (no authentication), simple password, and cryptographic. 
Simple password sends the key in clear text, so anyone who captures the traffic can read it. 
Cryptographic authentication, most commonly MD5 on Cisco devices, never sends the key. 
The sender attaches a hash computed from the packet, the key, and a sequence number, and the receiver recalculates the hash to compare. 
The sequence number also helps prevent replayed packets. Newer setups can use HMAC-SHA for stronger protection.

### Configuration

**Router 1 (Benjamin-Border)**
```
enable
config terminal
interface gigabitEthernet 0/0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 OSPFkey7
 exit
end
copy run start
```

**Router 2 (Benjamin-EastRouter)**
```
enable
config terminal
interface gigabitEthernet 0/0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 OSPFkey7
 exit
end
copy run start
```

**Router 3 (Benjamin-DataCenter)**
```
enable
config terminal
interface gigabitEthernet 0/0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 OSPFkey7
 exit
interface gigabitEthernet 0/1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 OSPFkey7
 exit
interface gigabitEthernet 0/2
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 OSPFkey7
 exit
end
copy run start
```

**Router 4 (Benjamin-WestRouter)**
```
enable
config terminal
interface gigabitEthernet 0/0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 OSPFkey7
 exit
end
copy run start
```

### What the Commands Do

- `ip ospf authentication message-digest` turns on MD5 authentication on that interface.
- `ip ospf message-digest-key 1 md5 OSPFkey7` sets the key ID and the shared secret. Both must match the neighbor on the other end of the link.

### Verification

- While configuring, neighbors drop and come back, because a router with authentication will not accept hellos from one without it.
After all four routers were configured, the neighbors returned to FULL.
- `show ip ospf neighbor` on the Data Center router should list all three neighbors as FULL.
- `show ip ospf interface gigabitEthernet 0/0` should include "Message digest authentication enabled".
- The East PC to West PC ping still works with authentication in effect.

### Takeaways

- OSPF without authentication trusts every neighbor, so anyone on the link can poison the routing table.
- MD5 is much better than a simple password because the key never crosses the wire.
- The key ID and key have to match on both ends of every link, or the adjacency will not form.
