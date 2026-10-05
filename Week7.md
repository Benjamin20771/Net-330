# NET-330: Week 07

## Reading: Chapter 39, Open Shortest Path First (OSPF)

This week I read about OSPF, an interior routing protocol. These are my notes on the main ideas.

### Why OSPF Exists

RIP is simple and works well for small networks, but it has limits. It is a distance-vector protocol, so it cannot always pick the best route or adapt well to changes. 
It also treats 16 hops as infinity, so it cannot be used when devices are more than 15 hops apart. The IETF started working on OSPF in 1988 to fix this. OSPF uses the link-state algorithm instead.

The name explains it. "Open" means it was developed through the public RFC process, so it is not proprietary. 
"Shortest Path First" is the algorithm it uses to find the best path between networks. OSPF version 2 (RFC 2328) is the only version in use today.

### How OSPF Works

Every router keeps a copy of a link-state database (LSDB). The LSDB describes the whole network as a graph of routers and networks, and each link has a cost. 
The cost can be based on anything the administrator cares about, not just hop count like RIP.

Routers share information with link-state advertisements (LSAs). Over time, every router ends up with the same data, and when something changes, routers send updates so everyone stays current.

### Features and Drawbacks

- Routes are picked dynamically based on the current state of the network
- Equal-cost routes can share traffic
- Supports authentication and classful, subnetted, and classless addressing
- Supports a hierarchical design for large networks
- The main drawback is complexity. OSPF takes more work and expertise than RIP, so I would use RIP for small networks and OSPF for larger ones.

---

## Basic Topology

In a small network, all routers are peers, and each one keeps an LSDB of the entire AS. The LSDB is a table showing which routers and networks can reach each other and what each link costs. 
Costs do not have to be the same in both directions. There is no cost to go from a network to a router, only from a router to a network.

Routers that connect the AS to other ASes are called boundary routers. They use OSPF inside the AS and an exterior protocol like BGP to talk to the outside.

---

## Hierarchical Topology

With dozens or hundreds of routers, having every router keep a full LSDB and exchange updates causes performance problems. 
OSPF solves this by splitting the AS into areas. Each area acts almost like its own small AS. The areas are connected through a backbone, which is always Area 0.

Router roles in this design:

| Router Type | What It Does |
|---|---|
| Internal Router | Connects only within one area and only knows that area |
| Area Border Router | Connects to more than one area, keeps an LSDB for each, and is part of the backbone |
| Backbone Router | Part of Area 0. Every area border router is a backbone router, but not every backbone router is an area border router |

A boundary router is a separate label. It describes a router that talks to outside networks, and it can also be an internal, border, or backbone router.

Traffic inside one area stays inside that area. Traffic between areas goes from the source to an area border router, across the backbone, and then to an area border router in the destination area. 
The benefit is that changes in one area only need to be shared within that area. The downside is more complexity, and the backbone has to be designed carefully because a broken backbone link can isolate an entire area.

---

## Route Determination with the SPF Tree

Each router takes the LSDB and builds a shortest path first tree with itself at the top. This gives it its own view of the network and shows the lowest-cost path to every router and network. 
The router then builds a routing table with the cost and next hop for each network. If the LSDB changes, the tree and the routing table are recalculated.

In the book's example, Router RC builds its tree level by level, always keeping only the cheapest path to each destination. The result for RC:

| Destination | Cost | Next Hop |
|---|---|---|
| N1 | 5 | RA |
| N2 | 3 | local |
| N3 | 6 | local |
| N4 | 10 | RD |

---

## OSPF Messages

OSPF does not use UDP like RIP. It builds IP packets directly and uses IP protocol number 89. There are five message types:

| Type | Message | Purpose |
|---|---|---|
| 1 | Hello | Finds neighbors and creates adjacencies |
| 2 | Database Description | Sends the contents of the LSDB to initialize a new router |
| 3 | Link State Request | Asks for newer information about specific links |
| 4 | Link State Update | Carries updated link information (LSAs) |
| 5 | Link State Acknowledgment | Confirms a Link State Update was received |

When a router starts, it sends Hello messages to find OSPF neighbors. Once it forms an adjacency, Database Description messages fill in its LSDB. After that, routers regularly send Link State Updates and send extra ones when the topology changes. 
Receivers answer with acknowledgments. In a hierarchical network, internal routers only talk within their area, while area border routers also exchange summaries on the backbone.

OSPF supports three authentication types: none, simple password, and cryptographic. Cryptographic is the most secure.

---

## Message Formats

All OSPF messages start with the same 24-byte common header:

| Field | Size | Description |
|---|---|---|
| Version | 1 byte | 2 for OSPFv2 |
| Type | 1 byte | Message type (1 to 5) |
| Packet Length | 2 bytes | Total length including the header |
| Router ID | 4 bytes | The router that sent the message |
| Area ID | 4 bytes | The area the message belongs to |
| Checksum | 2 bytes | Error check of the message |
| AuType | 2 bytes | 0 none, 1 simple password, 2 cryptographic |
| Authentication | 8 bytes | Used for authentication |

The Hello message adds fields such as the network mask, hello interval, router dead interval, router priority, designated router, backup designated router, and a list of neighbors. The Database Description message includes flags for the first message, more messages coming, and master or slave. LSAs carry the actual topology information and each has a 20-byte header. The main LSA types are Router, Network, Summary, and AS-External.

---

## Key Takeaways

- OSPF is a link-state protocol made for larger or more complex networks that RIP cannot handle well
- Each router keeps an LSDB, then builds its own SPF tree to find the lowest-cost routes
- Areas and a backbone (Area 0) let OSPF scale by limiting how far updates travel
- Five message types handle neighbor discovery, database exchange, and updates
- OSPF is more capable than RIP but harder to configure and maintain
