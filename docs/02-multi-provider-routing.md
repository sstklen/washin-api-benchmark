**Language:** English | [繁體中文](why-multi-provider-fallback.zh.md) | [日本語](why-multi-provider-fallback.ja.md)

# OpenClaw: Designing the Optimal AI API Route — 31 Providers Tested, 10 Best Paths Found

> AI agents need LLMs, search, translation, voice, and more — but no single provider is the best at everything. We tested 31 API providers across 4 rounds of exams to design the optimal routing for each task. This is the data and thinking behind every routing decision.

## Why We Did This

We're building API infrastructure for AI agents. An AI agent doesn't just call one API — it might need to search the web, read a page, translate the result, and summarize it, all in one workflow.

The question we faced: **for each of these tasks, which provider should we route to?**

The lazy answer is to pick the most popular name — or the most premium tier — for each category. But our goal was never to find the most expensive provider. It was to find **the best fit**: the provider that actually scores highest on the tasks our agents need, in the languages our users speak.

Popularity doesn't guarantee quality. Neither does a premium label. An 8-billion-parameter model beat GPT-4o-mini by 10 points in our exams. LLM-based translation matched DeepL's quality score. The providers that ranked #1 in our tests were often not the ones you'd expect.

So we decided to test everything first, then design our routing purely from the data — not from brand names, not from pricing tiers, not from assumptions.

## How We Tested: 4 Rounds, 31 Providers

We designed a 4-round exam system. Each round catches something the others can't.

| Round | The Question | What It Revealed |
|-------|-------------|-----------------|
| **P1 — Connectivity** | Is this API even alive? | 3 out of 30 providers were dead on arrival |
| **P2 — Capability** | What can it do? (tested in EN/CN/JP) | Same API scores 100 in English, 30 in Chinese |
| **P3 — Quality** | Who's the best at each task? | 8B model scores higher than GPT-4o-mini |
| **P4 — Stability** | Can it handle 100 calls in a row? | #1 quality scorer had the worst reliability |

**You need all four rounds.** If we'd stopped at P3, we would've picked DeepSeek for Chinese (highest quality score) and gotten rate-limited in production. P4 caught that.

