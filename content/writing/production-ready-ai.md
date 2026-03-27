---
title: "Production-Ready AI"
date: 2026-01-18
readTime: 5
draft: false
description: "Adding AI to a system is easy. Making it reliable is not."
---

Adding AI to a system is easy. You grab an API key, write a prompt, call the model, and show the result. It works in the demo. The stakeholders nod approvingly.

Then you deploy to production, and everything falls apart in ways you didn't anticipate.

The model hallucinates a response that contradicts your own data. Latency spikes to 8 seconds because the model is thinking really hard about a simple question. Your monthly bill triples because someone left a retry loop running against a large-context model. A customer sees a response that's technically correct but completely wrong for their context.

The gap between demo and production AI is enormous. And most of it has nothing to do with the model itself.

## The Things That Actually Matter

### 1. Deterministic Fallbacks

Here's something I learned early: your AI feature needs to work *without* the AI.

That sounds contradictory, but it's the most important design decision you'll make. If the AI model is unavailable — the API is down, latency is too high, the response is garbage — your system should degrade to something useful, not crash or show a blank page.

For a search system, that means falling back to keyword search when semantic search fails. For an auto-reply feature, it means sending a canned response instead of nothing. For a classification pipeline, it means defaulting to a "needs human review" bucket.

```python
def get_ai_response(prompt, context):
    try:
        response = llm.complete(
            prompt=prompt,
            timeout=3.0  # hard timeout, non-negotiable
        )
        if response.confidence < 0.7:
            return fallback_response(context)
        return response.text
    except (TimeoutError, APIError):
        return fallback_response(context)
```

The fallback isn't a failure — it's a feature. Users care far less about "AI-powered" than they care about "it works."

### 2. Observability

AI systems are black boxes by default. The model takes an input, does mysterious computation, and produces an output. If the output is wrong, you have almost no traditional debugging tools.

This is why observability for AI isn't optional — it's critical. For every AI call, I log:

- **The full prompt** (or a hash of it if it contains sensitive data)
- **The model's response**
- **Latency**
- **Token count** (input + output)
- **Confidence score** (if available)
- **Whether the fallback was triggered**
- **The user's reaction** (did they accept the suggestion? dismiss it? edit it?)

```python
def log_ai_interaction(prompt, response, metadata):
    logger.info("ai_interaction", extra={
        "prompt_hash": hash(prompt),
        "response_length": len(response),
        "model": metadata.model,
        "latency_ms": metadata.latency_ms,
        "input_tokens": metadata.input_tokens,
        "output_tokens": metadata.output_tokens,
        "fallback_used": metadata.fallback_used,
        "correlation_id": metadata.correlation_id,
    })
```

This lets you answer questions like: "Why did our AI accuracy drop last Tuesday?" Maybe a prompt template was deployed with a typo. Maybe the context window was being exceeded for longer inputs. Maybe the model provider quietly changed their default model version. Without these logs, you're guessing.

### 3. Cost Boundaries

AI API calls cost real money, and the cost model is different from anything else in your infrastructure. It's not per-request or per-second — it's per-token. A complex prompt with a large context can cost 100x more than a simple one.

I set cost boundaries at multiple levels:

- **Per-request budget.** If a single prompt would cost more than $0.05, truncate the context or use a smaller model.
- **Per-user daily budget.** Prevents a single heavy user from blowing up your bill.
- **Per-service monthly budget.** Hard alerts when spend exceeds projections.

```python
def estimate_cost(prompt, model="gpt-4"):
    token_count = tokenizer.count(prompt)
    cost_per_token = MODEL_COSTS[model]
    estimated_cost = token_count * cost_per_token
    
    if estimated_cost > MAX_COST_PER_REQUEST:
        # Switch to cheaper model or truncate context
        return use_lighter_model(prompt)
    return call_model(prompt, model)
```

The most expensive bug I've seen in an AI system wasn't a logic error — it was a retry loop that kept sending the same large prompt to GPT-4. The team didn't notice for two days. The bill was not pleasant.

### 4. Latency Budgets

AI model calls are slow compared to a database query. An LLM might take 2-5 seconds for a reasonable response. If your user expects a page load in under a second, you can't synchronously call an LLM in the request path.

Strategies that work:

- **Async processing.** Queue the AI call and show results when ready (works for background jobs like classification).
- **Streaming.** Stream tokens as they arrive so the user sees something happening (works for chat-like interfaces).
- **Pre-computation.** Run AI during off-peak hours and cache results (works for recommendations, summaries).
- **Model selection based on latency.** Use a fast, smaller model for real-time features and a large model for batch jobs.

The key is deciding your latency budget upfront and designing around it, rather than adding a loading spinner and hoping nobody complains.

## AI Should Never Be a Single Point of Failure

This is my one non-negotiable rule. AI features should enhance the experience, not gate it. If the AI is down, the core product should still work. If the AI gives bad output, there should be a way for the system (or the user) to recover gracefully.

Think of AI like a suggestion engine:

- Suggestions can be wrong. That's fine, as long as the user can override them.
- Suggestions can be slow. That's fine, as long as the rest of the system doesn't wait.
- Suggestions can be expensive. That's fine, as long as you have guardrails.

The best AI features I've shipped are the ones where users don't even realize AI is involved. It just feels like the product is smart. That's the bar — not flashy "Powered by AI" badges, but quietly useful features that degrade gracefully and cost predictably.

Making AI work in a demo takes an afternoon. Making it production-ready takes weeks. The difference is everything that surrounds the model — fallbacks, logging, cost controls, latency management, and the humble acceptance that sometimes the best AI response is no AI response at all.
