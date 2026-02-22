# OpenClaw：最適な AI API ルーティングの設計 — 31 プロバイダーをテストし、10 の最適経路を発見

> AI エージェントには LLM、検索、翻訳、音声など多くの機能が必要ですが、すべてに優れた単一のプロバイダーは存在しません。4 回のテストで 31 の API プロバイダーを評価し、各タスクに最適なルーティングを設計しました。本記事では、すべてのルーティング判断の根拠となったデータと設計思想を公開します。

## この調査を行った理由

私たちは AI エージェント向けの API インフラを構築しています。AI エージェントは単一の API を呼ぶだけではありません。Web 検索を行い、ページを読み取り、結果を翻訳し、要約する——これらすべてを一つのワークフロー内で処理する必要があります。

私たちが直面した問いはこうです。**これらのタスクそれぞれに対して、どのプロバイダーにルーティングすべきか？**

安易な方法は、カテゴリーごとに最も有名なプロバイダーや最上位のプランを選ぶことです。しかし私たちの目的は最も高価なプロバイダーを見つけることではありませんでした。目的は**最適な選択肢**を見つけること——実際のタスクで最高スコアを出し、ユーザーが使う言語で正しく動作するプロバイダーを特定することでした。

知名度は品質を保証しません。プレミアムというラベルも同様です。私たちのテストでは、80 億パラメータのモデルが GPT-4o-mini を 10 ポイント上回りました。LLM ベースの翻訳が DeepL と同等の品質スコアを記録しました。テストで 1 位になったプロバイダーは、多くの場合、予想外の名前でした。

そこで私たちは、まずすべてをテストし、その結果データだけに基づいてルーティングを設計するという方針を取りました。ブランド名でも、料金プランでも、思い込みでもなく、データだけで判断します。

## テスト方法：4 ラウンド、31 プロバイダー

4 段階の試験システムを設計しました。各ラウンドは、他のラウンドでは検出できない問題を捕捉します。

| Round | The Question | What It Revealed |
|-------|-------------|-----------------|
| **P1 — Connectivity** | Is this API even alive? | 3 out of 30 providers were dead on arrival |
| **P2 — Capability** | What can it do? (tested in EN/CN/JP) | Same API scores 100 in English, 30 in Chinese |
| **P3 — Quality** | Who's the best at each task? | 8B model scores higher than GPT-4o-mini |
| **P4 — Stability** | Can it handle 100 calls in a row? | #1 quality scorer had the worst reliability |

**4 ラウンドすべてが必要です。** もし P3 で止めていたら、中国語では DeepSeek を選んでいたでしょう（品質スコア最高）。しかし本番環境でレートリミットに引っかかっていたはずです。P4 がそれを検出しました。

