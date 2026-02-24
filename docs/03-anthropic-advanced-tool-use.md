**Language:** English | [繁體中文](03-anthropic-advanced-tool-use.zh.md) | [日本語](03-anthropic-advanced-tool-use.ja.md)

# Token 76% Down, Cost 96% Down, 4.6x Faster — Reading Anthropic's Tool Use Paper, 4 Commits Same Day

> Anthropic published [Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use), proposing three features. We run a 39-service API platform from rural Japan. Read the paper, shipped all three the same day — plus four things they didn't do.

---

## The Full Picture

| Anthropic Feature             | Their Numbers              | What We Did                            |
| ----------------------------- | -------------------------- | -------------------------------------- |
| **Tool Search**               | 77K→8.7K tokens (85%↓)    | 10.8KB→2.5KB, load on demand (**76%↓**) |
| **Tool Use Examples**         | Accuracy 72%→90%           | Real JSON examples on 11 endpoints      |
| **Programmatic Tool Calling** | tokens 37%↓               | PTC mode: **$0.02 vs $0.49, 96% off**  |

| What We Added         | One Line                                        |
| --------------------- | ----------------------------------------------- |
| **Fallback Chain** (L2)    | Picking the right tool doesn't mean it won't fail. 4-layer fallback catches it. |
| **Exam Routing** (P1-P4)  | Continuous exams drive provider rankings          |
| **Intent Routing** (L3)   | Speak plain language, system picks the tool       |
| **Quality Signal** (L4)   | Machine-readable result quality scores            |

---

## Background: Four Layers Born from Pain

```
L1  Proxy        — Pure passthrough (27 endpoints)     $0-$0.01
L2  Smart Gateway — Multi-provider fallback + routing   $0.006-$0.009
L3  Concierge    — Natural language → auto tool select  $0.02
L4  Task Engine  — Plan → Execute → Quality eval        $0.49-$2.99
```

None of this was designed on a whiteboard first. L1 is plain proxying. L2 was added after Brave went down for two hours and every client got 500 errors. L3 was added after Agents couldn't figure out whether to call search or news — or that they needed to translate to Japanese first. L4 was added after multi-step tasks like "compare quality across three translation APIs" kept failing. Every layer exists because of a specific pain point.

---

## Side by Side: Anthropic's Three Features

### Tool Search → defer_loading (76% Savings)

> *"With 200+ tools, the traditional approach consumed approximately 77K input tokens before any actual work began... With the Tool Search Tool, initial token consumption drops to approximately 8.7K."*

Our `/api/capabilities` returned 10.8KB in one shot. After reading the paper, we split it into two tiers the same day:

```bash
GET /api/services/brief     # 2.5KB menu
GET /api/services/{id}      # ~300B on demand
```

```json
// GET /api/services/brief response (excerpt)
{
  "v": "2.0", "total": 39,
  "services": [
    {"id": "brave-search", "price": 0.002, "cat": "search", "L": 1},
    {"id": "smart-search", "price": 0.009, "cat": "search", "L": 2},
    {"id": "smart",        "price": 0.02,  "cat": "concierge", "L": 3},
    {"id": "task",         "price": 0.05,  "cat": "orchestration", "L": 4}
  ],
  "free": ["weather", "wikipedia", "exchange-rate", "ip-geo", "geocode"],
  "detail": "/api/services/{id}"
}
```

| Before                   | After               | Savings |
| ---------------------- | ------------------- | ------- |
| 10.8KB (~2,700 tokens) | 2.5KB (~640 tokens) | **76%** |

Anthropic does it model-side (Claude searches internally, using `defer_loading: true`). We do it API-side (Agent loads what it needs, two endpoints). Theirs is more elegant. Ours is more direct. Same principle: **show the menu first, serve the dish later.**

**What we learned:** Anthropic gave us a name for the problem (`defer_loading`), quantified the impact (85%↓ + Opus 4 accuracy 49%→74%), and pointed us toward an implementation. "Feeling like it's too bloated" and "knowing exactly how to fix it" are two different things.

---

### Tool Use Examples (Anthropic measured +18% accuracy)

> *"Adding concrete examples to tool definitions improved accuracy from 72% to 90% on complex parameter handling."*

Our docs had URLs and parameters — but zero examples. Agents were guessing formats:

```
Guessed: {"search": "renewable energy Japan"}         ← wrong field name
Correct: {"query": "renewable energy Japan", "strategy": "fast"}
```

Same day, added real JSON request + response examples to all 11 endpoints:

```
## POST /api/v2/search
### Request Example:
{"query": "renewable energy Japan 2025", "strategy": "fast", "maxResults": 10}
### Response Example:
{"results": [...], "provider": "brave", "responseTimeMs": 420, "cost": "$0.009"}
```

**Months of building fallback chains might matter less than one day of adding examples — examples eliminate bad calls at the source. Agents don't call support. They read the docs.**

---

### Programmatic Tool Calling → PTC (96% Cost Reduction)

> *"Programmatic Tool Calling enables Claude to write and execute code that orchestrates multiple tool calls... On complex research tasks, this approach reduced average token usage from 43,588 to 27,297 — a 37% reduction."*

Our L4 already had three phases (plan → execute → quality eval). After reading the paper, we added PTC — Agents bring their own steps, skip the LLM planning phase:

