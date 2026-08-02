---
title: "ScamDB: The App Took a Weekend, The Data Took Three Months"
date: 2026-07-29
readTime: 5
draft: false
tags: ["build", "backend"]
slug: "scamdb"
description: "I built a search engine for suspicious Indian phone numbers and UPI IDs. The hard part was never the app. It was that the number is always inside a screenshot."
---

Someone in my family almost sent money to a "customs officer" who needed a clearance fee. The number was on WhatsApp. I googled it. Nothing. Two forums, one SEO farm, an answer from 2019.

That's the whole gap. Every scam number in India has already been reported by someone. It just isn't searchable.

So I built [scamdb.in](https://scamdb.in). Type a number or a UPI ID, see whether anyone has reported it.

The app was a weekend. Next.js 16, Supabase, Google OAuth, moderation queue, SEO-friendly URLs like `/phone/9876543210` so Google can actually index them. Boring, done, deployed.

Then I opened it and it had 34 entries. A search engine with 34 rows is not a search engine. It's a spreadsheet with a domain name.

The next three months were entirely about data.

### The pipeline I thought I was building

```
  r/indianscammers
        │
        ▼
  fetch posts  ──►  regex the number out  ──►  moderation  ──►  live
```

Two hours of work. Ship it.

### What actually happened

**~90% of posts came back `no_entities`.**

Not because the posts had no numbers. Because the number is in the screenshot.

Nobody types "here is the number that scammed me: 98765 43210." They post a WhatsApp screenshot. The number is 40 pixels tall, sitting in a green bubble, completely invisible to every regex I own.

Text scraping was never going to work here. OCR wasn't an optimization I could add later. It was the product.

### Reddit fought back on every front

Before I could OCR anything I had to actually get the posts. This took an embarrassing number of attempts:

* Anonymous `.json` endpoints: **403**.
* Browser-ish requests: you get the HTML webapp, not data.
* Move it to GitHub Actions so it runs on a schedule: datacenter IPs get rate-limited harder than my laptop did.
* RSS: works! Serves only new and hot. There is no historical anything.
* X/Twitter API free tier: **402**. It's write-only now. Reading costs $175/month.

The unblock was **arctic-shift**, the Pushshift successor. Full Reddit archive, no auth, no rate limits, doesn't care that I'm a datacenter, and it hands back direct full-res `i.redd.it` URLs.

First test: two screenshots in, nine numbers out. I stopped worrying about whether this would work.

### The actual pipeline

```
  arctic-shift archive
        │  paginate by created_utc
        ▼
  image posts only  ──►  i.redd.it full-res URL
        │
        ▼
  ┌──────────────┐
  │  vision OCR  │  Haiku, falls back to gpt-4o-mini
  │              │  model output regex-validated, always
  └──────┬───────┘
         │
         ▼
  raw_signals ──► deterministic extract ──► LLM enrich ──► moderation ──► live
```

Every OCR result gets re-validated by regex on the way out. Vision models are very happy to hand you a number that is *almost* the number in the image, and it looks fine until you dial it. So the transcription doesn't count for anything on its own. Only what survives the pattern goes in.

### Bugs that cost me real weekends

**Bug one: the crawler ran forever and made no progress.**

Each run pulled the same images, OCR'd them, found nothing, threw them away, and did it again six hours later. The dedup check only knew about images that had *produced* something. Dead images stayed permanently fresh.

Fix: write a "tried this, got nothing" marker. You have to record the failures too, or you just keep paying for them.

**Bug two: OpenAI's 200K TPM limit.**

Images are expensive in tokens. Bulk OCR hit the rate limit within a minute and spent the rest of the run 429-backing-off into a crawl. The correct fix was to pay for more throughput. I chose to let it accumulate slowly for free instead, which is a fix in the same way that "walking" is a fix for a broken car.

**Bug three: my regex was importing America.**

The phone pattern matched the last 10 digits of *any* number. `+1 863...`, `+977...`, all of it, cheerfully normalized into Indian mobile numbers. A US business had quietly become a suspicious Indian phone number in my database.

Fix: a `(?<!\d)` lookbehind so the match can't start mid-number, and a hard reject on any `+` prefix that isn't `+91`.

**Bug four: the Twitter crawler, where every single entity was a false positive.**

Not most. Every one. Flight numbers, order IDs, prices, dates. The signal-to-noise was so bad that the honest fix was `git rm`. That commit message is still my favourite in the repo.

### Where it is now

1,124 entities. 1,115 phones, 9 UPI IDs, 2,798 raw signals collected, 525 reports approved.

The 9 UPI IDs are not a bug. People screenshot the chat, not the payment page.

Once the archive was swept clean, new posts dropped to single digits per week. That's correct, and also boring, so I added two more sources.

Reddit **comments**: about 4% carry a number, pure text, zero vision cost. Every so often someone drops a bulk "here's the list" comment, which is a very good day.

Public aggregator pages: stored at low confidence, because they arrive with no story attached and I don't know who put them there.

Arctic-shift also 422s at random with "under maintenance." Treat it exactly like a 429, retry, move on.

### The honest part

999 reports are sitting in the moderation queue, unreviewed.

I built the auto-publish safeguard, wrote the whole "every report needs a human" rule, then became the human. There has been no moderation action since June 7. The bottleneck in my carefully designed pipeline is a guy who also has a job.

The other thing I'd say is that none of the interesting engineering was in the search. Search is a Postgres index. The interesting part was that the data had been sitting in public the entire time, in a format no computer could read.

The number that started this is in there now. Third search result.

