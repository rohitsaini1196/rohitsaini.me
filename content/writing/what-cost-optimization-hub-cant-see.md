---
title: "What AWS Cost Optimization Hub Can't See"
date: 2026-09-11
slug: what-cost-optimization-hub-cant-see
readTime: 5
draft: false
tags: ["systems"]
description: "AWS's cost engines are good. The gaps appear at their boundaries — and that's where the money hides."
---

AWS's native cost tooling is very good at the obvious questions: is this resource oversized, idle, or worth committing to? Cost Optimization Hub pulls those recommendations into one place and does a good job of finding the first layer of waste.

Then you act on everything it told you, and the bill is still surprising.

I pointed a read-only auditor at three AWS environments to find out why. The engines aren't the problem. The gaps appear at their boundaries: where billing isn't attributable to resources, where a recommendation exists in only one direction, where a service has no resource-level optimizer, or where a useful number sits in an API but never becomes a recommendation.

## The bill is per usage type, not per resource

Cost Explorer will tell you a region spent `$10,906` on `Aurora:StorageIOUsage` last month. It will not tell you which cluster caused it. One line, one region.

I/O-Optimized costs ~2.25× Standard on storage and ~30% more on instance hours, and buys you zero per-request I/O charges. Whether that trade pays depends on I/O volume — exactly the number the bill won't give you per cluster. You reconstruct it from CloudWatch:

```
cluster I/O cost ≈ (VolumeReadIOPs + VolumeWriteIOPs) / 1e6 × $0.20
```

Compute Optimizer now evaluates Aurora storage economics, but its documented recommendation path is Standard → I/O-Optimized. The cluster I found was already on I/O-Optimized and belonged back on Standard: `$1,184/month` of I/O it would have paid, against a `$2,722/month` storage premium it was paying to avoid it. The reverse direction wasn't surfaced.

## Some numbers exist but never become advice

`GetSavingsPlansUtilization` returns `UnusedCommitment` directly. On one account: `$3,371` for the month — committed `$42,000`, consumed `$38,629`. Unused commitment isn't refunded and doesn't roll over.

I didn't find an equivalent Cost Optimization Hub recommendation for under-consuming an existing Savings Plan. The number is already sitting in Cost Explorer's utilization API. You just have to go and ask for it.

Cross-AZ transfer is the same shape with worse visibility: billed `$0.01/GB` each direction, split across the line items of every service that generates it. On one account, `$6,888/month` across 688 TB that nobody had seen as a single figure. (Much of it is Multi-AZ replication you shouldn't touch. You should still know the number.)

## List price is not what you pay

This one cost me a wrong answer before it taught me anything.

I priced an Aurora change at `$5,601/month` using the published `$9.28/hour` for a `db.r5.16xlarge`. Then I divided the account's actual Cost Explorer cost by its usage quantity for that usage type: `$0.1289/hour`. Reserved-Instance covered. My estimate was **4× too high**.

> Storage, I/O and data transfer are never covered by a Reserved Instance or Savings Plan. Instance hours often are.

Those are different kinds of number and shouldn't share a headline. The commitment-independent half is recoverable this month; the instance-hour half may be worth nothing until a commitment is re-planned. Corrected figure: `$1,414/month`.

If you derive costs from the Price List API, you're computing what an unoptimised stranger would pay.

## MSK charges for the disk you asked for

The tiered-storage pitch is clean: short local retention on the brokers, everything older to a tier costing a fifth as much. I built a detector and reported a saving.

Wrong. MSK bills **provisioned** broker storage. Check your own bill:

```
provisioned      31,650 GB
data held        15,183 GB
billed GB-mo     30,901      → 0.976× provisioned, 2.03× held
```

So enabling tiered storage on an existing cluster saves nothing — the EBS volume keeps billing in full and the remote tier is added on top. The real saving needs less provisioned storage, and **MSK broker volumes expand but never shrink**. Cluster rebuild, not a config change. I retracted the claim.

## Purchase recommendations are not a shopping list

AWS returns one logical purchase as many rows: six term/payment variants (1yr/3yr × All/Partial/No Upfront), plus alternative coverage levels (5, 7 or 9 instances of the same type). Mutually exclusive choices. You buy one.

Summing them across three accounts gave `$130,000/month` of proposed commitment against a real `$28,020`. A **4.6× overstatement** from a naive `sum()`.

Group on `(account, region, commitment_type, instance_family)` and treat everything inside as alternatives. Savings Plans need extra care — each payment option quotes a different hourly rate for the same plan.

## Reconciliation needs a signal to reconcile against

Commitment recommendations start from historical usage. Cost Optimization Hub does reconcile overlapping recommendations — it groups related ones and reduces Savings Plan savings when, say, EC2 capacity is recommended for removal. But that only works when AWS has another optimization signal for the resource.

OpenSearch exposes the boundary nicely. The Hub can recommend OpenSearch Reserved Instances; Compute Optimizer doesn't provide OpenSearch rightsizing. So there may be no native signal to reconcile against.

That's how I found a three-year reservation recommended for a domain running three `i3.large` nodes — storage-optimised, ~440 GB NVMe each — holding **2.7 GB** at `$1,058/month`. Buying it would have fixed a 400×-oversized fleet in place for three years. Detecting it needs no data AWS doesn't already have; it needs someone to look at the domain before signing.

One number still didn't reconcile: the Hub reported `$2,649/month` of ECS rightsizing savings while Cost Explorer showed `$926/month` of ECS spend for the period I compared. I haven't reconciled the difference in scope or cost basis, so the auditor marks those recommendations as unreliable rather than treating the `$2,649` as real.

## Where this leaves you

Honest caveat, because the above reads more triumphant than the results were.

On the two accounts that had never been rightsized, AWS-native tooling found **25–39× more** than my auditor did. It's very good at generic over-provisioning, and that's where most of the money is in an un-tuned estate.

Everything above came almost entirely from the one account that had already been optimised.

| | AWS-native | The boundaries |
| --- | --- | --- |
| Un-tuned account | finds nearly everything | marginal |
| Optimised account | finds little new | where the money is |

So the order matters. Act on the Hub and Compute Optimizer first, *then* go looking at the edges.

And none of these are savings. They're estimates awaiting someone who owns the system agreeing with them — which never happened on my three accounts. An estimate is a hypothesis.

The tool is [on GitHub](https://github.com/rohitsaini1196/costlint) if you want to point it at your own bill. Read-only, explains every number it produces, and declines more often than it reports.
