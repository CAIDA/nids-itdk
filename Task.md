[README](README.md) | [Introduction](Introduction.md) | [Background](Background.md) | [Datasets](Datasets.md) | Tasks ⮕ | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-itdk.ipynb)

# Tasks

Complete the tasks below in order. All three tasks are completed inside [nids-itdk.ipynb](nids-itdk.ipynb) — replace the `# YOUR CODE HERE` sections with your code and answer the questions in the markdown cells that follow.

## Task 0: Set Up Your Local Environment and Run the Notebook

- step 1. Install dependencies and launch Jupyter — pick one:

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

- step 2. Copy `db_credentials.env.example` to `db_credentials.env` and fill in the read-only
  username/password from your instructor (see [Datasets](Datasets.md#connecting)).
- step 3. Complete each task by replacing the `# YOUR CODE HERE` sections and answer all questions.
- step 4. Save your completed notebook, commit, and push.

## Task 1: Border Routers Between Level3 (AS3356) and Netflix (AS2906)

Query the ITDK for all router-level links between Level3, a Tier-1 transit provider, and Netflix, a major content provider. Use geolocation to measure the distance between link endpoints, classify links as geographically adjacent (≤ 40 km) or not, and identify the distinct cities where the two networks peer.

- [ ] Q1.a: How many router-level links between Level3 and Netflix are geographically adjacent (within 40 km)? How many are not? Report the link IDs of the non-adjacent links.
- [ ] Q1.b: Excluding non-adjacent links, how many distinct locations did Level3 and Netflix appear to peer? Produce a table with the inferred locations and the number of links at each location.
- [ ] Q1.c: Write a paragraph summarizing your interpretation of the Level3–Netflix peering relationship. Focus on broad themes — economics, performance — rather than specific interconnection locations.

## Task 2: Routers Operated by China Unicom (AS4837)

Query the ITDK for all routers inferred to be operated by China Unicom, count routers per country, and compare that country distribution against Level3. Then, for the subset of China Unicom routers geolocated on the US West Coast, identify the peer autonomous systems that connect to them at the router level, using both AS assignment data.

- [ ] Q2.a: How does the concentration of China Unicom routers compare with those of Level3? What explains these differences?
- [ ] Q2.b: Which has more routers? Why might that be the case?
- [ ] Q2.c: Focusing on the West Coast China Unicom routers, produce a table showing, for the top 10 West Coast cities by number of connected routers, the total number of distinct China Unicom routers with at least one peer AS connection and a breakdown by peer ASN.
- [ ] Q2.d: Write a paragraph summarizing your interpretation. Why would China Unicom connect with the same ASes at the same location more than once? Why would China Unicom connect with the same ASes at different locations?

## Task 3: Multi-AS Interconnection Analysis

Query the ITDK for all router-level links between any pair of 18 major autonomous systems spanning Tier-1 transit, large ISPs, content providers, and CDNs. Visualize the results as a heatmap and interpret the structural patterns you observe.

- [ ] Q3: Write two paragraphs explaining what you observe about the router-level interconnections between these 18 autonomous systems. Focus on structural patterns — which categories of AS appear most interconnected, and which appear least? What does this tell you about the economics and architecture of the Internet?

[README](README.md) | [Introduction](Introduction.md) | [Background](Background.md) | [Datasets](Datasets.md) | Tasks ⮕ | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-itdk.ipynb)
