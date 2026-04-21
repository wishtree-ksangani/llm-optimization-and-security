# Gemini Prompt Caching — Complete Guide

---

## Why Prompt Caching?

Every time you call the Gemini API, **all tokens in your request are processed and billed** — even if the same large system prompt or context is sent repeatedly.

**The problem:** In many real-world applications, a large chunk of the prompt (system instructions, documents, background context) stays *identical* across hundreds or thousands of calls. You end up paying full price to re-process the same content over and over.

**Prompt caching solves this** by storing that repeated content server-side after the first call. Subsequent requests reference the cache instead of re-sending and re-processing the same tokens — resulting in **lower cost and faster responses**.

> In the notebook example, a 1,027-token travel assistant system prompt is cached once and reused across every user query, instead of being re-processed on each call.

---

## How It Works

```
First Request
─────────────────────────────────────────────────────
[System Prompt: 1,027 tokens]  ──► Processed + Cached
[User Query: ~20 tokens]       ──► Processed normally

Subsequent Requests
─────────────────────────────────────────────────────
[Cache Reference]              ──► Loaded from cache (discounted rate)
[User Query: ~20 tokens]       ──► Processed normally
```

### Step-by-Step (as in the notebook)

1. **Create a cache** — upload your large, reusable content once with a TTL (time-to-live)
2. **Get a cache name** — a unique identifier like `cachedContents/0pflf46xhvzd0v4w3l4qophnv9lrg126dkwrf8v2`
3. **Reference cache in requests** — pass the cache name in `GenerateContentConfig`
4. **Cache expires** — after the TTL, the cache is deleted automatically; recreate if needed

```python
# Step 1: Create cache
cache = client.caches.create(
    model="gemini-2.5-flash",
    config=types.CreateCachedContentConfig(
        system_instruction=prompt,   # large, reusable content
        ttl="3600s"                  # 1 hour TTL
    )
)

# Step 2: Use cache in requests
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="User question here...",
    config=types.GenerateContentConfig(
        cached_content=cache.name    # reference the cache
    )
)
```

---

## Pricing (Hourly Basis)

Caching has **two cost components**:

### 1. Cache Storage Cost (per hour)

| Model | Storage Cost |
|---|---|
| Gemini 2.5 Flash / Flash-Lite | **$1.00 / 1M tokens / hour** |
| Gemini 2.5 Pro | **$4.50 / 1M tokens / hour** |

> **Example:** Cache storing 1,027 tokens for 1 hour with `gemini-2.5-flash`:
> `(1,027 / 1,000,000) × $1.00 = ~$0.000001` per hour — negligible.

### 2. Cached Input Token Cost (per request)

When a request hits the cache, the cached portion is billed at a **discounted rate** (approx. 75% less than standard input tokens) instead of the full input price.

| Scenario | Cost |
|---|---|
| Regular input tokens | Full price |
| Cached input tokens | ~75% discount |
| New (non-cached) tokens in same request | Full price |
| Output tokens | Always full price |

### How to Think About It

```
Total Cost per Cached Request =
  (cached tokens × discounted input rate)
+ (new tokens × standard input rate)
+ (output tokens × output rate)
+ (cache storage × hourly rate × TTL hours)
```

**Break-even point:** If you send the same large prompt **more than ~4 times within the TTL window**, caching is almost always cheaper.

---

## When To Use Prompt Caching

| Use Case | Good Fit? |
|---|---|
| Large, fixed system prompt used across many queries | ✅ Yes |
| Long documents/PDFs queried multiple times | ✅ Yes |
| RAG context that doesn't change between calls | ✅ Yes |
| Chatbots with a fixed persona or instruction set | ✅ Yes |
| Short prompts (< minimum token threshold) | ❌ No |
| One-off or single requests | ❌ No |
| Prompts that change significantly every call | ❌ No |
| Very infrequent calls spread across hours | ⚠️ Calculate first |

---

## Minimum Token Limit for Caching

The cached content must meet a **minimum token threshold** — caching is ineffective for very small prompts.

| Model | Minimum Tokens to Cache |
|---|---|
| Gemini 2.5 Flash / Flash-Lite | **1,024 tokens** |
| Gemini 2.5 Pro | **4,096 tokens** |

> The notebook prompt is 1,027 tokens — just above the Flash minimum.  
> If your content is below the threshold, caching will either fail or provide no benefit.

---

## Best Practices

### Design Your Prompt for Caching
- Place **stable, large content at the beginning** of the prompt (system instructions, documents, context)
- Place **dynamic content at the end** (user query, session-specific data)
- The cache covers a *prefix* of your input — anything after the cached portion is processed normally

### Set TTL Appropriately
- Default TTL is **1 hour** (3600s) — same as used in the notebook
- Extend TTL for prompts used over a full workday: `ttl="28800s"` (8 hours)
- Shorter TTL = lower storage cost; longer TTL = fewer cache misses
- **Rule of thumb:** Set TTL slightly longer than your expected burst usage window

### Calculate Before You Cache
```
Storage cost for 1 hour = (token_count / 1,000,000) × hourly_rate
Savings per request     = (cached_tokens / 1,000,000) × (standard_rate - cached_rate)
Break-even requests     = storage_cost / savings_per_request
```
If you expect more requests than the break-even count within the TTL, **caching saves money**.

### Reuse the Cache Object
- Store `cache.name` and reuse it across your application session
- Check `cache.expire_time` before using — recreate if expired
- Do not create a new cache per request; that defeats the purpose

### Monitor Token Counts
- Use `response.usage_metadata` to track how many tokens were served from cache vs. freshly processed
- This helps validate that caching is working and quantify actual savings

---

## Quick Reference Summary

| Parameter | Gemini 2.5 Flash | Gemini 2.5 Pro |
|---|---|---|
| Minimum cache tokens | 1,024 | 4,096 |
| Storage cost | $1.00 / 1M tokens / hr | $4.50 / 1M tokens / hr |
| Cached input discount | ~75% off | ~75% off |
| Default TTL | 1 hour | 1 hour |
| Max TTL | Configurable | Configurable |

---

## Summary

| Question | Answer |
|---|---|
| **Why use it?** | Avoid paying to re-process the same large prompt on every call |
| **How does it work?** | Store prompt prefix server-side; reference by cache name in requests |
| **When to use?** | Large (>1K tokens), reusable content sent in many requests within hours |
| **Min token limit?** | 1,024 (Flash) / 4,096 (Pro) |
| **Pricing model?** | Storage per hour + discounted input rate per request |
| **Best practice?** | Put static content first, dynamic content last; match TTL to usage window |
