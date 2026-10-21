[README](README.md) | [Introduction](Introduction.md) | [Background](Background.md) | [Datasets](Datasets.md) | SQL ⮕ | [Tasks](Task.md) | [Task 1 Guide](Task-1.md) | [Task 2 Guide](Task-2.md) | [Task 3 Guide](Task-3.md) | [Notebook](nids-itdk.ipynb)

# SQL Guide: Querying the ITDK from Python

This guide builds up the SQL you need for the three tasks, one piece at a time,
and shows how to run each query from Python so the result comes back as a pandas
DataFrame. Every query targets the `caida_itdk` schema described in
[Datasets](Datasets.md); read that page first for the full column list and the
[Writing Efficient Queries](Datasets.md#writing-efficient-queries) rules that
the examples below follow.

Each [Task guide](Task.md) contains the complete, commented query for that task.
This page explains the *building blocks* those queries are assembled from.

## 1. Running SQL from Python

The notebook opens one SQLAlchemy `engine` (see [Datasets](Datasets.md#connecting)):

```python
from sqlalchemy import create_engine
engine = create_engine(READ_DSN)   # READ_DSN comes from db_credentials.env
```

You never call the database directly. Instead, hand a SQL string and the
`engine` to `pandas.read_sql`, which runs the query and returns the rows as a
DataFrame — one DataFrame column per selected SQL column:

```python
import pandas as pd

df = pd.read_sql("""
    SELECT node_id, asn
    FROM caida_itdk.itdk_node_as
    WHERE asn = 3356
    LIMIT 5
""", engine)

df.head()   # a normal pandas DataFrame from here on
```

Two useful variations you will see in the tasks:

- **Triple-quoted strings** (`"""..."""`) let the SQL span many lines.
- **f-strings** (`f"""..."""`) let you interpolate Python values into the SQL
  (see [§3.6](#36-interpolating-a-python-list-into-an-in-clause)). Only
  interpolate values you control — never untrusted input.

## 2. The tables

Four tables cover everything in this module (full detail in [Datasets](Datasets.md)):

| Table | One row per… | Key columns |
| --- | --- | --- |
| `itdk_node_as` | (router node, AS) | `node_id`, `asn` |
| `itdk_node_geolocation` | router node | `node_id`, `country`, `city`, `latitude`, `longitude` |
| `itdk_link_endpoints` | link endpoint | `link_id`, `node_id`, `endpoint_token` |
| `itdk_router_hostnames` | (IP, hostname) | `ip`, `hostname` |

Remember the grain: `itdk_node_as` and `itdk_node_geolocation` have (at most) one
row per `node_id`, while `itdk_link_endpoints` has many rows per `link_id` (one
per endpoint). Joining the one-row-per-node tables is cheap; self-joining the
link table multiplies rows and must be done carefully.

## 3. Building blocks

### 3.1 Filter routers by AS

`itdk_node_as` maps each router node to the AS that operates it, and is indexed
on `asn`, so filtering by AS number is fast:

```python
level3 = pd.read_sql("""
    SELECT node_id
    FROM caida_itdk.itdk_node_as
    WHERE asn = 3356          -- Level3
""", engine)
```

### 3.2 Enrich a node with its location

`itdk_node_geolocation` has one row per `node_id` (primary key), so joining it
onto a set of nodes adds columns without adding rows:

```python
located = pd.read_sql("""
    SELECT na.node_id, g.city, g.country, g.latitude, g.longitude
    FROM caida_itdk.itdk_node_as na
    JOIN caida_itdk.itdk_node_geolocation g ON g.node_id = na.node_id
    WHERE na.asn = 4837       -- China Unicom
""", engine)
```

Use an **INNER** `JOIN` when you want only nodes that have a location; use a
`LEFT JOIN` when you want to keep nodes even if they are not geolocated.

### 3.3 Both endpoints of a link (self-join)

A link connects two routers, but `itdk_link_endpoints` stores each endpoint on
its own row. To see both ends together, join the table to *itself* on `link_id`,
excluding the row from matching itself:

```sql
SELECT e1.link_id, e1.node_id AS node_a, e2.node_id AS node_b
FROM caida_itdk.itdk_link_endpoints e1
JOIN caida_itdk.itdk_link_endpoints e2
    ON e1.link_id = e2.link_id       -- same link…
   AND e1.node_id <> e2.node_id      -- …but the other endpoint
```

This is the core pattern for finding a router's neighbours (its *peers*): start
from a router on `e1`, and `e2.node_id` is whatever sits on the far end of each
of its links.

### 3.4 Staging a query with a CTE (`WITH`)

A **Common Table Expression** (`WITH name AS (...)`) names an intermediate
result so the main query can read from it. CTEs keep multi-step queries readable
and — crucially here — let you **collapse the grain before a fan-out**. Two
patterns appear in the tasks:

- Build a small seed set once, then join outward from it (Task 2's `cu` CTE of
  West Coast routers).
- Reduce a huge table to just the columns you need with `DISTINCT`, then
  self-join the small result (Task 3's `le_as` CTE).

```sql
WITH cu AS (                          -- step 1: a small, filtered seed set
    SELECT node_id, city
    FROM caida_itdk.itdk_node_as a
    JOIN caida_itdk.itdk_node_geolocation g ON g.node_id = a.node_id
    WHERE a.asn = 4837 AND g.country = 'US' AND g.longitude < -115
)
SELECT * FROM cu;                     -- step 2: build outward from cu
```

### 3.5 Interface hostnames (`split_part`, `NULLIF`, `::inet`)

`itdk_router_hostnames` is keyed by interface IP (`ip`, type `inet`), but the
link table carries the interface as text inside `endpoint_token`, formatted
`something:1.2.3.4`. To join them, extract the IP and cast it:

```sql
LEFT JOIN caida_itdk.itdk_router_hostnames h
    ON h.ip = NULLIF(split_part(e2.endpoint_token, ':', 2), '')::inet
```

- `split_part(endpoint_token, ':', 2)` takes the text after the `:`.
- `NULLIF(..., '')` turns an empty string into `NULL` so the `::inet` cast
  doesn't fail on endpoints that carry no IP.
- `::inet` casts the text to Postgres's IP type to match `h.ip`.
- Keep it a **`LEFT JOIN`**: many interfaces have no PTR record, and you still
  want the row (with `hostname` = `NULL`).

### 3.6 Interpolating a Python list into an `IN` clause

To filter on a list of ASNs held in Python, turn the list into a comma-separated
string and drop it into the SQL with an f-string:

```python
asn_list = [174, 701, 1299, 3257]
asn_sql = ", ".join(str(a) for a in asn_list)   # "174, 701, 1299, 3257"

pair_df = pd.read_sql(f"""
    SELECT node_id, asn
    FROM caida_itdk.itdk_node_as
    WHERE asn IN ({asn_sql})
""", engine)
```

Because the string is integers you built yourself, this is safe. Inside an
f-string, remember that any *literal* `{` or `}` in the SQL must be doubled
(`{{`, `}}`); `{asn_sql}` is the one thing being substituted.

### 3.7 Counting and de-duplicating pairs

`GROUP BY` collapses rows into groups; the aggregate then summarises each group.
Because the link table has repeated rows, count **distinct** ids to avoid
over-counting:

```sql
SELECT asn, country, COUNT(DISTINCT node_id) AS node_count
FROM caida_itdk.itdk_node_as na
JOIN caida_itdk.itdk_node_geolocation g ON g.node_id = na.node_id
WHERE asn IN (4837, 3356)
GROUP BY asn, country
```

When you count links *between* two ASes, a self-join produces each pair twice
(A–B and B–A). Keep only one direction with `a.asn < b.asn`:

```sql
SELECT a.asn AS asn1, b.asn AS asn2, COUNT(*) AS link_count
FROM le_as a
JOIN le_as b ON b.link_id = a.link_id AND a.asn < b.asn
GROUP BY a.asn, b.asn
```

## 4. Using the results in pandas

Once a query returns a DataFrame, the analysis is plain pandas. The idioms the
tasks use:

- **Row-wise computation** — apply a Python function across each row, e.g. the
  haversine distance between a link's two endpoints:

  ```python
  geo_df["distance_km"] = geo_df.apply(
      lambda r: haversine(r.l3_lat, r.l3_lon, r.nf_lat, r.nf_lon), axis=1
  )
  ```

- **Count distinct within groups** — `groupby(...).agg(nunique)` mirrors SQL's
  `GROUP BY … COUNT(DISTINCT …)` when you'd rather do it in pandas:

  ```python
  peering_locations = (
      adjacent.groupby(["l3_city", "l3_country"])
              .agg(link_count=("link_id", "nunique"))
              .reset_index()
  )
  ```

- **Reshape long → wide** — `pivot_table` turns one row per (city, peer) into a
  city × peer matrix:

  ```python
  wide = long_df.pivot_table(index="cu_city", columns="peer_asn",
                             values="router_count", fill_value=0)
  ```

- **Label codes with names** — `Series.map(dict)` swaps ASNs for readable names:

  ```python
  pair_df["name1"] = pair_df["asn1"].map(ASES)
  ```

## 5. Writing efficient queries

These tables run from ~2 million to ~14 million rows, so query shape matters.
The full rules are in [Datasets](Datasets.md#writing-efficient-queries); the
short version:

- **Filter on an indexed column first** (`asn`, `node_id`, `link_id`,
  `hostname`) — never start from a full-table scan.
- **Collapse the grain before you fan out.** `SELECT DISTINCT link_id, asn`
  inside a CTE, *then* self-join, instead of self-joining the raw 14-million-row
  link table.
- **Attach one-row-per-node tables (AS, geolocation) to a small seed set first**,
  and save row-multiplying joins for last.
- **Prototype with `LIMIT`**, confirm the plan with `EXPLAIN`, and select only
  the columns you need — avoid `SELECT *`.

[README](README.md) | [Introduction](Introduction.md) | [Background](Background.md) | [Datasets](Datasets.md) | SQL ⮕ | [Tasks](Task.md) | [Task 1 Guide](Task-1.md) | [Task 2 Guide](Task-2.md) | [Task 3 Guide](Task-3.md) | [Notebook](nids-itdk.ipynb)
