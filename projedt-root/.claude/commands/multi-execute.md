# Execute - マルチモデル協調実行

マルチモデル協調実行 - プランからプロトタイプを取得 → Claudeがリファクタリングして実装 → マルチモデル監査とデリバリー。

$ARGUMENTS

---

## コアプロトコル

- **言語プロトコル**: ツール/モデルとのやり取りには**英語**を使用、ユーザーとはユーザーの言語でコミュニケーション
- **コード主権**: 外部モデルは**ファイルシステムへの書き込みアクセスなし**、すべての変更はClaudeが実行
- **ダーティプロトタイプリファクタリング**: Codex/GeminiのUnified Diffは「ダーティプロトタイプ」として扱い、プロダクショングレードのコードにリファクタリングする必要あり
- **ストップロスメカニズム**: 現在のフェーズの出力が検証されるまで次のフェーズに進まない
- **前提条件**: ユーザーが `/ccg:plan` の出力に明示的に「Y」と返信した後にのみ実行（返信がない場合は先に確認が必要）

---

## マルチモデル呼び出し仕様

**呼び出し構文**（並列: `run_in_background: true` を使用）:

```
# Resume session call (recommended) - Implementation Prototype
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}resume <SESSION_ID> - \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Requirement: <task description>
Context: <plan content + target files>
</TASK>
OUTPUT: Unified Diff Patch ONLY. Strictly prohibit any actual modifications.
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "Brief description"
})

# New session call - Implementation Prototype
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}- \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Requirement: <task description>
Context: <plan content + target files>
</TASK>
OUTPUT: Unified Diff Patch ONLY. Strictly prohibit any actual modifications.
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "Brief description"
})
```

**監査呼び出し構文**（コードレビュー / 監査）:

```
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}resume <SESSION_ID> - \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Scope: Audit the final code changes.
Inputs:
- The applied patch (git diff / final unified diff)
- The touched files (relevant excerpts if needed)
Constraints:
- Do NOT modify any files.
- Do NOT output tool commands that assume filesystem access.
</TASK>
OUTPUT:
1) A prioritized list of issues (severity, file, rationale)
2) Concrete fixes; if code changes are needed, include a Unified Diff Patch in a fenced code block.
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
| 実装 | `~/.claude/.ccg/prompts/codex/architect.md` | `~/.claude/.ccg/prompts/gemini/frontend.md` |
| レビュー | `~/.claude/.ccg/prompts/codex/reviewer.md` | `~/.claude/.ccg/prompts/gemini/reviewer.md` |

**セッション再利用**: `/ccg:plan` がSESSION_IDを提供した場合、コンテキストを再利用するために `resume <SESSION_ID>` を使用。

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

**実行タスク**: $ARGUMENTS

### フェーズ0: プランの読み込み

`[Mode: Prepare]`

1. **入力タイプの識別**:
   - プランファイルパス（例: `.claude/plan/xxx.md`）
   - 直接のタスク記述

2. **プラン内容の読み込み**:
   - プランファイルパスが提供された場合、読み込みと解析
   - 抽出: タスクタイプ、実装ステップ、主要ファイル、SESSION_ID

3. **実行前確認**:
   - 入力が「直接のタスク記述」であるか、プランに `SESSION_ID` / 主要ファイルがない場合: ユーザーに先に確認
   - ユーザーがプランに「Y」と返信したことが確認できない場合: 進行前に再度確認が必要

4. **タスクタイプのルーティング**:

   | タスクタイプ | 検出方法 | ルート |
   |-----------|-----------|-------|
   | **フロントエンド** | ページ、コンポーネント、UI、スタイル、レイアウト | Gemini |
   | **バックエンド** | API、インターフェース、データベース、ロジック、アルゴリズム | Codex |
   | **フルスタック** | フロントエンドとバックエンドの両方を含む | Codex ∥ Gemini 並列 |

---

### フェーズ1: クイックコンテキスト取得

`[Mode: Retrieval]`

**ace-tool MCPが利用可能な場合**、クイックコンテキスト取得に使用:

プランの「主要ファイル」リストに基づき、`mcp__ace-tool__search_context` を呼び出す:

```
mcp__ace-tool__search_context({
  query: "<semantic query based on plan content, including key files, modules, function names>",
  project_root_path: "$PWD"
})
```

**取得戦略**:
- プランの「主要ファイル」テーブルからターゲットパスを抽出
- エントリファイル、依存モジュール、関連する型定義をカバーするセマンティッククエリを構築
- 結果が不十分な場合、1〜2回の再帰取得を追加

**ace-tool MCPが利用不可の場合**、Claude Codeの組み込みツールをフォールバックとして使用:
1. **Glob**: プランの「主要ファイル」テーブルからターゲットファイルを検出（例: `Glob("src/components/**/*.tsx")`）
2. **Grep**: 主要なシンボル、関数名、型定義をコードベース全体で検索
3. **Read**: 発見されたファイルを読み込み、完全なコンテキストを収集
4. **Task (Exploreエージェント)**: より広範な探索には、`Task` を `subagent_type: "Explore"` で使用

**取得後**:
- 取得したコードスニペットを整理
- 実装に必要な完全なコンテキストを確認
- フェーズ3へ進む

---

### フェーズ3: プロトタイプ取得

`[Mode: Prototype]`

**タスクタイプに基づいたルーティング**:

#### ルートA: フロントエンド/UI/スタイル → Gemini

**制限**: コンテキスト < 32kトークン

1. Geminiを呼び出す（`~/.claude/.ccg/prompts/gemini/frontend.md` を使用）
2. 入力: プラン内容 + 取得したコンテキスト + ターゲットファイル
3. OUTPUT: `Unified Diff Patch ONLY. Strictly prohibit any actual modifications.`
4. **GeminiはフロントエンドデザインのAauthority。そのCSS/React/Vueプロトタイプが最終的なビジュアルベースライン**
5. **警告**: Geminiのバックエンドロジック提案は無視
6. プランに `GEMINI_SESSION` がある場合: `resume <GEMINI_SESSION>` を優先

#### ルートB: バックエンド/ロジック/アルゴリズム → Codex

1. Codexを呼び出す（`~/.claude/.ccg/prompts/codex/architect.md` を使用）
2. 入力: プラン内容 + 取得したコンテキスト + ターゲットファイル
3. OUTPUT: `Unified Diff Patch ONLY. Strictly prohibit any actual modifications.`
4. **Codexはバックエンドロジックの権威。その論理的推論とデバッグ能力を活用**
5. プランに `CODEX_SESSION` がある場合: `resume <CODEX_SESSION>` を優先

#### ルートC: フルスタック → 並列呼び出し

1. **並列呼び出し**（`run_in_background: true`）:
   - Gemini: フロントエンド部分を担当
   - Codex: バックエンド部分を担当
2. `TaskOutput` で両方のモデルの完全な結果を待つ
3. 各モデルはプランの対応する `SESSION_ID` を `resume` に使用（ない場合は新しいセッションを作成）

**上記の `マルチモデル呼び出し仕様` の `重要` な指示に従ってください**

---

### フェーズ4: コード実装

`[Mode: Implement]`

**Claudeがコード主権者として以下のステップを実行**:

1. **Diffの読み込み**: Codex/Geminiが返したUnified Diff Patchを解析

2. **メンタルサンドボックス**:
   - ターゲットファイルへのDiff適用をシミュレート
   - 論理的整合性をチェック
   - 潜在的な競合や副作用を特定

3. **リファクタリングとクリーンアップ**:
   - 「ダーティプロトタイプ」を**高い可読性、保守性、エンタープライズグレードのコード**にリファクタリング
   - 冗長なコードを削除
   - プロジェクトの既存コード規約への準拠を確保
   - **必要な場合を除き、コメント/ドキュメントを生成しない** — コードは自己説明的であるべき

4. **最小スコープ**:
   - 変更は要件のスコープのみに限定
   - 副作用の**必須レビュー**
   - 対象を絞った修正を実施

5. **変更の適用**:
   - Edit/Writeツールを使用して実際の変更を実行
   - **必要なコードのみ変更**、ユーザーの他の既存機能には影響しない

6. **自己検証**（強く推奨）:
   - プロジェクトの既存lint / typecheck / テストを実行（最小限の関連スコープを優先）
   - 失敗した場合: 先にリグレッションを修正し、フェーズ5に進む

---

### フェーズ5: 監査とデリバリー

`[Mode: Audit]`

#### 5.1 自動監査

**変更が有効になった後、直ちに並列でCodexとGeminiを呼び出してコードレビューを実施する必要あり**:

1. **Codexレビュー**（`run_in_background: true`）:
   - ROLE_FILE: `~/.claude/.ccg/prompts/codex/reviewer.md`
   - 入力: 変更Diff + ターゲットファイル
   - フォーカス: セキュリティ、パフォーマンス、エラーハンドリング、ロジックの正確性

2. **Geminiレビュー**（`run_in_background: true`）:
   - ROLE_FILE: `~/.claude/.ccg/prompts/gemini/reviewer.md`
   - 入力: 変更Diff + ターゲットファイル
   - フォーカス: アクセシビリティ、デザインの一貫性、ユーザーエクスペリエンス

`TaskOutput` で両方のモデルの完全なレビュー結果を待つ。コンテキストの一貫性のためにフェーズ3のセッション（`resume <SESSION_ID>`）の再利用を優先。

#### 5.2 統合と修正

1. Codex + Geminiのレビューフィードバックを統合
2. 信頼ルールに基づいて重み付け: バックエンドはCodexに従い、フロントエンドはGeminiに従う
3. 必要な修正を実行
4. 必要に応じてフェーズ5.1を繰り返す（リスクが許容範囲になるまで）

#### 5.3 デリバリー確認

監査が通過した後、ユーザーに報告:

```markdown
## Execution Complete

### Change Summary
| File | Operation | Description |
|------|-----------|-------------|
| path/to/file.ts | Modified | Description |

### Audit Results
- Codex: <Passed/Found N issues>
- Gemini: <Passed/Found N issues>

### Recommendations
1. [ ] <Suggested test steps>
2. [ ] <Suggested verification steps>
```

---

## 重要なルール

1. **コード主権** – すべてのファイル変更はClaudeが実行、外部モデルは書き込みアクセスなし
2. **ダーティプロトタイプリファクタリング** – Codex/Geminiの出力はドラフトとして扱い、リファクタリングが必須
3. **信頼ルール** – バックエンドはCodexに従い、フロントエンドはGeminiに従う
4. **最小限の変更** – 必要なコードのみ変更、副作用なし
5. **必須監査** – 変更後にマルチモデルコードレビューを実施する必要あり

---

## 使い方

```bash
# Execute plan file
/ccg:execute .claude/plan/feature-name.md

# Execute task directly (for plans already discussed in context)
/ccg:execute implement user authentication based on previous plan
```

---

## /ccg:plan との関係

1. `/ccg:plan` がプラン + SESSION_IDを生成
2. ユーザーが「Y」で確認
3. `/ccg:execute` がプランを読み込み、SESSION_IDを再利用して実装を実行
