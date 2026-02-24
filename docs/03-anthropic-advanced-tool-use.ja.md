**Language:** [English](03-anthropic-advanced-tool-use.md) | [繁體中文](03-anthropic-advanced-tool-use.zh.md) | 日本語

# トークン 76% 削減、コスト 96% 削減、4.6 倍高速化——Anthropic の Tool Use 論文を読み、当日 4 commits で実装

> Anthropic が [Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use) を発表し、3 つの機能を提案しました。私たちは日本の田舎で 39 サービスの API プラットフォームを運営しています。論文を読んだ当日にすべて実装——さらに、彼らがやっていない 4 つのことも追加しました。

---

## 全体像

| Anthropic の機能                | 彼らのデータ                | 私たちがやったこと                       |
| ----------------------------- | ---------------------- | ---------------------------------- |
| **Tool Search**               | 77K→8.7K tokens (85%↓) | 10.8KB→2.5KB、オンデマンド読み込み (**76%↓**) |
| **Tool Use Examples**         | 精度 72%→90%             | 11 エンドポイントに実データの JSON サンプルを追加    |
| **Programmatic Tool Calling** | tokens 37%↓            | PTC モード：**$0.02 vs $0.49、96% 削減** |

| 私たちが追加したもの             | 概要                              |
| ----------------------- | -------------------------------- |
| **フォールバックチェーン** (L2)   | 正しいツールを選んでも落ちる。4 層のフォールバックで受け止める |
| **試験ルーティング** (P1-P4)   | 継続的な試験がプロバイダーランキングを駆動            |
| **インテントルーティング** (L3)   | 自然言語で話すだけ、システムが自動でツールを選択          |
| **品質シグナル** (L4)         | 機械が読める結果品質スコア                    |

---

## 背景：ペインポイントが生んだ 4 層アーキテクチャ

```
L1  Proxy        — 純粋な転送（27 エンドポイント）       $0-$0.01
L2  Smart Gateway — マルチプロバイダーフォールバック＋戦略ルーティング  $0.006-$0.009
L3  Concierge    — 自然言語 → 自動ツール選択           $0.02
L4  Task Engine  — 計画 → 実行 → 品質評価            $0.49-$2.99
```

ホワイトボードにアーキテクチャ図を描いてから作ったのではありません。L1 は単純な転送。L2 は Brave が 2 時間ダウンし、全クライアントが 500 エラーを受け取った後に追加したフォールバックチェーン。L3 は Agent が search と news のどちらを呼ぶべきかわからず、日本語への翻訳が必要なことにも気づけなかったために追加したインテントルーティング。L4 は「3 つの翻訳 API の品質を比較する」ような複数ステップのタスクが処理できなかったために追加したもの。すべての層に、それぞれ具体的なペインポイントがあります。

---

## 比較：Anthropic の 3 機能

### Tool Search → defer_loading（76% 削減）

> *"With 200+ tools, the traditional approach consumed approximately 77K input tokens before any actual work began... With the Tool Search Tool, initial token consumption drops to approximately 8.7K."*

私たちの `/api/capabilities` は一度に 10.8KB を返していました。論文を読んだ当日、2 層に分割しました。

```bash
GET /api/services/brief     # 2.5KB メニュー
GET /api/services/{id}      # ~300B オンデマンド
```

