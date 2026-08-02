---
title: "Sprout, Part 2: The Channel Shapes the Product"
date: 2026-08-03T14:00:00+05:30
readTime: 6
draft: false
tags: ["build", "backend", "systems"]
slug: "sprout-whatsapp-channel"
description: "WhatsApp is not a transport you send text down. It decides how identity works, how replies are shaped, when messages arrive, and what a tapped button looks like on the way back."
---

*Part 2 of two on [Sprout](https://sprout.fortwinai.com). [Part 1: taking the decision away from the model](/writing/sprout-decision-before-generation/).*

The routing layer in Part 1 was the interesting half to build. The half that decides whether people keep using it turned out to be the channel.

WhatsApp is not a pipe you push text down. It has opinions about identity, formatting, timing and what a button press looks like when it comes back to you, and each of those opinions became a design constraint.

### Identity is the phone number, not the conversation

The first version keyed everything on a conversation id, which is the obvious thing to do when a webhook hands you one.

Conversation ids are not stable the way a person is. The same human comes back later, arrives with a new conversation, and gets welcomed as a stranger by a bot that already knows which plants they own.

Identity moved to the phone key: welcome state at a 90 day TTL, and a plant collection at 180 days. A returning user is greeted by name and by plant, not by an introduction they have already read. Getting this wrong is quiet, because nothing errors. The system just behaves as though it has amnesia, which is the single fastest way to lose someone on a channel where every other conversation is with a human being.

### A tapped button comes back as plain text

Sprout offers quick replies: at most three buttons, at most 15 characters each, attached to the last bubble only.

When a user taps one, WhatsApp does not deliver a structured event through the same path as the message. It sends the button's own title back as an ordinary text message. The titles are short and friendly, which is exactly the shape the social-opener check looks for, so tapping a button could read as somebody saying hello for the first time.

Recent button titles are cached per user and matched before the opener check runs. Buttons also arrive shaped differently than you expect: the parser assumed a list of strings and had to learn to accept dictionaries, which failed silently and simply produced no buttons at all rather than an error.

### People type the way people talk

A single thought arrives as three messages in a row. Answering the first one is answering an interruption.

Incoming messages are buffered in Redis and released through SQS with an 8 second delay, so the pipeline sees the thought rather than its first fragment. That delay is also the cheapest quality improvement in the system: it costs nothing, needs no model, and removes an entire class of confused replies.

Everything before the model gets a chance to run is a cheap early return, in order: deduplicate on message id, honour a paused conversation, handle slash commands inline, send the welcome if it is owed, then daily and monthly limits. The expensive path is the last one anything reaches.

### Formatting is part of the answer

WhatsApp renders `*bold*` as literal asterisks, so markdown is stripped rather than written. Retrieval citations are removed before sending, because a plant diagnosis with a footnote reads like a term paper.

Replies are split into separate bubbles on a marker the model inserts, with a sentence-boundary fallback at 600 characters for when it ignores the marker. A typing indicator goes out before the first bubble.

None of this is cosmetic. A wall of text in a chat app reads as a form letter, and three short bubbles read as somebody answering you. The model produces the words, but the channel decides whether they land.

### Trust is a state problem, not a prompt problem

Two behaviours pushed people away, and neither was a wording issue.

**Agreeing with a correction and then ignoring it.** Someone says no, that is not tulsi. The model apologises and accepts it. Next turn, extraction runs over the conversation again, re-derives `plant=tulsi` from the earlier message, and the bot is back to talking about tulsi. The model was not being stubborn; the state was overwriting it. A rejection now clears the field and forces a plant-name question, so extraction cannot restore what the user just denied.

**Locking in a guess from a photo.** Vision identified a species and wrote it straight to the profile, where every later answer inherited it. Now a vision-identified plant lands as a candidate with `Yes ✓` and `Different plant` buttons, and only becomes the plant when the user confirms.

There is also a repetition guard. Responses are compared against the last three by Jaccard similarity, and anything over 0.8 triggers a single retry with varied openers and closers. All of this sits behind an environment flag, on by default, so any of it can be disabled in production without a redeploy.

### Memory that survives the thread

Plants are stored per phone number, and the collection is injected into diagnose, care and recovery prompts. The last five photo keys per plant are kept, so a follow-up can refer to what a leaf looked like a fortnight ago. There is a caption check for "this is an old photo" that skips image observations, which exists because a before-and-after photo is a normal thing to send and a naive pipeline diagnoses the before.

Older context is compressed rather than dropped: the last eight messages stay verbatim, everything earlier becomes a structured summary holding species, city, setup, symptoms, causes, advice given, remedies tried, what the photos showed, and which questions are still open.

### What a message costs

`gpt-4o-mini`, JSON mode, non-streaming, between $0.00015 and $0.00089 per message depending on whether retrieval and vision fire.

The levers are all about not calling things. `top_k` at 2 to 3 with a 0.45 to 0.50 threshold injects around 490 tokens instead of 730. A focus pre-check skips the model on roughly 15 % of messages. A cache avoids embedding a query for an agent with an empty knowledge base. Vision runs at low detail. An LRU chunk cache per Lambda instance takes a retrieval fetch from 2.3 seconds cold to effectively nothing warm. Quick-solve removes the model from the hottest path entirely, at around 200 ms.

When quick-solve misses, the honest latency is 3 to 10 seconds, and that is the number worth attacking next: widening deterministic coverage beyond the current four symptoms, and carrying those templates into Hindi and Hinglish rather than answering a Hinglish question in English.

Sprout runs on Lambda and SQS in ap-south-1, with Redis for conversation state and DynamoDB for the knowledge base. It is the first agent on a platform meant to carry others, which is why so little of what is described here is plant-specific. The channel constraints, the state model, the guard order and the cost levers belong to any assistant that has to live inside somebody's chat app.

Part 1 covers [the decision layer](/writing/sprout-decision-before-generation/), which is where the plant knowledge actually lives.
