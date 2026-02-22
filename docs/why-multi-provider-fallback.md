# We Tested 31 AI APIs in 3 Languages — Here's the Best Provider for Every Task

> Every routing decision below comes from real benchmark data: 4 rounds of standardized exams, 31 providers, tested from Tokyo. No guessing, no marketing claims — just numbers.

## Why We Did This

We needed to build reliable AI agent infrastructure. The problem: there are dozens of LLM, search, translation, and voice APIs — and none of them publish multilingual quality benchmarks. Every provider says they're "the best." Nobody shows the data.

So we tested all of them. Same prompts, same scoring rubric, 3 languages (English, Chinese, Japanese), 4 rounds each. The full methodology and raw data are open source: **[washin-api-benchmark](https://github.com/sstklen/washin-api-benchmark)**

What we found changed how we think about provider selection.

## The Findings That Surprised Us

### Finding 1: The Same API Scores 100 in English and 30 in Chinese

```
English prompt: "Explain quantum computing"
  Groq llama-3.3-70b:   100/100 ✓
  Cerebras llama3.1-8b:  100/100 ✓

Chinese prompt: "解釋量子計算"
  Groq llama-3.3-70b:    30/100 ✗  ← Returns English or garbage
  Cerebras llama3.1-8b:  30/100 ✗  ← Returns English or garbage
  Gemini 2.5 Flash:      93/100 ✓  ← No quality drop
  Mistral Small:         90/100 ✓  ← No quality drop
```

A 70-point quality collapse. Same API, same model, same call. The only difference is the input language. Your monitoring won't catch this — the HTTP response is a valid 200 with well-formed JSON.

### Finding 2: The Highest-Scoring Chinese LLM Is the Least Stable

DeepSeek Chat scored 87 overall with 100/100 on Chinese — the best multilingual performance in our P3 quality exam. But in P4 stability testing (100 sequential calls), it had the highest rate-limit failure rate.

**Quality and reliability are independent variables.** You can't pick a provider based on one metric alone.

### Finding 3: A 8B Parameter Model Outscores GPT-4o-mini

Cerebras llama3.1-8b scored **92** overall. GPT-4o-mini scored **82**. The 8B model is faster (316ms vs 1631ms) and scores 10 points higher.

### Finding 4: Some Providers Return 200 OK with Empty Bodies

In P4 stability testing, we discovered providers that return `200 OK` with `{"result": null}` or empty response bodies. Your agent gets a success status code and treats it as valid. The data is garbage or missing. **Always validate response shape, not just status code.**

## How We Tested: 4-Round Exam System

| Exam | Purpose | Key Finding |
|------|---------|-------------|
| **P1 — Connectivity** | Can we reach it? How fast? | 3 out of 30 providers were completely unreachable |
| **P2 — Capability** | What can it actually do? | Groq: EN 100, CN 30. Same API, 70-point gap |
| **P3 — Quality** | Who's best at each task? | Gemini 2.5 Flash beat GPT-4o-mini by 11 points |
| **P4 — Stability** | 100 calls — who drops? | Silent failures (200 OK, empty body) detected |

## Best Provider for Each Task — Based on Exam Data

Every routing order below is derived from P1–P4 exam scores. First in line = highest quality score. If it fails, the next one takes over automatically.

### LLM — Who Gives the Best Answer?

**P3 exam results (overall score / multilingual):**

| Provider | Score | Speed | CN | JP | EN | Why This Ranking |
|----------|-------|-------|----|----|-----|------------------|
| Gemini 2.5 Flash | 93 | 990ms | 100 | 100 | 100 | Highest score + perfect multilingual |
| Mistral Small | 87 | 557ms | 100 | 100 | 100 | Strong multilingual, fast |
| Cerebras 8B | 92 | 316ms | 30 | 60 | 60 | High score but weak CJK |
| Groq 70b | 83 | 306ms | 30 | 100 | 100 | Fastest, but Chinese collapses |
| Cohere R7B | 78 | 393ms | 100 | 100 | 0 | Good CJK, weak English |

**Our routing based on this data:**
```
English:                Gemini → Mistral → Cerebras → Groq → Cohere
Chinese/Japanese:       Gemini → Mistral → Cohere  (skip Cerebras & Groq — CN score 30)
```

**Key insight:** You need different routing chains for different languages. No existing API gateway (OpenRouter, Portkey, LiteLLM) does this automatically.

### Search — Who Finds the Best Results?

**P2/P3 exam results:**

| Provider | Score | Speed | Results/Query | Strength |
|----------|-------|-------|---------------|----------|
| Tavily | 100 | 1536ms | 5 | AI-ready format, high relevance |
| Brave Search | 100 | 1124ms | 10 | Volume, broad coverage |
| Gemini Grounding | — | — | — | Built-in to Gemini, last resort |

**Our routing:**
```
Tavily → Brave Search → Gemini Grounding → Groq AI summary
```
Tavily first for quality. Brave as fallback for coverage. Gemini Grounding as last resort. Then Groq summarizes.

### Translation — Who Translates Best?

**P3 exam results:**

| Provider | Score | Speed | Strength |
|----------|-------|-------|----------|
| DeepL | 93 | 641ms | Professional quality, most language pairs |
| Groq Translate | 94 | 526ms | Surprisingly high quality |
| Cerebras Translate | 94 | 335ms | Fastest |
| Gemini Translate | 93 | ~1s | Reliable multilingual |

**Our routing:**
```
DeepL → Groq → Gemini → Cerebras → Mistral
CJK targets: skip Groq/Cerebras (poor CJK output quality, same as LLM finding)
```
DeepL first for consistent professional quality. LLM-based translation as fallback — they score equally high but have the same CJK blind spot as their LLM counterparts.

### Web Reading — Who Extracts Clean Text?

| Provider | Strength | Weakness |
|----------|----------|----------|
| Jina Reader | Fast, clean Markdown | Fails on some JS-heavy sites |
| Firecrawl | Handles JS rendering | Slower |
| ScraperAPI | Good coverage | Less clean output |
| Apify | Most resilient | Slowest |

**Our routing:**
```
Jina Reader → Firecrawl → ScraperAPI → Apify
```
Ordered by speed and output quality. Each level handles cases the previous one can't.

### Embedding — Who Makes the Best Vectors?

| Provider | Strength |
|----------|----------|
| Cohere Embed | Best multilingual embedding quality |
| Gemini Embed | Good quality, fast |
| Jina Embed | Lightweight, reliable |

**Our routing:** `Cohere → Gemini → Jina`

### Voice: Speech-to-Text

| Provider | Strength |
|----------|----------|
| Deepgram Nova-2 | Fast, accurate, good multilingual |
| AssemblyAI | Strong accuracy, more features |

**Our routing:** `Deepgram → AssemblyAI`

### Voice: Text-to-Speech

| Provider | Strength |
|----------|----------|
| ElevenLabs | Best quality across EN/JP |

**Our routing:** `ElevenLabs` (single provider — no comparable alternative at this quality level)

### Geocoding

| Provider | Strength |
|----------|----------|
| Nominatim | Wide coverage, open data |
| OpenCage | Good formatting, reliable |
| Mapbox | Most accurate, premium |

**Our routing:** `Nominatim → OpenCage → Mapbox`

### News Search

| Provider | Strength |
|----------|----------|
| NewsAPI | Dedicated news index, fast |
| Web Search fallback | Broader but less structured |

**Our routing:** `NewsAPI → Web Search fallback`

### Structured Extraction

Pipeline, not fallback:
```
Web Reader (URL → clean text) → LLM (text → structured JSON per your schema)
```

## Summary: All 10 Routing Tables

| Service | Providers | Routing Order (by exam score) |
|---------|-----------|-------------------------------|
| LLM | 5 | Gemini → Mistral → Cerebras → Groq → Cohere |
| Search | 3 | Tavily → Brave → Gemini Grounding |
| Translate | 5 | DeepL → Groq → Gemini → Cerebras → Mistral |
| Web Read | 4 | Jina → Firecrawl → ScraperAPI → Apify |
| Embedding | 3 | Cohere → Gemini → Jina |
| STT | 2 | Deepgram → AssemblyAI |
| TTS | 1 | ElevenLabs |
| Geocoding | 3 | Nominatim → OpenCage → Mapbox |
| News | 2 | NewsAPI → Web Search |
| Extract | 2 | Web Reader + LLM pipeline |

**Every order above is data-driven.** First in line = highest exam score. Fallback = next highest. Language-aware services skip providers with CJK scores below 50.

## The Architecture Pattern: Language-Aware Fallback

The routing tables above are powered by a simple architecture:

```
                    ┌─────────────────┐
   Your Request ──→ │  Smart Gateway   │
                    │                  │
                    │  1. Detect lang  │
                    │  2. Filter by    │
                    │     exam score   │
                    │  3. Try best     │
                    │  4. Fallback     │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
         Provider A     Provider B     Provider C
         (highest       (2nd score)    (3rd score)
          exam score)
```

```javascript
// Simplified language-aware routing
function getProviderChain(language, examScores) {
  const minScore = 50;

  // Filter out providers that scored below threshold for this language
  const qualified = examScores.filter(p =>
    isCJK(language) ? p.cjkScore >= minScore : true
  );

  // Sort by score descending — best quality first
  return qualified.sort((a, b) => b.score - a.score);
}
```

**The uptime effect of multi-provider fallback:**
- 1 provider: ~99% uptime (7+ hours down/month)
- 3 providers: ~99.97% (2 minutes down/month)
- 5 providers: ~99.9999% (essentially never down)

## Bonus: Intent Routing Layer

On top of the per-service routing, we added an LLM-based intent router that parses natural language requests and dispatches to the appropriate service:

```
Input: "What's the weather in Tokyo and translate it to Japanese"

  → Intent analysis: weather + translate
  → Parallel execution:
      weather(lat=35.68, lon=139.69)
      translate(result, target=ja)
  → Combined response in ~2 seconds
```

This means the caller doesn't need to know which service to use — the router picks based on intent, then each service picks the best provider based on exam data.

## 5 Lessons from 31-Provider Testing

### 1. Test before you trust
DeepSeek is a Chinese company. We assumed it would ace Chinese. It scored #1 on quality — but had the worst stability under load. **Assumptions based on company origin are unreliable.**

### 2. Validate response shape, not just status code
`200 OK` with `{"result": null}` is a silent failure. We caught multiple providers doing this. **Parse and validate every response body.**

### 3. Quality-first fallback, not random load balancing
We tried random distribution first. Quality became unpredictable. **Always route to the highest-scoring provider first.** Fallback is for failures, not for spreading load.

### 4. Language detection takes 0.01ms and prevents garbage output
A regex + Unicode range check. Nearly zero overhead. Prevents routing Chinese prompts to providers that score 30/100 on Chinese. **Always detect language before selecting a provider.**

### 5. Don't assume — measure
An 8B model beat GPT-4o-mini by 10 points. LLM-based translation scored equal to DeepL. Established reputation does not predict benchmark performance. **Run standardized exams on every provider you consider.**

---

## Methodology & Data

- **31 providers** tested across 4 exam rounds (P1–P4)
- **3 languages** per test (English, Chinese, Japanese)
- **All tests from Tokyo** (AWS ap-northeast-1)
- **Open source** — scripts, raw data, scoring rubric

Full benchmark data: [github.com/sstklen/washin-api-benchmark](https://github.com/sstklen/washin-api-benchmark)
Interactive report: [api.washinmura.jp/api-benchmark](https://api.washinmura.jp/api-benchmark/en/)

---

*Published by [Washin Village](https://washinmura.jp) — an animal sanctuary in Boso Peninsula, Japan, where 28 cats and dogs share a roof with an API engineering team.*
