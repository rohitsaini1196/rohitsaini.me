---
title: "Blindspot, Part 1: Teaching a Pipeline to Watch the Whole Video"
date: 2026-08-03T10:00:00+05:30
readTime: 3
draft: false
tags: ["build", "backend"]
slug: "blindspot-reading-the-video"
description: "The first version processed a three-minute protest Reel successfully and missed the police crackdown at the end. Every frame it looked at came from the opening. Four rules came out of fixing that."
---

*Part 1 of three on [Blindspot](https://blindspot.buzz). [Part 2: extraction and the evidence boundary](/writing/blindspot-extraction-and-evidence/) · [Part 3: cost, queues, and delivery](/writing/blindspot-running-it/).*

I fed a three-minute protest Reel into the first version. It downloaded the video, transcribed the speech, pulled frames, searched the web, and produced a polished report.

It had missed the police crackdown near the end.

Every frame sent to the vision model came from the fast-cut opening. The pipeline reported success. It had never watched the second half, and nothing in the output said so.

Blindspot turns a public Instagram Reel or post into a claim-level report with citations, missing context, and explicit confidence. It is an evidence-retrieval system, not a truth detector. This part covers the media stage, which is where that first failure lived.

### Rule 1: never leave a section of the timeline unseen

The original sampler took the first N scene changes. On a fast-cut intro the frame budget was gone before the video reached anything substantive, so the model saw twelve variations of the same three seconds.

The fix was to stop looking for interesting frames and start guaranteeing coverage. `ffprobe` reports duration, FFmpeg samples evenly across the whole thing:

```
frame_count = clamp(round(duration_seconds / 14), 6, 16)
```

Roughly one frame per 14 seconds, floor of 6, ceiling of 16, scaled to 768px wide. Scene detection survives only as a fallback when duration probing fails.

The invariant changed from a heuristic about interestingness to a guarantee about coverage. Those fail very differently.

### Rule 2: one piece of content has two identities

Blindspot keys its cache twice, because platform identity and content identity answer different questions.

Platform identity is `instagram:<shortcode>`. Exact post already analysed, return the report.

Content identity is a SHA-256 of the media plus a 64-bit difference hash. SHA-256 catches byte-identical files. The perceptual hash survives re-encoding, resizing, and watermarks, so it recognises reposts of the same footage.

On a media-cache hit, Blindspot reuses the transcript and the extracted visual claims. It re-analyses the caption, account, publication date, and framing every time.

That split is the whole product. Authentic footage under a new false caption is the main case Blindspot exists to catch, so reusing the previous *verdict* would defeat it. Reusing the transcript is free; reusing the conclusion is wrong.

There's also a `FORCE_REEXTRACT=1` escape hatch. Without it, changing the extraction model and rerunning the same footage just benchmarks the old cached claims.

### Rule 3: every vendor is a seam

Local development runs `whisper.cpp` with Metal acceleration: free per run, a few seconds per Reel on Apple Silicon. Production prefers Azure Speech for Hindi-English code-mixed audio, with Bhashini dual ASR able to run both language tracks in parallel for comparison. Groq and OpenAI sit behind them.

All of it lives behind one transcription interface, selected by environment variable. Same for ingestion (`yt-dlp`, then Apify), LLM calls, and evidence retrieval.

This isn't architectural taste. Instagram changes its access behaviour, speech vendors change models, rate limits move. Baking one vendor into the pipeline turns every provider problem into an application rewrite.

### Rule 4: keep the derivatives, delete the source

FFmpeg pulls 16kHz mono PCM audio alongside the frames. Reels with no audio track are valid input and must not fail the run. Photo and carousel posts take a parallel path where the slides are the keyframes.

Once audio, keyframes, transcript, and hashes exist, the source video is deleted. Reports, transcripts, hashes, claims, citations, keyframes, and timings persist. The original media does not.

```
URL ──► ingest ──► ffprobe ──► even-interval keyframes ──► hashes
                      │                                      │
                      └──► 16kHz mono audio ──► transcribe ───┴──► cache
```

Cheap checks run before paid work, which is the only reason the cost numbers in Part 3 stay small.

Next: [why extraction, not research, sets the ceiling on report quality](/writing/blindspot-extraction-and-evidence/).