```json
// PTC Mode — Agent brings its own execution plan
POST /api/v2/task
{"goal": "Search and summarize AI news", "steps": [
  {"toolId": "smart-search", "params": {"query": "AI agent news 2026"}},
  {"toolId": "smart-llm", "params": {"prompt": "Summarize"}, "dependsOn": [1]}
]}

// Response
{"success": true, "mode": "ptc", "synthesis": "...",
 "meta": {"price": 0.02, "execution": [
   {"step": 1, "tool": "smart-search", "responseTimeMs": 1408},
   {"step": 2, "tool": "smart-llm",    "responseTimeMs": 1292}
 ], "totalTimeMs": 3979}}
```

| Scenario       | Auto        | PTC         | Savings          |
| -------------- | ----------- | ----------- | ---------------- |
| Single query   | ~12s, $0.49 | 2.8s, $0.02 | 4x faster, 96% off   |
| Search+summary | ~18s, $0.49 | 3.9s, $0.02 | 4.6x faster, 96% off |

Anthropic lets Claude write Python to orchestrate (more flexible). We let Agents submit JSON steps (more reliable — every step has L2 fallback). Agents are paying for results, not for a Python script that might run.

---

## Four Things We Did That They Didn't

### Fallback Chain — Picking the Right Tool Doesn't Mean It Won't Fail

Anthropic assumes that picking the right tool means you get a result. In production, **picking the right tool doesn't mean it won't fail.**

```
Agent calls POST /api/v2/search
  → Brave (8s) → Tavily (10s) → Firecrawl (20s) → Gemini (20s)
  Agent is completely unaware of this chain.
```

**Real incident:** One afternoon, Brave went down for 6 minutes:

```
14:25  Brave timeout → fallback Tavily → success, 1200ms
14:31  Brave recovered. Client never noticed.
```

Without fallback chain? 6 minutes of 500 errors. With fallback chain? `provider` changes from `"brave"` to `"tavily"`, results come back as normal.

---

### Exam Routing (P1-P4) — Static Examples Go Stale

Anthropic says adding examples improves accuracy by +18%. But **static examples go stale.** Last month's best provider might be degraded this month.

| Exam   | What It Tests         | Frequency | Drives                        |
| ------ | --------------------- | --------- | ----------------------------- |
| **P1** | Is the endpoint alive? | Every 6h  | Remove dead providers          |
| **P3** | Who returns the best results? | Weekly    | **Drives L2 routing rankings** |
| **P4** | Long-term stability?  | Monthly   | Determines fallback order      |

**Real discovery:** P3 automatically found that one provider's Japanese query relevance was 15% higher than English. Nobody designed this — the exam data surfaced it on its own. The system auto-adjusted: Japanese searches prefer Provider A, English prefers Provider B. Hand-written examples would never catch this kind of pattern.

---

### Intent Routing (L3) — Agents Don't Know Who to Call

Anthropic assumes the caller knows which tool to use. Often not true.

```
"What do Japanese people think about the Shinkansen extension?"
  → actual query: "新幹線延伸 世論" (Japanese)

L3 (<500ms): search(Japanese) → translate → summarize. Three steps, auto-executed.

Without L3: Agent trial-and-error → 4-5 calls → $0.03-0.05, 8-15s
With L3:    One natural language sentence → 3 precise calls → $0.009, 3-5s
```

Intent parsing costs ~$0.0002/call. ROI: 100-200x.

---

### Quality Signal (Phase 3) — How Do You Know the Result Is Good?

Anthropic optimizes the path up to execution. But **what about after the call returns?**

```
Phase 3 evaluation → overall: 0.49 (results too stale) → auto-retry → 0.83 ✅
```

Quality scores are **machine-readable**. Agents don't have to choose between trusting everything or trusting nothing — they can **trust conditionally**. 0.83 is good enough: use it to draft, but flag the weak sections for follow-up.

---

## The Philosophical Difference

|              | Anthropic                  | Washin Village              |
| ------------ | -------------------------- | --------------------------- |
| **Direction**  | Make the model smarter at picking tools | Make tools easier to be picked |
| **Control**    | Model-side (Claude decides) | API-side (Agent decides)    |
| **Scope**      | Claude only                | Any LLM/Agent               |
| **Pricing**    | Hidden in token costs      | Transparent per step (cost + time) |

**Anthropic optimizes "how the model picks tools." We optimize "how tools make themselves pickable." Different paths, same destination.**

---

## Numbers

| Metric         | Value                                |
| -------------- | ------------------------------------ |
| Services       | 39 (L1x27 + L2x10 + L3 + L4)       |
| Categories     | 15                                   |
| Defer Loading  | 10.8KB → 2.5KB (76%↓)               |
| PTC vs Auto    | $0.02 vs $0.49 (96%↓)               |
| Exam Cycles    | P1 every 6h / P3 weekly / P4 monthly |
| Development    | 7 months, engineering background **zero** |

---

Direction validated — architecture we arrived at independently matches a top research lab's. Room at the application layer — fallback chains, exam routing, quality signals are things the model layer can't do.

**Read the paper. Learned something. Shipped it the same day.**

---

**References** — [Anthropic Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use) (2025) · [Zero Engineer](https://github.com/sstklen/zero-engineer) · [112 Claude Code Skills](https://github.com/sstklen/washin-claude-skills) · [crawl-share](https://github.com/sstklen/crawl-share) · [Confucius Debug](https://github.com/sstklen/yanhui-ci)

```
ca35575  feat: input_examples — real JSON examples on 11 endpoints
b31168c  feat: defer_loading — lightweight index + on-demand loading
9174e59  feat: dynamic filtering — 5 filter parameters
8f4a50d  feat: PTC — L4 supports agent-supplied execution plans
```

**One afternoon, 4 commits.** *Built with 🦞 in Boso Peninsula, Japan.*
