# Plan - マルチモデル協調プランニング

マルチモデル協調プランニング - コンテキスト取得 + デュアルモデル分析 → ステップバイステップの実装プランを生成。

$ARGUMENTS

---

## コアプロトコル

- **言語プロトコル**: ツール/モデルとのやり取りには**英語**を使用、ユーザーとはユーザーの言語でコミュニケーション
- **必須並列**: Codex/Geminiの呼び出しは `run_in_background: true` を使用する必要あり（単一モデル呼び出しを含む、メインスレッドのブロックを避けるため）
- **コード主権**: 外部モデルは**ファイルシステムへの書き込みアクセスなし**、すべての変更はClaudeが実行
- **ストップロスメカニズム**: 現在のフェーズの出力が検証されるまで次のフェーズに進まない
- **プランニングのみ**: このコマンドはコンテキストの読み取りと `.claude/plan/*` プランファイルへの書き込みを許可するが、**本番コードは絶対に変更しない**

---

## マルチモデル呼び出し仕様

**呼び出し構文**（並列: `run_in_background: true` を使用）:

```
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}- \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Requirement: <enhanced requirement>
Context: <retrieved project context>
</TASK>
OUTPUT: Step-by-step implementation plan with pseudo-code. DO NOT modify any files.
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

**セッション再利用**: 各呼び出しは `SESSION_ID: xxx` を返します（通常はラッパーが出力）。後続の `/ccg:execute` 使用のために**必ず保存**してください。

**バックグラウンドタスクの待機**（最大タイムアウト 600000ms = 10分）:

```
TaskOutput({ task_id: "<task_id>", block: true, timeout: 600000 })
```

**重要**:
- `timeout: 600000` を必ず指定。指定しないとデフォルトの30秒でタイムアウトが早期発生
- 10分後もまだ未完了の場合は `TaskOutput` で継続ポーリング、**プロセスを絶対にkillしない**
- タイムアウトにより待機がスキップされた場合、**`AskUserQuestion` を呼び出してユーザーに待機継続かタスクのkillかを確認する必要あり**

---

## 実行ワークフロー

**プランニングタスク**: $ARGUMENTS

### フェーズ1: 完全なコンテキスト取得

`[Mode: Research]`

#### 1.1 プロンプト強化（必ず最初に実行）

**ace-tool MCPが利用可能な場合**、`mcp__ace-tool__enhance_prompt` ツールを呼び出す:

```
mcp__ace-tool__enhance_prompt({
  prompt: "$ARGUMENTS",
  conversation_history: "<last 5-10 conversation turns>",
  project_root_path: "$PWD"
})
```

強化されたプロンプトを待ち、**後続のすべてのフェーズで元の$ARGUMENTSを強化結果に置換**。

**ace-tool MCPが利用不可の場合**: このステップをスキップし、後続のすべてのフェーズで元の `$ARGUMENTS` をそのまま使用。

#### 1.2 コンテキスト取得

**ace-tool MCPが利用可能な場合**、`mcp__ace-tool__search_context` ツールを呼び出す:

```
mcp__ace-tool__search_context({
  query: "<semantic query based on enhanced requirement>",
  project_root_path: "$PWD"
})
```

- 自然言語（Where/What/How）を使用してセマンティッククエリを構築
- **仮定に基づいた回答は絶対にしない**

**ace-tool MCPが利用不可の場合**、Claude Codeの組み込みツールをフォールバックとして使用:
1. **Glob**: パターンで関連ファイルを検出（例: `Glob("**/*.ts")`、`Glob("src/**/*.py")`）
2. **Grep**: 主要なシンボル、関数名、クラス定義を検索（例: `Grep("className|functionName")`）
3. **Read**: 発見されたファイルを読み込み、完全なコンテキストを収集
4. **Task (Exploreエージェント)**: より深い探索には、`Task` を `subagent_type: "Explore"` でコードベース全体を検索

#### 1.3 完全性チェック

- 関連するクラス、関数、変数の**完全な定義とシグネチャ**を取得する必要あり
- コンテキストが不十分な場合、**再帰取得**をトリガー
- 出力の優先: エントリファイル + 行番号 + 主要シンボル名; 曖昧さ解消のために必要な場合のみ最小限のコードスニペットを追加

#### 1.4 要件の整合

- 要件にまだ曖昧さがある場合、ユーザーにガイド質問を**必ず**出力
- 要件の境界が明確になるまで（漏れなし、冗長なし）

### フェーズ2: マルチモデル協調分析

`[Mode: Analysis]`

#### 2.1 入力の配布

**並列呼び出し** CodexとGemini（`run_in_background: true`）:

**元の要件**（プリセットの意見なし）を両方のモデルに配布:

1. **Codexバックエンド分析**:
   - ROLE_FILE: `~/.claude/.ccg/prompts/codex/analyzer.md`
   - フォーカス: 技術的実現可能性、アーキテクチャへの影響、パフォーマンスの考慮、潜在的リスク
   - OUTPUT: 多角的ソリューション + 長所/短所の分析

2. **Geminiフロントエンド分析**:
   - ROLE_FILE: `~/.claude/.ccg/prompts/gemini/analyzer.md`
   - フォーカス: UI/UXへの影響、ユーザーエクスペリエンス、ビジュアルデザイン
   - OUTPUT: 多角的ソリューション + 長所/短所の分析

`TaskOutput` で両方のモデルの完全な結果を待つ。**SESSION_ID** (`CODEX_SESSION` と `GEMINI_SESSION`) を保存。

#### 2.2 クロスバリデーション

視点を統合し、最適化のために反復:

1. **合意点を特定**（強いシグナル）
2. **相違点を特定**（重み付けが必要）
3. **相補的な強み**: バックエンドロジックはCodexに従い、フロントエンドデザインはGeminiに従う
4. **論理的推論**: ソリューションの論理的ギャップを排除

#### 2.3（オプションだが推奨）デュアルモデルプランドラフト

Claudeの統合プランにおける漏れのリスクを軽減するため、両方のモデルに「プランドラフト」を並列出力させることが可能（ファイルの変更は**不可**）:

1. **Codexプランドラフト**（バックエンドの権威）:
   - ROLE_FILE: `~/.claude/.ccg/prompts/codex/architect.md`
   - OUTPUT: ステップバイステップのプラン + 疑似コード（フォーカス: データフロー/エッジケース/エラーハンドリング/テスト戦略）

2. **Geminiプランドラフト**（フロントエンドの権威）:
   - ROLE_FILE: `~/.claude/.ccg/prompts/gemini/architect.md`
   - OUTPUT: ステップバイステップのプラン + 疑似コード（フォーカス: 情報アーキテクチャ/インタラクション/アクセシビリティ/ビジュアルの一貫性）

`TaskOutput` で両方のモデルの完全な結果を待ち、提案の主要な違いを記録。

#### 2.4 実装プランの生成（Claude最終版）

両方の分析を統合し、**ステップバイステップの実装プラン**を生成:

```markdown
## Implementation Plan: <Task Name>