```json
// GET /api/services/brief レスポンス（抜粋）
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

| 変更前                    | 変更後                 | 削減幅    |
| ---------------------- | ------------------- | ------- |
| 10.8KB (~2,700 tokens) | 2.5KB (~640 tokens) | **76%** |

Anthropic はモデル側で実装しています（Claude が内部で検索し、`defer_loading: true` を使用）。私たちは API 側で実装しました（Agent が必要なものを自分で読み込む、2 つのエンドポイント）。彼らのほうがエレガント。私たちのほうが直接的。原理は同じです：**まずメニュー、料理はあとから。**

**学んだこと：** Anthropic が問題に名前をつけてくれました（`defer_loading`）。影響を定量化してくれました（85%↓ + Opus 4 精度 49%→74%）。実装の方向性を示してくれました。「肥大化してるな」と感じることと、「具体的にどう直すか」がわかることは別物です。

---

### Tool Use Examples（Anthropic の計測で精度 +18%）

> *"Adding concrete examples to tool definitions improved accuracy from 72% to 90% on complex parameter handling."*

私たちのドキュメントには URL とパラメータはありましたが、サンプルがゼロでした。Agent がフォーマットを推測していました。

```
推測: {"search": "renewable energy Japan"}         ← フィールド名が間違い
正解: {"query": "renewable energy Japan", "strategy": "fast"}
```

当日、11 エンドポイントすべてに実データの JSON リクエスト＋レスポンスサンプルを追加しました。

```
## POST /api/v2/search
### リクエストサンプル：
{"query": "renewable energy Japan 2025", "strategy": "fast", "maxResults": 10}
### レスポンスサンプル：
{"results": [...], "provider": "brave", "responseTimeMs": 420, "cost": "$0.009"}
```

**数ヶ月かけて構築したフォールバックチェーンよりも、1 日で追加したサンプルのほうが効果的かもしれません——サンプルは誤呼び出しを根本から減らします。Agent はサポートに電話しません。ドキュメントを読むだけです。**

---

### Programmatic Tool Calling → PTC（コスト 96% 削減）

> *"Programmatic Tool Calling enables Claude to write and execute code that orchestrates multiple tool calls... On complex research tasks, this approach reduced average token usage from 43,588 to 27,297 — a 37% reduction."*

私たちの L4 にはすでに 3 つのフェーズ（計画→実行→品質評価）がありました。論文を読んで PTC を追加しました——Agent が自分でステップを持ち込み、LLM による計画フェーズをスキップします。

```json
// PTC Mode — Agent が実行計画を持ち込む
POST /api/v2/task
{"goal": "Search and summarize AI news", "steps": [
  {"toolId": "smart-search", "params": {"query": "AI agent news 2026"}},
  {"toolId": "smart-llm", "params": {"prompt": "Summarize"}, "dependsOn": [1]}
]}

// レスポンス
{"success": true, "mode": "ptc", "synthesis": "...",
 "meta": {"price": 0.02, "execution": [
   {"step": 1, "tool": "smart-search", "responseTimeMs": 1408},
   {"step": 2, "tool": "smart-llm",    "responseTimeMs": 1292}
 ], "totalTimeMs": 3979}}
```

| シナリオ   | Auto        | PTC         | 改善            |
| ------- | ----------- | ----------- | ------------- |
| 単一クエリ  | ~12s, $0.49 | 2.8s, $0.02 | 4 倍高速、96% 削減  |
| 検索+要約  | ~18s, $0.49 | 3.9s, $0.02 | 4.6 倍高速、96% 削減 |

Anthropic は Claude に Python を書かせてオーケストレーションします（柔軟性が高い）。私たちは Agent に JSON ステップを提出させます（より確実——各ステップに L2 フォールバックあり）。Agent はお金を払って結果を求めています。実行できるかわからない Python スクリプトを求めているのではありません。

---

## 私たちが追加した 4 つのこと

### フォールバックチェーン——正しいツールを選んでも落ちる

Anthropic は、正しいツールを選べば結果が得られると仮定しています。本番環境では、**正しいツールを選んでも落ちます。**

```
Agent が POST /api/v2/search を呼び出す
  → Brave (8s) → Tavily (10s) → Firecrawl (20s) → Gemini (20s)
  Agent はこのチェーンの存在を一切知りません。
```

**実際のインシデント：** ある日の午後、Brave が 6 分間ダウンしました。

```
14:25  Brave タイムアウト → フォールバック Tavily → 成功、1200ms
14:31  Brave 復旧。クライアントは何も気づかず。
```

フォールバックチェーンなし？ 6 分間の 500 エラー。フォールバックチェーンあり？ `provider` が `"brave"` から `"tavily"` に変わるだけで、結果は通常通り返ります。

---

### 試験ルーティング（P1-P4）——静的なサンプルは陳腐化する

Anthropic はサンプルの追加で精度が +18% 向上すると述べています。しかし、**静的なサンプルは陳腐化します。** 先月のベストプロバイダーが、今月は品質低下しているかもしれません。

| 試験     | テスト内容            | 頻度     | 駆動する対象            |
| ------ | ----------------- | ------ | ------------------ |
| **P1** | エンドポイントは生きているか？  | 6 時間毎  | 死んだプロバイダーを除外       |
| **P3** | 誰の結果が最も良いか？      | 毎週     | **L2 ルーティングランキングを駆動** |
| **P4** | 長期的に安定しているか？     | 毎月     | フォールバック順序を決定       |

**実際の発見：** P3 が自動的に、あるプロバイダーの日本語クエリの関連性が英語より 15% 高いことを検出しました。誰もこれを設計していません——試験データが自然に浮かび上がらせたのです。システムは自動で調整されました：日本語検索はプロバイダー A を優先、英語はプロバイダー B を優先。手書きのサンプルでは、このようなパターンは絶対に見つかりません。

---

### インテントルーティング（L3）——Agent はどのツールを呼ぶべきかわからない

Anthropic は呼び出し側がどのツールを使うべきか知っていると仮定しています。実際にはそうでないことが多いです。

```
「日本人は新幹線の延伸をどう思っている？」