すべてのテストスクリプトと生データ：**[washin-api-benchmark](https://github.com/sstklen/washin-api-benchmark)**

## ルーティング設計を根本から変えた発見

P2 では、すべての LLM に対して同一のプロンプトを 3 言語でテストしました。その結果は、私たちのアプローチ全体を大きく変えるものでした。

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

同じ API、同じモデル、同じ呼び出しで 70 ポイントの品質崩壊が発生します。しかもレスポンスは正常な `200 OK` と整形された JSON を返すため、監視では完全に見えません。

**これが示した教訓：すべての言語に同じルーティングは使えません。** 中国語ユーザー向けのエージェントには、英語ユーザー向けとはまったく異なるプロバイダーチェーンが必要です。これを私たちは **language-aware routing** と呼んでいます。そして現時点で、既存の API ゲートウェイ（OpenRouter、Portkey、LiteLLM）はこの仕組みを実装していません。

## LLM 総合ランキング

以下は P3 品質試験の結果です。LLM ルーティングの基盤となるデータです。

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

注目すべき点が 3 つあります。
- **Cerebras 8B（92）が GPT-4o-mini（82）を上回りました** — モデルサイズは極めて小さく、速度は 5 倍、スコアは 10 ポイント高い
- **GPT-4o-mini は推論で 30 点** — 基本的な算数の問題を間違えました
- **CN 列に注目してください** — Groq 30、Cerebras 30、それ以外は 100。このギャップこそ、私たちがルーティングで回避する対象です

## ルーティング設計に影響を与えた 4 つの追加発見

**LLM ベースの翻訳が専用翻訳 API に匹敵する：**

| Provider | Type | Score |
|----------|------|-------|
| Groq Translate | LLM-based | 94 |
| Cerebras Translate | LLM-based | 94 |
| DeepL | Dedicated engine | 93 |

これにより、LLM プロバイダーを翻訳のフォールバックとして利用できることがわかりました。ただし CJK 言語に関する注意点は同様に適用されます。

**サイレント障害は実在します：** 一部のプロバイダーは `200 OK` を返しながら `{"result": null}` や空のボディを返すことがあります。P4 でこれを検出しました。HTTP ステータスコードだけをチェックしていると、ユーザーに空のデータを渡すことになります。

**DeepSeek のパラドックス：** P3 で中国語の品質スコアは最高。P4 ではレートリミット障害が最も多い。品質と信頼性は独立した変数であり、両方のテストが不可欠です。

**検索エンジンはすべて 100 点：** Tavily、Brave、Serper はいずれも品質テストに合格しました。差別化要因はフォーマット（Tavily：AI 向けに最適化）、件数（Brave：10 件）、速度（Serper：537ms）でした。

---

## 10 のルーティング決定

以下のルーティング順序はすべて試験スコアに基づいています。先頭 = P3 最高スコア。フォールバック = 次点。CJK 入力の場合、スコア 50 未満のプロバイダーはスキップされます。

### 1. LLM — 最も複雑なルート

これは最も重要なルーティング決定であり、最も慎重な検討を要したものです。

**English routing** — 5 プロバイダーすべてが利用可能：
```
Gemini(93) → Mistral(87) → Cerebras(92) → Groq(83) → Cohere(78)
```

**Chinese/Japanese routing** — Cerebras(CN:30) と Groq(CN:30) をスキップ：
```
Gemini(93) → Mistral(87) → Cohere(78)
```

Cerebras が 92 点であるにもかかわらず Gemini を先頭にした理由は、Gemini が 3 言語すべてで 100 点を記録しているためです。**まず言語をまたいだ品質の一貫性を最適化し**、次に速度を考慮します。

### 2. Search — 関連性優先、件数はフォールバック

```
Tavily(Q100, 5 results, best relevance)
  → Brave Search(Q100, 10 results, widest coverage)
  → Gemini Grounding(last resort)
  → then: Groq generates AI summary of results
```

3 つとも品質スコアは 100 です。Tavily を先頭に置いた理由は、AI 向けに最適化されたフォーマットにより、LLM に渡した際の下流での結果が最も良いためです。

### 3. Translation — DeepL がリード、LLM がバックアップ

```
DeepL(Q93) → Groq(Q94) → Gemini(Q93) → Cerebras(Q94) → Mistral
CJK targets: skip Groq and Cerebras (same language blind spot as LLM)
```

DeepL を先頭にした理由はプロフェッショナルな一貫性です。LLM 翻訳も同等のスコアを出しますが、CJK 言語のギャップを引き継ぎます。

### 4. Web Reading — 4 段階で 4 種類のページに対応

```
Jina Reader → Firecrawl → ScraperAPI → Apify
```

| Level | Handles | Fails On |
|-------|---------|----------|
| Jina | ~70% of pages, fast | JS-heavy SPAs |
| Firecrawl | JS rendering | Some anti-bot sites |
| ScraperAPI | Broad coverage, proxy rotation | Less clean output |
| Apify | Most resilient edge cases | Slowest |

各レベルが前のレベルで対応できなかったケースを補完します。

### 5. Embedding — 多言語品質を優先

```
Cohere(best multilingual vectors) → Gemini → Jina
```

### 6. Speech-to-Text — 速度 vs 機能

```
Deepgram Nova-2(faster) → AssemblyAI(more features)
```

### 7. Text-to-Speech — 単一プロバイダー（最大の弱点）

```
ElevenLabs
```

テストにおいて同等の代替プロバイダーが見つかりませんでした。フォールバックはゼロです。新しいプロバイダーの評価を継続中です。

### 8. Geocoding — カバレッジから精度へ

```
Nominatim(broadest coverage) → OpenCage(best formatting) → Mapbox(most accurate)
```

### 9. News — 専用インデックスから汎用検索へ

```
NewsAPI(dedicated news index) → Web Search fallback
```

### 10. Structured Extraction — 2 段階パイプライン

```
Web Reader(URL → clean text) → LLM(text → structured JSON per your schema)
```

これはフォールバックチェーンではなく、パイプラインです。Web Reader は 4 段階フォールバック（#4）を使用し、LLM は language-aware routing（#1）を使用します。

---

## まとめ：10 ルートの全体像

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

この表で気づくことがあるはずです。**各カテゴリーの第 1 候補は、最も有名なプロバイダーでも最も高価なプロバイダーでもありません。** GPT ではなく Gemini。Google ではなく Tavily。OpenAI Embeddings ではなく Cohere。Mapbox ではなく Nominatim。すべての第 1 候補は評判ではなく、試験スコアによって決定されました。その結果、実際に最も良く動作するものに最適化されたルーティングテーブルが完成しました——プレゼン資料で見栄えの良いものではなく。

---

## 実装：Language-Aware Fallback

10 のルートすべてに共通するパターンは以下の通りです。

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

**信頼性の計算：**
- プロバイダー 1 社：稼働率約 99%（月 7 時間以上のダウンタイム）
- プロバイダー 3 社：稼働率約 99.97%（月約 2 分のダウンタイム）
- プロバイダー 5 社：稼働率約 99.9999%

先月 Groq が 30 分間ダウンした際、ルーティングは自動的に Gemini に切り替わりました。エージェント側ではダウンタイムはゼロでした。

---

## 5 つの教訓

**1. 英語のみのベンチマークは誤解を招きます。** エージェントが実際に使用する言語でテストしてください。ランキングは完全に入れ替わります。

**2. ステータスコードだけでなく、レスポンスボディを検証してください。** `200 OK` でも中身が空というサイレント障害が存在します。必ずデータの構造を確認しましょう。

**3. 品質と信頼性は別物です。** P3（品質）と P4（安定性）の両方を実施してください。最高スコアのプロバイダーが最も不安定な場合があります。

**4. 言語検出のコストは 0.01ms です。** Unicode 範囲の正規表現だけで済みます。これにより、中国語を 30 点のプロバイダーにルーティングすることを防げます。

**5. 試験は毎月再実施してください。** プロバイダーは変化し、モデルは更新されます。先月の 1 位が今月の最下位になることもあります。

---

## テスト手法とデータ

- **31 プロバイダー**をテスト、4 ラウンド、3 言語（EN/CN/JP）
- **東京からテスト**（AWS ap-northeast-1）
- **オープンソース** — スクリプト、生データ、採点基準を公開

Benchmark repo: [github.com/sstklen/washin-api-benchmark](https://github.com/sstklen/washin-api-benchmark)
Interactive report: [api.washinmura.jp/api-benchmark](https://api.washinmura.jp/api-benchmark/en/)

---

*[Washin Village](https://washinmura.jp) チームより公開 — 28 匹の犬猫、1 つの API インフラチーム、房総半島から毎月ベンチマークをお届けします。*
