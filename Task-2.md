[README](README.md) | [Introduction](Introduction.md) | [Background](Background.md) | [Datasets](Datasets.md) | [SQL](SQL.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | Task 2 ⮕ | [Task 3](Task-3.md) | [Notebook](nids-itdk.ipynb)

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

Push the per-country histogram into a single SQL query — one `GROUP BY` covering
both ASNs — and do the ranking, percentages, and country-name lookup afterwards
in pandas:

```sql
SELECT
    na.asn,                                   -- 4837 (China Unicom) or 3356 (Level3)
    g.country,                                -- ISO 2-letter code, e.g. 'CN', 'US'
    COUNT(DISTINCT na.node_id) AS node_count  -- routers of that AS in that country
FROM caida_itdk.itdk_node_as na
JOIN caida_itdk.itdk_node_geolocation g ON g.node_id = na.node_id
WHERE na.asn IN (4837, 3356)                  -- both ASes in one round trip
GROUP BY na.asn, g.country;                   -- one row per (AS, country)
```

- Filtering on `na.asn` uses the index on that column, and attaching the
  one-row-per-node geolocation table before aggregating keeps the row count
  small — both [efficiency rules](SQL.md#5-writing-efficient-queries).
- `COUNT(DISTINCT na.node_id)` counts each router once even though the join
  could, in principle, repeat a node; see [SQL §3.7](SQL.md#37-counting-and-de-duplicating-pairs).
- Putting both ASNs in one `IN (...)` returns both distributions together, so
  you split them apart in pandas (`df[df.asn == 4837]` vs `3356`).

Then, in pandas: for each ASN compute `rank` (0 = most routers; ties share a
rank, e.g. `rank(method="min") - 1`) and `pct` (share of that AS's total),
map the `country` code to a name with `pycountry.countries.get(alpha_2=...)`,
and join China Unicom's top-10 countries against Level3's counts in the same
countries.

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

For each West Coast China Unicom router, join `itdk_link_endpoints` to itself on `link_id` to find the other endpoint of each link, then join that peer node to `itdk_node_as` (excluding AS4837 itself) and `itdk_router_hostnames` (for the peer interface's PTR record, if any). This is the complete query used in the notebook (it returns `cu_peers_df`):

```sql
WITH cu AS (
    -- Step 1: the seed set — West Coast China Unicom routers.
    -- Filter itdk_node_as to AS4837, attach geolocation, and keep only US
    -- nodes west of longitude -115 (see "Filtering for US West Coast Nodes").
    SELECT a.node_id, g.city
    FROM caida_itdk.itdk_node_as a
    JOIN caida_itdk.itdk_node_geolocation g ON g.node_id = a.node_id
    WHERE a.asn = 4837 AND g.country = 'US' AND g.longitude < -115
)
SELECT DISTINCT                               -- collapse duplicate (router, peer) rows
    cu.node_id  AS cu_node,
    cu.city     AS cu_city,
    e2.node_id  AS peer_node,
    a2.asn      AS peer_asn,
    h.hostname  AS peer_hostname
FROM cu
-- Step 2: walk each CU router's links to the router on the far end.
JOIN caida_itdk.itdk_link_endpoints e1 ON e1.node_id = cu.node_id   -- CU end of a link
JOIN caida_itdk.itdk_link_endpoints e2
    ON e2.link_id = e1.link_id       -- same link…
   AND e2.node_id <> cu.node_id      -- …but the OTHER endpoint (the peer)
JOIN caida_itdk.itdk_node_as a2
    ON a2.node_id = e2.node_id       -- who operates the peer router?
   AND a2.asn <> 4837                -- exclude China Unicom's own routers
-- Step 3: the peer interface's PTR hostname, if it has one.
LEFT JOIN caida_itdk.itdk_router_hostnames h
    ON h.ip = NULLIF(split_part(e2.endpoint_token, ':', 2), '')::inet;
```

The self-join on `link_id` (`e1`/`e2`) is the [both-endpoints pattern](SQL.md#33-both-endpoints-of-a-link-self-join);
the `LEFT JOIN` with `split_part`/`NULLIF`/`::inet` is the
[hostname-lookup pattern](SQL.md#35-interface-hostnames-split_part-nullif-inet).
In the notebook the outer query is given and you complete the `cu` CTE.

The result gives you one row per (China Unicom router, peer router) pair, with columns `cu_node | cu_city | peer_node | peer_asn | peer_hostname`. From this, report how many distinct China Unicom routers have at least one peer, and how many distinct peer ASNs appear — these two counts, compared against each other, tell you something about how concentrated the interconnection is.

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

[README](README.md) | [Introduction](Introduction.md) | [Background](Background.md) | [Datasets](Datasets.md) | [SQL](SQL.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | Task 2 ⮕ | [Task 3](Task-3.md) | [Notebook](nids-itdk.ipynb)
