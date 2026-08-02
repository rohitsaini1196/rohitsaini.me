---
title: "Brainrot: The Dumbest Video Format Taught Me the Most About Debugging"
date: 2026-07-07
readTime: 4
draft: false
tags: ["build", "debugging"]
slug: "brainrot-debugging"
description: "I automated those Reddit-story-over-Minecraft-parkour videos. The format is junk food. The bugs underneath were genuinely interesting."
---

You've seen the format. Reddit story, robotic narrator, Minecraft parkour underneath, captions bouncing word by word. It floods Shorts because it works, and because someone, somewhere, is paying editors per video to make it.

I automated the whole thing. Local TTS, free. Total cost per video: whatever one LLM call costs. Pennies.

### The pipeline

```
  mood ("funny")
        │
        ▼
  ┌─────────────┐
  │  LLM writes  │   8-20s micro-story
  │  the script  │   two synced versions (spoken + display)
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │   Kokoro     │   82M param TTS model
  │   narrates   │   Apache 2.0, runs on CPU, costs nothing
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │   Whisper    │   word-level timestamps
  │   times it   │   from the generated audio
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │   libass     │   karaoke captions
  │   burns it   │   word-by-word highlight
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │   FFmpeg     │   random gameplay slices
  │   cuts it    │   exact audio length, 1080x1920
  └─────────────┘
```

### The one design decision that actually mattered

Every story ships as **two scripts**.

A spoken script where "$13.60" becomes "thirteen dollars and sixty cents", because that's what the narrator says. And a display script where it stays "$13.60", because that's what belongs on screen.

Same sentences, same order, different formatting. Skip this and your captions drift the moment a number shows up.

### The alignment trick

Trust Whisper's ears, never its spelling.

Whisper transcribed my narrator saying "al dente" as "al dent." Doesn't matter. I already know the script, I wrote it. So the transcript text gets thrown away entirely and only the timestamps survive. Those timestamps get matched back onto the real script with a diff.

Sync stays word-perfect even when Whisper mishears.

That was the fun part. Then the machine fought back.

### The bugs that ate my weekend

**Bug one: the pipeline froze at 0% CPU.**

Not slow. Parked. PyTorch (for the TTS) and CTranslate2 (for Whisper) each ship their own OpenMP runtime. When both load into one process, their thread pools deadlock. The famous `KMP_DUPLICATE_LIB_OK=TRUE` workaround only silences the warning. macOS's `sample` tool showed the main thread suspended in `__kmp_suspend_64`, waiting for a signal that never comes.

Fix: force both libraries single-threaded before either one imports. Slower per step, but slow beats never.

**Bug two: models I already downloaded still phoned home.**

Hugging Face checks for updates on every load, and on my network that check just hung. `HF_HUB_OFFLINE=1`, gone.

**Bug three: Telegram delivery hung past its own timeout.**

The request had `timeout=120`. It hung anyway. Turns out my network's IPv6 route to `api.telegram.org` is dead. TCP sits in `SYN_SENT` at the OS level, below where Python's timeout lives. `curl -6`: hangs. `curl -4`: instant. One monkeypatch to force IPv4 and delivery worked forever after.

**None of these bugs were in my code.** All of them ate hours. That's the real cost of "local-first": you inherit every layer under you.

**One more:** emoji in burned-in captions render as tofu boxes, because libass has no color-emoji fallback. They get stripped from the video and live on in the post caption instead. Nobody misses them.

### The honest part

Renders take a couple of minutes each on CPU. The free LLM tier caps you around 20 videos a day. And the gameplay footage licensing is on you, "found it on YouTube" is not a license.

Also, this format is junk food. I know. That's why the interesting parts are underneath it.

Code's open, same repo as ReelGen: [github.com/rohitsaini1196/ReelGen](https://github.com/rohitsaini1196/ReelGen)
