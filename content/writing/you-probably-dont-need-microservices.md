---
title: "You Probably Don't Need Microservices"
date: 2023-06-15
readTime: 5
draft: false
description: "The microservices hype led teams to split too early. Most systems would be better off as a well-structured monolith."
---

In 2023, Amazon's Prime Video team published a blog post that broke the internet — at least the engineering corner of it. They moved a monitoring service from a distributed microservices architecture back to a monolith. The result? 90% cost reduction.

The reaction was predictable. Half the internet said "see, microservices are a scam." The other half said "you're missing the context." Both were partially right.

But the real takeaway wasn't about Amazon at all. It was about the rest of us — the teams that adopted microservices not because they needed to, but because it felt like the right thing to do.

## How We Got Here

Around 2015-2018, microservices became the default answer to every architecture question. Netflix talked about them. Uber talked about them. Conference talks were wall-to-wall microservices. If you were building a monolith, you were doing it wrong.

The pitch was compelling:

- Independent deployment
- Teams can move fast without stepping on each other
- Scale individual services based on demand
- Use the best language/framework for each service

All true. All real benefits. For organizations with hundreds of engineers deploying dozens of times a day.

The problem? Most of us aren't Netflix. Most teams have 5-15 engineers, deploy a few times a week, and are building products where the biggest bottleneck is figuring out what to build — not how to scale it.

## The Hidden Costs

What nobody talked about in those conference talks was the operational overhead. When you split a monolith into 10 services, you don't just get 10 independent deployables. You get:

- **10 CI/CD pipelines** to maintain
- **10 sets of logs** to search through when something breaks
- **Network calls** where you used to have function calls — each one a potential failure point
- **Distributed transactions** where you used to have database transactions
- **Service discovery, load balancing, circuit breakers** — infrastructure that didn't exist before because it didn't need to
- **Data consistency problems** because you now have 10 databases instead of one

I've seen teams spend more time debugging inter-service communication than building features. The service mesh becomes the product, and the actual product becomes an afterthought.

```
Monolith:
  function_a() → function_b() → function_c()
  Total failure points: 0 network calls

Microservices:
  Service A --HTTP-→ Service B --gRPC-→ Service C
  Total failure points: 2 network calls, 2 serializations,
                        2 deserializations, load balancer hops,
                        DNS resolution, TLS handshakes...
```

## The Monolith Isn't the Problem

When teams hit scaling issues with a monolith, the instinct is to break it apart. But most of the time, the monolith isn't the problem — the *structure* of the monolith is.

A well-structured monolith has clear module boundaries, separated concerns, and defined interfaces between components. It's basically microservices in a single process, without the network overhead.

```python
# A well-structured monolith — clear boundaries, single process

# orders/service.py
class OrderService:
    def create_order(self, user_id, items):
        order = self.order_repo.save(user_id, items)
        self.event_bus.emit("order.created", order)  # in-process event
        return order

# notifications/handlers.py
class NotificationHandler:
    def on_order_created(self, event):
        self.email_service.send_confirmation(event.order)

# billing/handlers.py  
class BillingHandler:
    def on_order_created(self, event):
        self.invoice_service.generate(event.order)
```

Same separation of concerns. Same independent logic. But function calls instead of HTTP calls. Shared transactions instead of distributed ones. One deployment instead of three.

If the day comes when the notifications team needs to deploy 10 times a day independently, or the billing module needs to scale to 100x the traffic of orders — *then* you extract it. Not before. Not because a conference talk said so.

## When Microservices Actually Make Sense

I'm not saying microservices are always wrong. They make sense when:

- **You have multiple teams** that need to deploy independently and you're stepping on each other's toes
- **One part of the system has fundamentally different scaling needs** — a real-time chat feature vs. a batch report generator
- **You're integrating systems across different technology stacks** — an ML pipeline in Python talking to a web app in Go
- **Organizational boundaries align with service boundaries** — Conway's Law is real

The key word is *need*. Not "might need someday" or "Netflix needs this." Actual, present-tense need backed by real pain.

## My Rule of Thumb

Start with a monolith. Structure it well. When a specific module has a clear, demonstrable reason to be independent — different scaling needs, different team ownership, different deployment cadence — extract it.

This isn't revolutionary advice. It's boring advice. And by now you know how I feel about boring.

The best architecture isn't the one with the most boxes on the diagram. It's the one where every box earns its operational overhead. For most teams, most of the time, that means fewer boxes than you think.
