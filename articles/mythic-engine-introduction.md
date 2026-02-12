---
title: "LLMの「なぜその答え？」に答える推論エンジンをRustで作った"
emoji: "🔍"
type: "tech"
topics: ["rust", "llm", "ai", "axum", "推論"]
published: false
---

## はじめに

ChatGPTやClaudeに質問すると、もっともらしい答えが返ってきます。しかし、こう思ったことはないでしょうか。

**「その答え、何を根拠に言っているの？」**

LLMは回答の根拠が不透明です。ハルシネーション（もっともらしい嘘）を見抜くには、人間が事後的に検証するしかありません。

この問題を解決するために、**Mythic Inference Engine** を作りました。
※公開準備中

https://github.com/cosmopanda432/mythic-engine

## Mythic Engine とは

GPU不要・決定論的・監査可能な推論エンジンです。LLMを「回答生成器」ではなく**「決裁機関」**として扱い、全ての回答に**根拠（Evidence）**と**判断過程（Audit Trail）**を付与します。

```
"日本の首都は？"
    ↓
[Mythic Engine]
    ↓
Decision: ASSERT (信頼度95%)
Response: "日本の首都は東京です。"
Evidence: geo-001 (Knowledge Base, score: 0.80)
Audit:    S1→S2→S3→...→S10(ASSERT)→S12→S13
```

従来のLLM APIとの違いを表にまとめます。

| | 従来のLLM API | Mythic Engine |
|---|---|---|
| 根拠 | 不透明 | Evidence付きで明示 |
| 判断過程 | ブラックボックス | 13状態マシンで全記録 |
| 再現性 | 温度パラメータに依存 | JCS+SHA-256で決定論的ハッシュ |
| 判断の種類 | 「回答」のみ | Assert/Hedge/Ask/Refuse/Defer |
| GPU | 必須 | 不要（RustコアはCPUのみで動作） |

## なぜ作ったのか

LLMは便利ですが、**業務判断**に使おうとすると壁にぶつかります。

- 「なぜこの判断をしたのか」を説明できない
- 判断の正しさを第三者が検証できない
- 同じ質問でも毎回違う答えが返る可能性がある

医療、法務、金融などの領域では「答えが正しいかどうか」以上に「なぜその答えに至ったか」が重要です。Mythic Engineはこの「なぜ」を構造化して記録します。

## RAGや既存OSSとの違い

### RAGとはレイヤーが違う

RAG（Retrieval-Augmented Generation）は「関連情報を検索してLLMに渡す」仕組みで、回答の品質を上げる手段です。Mythic Engineは、その回答に対して「本当に断言してよいのか」を判断し記録する**決裁・監査の仕組み**です。

```
RAG:           質問 → 検索 → LLMに渡す → 回答
Mythic Engine: 質問 → 検索 → 証拠評価 → 判断（5種類） → 監査ログ付き回答
```

競合ではなく、RAGの上に組み合わせられる関係です。

### 既存OSSとの比較

LLMの出力を制御するOSSとしては Guardrails AI や NeMo Guardrails が知られていますが、それぞれ目的が異なります。

| 特徴 | Guardrails AI | NeMo Guardrails | Mythic Inference Engine |
|------|---------------|-----------------|------------------------|
| 主目的 | 構造検証 (JSON, SQL) | 対話制御 (安全性, 逸脱防止) | 説明責任・監査 (根拠, ログ) |
| 制御対象 | 出力フォーマット・データ品質 | 会話の流れ・トピック | 推論プロセスそのもの |
| アプローチ | バリデータによる事後チェック | ガイドレールによる誘導 | ステートマシンによる決定論的処理 |
| 判断の根拠 | ルール適合性 | セーフティポリシー | Evidence (証拠) |
| 実装言語 | Python | Python | Rust (GPU不要・バイナリ) |

Guardrails AI は「出力が正しいフォーマットか」、NeMo Guardrails は「会話が安全な範囲に収まっているか」をチェックします。Mythic Engine は「**なぜその答えに至ったか**」を証拠と共に記録し、判断に自信がなければ断言しないという、**説明責任と監査可能性**にフォーカスしています。

## アーキテクチャ

### 全体像

Mythic EngineはRust製のシングルバイナリです。v0.12でPython依存を完全に排除し、LLMとの通信もRustから直接行います。

