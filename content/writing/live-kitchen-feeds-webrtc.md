---
title: "Live Kitchen Feeds: Transparency Without the Cloud Bill"
date: 2026-03-31
readTime: 3
draft: false
description: "Why food delivery apps need live streams, and how to build them cheaply with WebRTC."
---

For the last 3-4 years, most of my meals have come from apps like Zomato and Swiggy. And honestly, I’m pretty sure that food is often prepared in ways I probably wouldn’t want to see.

That got me thinking: What if customers could watch their food being cooked live? Not some polished marketing B-roll. Just a raw, live feed from the kitchen station while your order is actively being prepped.

Transparency builds trust. Simple as that.

The immediate pushback to an idea like this is always the infrastructure cost. If food delivery giants tried to stream thousands of kitchens using traditional cloud video architecture, their AWS bill would look like a phone number. 

But with today’s tech, it doesn't have to be that way.

### The Low-Cost Architecture

Instead of routing all that dense video data through centralized servers, we can just use peer-to-peer streaming via WebRTC. 

The setup is fairly basic:
You mount a cheap Android phone or a Raspberry Pi near the prep counter. It wakes up and automatically starts streaming only when an order drops. 

On the backend, you just need a lightweight signaling server (maybe Node.js or Go with WebSockets). This server's only job is to introduce the customer's phone to the kitchen's camera so they can shake hands. 

Once they connect, it establishes a direct P2P link. That means the heavy video traffic **never actually touches expensive cloud servers**. You're mostly just paying a few bucks a month for the signaling server and a STUN/TURN fallback. 

### What about millions of orders?

Scaling this is actually surprisingly chill. Since the heavy lifting, the video streaming, happens at the edge (between the two phones), your central servers are barely doing anything. 

A standard WebSocket signaling server can juggle tens of thousands of concurrent connections easily because it’s just passing tiny text tokens (SDPs) back and forth. To handle millions of daily orders, you just scale that signaling layer horizontally using something like Redis Pub/Sub. The cloud infrastructure cost barely flinches, even during the Friday night dinner rush.

### Handling the Dinner Rush

Okay, but what happens when a single cook is firing five different orders at the exact same time? You can't force a budget kitchen tablet on a greasy Wi-Fi network to upload five separate video streams. It would instantly choke.

The fix here is **station-based multicasting**. 

Instead of opening a 1:1 stream for every single order, the kitchen device captures one continuous video feed of the entire prep station. If five active orders are sitting there, the device pushes that *single* stream to a lightweight SFU (Selective Forwarding Unit), which then mirrors it out to all five customers.

But how do you know which burger is yours? 

This is where it gets fun. The kitchen app can run basic on-device object tracking or OCR to read the physical ticket numbers sitting on the counter. It then draws a simple AR overlay, like a glowing digital tag hovering over the plate reading “Order #1234”, directly onto the feed. You watch the wider station workflow, but you immediately spot your meal moving down the line.

### Why go through the trouble?

Beyond basic transparency, just having a camera on the line unlocks a lot of interesting data. You could automatically generate time-lapse cooking reels for social media, run lightweight AI models for hygiene detection, or even give restaurants a true, verifiable "Quality Score" based on how they actually handle the food.

Food delivery crushed the convenience problem years ago. But it still completely lacks transparency. With WebRTC and dirt-cheap hardware, letting users watch their food being prepped might just be the most effective trust-building feature we could ship right now. 

And surprisingly, it wouldn't even cost a fortune to build.
