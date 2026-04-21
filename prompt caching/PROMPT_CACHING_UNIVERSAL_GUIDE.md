# Prompt Caching — Universal Guide (All APIs & Open Source)

> Covers: OpenAI · Anthropic Claude · Google Gemini · DeepSeek · Mistral · vLLM · Ollama · SGLang

---

## What Is Prompt Caching?

Every LLM request processes every token in the input from scratch — even if the same large system prompt or document was sent in the previous request. **Prompt caching** stores that repeated prefix server-side (or in local GPU memory) so subsequent requests can reuse it at a much lower cost and with lower latency.

```
Without Caching                  With Caching
─────────────────                ──────────────────────────────
[System prompt: 2000 tok]  ──►   [Cache reference]   (cheap read)
[User query: 50 tok]       ──►   [User query: 50 tok] (full price)
Process 2050 tokens              Process ~50 new tokens
Bill for 2050 tokens             Bill for 50 tokens + tiny storage
```

---

## How Caching Works (Conceptually)

All providers use the same core concept: **prefix matching on the KV cache**.

1. The model processes your prompt and produces internal key-value (KV) states for each token
2. If the beginning of a new request exactly matches a previously cached prefix, those KV states are reloaded instead of recomputed
3. Only the new (non-matching) tokens are processed from scratch
4. **Golden rule:** Put stable content first, dynamic content last

---

## Provider-by-Provider Breakdown

---

### 1. OpenAI

**Models:** GPT-4.1, GPT-4o, GPT-5.4, o-series, and most production models

**How it works:** Fully **automatic** — no code changes needed. OpenAI detects shared prefixes across requests and caches them transparently.

#### Pricing

| Model | Standard Input | Cached Input | Output |
|---|---|---|---|
| GPT-4.1 | $2.00 / 1M | $0.50 / 1M (75% off) | $8.00 / 1M |
| GPT-4o | $2.50 / 1M | $1.25 / 1M (50% off) | $10.00 / 1M |
| GPT-5.4 Standard | ~$2.50 / 1M | $0.25 / 1M (~90% off) | varies |
| o3-mini | $1.10 / 1M | $0.275 / 1M (75% off) | $4.40 / 1M |

> No separate storage fee. No write premium. Caching is a free optimization on top of standard billing.

#### TTL Options

| Policy | Duration | How to Set |
|---|---|---|
| `in_memory` (default) | 5–10 min idle, max 1 hour | Automatic |
| `24h` | Up to 24 hours | `prompt_cache_retention: "24h"` |

> Extended 24h retention is available on GPT-4.1, GPT-5.1, GPT-5.4, and select models.

#### Minimum Token Limit
**1,024 tokens** — caching does not activate below this threshold.

#### Code Example
```python
from openai import OpenAI
client = OpenAI()

# No extra setup — just send your request normally
response = client.chat.completions.create(
    model="gpt-4.1",
    messages=[
        {"role": "system", "content": "...your large system prompt here..."},
        {"role": "user", "content": "User question here"}
    ]
)

# Check cache hits
cached = response.usage.prompt_tokens_details.cached_tokens
print(f"Cached tokens: {cached}")
```

---

### 2. Anthropic Claude

**Models:** All active Claude models (Opus 4.x, Sonnet 4.x, Haiku 3.x/4.x)

**How it works:** **Explicit** — you must mark content with `cache_control` blocks to tell Claude what to cache. Gives you fine-grained control over what gets cached.

#### Pricing (Multipliers on Base Input Rate)

| Operation | Cost Multiplier | Effect |
|---|---|---|
| Cache Write (5 min TTL) | **1.25×** base input | Slightly more expensive first time |
| Cache Write (1 hour TTL) | **2.0×** base input | More expensive, but longer reuse |
| Cache Read (hit) | **0.1×** base input | **90% off** — massive savings |

**Example — Claude Sonnet 4.6 (base input: $3.00/MTok):**

| Operation | Cost |
|---|---|
| Regular input | $3.00 / 1M tokens |
| 5-min cache write | $3.75 / 1M tokens |
| 1-hour cache write | $6.00 / 1M tokens |
| Cache read (hit) | $0.30 / 1M tokens |

#### TTL Options

| TTL | Write Cost | Best For |
|---|---|---|
| **5 minutes** (default) | 1.25× base | Short burst sessions |
| **1 hour** | 2.0× base | Long sessions, production apps |