```
┌─────────────────────────────────────────────────┐
│           Mythic Engine (Rust / axum)            │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │  API Layer                                 │  │
│  │  REST / SSE / JSON-RPC 2.0                 │  │
│  └────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────┐  │
│  │  13-State Machine (推論コア)                │  │
│  │  Receive → Freeze → ... → Decide → Emit   │  │
│  └────────────────────────────────────────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌───────────────┐   │
│  │ Evidence  │ │ LLM      │ │ Audit Log     │   │
│  │ Retrieval │ │ Provider │ │ (SQLite)      │   │
│  └──────────┘ └──────────┘ └───────────────┘   │
└─────────────────────────────────────────────────┘
         │              │
         ▼              ▼
   Knowledge Base   LLM APIs
   (JSON)           (vLLM/Gemini/OpenAI/Claude/Ollama/DeepSeek)
```

### 13-State Machine

Mythic Engineの心臓部は13の状態を持つステートマシンです。入力から出力まで、全てのステップが明示的に定義されています。

```
S1:  Receive           入力を受け取る
S2:  Freeze            JCS正規化 + SHA-256ハッシュで入力を固定
S3:  Translator        自然言語 → IntentEnvelope（構造化された意図）に変換
S4:  Taboo             禁忌ルールチェック（危険な質問をブロック）
S5:  Sanity            入力品質チェック（曖昧すぎないか等）
S6:  InvocationDecide  どのプロバイダを呼ぶか決定
S7:  Invoke            証拠収集（Knowledge Base / Web検索 / Vector検索）
S8:  Check             収集結果の検証
S9:  Merge             複数ソースの証拠を統合
S10: Decide            証拠に基づき判断（Assert/Hedge/Ask/Refuse/Defer）
S11: Genealogy         証拠の系譜グラフを構築
S12: Emit              結果を出力
S13: End               処理完了
```

ポイントは **S2: Freeze** です。入力をJSON Canonicalization Scheme (JCS, RFC 8785) で正規化し、SHA-256でハッシュ化することで、同じ入力に対して常に同じハッシュが得られます。これにより**再現性と改ざん検知**が可能になります。

### 5つの判断タイプ

LLMは「回答する」か「回答しない」の二択ですが、Mythic Engineは5種類の判断を下します。

| 判断 | 意味 | いつ使われるか |
|------|------|----------------|
| **Assert** | 断言 | 信頼度80%以上の証拠がある |
| **Hedge** | 推測 | 証拠はあるが信頼度が50-80% |
| **Ask** | 質問 | 情報が足りないので聞き返す |
| **Refuse** | 拒否 | 禁忌ルールに抵触 |
| **Defer** | 保留 | 証拠が不十分で判断できない |

これにより、「自信がないのに断言する」というLLMの典型的な問題を構造的に防ぎます。

### 独自Knowledge Base（辞書）

Mythic Engineが回答の根拠として参照する知識ベースは、自由に追加できます。社内ナレッジや製品情報など、独自の辞書を構築してEvidence（証拠）として活用できます。

CSVで辞書を作成し、付属の変換ツールでJSONに変換するのが最も簡単です。

```csv
id,content,source_uri
product-001,"製品Aは月額1000円で利用できます。",料金表
policy-001,"返品は購入から30日以内に受け付けます。",https://example.com/policy
faq-001,"営業時間は9:00〜18:00です。",FAQ
```

```bash
# CSV → JSON変換（バリデーション付き）
python tools/csv2knowledge.py knowledge/my_data.csv

# バリデーションのみ（変換せず確認だけ）
python tools/csv2knowledge.py knowledge/my_data.csv --validate-only
```

変換ツールはUTF-8 BOMの自動除去、重複IDの検出、空行のスキップなど、実運用で起きがちな問題を自動的に処理します。

生成されたJSONを起動時に指定するだけで、エンジンがキーワードマッチやベクトル検索で関連エントリを見つけ出し、回答の証拠として提示します。

```bash
./target/release/mythic-engine serve \
  --knowledge ./knowledge/my_data.json
```

## 技術スタック

| 領域 | 技術 |
|------|------|
| 言語 | Rust (Edition 2021) |
| Webフレームワーク | axum 0.7 |
| 非同期ランタイム | tokio |
| シリアライズ | serde + serde_json |
| ハッシュ | serde_jcs (JCS) + sha2 (SHA-256) |
| LLM統合 | genai crate |
| DB | SQLite (Diesel ORM) |
| ストリーミング | Server-Sent Events (SSE) |

## 使ってみる

### セットアップ

```bash
git clone https://github.com/cosmopanda432/mythic-engine.git
cd mythic-engine
cargo build --release
```

### LLMプロバイダの設定

`config/server.yaml` でLLMプロバイダを選択します。ローカルLLM（vLLM, Ollama）とクラウドAPI（Gemini, OpenAI, Claude, DeepSeek）の両方に対応しています。

