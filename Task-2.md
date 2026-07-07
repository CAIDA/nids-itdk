[README](README.md) | [Introduction](Introduction.md) | [Background](Background.md) | [Datasets](Datasets.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | Task 2 ⮕ | [Task 3](Task-3.md) | [Notebook](nids-itdk.ipynb)

# Task 2: China Unicom — How-To Guide

## Country-Wide Comparison: China Unicom vs. Level3

Before narrowing to the US West Coast, look at China Unicom's (AS4837) router footprint globally, and compare it against Level3 (AS3356). Group geolocated nodes by `country` for both ASNs in one query, then rank each ASN's countries by router count. Rank is defined as the number of countries with strictly more routers (ties share a rank):

| country | number_router | rank |
|:---:|---:|---:|
| A  | 23 | 0 |
| B  | 21 | 1 |
| C  | 21 | 1 |
| D  | 18 | 3 |
| E  |  0 | 4 |
| F  |  0 | 4 |

Use `pycountry` to map the ISO 2-letter `country` code to a full country name.

Report the total number of geolocated nodes for each ASN, then produce a joined table with columns like `country_name | cu_num_router | cu_rank | cu_pct | l3_pct | l3_rank | l3_num_routers` — `pct` is each country's share of that ASN's total geolocated nodes, so the two ASNs' distributions are directly comparable regardless of their different totals.

## Filtering for US West Coast Nodes

China Unicom's non-Chinese footprint is small in absolute terms, but its US West Coast presence is where it directly touches other major networks. To isolate it, filter geolocated AS4837 nodes by `country = 'US'` and a longitude threshold:

```python
us_west_df = cu_nodes_df[
    (cu_nodes_df["country"] == "US") &
    (cu_nodes_df["longitude"] < -115)
]
```

Longitude < −115° captures roughly everything west of the Rocky Mountains: California, Oregon, Washington, Nevada, and Arizona. The −115th meridian passes through Nevada; this threshold includes the major Pacific coast cities while excluding inland hubs like Denver or Salt Lake City.

## Identifying Peer ASes at the Router Level

For each West Coast China Unicom router, join `itdk_link_endpoints` to itself on `link_id` to find the other endpoint of each link, then join that peer node to `itdk_node_as` (excluding AS4837 itself) and `itdk_router_hostnames` (for the peer interface's PTR record, if any).

The result should give you one row per (China Unicom router, peer router) pair, with columns like `cu_node | cu_city | peer_node | peer_asn | peer_hostname`. From this, report how many distinct China Unicom routers have at least one peer, and how many distinct peer ASNs appear — these two counts, compared against each other, tell you something about how concentrated the interconnection is.

## Reading Interface Hostnames

Interface hostnames frequently embed structured location and operator information. For example, a hostname pattern like this on a peer interface:

```
ws-chinaunicom-hk-operationsltd.customer.alter.net
```

would break apart as:
- `alter.net` — the domain belongs to Lumen/CenturyLink (formerly Alter.net), identifying the peer network's operator
- `chinaunicom-hk` — China Unicom's Hong Kong operations entity, the legal entity that holds the peering agreement with Lumen
- `customer` — this interface is on the Lumen side, classifying China Unicom as a transit customer

(This is illustrative of the naming convention, not a guaranteed literal match in the current data — check what `peer_hostname` actually returns for your query.) Common IATA city codes in hostnames: `lax` (Los Angeles), `sjc` (San Jose), `sfo` (San Francisco), `sea` (Seattle), `ord` (Chicago), `nyc` (New York).

## Top West Coast Cities by Peer AS

Focusing on `cu_peers_df`, take the top 10 West Coast cities by number of distinct connected China Unicom routers, and for each city break down how many of those routers connect to each peer ASN — a table with one row per city, a `total` column, and one column per peer ASN. If the number of distinct peer ASNs is large, consider showing only the busiest few columns for readability and noting how many were omitted.

Use a lookup (e.g. a small hardcoded ASN → name dict, or an external ASN name source if available) to label peer ASNs by network name rather than leaving them as bare numbers — this makes the table interpretable without cross-referencing.

**Note on join behavior**: the peer-identification query inner-joins each peer node to `itdk_node_as`, so any peer without an AS assignment is excluded from the result set entirely rather than appearing as a `peer_asn = NULL` row. Only `peer_hostname` can be NULL, via the `LEFT JOIN` to `itdk_router_hostnames` — that's expected for peer interfaces without a PTR record, and does not need to be filtered out. See [Datasets](Datasets.md) for background on why some interfaces have no PTR record.

## What Your Write-Up Should Address

Ground each answer in what your own queries actually return, not in the illustrative examples above:

- **Q2.a** — How does the country-level concentration of China Unicom's routers compare with Level3's? What does each network's business model (national domestic carrier vs. Tier-1 backbone) suggest about why their footprints would differ?
- **Q2.b** — Which ASN has more total geolocated routers, and why might that follow from the two networks' different roles and scale?
- **Q2.c** — Build the top-10 West Coast cities × peer-ASN table (see *Top West Coast Cities by Peer AS* above): for each city, the total number of distinct China Unicom routers with at least one peer, broken down by peer ASN. Label peer ASNs by network name rather than bare numbers, and note how many columns (if any) you collapsed for readability.
- **Q2.d** — Looking at your West Coast city/peer-ASN table: why would China Unicom connect to the *same* peer AS more than once *at the same location*? Why would it connect to the same peer AS at *different* locations? Consider what a single router-level link represents versus what a citywide or national interconnection relationship represents.

[README](README.md) | [Introduction](Introduction.md) | [Background](Background.md) | [Datasets](Datasets.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | Task 2 ⮕ | [Task 3](Task-3.md) | [Notebook](nids-itdk.ipynb)
