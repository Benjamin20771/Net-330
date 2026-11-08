# NET-330: Week 11

## Quiz 2 Review: Network Zoning and Access Lists

### Network Zoning

Network zoning groups systems that have similar security requirements into "zones" and controls the traffic that can flow between the zones. 
Systems are grouped by what they do and how sensitive their data is, instead of just by where they sit in the building. 
A zone is protected by controls such as firewalls and access lists, which only allow the traffic a zone actually needs.

Zoning is a context-based approach to traffic flow policy, as opposed to a perimeter approach. 
A perimeter approach protects the edge of the network and trusts everything inside. 
Zoning assumes that something inside can be compromised, so it puts controls between the internal zones as well. 
This follows the information security idea of protecting resources by controlling access to them, and it applies least privilege to the network, so a system can only reach what it needs to reach.

Benefits of zoning:
- It adds defense in depth, since an attacker who gets into one zone still has to get past the controls to reach another
- It improves intrusion prevention and monitoring, because traffic between zones can be inspected and logged
- It makes troubleshooting and incident response faster, because a problem can be isolated to one zone

The cost is more complexity. Zoning increases the number of VLANs and subnets instead of reducing them, and each zone needs its own policy.

### Choosing a Zone

To place a system, I look at three things in the context and VLAN table: who needs to access it, what kind of data it stores, and who administers it. 
Regulated data is the strongest factor. For example, a server holding protected health information belongs in the HIPAA zone, even if all internal users need to reach it, because the zone controls how it can be accessed and who can manage it.

No single control is enough, so the best network defense is layered. Stronger isolation, like a stand-alone network, gives more protection, but it also makes the network harder to use, so the right level depends on the data being protected.

### Access Lists

A Cisco access list (ACL) is an ordered list of permit and deny rules used to filter traffic.

| Type | Works at | What it can match |
|---|---|---|
| Standard | Layer 3 | Source IP address or network only |
| Extended | Layers 3 and 4 | Source and destination IP address or network, protocol, and TCP or UDP ports |

Key rules:
- ACLs use wildcard masks, such as 0.0.0.255, which is the inverse of a subnet mask. A 0 means the bit has to match, and a 1 means it does not matter.
- Rules are checked from top to bottom, and the first match wins, so the order matters. Specific rules go before general ones.
- Every ACL ends with a hidden "deny all". Any traffic that does not match a rule is blocked, so an ACL that should let other traffic through needs a `permit any` (standard) or `permit ip any any` (extended) at the end.
- The ACL and the interface use two different commands. `ip access-list` (or `access-list`) creates the ACL and its rules. `ip access-group` applies the ACL to an interface, with `in` or `out` for the direction. I need both, since an ACL that is not applied does nothing.
- Placement: a standard ACL goes as close to the destination as possible, because it only looks at the source and could block too much if it is near the source. An extended ACL goes as close to the source as possible, since it is specific enough to stop unwanted traffic early.
- To limit remote administration, an ACL can be applied to the VTY lines with `access-class` instead of `ip access-group`.

---

## Lab 11-1: Standard and Extended ACLs

In this lab, I used a guided Packet Tracer activity to configure named standard and extended ACLs on three routers, and then built an additional ACL on R1. 
The enable password was `cisco`, the ISP network is 200.200.200.0/24, and the VTY step was skipped.

### Standard ACL on R3

Goal: block the 192.168.11.0/24 network from entering R3 on serial 0/0/0. PC3 (192.168.11.10) should not be able to ping PC5 (192.168.30.10), but PC1 should still be able to reach PC5.

```
enable
config terminal
ip access-list standard STND-1
 deny 192.168.11.0 0.0.0.255
 permit any
 exit
interface serial 0/0/0
 ip access-group STND-1 in
end
copy run start
```

This is a standard ACL because it only needs the source network. It is applied inbound close to the destination, and the `permit any` line keeps every other source working.

### Extended ACL on R2

Goal: stop the 192.168.10.0/24 network from reaching the ISP address 200.200.200.1. PC1 should not be able to ping 200.200.200.1, but should be able to ping everything else.

```
enable
config terminal
ip access-list extended EXTEND-1
 deny ip 192.168.10.0 0.0.0.255 host 200.200.200.1
 permit ip any any
 exit
interface serial 0/0/0
 ip access-group EXTEND-1 out
end
copy run start
```

This needs an extended ACL since the rule depends on both the source network and the destination host. `host 200.200.200.1` matches that single address. 
It is applied outbound on the serial interface leaving R2, close to the source.

### Additional ACL on R1 (Bonus Tasks)

Goals: deny the ISP access to the file server (192.168.20.210), allow only mail to the mail server (192.168.20.200), and allow only web to the web server (192.168.20.201). 
I put all three into one extended ACL on the R1 interface facing the server network (FastEthernet0/0, 192.168.20.1).

```
enable
config terminal
ip access-list extended SERVER-ACL
 deny ip 200.200.200.0 0.0.0.255 host 192.168.20.210
 permit tcp any host 192.168.20.200 eq 25
 permit tcp any host 192.168.20.201 eq 80
 deny ip any host 192.168.20.200
 deny ip any host 192.168.20.201
 permit ip any any
 exit
interface fastEthernet 0/0
 ip access-group SERVER-ACL out
end
copy run start
```

How it works:
- The first line blocks only the ISP network from the file server. Everything else can still reach it.
- Mail uses TCP port 25 and web uses TCP port 80. The permit lines for those ports come first, and the denies after them block every other kind of traffic to those two servers, including ping.
- The last line allows all other traffic, since the hidden deny would block it otherwise.
- The order matters. If the denies were above the permits, the mail and web traffic would match the deny first and be blocked.

To test, I browsed to 192.168.20.201 from a PC (web) and used `telnet 192.168.20.200 25` from a PC command line (mail, where `Trying 192.168.20.200 ... Open` means the port is open). I checked the result on the router with `show access-lists`, which lists each rule and how many times it matched, and `show ip interface fastEthernet 0/0`, which shows "Outgoing access list is SERVER-ACL".

### Issue Uncovered and Troubleshooting

**Issue:** `show running-config` on R1 only displayed the top of the file in the CLI. 
I used `show access-lists` and `show ip interface fastEthernet 0/0` instead to show the bonus ACL and where it was applied. 
The Config tab in Packet Tracer also shows the whole running config.

### Command Summary

| Task | Command |
|---|---|
| Create a named standard ACL | `ip access-list standard <name>` |
| Create a named extended ACL | `ip access-list extended <name>` |
| Deny or permit in a standard ACL | `deny 192.168.11.0 0.0.0.255` / `permit any` |
| Deny or permit in an extended ACL | `deny ip <source> <wildcard> host <ip>` / `permit tcp any host <ip> eq <port>` / `permit ip any any` |
| Apply to an interface | `ip access-group <name> in` or `out` |
| Remove from an interface | `no ip access-group <name> in` or `out` |
| Delete an ACL | `no ip access-list standard <name>` or `no ip access-list extended <name>` |
| View ACLs and match counts | `show access-lists` |
| Check which ACL is on an interface | `show ip interface <interface>` |
