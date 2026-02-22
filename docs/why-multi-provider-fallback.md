# We Tested 31 AI APIs So You Don't Have To — Here's What We'd Pick for Every Task

> We run AI agent infrastructure from a small server in Tokyo. Before building anything, we put 31 API providers through 4 rounds of standardized exams — same prompts, same scoring, 3 languages. This article shares our complete findings and the routing decisions we derived from the data.

---

## The Starting Point

When you're building infrastructure that AI agents depend on, you face a question for every service category: **which provider do you use?**

The obvious answer is to go with the biggest name. OpenAI for LLM. Google for search. DeepL for translation. But we had a nagging suspicion that "biggest" and "best" might not be the same thing — especially for non-English languages.

So before writing a single line of routing code, we designed an exam system and ran every provider through it.

## The Exam System: P1 Through P4

We didn't invent anything fancy. We just applied the simplest idea that works: **standardized testing**. Same test for everyone. Score what matters. Run it multiple times.

### P1 — Connectivity: Are You Even Alive?

The most basic question. Can we reach the API? Does it respond? How fast?

**What we found:** 3 out of 30 providers failed this exam completely. Dead endpoints, expired certificates, DNS failures. They advertise API access but nobody's home. We eliminated them immediately.

**Takeaway:** Never assume an API works because the docs page exists. Ping it first.

### P2 — Capability: What Can You Actually Do?

Now it gets interesting. We sent every LLM the same set of tasks:
- Reasoning: a math word problem requiring multi-step logic
- Code generation: write a function, we run it
- Multilingual: answer the same question in English, Chinese, and Japanese

This is where the first major surprise hit us.

```
English: "Explain quantum computing in 3 sentences"
  Groq llama-3.3-70b:    100/100
  Cerebras llama3.1-8b:   100/100

Chinese: "用三句話解釋量子計算"
  Groq llama-3.3-70b:     30/100  ← returned English
  Cerebras llama3.1-8b:   30/100  ← returned garbage

  Gemini 2.5 Flash:       100/100 ← no quality drop
  Mistral Small:          100/100 ← no quality drop
```

**Same API. Same model. Same call. The only variable is the input language. A 70-point quality collapse.**

This isn't a minor issue. If your agent serves Chinese or Japanese users, Groq and Cerebras will silently return English or malformed text. The HTTP response is a valid 200 with well-formed JSON. Your monitoring sees nothing wrong. Your users see garbage.

**Takeaway:** Always test in the languages your users actually speak. English-only benchmarks hide critical quality gaps.

### P3 — Quality Ranking: Who's Actually the Best?

We ranked every provider within each service category. Here's the LLM ranking that surprised us the most:

| # | Model | Overall | Speed | Reasoning | Code | CN/JP/EN |
|---|-------|---------|-------|-----------|------|----------|
| 1 | Gemini 2.5 Flash | **93** | 990ms | 100 | 100 | 100/100/100 |
| 2 | xAI Grok 4.1 | **93** | 1621ms | 100 | 100 | 100/100/100 |
| 3 | Cerebras 8B | **92** | 316ms | 100 | 100 | 30/60/60 |
| 4 | DeepSeek Chat | **87** | 1046ms | 100 | 60 | 100/100/100 |
| 5 | Mistral Small | **87** | 557ms | 100 | 60 | 100/100/100 |
| 6 | Groq 70b | **83** | 306ms | 100 | 60 | 30/100/100 |
| 7 | GPT-4o-mini | **82** | 1631ms | 30 | 60 | 100/100/100 |

Look at that table carefully:
- **Cerebras 8B (8 billion parameters) scores 92. GPT-4o-mini scores 82.** A model that's a fraction of the size, runs 5x faster, and scores 10 points higher.
- **GPT-4o-mini gets 30/100 on reasoning.** We asked it a math word problem. It got the arithmetic wrong.
- **Gemini 2.5 Flash tops the chart** with perfect scores across all three languages. Not GPT. Not Claude. Gemini.

**Takeaway:** Run your own benchmarks. The marketing hierarchy (OpenAI > Google > everyone else) does not match the data.

### P4 — Stability: Can You Handle 100 Calls in a Row?

Quality means nothing if the provider drops requests under load. We fired 100 sequential calls at each provider and tracked failures.

**The DeepSeek paradox:** DeepSeek scored highest on Chinese quality in P3. In P4, it had the most rate-limit failures. The best quality provider was the least reliable one.

