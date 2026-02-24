<p align="center">
  <h1 align="center">From Benchmarks to Architecture</h1>
  <p align="center"><strong>We tested 30+ AI APIs. Designed routing from the data. Then Anthropic published a paper describing the same architecture.</strong></p>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/APIs_Tested-30+-blue?style=for-the-badge" alt="APIs Tested"/>
  <img src="https://img.shields.io/badge/Articles-3_Part_Series-green?style=for-the-badge" alt="3 Articles"/>
  <img src="https://img.shields.io/badge/Location-Tokyo_🗼-red?style=for-the-badge" alt="Location"/>
  <img src="https://img.shields.io/badge/Sponsors-Zero-orange?style=for-the-badge" alt="No Sponsors"/>
</p>

<p align="center">
  <a href="https://github.com/sstklen/washin-api-benchmark/stargazers"><img src="https://img.shields.io/github/stars/sstklen/washin-api-benchmark?style=social" alt="Stars"/></a>
</p>

---

## The Stack

Three articles. Each one builds on the previous. Together, they're the complete story of how we built intelligent API routing — from raw data to production architecture.

```
Article 1: OBSERVE     — Benchmark 30+ APIs (who's fast, who's accurate, who lies)
    ↓
Article 2: DESIGN      — Use exam data to design multi-provider routing
    ↓
Article 3: ARCHITECT   — Build L1→L4, then Anthropic publishes the same patterns
```

This is [Observability-Driven Routing](https://en.wikipedia.org/wiki/Observability_(software)): measure first, decide from data, automate decisions, verify results. We didn't know it had a name. We just kept solving the next problem.

---

## Article 1: The Benchmark

**[30+ AI APIs Tested from Tokyo — February 2026](docs/01-benchmark-2026-02.md)**

15 LLMs, 3 search engines, 5 translation APIs, 3 voice APIs, 6 data APIs. Four rounds of real HTTP requests, not synthetic benchmarks.

**Key findings:**

| Discovery | Why it matters |
|-----------|---------------|
| GPT-4o-mini can't do basic math (30/100 reasoning) | Don't trust brand names |
| Free 8B model beats GPT (Cerebras 92 vs GPT 82) | Cost ≠ quality |
| Groq is 8x faster but scores 30/100 on Chinese | Speed ≠ quality across languages |
| Free LLM translation beats DeepL (94 vs 93) | Dedicated APIs aren't always better |

> *If we'd picked providers by reputation instead of data, we'd have chosen wrong on every one of these.*

---

## Article 2: The Routing Design

**[31 Providers Tested, 10 Best Paths Found](docs/02-multi-provider-routing.md)**

The benchmark data revealed a critical insight: **no single provider is best at everything.** Groq is fastest but fails on Chinese. Cerebras scores highest but can't do Japanese. GPT-4o-mini handles all languages but can't do math.

So we designed **language-aware, quality-driven routing** — different provider chains for different languages and tasks, all driven by exam data.

**Key insight:** A provider scoring 100 in English and 30 in Chinese isn't a bug in our test. It's a 70-point quality collapse that's completely invisible to monitoring (valid 200 OK, well-formed JSON). Our routing catches this. No existing API gateway does.

Available in: [English](docs/02-multi-provider-routing.md) · [繁體中文](docs/02-multi-provider-routing.zh.md) · [日本語](docs/02-multi-provider-routing.ja.md)

---

## Article 3: The Architecture ⭐ NEW

**[Token 76% Down, Cost 96% Down, 4.6x Faster — Reading Anthropic's Tool Use Paper, 4 Commits Same Day](docs/03-anthropic-advanced-tool-use.md)**

Anthropic published ["Advanced Tool Use"](https://www.anthropic.com/engineering/advanced-tool-use) describing three techniques: Tool Search, Tool Use Examples, and Programmatic Tool Calling.

We read it. Recognized the same patterns we'd been building. Implemented two features the same day. And realized we'd independently built four things they hadn't described.

| What Anthropic published | What we already had | What we built after reading |
|---|---|---|
| Tool Search (85% token reduction) | — | Defer loading: 10.8KB→2.5KB (**76%↓**), same day |
| Tool Use Examples (72%→90% accuracy) | — | 11 endpoints with real JSON examples, same day |
| Dynamic Filtering | L2 Smart Gateway (strategy routing) | — |
| Programmatic Tool Calling (37% token↓) | L4 Task Engine | PTC mode: **$0.02 vs $0.49 (96%↓)**, same day |

**What we have that they didn't describe:**
- Multi-provider fallback chains (L2) — because the right tool can still go down
- Exam-driven routing (P1-P4) — because static examples go stale
- Intent routing (L3) — because agents don't always know which tool to call
- Post-execution quality signals (L4 Phase 3) — because knowing it ran ≠ knowing it worked

Available in: [English](docs/03-anthropic-advanced-tool-use.md) · [繁體中文](docs/03-anthropic-advanced-tool-use.zh.md) · [日本語](docs/03-anthropic-advanced-tool-use.ja.md)

---

## Why This Matters

Companies working on similar problems: Martian (LLM Router), Not Diamond (Model Router), Portkey, LiteLLM (AI Gateway). They all solve pieces of this puzzle.

What makes our approach different:

| | Existing AI Gateways | Our Approach |
|---|---|---|
| Routing basis | Static config or cost | **Exam data + live performance** |
| Language awareness | None | **Per-language provider ranking** |
| Fallback | Simple retry | **4-deep provider chain with different timeouts** |
| Quality check | None | **Post-execution scoring (0-1)** |
| Intent routing | None | **Natural language → auto tool selection** |

We're not a company. We're an animal sanctuary in rural Japan that needed APIs to work reliably. The architecture emerged from solving real problems, one layer at a time.

---

## Who We Are

**[Washin Village](https://washinmura.jp)** (和心村) — an animal sanctuary on Japan's Boso Peninsula. 28 cats & dogs. Zero engineering background. Built a production API platform with AI coding agents in 7 months.

The benchmarks, the routing design, and the architecture — all open. Because the problems are universal, and the solutions should be too.

---

## Related

| Project | What it does |
|---------|-------------|
| [Zero Engineer](https://github.com/sstklen/zero-engineer) | The full story: animal sanctuary → production API platform |
| [112 Claude Code Skills](https://github.com/sstklen/washin-claude-skills) | Every production bug fix as a reusable skill |
| [crawl-share](https://github.com/sstklen/crawl-share) | 200+ Apify actors battle-tested, community-shared |
| [Confucius Debug](https://github.com/sstklen/yanhui-ci) | Debug AI with 4,300+ shared solutions |

---

## License

Data and reports: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Share and adapt with attribution.
