# OpenClaw：設計最佳 AI API 路由 — 測試 31 家供應商，找出 10 條最佳路徑

> AI agent 需要 LLM、搜尋、翻譯、語音等多種能力 — 但沒有任何一家供應商在所有項目都是最強的。我們對 31 家 API 供應商進行了 4 輪考試，為每項任務設計出最佳路由。以下是每個路由決策背後的數據與思考邏輯。

## 為什麼要做這件事

我們正在為 AI agent 建構 API 基礎設施。一個 AI agent 不只呼叫一個 API — 它可能需要搜尋網頁、讀取頁面、翻譯結果、再進行摘要，全部在一個工作流程中完成。

我們面臨的問題是：**對於每一項任務，應該路由到哪個供應商？**

最偷懶的做法是為每個類別選最知名的品牌 — 或最高等級的方案。但我們的目標從來不是找最貴的供應商，而是找到**最適合的**：在我們 agent 需要的任務上、在我們使用者使用的語言中，實際得分最高的供應商。

知名度不保證品質，高級標籤也不保證。一個 80 億參數的模型在我們的考試中贏了 GPT-4o-mini 10 分。基於 LLM 的翻譯品質分數追平了 DeepL。在我們測試中排名第一的供應商，往往不是你預期的那些。

因此我們決定先測試一切，再純粹根據數據來設計路由 — 不靠品牌、不靠定價層級、不靠假設。

## 測試方法：4 輪考試、31 家供應商

我們設計了一套 4 輪考試系統。每一輪都能捕捉到其他輪次捕捉不到的問題。

| Round | The Question | What It Revealed |
|-------|-------------|-----------------|
| **P1 — Connectivity** | Is this API even alive? | 3 out of 30 providers were dead on arrival |
| **P2 — Capability** | What can it do? (tested in EN/CN/JP) | Same API scores 100 in English, 30 in Chinese |
| **P3 — Quality** | Who's the best at each task? | 8B model scores higher than GPT-4o-mini |
| **P4 — Stability** | Can it handle 100 calls in a row? | #1 quality scorer had the worst reliability |

**四輪缺一不可。** 如果我們在 P3 就停下來，會選 DeepSeek 作為中文路由（品質分數最高），然後在正式環境被限速。P4 抓到了這個問題。

