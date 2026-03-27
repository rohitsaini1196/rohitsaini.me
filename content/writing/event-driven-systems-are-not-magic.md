---
title: "Event-Driven Systems Are Not Magic"
date: 2026-02-20
readTime: 7
draft: false
description: "Event-driven systems are powerful, but they redistribute complexity rather than eliminating it."
---

There's a moment in every growing system's life where someone says, "We should make this event-driven." And most of the time, they're right. Decoupling producers from consumers is genuinely powerful. It lets teams work independently, services scale independently, and the overall system becomes more resilient.

But here's what nobody mentions in the conference talks: events don't eliminate complexity. They redistribute it.

And if you're not prepared for where that complexity lands, you're going to have a worse time than you did with synchronous calls.

## What Event-Driven Actually Gives You

Let me start with the good parts, because there are genuinely good parts.

Say you have a system where creating a new user account needs to:
1. Store the user in the database
2. Send a welcome email
3. Create a default workspace
4. Notify the analytics service

The synchronous version looks like this:

```python
def create_user(request):
    user = db.create_user(request.data)
    email_service.send_welcome(user.email)
    workspace_service.create_default(user.id)
    analytics.track("user_created", user.id)
    return user
```

This is simple to read and simple to debug. But it has problems. If the email service is slow, the API response is slow. If the analytics service is down, user creation fails. Every dependency is a single point of failure.

The event-driven version:

```python
def create_user(request):
    user = db.create_user(request.data)
    event_bus.publish("user.created", {"user_id": user.id, "email": user.email})
    return user

# Separate consumers, running independently
def on_user_created_send_email(event):
    email_service.send_welcome(event["email"])

def on_user_created_setup_workspace(event):
    workspace_service.create_default(event["user_id"])

def on_user_created_track_analytics(event):
    analytics.track("user_created", event["user_id"])
```

Now the API is fast — it does one thing and publishes an event. Each side effect runs independently. If the analytics service is having a bad day, users still get created and emails still go out.

This is the pitch. And it's a real improvement.

But now let's talk about what comes next.

## Problem 1: Ordering

Events in a queue aren't guaranteed to arrive in the order you think.

Say a user creates an account and immediately updates their profile. You now have two events:

```
Event 1: user.created  { user_id: 42, name: "Rohit" }
Event 2: user.updated  { user_id: 42, name: "Rohit S." }
```

If both events land on different partitions, or if Consumer A processes events faster than Consumer B, you might process the `updated` event before the `created` event. Now your downstream service is trying to update a user that doesn't exist yet.

In Kafka, you can solve this by using the `user_id` as the partition key. All events for the same user land on the same partition and are processed in order. But this only works within a single topic. Across topics? You're on your own.

```python
# Producing with a partition key
producer.send(
    topic="user-events",
    key=str(user_id).encode(),  # same user → same partition → ordered
    value=serialize(event)
)
```

There's no universal ordering across an event-driven system. You need to design for partial ordering and be very explicit about which ordering guarantees you actually need.

## Problem 2: Idempotency

Events will be delivered more than once. This isn't a maybe — it's a guarantee. Network blips, consumer restarts, rebalances — all of these cause redelivery.

If your consumer does this:

```python
def on_payment_received(event):
    db.add_credit(event["user_id"], event["amount"])
```

And the event gets delivered twice, the user just got double credit. Congratulations, you now have a financial accounting bug.

The fix is making every consumer idempotent. The simplest way is to track which events you've already processed:

```python
def on_payment_received(event):
    if db.event_already_processed(event["event_id"]):
        return  # skip duplicate
    
    db.add_credit(event["user_id"], event["amount"])
    db.mark_event_processed(event["event_id"])
```

This works, but now you have an extra database table, extra queries on every event, and you need to think about cleanup (that table grows forever). For simpler cases, you can use natural idempotency — operations like `SET status = 'active'` are inherently idempotent, while `INCREMENT balance BY 100` is not.

## Problem 3: Schema Evolution

Your event schema *will* change. New fields get added. Old fields get deprecated. The structure of the payload evolves as the product evolves.

But here's the catch: producers and consumers are deployed independently. When you add a new field to the `user.created` event, there's a window where the producer is sending the new schema but the consumer is still running the old code.

```python
# Version 1 of the event
{"user_id": 42, "email": "rohit@example.com"}

# Version 2, six months later
{"user_id": 42, "email": "rohit@example.com", "plan": "premium", "source": "referral"}
```

Consumers need to handle both versions gracefully. This means:

- **Always use optional fields for new data.** Never require a field that didn't exist in v1.
- **Never remove or rename fields.** Add new ones, deprecate old ones.
- **Version your events.** Include a `schema_version` field so consumers can branch on it if needed.

Some teams use schema registries (like Confluent Schema Registry for Kafka) to enforce compatibility. That adds infrastructure but catches breaking changes before they hit production.

## Problem 4: Debugging Across Services

This is the one that hurts the most. In a monolith, debugging is straightforward. You add a breakpoint, step through the code, and follow the logic. In an event-driven system, the "logic" spans multiple services, multiple queues, and multiple timelines.

A user reports: "I created an account but never got my welcome email."

In the monolith: check `create_user`, find the bug. Ten minutes.

In the event-driven system:
1. Did the `user.created` event get published? Check producer logs.
2. Did Kafka receive it? Check the topic.
3. Did the email consumer pick it up? Check consumer group lag.
4. Did the consumer process it successfully? Check consumer logs.
5. Did the email service actually send the email? Check that service's logs.

You're now debugging across five different log streams, and you need a correlation ID to tie them all together.

```python
# Always propagate a correlation ID through your events
event = {
    "event_id": str(uuid4()),
    "correlation_id": request.headers.get("X-Correlation-ID", str(uuid4())),
    "user_id": user.id,
    "email": user.email,
    "timestamp": datetime.utcnow().isoformat()
}
```

If you don't set this up from day one, you'll wish you had by month two.

## So When Should You Use Events?

Events are great when:

- The producer doesn't need to know the result of the consumer's work
- You have multiple independent reactions to the same trigger
- You need resilience — consumers can retry independently without blocking each other
- Workloads are bursty and you need a buffer

Events are overkill when:

- You have a simple request-response flow
- You need strong consistency and the result immediately
- Your system is small enough that the operational overhead of queues outweighs the benefit
- You only have one consumer (just call it directly)

## The Real Work

Using events isn't the hard part. *Anyone* can publish a message to Kafka. The real work is everything that comes after:

- Designing for out-of-order delivery
- Making every consumer idempotent
- Evolving schemas without breaking things
- Building observability that spans service boundaries

The first message you publish is satisfying. The hundredth production bug you debug across an event-driven pipeline is humbling.

Events are a tool. A powerful one. But they're not magic. Treat them with the respect they deserve, and they'll serve you well. Treat them as a silver bullet, and they'll redistribute your complexity to places you weren't looking.
