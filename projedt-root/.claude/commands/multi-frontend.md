# Frontend - フロントエンド特化型開発

フロントエンド特化型ワークフロー（リサーチ → アイデア → プラン → 実行 → 最適化 → レビュー）、Gemini主導。

## 使い方

```bash
/frontend <UI task description>
```

## コンテキスト

- フロントエンドタスク: $ARGUMENTS
- Gemini主導、Codexは補助参照用
- 適用対象: コンポーネント設計、レスポンシブレイアウト、UIアニメーション、スタイル最適化

## あなたの役割

あなたは **フロントエンドオーケストレーター** です。UI/UXタスクのためのマルチモデル協調を調整します（リサーチ → アイデア → プラン → 実行 → 最適化 → レビュー）。

**協調モデル**:
- **Gemini** – フロントエンドUI/UX（**フロントエンドの権威、信頼性あり**）
- **Codex** – バックエンドの視点（**フロントエンドの意見は参考程度**）
- **Claude (self)** – オーケストレーション、プランニング、実行、デリバリー

---

## マルチモデル呼び出し仕様

**呼び出し構文**:

```
# New session call
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend gemini --gemini-model gemini-3-pro-preview - \"$PWD\" <<'EOF'
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
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend gemini --gemini-model gemini-3-pro-preview resume <SESSION_ID> - \"$PWD\" <<'EOF'
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

| フェーズ | Gemini |
|-------|--------|
| 分析 | `~/.claude/.ccg/prompts/gemini/analyzer.md` |
| プランニング | `~/.claude/.ccg/prompts/gemini/architect.md` |
| レビュー | `~/.claude/.ccg/prompts/gemini/reviewer.md` |

**セッション再利用**: 各呼び出しは `SESSION_ID: xxx` を返します。後続フェーズには `resume xxx` を使用してください。フェーズ2で `GEMINI_SESSION` を保存し、フェーズ3と5で `resume` を使用します。

---

## コミュニケーションガイドライン

1. 応答はモードラベル `[Mode: X]` で開始、初期は `[Mode: Research]`
2. 厳格なシーケンスに従う: `Research → Ideation → Plan → Execute → Optimize → Review`
3. 必要に応じて `AskUserQuestion` ツールでユーザーとやり取り（確認/選択/承認など）

---

## コアワークフロー

### フェーズ0: プロンプト強化（オプション）

`[Mode: Prepare]` - ace-tool MCPが利用可能な場合、`mcp__ace-tool__enhance_prompt` を呼び出し、**元の$ARGUMENTSを強化結果に置換して後続のGemini呼び出しに使用**。利用不可の場合は `$ARGUMENTS` をそのまま使用。

### フェーズ1: リサーチ

`[Mode: Research]` - 要件の理解とコンテキストの収集

1. **コード取得**（ace-tool MCPが利用可能な場合）: `mcp__ace-tool__search_context` を呼び出して既存のコンポーネント、スタイル、デザインシステムを取得。利用不可の場合は組み込みツールを使用: ファイル検出に `Glob`、コンポーネント/スタイル検索に `Grep`、コンテキスト収集に `Read`、より深い探索に `Task`（Exploreエージェント）。
2. 要件完全性スコア (0-10): 7以上は続行、7未満は中止して補足

### フェーズ2: アイデア

`[Mode: Ideation]` - Gemini主導の分析

**Geminiを呼び出す必要あり**（上記の呼び出し仕様に従う）:
- ROLE_FILE: `~/.claude/.ccg/prompts/gemini/analyzer.md`
- Requirement: 強化された要件（または未強化の場合は$ARGUMENTS）
- Context: フェーズ1のプロジェクトコンテキスト
- OUTPUT: UI実現可能性分析、推奨ソリューション（最低2つ）、UX評価

**SESSION_ID** (`GEMINI_SESSION`) を後続フェーズの再利用のために保存。

ソリューション（最低2つ）を出力し、ユーザーの選択を待つ。

### フェーズ3: プランニング

`[Mode: Plan]` - Gemini主導のプランニング

**Geminiを呼び出す必要あり**（セッション再利用に `resume <GEMINI_SESSION>` を使用）:
- ROLE_FILE: `~/.claude/.ccg/prompts/gemini/architect.md`
- Requirement: ユーザーが選択したソリューション
- Context: フェーズ2の分析結果
- OUTPUT: コンポーネント構造、UIフロー、スタイリングアプローチ

Claudeがプランを統合し、ユーザーの承認後に `.claude/plan/task-name.md` に保存。

### フェーズ4: 実装

`[Mode: Execute]` - コード開発

- 承認されたプランに厳密に従う
- 既存プロジェクトのデザインシステムとコード規約に準拠
- レスポンシブ性、アクセシビリティを確保

### フェーズ5: 最適化

`[Mode: Optimize]` - Gemini主導のレビュー

**Geminiを呼び出す必要あり**（上記の呼び出し仕様に従う）:
- ROLE_FILE: `~/.claude/.ccg/prompts/gemini/reviewer.md`
- Requirement: 以下のフロントエンドコード変更をレビュー
- Context: git diffまたはコード内容
- OUTPUT: アクセシビリティ、レスポンシブ性、パフォーマンス、デザインの一貫性の問題リスト

レビューフィードバックを統合し、ユーザー確認後に最適化を実行。

### フェーズ6: 品質レビュー

`[Mode: Review]` - 最終評価

- プランに対する完了度をチェック
- レスポンシブ性とアクセシビリティを検証
- 問題と推奨事項を報告

---

## 重要なルール

1. **Geminiのフロントエンド意見は信頼できる**
2. **Codexのフロントエンド意見は参考程度**
3. 外部モデルは**ファイルシステムへの書き込みアクセスなし**
4. Claudeがすべてのコード書き込みとファイル操作を担当
