# Same API, Same Model, Same Call: 100 in English, 30 in Chinese

> We benchmarked 31 AI API providers in 3 languages across 4 rounds of standardized tests. The biggest finding: **LLM quality is language-dependent, and nobody is measuring it.**

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

**Same model. Same endpoint. Same HTTP call. A 70-point quality collapse — invisible to your monitoring because the response is a valid `200 OK` with well-formed JSON.**

If your AI agent serves Chinese, Japanese, or Korean users, you have a silent quality problem right now. No existing API gateway (OpenRouter, Portkey, LiteLLM) tests for this or routes around it.

---

## 4 More Findings That Broke Our Assumptions

### An 8B model outscores GPT-4o-mini

| Model | Parameters | Score | Speed | Reasoning |
|-------|-----------|-------|-------|-----------|
| Cerebras llama3.1-8b | 8B | **92** | 316ms | 100/100 |
| GPT-4o-mini | undisclosed | **82** | 1631ms | 30/100 |

Cerebras is smaller, 5x faster, and scores 10 points higher. GPT-4o-mini failed our reasoning test — a multi-step math problem that requires basic arithmetic.

### The best Chinese LLM is the least stable

DeepSeek Chat scored #1 on Chinese quality (100/100). In our stability test (100 sequential calls), it had the highest rate-limit failure rate of all providers. **Quality and reliability are independent variables.**

### `200 OK` doesn't mean it worked

Multiple providers return HTTP 200 with `{"result": null}` or empty response bodies. Your code checks `response.ok`, sees `true`, and passes garbage downstream. We caught this by validating response shape in our P4 stability exam — not something most teams test for.

### LLM-based translation matches DeepL

| Provider | Type | Score |
|----------|------|-------|
| Groq Translate | LLM-based | 94 |
| Cerebras Translate | LLM-based | 94 |
| DeepL | Dedicated engine | 93 |

LLM translation scored equal to or higher than a dedicated translation API. But LLM translators inherit the same CJK blind spots — Groq and Cerebras produce poor Chinese/Japanese translations, just like their LLM outputs.

---

## The Full LLM Ranking (P3 Quality Exam)

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

**Read the CN column.** Groq: 30. Cerebras: 30. Everyone else: 100. That's the gap nobody talks about.

---

## How We Tested: A 4-Round Exam System

| Round | What It Tests | What It Catches |
|-------|--------------|-----------------|
| **P1 — Connectivity** | Is the API alive? Response time? | 3/30 providers were completely unreachable |
| **P2 — Capability** | What can it do? Test in 3 languages | The 70-point CJK quality gap |
| **P3 — Quality** | Rank all providers per category | Gemini #1, 8B > GPT, LLM translation ≈ DeepL |
| **P4 — Stability** | 100 sequential calls per provider | Silent failures (200 OK, empty body), DeepSeek rate limits |

Each round filters what the previous one can't. P1 eliminates dead providers. P2 maps capabilities. P3 ranks quality. P4 tests reliability under sustained load. **You need all four rounds.** Skipping P4 means you'll pick DeepSeek for Chinese (P3 winner) and get rate-limited in production.