```yaml
llm:
  provider: gemini  # vllm | gemini | openai | claude | ollama | deepseek
  gemini:
    model: "gemini-2.0-flash"
```

```bash
# Geminiの場合
export GEMINI_API_KEY=your-api-key
```

### サーバー起動

```bash
./target/release/mythic-engine serve \
  --config ./config/server.yaml \
  --knowledge ./knowledge/demo.json \
  --rules ./rules
```

ブラウザで `http://localhost:8082/` を開くとWeb UIが使えます。

### APIで問い合わせ

```bash
curl -X POST http://localhost:8082/api/v1/ask \
  -H "Content-Type: application/json" \
  -d '{"text": "日本の首都は？"}'
```

レスポンスには判断・証拠・監査ログが含まれます。

```json
{
  "success": true,
  "decision": {
    "decision_type": "assert",
    "confidence": 0.95
  },
  "generated_response": "日本の首都は東京です。",
  "evidence": [
    {
      "id": "geo-001",
      "source": "knowledge",
      "score": 0.80,
      "content": "日本の首都は東京である"
    }
  ],
  "trace_id": "req-abc123"
}
```

SSEストリーミングにも対応しており、リアルタイムで状態遷移を確認できます。

```bash
curl -N -X POST http://localhost:8082/api/v1/ask/stream \
  -H "Content-Type: application/json" \
  -d '{"text": "日本の首都は？"}'
```

## 設計上のこだわり

### 1. LLMを「決裁機関」として扱う

Mythic EngineはLLMに「答えを生成させる」のではなく、集めた証拠に基づいて「判断を下す」役割を与えています。LLMは裁判官のように、証拠を見て「断言できる」「推測にとどまる」「判断を保留する」といった決裁を行います。

### 2. GPU不要

推論エンジンのコア（13状態マシン、証拠照合、判断ロジック）はRustで書かれており、GPUは不要です。LLMの呼び出しは外部API経由で行うため、エンジン自体はCPUのみで動作します。

### 3. 全過程を監査可能

全ての状態遷移がSQLiteに記録されます。「いつ、どの証拠を使って、なぜその判断に至ったか」を事後的に追跡できます。

### 4. Python依存の排除

v0.11まではPython製のMCPサーバーがLLMとの仲介役でしたが、v0.12で `genai` crateを導入し、Rust単体で完結するアーキテクチャに移行しました。デプロイがシングルバイナリで済むようになり、運用が大幅に簡素化されました。

## 開発の経緯

| Version | マイルストーン |
|---------|---------------|
| v0.1 | 13状態マシンとAudit Trailの基盤 |
| v0.2-v0.5 | 証拠取得層（ファイル検索、ベクトル検索） |
| v0.6 | Web UI + HTTP API |
| v0.7-v0.8 | Web検索統合、Evidence Ranking |
| v0.9-v0.10 | LLMプロバイダ統一、SSEストリーミング |
| v0.11 | JSON-RPC 2.0、API Key認証 |
| v0.12 | Python MCP Server廃止 → Rust直接統合 |
| v0.13 | SQLite監査ログ (Diesel ORM) |

13バージョンで、「ステートマシンだけの実験コード」から「Web UI・複数LLM・ストリーミング・監査ログを備えた推論エンジン」に育ちました。

## 今後の展望

- **Evidence Graph**: 証拠間の関係性をグラフ構造で表現し、弁証法的な推論を可能にする
- **v1.0安定版**: ドキュメント整備とAPIの安定化

## まとめ

Mythic Inference Engineは「LLMに根拠と説明責任を持たせる」ための推論エンジンです。

- 全ての回答に**証拠（Evidence）**を紐付け
- 全ての判断を**13状態マシン**で構造化
- 全ての過程を**監査ログ**に記録
- **5種類の判断**で「自信がないのに断言する」問題を防止
- GPU不要、Rustシングルバイナリで動作

「LLMの出力をそのまま信じるのは怖い」と感じている方、ぜひ試してみてください。

https://github.com/cosmopanda432/mythic-engine

フィードバックやIssue、Pull Request歓迎です！

## 謝辞

Mythic Engineは以下のOSSに支えられています。

- [axum](https://github.com/tokio-rs/axum) - Webフレームワーク
- [genai](https://github.com/jeremychone/rust-genai) - マルチLLMプロバイダ
- [Diesel](https://diesel.rs/) - ORM & クエリビルダ
- [serde](https://serde.rs/) - シリアライズ
- [Tavily](https://tavily.com/) - Web検索API
