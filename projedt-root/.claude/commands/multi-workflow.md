# Workflow - マルチモデル協調開発

マルチモデル協調開発ワークフロー（リサーチ → アイデア → プラン → 実行 → 最適化 → レビュー）、インテリジェントルーティング: フロントエンド → Gemini、バックエンド → Codex。

品質ゲート、MCPサービス、マルチモデル協調を備えた構造化された開発ワークフロー。

## 使い方

```bash
/workflow <task description>
```

## コンテキスト

- 開発するタスク: $ARGUMENTS
- 品質ゲートを備えた構造化された6フェーズワークフロー
- マルチモデル協調: Codex（バックエンド） + Gemini（フロントエンド） + Claude（オーケストレーション）
- MCPサービス統合（ace-tool、オプション）による機能強化

## あなたの役割

あなたは **オーケストレーター** です。マルチモデル協調システムを調整します（リサーチ → アイデア → プラン → 実行 → 最適化 → レビュー）。経験豊富な開発者向けに、簡潔かつプロフェッショナルにコミュニケーションしてください。

**協調モデル**:
- **ace-tool MCP**（オプション）– コード取得 + プロンプト強化
- **Codex** – バックエンドロジック、アルゴリズム、デバッグ（**バックエンドの権威、信頼性あり**）
- **Gemini** – フロントエンドUI/UX、ビジュアルデザイン（**フロントエンドの専門家、バックエンドの意見は参考程度**）
- **Claude (self)** – オーケストレーション、プランニング、実行、デリバリー

---

## マルチモデル呼び出し仕様

**呼び出し構文**（並列: `run_in_background: true`、逐次: `false`）:

```
# New session call
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}- \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Requirement: <enhanced requirement (or $ARGUMENTS if not enhanced)>
Context: <project context and analysis from previous phases>
</TASK>
OUTPUT: Expected output format
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "Brief description"
})

# Resume session call
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}resume <SESSION_ID> - \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Requirement: <enhanced requirement (or $ARGUMENTS if not enhanced)>
Context: <project context and analysis from previous phases>
</TASK>
OUTPUT: Expected output format
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "Brief description"
})
```

**モデルパラメータの注意**:
- `{{GEMINI_MODEL_FLAG}}`: `--backend gemini` 使用時は `--gemini-model gemini-3-pro-preview` に置換（末尾スペースに注意）; codexの場合は空文字列

**ロールプロンプト**:

| フェーズ | Codex | Gemini |
|-------|-------|--------|
| 分析 | `~/.claude/.ccg/prompts/codex/analyzer.md` | `~/.claude/.ccg/prompts/gemini/analyzer.md` |
| プランニング | `~/.claude/.ccg/prompts/codex/architect.md` | `~/.claude/.ccg/prompts/gemini/architect.md` |
| レビュー | `~/.claude/.ccg/prompts/codex/reviewer.md` | `~/.claude/.ccg/prompts/gemini/reviewer.md` |

**セッション再利用**: 各呼び出しは `SESSION_ID: xxx` を返します。後続フェーズには `resume xxx` サブコマンドを使用してください（`--resume` ではなく `resume` に注意）。

**並列呼び出し**: `run_in_background: true` で開始し、`TaskOutput` で結果を待ちます。**次のフェーズに進む前に、すべてのモデルの返却を待つ必要あり**。

**バックグラウンドタスクの待機**（最大タイムアウト 600000ms = 10分を使用）:

```
TaskOutput({ task_id: "<task_id>", block: true, timeout: 600000 })
```

**重要**:
- `timeout: 600000` を必ず指定。指定しないとデフォルトの30秒でタイムアウトが早期発生。
- 10分後もまだ未完了の場合は `TaskOutput` で継続ポーリング、**プロセスを絶対にkillしない**。
- タイムアウトにより待機がスキップされた場合、**`AskUserQuestion` を呼び出してユーザーに待機継続かタスクのkillかを確認する必要あり。直接killしてはならない。**

---

## コミュニケーションガイドライン

1. 応答はモードラベル `[Mode: X]` で開始、初期は `[Mode: Research]`。
2. 厳格なシーケンスに従う: `Research → Ideation → Plan → Execute → Optimize → Review`。
3. 各フェーズ完了後にユーザーの確認を求める。
4. スコアが7未満またはユーザーが承認しない場合は強制停止。
5. 必要に応じて `AskUserQuestion` ツールでユーザーとやり取り（確認/選択/承認など）。

