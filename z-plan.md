<goal>
The ../nids-itdk-key/nids-itdk-key.ipynb has changed. We need to do a review of 
all the markdown files and the notebook to make sure they agree with the key, the source of truth. 
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

<instructions> 
Review the source of truth ../nids-itdk-key/nids-itdk-key.ipynb
Review the CVS files caida_itdk_csv/* for information on the table sizes.
Review current module *.md and nids-itdk.ipynb
Identify what needs to be fixed to bring the student version in align with the key.
Don't assume, ask questions. 
</instructions>
