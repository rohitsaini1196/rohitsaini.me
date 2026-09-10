---
title: "What AWS Cost Optimization Hub Can't See"
date: 2026-09-11
slug: what-cost-optimization-hub-cant-see
readTime: 5
draft: false
tags: ["systems"]
description: "AWS runs several recommendation engines that don't reconcile with each other, and the money hides in the seams."
---

AWS Cost Optimization Hub is good at one question: *is this resource bigger than its metrics justify?* Between it and Compute Optimizer, rightsizing is covered well enough that on most accounts they'll find more than you will by hand.

Then you fix everything they told you to fix, and the bill is still surprising.

I pointed a read-only auditor at three production accounts to find out why. The answer is structural: AWS runs several recommendation engines that don't reconcile with each other, and the money hides in the seams.

## The bill is per usage type, not per resource

Cost Explorer will tell you a region spent `$10,906` on `Aurora:StorageIOUsage` last month. It will not tell you which cluster caused it. No console view, no tag dimension, no drill-down. One line, one region.

So a cluster can sit on the wrong Aurora storage class indefinitely. I/O-Optimized costs ~2.25× Standard on storage and ~30% more on instance hours, and buys you zero per-request I/O charges. Whether that trade pays depends on I/O volume — the exact number the bill won't give you per cluster.

You reconstruct it from CloudWatch:

```
cluster I/O cost ≈ (VolumeReadIOPs + VolumeWriteIOPs) / 1e6 × $0.20
```

One cluster came out at `$1,184/month` of I/O — against a `$2,722/month` storage premium it was paying to avoid them. Wrong side of the break-even, and nothing in AWS was going to say so.

## Cost Explorer knows things it never recommends

Some findings AWS has already computed and will happily render, but never surfaces as advice.

`GetSavingsPlansUtilization` returns `UnusedCommitment` directly. On one account: `$3,371` for the month, committed `$42,000`, consumed `$38,629`. Unused commitment isn't refunded and doesn't roll over — the purest waste on a bill.

The Hub never mentions it. The Hub recommends *purchases*. Under-consuming an existing commitment isn't a purchase, so it's outside the model.

Same shape, worse visibility: cross-AZ transfer, billed `$0.01/GB` each direction, split across the line items of every service that generates it. On one account, `$6,888/month` across 688 TB that nobody had seen as a single figure — because no view produces one. (Much of it is Multi-AZ replication you shouldn't touch. You should still know the number.)

## List price is not what you pay

This one cost me a wrong answer before it taught me anything.

I priced an Aurora change at `$5,601/month` using the published `$9.28/hour` for a `db.r5.16xlarge`. Then I divided the account's actual Cost Explorer cost by its usage quantity for that usage type: `$0.1289/hour`. Reserved-Instance covered. My estimate was **4× too high**.

The fix is a rule:

> Storage, I/O and data transfer are never covered by a Reserved Instance or Savings Plan. Instance hours often are.

Those are not the same kind of number and shouldn't be added into one headline. The commitment-independent half is recoverable this month; the instance-hour half may be worth nothing until a commitment is re-planned. Corrected figure: `$1,414/month`.

If you're deriving costs from the Price List API, you're computing what an unoptimised stranger would pay.

## MSK charges for the disk you asked for, not the disk you used

The tiered-storage pitch is clean: short local retention on the brokers, everything older to a tier costing a fifth as much. I built a detector and reported a saving.

Wrong. MSK bills **provisioned** broker storage. Verify against your own bill — billed `Kafka.Storage` GB-months against provisioned capacity and `KafkaDataLogsDiskUsed`:

```
provisioned      31,650 GB
data held        15,183 GB
billed GB-mo     30,901      → 0.976× provisioned, 2.03× held
```

Provisioned wins. So enabling tiered storage on an existing cluster saves nothing — the EBS volume keeps billing in full and the remote tier is added *on top*. Strictly more expensive.

The real saving needs less provisioned storage, and **MSK broker volumes expand but never shrink**. Cluster rebuild, not a config change. I retracted the claim.

## AWS recommendations are not a shopping list

Export the Hub's purchase recommendations, sum the column, and you'll get a badly wrong number.

AWS returns one logical purchase as many rows: six term/payment variants (1yr/3yr × All/Partial/No Upfront), plus alternative coverage levels (5, 7 or 9 instances of the same type). Mutually exclusive choices. You buy one.

Summing them across three accounts: `$130,000/month` of proposed commitment against a real `$28,020`. A **4.6× overstatement** from a naive `sum()`.

Group on `(account, region, commitment_type, instance_family)` and treat everything inside as alternatives. Savings Plans need extra care — each payment option quotes a different hourly rate for the same plan, so deduplicating on the summary string still triple-counts.

## Two AWS products, one console, opposite advice

The most interesting failure isn't a gap. It's a contradiction.

The Hub sizes purchase recommendations from *observed usage*. It never asks whether that usage is justified. So it will recommend a three-year commitment against a fleet Compute Optimizer is simultaneously telling you to shrink.

Worse where no AWS engine covers the service. Compute Optimizer doesn't model OpenSearch — so nothing stopped the Hub recommending a three-year reservation on a domain running three `i3.large` nodes (storage-optimised, ~440 GB NVMe each) holding **2.7 GB** at `$1,058/month`. Buying it would have fixed a 400×-oversized fleet in place for three years.

Detecting that needs no data AWS doesn't already have. You just have to read the two recommendation sets against each other, which nothing in AWS does.

On a related note I still can't explain: one account's Hub claimed `$2,649/month` of ECS rightsizing against `$926/month` of total ECS spend. Savings cannot exceed spend.

## Where this leaves you

Honest caveat, because the above reads more triumphant than the results were.

On the two accounts that had never been rightsized, AWS-native tooling found **25–39× more** than my auditor did. It's very good at generic over-provisioning, and that's where most of the money is in an un-tuned estate.

Everything above came almost entirely from the one account that had already been optimised — where AWS's own tools had nothing left to say.

| | AWS-native | The seams |
| --- | --- | --- |
| Un-tuned account | finds nearly everything | marginal |
| Optimised account | finds nothing new | where the money is |

So the order matters. Enable the Hub and Compute Optimizer, act on them, *then* go looking in the gaps.

And none of these are savings. They're estimates awaiting someone who owns the system agreeing with them — which never happened on my three accounts. An estimate is a hypothesis.

The tool is [on GitHub](https://github.com/rohitsaini1196/costlint) if you want to point it at your own bill. Read-only, explains every number it produces, and declines more often than it reports.
