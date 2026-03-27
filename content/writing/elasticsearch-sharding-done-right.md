---
title: "Elasticsearch Sharding, Done Right"
date: 2024-11-10
readTime: 7
draft: false
description: "Sharding and index mappings are the foundation of a healthy Elasticsearch cluster. Get them wrong, and no amount of hardware will save you."
---

Elasticsearch is deceptively easy to get started with. Throw some JSON at it, and it indexes. Query it, and you get results. It feels like magic — until your cluster has 500 million documents and everything is on fire.

I've spent a lot of time operating Elasticsearch (well, OpenSearch these days, but the concepts are the same). And the lesson I keep learning is that the problems you hit at scale are almost never about Elasticsearch itself. They're about decisions you made early — sharding strategy and index mappings — that seemed harmless at the time.

## What Sharding Actually Is

Let's start from the basics, because I've seen too many engineers use shards without really understanding what they are.

An Elasticsearch index is not a single blob of data sitting on a single machine. It's split into **shards**, and each shard is an independent Lucene index. When you create an index with 5 primary shards, you're essentially creating 5 smaller search engines that work together.

```
Index: "messages"
├── Shard 0  →  Node A
├── Shard 1  →  Node B
├── Shard 2  →  Node A
├── Shard 3  →  Node C
└── Shard 4  →  Node B

Each shard is a complete, independent Lucene index.
A search query hits ALL shards in parallel, results are merged.
```

Why does this matter? Because:

- **Writes are distributed.** Each document gets routed to one shard (by default, based on a hash of the document ID). More shards = writes spread across more nodes.
- **Reads are parallelized.** A search query runs on every shard simultaneously. The results from each shard are merged and returned.
- **Each shard has overhead.** Memory for segment metadata, file handles, thread pools. Shards aren't free.

That last point is the one most people learn too late.

## The Oversharding Problem

The most common mistake I see: too many shards. Someone creates an index with 20 primary shards "just in case" on a 3-node cluster. Each shard has a replica. That's 40 shards for a single index across 3 nodes — roughly 13 shards per node.

Now multiply that by 30 indices (one per day, for a month of logs). That's 1,200 shards on a 3-node cluster. Each shard consumes heap memory, file descriptors, and thread pool resources. The cluster spends more time managing shard metadata than actually searching.

Symptoms of oversharding:

- Cluster state updates become slow
- JVM heap pressure is high even with small indices
- Search latency increases despite low query volume
- Node startup takes forever because it's recovering hundreds of shards

### How Many Shards Do You Actually Need?

My rule of thumb:

- **Target 20-40 GB per shard.** This is the sweet spot Elastic themselves recommend. Small enough to recover quickly, large enough to be efficient.
- **Keep total shards per node under 600-800.** Beyond this, cluster state management becomes a bottleneck.
- **Start with 1 shard** for small indices. Seriously. If your index is under 10 GB, one primary shard is fine.

```
Example calculation:

Expected index size: 100 GB
Target shard size: 30 GB
Primary shards needed: ceil(100 / 30) = 4

With 1 replica: 4 × 2 = 8 total shards
On a 3-node cluster: ~2-3 shards per node ✓
```

You can always add more shards later via reindexing or the split API. You can't easily reduce them. Start small.

## Index Mappings: The Thing You Can't Fix Later

If sharding is the physical layout, mappings are the logical layout. They define how Elasticsearch interprets and stores each field in your documents.

And here's the thing people don't realize: **once a field is mapped, you can't change its type without reindexing.** If you accidentally let Elasticsearch auto-detect that a field is `text` when it should be `keyword`, you're stuck with it until you create a new index and migrate your data.

### Text vs Keyword

This is the single most important mapping decision:

- **`text`**: The field is analyzed — broken into tokens, lowercased, stemmed. Good for full-text search. "Quick brown fox" becomes `["quick", "brown", "fox"]`.
- **`keyword`**: The field is stored as-is. Good for exact matches, aggregations, sorting. "Quick brown fox" stays `"Quick brown fox"`.

