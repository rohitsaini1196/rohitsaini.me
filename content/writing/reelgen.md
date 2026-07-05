---
title: "ReelGen: Stop Scrolling Stock Footage, Start Shipping Reels"
date: 2026-07-06
readTime: 3
draft: false
slug: "reelgen"
description: "I got tired of AI reels that look like AI reels. So I built a pipeline that actually cares about color, pacing, and footage quality."
---

Every AI-generated Reel looks the same. Same overused Pexels clips, same robotic zoom, same text that screams "a bot made this."

Got tired of it. Built something that doesn't.

Topic in, finished reel out. Color-graded, captioned, ready to post. About a penny per run.

### How it works

```
  "Hidden cafes in Tokyo"
          │
          ▼
   ┌──────────────┐
   │   LLM Plans   │    hook, beats, closer
   │   the Story    │    mood arc, color palette
   └──────┬───────┘    caption + hashtags
          │
          ▼
   ┌──────────────┐
   │  CLIP Finds   │    semantic match
   │  the Footage   │    not keyword luck
   └──────┬───────┘
          │
          ▼
   ┌──────────────┐
   │  FFmpeg Does   │    color grading
   │  the Grading   │    Ken Burns + handheld jitter
   └──────┬───────┘
          │
          ▼
   ┌──────────────┐
   │  Pillow Adds   │    animated text
   │  the Text      │    mood-matched font + style
   └──────┬───────┘
          │
          ▼
      reel.mp4
   1080×1920, IG-ready
       ~$0.01
```

### The actual problem with AI video tools

**Zero taste.**

Keyword search grabs the first "beach sunset" it finds. Warm or cold, sharp or mushy, doesn't matter. No color consistency across clips. No pacing, either hard cuts everywhere or dissolves everywhere, never a mix that feels human-edited.

Fixing that turned out to be a plumbing problem, not a model-size problem.

### What I did differently

```
┌─────────────────────────────────────────────────┐
│                                                 │
│   TWO HOOKS, ONE KEEPS                          │
│   LLM writes two opening angles.                │
│   Whichever finds better footage wins.          │
│                                                 │
├─────────────────────────────────────────────────┤
│                                                 │
│   SCORE EVERYTHING, FOUR WAYS                   │
│                                                 │
│   Every candidate clip gets ranked on:          │
│                                                 │
│     ✓ Semantic match (CLIP)                     │
│     ✓ Sharpness                                 │
│     ✓ Palette fit                               │
│     ✓ Motion stability                          │
│                                                 │
│   Pick the winner BEFORE settling.              │
│                                                 │
├─────────────────────────────────────────────────┤
│                                                 │
│   WEAK MATCH? TRY AGAIN, DIFFERENTLY            │
│                                                 │
│   Abstract caption like "you hit your limit"    │
│   has no literal visual. LLM rewrites it to     │
│   something more concrete. If footage still     │
│   can't carry it, text gets a stronger visual   │
│   anchor automatically.                         │
│                                                 │
└─────────────────────────────────────────────────┘
```

None of that needed a bigger model. Just better plumbing.

### The honest part

Free-tier LLM quotas cap out fast. I burned a full day's limit just testing this thing.

And good editing doesn't guarantee views. Posting cadence and trending audio still matter more than the cut. This makes reels you won't be embarrassed to post. It doesn't make them go viral for you.

### Try it

```bash
python reelgen.py generate \
  --topic "Hidden cafes in Tokyo's backstreets" \
  --mood calm \
  --style warm-analog
```

Everything runs locally. No cloud rendering, no subscription.

Code's open: [github.com/rohitsaini1196/ReelGen](https://github.com/rohitsaini1196/ReelGen)
