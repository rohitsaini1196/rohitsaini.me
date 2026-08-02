---
title: "Blindspot, Part 2: Extraction Sets the Ceiling"
date: 2026-08-03T11:00:00+05:30
readTime: 3
draft: false
tags: ["build", "llm", "systems"]
slug: "blindspot-extraction-and-evidence"
description: "A weak extraction model found 1 of 7 checkable claims in a Reel. No amount of good research recovers a claim that never entered the request. Four rules about where the quality actually comes from."
---

*Part 2 of three on [Blindspot](https://blindspot.buzz). [Part 1: reading the video](/writing/blindspot-reading-the-video/) · [Part 3: cost, queues, and delivery](/writing/blindspot-running-it/).*

I expected evidence retrieval to be the main quality problem. The first measurements pointed one stage earlier.

On a single Reel, a nano extraction model found 1 of 7 material claims. A stronger model found all seven, and flagged the central framing problem on top: the Reel presented a contested claim still under inquiry as conclusively established.

No amount of excellent research recovers a claim that never entered the evidence request.

### Rule 1: a Reel is not a spoken article

The naive design treats a Reel as an article read aloud. Download, transcribe, fact-check the transcript.

But meaning is split across speech, on-screen text, the caption, charts visible for one second, the choice of footage, the publication date, and editing that changes which comparison the viewer sees.

A clip can be authentic while its caption supplies a false location. A number can be correct while the denominator is missing. One Reel can hold seven factual claims, two opinions, and a causal allegation no cited source establishes.

So extraction is one multimodal structured call over transcript, caption, post metadata, and keyframes together, returning strict JSON: checkable claims, on-screen OCR, named entities, events worth retrieving on, and observable framing indicators.

Parsed with `JSON.parse`, never regex. Claims and framing stay in separate fields. "The payment was ₹10 crore" is checkable. "This proves negligence" is a different kind of claim, extracted only if the Reel actually asserts it as fact.

### Rule 2: decide what not to spend money on

Most Instagram posts don't need a factual investigation. Music, aesthetic edits, jokes, vlogs, and pure political opinion should not be forced into a fact-check verdict.

A cheap triage stage runs after transcription and picks one of three routes:

```
A  no checkable claim   ──►  deterministic "nothing to check" report
B  opinion / framing    ──►  observable framing only, skip retrieval
C  factual claims       ──►  extraction, retrieval, synthesis
```

Routes A and B are safety decisions before they are cost decisions. A system that always returns a decisive factual verdict will manufacture one for input that doesn't support it.

### Rule 3: retrieval never gets a verdict

Evidence is another provider seam: built-in web search, an external facts corpus of ~14,000 documents, or both. Blindspot batches every extracted signal into one request, and the service returns passages with provenance: source, URL, publication date, and which claim or keyframe they address.

It is explicitly not allowed to decide whether the Reel is true.

```
Blindspot                          Facts service
---------                          -------------
read media, extract claims  ───►   retrieve relevant passages
send keyframes              ───►   match known images
synthesise report           ◄───   return facts with provenance
assign verdicts                    never assigns a verdict
```

The point is debuggability. When a conclusion is wrong I can tell whether extraction missed the claim, retrieval returned weak evidence, or synthesis overreached. One end-to-end answer hides all three behind fluent prose.

### Rule 4: tell the model what actually happened

An early report claimed Instagram could not be accessed. Plausible-sounding limitation. Completely false. The pipeline had downloaded the video, extracted audio, read keyframes, and produced a transcript.

What had actually happened was that web research found no corroboration. The model turned "search returned nothing" into "the media could not be fetched", and filed non-evidence notes as sources.

Synthesis input now carries `mediaAcquired` and `keyframeCount`, and the prompt states plainly what the pipeline completed. Missing external evidence and missing media are different sentences. Evidence entries must contain URLs; a null search result is represented as no external source found.

Give the model operational state. Don't expect it to infer operational state from incomplete context.

### Verdicts are categories, not a score

No truth percentage. A single number implies precision the evidence doesn't have and erases disagreement between claims inside one Reel. Outcomes are `supported`, `mixed_evidence`, `misleading_context`, `unsupported`, `satire`, `not_yet_verifiable`.

Every factual conclusion points at cited evidence, and confidence is shown separately from the verdict. A supported claim resting on indirect reporting is not the same object as one backed by a primary document.

The system is allowed to say it can't verify something yet. Thin sourcing and developing events are normal input, not exceptions.

Next: [what it costs, and the parts that break in production](/writing/blindspot-running-it/).
