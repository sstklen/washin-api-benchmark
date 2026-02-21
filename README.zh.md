**🌐 語言：** [English](README.md) | 繁體中文 | [日本語](README.ja.md)

# AI Agent API 評測 — 月度報告

> **我們每月測試 30+ 個 AI API，讓你不用親自動手。**
> 公開方法論。無贊助商。來自東京的真實數據。

🌐 **完整互動式報告：** [English](https://api.washinmura.jp/api-benchmark/en/) | [繁體中文](https://api.washinmura.jp/api-benchmark/zh/) | [日本語](https://api.washinmura.jp/api-benchmark/ja/)

📡 **立即試用這些 API：** [MCP Server（免費）](https://api.washinmura.jp/mcp/free) | [API 文件](https://api.washinmura.jp/docs)

---

## 2026 年 2 月結果

**測試項目：** 15 個 LLM · 3 個搜尋引擎 · 5 個翻譯服務 · 3 個語音服務 · 6 個資料服務
**日期：** 2026-02-20 · **地點：** 日本東京 · **方法：** 每個 API 測試 4 輪

### 🏆 LLM 品質排名（前 15 名）

| # | 模型 | 分數 | 速度 | 推理 | 程式碼 | CN/JP/EN |
|---|------|------|------|------|--------|----------|
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

> **推理測試題：** 「一間庇護所有 28 隻動物。3/7 是貓。貓每月吃 2kg，其他動物每月吃 1.5kg。每月總飼料量？」（答案：48kg）

### 🔍 搜尋引擎

| 供應商 | 分數 | 速度 | 結果數 | 最適合用途 |
|--------|------|------|--------|-----------|
| **Brave Search** | 100 | 1124ms | 每次查詢 10 筆 | 量大（最多結果） |
| **Tavily** | 100 | 1536ms | 每次查詢 5 筆 | 品質佳 + AI 友善 |
| **Serper (Google)** | 100 | 537ms | 每次查詢 8 筆 | 速度快 + Google 資料 |

### 🌐 翻譯

| 供應商 | 分數 | 速度 | 最適合用途 |
|--------|------|------|-----------|
| **Groq Translate** | 94 | 526ms | 最佳品質（免費） |
| **DeepL** | 93 | 641ms | 專業用途 |
| **Cerebras Translate** | 94 | 335ms | 最快 + 高品質 |

> 💡 免費的 LLM 翻譯（Groq/Cerebras）分數比 DeepL **更高**。

### 📊 總結統計

| 指標 | 數值 |
|------|------|
| API 連線成功率 | 86.7%（26/30 通過） |
| 24 小時穩定性 | 96.9%（31/32 穩定） |
| 最快 LLM | Groq 306ms |
| 最高 LLM 分數 | 93（Gemini 2.5 Flash / xAI Grok） |

---

## 3 個令人驚訝的發現

### 1. 🤯 GPT-4o-mini 連基本數學都算不對
問「17 + 35」→ 回答 **54**（完整題目正確答案：48）。推理分數：30/100。
如果你的 AI Agent 依賴 GPT-4o-mini 做計算，那你有麻煩了。

### 2. 💪 免費的 8B 模型打敗 GPT
Cerebras llama3.1-8b（免費，80 億參數）拿到 **92** 分，而 GPT-4o-mini 只有 **82** 分。
延遲 316ms。免費。比 GPT 更強。

### 3. ⚡ 最快 ≠ 最好
Groq 速度比平均快 8 倍（306ms），但中文分數暴跌至 **30/100**。
對於非英語 Agent 來說，沒有多語言品質的速度只是個陷阱。

---

## 使用場景推薦

| 場景 | 推薦組合 |
|------|----------|
| **研究型 Agent** | Brave Search → Firecrawl → Gemini 2.5 Flash |
| **即時聊天 Agent** | Groq 306ms（英文）/ Mistral Small 557ms（多語言） |
| **翻譯 Agent** | Groq Translate（94 分）或 DeepL（93 分） |
| **數學/推理** | Gemini 2.5 Flash 或 DeepSeek Chat（皆 100 分） |
| **程式碼生成** | Gemini 2.5 Flash / xAI Grok / Cerebras 8B（皆 100 分） |
| **語音助手** | AssemblyAI STT → Groq LLM → ElevenLabs TTS |
| **新聞監控** | Brave Search + NewsAPI → Mistral Small |

---

## ⚠️ 5 個 API 欄位名稱陷阱

這些欄位名稱**不是**你以為的那樣。搞錯 = 無聲失敗：

| API | ❌ 預期 | ✅ 實際 |
|-----|---------|---------|
| Vision | `imageUrl` | `image` |
| Geocode | `query` | `q` |
| CoinGecko | `coin` | `coins` |
| Serper | `results` | `organic` |
| X Search | `results` | `tweets` |

---

## 測試方法

1. **真實 API 呼叫** — 沒有合成測試。每個數字都來自真實的 HTTP 請求。
2. **每個 API 測試 4 輪** — 每項測試執行 4 次以消除變異。
3. **從東京發起** — 所有測試都從東京伺服器執行（AWS ap-northeast-1）。
4. **公開評分標準** — 推理 = 數學正確性、程式碼 = 函式輸出、多語言 = CN/JP/EN 準確度。
5. **無贊助商** — 排名完全由數據驅動。所有 API 費用由我們自行負擔。

---

## 關於我們

由 [washinmura](https://washinmura.jp) 發布 — 位於日本房總半島的動物庇護所，同時經營 AI Agent 的 API 市集。

- 🐾 28 隻貓狗
- 🤖 30+ 個 API 服務
- 📊 自 2026 年 2 月起每月發布評測報告

**下期報告：** 2026 年 3 月

---

## 授權條款

資料與報告以 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權發布。
歡迎分享與改作，請附上出處。
