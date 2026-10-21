[README](README.md) | [Introduction](Introduction.md) | Background ⮕ | [Datasets](Datasets.md) | [SQL](SQL.md) | [Tasks](Task.md) | [Task 1 Guide](Task-1.md) | [Task 2 Guide](Task-2.md) | [Task 3 Guide](Task-3.md) | [Notebook](nids-itdk.ipynb)

## Background

This module examines the router-level topology of three autonomous systems that occupy very different positions in the Internet's economic and physical structure: **Level3 (AS3356)**, a large global transit ISP; **Netflix (AS2906)**, a large content provider; and **China Unicom (AS4837)**, a national ISP that serves as one of a small number of international gateways for a large domestic network. Each network exists to solve a different problem, and each problem leaves a distinct fingerprint on the router-level graph you will query in the tasks that follow. Keeping these business models in mind will help you interpret what you find.

### Level3 / Lumen (AS3356) — ISP

Level 3 Communications operated one of the largest long-haul fiber backbones in the world and was a founding-generation global transit ISP (the network is now part of Lumen Technologies, but the ASN and much of the infrastructure persist as "Level 3 Parent, LLC"). Its defining trait is that it reaches the entire Internet through settlement-free peering with the other largest carriers and paid transit customers, never buying transit itself. Its business is to sell connectivity — to smaller ISPs who need upstream reach, to enterprises who need private circuits, and to content networks who need to get their traffic close to eyeballs.

**What it offers:** raw, global IP transit and long-haul fiber capacity. Level3's customers are other networks, not end users, so its value proposition is coverage and capacity, not any particular piece of content.

**Expected topological signature:**

- A very large number of border routers, connected to an unusually wide diversity of neighboring ASes — Level3 peers or transits with essentially every category of network (other large ISPs, regional ISPs, CDNs, content providers, and enterprises).
- Points of interconnection spread across many metro areas worldwide, concentrated in the large carrier-neutral colocation facilities and Internet exchange points that anchor global backbone routes.
- Because large transit ISPs build for global reach rather than proximity to any one customer, Level3's own PoPs tend to sit in major long-haul hub cities, and some links you observe to content providers may show larger physical distances between endpoints than you would expect from a network optimizing purely for latency.

### Netflix (AS2906) — Content Provider

Netflix generates a disproportionate share of the world's video streaming traffic, and its network engineering is built around one goal: get bytes as close to the viewer as possible while minimizing what it must pay to reach them. Historically, this is the same network at the center of the well-known 2014 Level3–Netflix peering dispute, when Level3 (then transiting traffic on Netflix's behalf) accused several last-mile ISPs of deliberately congesting interconnection ports — a dispute that only makes sense once you understand Netflix's incentive to peer directly rather than pay for transit wherever possible.

**What it offers:** video content, delivered at massive and highly asymmetric (outbound-heavy) volume. Netflix's Open Connect program places caching appliances directly inside many access ISPs' networks, but AS2906 itself is the network Netflix uses to reach the ISPs it does not embed hardware within, and to interconnect with the transit and peering fabric more broadly.

**Expected topological signature:**

- Far fewer border routers than a large ISP, and a narrower set of neighboring ASes — a content provider only needs to interconnect with networks that carry traffic to its viewers (large eyeball networks, IXP route servers, and a handful of transit providers for reaching smaller or distant networks).
- A strong preference for direct peering with large broadband/access networks at shared metro locations, rather than routing through multiple intermediate transit hops — this minimizes latency and cost simultaneously.
- Where Netflix does interconnect with an ISP like Level3, expect the links to cluster in a small number of well-defined, geographically adjacent peering locations rather than being spread evenly across the globe, since Netflix has little incentive to peer somewhere it doesn't need capacity.

### China Unicom (AS4837) — National Gateway ISP

China Unicom is one of China's state-affiliated telecommunications carriers and operates AS4837 ("China169"), one of a small number of ASes authorized to carry international IP traffic in and out of mainland China. Unlike Level3 or Netflix, its topology is shaped as much by regulation and national infrastructure policy as by commercial peering strategy: nearly all international traffic to and from China Unicom's domestic customers must pass through a limited set of state-sanctioned international gateways and submarine cable landing points.

**What it offers:** domestic transit and access within China, plus a constrained set of international on/off-ramps connecting the domestic network to the rest of the global Internet.

**Expected topological signature:**

- Router-level presence outside China concentrated in a small number of major international gateway cities rather than distributed broadly — historically, key hubs on the US West Coast (e.g., Los Angeles and the San Jose/Silicon Valley area) sit at the landing points of trans-Pacific submarine cables and major carrier-neutral facilities, making them natural places to find China Unicom routers.
- At each gateway location, expect to see China Unicom peering repeatedly with the same set of large transit and content networks — a consequence of a small number of gateways needing to reach a large number of destinations, so the same neighboring ASes appear at multiple router-level interconnection points.
- Compared to Level3's globally dispersed footprint, China Unicom's international presence should look comparatively sparse and clustered, reflecting a network optimized for a small number of controlled international chokepoints rather than pervasive global reach.

### The Multi-AS Landscape (Task 3): A Taxonomy of Autonomous Systems

Most of the roughly 70,000 ASNs on the Internet are **edge networks** run by a single organization — anything from a regional access ISP delivering broadband to its own subscribers to a business serving only itself. Their router-level footprint is narrow and predictable: buy transit from one or two upstream providers, maybe peer with a handful of directly adjacent networks, and little else. Level3, Netflix, and China Unicom above are each, in their own way, close to this single-purpose pattern — one sells transit, one delivers content, one provides domestic access with a constrained set of international gateways.

The largest ASes often break that pattern. At sufficient scale, a single AS can provide several of these functions at once, making it a **multi-service AS** whose router-level footprint layers more than one business model on top of another rather than reflecting just one. Verizon (AS701), for example, simultaneously operates a global transit backbone _and_ a residential access network (FiOS); Google (AS15169) is nominally a "content" network but operates a global backbone and cloud infrastructure comparable in scale to a global ISP's. Keep this in mind when interpreting Task 3's heatmap: the category table below describes each AS's _primary_ business, not the full range of what it actually does at the router level — a link involving a multi-service AS may exist for a different reason than its category label suggests.

| Category | What it optimizes for                                                                | Representative ASes                                                                                                |
| -------- | ------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| ISP      | Global and regional/national reach; sells transit and access to other networks and its own customers | Cogent, Verizon, Arelion, GTT, PCCW, Orange, TATA, Deutsche Telekom, Zayo, Telecom Italia, Liberty Global, Telefonica |
| CDN      | Distributing cached content close to end users                                       | Edgecast, Akamai                                                                                                  |
| Content  | Delivering its own content/services to end users at scale                            | Apple, Netflix, Cloudflare, Google                                                                                |

### Why This Matters for the Tasks

Tasks 1–2 each isolate a pairwise relationship (Level3–Netflix, China Unicom's country-wide and West Coast peers) so you can see one of these business models play out directly in the data. Task 3 broadens the lens across the 18 ASes above — watch for multi-service ASes complicating the clean categories, and for the differences between ISPs, content providers, and CDNs appearing as visible structure in the interconnection heatmap, rather than as something you have to take on faith from this background reading.

[README](README.md) | [Introduction](Introduction.md) | Background ⮕ | [Datasets](Datasets.md) | [SQL](SQL.md) | [Tasks](Task.md) | [Task 1 Guide](Task-1.md) | [Task 2 Guide](Task-2.md) | [Task 3 Guide](Task-3.md) | [Notebook](nids-itdk.ipynb)