### Task Type
- [ ] Frontend (→ Gemini)
- [ ] Backend (→ Codex)
- [ ] Fullstack (→ Parallel)

### Technical Solution
<Optimal solution synthesized from Codex + Gemini analysis>

### Implementation Steps
1. <Step 1> - Expected deliverable
2. <Step 2> - Expected deliverable
...

### Key Files
| File | Operation | Description |
|------|-----------|-------------|
| path/to/file.ts:L10-L50 | Modify | Description |

### Risks and Mitigation
| Risk | Mitigation |
|------|------------|

### SESSION_ID (for /ccg:execute use)
- CODEX_SESSION: <session_id>
- GEMINI_SESSION: <session_id>
```

### フェーズ2終了: プランのデリバリー（実行ではない）

**`/ccg:plan` の責任はここで終了、以下のアクションを必ず実行**:

1. 完全な実装プランをユーザーに提示（疑似コードを含む）
2. プランを `.claude/plan/<feature-name>.md` に保存（要件からフィーチャー名を抽出、例: `user-auth`、`payment-module`）
3. **太字テキスト**でプロンプトを出力（実際に保存されたファイルパスを使用する必要あり）:

   ---
   **Plan generated and saved to `.claude/plan/actual-feature-name.md`**

   **Please review the plan above. You can:**
   - **Modify plan**: Tell me what needs adjustment, I'll update the plan
   - **Execute plan**: Copy the following command to a new session

   ```
   /ccg:execute .claude/plan/actual-feature-name.md
   ```
   ---

   **NOTE**: 上記の `actual-feature-name.md` は実際に保存されたファイル名に置き換える必要あり！

4. **現在のレスポンスを即座に終了**（ここで停止。これ以上のツール呼び出しなし。）

**絶対禁止**:
- ユーザーに「Y/N」を聞いてから自動実行（実行は `/ccg:execute` の責任）
- 本番コードへの書き込み操作
- `/ccg:execute` や実装アクションの自動呼び出し
- ユーザーが明示的に変更を要求していない状態でモデル呼び出しを継続トリガー

---

## プランの保存

プランニング完了後、プランを以下に保存:

- **初回プランニング**: `.claude/plan/<feature-name>.md`
- **イテレーションバージョン**: `.claude/plan/<feature-name>-v2.md`、`.claude/plan/<feature-name>-v3.md`...

プランファイルの書き込みは、プランをユーザーに提示する前に完了する必要あり。

---

## プラン変更フロー

ユーザーがプランの変更を要求した場合:

1. ユーザーのフィードバックに基づいてプラン内容を調整
2. `.claude/plan/<feature-name>.md` ファイルを更新
3. 変更されたプランを再提示
4. ユーザーにレビューまたは実行を再度プロンプト

---

## 次のステップ

ユーザーの承認後、**手動で**実行:

```bash
/ccg:execute .claude/plan/<feature-name>.md
```

---

## 重要なルール

1. **プランのみ、実装なし** – このコマンドはコード変更を実行しない
2. **Y/Nプロンプトなし** – プランを提示するのみ、次のステップはユーザーが決定
3. **信頼ルール** – バックエンドはCodexに従い、フロントエンドはGeminiに従う
4. 外部モデルは**ファイルシステムへの書き込みアクセスなし**
5. **SESSION_IDの引き継ぎ** – プランの末尾に `CODEX_SESSION` / `GEMINI_SESSION` を含める必要あり（`/ccg:execute resume <SESSION_ID>` 使用のため）
