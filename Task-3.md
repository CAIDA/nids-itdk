[README](README.md) | [Introduction](Introduction.md) | [Background](Background.md) | [Datasets](Datasets.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | Task 3 ⮕ | [Notebook](nids-itdk.ipynb)

# Task 3: Multi-AS Interconnection Analysis — How-To Guide

## The 18 Autonomous Systems

| ASN   | Name             | Code | Category       |
| ----- | ---------------- | ---- | -------------- |
| 174   | Cogent           | T    | Tier-1 transit |
| 701   | Verizon          | T    | Tier-1 transit |
| 1299  | Arelion          | T    | Tier-1 transit |
| 3257  | GTT              | T    | Tier-1 transit |
| 3491  | PCCW             | T    | Tier-1 transit |
| 5511  | Orange           | T    | Tier-1 transit |
| 6453  | TATA             | T    | Tier-1 transit |
| 3320  | Deutsche Telekom | L    | Large ISP      |
| 6461  | Zayo             | L    | Large ISP      |
| 6762  | Telecom Italia   | L    | Large ISP      |
| 6830  | Liberty Global   | L    | Large ISP      |
| 12956 | Telefonica       | L    | Large ISP      |
| 15133 | Edgecast         | D    | CDN            |
| 20940 | Akamai           | D    | CDN            |
| 714   | Apple            | C    | Content        |
| 2906  | Netflix          | C    | Content        |
| 13335 | Cloudflare       | C    | Content        |
| 15169 | Google           | C    | Content        |

The static table above lists the 18 ASes grouped by category (transit backbone first, content last), then by ASN, with the single-letter `Code` used to annotate the heatmap. The heatmap itself does **not** follow this order — see the next section.

## Reordering the Heatmap with the Quadratic Assignment Problem (QAP)

If the heatmap axes stayed fixed in category/ASN order, clustering would only be visible if the real interconnection structure happened to line up with that category grouping. Instead, the notebook solves a **Quadratic Assignment Problem (QAP)** to find a placement of the 18 ASes along the axis that puts heavily-linked pairs next to each other, so dense clusters show up as visible blocks near the diagonal regardless of category.

Concretely, this is a linear-arrangement (seriation) problem: assign each AS a position `0..17` on the axis, minimizing

```
sum_ij  W[i,j] * |pos_i - pos_j|
```

where `W[i,j]` is the router-level link count between AS `i` and AS `j`. In words: pairs with more links should end up spatially closer together on both axes. `scipy.optimize.quadratic_assignment(W, D)` solves exactly this — it minimizes `trace(Wᵀ P D Pᵀ)` over permutation matrices `P`, with `D[p, q] = |p - q|` encoding "distance along the axis."

The default single solver run (barycenter start) can land in a mediocre local optimum, so the notebook restarts 30 times from randomized starting permutations and keeps the lowest-objective (best-clustering) result, using a fixed RNG seed for reproducibility. Print `best.fun` and the resulting AS order so you can see how the axis was reordered.

Because the QAP reordering breaks the category grouping (a Tier-1 transit AS can end up next to a CDN if the two are heavily linked), the heatmap adds a one-letter `Code` strip (T/L/D/C, from the table above) along the top axis so each column's category is still identifiable after reordering.

## Building the Symmetric Matrix

The query counts distinct `link_id` values for each AS pair (with `asn1 < asn2` to avoid double-counting). After running the query, build a symmetric matrix — indexed by the QAP order, not the table order — and mask zeros so the heatmap shows only pairs with at least one observed link:

```python
matrix = pd.DataFrame(0, index=qap_names, columns=qap_names)
for _, row in pair_df.iterrows():
    n1, n2 = ASES[row.asn1], ASES[row.asn2]
    matrix.loc[n1, n2] = row.link_count
    matrix.loc[n2, n1] = row.link_count

data = matrix.values.astype(float)
data[data == 0] = np.nan   # mask zero pairs — no link observed
```

Apply `np.log1p` before calling `imshow` so that pairs with 1 link and pairs with thousands of links are both visible on the colormap (`YlOrRd`). The category-code strip is added afterward via a `secondary_xaxis("top")` labeled with `qap_codes`.

To sanity-check the matrix before plotting, sort `pair_df` by `link_count` descending and print the top pairs — this shows which AS pairs dominate and confirms the query returned sensible counts.

## What Your Write-Up Should Address

![heatmap](images/heatmap.png)

Question 3 asks for **two paragraphs** interpreting the heatmap. Use the QAP ordering and the category-code strip to reason about structure rather than describing individual cells. Things to consider:

- Which categories (T / L / D / C) cluster together into dense blocks, and which categories stay sparse or isolated?
- Do the most heavily-interconnected ASes line up with the roles described in [Background](Background.md) (Tier-1 transit, large ISP, CDN, content)? Where does a multi-service AS complicate the clean category label?
- What does the pattern you see imply about the economics and architecture of the Internet — who needs to interconnect densely, and who does not?

Ground each claim in what the heatmap actually shows for your run, not in these prompts.

[README](README.md) | [Introduction](Introduction.md) | [Background](Background.md) | [Datasets](Datasets.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | Task 3 ⮕ | [Notebook](nids-itdk.ipynb)