```json
{
  "mappings": {
    "properties": {
      "message_body": {
        "type": "text",
        "analyzer": "standard"
      },
      "status": {
        "type": "keyword"
      },
      "user_email": {
        "type": "keyword"
      },
      "created_at": {
        "type": "date",
        "format": "strict_date_optional_time||epoch_millis"
      }
    }
  }
}
```

Common mistakes I've seen:

- **Email addresses mapped as `text`.**  Elasticsearch tokenizes `rohit@example.com` into `["rohit", "example.com"]`. Now searching for the exact email doesn't work as expected, and aggregations on email are meaningless.
- **Status fields mapped as `text`.** You want to filter by `status: "active"`, but because it's analyzed, you also match documents containing the word "active" anywhere in a text field.
- **Numeric IDs mapped as `text`.** Thread IDs, user IDs — these should be `keyword`. You're never going to full-text search an ID.

### Turn Off Dynamic Mapping

By default, Elasticsearch will auto-detect field types when it sees a new field. This sounds helpful but causes problems:

- A field that looks like a number gets mapped as `long`. Later you send a string value for the same field. Indexing fails.
- String fields default to both `text` and `keyword` (a multi-field mapping), doubling the storage.
- You end up with hundreds of fields you didn't plan for, all consuming resources.

```json
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "thread_id": { "type": "keyword" },
      "message":   { "type": "text" },
      "timestamp": { "type": "date" }
    }
  }
}
```

With `"dynamic": "strict"`, Elasticsearch rejects any document with a field not in your mapping. It feels restrictive, but it prevents mapping surprises in production.

## Managing Shards Over Time

For time-series data (logs, events, metrics), the standard approach is **time-based indices** — one index per day or per week. This gives you natural data lifecycle management: old indices can be closed, frozen, or deleted without affecting current data.

But time-based indices can lead to oversharding if you're not careful. 365 days × 5 shards × 2 (with replicas) = 3,650 shards per year for a single data type.

### Index Lifecycle Management (ILM)

ILM automates the lifecycle of your indices through phases:

```
Hot  →  Warm  →  Cold  →  Delete
```

- **Hot**: Actively writing. Fast storage. Full replicas.
- **Warm**: No longer writing. Can reduce replicas. Move to cheaper storage.
- **Cold**: Rarely queried. Frozen indices use minimal resources.
- **Delete**: Data retention period expired.

```json
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "30gb",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

The `shrink` action in the warm phase is particularly useful — it reduces the number of shards in an index. That 5-shard index that made sense during active writes can become a 1-shard index once it's read-only. This keeps your total shard count manageable.

### Rollover Instead of Date-Based Names

Instead of creating `logs-2024-11-10`, `logs-2024-11-11`, etc., use rollover aliases. Elasticsearch creates a new index when the current one hits a size or age threshold.

This means your shards stay at a consistent, healthy size regardless of traffic patterns. On a busy day, you might roll over twice. On a quiet day, not at all. The data drives the sharding, not the calendar.

## The Cheat Sheet

Here's what I wish someone had told me early on:

| Decision | Recommendation |
|----------|---------------|
| Shard size | Target 20-40 GB per shard |
| Starting shards | 1 for small indices, calculate for large ones |
| Replicas | At least 1 for production, 0 for dev |
| Dynamic mapping | Set to `strict` — define your fields explicitly |
| IDs and status fields | `keyword`, never `text` |
| Free-text content | `text` with an appropriate analyzer |
| Time-series data | Use ILM with rollover |
| Old indices | Shrink, then freeze or delete |

The boring truth about Elasticsearch is that the clusters I've seen run smoothly aren't the ones with the most nodes or the cleverest queries. They're the ones where someone thought carefully about shards and mappings before the first document was indexed.

Get the foundation right, and everything else follows. Get it wrong, and no amount of hardware will save you.
