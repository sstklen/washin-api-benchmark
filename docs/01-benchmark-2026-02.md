<p align="center">
  <h1 align="center">AI Agent API Benchmark</h1>
  <p align="center"><strong>We test 30+ AI APIs every month from Tokyo so you don't have to.</strong></p>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/APIs_Tested-30+-blue?style=for-the-badge" alt="APIs Tested"/>
  <img src="https://img.shields.io/badge/Updated-Feb_2026-green?style=for-the-badge" alt="Last Updated"/>
  <img src="https://img.shields.io/badge/Location-Tokyo_🗼-red?style=for-the-badge" alt="Location"/>
  <img src="https://img.shields.io/badge/Sponsors-Zero-orange?style=for-the-badge" alt="No Sponsors"/>
</p>

<p align="center">
  <a href="https://github.com/sstklen/washin-api-benchmark/stargazers"><img src="https://img.shields.io/github/stars/sstklen/washin-api-benchmark?style=social" alt="Stars"/></a>
  <a href="https://api.washinmura.jp/api-benchmark/en/">🌐 Interactive Report</a> ·
  <a href="https://api.washinmura.jp/mcp/free">📡 Free MCP Server</a> ·
  <a href="https://api.washinmura.jp/docs">📖 API Docs</a>
</p>

<p align="center"><sub>🌐 <a href="README.zh.md">繁體中文</a> · <a href="README.ja.md">日本語</a></sub></p>

---

## 🤯 3 Things We Discovered This Month

> *These findings are from real API calls, not marketing pages.*

### GPT-4o-mini can't do basic math

We asked every LLM: *"A shelter has 28 animals. 3/7 are cats. Cats eat 2kg/month, others 1.5kg. Total feed?"*

GPT-4o-mini answered **54**. The correct answer is **48**.

**Reasoning score: 30/100.** If your AI Agent does calculations with GPT-4o-mini, you have a problem.

### A free 8B model beats GPT

| Model | Score | Speed | Cost |
|-------|-------|-------|------|
| Cerebras llama3.1-8b | **92** | ⚡ 316ms | Free |
| GPT-4o-mini | **82** | 1631ms | Paid |

An 8-billion-parameter open-source model, running for free, outperforms GPT-4o-mini in quality *and* speed.

### Fastest ≠ Best (the multilingual trap)

Groq is 8x faster than average (306ms). Looks amazing on paper.

But Chinese accuracy: **30/100**. If your users speak anything other than English, pure speed is a trap.

---

## 🏆 February 2026 Rankings

**15 LLMs · 3 Search · 5 Translation · 3 Voice · 6 Data** — tested from Tokyo, 4 rounds each

### LLM Quality

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

### Search Engines

| Provider | Score | Speed | Results | Best For |
|----------|-------|-------|---------|----------|
| **Brave Search** | 100 | 1124ms | 10/query | Volume |
| **Tavily** | 100 | 1536ms | 5/query | AI-ready quality |
| **Serper (Google)** | 100 | 537ms | 8/query | Speed + Google data |

### Translation

| Provider | Score | Speed | Cost |
|----------|-------|-------|------|
| **Cerebras Translate** | 94 | ⚡ 335ms | Free |
| **Groq Translate** | 94 | 526ms | Free |
| **DeepL** | 93 | 641ms | Paid |

> Free LLM-based translation now scores **higher** than DeepL.

### At a Glance

| Metric | Value |
|--------|-------|
| API Connectivity | 86.7% (26/30 passed) |
| 24h Stability | 96.9% (31/32 stable) |
| Fastest LLM | Groq 306ms |
| Best Overall | Gemini 2.5 Flash (93pts) |

---

## 🛠️ Pick Your Stack

Building an AI Agent? Here's what we'd use based on the data:

| Your Agent Does... | Recommended Stack | Why |
|--------------------|-------------------|-----|
| **Research** | Brave Search → Firecrawl → Gemini 2.5 Flash | Best quality + multilingual |
| **Realtime Chat** | Groq 306ms (EN) / Mistral Small 557ms (multilingual) | Speed vs language trade-off |
| **Translation** | Cerebras Translate (94pts, free, 335ms) | Beats DeepL, costs nothing |
| **Math/Reasoning** | Gemini 2.5 Flash or DeepSeek Chat | Both score 100 on reasoning |
| **Code Gen** | Gemini 2.5 Flash / xAI Grok / Cerebras 8B | All score 100 on code |
| **Voice** | AssemblyAI STT → Groq LLM → ElevenLabs TTS | Best pipeline we've found |
| **News** | Brave Search + NewsAPI → Mistral Small | Volume + multilingual |

---

## ⚠️ API Field Name Traps

We learned these the hard way. Getting them wrong = silent failures:

| API | ❌ You'd expect | ✅ Actually |
|-----|----------------|-------------|
| Vision | `imageUrl` | `image` |
| Geocode | `query` | `q` |
| CoinGecko | `coin` | `coins` |
| Serper | `results` | `organic` |
| X Search | `results` | `tweets` |

---

## 📐 Methodology

| Aspect | How |
|--------|-----|
| **Data** | Real HTTP requests, not synthetic benchmarks |
| **Rounds** | 4 per API to account for variance |
| **Location** | Tokyo server (AWS ap-northeast-1) |
| **Scoring** | Reasoning = math correctness, Code = function output, Multilingual = CN/JP/EN accuracy |
| **Bias** | Zero sponsors. We pay for everything ourselves |

---

## 📅 Get Next Month's Report

**⭐ Star this repo** to get notified when March 2026 results drop.

We publish on the 20th of every month.

| Month | Status |
|-------|--------|
| February 2026 | ✅ Published |
| March 2026 | 🔜 Coming |

---

## 🔗 Deep Dive

- **[Designing the Optimal AI API Route](docs/why-multi-provider-fallback.md)** — We tested 31 providers to find the best path for every task. The exam data, the routing decisions, and why language-aware fallback matters.

---

## About

Published by **[Washin Village](https://washinmura.jp)** — an animal sanctuary in Boso Peninsula, Japan, building an API marketplace for AI Agents.

🐾 28 cats & dogs · 🤖 30+ APIs · 📊 Monthly benchmarks

## License

Data and reports: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Share and adapt with attribution.
