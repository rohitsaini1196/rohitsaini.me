---
title: "Debugging Distributed Systems"
date: 2026-02-05
readTime: 6
draft: false
description: "Debugging distributed systems is fundamentally different. You're reading timelines, not stack traces."
---

In a monolith, debugging is straightforward. Something breaks, you get a stack trace, you find the line, you fix it. The entire execution happened in one process, on one machine, in one timeline.

Distributed systems blow all of that up. The "stack trace" is scattered across five services, three message queues, and two databases. The bug might not even be a bug — it might be a timing issue that only happens under specific load patterns on the third Tuesday of the month.

I've spent enough time debugging distributed systems that I've developed a set of habits and patterns that consistently save me time. None of them are groundbreaking. They're just *discipline* applied before things break.

## Correlation IDs: Your Lifeline

The single most valuable piece of infrastructure in any distributed system is a correlation ID. It's a unique identifier generated at the entry point of a request that flows through every service, every queue, every log line.

Without it, debugging looks like this:

```
Service A logs: "Processed request for user 42"
Service B logs: "Received event for user 42"
Service C logs: "Sent notification for user 42"
```

Seems traceable, right? Now imagine 500 requests per second for user 42. Which log line in Service B corresponds to which log line in Service A?

With a correlation ID:

```
Service A: {"correlation_id": "abc-123", "action": "process_request", "user_id": 42}
Service B: {"correlation_id": "abc-123", "action": "handle_event", "user_id": 42}
Service C: {"correlation_id": "abc-123", "action": "send_notification", "user_id": 42}
```

Now you can grep for `abc-123` across all your services and reconstruct the exact path of a single request. This is non-negotiable. If you don't have this, add it today.

```python
# Middleware that generates or propagates correlation IDs
def correlation_middleware(request, next_handler):
    correlation_id = request.headers.get(
        "X-Correlation-ID", 
        str(uuid4())
    )
    # Attach to thread-local context for logging
    context.correlation_id = correlation_id
    
    response = next_handler(request)
    response.headers["X-Correlation-ID"] = correlation_id
    return response
```

The key rule: every time you publish a message, make an HTTP call, or write a log line, include the correlation ID. It should propagate like a virus through your entire system.

## Structured Logging, Not String Logging

When you're debugging a monolith with 100 log lines, `printf` debugging works fine. When you're debugging a distributed system with 10 million log lines, you need structured logging.

The difference:

```python
# Bad: stringly-typed and impossible to query
logger.info(f"Processing message {msg_id} for thread {thread_id}, took {latency}ms")

# Good: structured, filterable, aggregatable
logger.info("message_processed", extra={
    "message_id": msg_id,
    "thread_id": thread_id,
    "latency_ms": latency,
    "consumer_group": "notification-sender",
    "partition": partition,
    "offset": offset,
})
```

With structured logs, you can do things like:

- Find all messages where `latency_ms > 5000` across all services
- Filter by `consumer_group` to see only one consumer's perspective
- Aggregate `latency_ms` by `thread_id` to find hot threads
- Correlate high latency with specific Kafka partitions

This turns debugging from "reading log files" into "querying a database." Most log aggregation tools (Datadog, Kibana, CloudWatch) work dramatically better with structured data.

## Think in Timelines, Not Call Stacks

This is the biggest mental shift. In a monolith, execution is vertical — function A calls function B calls function C. You read a stack trace top to bottom.

In a distributed system, execution is horizontal — things happen *across time* on *different machines*. The order matters, the timing matters, and the gaps between events often tell you more than the events themselves.

When debugging, I build a timeline:

```
10:00:00.000  Service A    Received request          correlation_id=abc-123
10:00:00.015  Service A    Published event to Kafka   correlation_id=abc-123
10:00:00.015  Service A    Returned 200 to client     correlation_id=abc-123
10:00:02.340  Service B    Consumed event             correlation_id=abc-123
              ^^^ 2.3 second gap — why? Consumer lag? Slow deserialization?
10:00:02.380  Service B    Called Service C            correlation_id=abc-123
10:00:07.380  Service B    Timeout from Service C     correlation_id=abc-123
              ^^^ 5 second timeout — Service C is the problem
10:00:07.400  Service B    Sent to dead-letter queue  correlation_id=abc-123
```

The timeline immediately reveals the problem: Service C took too long to respond. But without seeing the full timeline, you might have spent an hour looking at Service B's code trying to figure out why it sent the message to the dead-letter queue.

## Replay Events to Reproduce Bugs

One of the advantages of event-driven systems is that events are stored (at least for a while). This means you can reproduce bugs by replaying events — something you can't do with synchronous HTTP calls.

In Kafka, you can reset a consumer group's offset to a specific timestamp:

```bash
# Reset consumer to replay events from a specific time
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --topic my-topic \
  --reset-offsets \
  --to-datetime 2026-02-05T10:00:00.000 \
  --execute
```

I've used this many times to reproduce production bugs in a staging environment. Copy the events from around the time of the bug, replay them against a staging consumer, and watch what happens. It's like having a time machine for your system.

Of course, be careful with this. Replaying events against a production database can cause duplicate processing. Always replay against an isolated environment.

## The Dashboard That Actually Helps

Most monitoring dashboards show you CPU, memory, and request counts. These are useful for capacity planning but nearly useless for debugging distributed systems.

The dashboard I actually look at during an incident has:

- **Consumer lag per consumer group.** If a consumer is falling behind, something is slow or failing.
- **P99 latency per service, per endpoint.** Not average latency (which hides problems) but the 99th percentile.
- **Error rate by error type.** Not just "errors went up" but "TimeoutError to Service C went from 0.1% to 15%."
- **Queue depth over time.** Is the queue growing? At what rate? This tells you if your consumers can't keep up.
- **Dead-letter queue count.** If messages are landing here, something is wrong and needs human attention.

```
┌─────────────────────────────────────────────┐
│  Consumer Lag                                │
│  notification-sender: 0 (healthy)            │
│  analytics-writer:    45,000 (⚠️ behind)     │
│  search-indexer:      0 (healthy)            │
├─────────────────────────────────────────────┤
│  DLQ Messages (last 24h)                     │
│  notification-sender: 3                      │
│  analytics-writer:    0                      │
│  search-indexer:      127 (🔴 investigate)   │
└─────────────────────────────────────────────┘
```

## Embrace the Uncertainty

Here's the hardest part about debugging distributed systems: sometimes you won't find a clean root cause. Sometimes the answer is "there was a 200ms network partition between zone A and zone B that caused a cascade of timeouts." There's no line of code to fix. There's no bug to close.

In those cases, the fix isn't fixing the bug — it's making the system resilient to that class of failure. Add retries. Add circuit breakers. Increase timeouts. Add fallbacks. Accept that partial failures are normal and design for them.

Distributed systems are fundamentally about trade-offs. You trade simplicity for scalability, consistency for availability, debuggability for independence. The trick is knowing which trade-offs you've made, and being prepared when they come to collect.

The best distributed systems debuggers I know aren't the ones who find bugs the fastest. They're the ones who build systems that explain themselves when things go wrong.

Invest in correlation IDs, structured logging, and good dashboards. You're building the tools your 3 AM self will thank you for.
