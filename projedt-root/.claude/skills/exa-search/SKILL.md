---
name: exa-search
description: Exa MCPによるニューラル検索。ウェブ、コード、企業リサーチに対応。ウェブ検索、コード例、企業インテリジェンス、人物検索、AIによるディープリサーチが必要な場合に使用。
origin: ECC
---

# Exa Search

Exa MCPサーバーを介したウェブコンテンツ、コード、企業、人物のニューラル検索。

## アクティベーション条件

- 最新のウェブ情報やニュースが必要な場合
- コード例、APIドキュメント、技術リファレンスの検索
- 企業、競合他社、マーケットプレイヤーの調査
- 特定ドメインの専門家やプロフェッショナルの検索
- 開発タスクのバックグラウンドリサーチ
- ユーザーが「検索して」「調べて」「探して」「最新情報は」と言った場合

## MCP要件

Exa MCPサーバーの設定が必要です。`~/.claude.json`に追加：

```json
"exa-web-search": {
  "command": "npx",
  "args": ["-y", "exa-mcp-server"],
  "env": { "EXA_API_KEY": "YOUR_EXA_API_KEY_HERE" }
}
```

APIキーは[exa.ai](https://exa.ai)で取得できます。
このリポジトリの現在のExa設定では、`web_search_exa`と`get_code_context_exa`のツールサーフェスが公開されています。
Exaサーバーが追加ツールを公開している場合は、ドキュメントやプロンプトで依存する前に正確な名前を確認してください。

## コアツール

### web_search_exa
最新情報、ニュース、事実のための一般的なウェブ検索。

```
web_search_exa(query: "latest AI developments 2026", numResults: 5)
```

**パラメータ：**

| パラメータ | 型 | デフォルト | 備考 |
|-------|------|---------|-------|
| `query` | string | 必須 | 検索クエリ |
| `numResults` | number | 8 | 結果の数 |
| `type` | string | `auto` | 検索モード |
| `livecrawl` | string | `fallback` | 必要に応じてライブクロールを優先 |
| `category` | string | なし | `company`や`research paper`などのオプションフォーカス |

### get_code_context_exa
GitHub、Stack Overflow、ドキュメントサイトからコード例やドキュメントを検索。

```
get_code_context_exa(query: "Python asyncio patterns", tokensNum: 3000)
```

**パラメータ：**

| パラメータ | 型 | デフォルト | 備考 |
|-------|------|---------|-------|
| `query` | string | 必須 | コードまたはAPI検索クエリ |
| `tokensNum` | number | 5000 | コンテンツトークン数（1000-50000） |

## 使用パターン

### クイックルックアップ
```
web_search_exa(query: "Node.js 22 new features", numResults: 3)
```

### コードリサーチ
```
get_code_context_exa(query: "Rust error handling patterns Result type", tokensNum: 3000)
```

### 企業・人物リサーチ
```
web_search_exa(query: "Vercel funding valuation 2026", numResults: 3, category: "company")
web_search_exa(query: "site:linkedin.com/in AI safety researchers Anthropic", numResults: 5)
```

### 技術ディープダイブ
```
web_search_exa(query: "WebAssembly component model status and adoption", numResults: 5)
get_code_context_exa(query: "WebAssembly component model examples", tokensNum: 4000)
```

## ヒント

- 最新情報、企業検索、幅広い発見には`web_search_exa`を使用
- `site:`、引用フレーズ、`intitle:`などの検索演算子で結果を絞り込む
- フォーカスしたコードスニペットには低い`tokensNum`（1000-2000）、包括的なコンテキストには高い値（5000+）を使用
- 一般的なウェブページよりもAPI使用法やコード例が必要な場合は`get_code_context_exa`を使用

## 関連スキル

- `deep-research` — firecrawl + exaを併用した完全なリサーチワークフロー
- `market-research` — 意思決定フレームワークを備えたビジネス指向リサーチ