**Silent failures:** Some providers return `200 OK` with an empty body or `{"result": null}`. From your code's perspective, the call succeeded. But there's no data. If you don't validate the response body's shape, you'll pass empty results to your users and never know.

**Takeaway:** Quality and stability are independent variables. You must test both. A provider can ace quality and fail stability, or vice versa.

---

## From Exam Data to Routing Decisions

With P1–P4 results in hand, we had enough data to make informed decisions. For each of 10 service categories, we asked:

1. **Who scored highest?** → Goes first
2. **Who scored second?** → Fallback #1
3. **Who has language blind spots?** → Skip for CJK inputs
4. **Who failed stability?** → Demote or remove

Here's every routing decision we made, and the data behind it.

### LLM: 5 Providers, Language-Aware

**The problem:** You need an LLM that's fast, accurate, and works in multiple languages. No single provider does all three perfectly.

**What the data told us:**
- Gemini: Highest overall (93), perfect multilingual, moderate speed
- Mistral: Strong all-rounder (87), perfect multilingual, fast
- Cerebras: Incredible speed (316ms) and high score (92), but Chinese collapses to 30
- Groq: Fastest (306ms), but Chinese collapses to 30
- Cohere: Decent multilingual, weakest overall

**Our routing:**
```
English requests:         Gemini → Mistral → Cerebras → Groq → Cohere
Chinese/Japanese requests: Gemini → Mistral → Cohere
```

For CJK languages, we skip Cerebras and Groq entirely. Their Chinese scores (30/100) are below our threshold. This is a routing decision that no existing API gateway makes automatically — they all treat providers as language-agnostic. The data says they're not.

### Search: 3 Engines

**What the data told us:**
- Tavily: Perfect score (100), 5 results per query, AI-ready format, best relevance
- Brave Search: Perfect score (100), 10 results per query, broadest coverage
- Serper (Google): Perfect score (100), 8 results, fastest (537ms)

All three scored 100 on quality. The differentiator is format and coverage.

**Our routing:**
```
Tavily → Brave Search → Gemini Grounding
```
Tavily first for relevance quality. Brave as fallback for volume. Gemini Grounding as emergency last resort. After retrieving results, Groq generates an AI summary.

### Translation: 5 Engines

**What the data told us:**
- DeepL: Score 93, most consistent across language pairs, professional-grade
- Groq Translate: Score 94, surprisingly strong — LLM-based translation matches dedicated translation APIs
- Cerebras Translate: Score 94, fastest (335ms)
- Gemini Translate: Score 93, reliable across all languages
- Mistral Translate: Decent quality, good fallback

A revelation from P3: **LLM-based translation (Groq, Cerebras) scored equal to or higher than DeepL.** But they inherit the same CJK blind spots as their LLM counterparts.

**Our routing:**
```
DeepL → Groq → Gemini → Cerebras → Mistral
CJK target languages: skip Groq and Cerebras (same language gap as LLM)
```

### Web Reading: 4 Levels

Converting URLs to clean, structured text. Each provider handles a different slice of the web:

| Provider | Handles Well | Fails On |
|----------|-------------|----------|
| Jina Reader | Most static pages, fast | JavaScript-heavy SPAs |
| Firecrawl | JS rendering, dynamic content | Some anti-bot sites |
| ScraperAPI | Broad coverage, proxy rotation | Less clean output |
| Apify | Most resilient, handles edge cases | Slowest |

**Our routing:** `Jina → Firecrawl → ScraperAPI → Apify`

Each level catches what the previous one missed. Jina handles 70%+ of pages. The remaining 30% cascade through increasingly robust (but slower) alternatives.

### Embedding: 3 Engines

For text-to-vector conversion (RAG pipelines, semantic search):

**Our routing:** `Cohere → Gemini → Jina`

Cohere Embed has the strongest multilingual embedding quality in our testing. Gemini and Jina as fallback.

### Speech-to-Text: 2 Engines

**Our routing:** `Deepgram Nova-2 → AssemblyAI`

Both are strong. Deepgram is faster; AssemblyAI has more features. Either one failing triggers the other.

### Text-to-Speech: 1 Engine

**Our routing:** `ElevenLabs`

Single provider. In our testing, no alternative matched ElevenLabs' output quality for English and Japanese. This is our weakest link — no fallback. We're actively testing alternatives.

### Geocoding: 3 Levels

**Our routing:** `Nominatim → OpenCage → Mapbox`

