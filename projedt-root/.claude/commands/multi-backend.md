# Backend - バックエンド特化型開発

バックエンド特化型ワークフロー（リサーチ → アイデア → プラン → 実行 → 最適化 → レビュー）、Codex主導。

## 使い方

```bash
/backend <backend task description>
```

## コンテキスト

- バックエンドタスク: $ARGUMENTS
- Codex主導、Geminiは補助参照用
- 適用対象: API設計、アルゴリズム実装、データベース最適化、ビジネスロジック

## あなたの役割

あなたは **バックエンドオーケストレーター** です。サーバーサイドタスクのためのマルチモデル協調を調整します（リサーチ → アイデア → プラン → 実行 → 最適化 → レビュー）。

**協調モデル**:
- **Codex** – バックエンドロジック、アルゴリズム（**バックエンドの権威、信頼性あり**）
- **Gemini** – フロントエンドの視点（**バックエンドの意見は参考程度**）
- **Claude (self)** – オーケストレーション、プランニング、実行、デリバリー

---

## マルチモデル呼び出し仕様

**呼び出し構文**:

```
# New session call
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend codex - \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Requirement: <enhanced requirement (or $ARGUMENTS if not enhanced)>
Context: <project context and analysis from previous phases>
</TASK>
OUTPUT: Expected output format
EOF",
  run_in_background: false,
  timeout: 3600000,
  description: "Brief description"
})

# Resume session call
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend codex resume <SESSION_ID> - \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Requirement: <enhanced requirement (or $ARGUMENTS if not enhanced)>
Context: <project context and analysis from previous phases>
</TASK>
OUTPUT: Expected output format
EOF",
  run_in_background: false,
  timeout: 3600000,
  description: "Brief description"
})
```

**ロールプロンプト**:

| フェーズ | Codex |
|-------|-------|
| 分析 | `~/.claude/.ccg/prompts/codex/analyzer.md` |
| プランニング | `~/.claude/.ccg/prompts/codex/architect.md` |
| レビュー | `~/.claude/.ccg/prompts/codex/reviewer.md` |

**セッション再利用**: 各呼び出しは `SESSION_ID: xxx` を返します。後続フェーズには `resume xxx` を使用してください。フェーズ2で `CODEX_SESSION` を保存し、フェーズ3と5で `resume` を使用します。

---

## コミュニケーションガイドライン

1. 応答はモードラベル `[Mode: X]` で開始、初期は `[Mode: Research]`
2. 厳格なシーケンスに従う: `Research → Ideation → Plan → Execute → Optimize → Review`
3. 必要に応じて `AskUserQuestion` ツールでユーザーとやり取り（確認/選択/承認など）

---

## コアワークフロー

### フェーズ0: プロンプト強化（オプション）

`[Mode: Prepare]` - ace-tool MCPが利用可能な場合、`mcp__ace-tool__enhance_prompt` を呼び出し、**元の$ARGUMENTSを強化結果に置換して後続のCodex呼び出しに使用**。利用不可の場合は `$ARGUMENTS` をそのまま使用。

### フェーズ1: リサーチ

`[Mode: Research]` - 要件の理解とコンテキストの収集

1. **コード取得**（ace-tool MCPが利用可能な場合）: `mcp__ace-tool__search_context` を呼び出して既存のAPI、データモデル、サービスアーキテクチャを取得。利用不可の場合は組み込みツールを使用: ファイル検出に `Glob`、シンボル/API検索に `Grep`、コンテキスト収集に `Read`、より深い探索に `Task`（Exploreエージェント）。
2. 要件完全性スコア (0-10): 7以上は続行、7未満は中止して補足

### フェーズ2: アイデア

`[Mode: Ideation]` - Codex主導の分析

**Codexを呼び出す必要あり**（上記の呼び出し仕様に従う）:
- ROLE_FILE: `~/.claude/.ccg/prompts/codex/analyzer.md`
- Requirement: 強化された要件（または未強化の場合は$ARGUMENTS）
- Context: フェーズ1のプロジェクトコンテキスト
- OUTPUT: 技術的実現可能性分析、推奨ソリューション（最低2つ）、リスク評価

**SESSION_ID** (`CODEX_SESSION`) を後続フェーズの再利用のために保存。

ソリューション（最低2つ）を出力し、ユーザーの選択を待つ。

### フェーズ3: プランニング

`[Mode: Plan]` - Codex主導のプランニング

**Codexを呼び出す必要あり**（セッション再利用に `resume <CODEX_SESSION>` を使用）:
- ROLE_FILE: `~/.claude/.ccg/prompts/codex/architect.md`
- Requirement: ユーザーが選択したソリューション
- Context: フェーズ2の分析結果
- OUTPUT: ファイル構造、関数/クラス設計、依存関係

Claudeがプランを統合し、ユーザーの承認後に `.claude/plan/task-name.md` に保存。

### フェーズ4: 実装

`[Mode: Execute]` - コード開発

- 承認されたプランに厳密に従う
- 既存プロジェクトのコード規約に準拠
- エラーハンドリング、セキュリティ、パフォーマンス最適化を確保

### フェーズ5: 最適化

`[Mode: Optimize]` - Codex主導のレビュー

**Codexを呼び出す必要あり**（上記の呼び出し仕様に従う）:
- ROLE_FILE: `~/.claude/.ccg/prompts/codex/reviewer.md`
- Requirement: 以下のバックエンドコード変更をレビュー
- Context: git diffまたはコード内容
- OUTPUT: セキュリティ、パフォーマンス、エラーハンドリング、APIコンプライアンスの問題リスト

レビューフィードバックを統合し、ユーザー確認後に最適化を実行。

### フェーズ6: 品質レビュー

`[Mode: Review]` - 最終評価

- プランに対する完了度をチェック
- テストを実行して機能を検証
- 問題と推奨事項を報告

---

## 重要なルール

1. **Codexのバックエンド意見は信頼できる**
2. **Geminiのバックエンド意見は参考程度**
3. 外部モデルは**ファイルシステムへの書き込みアクセスなし**
4. Claudeがすべてのコード書き込みとファイル操作を担当