All scripts, raw data, and scoring methodology: **[washin-api-benchmark](https://github.com/sstklen/washin-api-benchmark)**

---

## What We Built From the Data: Routing Tables for 10 Service Categories

Every routing order below is derived from exam scores. #1 = highest P3 score. Fallback = next highest. Language-aware services skip providers with CJK scores below 50.

### LLM — Language-Aware Routing

```
English:         Gemini(93) → Mistral(87) → Cerebras(92) → Groq(83) → Cohere(78)
Chinese/Japanese: Gemini(93) → Mistral(87) → Cohere(78)
                                              ↑ skip Cerebras(CN:30) and Groq(CN:30)
```
Why Gemini before Cerebras despite lower speed? Because Gemini scores 100 across all 3 languages. Cerebras scores 30 on Chinese. **We optimize for quality first, speed second.**

### Search — 3 Engines

```
Tavily(Q100, 5 results, best relevance)
  → Brave Search(Q100, 10 results, best volume)
  → Gemini Grounding(last resort)
  → Groq AI summary on results
```

### Translation — 5 Engines, Language-Aware

```
DeepL(Q93, professional) → Groq(Q94) → Gemini(Q93) → Cerebras(Q94) → Mistral
CJK targets: skip Groq and Cerebras (same CJK blind spot as LLM)
```

### Web Reading — 4 Levels of Resilience

```
Jina Reader(fastest, handles ~70% of pages)
  → Firecrawl(JS rendering)
  → ScraperAPI(proxy rotation, broad coverage)
  → Apify(most resilient, slowest)
```

### Embedding — 3 Engines
`Cohere(best multilingual) → Gemini → Jina`

### Speech-to-Text — 2 Engines
`Deepgram Nova-2(faster) → AssemblyAI(more features)`

### Text-to-Speech — 1 Engine
`ElevenLabs` — No comparable alternative in our testing. This is our weakest link: zero fallback.

### Geocoding — 3 Levels
`Nominatim(broadest) → OpenCage(best formatting) → Mapbox(most accurate)`

### News — 2 Engines
`NewsAPI(dedicated news index) → Web Search fallback`

### Structured Extraction — Pipeline
`Web Reader(URL → text) → LLM(text → structured JSON per schema)`

### Summary

| Service | Providers | Routing Order | Key Factor |
|---------|-----------|---------------|------------|
| LLM | 5 | Gemini → Mistral → Cerebras → Groq → Cohere | Language-aware: CJK skips 2 |
| Search | 3 | Tavily → Brave → Gemini Grounding | Relevance → Volume → Fallback |
| Translate | 5 | DeepL → Groq → Gemini → Cerebras → Mistral | CJK-aware: skips 2 |
| Web Read | 4 | Jina → Firecrawl → ScraperAPI → Apify | Speed → JS → Coverage → Resilience |
| Embedding | 3 | Cohere → Gemini → Jina | Multilingual quality |
| STT | 2 | Deepgram → AssemblyAI | Speed → Features |
| TTS | 1 | ElevenLabs | No fallback (weakest link) |
| Geocoding | 3 | Nominatim → OpenCage → Mapbox | Coverage → Format → Accuracy |
| News | 2 | NewsAPI → Web Search | Dedicated → General |
| Extract | 2 | Reader → LLM | Pipeline, not fallback |

---

## The Architecture Pattern: Language-Aware Fallback

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
         (P3 rank #1       (P3 rank #2       (P3 rank #3
          for this lang)    for this lang)    for this lang)
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

**Uptime math:**
- 1 provider: ~99% (down 7+ hours/month)
- 3 providers with fallback: ~99.97% (down ~2 minutes/month)
- 5 providers with fallback: ~99.9999%

---

## 5 Lessons

**1. English-only benchmarks are misleading.** Add Chinese or Japanese to your test suite and watch the rankings reshuffle. Published benchmarks won't tell you this.

**2. Validate response bodies, not just status codes.** `200 OK` with `null` data is a silent failure. Parse and validate every response.

**3. Quality ≠ reliability.** Test both independently (P3 + P4). The best scorer may be the least stable.

**4. Language detection takes 0.01ms.** A regex on Unicode ranges. Nearly zero overhead. Prevents routing Chinese to a provider that scores 30.

**5. Rerun monthly.** Provider quality changes. Models get updated. Rate limits shift. Stale benchmark data is dangerous data.

---

## Methodology

- **31 providers**, 4 exam rounds, 3 languages (EN/CN/JP)
- **Tested from Tokyo** (AWS ap-northeast-1)
- **Open source** — all scripts, raw data, scoring rubrics
- **No sponsors** — rankings are purely data-driven

Benchmark repo: [github.com/sstklen/washin-api-benchmark](https://github.com/sstklen/washin-api-benchmark)
Interactive report: [api.washinmura.jp/api-benchmark](https://api.washinmura.jp/api-benchmark/en/)

---

*Published by the team at [Washin Village](https://washinmura.jp) — 28 cats and dogs, one API infrastructure team, monthly benchmarks from Boso Peninsula, Japan.*