Ordered by coverage breadth → formatting quality → accuracy. Three levels of fallback for address-to-coordinate conversion.

### News: 2 Engines

**Our routing:** `NewsAPI → Web Search fallback`

NewsAPI has a dedicated news index. When it fails, we fall back to general web search filtered for recent results.

### Structured Extraction: Pipeline

This isn't a fallback chain — it's a two-stage pipeline:
```
Web Reader (URL → clean text) → LLM (text → structured JSON matching your schema)
```
The Web Reader stage uses the 4-level fallback above. The LLM stage uses the language-aware LLM routing.

---

## The Pattern: Exam-Driven, Language-Aware Routing

Across all 10 services, the same pattern emerges:

```
                    ┌──────────────────────┐
   Your Request ──→ │  1. Detect language   │
                    │  2. Look up exam      │
                    │     scores for this   │
                    │     language           │
                    │  3. Route to highest  │
                    │     scoring provider  │
                    │  4. On failure, try   │
                    │     next highest      │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
         Provider A       Provider B       Provider C
         (exam #1)        (exam #2)        (exam #3)
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

**The uptime effect:**
- 1 provider: ~99% (down 7+ hours/month)
- 3 providers: ~99.97% (down ~2 minutes/month)
- 5 providers: ~99.9999% (essentially never)

This isn't theoretical. When Groq went down for 30 minutes last month, our LLM routing automatically shifted to Gemini. Users saw zero downtime.

---

## Summary Table: All Routing Decisions

| Service | Chain | Why This Order |
|---------|-------|---------------|
| LLM | Gemini → Mistral → Cerebras → Groq → Cohere | P3 score: 93 → 87 → 92* → 83 → 78 (*CJK skip) |
| Search | Tavily → Brave → Gemini Grounding | Relevance → Volume → Last resort |
| Translate | DeepL → Groq → Gemini → Cerebras → Mistral | P3 score: 93 → 94* → 93 → 94* → 87 (*CJK skip) |
| Web Read | Jina → Firecrawl → ScraperAPI → Apify | Speed → JS render → Coverage → Resilience |
| Embedding | Cohere → Gemini → Jina | Multilingual quality ranking |
| STT | Deepgram → AssemblyAI | Speed → Features |
| TTS | ElevenLabs | No comparable alternative |
| Geocoding | Nominatim → OpenCage → Mapbox | Coverage → Format → Accuracy |
| News | NewsAPI → Web Search | Dedicated index → General fallback |
| Extract | Web Reader → LLM | Pipeline (not fallback) |

---

## 5 Things We Learned the Hard Way

**1. English benchmarks hide the real quality gaps.**
Every public LLM benchmark is English-only. When we added Chinese and Japanese, the rankings reshuffled completely. If your users speak anything other than English, published benchmarks are misleading.

**2. 200 OK doesn't mean it worked.**
Multiple providers return successful HTTP status codes with empty or null response bodies. If you only check `response.ok`, you'll pass silent failures to your users. Always validate the response shape.

**3. The best-quality provider may be the least reliable.**
DeepSeek: #1 on Chinese quality, worst on stability. You can't optimize for one metric. You need both P3 (quality) and P4 (stability) data to make a real decision.

**4. Language detection takes 0.01ms and prevents garbage output.**
A regex checking Unicode ranges. Nearly zero overhead. It's the difference between routing Chinese to a provider that scores 93 vs one that scores 30. There's no reason not to do this.

**5. Run your own exams. Every month.**
Provider quality changes. Models get updated. Rate limits shift. Our exam system runs monthly. Last month's best provider might be this month's worst. Stale data is dangerous data.

---

## Methodology

- **31 providers** tested across 4 rounds (P1–P4)
- **3 languages** per test: English, Chinese, Japanese
- **Location:** Tokyo, Japan (AWS ap-northeast-1)
- **Scoring:** Reasoning = mathematical correctness, Code = executable output, Multilingual = response accuracy per language
- **Open source:** All scripts, raw data, and scoring rubrics

Full benchmark data: [github.com/sstklen/washin-api-benchmark](https://github.com/sstklen/washin-api-benchmark)
Interactive report: [washinmura.jp/api-benchmark](https://api.washinmura.jp/api-benchmark/en/)

---

*Published by the engineering team at [Washin Village](https://washinmura.jp) — an animal sanctuary in Boso Peninsula, Japan, where 28 cats and dogs share a roof with a small API infrastructure team. We run these benchmarks monthly and publish all results openly.*