> Set 1-hour TTL with: `{"type": "ephemeral", "ttl": "1h"}`

#### Minimum Token Limits by Model

| Minimum Tokens | Models |
|---|---|
| **1,024** | Claude Sonnet 4.5, Opus 4.1, Opus 4, Sonnet 4, Sonnet 3.7 |
| **2,048** | Claude Sonnet 4.6, Haiku 3.5, Haiku 3 |
| **4,096** | Claude Opus 4.7, Opus 4.6, Opus 4.5, Haiku 4.5 |

> If your content is below the threshold, the request processes normally with no error — caching is silently skipped.

#### Code Example
```python
import anthropic
client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "...your large system prompt...",
            "cache_control": {"type": "ephemeral", "ttl": "1h"}  # explicit cache
        }
    ],
    messages=[{"role": "user", "content": "User question here"}]
)

# Check usage
print(response.usage.cache_read_input_tokens)   # tokens served from cache
print(response.usage.cache_creation_input_tokens)  # tokens written to cache
```

---

### 3. Google Gemini

**Models:** Gemini 2.5 Flash, Flash-Lite, 2.5 Pro, and prior generations

**How it works:** **Explicit** — you create a named cache object with a TTL, then reference it by name in each request. Billing has two components: storage (per hour) + discounted input rate per request.

#### Pricing

**Storage Cost (per hour while cached):**

| Model Family | Storage Rate |
|---|---|
| Flash / Flash-Lite | **$1.00 / 1M tokens / hr** |
| Pro | **$4.50 / 1M tokens / hr** |

**Cached Input Token Rate:**
- Approximately **75% off** standard input rate when a request hits the cache

#### TTL

- Default: **1 hour** (3600 seconds)
- Configurable: set any duration in seconds, e.g., `ttl="28800s"` for 8 hours
- Cache auto-deletes on expiry; recreate if needed

#### Minimum Token Limits

| Model | Minimum Tokens |
|---|---|
| Gemini 2.5 Flash / Flash-Lite | **1,024 tokens** |
| Gemini 2.5 Pro | **4,096 tokens** |

#### Code Example
```python
from google import genai
from google.genai import types

client = genai.Client()

# Step 1: Create and store the cache
cache = client.caches.create(
    model="gemini-2.5-flash",
    config=types.CreateCachedContentConfig(
        system_instruction="...your large prompt...",
        ttl="3600s"
    )
)

# Step 2: Use the cache in requests
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="User question here",
    config=types.GenerateContentConfig(
        cached_content=cache.name
    )
)

# cache.name looks like: "cachedContents/abc123xyz"
# cache.expire_time tells you when it will be deleted
```

---

### 4. DeepSeek

**Models:** DeepSeek-V3.2 (Chat), DeepSeek-R1 (Reasoner), DeepSeek-V4

**How it works:** **Automatic** — DeepSeek detects shared prefixes and applies cache pricing automatically. Called "cache hit" vs "cache miss" in their pricing model.

#### Pricing (per 1M tokens)

| Model | Cache Miss (Input) | Cache Hit (Input) | Discount | Output |
|---|---|---|---|---|
| DeepSeek-V3.2 | $0.28 | $0.028 | **90% off** | $0.42 |
| DeepSeek-R1 | $0.55 | $0.14 | **75% off** | $2.19 |
| DeepSeek-V4 | $0.30 | $0.03 | **90% off** | $0.50 |

> No storage fee, no write premium — just two tiers of input pricing.

**Off-Peak Bonus:** Extra 50–75% discount during off-peak hours (16:30–00:30 GMT).

#### Minimum Token Limit
No official public minimum documented; caching activates automatically on shared prefixes.

#### Code Example
```python
from openai import OpenAI  # DeepSeek uses OpenAI-compatible API

client = OpenAI(
    api_key="your-deepseek-api-key",
    base_url="https://api.deepseek.com"
)

# No setup needed — caching is automatic
response = client.chat.completions.create(
    model="deepseek-chat",
    messages=[
        {"role": "system", "content": "...large system prompt..."},
        {"role": "user", "content": "User question"}
    ]
)

# Check cache status in usage
print(response.usage.prompt_cache_hit_tokens)
print(response.usage.prompt_cache_miss_tokens)
```

---

### 5. Mistral

**Models:** Mistral Small, Mistral Large, Codestral, etc.

**Status:** Mistral does **not** currently offer explicit prompt caching as a billable feature in their API. There is internal prefix caching at the infrastructure level but it is not exposed as a pricing tier or user-controlled feature.

