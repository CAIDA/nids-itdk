[README](README.md) | Introduction ⮕ | [Background](Background.md) | [Datasets](Datasets.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-itdk.ipynb)

## What Is the Internet Topology Data Kit?

The Internet Topology Data Kit (ITDK) is a dataset produced by CAIDA from large-scale traceroute measurement campaigns. Each campaign sends probes toward millions of destination IP addresses from dozens of vantage points distributed across the Internet. As each probe travels from source to destination, intermediate routers reveal themselves by sending ICMP TTL-exceeded messages — the basis of the classic `traceroute` tool.

### From Traceroutes to a Topology Graph

Raw traceroutes produce sequences of IP addresses: one IP per hop per vantage point. The ITDK post-processes these sequences in several stages:

1. **Alias resolution.** A single physical router has many interfaces, each with its own IP address. Alias resolution groups interface IPs that belong to the same router into a single _node_. CAIDA uses the Midar and Hoiho tools for this; the result is a node-to-IP mapping stored in the routers file.

2. **Link inference.** Two nodes are connected by a _link_ if their IPs appear consecutively in at least one traceroute path. Cycle elimination removes spurious adjacencies caused by routing loops or ICMP source-selection anomalies.

3. **Non-responsive Hops.** Not all hops respond with a valid ICMP TTL expired message. ITDK replaces these non-responsive hops with a **placeholder node**. This maintains reachability, but creates large numbers of artificial nodes that require careful thought about how to handle them.

4. **Hyperlinks.** While large parts of the Internet are router to router (point-to-point), it also contains mesh networks with a single link connecting multiple routers. While these **hyperlinks** match the underlying reality, they cause problems for systems that expect node to node links.

The result is a graph of (node, link) pairs representing the router-level topology of the public Internet as observed from CAIDA's vantage points.

### What the 2026 ITDK Contains

The 2026 ITDK used in this module contains:

- Millions of router nodes, each with one or more interface IP addresses
- Inferred links between router nodes observed to be adjacent in traceroute paths
- AS assignments mapping each node to the autonomous system that operates it
- Geolocation for nodes using two complementary methods (see below)
- DNS PTR (hostname) records for interface IPs

## How to Read Router-Level Topology

### Nodes and Links

A **node** in the ITDK represents one physical router. The database stores it as a `node_id` (e.g., `N1967`) with a set of interface IPs. A **link** represents an observed adjacency — two routers that appeared directly connected in at least one traceroute path. Links are undirected: if N1967 and N2129 are linked, traffic can flow either way. A link can also connect more than two nodes.

### Border Routers

A **border router** is a router whose node belongs to one AS while one of its link neighbors belongs to a different AS. Border routers are the physical interconnection points where one network hands traffic to another. Every packet crossing an AS boundary passes through at least one pair of border routers.

### AS Assignment at the Router Level

The ITDK assigns each node to an AS based on the interface IPs' BGP origin — the AS that originates the most-specific prefix covering that IP in a BGP routing table snapshot. This is different from prefix-level analysis (as in _nids-bgp-control-plane_), where you see which AS announces a block of addresses. Here, you see which AS physically operates a specific piece of hardware.

## Geolocation Methods

The ITDK provides geolocation for nodes using two independent methods. Both appear in the `method` column of `itdk_node_geolocation` as `'hoiho'` or `'maxmind'`.

### Hoiho — Hostname-Based Geolocation

Many network operators embed structured location hints in the DNS PTR (reverse lookup) records they assign to router interfaces. Common patterns include:

- **IATA airport codes**: `lax` = Los Angeles, `ams` = Amsterdam, `ord` = Chicago, `sjc` = San Jose
- **City abbreviations**: `nyc`, `lon`, `fra`, `par`
- **Structured operator hostnames**: `xe-0-0-0.cr1.lax01.alter.net`

Hoiho parses these patterns and maps them to geographic coordinates. When a PTR record contains a recognizable location code, Hoiho's estimate is typically accurate to the city level.

**Limitation:** PTR records are set manually by network operators and may not be updated when equipment is moved or IP addresses are reassigned. A hostname containing `lax` for a router now physically located in Dallas is a _stale hostname_ — the geolocation method extracted a historically accurate city code that no longer reflects reality.

### Maxmind — IP Registry Geolocation

Maxmind infers location from IP address registry records maintained by Regional Internet Registries (ARIN, RIPE, APNIC, etc.). It assigns a location to each IP prefix based on where the address block was registered or where the organization's registered address is.

**Limitation:** Global transit providers allocate large IP address blocks centrally, often with the organization's headquarters address as the registered location. When Maxmind cannot find a sub-allocation record specific enough to identify the PoP, it falls back to the organization's headquarters city — which may be thousands of kilometers from where the router actually sits.

### When Methods Agree vs. Disagree

When both methods agree on the location, confidence is high. When they disagree — one says Chicago and the other says Stockholm — the discrepancy is a signal worth investigating:

- **Stale hostname**: the PTR record encodes an old location; Maxmind reflects the current IP registration.
- **Maxmind HQ fallback**: the IP registry places the organization at its headquarters; Hoiho correctly identifies the PoP from the hostname.
- **Genuine uncertainty**: neither method has precise data for this particular IP.

Task 1 exercises this directly: a handful of Level3–Netflix links come out non-adjacent, and distinguishing a genuine long-distance circuit from a geolocation error is exactly this kind of judgment call.

### Optional Reading

- [CAIDA ITDK Dataset](https://www.caida.org/catalog/datasets/internet-topology-data-kit/) — full dataset description, data formats, and download links
- [Hoiho: Alias Resolution and Geolocation](https://www.caida.org/publications/papers/2021/hoiho/) — the paper describing the hostname-based geolocation and alias resolution methodology

[README](README.md) | Introduction ⮕ | [Background](Background.md) | [Datasets](Datasets.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-itdk.ipynb)
