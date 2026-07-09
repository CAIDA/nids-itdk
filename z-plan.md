<goal>
Replace the SQL in code with a student guide modeled on model.
</goal>

<model>
# Country distribution of all geolocated AS4837 nodes, compared against Level3 (AS3356)
# Push the histogram into SQL (one GROUP BY, both ASNs in a single round trip),
# then shape the comparison table in pandas. The full, commented SQL is in
# Task-2.md ("Country-Wide Comparison"); SQL.md explains GROUP BY / COUNT(DISTINCT).
country_counts = pd.read_sql("""
    --- student_code_start -------------------------------
    --- YOUR CODE HERE
    ---
    --- Write a query to calculate the number of routers per country for 
    --- China Unicom (AS 4837) and Level3 (AS 3356).
    --- 
    --- Steps to build your query:
    --- 1. Select the ASN, the country, and count the distinct node_ids.
    --- 2. Query from the `caida_itdk.itdk_node_as` table and JOIN the 
    ---    `caida_itdk.itdk_node_geolocation` table on their `node_id`.
    --- 3. Filter (WHERE) for only the two ASNs in question using an IN clause 
    ---    so you get both in one database round trip.
    --- 4. GROUP BY the ASN and country to get one row per (AS, country) pair.
    ---
    --- See Task-2.md and SQL.md for syntax help and examples.
    --- student_code_end ---------------------------------
""", engine)
</model>

<code>
# Count router-level links between each pair of the 18 ASes.
# The Python glue is provided; fill in the SQL body. The full, commented query
# is in Task-3.md, and SQL.md explains the DISTINCT-then-self-join pattern and
# the IN (...) interpolation.
asn_sql = ", ".join(str(a) for a in asn_list)   # "174, 701, 1299, ..." for IN (...)
pair_df = pd.read_sql(f"""
    --- student_code_start -----------------------------
    --- YOUR CODE HERE
    --- Build pair_df with columns asn1, asn2, link_count (one row per connected
    --- AS pair among the 18 in asn_list):
    ---   1. CTE le_as: SELECT DISTINCT link_id, asn from itdk_link_endpoints
    ---      JOIN itdk_node_as, filtered with asn IN (asn_sql). Collapse the
    ---      grain with DISTINCT before self-joining, or the query fans out.
    ---   2. Self-join le_as on link_id with asn1 < asn2 (each unordered pair
    ---      once), GROUP BY the pair, COUNT(*) AS link_count, ORDER BY it DESC.
    --- Interpolate asn_sql into the IN (...) clause via this f-string. See
    --- Task-3.md / SQL.md.
    --- student_code_end ----------------------------------------
""", engine)

pair_df["name1"] = pair_df["asn1"].map(ASES)
pair_df["name2"] = pair_df["asn2"].map(ASES)

print(f"AS pairs with at least one router-level link: {len(pair_df)}")
print()
print("Top 20 pairs by link count:")
print(pair_df[["name1", "name2", "link_count"]].head(20).to_string(index=False))
</code> 