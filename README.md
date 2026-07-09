README ⮕ | [Introduction](Introduction.md) | [Background](Background.md) | [Datasets](Datasets.md) | [SQL](SQL.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-itdk.ipynb)

# Exploring Internet Topology with CAIDA's ITDK

**GitHub:** https://github.com/CAIDA/nids-itdk

## Learning Objectives

In this module you will use CAIDA's Internet Topology Data Kit (ITDK) to explore router-level Internet topology. You will query a PostgreSQL database of traceroute-inferred links between routers, use their geolocation, ASN, and hostname annotations to locate and attribute each router, and analyze how major autonomous systems interconnect. By the end of this module you should understand how Tier-1 transit providers, large ISPs, CDNs, and content networks interconnect at the router level, and be able to judge when a router's inferred geolocation is trustworthy.

## Overview

Start by reading **Introduction** to get the background needed for the assignment. **Background** introduces the three autonomous systems (Level3, Netflix, China Unicom) examined throughout the module. **Datasets** explains the database schema and the role of placeholder nodes.

- step 1 [read the introduction](Introduction.md)
- step 2 [read the AS background](Background.md)
- step 3 [read the dataset overview](Datasets.md)
- step 4 [review the tasks](Task.md)
- step 5 set up your local Python environment (see "Running Locally" below)
- step 6 launch Jupyter and complete nids-itdk.ipynb
  - complete each task by replacing the `# YOUR CODE HERE` sections
  - answer all eight questions
- step 7 save the notebook, commit, and push to github

### Running Locally

This notebook runs on your own machine — no hosted JupyterHub required. Pick one:

```bash
# Option A: uv
uv sync
uv run jupyter lab nids-itdk.ipynb
```

```bash
# Option B: pip + venv
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
jupyter lab nids-itdk.ipynb
```

Before running any database queries, set up `db_credentials.env` — see [Datasets](Datasets.md#connecting).

### Directory Structure

```
nids-itdk
├- Introduction.md                          # Introduction and background
├- Background.md                            # Level3, Netflix, and China Unicom background
├- Datasets.md                              # Database schema and dataset notes
├- Task.md                                  # Task checklist and instructions
├- Task-1.md                                # Task 1 how-to guide
├- Task-2.md                                # Task 2 how-to guide
├- Task-3.md                                # Task 3 how-to guide
├- nids-itdk.ipynb                      ⬅  # Complete / Commit / Push
├- requirements.txt                         # Dependencies (pip + venv)
├- pyproject.toml / uv.lock                 # Dependencies (uv)
```

### Glossary

- **ITDK (Internet Topology Data Kit)**: A CAIDA dataset built from large-scale traceroute campaigns, containing the inferred router-level topology of the Internet.
- **Node**: A single router, represented in the ITDK as the set of all IP addresses inferred to belong to that router.
- **Link**: An inferred physical adjacency between two routers, observed when one router's IP appears directly before another's in a traceroute path.
- **Alias Resolution**: The process of determining which interface IP addresses belong to the same physical router.
- **Traceroute**: A network measurement technique that probes each hop along a path to a destination, using ICMP TTL-exceeded replies to reveal intermediate routers.
- **Border Router**: A router that sits at the edge of an AS, connecting it to a neighboring AS at the router level.
- **Geolocation**: The assignment of a physical location (city, country, latitude/longitude) to a router's IP address or hostname.
- **Hoiho**: CAIDA's hostname-based geolocation method, which parses DNS PTR records for IATA airport codes and city abbreviations to infer router location.
- **Maxmind**: An IP-geolocation database that infers location from IP registry records; falls back to the organization's registered address when fine-grained data is unavailable.
- **Stale Hostname**: A DNS PTR record that was accurate when assigned but no longer reflects the router's current physical location.
- **Haversine Distance**: The great-circle distance between two lat/lon points on Earth's surface, used here to measure the physical separation between router endpoints on a link.
- **Tier-1 Transit**: An AS that provides transit to other ASes without purchasing upstream transit itself, because it has settlement-free peering agreements with all other Tier-1s.
- **Peering**: A bilateral agreement between two ASes to exchange traffic directly, rather than through a third-party transit provider.
- **CDN (Content Delivery Network)**: A network of servers distributed geographically to serve content closer to end users; examples include Akamai, Cloudflare, and Edgecast.
- **Placeholder Node**: An ITDK node representing a router that did not respond to traceroute probes but whose existence was inferred from surrounding hops. Placeholder nodes have synthetic IP addresses in reserved ranges and carry no geolocation, AS, or hostname data.

README ⮕ | [Introduction](Introduction.md) | [Background](Background.md) | [Datasets](Datasets.md) | [SQL](SQL.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-itdk.ipynb)
