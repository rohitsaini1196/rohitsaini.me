---
title: "Data Flow Is Everything"
date: 2025-11-22
readTime: 6
draft: false
description: "Every backend problem eventually becomes a data flow problem."
---

I've noticed a pattern over the years. Whenever a system starts behaving weirdly — data shows up late, duplicates appear, something silently goes stale — the root cause is almost never a bug in the traditional sense.

It's a data flow problem.

Someone didn't think about where the data comes from, who transforms it, who owns the authoritative copy, and what happens when something in that chain fails.

## Start With a Map

Before writing any code for a new system, I draw the data flow. Not a UML diagram — nobody has time for that. Just boxes and arrows on a whiteboard (or, honestly, a notes app). Something like:

```
User Request
    ↓
API Gateway → Validation → Service A
                              ↓
                        Kafka Topic
                         ↓        ↓
                   Consumer B   Consumer C
                       ↓            ↓
                   Postgres     Elasticsearch
```

This simple sketch already surfaces questions you'd otherwise discover in production:

- Consumer B and Consumer C both process the same event. Are they idempotent? What if one succeeds and the other fails?
- Postgres and Elasticsearch will eventually diverge. Which one is the source of truth? How do you reconcile?
- If the API Gateway retries a failed request, does Service A handle duplicates?

I can't count the number of times a 10-minute whiteboard session like this prevented weeks of debugging later.

## The Four Questions

Whenever I look at any data flowing through a system, I ask four questions:

### 1. Where does it originate?

Every piece of data has a birth place. A user action, an external API, a scheduled job. Knowing the origin tells you about reliability — user inputs are messy, external APIs are flaky, scheduled jobs can overlap.

### 2. Who transforms it?

Data rarely stays in its original form. It gets validated, enriched, aggregated, filtered. Each transformation is a place where bugs hide. The more transformations, the harder it is to trace a value back to its origin.

Here's a pattern that bites people: Service A publishes an event with a raw user input. Service B enriches it by calling an external API. Service C receives the enriched version. Now, when something's wrong with the enriched data, you need to check three services to find the problem. Sometimes the external API returned bad data. Sometimes Service B had a caching bug. Sometimes Service A sent the wrong field.

### 3. Who owns it?

This is the question teams skip most often, and it causes the most pain. Ownership means: who is responsible when this data is wrong? If two services both write to the same row, you don't have shared ownership — you have *no* ownership.

A rule I follow: **one writer per data entity.** If `orders` service writes order status, nobody else does. Other services read it, react to it, cache it — but they don't write to it. This single constraint prevents an enormous class of data inconsistency bugs.

### 4. How does it fail?

This is where it gets interesting. Data flow failures are rarely boolean (works/doesn't work). They're usually partial, delayed, or silent:

- A Kafka consumer falls behind. Data is technically flowing, but 4 hours late.
- An API times out and a retry publishes a duplicate event.
- A database migration adds a new column, but the producer hasn't been updated. Now half the events have `null` in a field downstream services expect.

Drawing the failure paths — not just the happy paths — is what separates design docs that look good from systems that actually survive production.

## An Example: The Accidental Dual Write

A while back, I worked on a messaging system where we needed to update both a thread's status and send a notification whenever a new message arrived.

The first implementation was simple:

```python
def handle_new_message(message):
    db.update_thread_status(message.thread_id, "active")
    notification_service.send(message.thread_id, "new_message")
```

What could go wrong?

Everything.

If the database update succeeds but the notification call fails, the thread is marked active but nobody gets notified. If I wrapped both in a try-except and retried, I'd potentially send duplicate notifications. If I used a database transaction, it wouldn't matter because `notification_service` is a separate system — you can't transact across two different services.

The fix was to stop trying to do both at once. Instead, we leaned on the data flow:

```python
def handle_new_message(message):
    db.update_thread_status(message.thread_id, "active")
    # That's it. The DB change triggers a CDC event.

# Separate consumer picks up the change event
def on_thread_status_changed(event):
    if event.new_status == "active":
        notification_service.send(event.thread_id, "new_message")
```

Now each step does exactly one thing. The data flow is clear: message arrives → DB update → change event → notification. If the notification fails, we retry the event. No dual writes, no partial states.

## Why This Matters

Most architecture debates — monolith vs. microservices, SQL vs. NoSQL, sync vs. async — are actually data flow debates in disguise. Once you draw the flow clearly, the right architecture usually becomes obvious.

The system's behavior isn't defined by its components. It's defined by how data moves between them.

Start with the flow. The code follows.
