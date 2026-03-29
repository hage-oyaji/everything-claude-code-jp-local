---
name: claude-devfleet
description: Claude DevFleet を介したマルチエージェントコーディングタスクのオーケストレーション — プロジェクトの計画、分離されたワークツリーへの並列エージェントのディスパッチ、進捗の監視、構造化レポートの読み取り。
origin: community
---

# Claude DevFleet マルチエージェントオーケストレーション

## いつ使うか

複数の Claude Code エージェントを並列にディスパッチしてコーディングタスクに取り組む必要がある場合にこのスキルを使用する。各エージェントは完全なツールセットを備えた分離された git ワークツリーで実行される。

MCP 経由で接続された Claude DevFleet インスタンスが動作している必要がある：
```bash
claude mcp add devfleet --transport http http://localhost:18801/mcp
```

## 仕組み

```
User → "Build a REST API with auth and tests"
  ↓
plan_project(prompt) → project_id + mission DAG
  ↓
Show plan to user → get approval
  ↓
dispatch_mission(M1) → Agent 1 spawns in worktree
  ↓
M1 completes → auto-merge → auto-dispatch M2 (depends_on M1)
  ↓
M2 completes → auto-merge
  ↓
get_report(M2) → files_changed, what_done, errors, next_steps
  ↓
Report back to user
```

### ツール

| ツール | 目的 |
|------|---------|
| `plan_project(prompt)` | AI が説明文をチェーンされたミッションを持つプロジェクトに分解する |
| `create_project(name, path?, description?)` | 手動でプロジェクトを作成し、`project_id` を返す |
| `create_mission(project_id, title, prompt, depends_on?, auto_dispatch?)` | ミッションを追加。`depends_on` はミッション ID 文字列のリスト（例：`["abc-123"]`）。依存関係が満たされたときに自動開始するには `auto_dispatch=true` を設定。 |
| `dispatch_mission(mission_id, model?, max_turns?)` | ミッションでエージェントを開始 |
| `cancel_mission(mission_id)` | 実行中のエージェントを停止 |
| `wait_for_mission(mission_id, timeout_seconds?)` | ミッションが完了するまでブロック（以下の注意を参照） |
| `get_mission_status(mission_id)` | ブロックせずにミッションの進捗を確認 |
| `get_report(mission_id)` | 構造化レポートを読む（変更されたファイル、テスト済み、エラー、次のステップ） |
| `get_dashboard()` | システム概要：実行中のエージェント、統計、最近のアクティビティ |
| `list_projects()` | すべてのプロジェクトを一覧表示 |
| `list_missions(project_id, status?)` | プロジェクト内のミッションを一覧表示 |

> **`wait_for_mission` に関する注意：** これは最大 `timeout_seconds`（デフォルト 600）の間会話をブロックする。長時間実行されるミッションの場合、ユーザーに進捗状況を表示するため、代わりに 30〜60 秒ごとに `get_mission_status` でポーリングすることを推奨。

### ワークフロー：計画 → ディスパッチ → 監視 → レポート

1. **計画**: `plan_project(prompt="...")` を呼び出す → `project_id` と `depends_on` チェーンおよび `auto_dispatch=true` が設定されたミッションのリストを返す。
2. **計画の表示**: ミッションのタイトル、タイプ、依存関係チェーンをユーザーに提示する。
3. **ディスパッチ**: ルートミッション（`depends_on` が空のもの）に対して `dispatch_mission(mission_id=<first_mission_id>)` を呼び出す。`plan_project` が `auto_dispatch=true` を設定しているため、残りのミッションは依存関係の完了に伴い自動ディスパッチされる。
4. **監視**: `get_mission_status(mission_id=...)` または `get_dashboard()` を呼び出して進捗を確認。
5. **レポート**: ミッション完了時に `get_report(mission_id=...)` を呼び出す。ハイライトをユーザーと共有する。

### 同時実行

DevFleet はデフォルトで最大 3 つの同時エージェントを実行する（`DEVFLEET_MAX_AGENTS` で設定可能）。すべてのスロットが埋まっている場合、`auto_dispatch=true` のミッションはミッションウォッチャーのキューに入り、スロットが空くと自動的にディスパッチされる。現在のスロット使用状況は `get_dashboard()` で確認。

## 例

### フルオート：計画と起動

1. `plan_project(prompt="...")` → ミッションと依存関係を含む計画を表示。
2. 最初のミッション（`depends_on` が空のもの）をディスパッチ。
3. 依存関係が解決されると残りのミッションが自動ディスパッチされる（`auto_dispatch=true` が設定済み）。
4. プロジェクト ID とミッション数をユーザーに報告し、何が起動されたかを伝える。
5. すべてのミッションが最終状態（`completed`、`failed`、または `cancelled`）に達するまで、`get_mission_status` または `get_dashboard()` で定期的にポーリング。
6. 各最終状態のミッションについて `get_report(mission_id=...)` — 成功をまとめ、エラーと次のステップとともに失敗を報告。

### マニュアル：ステップバイステップ制御

1. `create_project(name="My Project")` → `project_id` を返す。
2. 最初の（ルート）ミッションに `create_mission(project_id=project_id, title="...", prompt="...", auto_dispatch=true)` → `root_mission_id` を取得。
   後続の各タスクに `create_mission(project_id=project_id, title="...", prompt="...", auto_dispatch=true, depends_on=["<root_mission_id>"])`。
3. 最初のミッションに `dispatch_mission(mission_id=...)` でチェーンを開始。
4. 完了時に `get_report(mission_id=...)`。

### レビュー付きシーケンシャル

1. `create_project(name="...")` → `project_id` を取得。
2. `create_mission(project_id=project_id, title="Implement feature", prompt="...")` → `impl_mission_id` を取得。
3. `dispatch_mission(mission_id=impl_mission_id)` を実行し、完了まで `get_mission_status` でポーリング。
4. `get_report(mission_id=impl_mission_id)` で結果をレビュー。
5. `create_mission(project_id=project_id, title="Review", prompt="...", depends_on=[impl_mission_id], auto_dispatch=true)` — 依存関係がすでに満たされているため自動開始。

## ガイドライン

- ユーザーが事前に了承していない限り、ディスパッチ前に必ず計画をユーザーに確認する。
- ステータス報告時にはミッションタイトルと ID を含める。
- ミッションが失敗した場合、リトライ前にレポートを確認する。
- 一括ディスパッチ前に `get_dashboard()` でエージェントスロットの空き状況を確認する。
- ミッションの依存関係は DAG を構成する — 循環依存を作成しないこと。
- 各エージェントは分離された git ワークツリーで実行され、完了時に自動マージされる。マージコンフリクトが発生した場合、変更はエージェントのワークツリーブランチに残り、手動で解決する。
- 手動でミッションを作成する場合、依存関係の完了時に自動トリガーしたければ必ず `auto_dispatch=true` を設定する。このフラグがないと、ミッションは `draft` ステータスのまま残る。
