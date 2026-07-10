[README](README.md) | [Introduction](Introduction.md) | [Background](Background.md) | Datasets ⮕ | [SQL](SQL.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-itdk.ipynb)

## Database

This module uses a PostgreSQL database containing a relational representation of the 2026 ITDK. All tables live in the `caida_itdk` schema.
We focus for this study only on nodes that transited a packet in at least one traceroute. These are known routers, and restricting to them decreases the size of the database and should allow for faster development.

#### Tables

| Table                   | Size      | Description                                               |
| ----------------------- | --------- | --------------------------------------------------------- |
| `itdk_router_hostnames` | 414.45 MB | DNS PTR hostnames for interface IPs                       |
| `v_itdk_ifaces_transit` | 452.48 MB | Interface IPs for transit-observed nodes, with their link |
| `itdk_node_as`          | 391.16 MB | Router node → operating autonomous system                 |
| `itdk_node_geolocation` | 518.91 MB | Router node → physical location                           |
| `itdk_link_endpoints`   | 1.5 GB    | Inferred links → their endpoint nodes                     |

#### Indexes

Which columns are indexed on each table, so you can pick a `WHERE`/`JOIN` column that avoids a full-table scan. `PK` = primary-key column, `IX` = secondary index, `PK·IX` = both.

| Table                     | Rows | node_id | link_id | endpoint_ordinal | asn   | country | longitude | ip | hostname |
| ------------------------- | ---- | ------- | ------- | ---------------- | ----- | ------- | --------- | -- | -------- |
| `itdk_link_endpoints`     | 14M  | IX      | PK      | PK               |       |         |           |    |          |
| `itdk_node_as`            | 4M   | PK      |         |                  | PK·IX |         |           |    |          |
| `itdk_node_geolocation`   | 4M   | PK      |         |                  |       | IX      | IX        |    |          |
| `v_itdk_ifaces_transit`   | 3M   | IX      | IX      |                  |       |         |           |    |          |
| `itdk_router_hostnames`   | 2M   |         |         |                  |       |         |           | PK | PK·IX    |

