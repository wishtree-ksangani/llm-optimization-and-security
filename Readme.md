# AI Learning — Resource Index

A collection of notebooks, guides, and reference material covering applied AI concepts with real API implementations.

---

## Topics

### Prompt Caching

Learn how to reduce LLM API costs and latency by caching large, reusable prompt prefixes — covering Google Gemini, OpenAI, Anthropic Claude, DeepSeek, and open-source inference engines.

| Resource | Description | Path |
|---|---|---|
| Gemini Notebook | Hands-on implementation of prompt caching with `gemini-2.5-flash` using a travel assistant prompt | [`prompt caching/gemini.ipynb`](prompt%20caching/gemini.ipynb) |
| Gemini Caching Guide | Concise reference for Gemini-specific caching — pricing (hourly), TTL, min token limits, best practices | [`prompt caching/PROMPT_CACHING_GUIDE.md`](prompt%20caching/PROMPT_CACHING_GUIDE.md) |
| Universal Caching Guide | Generalized guide covering all major APIs (OpenAI, Claude, Gemini, DeepSeek, Mistral) and open-source engines (vLLM, Ollama, SGLang) | [`prompt caching/PROMPT_CACHING_UNIVERSAL_GUIDE.md`](prompt%20caching/PROMPT_CACHING_UNIVERSAL_GUIDE.md) |

---

## Folder Structure

```
AI/Learning/
├── Readme.md                                     ← you are here
└── prompt caching/
    ├── gemini.ipynb                              ← Gemini prompt caching notebook
    ├── PROMPT_CACHING_GUIDE.md                   ← Gemini-specific guide
    └── PROMPT_CACHING_UNIVERSAL_GUIDE.md         ← All APIs & open-source guide
```