> **Recommendation:** If cost optimization on repeated large prompts is critical with Mistral, consider self-hosting via vLLM (see below) or batching requests to maximize natural prefix reuse.

---

## Open Source / Self-Hosted Models

When you run models locally or on your own infrastructure, prompt caching works at the **inference engine** level — there is no dollar cost, only compute and memory savings.

---

### 6. vLLM — Automatic Prefix Caching (APC)

**Use with:** Any HuggingFace-compatible model (Llama 3.x, Mistral, Qwen, Gemma, DeepSeek-R1, etc.)

**How it works:** vLLM maintains a **radix tree** of KV cache blocks. When a new request shares a prefix with a cached request, those KV blocks are reused instead of recomputed. Reduces both latency and GPU compute.

#### Benefits
- No cost (compute savings only)
- Works automatically across all requests sharing a prefix
- Significant throughput improvement for chatbot and RAG workloads
- Supports multi-GPU tensor parallelism

#### How to Enable
```bash
# Enable prefix caching at server startup
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --enable-prefix-caching \
  --gpu-memory-utilization 0.9

# Or in Python
from vllm import LLM, SamplingParams

llm = LLM(
    model="meta-llama/Llama-3.1-8B-Instruct",
    enable_prefix_caching=True
)
```

#### Best Practices
- Keep the system prompt identical across requests (even a single space difference breaks prefix matching)
- Use `--max-model-len` to manage memory when caching long prompts
- Monitor with `--collect-detailed-traces` to see cache hit rates

---

### 7. llama.cpp / Ollama

**Use with:** GGUF-quantized models (Llama, Mistral, Phi, Gemma, etc.)

**How it works:** llama.cpp maintains a **prompt cache file** on disk or in memory. When a request starts with a previously processed prefix, it loads the saved KV state instead of re-running the forward pass.

#### llama.cpp
```bash
# Save and reuse prompt cache
./llama-cli \
  -m model.gguf \
  --prompt-cache prompt_cache.bin \
  --prompt-cache-all \
  -p "...your system prompt here..."
```

#### Ollama
Ollama handles KV caching automatically when the same model is kept loaded in memory (`--keep-alive`). The system prompt is cached between calls as long as the model stays resident.

```bash
# Keep model in memory to maximize cache reuse
ollama run llama3.1 --keep-alive 10m
```

```python
import ollama

# Ollama reuses prefix automatically if model is resident
response = ollama.chat(
    model='llama3.1',
    messages=[
        {'role': 'system', 'content': '...large system prompt...'},
        {'role': 'user', 'content': 'User question'}
    ]
)
```

#### Limitations
- Cache is invalidated if the model is unloaded from memory
- No built-in metrics for cache hit/miss rates in Ollama
- Disk cache in llama.cpp is specific to exact prompt match

---

### 8. SGLang

**Use with:** Llama, Qwen, DeepSeek, Mistral, and other HuggingFace models

**How it works:** SGLang uses a **RadixAttention** mechanism — an automatic radix tree-based cache that reuses KV states across requests. Generally faster cache hit rates than vLLM for highly branching prompt structures.

```python
from sglang import Engine

engine = Engine(model_path="meta-llama/Llama-3.1-8B-Instruct")

# Prefix caching is automatic — no extra flags needed
state = engine.chat([
    {"role": "system", "content": "...large system prompt..."},
    {"role": "user", "content": "Question 1"}
])
```

---

## Comparison Table — All Providers

| Provider | Type | Minimum Tokens | Cache Discount | Storage Fee | TTL |
|---|---|---|---|---|---|
| **OpenAI** | Automatic | 1,024 | 50–90% off input | None | 5 min – 24h |
| **Anthropic** | Explicit | 1,024 – 4,096 | 90% off on hit | None (write premium) | 5 min or 1 hr |
| **Gemini** | Explicit | 1,024 – 4,096 | ~75% off input | Yes ($/1M/hr) | Custom (default 1h) |
| **DeepSeek** | Automatic | None documented | 75–90% off input | None | Auto |
| **Mistral** | Not available | — | — | — | — |
| **vLLM** | Automatic | None | Compute savings | None | In-memory |
| **llama.cpp** | Manual/Auto | None | Compute savings | None | Memory resident |
| **Ollama** | Automatic | None | Compute savings | None | Keep-alive duration |
| **SGLang** | Automatic | None | Compute savings | None | In-memory |

