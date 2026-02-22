# Why We Built a Multi-Provider Fallback Gateway (And What We Learned)

> We tested 31 API providers over 4 rounds of exams. Here's what broke, what surprised us, and how we built a 99.97% uptime gateway from unreliable parts.

## The Problem

Every AI API will fail. Not "might fail" — **will fail**.

We learned this the hard way. We run an API marketplace for AI agents, and our agents were going down every time a single provider had issues. OpenAI goes down? Agent dead. Brave Search rate-limited? Agent dead.

**The math is simple:**

- Single provider uptime: ~99% (down 7+ hours/month)
- 3 providers with fallback: ~99.97% (down 2 minutes/month)
- 5 providers with fallback: ~99.9999% (basically never)

## What We Did: Test Everything First

Before building the gateway, we ran **4 rounds of standardized exams** on 31 providers.
All test scripts, raw results, and scoring methodology are open source:
**[washin-api-benchmark](https://github.com/sstklen/washin-api-benchmark)**

| Exam                     | What We Tested                  | What We Found                                     |
| ------------------------ | ------------------------------- | ------------------------------------------------- |
| **P1 — Connectivity**    | Can we reach it? Response time? | 3 providers were dead on arrival                  |
| **P2 — Capability**      | What can it actually do?        | Groq scores 30/100 on Chinese, 100/100 on English |
| **P3 — Quality Ranking** | Who's the best at each task?    | Gemini beat GPT-4o on multilingual tasks          |
| **P4 — Stability**       | 100 calls in a row — who drops? | Some providers fail silently (200 OK, empty body) |

### The Biggest Surprise: Language Matters

We discovered that **LLM quality varies wildly by language**:

```
English prompt: "Explain quantum computing"
  Groq:     100/100 ✓ Great
  Cerebras: 100/100 ✓ Great

Chinese prompt: "解釋量子計算"
  Groq:     30/100  ✗ Returns English or garbage
  Cerebras: 30/100  ✗ Returns English or garbage
  Gemini:   93/100  ✓ Perfect Chinese
  Mistral:  90/100  ✓ Good Chinese
```

**No API gateway on the market handles this.** OpenRouter, Portkey, LiteLLM — they all let you pick a provider, but none of them automatically skip providers that can't handle your language.

## Architecture: Language-Aware Fallback Routing

Here's what we built:

```
                    ┌─────────────────┐
   Your Request ──→ │  Smart Gateway   │
                    │                  │
                    │  1. Detect lang  │
                    │  2. Filter bad   │
                    │  3. Try best     │
                    │  4. Fallback     │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
         Provider A     Provider B     Provider C
         (best for      (fallback)     (last resort)
          this lang)
```

### Example: Smart LLM Routing

```javascript
// Simplified routing logic
function getProviderChain(language, strategy) {
  const allProviders = [
    { name: 'gemini',   cjkScore: 93, enScore: 93 },
    { name: 'mistral',  cjkScore: 90, enScore: 90 },
    { name: 'cerebras', cjkScore: 30, enScore: 88 },
    { name: 'groq',     cjkScore: 30, enScore: 80 },
    { name: 'cohere',   cjkScore: 55, enScore: 55 },
  ];

  // Skip providers that score < 50 for detected language
  const qualified = allProviders.filter(p =>
    isCJK(language) ? p.cjkScore >= 50 : true
  );

  return qualified; // Sorted by score, tried in order
}
```

**Result:** Chinese/Japanese requests automatically skip Groq and Cerebras. English requests use all 5 providers. The user doesn't need to know or care.

## 10 Services, Same Pattern

We applied this pattern across 10 different service types:

| Service   | Providers | Fallback Chain                              |
| --------- | --------- | ------------------------------------------- |
| Search    | 3         | Tavily → Brave → Gemini Grounding           |
| LLM       | 5         | Gemini → Mistral → Cerebras → Groq → Cohere |
| Translate | 5         | DeepL → Groq → Gemini → Cerebras → Mistral  |
| Web Read  | 4         | Jina → Firecrawl → ScraperAPI → Apify       |
| Embedding | 3         | Cohere → Gemini → Jina                      |
| STT       | 2         | Deepgram → AssemblyAI                       |
| TTS       | 1         | ElevenLabs                                  |
| Geocoding | 3         | Free API → OpenCage → Mapbox                |
| News      | 2         | NewsAPI → Smart Search                      |
| Extract   | 2         | Smart Read + LLM pipeline                   |

## The AI Concierge Layer

On top of the gateway, we built an **intent router** — an LLM that reads natural language and picks which tools to call:

```
Human: "What's the weather in Tokyo and translate it to Japanese"

Concierge AI:
  → Intent: weather + translate
  → Execute: weather(lat=35.68, lon=139.69) || translate(text, ja)
  → Synthesize: Combined result in natural language
  → Cost: $0.02 total
```

**Why this matters for AI agents:**
Your agent doesn't need to know our API schema. It just sends a sentence. The Concierge figures out the rest — which tools, what parameters, parallel vs sequential execution, result formatting.

## Lessons Learned

### 1. Test before you trust

We assumed DeepSeek would be great at Chinese (it's a Chinese company). It scored #1. But it also had the highest rate-limit failures in P4 stability tests. Numbers don't lie — run the exams.

### 2. Silent failures are the worst

Some providers return `200 OK` with an empty body or `{"result": null}`. Your agent thinks it worked. It didn't. **Always validate the response shape**, not just the status code.

### 3. Fallback order matters more than you think

We tried random load balancing first. Bad idea. The best provider should always go first — you're optimizing for quality, not just availability. Fallback is for when quality fails, not for spreading load.

### 4. Language detection is cheap insurance

A simple regex + Unicode range check costs ~0.01ms. It saves you from sending Chinese to Groq and getting English garbage back. **Always detect input language before routing.**

### 5. Price ≠ Quality

Free APIs (Gemini, Groq, Cohere) scored higher than some paid ones in our P3 quality exams. Don't assume expensive = better. Test everything.

## Numbers

- **31 providers** tested across 4 exam rounds
- **10 Smart services** with automatic fallback
- **99.97% effective uptime** (3-5 provider chains)
- **$0.006–$0.009 per L2 call** (cheaper than most single-provider alternatives)
- **$0.02 per Concierge call** (AI picks tools + executes + synthesizes)

## Try It

```bash
# Create account
curl -X POST https://api.washinmura.jp/api/proxy/account/create \
  -H "Content-Type: application/json" \
  -d '{"name": "My Agent"}'

# Smart Search (3-engine fallback + AI summary)
curl -X POST https://api.washinmura.jp/api/v2/search \
  -H "Authorization: Bearer wv_YOUR_KEY" \
  -d '{"query": "multi-provider API fallback patterns"}'

# AI Concierge (natural language → auto tool selection)
curl -X POST https://api.washinmura.jp/api/v2/smart \
  -H "Authorization: Bearer wv_YOUR_KEY" \
  -d '{"message": "Compare Bitcoin and Ethereum prices, then translate the summary to Japanese"}'
```

Full docs: [api.washinmura.jp/docs](https://api.washinmura.jp/docs)
Benchmark data: [github.com/sstklen/washin-api-benchmark](https://github.com/sstklen/washin-api-benchmark)

---

*Built at [Washin Village](https://washinmura.jp) — an animal sanctuary in Boso Peninsula, Japan, where 28 cats and dogs fund API infrastructure through a token economy.*
