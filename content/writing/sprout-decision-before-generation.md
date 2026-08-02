---
title: "Sprout, Part 1: A Prompt Is Not a Diagnosis"
date: 2026-08-03T13:00:00+05:30
readTime: 5
draft: false
tags: ["build", "llm", "systems"]
slug: "sprout-decision-before-generation"
description: "Sprout diagnoses plant problems over WhatsApp. Given a good prompt and a knowledge base it produced fluent, generic advice and asked questions forever. The fix was moving the decision out of the model and into Python."
---

*Part 1 of two on [Sprout](https://sprout.fortwinai.com). [Part 2: the channel shapes the product](/writing/sprout-whatsapp-channel/).*

Someone sends a photo of a tulsi plant with yellowing leaves and the message "mitti bahut geela hai". The soil is very wet.

A prompt-only bot asks how often they water it.

The answer is already in the message. Overwatering, on a plant that has been named, with a moisture signal supplied unprompted. But a language model handed a role and a knowledge base does what it is built to do, which is continue the conversation.

[Sprout](https://sprout.fortwinai.com) is a WhatsApp gardening companion for Indian home gardeners. No app, no signup: you message a number and it diagnoses plants from text, a photo, or a Hindi voice note, in English, Hindi or Hinglish. This part is about the layer that decides what to say, which turned out to matter more than what the model can write.

### The naive version was a good prompt and a knowledge base

```
  message ──► retrieve from KB ──► gpt-4o-mini ──► reply
```

Two hours of work, and most of that was writing the plant care documents.

It worked immediately, in the way that demos work. Fluent answers, correctly split into WhatsApp bubbles, grounded in a knowledge base written for Indian conditions rather than American ones. Tulsi, curry leaf, money plant, 45 °C summers, monsoon humidity, balconies instead of gardens.

They were also generic, and the conversation never converged. Every message got a model call. Every model call had the option of asking another question, so it asked another question.

### The assumption that was false

I assumed the model would decide when it had enough information. It cannot. A model has no stake in resolving your problem, and one more clarifying question is always locally reasonable. Left in charge of its own control flow it produces an interrogation that reads as diligence.

So the decision moved out of the model and into Python, and the model kept the part it is genuinely good at, which is phrasing.

```
  message
     │
     ▼
  extract entities        regex first, model as fallback
     │                    plant, symptom, city, setup, language
     ▼
  score certainty         pure arithmetic, no model call
     │
     ▼
  pick mode + action      social | diagnose | care | recovery
     │                    solve | ask_one_question | ask_photo
     ▼
  quick-solve?  ──yes──►  deterministic answer, model never runs
     │ no
     ▼
  mode-specific prompt  +  gated retrieval  +  last 4 turns
     │
     ▼
  gpt-4o-mini, JSON mode ──► split on [MSG] ──► bubbles
```

### Certainty is arithmetic, not a feeling

Certainty is a number computed from what is known, before anything is generated. A base score by context, then boosts.

| Known | Base |
|---|---:|
| Image + plant + symptom | 0.85 |
| Image + plant | 0.72 |
| Plant + symptom | 0.60 |
| Plant only | 0.38 |
| Symptom only | 0.32 |
| Nothing | 0.15 |

Moisture or soil mentioned adds 0.12. Visual detail 0.08. Light 0.06. City and setup 0.04 each. Each question the user answers adds 0.08, capped at two.

Then thresholds: 0.70 and above, solve. Between 0.45 and 0.69, ask exactly one question. Below 0.45, ask for a photo.

Two overrides earn their place. Root rot or pests with a known plant floors at 0.75, because those have visible signatures and asking more is theatre. Obvious-cause language, white powder, mushy, webbing, floors at 0.72.

A third one exists because the scoring was too eager: if the plant is unknown, cap at 0.64. Without that cap a vivid symptom description could clear 0.70 and produce a confident diagnosis of a plant nobody had identified. Certainty about the problem is not certainty about the subject.

### One question, then commit

`q_count >= 2` forces a solve or a photo request. There is no path through the decision engine that asks a third question.

This is a product rule enforced in code rather than in a prompt, because a prompt instruction not to over-ask survives about four turns of conversation. A counter does not care how reasonable the next question sounds.

The phrasing rule sits alongside it: a question is never allowed to be bare. The prompt requires inference first. "This often happens when the soil stays wet. Does the pot drain?" rather than "how often do you water it?". The user can answer the second half, or ignore it and still have learned something.

### The cheapest call is the one that never happens

Four symptoms answer themselves once the plant, the symptom and a moisture signal are known.

| Symptom | Gate | Answer |
|---|---|---|
| `yellow_leaves` | wet, soggy, geela | overwatering, pause 5 days |
| `drooping` | dry, sukha | underwatering, water deeply |
| `root_rot` | none | trim roots, repot |
| `pests` | none | neem oil every 5 days, 3 weeks |

When one fires the model is never called. Around 200 ms and zero tokens, against 5 to 10 seconds when it runs.

Back to the tulsi message. Plant tulsi, symptom yellow leaves, language hinglish. Base 0.60, plus 0.12 for geela, plus 0.08 for the visual detail: 0.80. Solve. Quick-solve matches. Three bubbles, no model call.

### Retrieval that always fires is retrieval you can't trust

The knowledge base is 17 India-specific files, 147 chunks, `text-embedding-3-small`, stored in DynamoDB. Retrieval is gated by the same mode and action decision. It fires for care and recovery answers and for diagnostic questions. It is skipped for `diagnose+solve`, where the diagnostic knowledge is already in the prompt, and for anything social. `top_k=3`, threshold 0.45, fails open.

Photos go through vision first, and the generated description becomes the retrieval query. Those queries score 0.68 and above, comfortably clear of text-only ones, because a model describing a photograph writes better search terms than a person typing on a phone.

### What the split buys

The routing layer is covered by its own tests: 83 across extraction, certainty, decision routing and retrieval gating, and 15 of 15 end-to-end routing scenarios, including Hinglish care questions that the earlier keyword set missed.

Those tests are possible because the decision is deterministic. You cannot unit test a prompt's judgement about whether it has enough information. You can unit test a function that returns `0.80` and `solve` for a given set of known fields, and that is the difference between a demo and something you are willing to leave running.

Next: [what the channel itself demands](/writing/sprout-whatsapp-channel/).
