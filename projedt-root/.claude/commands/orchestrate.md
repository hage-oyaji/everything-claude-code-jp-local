---
description: マルチエージェントワークフローのための逐次およびtmux/worktreeオーケストレーションガイダンス。
---

# オーケストレートコマンド

複雑なタスクのための逐次エージェントワークフロー。

## 使い方

`/orchestrate [workflow-type] [task-description]`

## ワークフロータイプ

### feature
フル機能実装ワークフロー:
```
planner -> tdd-guide -> code-reviewer -> security-reviewer
```

### bugfix
バグ調査と修正ワークフロー:
```
planner -> tdd-guide -> code-reviewer
```

### refactor
安全なリファクタリングワークフロー:
```
architect -> code-reviewer -> tdd-guide
```

### security
セキュリティ重視のレビュー:
```
security-reviewer -> code-reviewer -> architect
```

## 実行パターン

ワークフロー内の各エージェントについて:

1. **エージェントを呼び出し** 前のエージェントからのコンテキストを渡す
2. **出力を収集** 構造化されたハンドオフドキュメントとして
3. **次のエージェントに渡す** チェーン内の次へ
4. **結果を集約** 最終レポートに

## ハンドオフドキュメントフォーマット

エージェント間で以下のハンドオフドキュメントを作成:

```markdown
## HANDOFF: [previous-agent] -> [next-agent]

### Context
[Summary of what was done]

### Findings
[Key discoveries or decisions]

### Files Modified
[List of files touched]

### Open Questions
[Unresolved items for next agent]

### Recommendations
[Suggested next steps]
```

## 例: 機能ワークフロー

```
/orchestrate feature "Add user authentication"
```

実行内容:

1. **Plannerエージェント**
   - 要件を分析
   - 実装プランを作成
   - 依存関係を特定
   - 出力: `HANDOFF: planner -> tdd-guide`

2. **TDD Guideエージェント**
   - Plannerのハンドオフを読み取り
   - テストを先に作成
   - テストを通す実装
   - 出力: `HANDOFF: tdd-guide -> code-reviewer`

3. **Code Reviewerエージェント**
   - 実装をレビュー
   - 問題をチェック
   - 改善を提案
   - 出力: `HANDOFF: code-reviewer -> security-reviewer`

4. **Security Reviewerエージェント**
   - セキュリティ監査
   - 脆弱性チェック
   - 最終承認
   - 出力: 最終レポート

## 最終レポートフォーマット

```
ORCHESTRATION REPORT
====================
Workflow: feature
Task: Add user authentication
Agents: planner -> tdd-guide -> code-reviewer -> security-reviewer

SUMMARY
-------
[One paragraph summary]

AGENT OUTPUTS
-------------
Planner: [summary]
TDD Guide: [summary]
Code Reviewer: [summary]
Security Reviewer: [summary]

FILES CHANGED
-------------
[List all files modified]

TEST RESULTS
------------
[Test pass/fail summary]

SECURITY STATUS
---------------
[Security findings]

RECOMMENDATION
--------------
[SHIP / NEEDS WORK / BLOCKED]
```

## 並列実行

独立したチェックの場合、エージェントを並列実行:

```markdown
### Parallel Phase
Run simultaneously:
- code-reviewer (quality)
- security-reviewer (security)
- architect (design)

### Merge Results
Combine outputs into single report
```

独立したgit worktreeを持つ外部tmuxペインワーカーには、`node scripts/orchestrate-worktrees.js plan.json --execute` を使用します。組み込みのオーケストレーションパターンはインプロセスで実行されます。ヘルパーは長時間実行またはクロスハーネスセッション用です。

ワーカーがメインチェックアウトのダーティファイルや未トラックのローカルファイルを参照する必要がある場合、プランファイルに `seedPaths` を追加します。ECCは `git worktree add` の後に選択されたパスのみを各ワーカーworktreeにオーバーレイし、ブランチを隔離しつつ進行中のローカルスクリプト、プラン、ドキュメントを公開します。

```json
{
  "sessionName": "workflow-e2e",
  "seedPaths": [
    "scripts/orchestrate-worktrees.js",
    "scripts/lib/tmux-worktree-orchestrator.js",
    ".claude/plan/workflow-e2e-test.json"
  ],
  "workers": [
    { "name": "docs", "task": "Update orchestration docs." }
  ]
}
```

ライブのtmux/worktreeセッションのコントロールプレーンスナップショットをエクスポートするには:

```bash
node scripts/orchestration-status.js .claude/plan/workflow-visual-proof.json
```

スナップショットには、セッションアクティビティ、tmuxペインメタデータ、ワーカーの状態、目標、シードされたオーバーレイ、最近のハンドオフサマリーがJSON形式で含まれます。

## オペレーターコマンドセンターハンドオフ

ワークフローが複数のセッション、worktree、またはtmuxペインにまたがる場合、最終ハンドオフにコントロールプレーンブロックを追加:

```markdown
CONTROL PLANE
-------------
Sessions:
- active session ID or alias
- branch + worktree path for each active worker
- tmux pane or detached session name when applicable

Diffs:
- git status summary
- git diff --stat for touched files
- merge/conflict risk notes

Approvals:
- pending user approvals
- blocked steps awaiting confirmation

Telemetry:
- last activity timestamp or idle signal
- estimated token or cost drift
- policy events raised by hooks or reviewers
```

これにより、プランナー、実装者、レビュアー、ループワーカーがオペレーターサーフェスから可読になります。

## 引数

$ARGUMENTS:
- `feature <description>` - フル機能ワークフロー
- `bugfix <description>` - バグ修正ワークフロー
- `refactor <description>` - リファクタリングワークフロー
- `security <description>` - セキュリティレビューワークフロー
- `custom <agents> <description>` - カスタムエージェントシーケンス

## カスタムワークフローの例

```
/orchestrate custom "architect,tdd-guide,code-reviewer" "Redesign caching layer"
```

## ヒント

1. **複雑な機能にはplannerから開始**
2. **マージ前には必ずcode-reviewerを含める**
3. **認証/決済/PIIにはsecurity-reviewerを使用**
4. **ハンドオフは簡潔に** — 次のエージェントが必要な情報に焦点を当てる
5. **必要に応じてエージェント間で検証を実行**