---

## When To Use Prompt Caching

| Scenario | Recommended? |
|---|---|
| Large fixed system prompt (agent persona, instructions) | ✅ Always |
| Long reference documents queried repeatedly | ✅ Always |
| RAG context that stays the same across a session | ✅ Always |
| Multi-turn conversations with growing history | ✅ Yes — cache the stable prefix |
| Few-shot examples in a prompt template | ✅ Yes |
| Short prompts under the minimum token threshold | ❌ No benefit |
| Single one-off requests | ❌ No benefit |
| Prompts that change completely every call | ❌ No benefit |
| Very low request frequency over long periods | ⚠️ Calculate first |

---

## When NOT to Use (And Why)

- **Below minimum token threshold:** The overhead of checking/writing cache exceeds savings. No error is returned — it just doesn't cache.
- **Highly dynamic prompts:** If the prefix changes every call (e.g., timestamp injected at the top), there will be no cache hit. Place dynamic content at the **end**, not the beginning.
- **Very infrequent calls on paid APIs:** For APIs with storage fees (Gemini), if you make 1 call per day, the hourly storage cost may exceed the input token savings.
- **Short TTL mismatches:** On Anthropic, if your calls are spaced more than 5 minutes apart and you're using the default 5-min TTL, every call will be a cache write (expensive) not a read (cheap). Switch to 1-hour TTL.

---

## Best Practices (Universal)

### 1. Prompt Structure — Stable Content First
```
[CACHED — Static]
  └── System instructions
  └── Document / RAG context
  └── Few-shot examples
  └── Tool definitions

[NOT CACHED — Dynamic]
  └── Conversation history
  └── User query
  └── Session-specific data
```

### 2. Match TTL to Your Usage Pattern

| Usage Pattern | Recommended TTL |
|---|---|
| Burst of requests within minutes | 5–10 min (OpenAI default, Claude 5-min) |
| Continuous session over hours | 1 hour (Claude, Gemini) |
| Daily production workload | 24 hours (OpenAI extended) |
| Self-hosted (vLLM/Ollama) | Keep model loaded in memory |

### 3. Calculate Break-Even Before Caching (For Storage-Fee APIs)

```
# For Gemini:
storage_cost_per_call = (tokens / 1_000_000) × hourly_rate × (ttl_hours / expected_calls)
savings_per_call      = (tokens / 1_000_000) × (standard_rate - cached_rate)
cache_if              = savings_per_call > storage_cost_per_call
```

### 4. Monitor Cache Effectiveness

| Provider | How to Check Cache Hits |
|---|---|
| OpenAI | `response.usage.prompt_tokens_details.cached_tokens` |
| Anthropic | `response.usage.cache_read_input_tokens` |
| DeepSeek | `response.usage.prompt_cache_hit_tokens` |
| vLLM | Server metrics / `--collect-detailed-traces` |
| Ollama | Inspect response `eval_count` vs `prompt_eval_count` |

### 5. Keep Prefixes Byte-Identical
Even a single character difference in the cached portion will cause a cache miss. Avoid:
- Injecting current timestamps in the system prompt
- Adding random seeds or session IDs at the top
- Whitespace differences from templating logic

---

## Quick Decision Guide

```
Is your repeated content > minimum token threshold?
  └── No  → Don't bother caching; no gain
  └── Yes → Will you make multiple requests per hour?
              └── No  → Caching likely not worth it
              └── Yes → Choose your API:
                          ├── OpenAI    → Automatic, no action needed
                          ├── Claude    → Add cache_control blocks, pick TTL
                          ├── Gemini    → Create cache object, pass cache.name
                          ├── DeepSeek  → Automatic, no action needed
                          └── OSS/Local → Enable prefix caching in engine
```

---

## Summary

| Question | Answer |
|---|---|
| **Why use it?** | Avoid paying full price to re-process the same large prompt on every call |
| **How does it work?** | KV cache states for a shared token prefix are stored and reused |
| **Cheapest option?** | DeepSeek (90% off, no storage fee) or self-hosted vLLM/Ollama (free) |
| **Most control?** | Anthropic Claude (explicit blocks, choice of TTL, granular usage stats) |
| **Easiest setup?** | OpenAI or DeepSeek (fully automatic, no code changes) |
| **Storage fee?** | Only Gemini charges per-hour storage; others do not |
| **Open source?** | vLLM, SGLang, llama.cpp, Ollama — all support prefix caching for free |