L3 (<500ms)：検索(日本語) → 翻訳 → 要約。3 ステップ自動実行。

L3 なし：Agent が試行錯誤 → 4-5 回の呼び出し → $0.03-0.05、8-15 秒
L3 あり：自然言語 1 文     → 3 回の的確な呼び出し → $0.009、3-5 秒
```

インテント解析のコストは約 $0.0002/回。ROI：100〜200 倍。

---

### 品質シグナル（Phase 3）——結果が良いかどうかをどう判断するか

Anthropic は「実行前まで」のパスを最適化しています。しかし、**呼び出しの結果が返ってきた後は？**

```
Phase 3 評価 → overall: 0.49（結果が古すぎる）→ 自動リトライ → 0.83 ✅
```

品質スコアは**機械が読むためのもの**です。Agent は「全部信じる」か「全部信じない」かの二者択一をする必要はありません。**条件付きで信頼する**ことができます——0.83 なら十分、初稿の作成に使い、弱い箇所にフラグを立てて後で補足する。

---

## 哲学の違い

|           | Anthropic              | 和心村（わしんむら）            |
| --------- | ---------------------- | ---------------------- |
| **方向性**   | モデルをよりスマートにツールを選ばせる    | ツールが選ばれやすくなるようにする      |
| **制御権**   | モデル側（Claude が決定）       | API 側（Agent が決定）       |
| **スコープ**  | Claude のみ              | あらゆる LLM/Agent         |
| **価格設計**  | トークンコストに内包             | 各ステップ透明（金額＋時間）        |

**Anthropic は「モデルがどうツールを選ぶか」を最適化する。私たちは「ツールがどう選ばれやすくなるか」を最適化する。アプローチは違えど、行き着く先は同じです。**

---

## データ

| 指標             | 値                            |
| -------------- | ---------------------------- |
| サービス数          | 39（L1×27 + L2×10 + L3 + L4） |
| カテゴリ数          | 15                           |
| Defer Loading  | 10.8KB → 2.5KB（76%↓）        |
| PTC vs Auto    | $0.02 vs $0.49（96%↓）        |
| 試験サイクル         | P1 6 時間毎 / P3 毎週 / P4 毎月    |
| 開発期間           | 7 ヶ月、エンジニアリング経験**ゼロ**        |

---

方向性は正しかった——独立してたどり着いたアーキテクチャが、トップクラスの研究機関と一致していました。アプリケーション層には余地がある——フォールバックチェーン、試験ルーティング、品質シグナルは、モデル層では実現できないものです。

**論文を読んだ。学びがあった。当日実装した。**

---

**References** — [Anthropic Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use) (2025) · [Zero Engineer](https://github.com/sstklen/zero-engineer) · [112 Claude Code Skills](https://github.com/sstklen/washin-claude-skills) · [crawl-share](https://github.com/sstklen/crawl-share) · [Confucius Debug](https://github.com/sstklen/yanhui-ci)

```
ca35575  feat: input_examples — 11 エンドポイントに実データサンプルを追加
b31168c  feat: defer_loading — 軽量インデックス＋オンデマンド読み込み
9174e59  feat: dynamic filtering — 5 種類のフィルタリングパラメータ
8f4a50d  feat: PTC — L4 が Agent 提出の実行計画に対応
```

**午後のひとときで、4 commits。** *Built with 🦞 in Boso Peninsula, Japan.*
