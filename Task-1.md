[README](README.md) | [Introduction](Introduction.md) | [Background](Background.md) | [Datasets](Datasets.md) | [Tasks](Task.md) | Task 1 ⮕ | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-itdk.ipynb)

# Task 1: Border Routers Between Level3 and Netflix — How-To Guide

## Querying Cross-AS Links

To find router-level links between two specific ASes, join `itdk_link_endpoints` twice on the same `link_id`, then filter each side by AS number:

```sql
SELECT DISTINCT
    e1.link_id,
    e1.node_id   AS l3_node,
    e2.node_id   AS nf_node
FROM caida_itdk.itdk_link_endpoints e1
JOIN caida_itdk.itdk_link_endpoints e2
    ON e1.link_id = e2.link_id AND e1.node_id != e2.node_id
JOIN caida_itdk.itdk_node_as a1
    ON e1.node_id = a1.node_id AND a1.asn = 3356
JOIN caida_itdk.itdk_node_as a2
    ON e2.node_id = a2.node_id AND a2.asn = 2906
```

Add joins to `itdk_node_geolocation` (for latitude/longitude) and `itdk_router_hostnames` (for PTR records) to enrich each endpoint. Use `DISTINCT` on `link_id` to collapse multi-interface rows for the same link.

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

[README](README.md) | [Introduction](Introduction.md) | [Background](Background.md) | [Datasets](Datasets.md) | [Tasks](Task.md) | Task 1 ⮕ | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-itdk.ipynb)