## 外部オーケストレーションの使用タイミング

作業を並列ワーカーに分割する必要があり、それらが独立したgit状態、独立したターミナル、または個別のビルド/テスト実行を必要とする場合、外部のtmux/worktreeオーケストレーションを使用します。軽量な分析、プランニング、レビューでは、メインセッションが唯一のライターである場合、インプロセスのサブエージェントを使用します。

```bash
node scripts/orchestrate-worktrees.js .claude/plan/workflow-e2e-test.json --execute
```

---

## 実行ワークフロー

**タスク記述**: $ARGUMENTS

### フェーズ1: リサーチと分析

`[Mode: Research]` - 要件の理解とコンテキストの収集:

1. **プロンプト強化**（ace-tool MCPが利用可能な場合）: `mcp__ace-tool__enhance_prompt` を呼び出し、**後続のすべてのCodex/Gemini呼び出しで元の$ARGUMENTSを強化結果に置換**。利用不可の場合は `$ARGUMENTS` をそのまま使用。
2. **コンテキスト取得**（ace-tool MCPが利用可能な場合）: `mcp__ace-tool__search_context` を呼び出す。利用不可の場合は組み込みツールを使用: ファイル検出に `Glob`、シンボル検索に `Grep`、コンテキスト収集に `Read`、より深い探索に `Task`（Exploreエージェント）。
3. **要件完全性スコア** (0-10):
   - 目標の明確さ (0-3)、期待される成果 (0-3)、スコープの境界 (0-2)、制約 (0-2)
   - 7以上: 続行 | 7未満: 停止、明確化の質問を行う

### フェーズ2: ソリューションのアイデア

`[Mode: Ideation]` - マルチモデル並列分析:

**並列呼び出し**（`run_in_background: true`）:
- Codex: analyzerプロンプトを使用、技術的実現可能性、ソリューション、リスクを出力
- Gemini: analyzerプロンプトを使用、UI実現可能性、ソリューション、UX評価を出力

`TaskOutput` で結果を待つ。**SESSION_ID** (`CODEX_SESSION` と `GEMINI_SESSION`) を保存。

**上記の `マルチモデル呼び出し仕様` の `重要` な指示に従ってください**

両方の分析を統合し、ソリューション比較（最低2オプション）を出力、ユーザーの選択を待つ。

### フェーズ3: 詳細プランニング

`[Mode: Plan]` - マルチモデル協調プランニング:

**並列呼び出し**（`resume <SESSION_ID>` でセッションを再開）:
- Codex: architectプロンプト + `resume $CODEX_SESSION` を使用、バックエンドアーキテクチャを出力
- Gemini: architectプロンプト + `resume $GEMINI_SESSION` を使用、フロントエンドアーキテクチャを出力

`TaskOutput` で結果を待つ。

**上記の `マルチモデル呼び出し仕様` の `重要` な指示に従ってください**

**Claude統合**: Codexのバックエンドプラン + Geminiのフロントエンドプランを採用し、ユーザーの承認後に `.claude/plan/task-name.md` に保存。

### フェーズ4: 実装

`[Mode: Execute]` - コード開発:

- 承認されたプランに厳密に従う
- 既存プロジェクトのコード規約に準拠
- 主要なマイルストーンでフィードバックを求める

### フェーズ5: コード最適化

`[Mode: Optimize]` - マルチモデル並列レビュー:

**並列呼び出し**:
- Codex: reviewerプロンプトを使用、セキュリティ、パフォーマンス、エラーハンドリングにフォーカス
- Gemini: reviewerプロンプトを使用、アクセシビリティ、デザインの一貫性にフォーカス

`TaskOutput` で結果を待つ。レビューフィードバックを統合し、ユーザー確認後に最適化を実行。

**上記の `マルチモデル呼び出し仕様` の `重要` な指示に従ってください**

### フェーズ6: 品質レビュー

`[Mode: Review]` - 最終評価:

- プランに対する完了度をチェック
- テストを実行して機能を検証
- 問題と推奨事項を報告
- ユーザーの最終確認を求める

---

## 重要なルール

1. フェーズのシーケンスはスキップ不可（ユーザーが明示的に指示しない限り）
2. 外部モデルは**ファイルシステムへの書き込みアクセスなし**、すべての変更はClaudeが実行
3. スコアが7未満またはユーザーが承認しない場合は**強制停止**
