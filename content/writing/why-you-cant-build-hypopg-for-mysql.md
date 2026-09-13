---
title: "Why You Can't Build HypoPG for MySQL"
date: 2026-09-13
slug: why-you-cant-build-hypopg-for-mysql
readTime: 5
draft: false
tags: ["systems", "backend"]
description: "Postgres has HypoPG and pganalyze. Why trying to fake indexes in MySQL fails on index dives, and what InnoDB's optimizer insists on reading."
---

Postgres has a category of tooling that MySQL doesn't: index advisors that tell you which index to add before you add it.

Most of them rest on [HypoPG](https://github.com/HypoPG/hypopg), an extension that lets you create an index that doesn't exist, run `EXPLAIN`, and see whether the planner would have used it. No disk, no lock, no waiting. Dexter and Supabase's `index_advisor` both work this way.

pganalyze went further and skipped the extension entirely. Their Indexing Engine runs its own copy of the Postgres planner inside their app, feeding it your schema and statistics through config variables, with nothing installed on your database. Give the planner numbers about your data and it will cost a query it has never seen touch a row of.

That detail is the whole story, and I didn't understand it until I'd spent two weekends failing.

(MySQL isn't advisor-free, to be clear. Oracle's HeatWave has an Autopilot Index Advisor that reads workloads from Performance Schema and recommends indexes to add and drop. It's a closed, managed service, and it runs inside the server. Hold that thought.)

## The idea

InnoDB stores its statistics in two ordinary tables, `mysql.innodb_table_stats` and `mysql.innodb_index_stats`. Ordinary as in you can `UPDATE` them.

MySQL's optimizer is cost-based. So, in theory, the pganalyze trick should port:

1. `CREATE TABLE shadow LIKE orders` — empty.
2. Create the candidate index on the empty shadow. Instant.
3. Compute the statistics that index *would* have on the real 10M-row table, and inject them.
4. `EXPLAIN` against the shadow.

Computing the stats is mechanical. `n_diff_pfxNN` is cardinality per key prefix, straight from `COUNT(DISTINCT ...)`. Page counts you estimate from row count and key width.

I built 24 query-and-candidate-index pairs across four categories, against a 10M-row table with deliberately skewed data. Ground truth was creating each index for real and measuring with `EXPLAIN ANALYZE`. I only cared about ranking, not absolute cost. An advisor doesn't need the right number, it needs the right order.

## The mechanism worked. The result didn't.

All 39 stat injections survived `FLUSH TABLE` and were never silently recalculated. Fill factor, which I expected to be the fragile part, turned out irrelevant: varying it across the plausible range moved one cost estimate by 0.05.

So the part I thought was hard was free. Here's the part that wasn't.

| category | ranking agreement |
|---|---|
| equality | 6/6 |
| equality + range | 3/6 |
| `ORDER BY` / `LIMIT` | 3/6 |
| covering | 6/6 |
| **overall** | **18/24 (75%)** |

Spearman correlation between shadow cost and measured execution time: **0.00**.

The range rows fail because of index dives. For a predicate like `created_at >= ?`, MySQL doesn't consult statistics. It walks into the actual B-tree and counts rows. An empty table has nothing to walk, so the dive returns 1 row and every range candidate gets a cost of 2. The shadow can't tell `(tenant_id, created_at)` from `(created_at, tenant_id)`, which is the exact distinction an advisor exists to make.

Dives aren't only a range problem. `eq_range_index_dive_limit` decides when equality predicates use dives instead of statistics, and its default is 200. At the default, even a single `col = const` dives into the empty table and overall agreement falls to **29%**. My equality results only held because I set it to 1.

The `ORDER BY`/`LIMIT` rows fail for two different reasons, and neither is dives. First, MySQL's own cost model ignores the `LIMIT`: one plan read 20 rows and was costed at 832,000. The real optimizer, with the real index in place, only reached 67% agreement with measured time on those rows, so the shadow was imitating something already unreliable.

Second, skew. The shadow scored a 9.7-second plan identically to a 1-millisecond one, because it believed my hot tenant had 10,600 rows. It had 4 million. Statistics carry distinct counts, from which the optimizer infers an average group size, and averages are blind to skew.

Equality and covering went 12/12 together, and it would be easy to call that a partial win. It isn't. That's four queries, two of the twelve pairs are ties the shadow gets right for free, and none of them test skew adversarially.

## The consolation prize that wasn't

That skew number looked like a product. Multi-tenant apps are skewed by construction, one big customer and a long tail. If MySQL systematically misjudges the big ones, there's something to sell.

So I tested it on real data instead of a shadow. Same table, one tenant holding 40% of rows, comparing the optimizer's estimate against the true count.

| tenant | true rows | no index | no index + histogram | with index |
|---|---|---|---|---|
| hot (40%) | 3,999,189 | 0.32× | 1.28× | 1.47× |
| large (1%) | 100,561 | 12.7× | 1.28× | 2.05× |
| median | 5,002 | 255× | 2.55× | 1.00× |
| tail | 4,769 | 267× | 2.67× | 1.00× |

With an index on the tenant column, which is how any multi-tenant app actually runs, MySQL stays within about 2× across the whole distribution. My premise was wrong and I stopped there.

Two details from the optimizer trace. The estimates come from dives, which sample real pages, so they're exact for small tenants and close for large ones. And with an index present the histogram is ignored entirely, matching the documented behaviour of histograms being consulted mainly when no suitable index exists.

Without an index it's a flat 10% guess, off by 260× at the small end, which a histogram brings to roughly 2.6×. But the plan is a full table scan either way, so the error can't change the decision.

One important limit: I tested `col = constant`. In a join, the lookup value comes from the other table, so there's nothing to dive with and MySQL falls back to the average from statistics. That's the same skew blindness, over a whole category of queries, and I haven't measured it. Open question.

## The same fact, twice

Dives are why you can't fake an index in MySQL. Dives are also why, for constant equality, you don't need to worry about skew.

Postgres's planner reasons from statistics, which is why pganalyze can run it in their own app on nothing but numbers. MySQL's optimizer insists on reading the actual data. That makes it more accurate than a statistics-based story suggests, and it makes it impossible to reason about from outside. My second hypothesis was only plausible because my first one had failed.

Two things people reasonably raise. MySQL 8.0's invisible indexes aren't hypothetical: they're real indexes with the full build cost, just hidden from the optimizer. And an `IN` list at or above `eq_range_index_dive_limit` does switch from dives to statistics, which is a tuning footnote rather than a way in.

So an index advisor for MySQL probably can't be bolted on from outside. It has to live inside the server, with access to the pages. Which, judging by where Oracle put theirs, is roughly the conclusion they reached too.

---

*Harness and raw `EXPLAIN` output: [github.com/rohitsaini1196/dowser](https://github.com/rohitsaini1196/dowser). Runs in about six minutes against MySQL 8.0.41 in Docker. The table is 10M rows, 1.66 GB, with a deliberately skewed tenant distribution. If someone finds a way past the dives, I'd like to hear it.*
