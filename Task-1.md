[README](README.md) | [Introduction](Introduction.md) | [Background](Background.md) | [Datasets](Datasets.md) | [SQL](SQL.md) | [Tasks](Task.md) | Task 1 ⮕ | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-itdk.ipynb)

# Task 1: Border Routers Between Level3 and Netflix — How-To Guide

## Querying Cross-AS Links

To find router-level links between two specific ASes, build one router-set per
AS — each already enriched with geolocation — and join the two sets on
`link_id`. The notebook stages each set in a [CTE](SQL.md#34-staging-a-query-with-a-cte-with)
(`level3_nodes`, `netflix_nodes`) and returns the result into a DataFrame
(`geo_df`) with `pd.read_sql`. This is the complete query used in the notebook:

```sql
WITH level3_nodes AS (
    -- step 1: every Level3 (AS3356) router endpoint, with its location.
    -- Start from the link table (one row per endpoint), attach the AS so we can
    -- filter to Level3, and attach geolocation. Both joins are on node_id:
    --   * itdk_node_as is indexed on asn, so `asn = 3356` is a cheap lookup;
    --   * itdk_node_geolocation has one row per node (INNER join => keep only
    --     endpoints we can actually place on a map).
    SELECT
        le.link_id,                       -- the link this endpoint belongs to
        le.node_id AS l3_node_id,         -- the Level3 router on this end
        g.latitude AS l3_lat,
        g.longitude AS l3_lon,
        g.city AS l3_city,
        g.region AS l3_region,
        g.country AS l3_country
    FROM caida_itdk.itdk_link_endpoints le
    JOIN caida_itdk.itdk_node_as na ON le.node_id = na.node_id
    JOIN caida_itdk.itdk_node_geolocation g ON le.node_id = g.node_id
    WHERE na.asn = 3356                   -- Level3
),
netflix_nodes AS (
    -- step 2: the same thing for Netflix (AS2906). Identical shape to
    -- level3_nodes, but filtered to asn = 2906 and aliased nf_* so step 3 can
    -- tell the two endpoints apart.
    SELECT
        le.link_id,
        le.node_id AS nf_node_id,
        g.latitude AS nf_lat,
        g.longitude AS nf_lon,
        g.city AS nf_city,
        g.region AS nf_region,
        g.country AS nf_country
    FROM caida_itdk.itdk_link_endpoints le
    JOIN caida_itdk.itdk_node_as na ON le.node_id = na.node_id
    JOIN caida_itdk.itdk_node_geolocation g ON le.node_id = g.node_id
    WHERE na.asn = 2906                   -- Netflix
)
-- step 3: a link is a Level3<->Netflix link exactly when the SAME link_id
-- appears in both sets. Joining on link_id keeps only those links and puts both
-- endpoints (and both locations) on one row.
SELECT
    l3.link_id,
    l3.l3_node_id, l3.l3_lat, l3.l3_lon, l3.l3_city, l3.l3_region, l3.l3_country,
    n.nf_node_id, n.nf_lat, n.nf_lon, n.nf_city, n.nf_region, n.nf_country
FROM level3_nodes l3
JOIN netflix_nodes n ON l3.link_id = n.link_id;
```

Run it from Python and the rows come back as a DataFrame:

```python
geo_df = pd.read_sql(that_sql, engine)   # one row per Level3<->Netflix link
```

In the notebook, `level3_nodes` is given as a worked example and you complete
`netflix_nodes` by analogy. Each surviving row is a single link with both
endpoints' coordinates — exactly what the distance step below needs.

> An equivalent way to write this is a single [self-join](SQL.md#33-both-endpoints-of-a-link-self-join)
> of `itdk_link_endpoints` (`e1`/`e2` on `link_id`) with an AS filter on each
> side; the two-CTE form above is used because it enriches each side with
> geolocation before the sets are joined.

## Computing the Haversine Distance

Once you have lat/lon for both endpoints, the haversine formula gives the great-circle distance in km:

```python
import math

def haversine(lat1, lon1, lat2, lon2):
    R = 6371  # Earth radius in km
    dlat = math.radians(lat2 - lat1)
    dlon = math.radians(lon2 - lon1)
    a = (math.sin(dlat / 2) ** 2
         + math.cos(math.radians(lat1)) * math.cos(math.radians(lat2))
         * math.sin(dlon / 2) ** 2)
    return 2 * R * math.asin(math.sqrt(a))

geo_df["distance_km"] = geo_df.apply(
    lambda r: haversine(r.l3_lat, r.l3_lon, r.nf_lat, r.nf_lon), axis=1
)
```

## Classifying Adjacent vs. Non-Adjacent Links

Once every link has a computed `distance_km`, classify each as **adjacent** (`<= 40 km`) or **non-adjacent** (`> 40 km`) and count how many links fall into each bucket. Report both the counts and the percentages.

As described in [Introduction](Introduction.md), when a Hoiho-geolocated endpoint is paired with a Maxmind-geolocated endpoint, large apparent distances are more likely to reflect a geolocation error than a genuine long-distance circuit. Before concluding that a large distance represents a real long-haul link, check whether it's just over the 40 km threshold (likely a genuine nearby link with imprecise coordinates) versus thousands of kilometers away (worth flagging as a probable geolocation artifact rather than treating at face value).

## Peering Locations (Adjacent Links Only)

For the adjacent-link subset, group by `city`/`country` and count distinct links per location, producing a table like:

| city | country | link_count |
| --- | --- | --- |
| *(example city)* | *(example country)* | *(count)* |

Also report the total number of distinct peering locations found.

## What Your Write-Up Should Address

Question 1 asks for three short answers (Q1.a–c). Ground each in what your own query actually returns, not in the example above:

- **Q1.a** — How many distinct metropolitan areas do Level3 and Netflix peer at, and how is that count distributed across cities? Is it concentrated in a few hubs or spread broadly?
- **Q1.b** — What does it mean when the same city has more than one link? Think about what a "link" represents at the router level versus what a single peering agreement or facility represents.
- **Q1.c** — Why would a Tier-1 transit provider and a large content provider prefer to peer at the same physical location (adjacent) rather than across a long distance? What would explain the links that come out non-adjacent — are they likely genuine long-haul circuits, or could some be geolocation error given the caveat above?

[README](README.md) | [Introduction](Introduction.md) | [Background](Background.md) | [Datasets](Datasets.md) | [SQL](SQL.md) | [Tasks](Task.md) | Task 1 ⮕ | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-itdk.ipynb)