所有測試腳本與原始數據：**[washin-api-benchmark](https://github.com/sstklen/washin-api-benchmark)**

## 改變路由設計的關鍵發現

在 P2 中，我們用三種語言的相同提示詞測試了每個 LLM。結果徹底改變了我們的思路：

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

同一個 API、同一個模型、同一種呼叫方式。品質崩跌 70 分 — 而且對監控系統完全不可見，因為回應是一個格式正確的 `200 OK` 加上合法的 JSON。

**這告訴我們：不能對所有語言使用相同的路由。** 服務中文使用者的 agent 需要一條完全不同於英文使用者的供應商鏈。我們稱之為 **language-aware routing**（語言感知路由）— 截至目前，現有的 API gateway（OpenRouter、Portkey、LiteLLM）都沒有做到這一點。

## 完整 LLM 排名

以下是我們 P3 品質考試的結果 — 這是 LLM 路由設計的基礎：

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

三個重點：
- **Cerebras 8B（92 分）擊敗 GPT-4o-mini（82 分）** — 模型小了好幾倍、速度快 5 倍、分數高 10 分
- **GPT-4o-mini 推理只拿 30 分** — 它算錯了一道基礎數學題
- **看 CN 欄** — Groq 30、Cerebras 30，其他都是 100。這就是我們要用路由繞過的斷層

## 另外 4 個影響決策的發現

**基於 LLM 的翻譯品質追平專用翻譯 API：**

| Provider | Type | Score |
|----------|------|-------|
| Groq Translate | LLM-based | 94 |
| Cerebras Translate | LLM-based | 94 |
| DeepL | Dedicated engine | 93 |

這意味著我們可以用 LLM 供應商作為翻譯的備援 — 但 CJK 的盲區同樣存在。

**靜默失敗確實存在：** 有些供應商回傳 `200 OK`，但 body 是 `{"result": null}` 或空的。我們在 P4 中捕捉到了這種情況。如果你只檢查 HTTP 狀態碼，就會把空資料傳給使用者。

**DeepSeek 矛盾：** P3 中文品質分數最高。P4 限速失敗次數最多。品質與可靠性是兩個獨立變數 — 兩者都必須測試。

**搜尋引擎全部拿 100 分：** Tavily、Brave 和 Serper 都通過了品質測試。差異在於格式（Tavily：AI-ready）、數量（Brave：10 筆結果）和速度（Serper：537ms）。

---

## 我們做出的 10 個路由決策

以下每條路由順序都直接來自考試分數。第一順位 = P3 最高分。備援 = 次高分。對於 CJK 輸入，低於 50 分的供應商會被跳過。

### 1. LLM — 最複雜的路由

這是我們最重要的路由決策，也是需要最多思考的一條。

**英文路由** — 全部 5 家供應商可用：
```
Gemini(93) → Mistral(87) → Cerebras(92) → Groq(83) → Cohere(78)
```

**中文/日文路由** — 跳過 Cerebras(CN:30) 和 Groq(CN:30)：
```
Gemini(93) → Mistral(87) → Cohere(78)
```

為什麼 Gemini 排在 Cerebras 前面，即使 Cerebras 拿了 92 分？因為 Gemini 在三種語言中都拿 100 分。我們的優先策略是**跨語言的一致品質優先**，其次才是速度。

### 2. Search（搜尋）— 相關性優先，數量作備援

```
Tavily(Q100, 5 results, best relevance)
  → Brave Search(Q100, 10 results, widest coverage)
  → Gemini Grounding(last resort)
  → then: Groq generates AI summary of results
```

三家品質分數都是 100。我們把 Tavily 排第一，因為它的 AI-ready 格式在餵給 LLM 時能產生最好的下游結果。

### 3. Translation（翻譯）— DeepL 領頭，LLM 備援

```
DeepL(Q93) → Groq(Q94) → Gemini(Q93) → Cerebras(Q94) → Mistral
CJK targets: skip Groq and Cerebras (same language blind spot as LLM)
```

DeepL 排第一是為了專業一致性。LLM 翻譯器分數一樣高，但繼承了 CJK 的盲區。

### 4. Web Reading（網頁讀取）— 4 個層級對應 4 種頁面

```
Jina Reader → Firecrawl → ScraperAPI → Apify
```

| Level | Handles | Fails On |
|-------|---------|----------|
| Jina | ~70% of pages, fast | JS-heavy SPAs |
| Firecrawl | JS rendering | Some anti-bot sites |
| ScraperAPI | Broad coverage, proxy rotation | Less clean output |
| Apify | Most resilient edge cases | Slowest |

每一層都能接住上一層漏掉的。

### 5. Embedding（向量嵌入）— 多語言品質優先

```
Cohere(best multilingual vectors) → Gemini → Jina
```

### 6. Speech-to-Text（語音轉文字）— 速度 vs 功能

```
Deepgram Nova-2(faster) → AssemblyAI(more features)
```

### 7. Text-to-Speech（文字轉語音）— 單一供應商（我們最弱的環節）

```
ElevenLabs
```

在我們的測試中沒有找到可比較的替代方案。零備援。我們正在積極評估新的供應商。

### 8. Geocoding（地理編碼）— 從覆蓋率到精準度

```
Nominatim(broadest coverage) → OpenCage(best formatting) → Mapbox(most accurate)
```

### 9. News（新聞）— 專用索引到通用搜尋

```
NewsAPI(dedicated news index) → Web Search fallback
```

### 10. Structured Extraction（結構化擷取）— 兩階段 Pipeline

```
Web Reader(URL → clean text) → LLM(text → structured JSON per your schema)
```

這不是備援鏈 — 而是一條 pipeline。Web Reader 使用第 4 條的 4 層備援。LLM 使用第 1 條的語言感知路由。

---

## 總覽：10 條路由一覽表

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

注意這張表的一個特點：**每個類別的首選幾乎都不是最知名或最高級的供應商。** Gemini 勝過 GPT、Tavily 勝過 Google、Cohere 勝過 OpenAI embeddings、Nominatim 勝過 Mapbox。每一個首選決策都是靠考試分數做出的，不是靠名氣。結果是一張針對「實際表現最好」而非「在簡報上看起來最厲害」優化的路由表。

---

## 實作方式：語言感知的備援機制

所有 10 條路由背後的共通模式：

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

**可靠性的數學：**
- 1 家供應商：約 99% uptime（每月停機 7 小時以上）
- 3 家供應商：約 99.97%（每月停機約 2 分鐘）
- 5 家供應商：約 99.9999%

上個月 Groq 停機 30 分鐘時，我們的路由自動切換到 Gemini。Agent 完全零停機。

---

## 我們學到的 5 件事

**1. 只用英文的 benchmark 會誤導你。** 用你的 agent 實際使用的語言來測試。排名會完全洗牌。

**2. 驗證回應內容，不只是狀態碼。** `200 OK` 搭配空資料就是靜默失敗。永遠要檢查資料結構。

**3. 品質 ≠ 可靠性。** P3（品質）和 P4（穩定性）都要跑。最高分的供應商可能也是最不穩定的。

**4. 語言偵測只花 0.01ms。** 一個 Unicode 範圍的 regex。就能避免把中文請求路由到 30 分的供應商。

**5. 每月重跑考試。** 供應商會變、模型會更新。上個月的冠軍可能是這個月的墊底。

---

## 方法論與數據

- **31 家供應商**測試，4 輪考試，3 種語言（EN/CN/JP）
- **從東京測試**（AWS ap-northeast-1）
- **完全開源** — 腳本、原始數據、評分標準

Benchmark 倉庫：[github.com/sstklen/washin-api-benchmark](https://github.com/sstklen/washin-api-benchmark)
互動式報告：[api.washinmura.jp/api-benchmark](https://api.washinmura.jp/api-benchmark/en/)

---

*由 [Washin Village](https://washinmura.jp) 團隊發布 — 28 隻貓狗、一個 API 基礎設施團隊，每月從日本房總半島發布 benchmark 報告。*
