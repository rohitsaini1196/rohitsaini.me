---
title: "Blindspot, Part 3: $0.0215 a Report, and the Things That Broke"
date: 2026-08-03T12:00:00+05:30
readTime: 4
draft: false
tags: ["build", "backend", "debugging"]
slug: "blindspot-running-it"
description: "Budgets checked between stages, idempotent Instagram DM delivery, and two production failures: a gitignore comment that ignored nothing, and exit code 139 from an architecture mismatch."
---

*Part 3 of three on [Blindspot](https://blindspot.buzz). [Part 1: reading the video](/writing/blindspot-reading-the-video/) · [Part 2: extraction and evidence](/writing/blindspot-extraction-and-evidence/).*

The analysis pipeline is the interesting half. The half that decides whether it survives contact with real traffic is cheaper to describe and more expensive to get wrong.

### Check the budget between stages, not after

Every model call records usage and estimated cost, and a per-report budget is checked *between* pipeline stages. If research exhausts it, the system stops gathering evidence and records that coverage may be incomplete. It does not keep going until the bill happens to stop.

Measured on real Reels:

| | cost |
|---|---:|
| Lean profile, per checkable Reel | ~$0.015 |
| Strong profile | $0.20–0.25 |
| Built-in web search, per call | ~$0.01 |

Evidence retrieval dominates, because retrieved documents arrive as context tokens.

The local database currently holds 32 completed reports across 13 unique posts, plus one failed job. Those runs total **$0.6884**, an average of **$0.0215** per report.

Everything else exists to keep that number down: exact-report caching, media caching so reposts aren't retranscribed, triage stopping non-factual input early, a rolling daily budget that can pause the worker, per-user quotas, endpoint rate limits. Promptry records model, prompt version, token counts, latency, and cost for every invocation.

### A crashed job should not crash you forever

The worker polls a SQLite-backed job queue. On startup it marks orphaned running jobs as failed rather than replaying them.

A job that reliably crashes the process must not be allowed to keep crashing it. Replay-on-boot turns one poison message into an outage.

### Delivery has to be idempotent, because the platform isn't

Website and Instagram DM share one analysis pipeline but differ at admission and delivery. On a Meta message webhook, Blindspot validates and deduplicates the event, resolves the shared Reel or carousel, reserves the sender's quota, acknowledges, enqueues, then sends a compact result and a public link when it finishes.

Instagram delivers duplicate webhooks. Users share a second post while the first is still running. So message state is persisted and every step is idempotent: acknowledgement, processing, retry, final send. Failed delivery retries up to five times before being marked terminal.

Report links carry opaque attribution tokens, which connect later feedback to a delivery without exposing the Instagram sender on a public page.

### Boring infrastructure, two sharp edges

It all runs on a shared Oracle ARM host behind Cloudflare and nginx: Next.js app, worker, FFmpeg, ingestion tools, and SQLite on one machine, because SQLite needs local shared storage.

The worker doesn't hot-reload. Editing pipeline code while the web app is running leaves the interface on new code and the worker on old code, which produces results that make no sense until you remember why. `serve.sh` supervises both, but pipeline changes still need a worker restart.

Two failures worth writing down.

**Exit code 139, no error message.** A Rosetta x86 Node process was loading an arm64 `better-sqlite3` binary. Scripts now run through `pnpm exec node` so the Node architecture matches the installed native module.

**A `.gitignore` that ignored nothing.** The file contained:

```
/data # database
```

Git treats the comment as part of the pattern, so it was looking for a directory literally named `data # database`. The real database and secrets directories were never ignored. Comments go on their own lines now.

### What I don't have yet

The evaluation harness scores stored reports without new API calls. Fixtures carry expected material claims, acceptable verdict sets, minimum evidence requirements, and known missing context.

The target is 50 reviewed Reels, 10 each of directly false claims, real media in the wrong context, defensible-but-selective framing, health/scam/finance, and ambiguous developing stories.

The fixture set currently contains four.

Labels can't come from Blindspot's own reports; a reviewer has to establish them from primary sources, and anything political, communal, health, finance, or accusation-related needs a second reviewer. The comparison that matters is strong extraction plus lean research against all-lean, scored on material-claim recall, citation precision, source quality, verdict acceptability, confidence calibration, latency, cost, and unsafe overclaim rate. A confident unsafe error counts for more than an ordinary omission.

Blindspot is live, the pipeline runs end to end, the reports are shareable. What I don't have is a representative error rate.

Four reviewed Reels can find bugs. They can't establish trust.
