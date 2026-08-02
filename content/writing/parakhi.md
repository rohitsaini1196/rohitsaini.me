---
title: "Parakhi: Where Does Your ₹5 Actually Go?"
date: 2026-08-01
readTime: 4
draft: false
tags: ["build", "backend", "systems"]
slug: "parakhi"
description: "Parakhi breaks an Indian product's MRP into what stays in India, what goes to tax, and what leaves the country. Every number is computed, sourced, and labelled with how much you should trust it."
---

Type "Parle-G" into [parakhi.in](https://parakhi.in) and you get a ₹5 biscuit packet taken apart: wheat, sugar, palm oil, the packaging film, GST at 18%, the retailer's cut, the slice that leaves the country. Each row carries a source and a confidence dot.

It works on 1002 products, 44 categories, and fuel across 12 states. Here is what it took to make those numbers defensible.

### The rule that shaped everything

**The model never touches a number.**

That decision came first and everything else follows from it. An LLM can read a messy search box, guess which category a product belongs to, and normalise a brand name. It cannot be allowed to say "12% of this is imported", because it will say it confidently, differently each time, with nothing behind it.

So the arithmetic lives in `computeBaseline()`, a pure function with no network calls in it. Same product in, same breakdown out, every time, forever.

```
  free text ─┐
  barcode  ──┼─►  resolve       BrandIndex fuzzy match
  url      ─┘                   (LLM only as fallback)
                  │
                  ▼
             categorize         keyword + HSN rules ──► one of 44
                  │             hand-written category templates
                  ▼
        ┌──────────────────────┐
        │  computeBaseline()   │   pure function, no LLM
        └──────────┬───────────┘
                   │
                   │  category template composition
                   │  + CBIC HSN→GST table       (56 rows)
                   │  + Wikidata brand→parent→country
                   │  + Agmarknet daily mandi prices
                   │  + Open Food Facts label     (confirms only)
                   ▼
        Postgres cache ──► ISR page
```

The 56-row GST table is typed by hand from CBIC notifications. The 44 category templates are each a couple of hours of reading trade-body reports. That work doesn't scale and that is the point: it's the part a model can't do for you.

Every number renders with a source tier and a confidence dot, so a figure backed by a government notification looks different on the page from one backed by an industry estimate. You can see which is which before you decide to believe it.

### Problem: a label source that could delete an ingredient

Open Food Facts has real ingredient lists for a chunk of the catalog, which is better data than any template. The first version let the label override the template composition.

Frooti broke it. Its mango pulp row didn't fuzzy-match anything on the label, so it got dropped. The composition collapsed to sugar and water, both fully Indian, and the score shot up to 99%. The pipeline had made a product look *better* by deleting its main ingredient.

The fix is a one-way door. A label can add a "✓ on label" confirmation to a component. It can never remove one. On top of that, a generic-token blocklist, because "oil", "powder" and "flavour" happily match across materials that have nothing to do with each other.

### Problem: a rate limiter that limited nothing

Per-IP counts lived in an in-memory `Map`. On Vercel each lambda gets its own memory, so the counter reset more or less at random.

The global daily cap had a worse bug: it counted rows in the `LlmCall` table, and the deterministic path never makes an LLM call. That counter sat at zero permanently. Anyone could hammer `/api/search` and fill the database with junk products for free.

Now there's a `RateHit` table in Postgres with an upsert-increment keyed on (scope, window). 10 new analyses per IP per hour, 500 site-wide per day, and the limit applies only on a cache miss. Looking at a product someone already analysed costs nothing and is never throttled.

### Problem: migrations through a connection pooler

`prisma migrate deploy` ran during the Vercel build, so schema changes shipped with the code that needed them. Neat, and it does not work. `migrate deploy` can't hold its advisory lock over Neon's pgbouncer pooler, so every production deploy died on a P1002 timeout. One crashed run left lock `72707369` held and blocked every migration after it. Migrating mid-build also kills the build's own connection: `terminating connection due to administrator command`.

Migrations now run in a GitHub Actions workflow over a direct, non-pooled connection, and they clear any stale backend sitting on the lock before starting. Deploys got boring again.

### Problem: prerendering 1000 pages through a pool of 3

For SEO, `generateStaticParams` returned every product slug. Prisma's pool limit is 3 with a 10-second timeout. Builds crawled, then died around page 280, taking half an hour to fail.

Hero products are prerendered now and everything else is on-demand ISR, cached in Postgres after the first hit. Builds finish in seconds, every page is still server-rendered and indexable, and the sitemap still lists all 1006 URLs.

### What's live

1002 products across 44 categories. 56 HSN→GST rows, ~140 brand-to-country mappings, 25 fuel buildup rows covering petrol and diesel in 12 states plus national LPG. Daily mandi prices from Agmarknet feed the agricultural inputs.

The calibration harness runs against published anchors and currently reports GST exactness at 100%, label match at 84%, and every anchor inside its expected range.

Cost per query is about $0.0001, with a hard $20/month circuit breaker that has never come close to tripping. The bill stays near zero because the expensive part, the arithmetic, doesn't call anyone.

Try it on something in your kitchen: [parakhi.in](https://parakhi.in).
