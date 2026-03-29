---
name: search-first
description: コーディング前のリサーチワークフロー。カスタムコードを書く前に既存のツール、ライブラリ、パターンを検索する。リサーチャーエージェントを起動。
origin: ECC
---

# /search-first — コードを書く前にリサーチ

「実装前に既存のソリューションを検索する」ワークフローを体系化します。

## トリガー

以下の場合にこのスキルを使用します：
- 既存のソリューションがある可能性の高い新機能を開始するとき
- 依存関係やインテグレーションを追加するとき
- ユーザーが「X機能を追加して」と言い、コードを書こうとしているとき
- 新しいユーティリティ、ヘルパー、抽象化を作成する前

## ワークフロー

```
┌─────────────────────────────────────────────┐
│  1. NEED ANALYSIS                           │
│     Define what functionality is needed      │
│     Identify language/framework constraints  │
├─────────────────────────────────────────────┤
│  2. PARALLEL SEARCH (researcher agent)      │
│     ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│     │  npm /   │ │  MCP /   │ │  GitHub / │  │
│     │  PyPI    │ │  Skills  │ │  Web      │  │
│     └──────────┘ └──────────┘ └──────────┘  │
├─────────────────────────────────────────────┤
│  3. EVALUATE                                │
│     Score candidates (functionality, maint, │
│     community, docs, license, deps)         │
├─────────────────────────────────────────────┤
│  4. DECIDE                                  │
│     ┌─────────┐  ┌──────────┐  ┌─────────┐  │
│     │  Adopt  │  │  Extend  │  │  Build   │  │
│     │ as-is   │  │  /Wrap   │  │  Custom  │  │
│     └─────────┘  └──────────┘  └─────────┘  │
├─────────────────────────────────────────────┤
│  5. IMPLEMENT                               │
│     Install package / Configure MCP /       │
│     Write minimal custom code               │
└─────────────────────────────────────────────┘
```

## 判断マトリクス

| シグナル | アクション |
|--------|--------|
| 完全一致、メンテナンス良好、MIT/Apacheライセンス | **Adopt** — インストールして直接使用 |
| 部分一致、良い基盤 | **Extend** — インストール＋薄いラッパーを作成 |
| 複数の弱い一致 | **Compose** — 2-3個の小さなパッケージを組み合わせ |
| 適切なものなし | **Build** — カスタムコードを書くが、リサーチに基づいて |

## 使い方

### クイックモード（インライン）

ユーティリティを書いたり機能を追加する前に、以下を頭の中で実行します：

0. リポジトリに既に存在するか？ → 関連モジュール/テストを`rg`で検索
1. 一般的な問題か？ → npm/PyPIを検索
2. これ用のMCPがあるか？ → `~/.claude/settings.json`を確認して検索
3. これ用のスキルがあるか？ → `~/.claude/skills/`を確認
4. GitHubの実装/テンプレートがあるか？ → 新規コードを書く前にメンテナンスされているOSSのGitHubコード検索を実行

### フルモード（エージェント）

重要な機能については、リサーチャーエージェントを起動します：

```
Task(subagent_type="general-purpose", prompt="
  Research existing tools for: [DESCRIPTION]
  Language/framework: [LANG]
  Constraints: [ANY]

  Search: npm/PyPI, MCP servers, Claude Code skills, GitHub
  Return: Structured comparison with recommendation
")
```

## カテゴリ別検索ショートカット

### 開発ツール
- リンティング → `eslint`, `ruff`, `textlint`, `markdownlint`
- フォーマッティング → `prettier`, `black`, `gofmt`
- テスティング → `jest`, `pytest`, `go test`
- プリコミット → `husky`, `lint-staged`, `pre-commit`

### AI/LLMインテグレーション
- Claude SDK → Context7で最新ドキュメント
- プロンプト管理 → MCPサーバーを確認
- ドキュメント処理 → `unstructured`, `pdfplumber`, `mammoth`

### データ＆API
- HTTPクライアント → `httpx`（Python）、`ky`/`got`（Node）
- バリデーション → `zod`（TS）、`pydantic`（Python）
- データベース → まずMCPサーバーを確認

### コンテンツ＆パブリッシング
- Markdownの処理 → `remark`, `unified`, `markdown-it`
- 画像最適化 → `sharp`, `imagemin`

## 統合ポイント

### plannerエージェントとの統合
plannerはフェーズ1（アーキテクチャレビュー）の前にresearcherを呼び出すべきです：
- Researcherが利用可能なツールを特定
- Plannerがそれらを実装計画に組み込む
- 計画段階での「車輪の再発明」を回避

### architectエージェントとの統合
architectは以下のためresearcherに相談すべきです：
- 技術スタック決定
- インテグレーションパターンの発見
- 既存のリファレンスアーキテクチャ

### iterative-retrievalスキルとの統合
段階的な発見のために組み合わせます：
- サイクル1：広範検索（npm、PyPI、MCP）
- サイクル2：トップ候補の詳細評価
- サイクル3：プロジェクト制約との互換性テスト

## 例

### 例 1: 「デッドリンクチェックを追加」
```
Need: Check markdown files for broken links
Search: npm "markdown dead link checker"
Found: textlint-rule-no-dead-link (score: 9/10)
Action: ADOPT — npm install textlint-rule-no-dead-link
Result: Zero custom code, battle-tested solution
```

### 例 2: 「HTTPクライアントラッパーを追加」
```
Need: Resilient HTTP client with retries and timeout handling
Search: npm "http client retry", PyPI "httpx retry"
Found: got (Node) with retry plugin, httpx (Python) with built-in retry
Action: ADOPT — use got/httpx directly with retry config
Result: Zero custom code, production-proven libraries
```

### 例 3: 「設定ファイルリンターを追加」
```
Need: Validate project config files against a schema
Search: npm "config linter schema", "json schema validator cli"
Found: ajv-cli (score: 8/10)
Action: ADOPT + EXTEND — install ajv-cli, write project-specific schema
Result: 1 package + 1 schema file, no custom validation logic
```

## アンチパターン

- **コードにすぐ着手**: 既に存在するか確認せずにユーティリティを書く
- **MCPの無視**: MCPサーバーが既にその機能を提供しているか確認しない
- **過度なカスタマイズ**: ライブラリをあまりにも厚くラップしてメリットを失う
- **依存関係の肥大化**: 小さな機能のために巨大なパッケージをインストールする
