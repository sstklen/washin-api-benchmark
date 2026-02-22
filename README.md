**🌐 Language:** English | [繁體中文](README.zh.md) | [日本語](README.ja.md)

# AI Agent API Benchmark — Monthly Report

> **We test 30+ AI APIs every month so you don't have to.**
> Open methodology. No sponsors. Real data from Tokyo.

🌐 **Full Interactive Report:** [English](https://api.washinmura.jp/api-benchmark/en/) | [繁體中文](https://api.washinmura.jp/api-benchmark/zh/) | [日本語](https://api.washinmura.jp/api-benchmark/ja/)

📡 **Try these APIs instantly:** [MCP Server (free)](https://api.washinmura.jp/mcp/free) | [API Docs](https://api.washinmura.jp/docs)

---

## February 2026 Results

**Tested:** 15 LLMs · 3 Search Engines · 5 Translation · 3 Voice · 6 Data Services
**Date:** 2026-02-20 · **Location:** Tokyo, Japan · **Method:** 4 rounds per API

### 🏆 LLM Quality Ranking (Top 15)

| # | Model | Score | Speed | Reasoning | Code | CN/JP/EN |
|---|-------|-------|-------|-----------|------|----------|
| 🥇 | **Gemini 2.5 Flash** | **93** | 990ms | ✅ 100 | ✅ 100 | 100/100/100 |
| 🥈 | **xAI Grok 4.1 Fast** | **93** | 1621ms | ✅ 100 | ✅ 100 | 100/100/100 |
| 🥉 | **Cerebras llama3.1-8b** | **92** | ⚡ 316ms | ✅ 100 | ✅ 100 | 30/60/60 |
| 4 | Gemini 2.0 Flash | 88 | 668ms | ❌ 30 | ✅ 100 | 100/100/100 |
| 5 | DeepSeek Chat | 87 | 1046ms | ✅ 100 | 60 | 100/100/100 |
| 5 | Mistral Small | 87 | 557ms | ✅ 100 | 60 | 100/100/100 |
| 7 | DeepSeek Reasoner (R1) | 83 | 2696ms | ✅ 100 | 0 | 100/100/100 |
| 7 | Groq llama-3.3-70b | 83 | ⚡ 306ms | ✅ 100 | 60 | 30/100/100 |
| 9 | OpenAI GPT-4o-mini | 82 | 1631ms | ❌ 30 | 60 | 100/100/100 |
| 10 | Cerebras GPT-OSS-120B | 80 | 382ms | ✅ 100 | 20 | 100/100/100 |
| 11 | Cohere Command R7B | 78 | 393ms | ✅ 100 | ✅ 100 | 100/100/0 |
| 11 | Mistral Codestral | 78 | 479ms | ❌ 30 | 60 | 100/100/100 |

> **Reasoning test:** "A shelter has 28 animals. 3/7 are cats. Cats eat 2kg/month, others eat 1.5kg/month. Total monthly feed?" (Answer: 48kg)

### 🔍 Search Engines

| Provider | Score | Speed | Results | Best For |
|----------|-------|-------|---------|----------|
| **Brave Search** | 100 | 1124ms | 10 per query | Volume (most results) |
| **Tavily** | 100 | 1536ms | 5 per query | Quality + AI-ready |
| **Serper (Google)** | 100 | 537ms | 8 per query | Speed + Google data |

### 🌐 Translation

| Provider | Score | Speed | Best For |
|----------|-------|-------|----------|
| **Groq Translate** | 94 | 526ms | Best quality (free) |
| **DeepL** | 93 | 641ms | Professional use |
| **Cerebras Translate** | 94 | 335ms | Fastest + quality |

> 💡 Free LLM-based translation (Groq/Cerebras) scores **higher** than DeepL.

### 📊 Summary Stats

| Metric | Value |
|--------|-------|
| API Connectivity | 86.7% (26/30 passed) |
| 24h Stability | 96.9% (31/32 stable) |
| Fastest LLM | Groq 306ms |
| Highest LLM Score | 93 (Gemini 2.5 Flash / xAI Grok) |

---

## 3 Surprising Findings

### 1. 🤯 GPT-4o-mini Can't Do Basic Math
Asked "17 + 35" → Answered **54** (correct: 48 for the full problem). Reasoning score: 30/100.
If your AI Agent relies on GPT-4o-mini for calculations, you have a problem.

### 2. 💪 A Free 8B Model Beats GPT
Cerebras llama3.1-8b (free, 8 billion parameters) scored **92** vs GPT-4o-mini's **82**.
316ms latency. Free. Better than GPT.

### 3. ⚡ Fastest ≠ Best
Groq is 8x faster than the average (306ms), but Chinese score collapsed to **30/100**.
Speed without multilingual quality is a trap for non-English agents.

---

## Use Case Recommendations

| Scenario | Recommended Stack |
|----------|-------------------|
| **Research Agent** | Brave Search → Firecrawl → Gemini 2.5 Flash |
| **Chat Agent (realtime)** | Groq 306ms (English) / Mistral Small 557ms (multilingual) |
| **Translation Agent** | Groq Translate (94pts) or DeepL (93pts) |
| **Math/Reasoning** | Gemini 2.5 Flash or DeepSeek Chat (both 100) |
| **Code Generation** | Gemini 2.5 Flash / xAI Grok / Cerebras 8B (all 100) |
| **Voice Assistant** | AssemblyAI STT → Groq LLM → ElevenLabs TTS |
| **News Monitoring** | Brave Search + NewsAPI → Mistral Small |

---

## ⚠️ 5 API Field Name Traps

These field names are **not** what you'd expect. Getting them wrong = silent failures:

| API | ❌ Expected | ✅ Actual |
|-----|------------|-----------|
| Vision | `imageUrl` | `image` |
| Geocode | `query` | `q` |
| CoinGecko | `coin` | `coins` |
| Serper | `results` | `organic` |
| X Search | `results` | `tweets` |

---

## Methodology

1. **Real API calls** — No synthetic benchmarks. Every number is from a real HTTP request.
2. **4 rounds per API** — Each test runs 4 times to account for variance.
3. **From Tokyo** — All tests run from a Tokyo server (AWS ap-northeast-1).
4. **Open scoring** — Reasoning = math correctness, Code = function output, Multilingual = accuracy in CN/JP/EN.
5. **No sponsors** — Rankings are purely data-driven. We pay for all API access ourselves.

---

## Deep Dive

- **[Same API, Same Model, Same Call: 100 in English, 30 in Chinese](docs/why-multi-provider-fallback.md)** — 31 providers, 3 languages, 4 rounds of exams. The data behind every routing decision, and why language-aware fallback doesn't exist yet.

---

## About

Published by [washinmura](https://washinmura.jp) — an animal sanctuary in Boso Peninsula, Japan, running an API marketplace for AI Agents.

- 🐾 28 cats & dogs
- 🤖 30+ API services
- 📊 Monthly benchmarks since February 2026

**Next report:** March 2026

---

## License

Data and reports are published under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
You may share and adapt with attribution.
