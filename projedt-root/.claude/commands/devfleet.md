---
description: Claude DevFleetを介して並列のClaude Codeエージェントをオーケストレーション — 自然言語からプロジェクトを計画し、分離されたワークツリーでエージェントをディスパッチし、進捗を監視し、構造化されたレポートを読む。
---

# DevFleet — マルチエージェントオーケストレーション

Claude DevFleetを介して並列のClaude Codeエージェントをオーケストレーションします。各エージェントは完全なツールを備えた分離されたgitワークツリーで実行されます。

DevFleet MCPサーバーが必要です: `claude mcp add devfleet --transport http http://localhost:18801/mcp`

## フロー

```
ユーザーがプロジェクトを説明
  → plan_project(prompt) → 依存関係付きのミッションDAG
  → プランを表示、承認を取得
  → dispatch_mission(M1) → ワークツリーでエージェントが起動
  → M1完了 → 自動マージ → M2が自動ディスパッチ（M1に依存）
  → M2完了 → 自動マージ
  → get_report(M2) → files_changed, what_done, errors, next_steps
  → ユーザーにサマリーを報告
```

## ワークフロー

1. ユーザーの説明から**プロジェクトを計画**する:

```
mcp__devfleet__plan_project(prompt="<ユーザーの説明>")
```

これにより、チェーンされたミッションを持つプロジェクトが返されます。ユーザーに以下を表示する:
- プロジェクト名とID
- 各ミッション: タイトル、タイプ、依存関係
- 依存関係DAG（どのミッションがどのミッションをブロックするか）

2. ディスパッチ前に**ユーザーの承認を待つ**。プランを明確に表示する。

3. **最初のミッションをディスパッチ**する（`depends_on` が空のもの）:

```
mcp__devfleet__dispatch_mission(mission_id="<first_mission_id>")
```

残りのミッションは依存関係が完了すると自動的にディスパッチされます（`plan_project` が `auto_dispatch=true` で作成するため）。`create_mission` で手動作成する場合は、明示的に `auto_dispatch=true` を設定する必要があります。

4. **進捗を監視**する — 実行中の状態を確認:

```
mcp__devfleet__get_dashboard()
```

または特定のミッションを確認:

```
mcp__devfleet__get_mission_status(mission_id="<id>")
```

長時間実行されるミッションでは、`wait_for_mission` よりも `get_mission_status` でのポーリングを推奨します。これによりユーザーが進捗の更新を確認できます。

5. 完了した各ミッションの**レポートを読む**:

```
mcp__devfleet__get_report(mission_id="<mission_id>")
```

終了状態に達したすべてのミッションに対してこれを呼び出します。レポートには以下が含まれます: files_changed、what_done、what_open、what_tested、what_untested、next_steps、errors_encountered。

## 利用可能なツール一覧

| ツール | 用途 |
|------|---------|
| `plan_project(prompt)` | AIが説明を `auto_dispatch=true` のチェーンされたミッションに分解 |
| `create_project(name, path?, description?)` | プロジェクトを手動作成、`project_id` を返す |
| `create_mission(project_id, title, prompt, depends_on?, auto_dispatch?)` | ミッションを追加。`depends_on` はミッションID文字列のリスト。 |
| `dispatch_mission(mission_id, model?, max_turns?)` | エージェントを開始 |
| `cancel_mission(mission_id)` | 実行中のエージェントを停止 |
| `wait_for_mission(mission_id, timeout_seconds?)` | 完了までブロック（長いタスクにはポーリングを推奨） |
| `get_mission_status(mission_id)` | ブロックせずに進捗を確認 |
| `get_report(mission_id)` | 構造化されたレポートを読む |
| `get_dashboard()` | システム概要 |
| `list_projects()` | プロジェクト一覧 |
| `list_missions(project_id, status?)` | ミッション一覧 |

## ガイドライン

- ユーザーが「進めてください」と言わない限り、ディスパッチ前に必ずプランを確認する
- ステータス報告時にはミッションのタイトルとIDを含める
- ミッションが失敗した場合、再試行する前にレポートを読んでエラーを理解する
- エージェントの並行数は設定可能（デフォルト: 3）。超過したミッションはキューに入り、スロットが空くと自動ディスパッチされます。`get_dashboard()` でスロットの空き状況を確認。
- 依存関係はDAGを形成する — 循環依存を作成してはならない
- 各エージェントは完了時にワークツリーを自動マージします。マージコンフリクトが発生した場合、変更はワークツリーブランチに残り、手動解決が必要です。