All test scripts and raw data: **[washin-api-benchmark](https://github.com/sstklen/washin-api-benchmark)**

## The Discovery That Changed Our Routing Design

In P2, we tested every LLM with the same prompt in three languages. The result reshaped our entire approach:

```
English: "Explain quantum computing in 3 sentences"
  Groq llama-3.3-70b     100/100 ✓
  Cerebras llama3.1-8b   100/100 ✓

Chinese: "用三句話解釋量子計算"
  Groq llama-3.3-70b      30/100 ✗  ← returns English text
  Cerebras llama3.1-8b    30/100 ✗  ← returns malformed text
  Gemini 2.5 Flash       100/100 ✓  ← zero quality drop
  Mistral Small          100/100 ✓  ← zero quality drop
```

Same API, same model, same call. A 70-point quality collapse — completely invisible to monitoring because the response is a valid `200 OK` with well-formed JSON.

**This told us: we can't use the same routing for every language.** An agent serving Chinese users needs a completely different provider chain than one serving English users. We call this **language-aware routing** — and as of today, no existing API gateway (OpenRouter, Portkey, LiteLLM) does this.

## The Full LLM Ranking

Here's our P3 quality exam result — the foundation of our LLM routing:

| # | Model | Score | Speed | Reasoning | Code | CN | JP | EN |
|---|-------|-------|-------|-----------|------|----|----|-----|
| 1 | **Gemini 2.5 Flash** | **93** | 990ms | 100 | 100 | 100 | 100 | 100 |
| 1 | **xAI Grok 4.1** | **93** | 1621ms | 100 | 100 | 100 | 100 | 100 |
| 3 | Cerebras 8B | 92 | 316ms | 100 | 100 | 30 | 60 | 60 |
| 4 | DeepSeek Chat | 87 | 1046ms | 100 | 60 | 100 | 100 | 100 |
| 4 | Mistral Small | 87 | 557ms | 100 | 60 | 100 | 100 | 100 |
| 6 | Groq 70b | 83 | 306ms | 100 | 60 | 30 | 100 | 100 |
| 7 | GPT-4o-mini | 82 | 1631ms | 30 | 60 | 100 | 100 | 100 |
| 8 | Cohere R7B | 78 | 393ms | 100 | 100 | 100 | 100 | 0 |

Three things jump out:
- **Cerebras 8B (92) beats GPT-4o-mini (82)** — a fraction of the size, 5x faster, 10 points higher
- **GPT-4o-mini scores 30 on reasoning** — it got a basic math problem wrong
- **Look at the CN column** — Groq 30, Cerebras 30, everyone else 100. That's the gap we're routing around

## 4 More Findings That Shaped Our Decisions

**LLM-based translation matches dedicated translation APIs:**

| Provider | Type | Score |
|----------|------|-------|
| Groq Translate | LLM-based | 94 |
| Cerebras Translate | LLM-based | 94 |
| DeepL | Dedicated engine | 93 |

This meant we could use LLM providers as translation fallbacks — but with the same CJK caveat.

**Silent failures are real:** Some providers return `200 OK` with `{"result": null}` or empty bodies. We caught this in P4. If you only check HTTP status, you'll pass empty data to your users.

**The DeepSeek paradox:** Highest Chinese quality score in P3. Most rate-limit failures in P4. Quality and reliability are independent variables — you must test both.

**Search engines all scored 100:** Tavily, Brave, and Serper all passed quality. The differentiator was format (Tavily: AI-ready), volume (Brave: 10 results), and speed (Serper: 537ms).

---

## The 10 Routing Decisions We Made

Every routing order below comes directly from exam scores. First position = highest P3 score. Fallback = next highest. For CJK inputs, providers scoring below 50 are skipped.

### 1. LLM — The Most Complex Route

This is our most important routing decision, and the one that required the most thought.

**English routing** — all 5 providers available:
```
Gemini(93) → Mistral(87) → Cerebras(92) → Groq(83) → Cohere(78)
```

**Chinese/Japanese routing** — skip Cerebras(CN:30) and Groq(CN:30):
```
Gemini(93) → Mistral(87) → Cohere(78)
```

Why Gemini before Cerebras despite Cerebras scoring 92? Because Gemini scores 100 across all three languages. We optimize for **consistent quality across languages first**, then speed.

### 2. Search — Relevance First, Volume as Fallback

```
Tavily(Q100, 5 results, best relevance)
  → Brave Search(Q100, 10 results, widest coverage)
  → Gemini Grounding(last resort)
  → then: Groq generates AI summary of results
```

All three scored 100 on quality. We put Tavily first because its AI-ready format gives the best downstream results when feeding into an LLM.

### 3. Translation — DeepL Leads, LLM Backs Up

```
DeepL(Q93) → Groq(Q94) → Gemini(Q93) → Cerebras(Q94) → Mistral
CJK targets: skip Groq and Cerebras (same language blind spot as LLM)
```

DeepL first for professional consistency. LLM translators score equally high but inherit CJK gaps.

### 4. Web Reading — 4 Levels for 4 Types of Pages

```
Jina Reader → Firecrawl → ScraperAPI → Apify
```

| Level | Handles | Fails On |
|-------|---------|----------|
| Jina | ~70% of pages, fast | JS-heavy SPAs |
| Firecrawl | JS rendering | Some anti-bot sites |
| ScraperAPI | Broad coverage, proxy rotation | Less clean output |
| Apify | Most resilient edge cases | Slowest |

Each level catches what the previous one missed.

### 5. Embedding — Multilingual Quality Priority

```
Cohere(best multilingual vectors) → Gemini → Jina
```

### 6. Speech-to-Text — Speed vs Features

```
Deepgram Nova-2(faster) → AssemblyAI(more features)
```

### 7. Text-to-Speech — Single Provider (Our Weakest Link)

```
ElevenLabs
```

No comparable alternative in our testing. Zero fallback. We're actively evaluating new providers.

### 8. Geocoding — Coverage to Accuracy

```
Nominatim(broadest coverage) → OpenCage(best formatting) → Mapbox(most accurate)
```

### 9. News — Dedicated Index to General Search

```
NewsAPI(dedicated news index) → Web Search fallback
```

### 10. Structured Extraction — Two-Stage Pipeline

```
Web Reader(URL → clean text) → LLM(text → structured JSON per your schema)
```

Not a fallback chain — a pipeline. Web Reader uses the 4-level fallback (#4). LLM uses language-aware routing (#1).

---

## Summary: All 10 Routes at a Glance

| Task | Route | Key Design Choice |
|------|-------|-------------------|
| LLM | Gemini → Mistral → Cerebras → Groq → Cohere | CJK: skip Cerebras & Groq |
| Search | Tavily → Brave → Gemini Grounding | Relevance → Volume → Fallback |
| Translate | DeepL → Groq → Gemini → Cerebras → Mistral | CJK: skip Groq & Cerebras |
| Web Read | Jina → Firecrawl → ScraperAPI → Apify | Each level catches different pages |
| Embedding | Cohere → Gemini → Jina | Multilingual vector quality |
| STT | Deepgram → AssemblyAI | Speed → Features |
| TTS | ElevenLabs | No fallback (weakest link) |
| Geocoding | Nominatim → OpenCage → Mapbox | Coverage → Format → Accuracy |
| News | NewsAPI → Web Search | Dedicated → General |
| Extract | Reader → LLM | Pipeline, not fallback |

Notice something about this table: **the #1 pick in each category is rarely the most well-known or most premium provider.** Gemini over GPT. Tavily over Google. Cohere over OpenAI embeddings. Nominatim over Mapbox. Every first-choice decision was made by exam score, not by reputation. The result is a routing table optimized for what actually works best — not for what looks most impressive on a pitch deck.

---

## The Implementation: Language-Aware Fallback

The pattern behind all 10 routes:

```
                    ┌───────────────────────┐
   Your Request ──→ │  1. Detect language    │
                    │  2. Look up P3 scores  │
                    │     for that language   │
                    │  3. Skip providers     │
                    │     below threshold    │
                    │  4. Route to #1        │
                    │  5. On failure → #2    │
                    └───────────┬───────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
         Provider A        Provider B        Provider C
         (P3 #1 for       (P3 #2 for       (P3 #3 for
          this language)    this language)    this language)
```

```javascript
function getProviderChain(language, examScores) {
  const MIN_SCORE = 50;

  const qualified = examScores.filter(provider =>
    isCJK(language) ? provider.cjkScore >= MIN_SCORE : true
  );

  return qualified.sort((a, b) => b.score - a.score);
}
```

**The reliability math:**
- 1 provider: ~99% uptime (7+ hours down/month)
- 3 providers: ~99.97% (down ~2 minutes/month)
- 5 providers: ~99.9999%

When Groq went down for 30 minutes last month, our routing shifted to Gemini automatically. Agents saw zero downtime.

---

## 5 Things We Learned

**1. English-only benchmarks are misleading.** Test in the languages your agents actually use. The rankings reshuffle completely.

**2. Validate response bodies, not just status codes.** `200 OK` with empty data is a silent failure. Always check the shape.

**3. Quality ≠ reliability.** Run both P3 (quality) and P4 (stability). The best scorer may be the least stable.

**4. Language detection costs 0.01ms.** A regex on Unicode ranges. Prevents routing Chinese to a 30-score provider.

**5. Rerun exams monthly.** Providers change. Models update. Last month's winner might be this month's worst.

---

## Methodology & Data

- **31 providers** tested, 4 rounds, 3 languages (EN/CN/JP)
- **Tested from Tokyo** (AWS ap-northeast-1)
- **Open source** — scripts, raw data, scoring rubrics

Benchmark repo: [github.com/sstklen/washin-api-benchmark](https://github.com/sstklen/washin-api-benchmark)
Interactive report: [api.washinmura.jp/api-benchmark](https://api.washinmura.jp/api-benchmark/en/)

---

*Published by the team at [Washin Village](https://washinmura.jp) — 28 cats and dogs, one API infrastructure team, monthly benchmarks from Boso Peninsula, Japan.*
