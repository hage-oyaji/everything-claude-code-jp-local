---
name: dmux-workflows
description: dmux（AIエージェント用tmuxペインマネージャ）を使用したマルチエージェントオーケストレーション。Claude Code、Codex、OpenCodeなどのハーネス間での並列エージェントワークフローのパターン。複数エージェントセッションの並列実行やマルチエージェント開発ワークフローの調整に使用。
origin: ECC
---

# dmuxワークフロー

dmux（エージェントハーネス用tmuxペインマネージャ）を使用して、並列AIエージェントセッションをオーケストレーションします。

## アクティベーション条件

- 複数のエージェントセッションを並列実行する時
- Claude Code、Codex、その他のハーネス間で作業を調整する時
- 分割統治型の並列処理が有効な複雑なタスク
- ユーザーが「並列実行」「作業を分割」「dmuxを使う」「マルチエージェント」と言った時

## dmuxとは

dmuxはAIエージェントペインを管理するtmuxベースのオーケストレーションツールです：
- `n`を押して新しいペインをプロンプト付きで作成
- `m`を押してペイン出力をメインセッションにマージ
- 対応: Claude Code、Codex、OpenCode、Cline、Gemini、Qwen

**インストール:** リポジトリからdmuxをインストールしてください（パッケージを確認後）。[github.com/standardagents/dmux](https://github.com/standardagents/dmux)を参照

## クイックスタート

```bash
# Start dmux session
dmux

# Create agent panes (press 'n' in dmux, then type prompt)
# Pane 1: "Implement the auth middleware in src/auth/"
# Pane 2: "Write tests for the user service"
# Pane 3: "Update API documentation"

# Each pane runs its own agent session
# Press 'm' to merge results back
```

## ワークフローパターン

### パターン1: リサーチ + 実装

リサーチと実装を並列トラックに分割：

```
Pane 1 (Research): "Research best practices for rate limiting in Node.js.
  Check current libraries, compare approaches, and write findings to
  /tmp/rate-limit-research.md"

Pane 2 (Implement): "Implement rate limiting middleware for our Express API.
  Start with a basic token bucket, we'll refine after research completes."

# After Pane 1 completes, merge findings into Pane 2's context
```

### パターン2: マルチファイル機能

独立したファイル間で作業を並列化：

```
Pane 1: "Create the database schema and migrations for the billing feature"
Pane 2: "Build the billing API endpoints in src/api/billing/"
Pane 3: "Create the billing dashboard UI components"

# Merge all, then do integration in main pane
```

### パターン3: テスト + 修正ループ

一方のペインでテストを実行し、もう一方で修正：

```
Pane 1 (Watcher): "Run the test suite in watch mode. When tests fail,
  summarize the failures."

Pane 2 (Fixer): "Fix failing tests based on the error output from pane 1"
```

### パターン4: クロスハーネス

異なるタスクに異なるAIツールを使用：

```
Pane 1 (Claude Code): "Review the security of the auth module"
Pane 2 (Codex): "Refactor the utility functions for performance"
Pane 3 (Claude Code): "Write E2E tests for the checkout flow"
```

### パターン5: コードレビューパイプライン

並列レビューの視点：

```
Pane 1: "Review src/api/ for security vulnerabilities"
Pane 2: "Review src/api/ for performance issues"
Pane 3: "Review src/api/ for test coverage gaps"

# Merge all reviews into a single report
```

## ベストプラクティス

1. **独立したタスクのみ。** 互いの出力に依存するタスクは並列化しない。
2. **明確な境界。** 各ペインは異なるファイルや関心事に取り組む。
3. **戦略的にマージ。** 競合を避けるため、マージ前にペイン出力をレビュー。
4. **gitワークツリーを使用。** ファイル競合が起きやすい作業には、ペインごとに別のワークツリーを使用。
5. **リソースの認識。** 各ペインはAPIトークンを使用 — ペイン合計は5-6以下に。

## Gitワークツリー統合

重複するファイルに触れるタスクの場合：

```bash
# Create worktrees for isolation
git worktree add -b feat/auth ../feature-auth HEAD
git worktree add -b feat/billing ../feature-billing HEAD

# Run agents in separate worktrees
# Pane 1: cd ../feature-auth && claude
# Pane 2: cd ../feature-billing && claude

# Merge branches when done
git merge feat/auth
git merge feat/billing
```

## 補完ツール

| ツール | 機能 | 使用するタイミング |
|------|-------------|-------------|
| **dmux** | エージェント用tmuxペイン管理 | 並列エージェントセッション |
| **Superset** | 10以上の並列エージェント用ターミナルIDE | 大規模オーケストレーション |
| **Claude Code Taskツール** | プロセス内サブエージェント生成 | セッション内のプログラム的並列処理 |
| **Codex multi-agent** | 組み込みエージェントロール | Codex固有の並列作業 |

## ECCヘルパー

ECCには、個別のgitワークツリーを使用した外部tmuxペインオーケストレーション用のヘルパーが含まれています：

```bash
node scripts/orchestrate-worktrees.js plan.json --execute
```

`plan.json`の例：

```json
{
  "sessionName": "skill-audit",
  "baseRef": "HEAD",
  "launcherCommand": "codex exec --cwd {worktree_path} --task-file {task_file}",
  "workers": [
    { "name": "docs-a", "task": "Fix skills 1-4 and write handoff notes." },
    { "name": "docs-b", "task": "Fix skills 5-8 and write handoff notes." }
  ]
}
```

ヘルパーの機能：
- ワーカーごとにブランチバックのgitワークツリーを1つ作成
- オプションで選択した`seedPaths`をメインチェックアウトから各ワーカーワークツリーにオーバーレイ
- `.orchestration/<session>/`配下にワーカーごとの`task.md`、`handoff.md`、`status.md`ファイルを作成
- ワーカーごとに1ペインのtmuxセッションを開始
- 各ワーカーコマンドを自身のペインで起動
- メインペインをオーケストレーター用に空けておく

`seedPaths`は、ローカルのオーケストレーションスクリプト、ドラフトプラン、ドキュメントなど、まだ`HEAD`の一部ではないダーティまたは未追跡のローカルファイルにワーカーがアクセスする必要がある場合に使用します：

```json
{
  "sessionName": "workflow-e2e",
  "seedPaths": [
    "scripts/orchestrate-worktrees.js",
    "scripts/lib/tmux-worktree-orchestrator.js",
    ".claude/plan/workflow-e2e-test.json"
  ],
  "launcherCommand": "bash {repo_root}/scripts/orchestrate-codex-worker.sh {task_file} {handoff_file} {status_file}",
  "workers": [
    { "name": "seed-check", "task": "Verify seeded files are present before starting work." }
  ]
}
```

## トラブルシューティング

- **ペインが応答しない:** ペインに直接切り替えるか、`tmux capture-pane -pt <session>:0.<pane-index>`で検査。
- **マージの競合:** ペインごとのファイル変更を分離するためにgitワークツリーを使用。
- **高トークン使用量:** 並列ペイン数を削減。各ペインは完全なエージェントセッション。
- **tmuxが見つからない:** `brew install tmux`（macOS）または`apt install tmux`（Linux）でインストール。