Composite indexes only match left-to-right (see [Writing Efficient Queries](#writing-efficient-queries)): the leading column is listed first in the key above — `link_id` for `itdk_link_endpoints`, `node_id` for `itdk_node_as`, `country` for the geolocation index, `ip` for `itdk_router_hostnames`.

### Writing Efficient Queries

These tables range from under 400 MB to about 1.5 GB, with row counts from roughly two million up to fourteen million. A query that ignores this will either time out or come back so slowly that you can't iterate on it. Before you write a `SELECT` against any table below, check its indexed columns (listed with each table) and keep these in mind:

- **Never start from a full-table scan.** Every table has a primary key, and most have one or more secondary indexes — filter or join on those columns first (`node_id`, `asn`, `hostname`, `link_id`). A `WHERE` clause on a column with no index forces Postgres to read every row.
- **Composite indexes only help left-to-right.** The geolocation table's index is on `(country, longitude)` together — filtering by `country` alone uses it, filtering by both uses it, but filtering by `longitude` alone does not.
- **Build outward from a small seed set.** Start from a handful of `node_id`s or a single ASN, then join outward to discover their neighbors — don't start from an unfiltered table and join everything else onto it.
- **Attach cheap enrichment before you fan out.** The AS and geolocation tables have exactly one row per `node_id`; the link and interface tables have many rows per node. Join the one-row-per-node tables onto your seed set first, and save joins that multiply your row count for last — enriching a small set is cheap, enriching after a fan-out multiplies that cost across every duplicated row.
- **Prototype small, then scale up.** Add a `LIMIT` while you're still getting a query's shape right, and run `EXPLAIN` (or `EXPLAIN ANALYZE` on a bounded version of the query) before removing it, to confirm Postgres is using an index scan and not a sequential scan.
- **Select only the columns you need.** Avoid `SELECT *` — pulling unused columns across millions of rows still costs real time and memory even at this database's scale.

### Connecting

Ask your instructor for the read-only username and password. Copy `db_credentials.env.example` to
`db_credentials.env`, fill in `db_read_user` and `db_read_password` (host `db.nids.caida.org`),
and place `db_credentials.env` in the same directory as the notebook — it is git-ignored,
so never commit real credentials.

```python
import os
from pathlib import Path
from sqlalchemy import create_engine
from dotenv import dotenv_values

CREDS_FILE = Path(os.environ.get("ITDK_CREDS_FILE", "./db_credentials.env"))
_creds = dotenv_values(CREDS_FILE)
READ_DSN = (
    os.environ.get("ITDK_READ_DSN")
    or _creds.get("ITDK_READ_DSN")
    or "postgresql://nidsitdk:CHANGE_ME@127.0.0.1:5432/mydatabase"
)
engine = create_engine(READ_DSN)
```

### Tables

#### `caida_itdk.itdk_link_endpoints`

Maps each inferred link to its endpoint nodes. Most links have exactly two rows — one per endpoint — but a link can have more than two if it is a **hyperlink** (see [Introduction](Introduction.md)): a mesh connection observed among more than two routers. 14,222,743 rows, 1,537.84 MB. Indexed on `node_id`; the primary key is the composite `(link_id, endpoint_ordinal)`.

| Column             | Type    | Description                                                                                     |
| ------------------ | ------- | ----------------------------------------------------------------------------------------------- |
| `link_id`          | text    | Unique identifier for the link                                                                  |
| `endpoint_ordinal` | integer | Distinguishes this row from the link's other endpoint row                                       |
| `endpoint_token`   | text    | Raw per-endpoint identifier carried over from the underlying link data; use `node_id` for joins |
| `node_id`          | text    | One endpoint of the link (e.g., `N1967`)                                                        |

**Pattern for getting both endpoints of a link:**

```sql
SELECT e1.link_id, e1.node_id AS node_a, e2.node_id AS node_b
FROM caida_itdk.itdk_link_endpoints e1
JOIN caida_itdk.itdk_link_endpoints e2
    ON e1.link_id = e2.link_id AND e1.node_id != e2.node_id
```

#### `caida_itdk.itdk_node_as`

Maps each router node to the autonomous system that operates it. 3,737,018 rows, 391.16 MB. Indexed on `asn`; the primary key is the composite `(node_id, asn)`.

| Column    | Type   | Description                                  |
| --------- | ------ | -------------------------------------------- |
| `node_id` | text   | Router node identifier                       |
| `asn`     | bigint | Autonomous System Number (ASN)               |
| `method`  | text   | The technique used to assign this node's ASN |

#### `caida_itdk.itdk_node_geolocation`

Physical location of each node, assigned by one of two methods. 3,735,246 rows, 518.91 MB. Indexed on the composite `(country, longitude)`; the primary key is `node_id`.

| Column      | Type             | Description                                  |
| ----------- | ---------------- | -------------------------------------------- |
| `node_id`   | text             | Router node identifier                       |
| `continent` | text             | Continent name                               |
| `country`   | text             | ISO 2-letter country code                    |
| `region`    | text             | State or province (may be NULL)              |
| `city`      | text             | City name                                    |
| `latitude`  | double precision | Latitude in decimal degrees                  |
| `longitude` | double precision | Longitude in decimal degrees                 |
| `method`    | text             | Geolocation method: `'hoiho'` or `'maxmind'` |

#### `caida_itdk.itdk_router_hostnames`

DNS PTR (reverse lookup) records for interface IP addresses. 2,240,516 rows, 414.45 MB. Indexed on `hostname`; the primary key is the composite `(ip, hostname)`.

| Column     | Type | Description                                                 |
| ---------- | ---- | ----------------------------------------------------------- |
| `ip`       | inet | Interface IP address                                        |
| `hostname` | text | DNS PTR hostname for this IP (NULL if no PTR record exists) |

#### `caida_itdk.v_itdk_ifaces_transit`

A view over interface IP addresses, restricted to nodes directly observed transiting traffic (see [Only use transit node](#how-to-handle-them) below). 3,231,689 rows, 452.48 MB. Indexed on `node_id` and `link_id`.

| Column    | Type | Description                                                                                 |
| --------- | ---- | ------------------------------------------------------------------------------------------- |
| `ip`      | inet | One interface IP address of this node                                                       |
| `node_id` | text | Router node identifier                                                                      |
| `link_id` | text | The link this interface participates in                                                     |
| `flags`   | text | Interface/role flag data carried over from the source router data; treat as opaque metadata |

A node with multiple interfaces or link memberships can appear across several rows.

---

## Placeholder Nodes

![Network with placeholder notes](images/placeholders.jpg)

### What They Are

A **placeholder node** is created for each non-responding hop (TTL value) in a path between a pair of responding nodes. The most common reason for these missing hops is a router along a traceroute path that does not respond to probes — for example, because it filters ICMP TTL-exceeded messages. Even when a router stays silent, ITDK still infers its existence from the surrounding hops and creates a **placeholder node**. Placeholder nodes are assigned synthetic IP addresses in reserved ranges — for example, addresses in the `224.x.x.x` multicast block — rather than real routable IPs. Since the ITDK has no information to resolve aliases between placeholders, a new placeholder is created for each responding pair.

### What Problem They Solve

Placeholders preserve the topology graph's connectivity even when intermediate routers are silent.
Without them, a traceroute path `A → * → B` would need to create a false direct link between A-B or break the path with no connection at all.

### What Problems They Create

Placeholders have no known IP address, so we cannot geolocate them or assign ASNs directly. We also cannot do alias resolution on them, so multiple placeholder nodes could be a single real node. Since they appear to be normal nodes, it is easy to mistake them for real nodes and naively include them.

### How to Handle Them

![Network with placeholder Stategies](images/placeholders-strategies.png)

The right approach depends on your goal:

- **Drop completely**: this is the easiest solution, but will prevent you from calculating distances or doing connectivity calculations with the topology.
- **Replace with weighted links**: each chain of placeholders is replaced with a single link whose weight equals the number of hops `A-*-*-B` -> `A-(2)-B`. This preserves topology, but degree calculation will be wrong if neighbors connected by weighted links are naively included.
- **Count only the first placeholder**: count a single placeholder node per gap, since we know there must be at least one.
- **Only use transit nodes**: this counts only IP addresses that we observe transiting traffic, i.e., we have observed at least one IP address after the hop. In `A-B-C`, C is not a router. These transit nodes are exposed as the interfaces-restricted-to-transit-observed-nodes view. You have a smaller part of the topology, but each node is a known router.

[README](README.md) | [Introduction](Introduction.md) | [Background](Background.md) | Datasets ⮕ | [SQL](SQL.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-itdk.ipynb)
