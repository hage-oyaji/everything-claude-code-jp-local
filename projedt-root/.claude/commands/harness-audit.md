# ハーネス監査コマンド

決定論的なリポジトリハーネス監査を実行し、優先順位付きのスコアカードを返します。

## 使い方

`/harness-audit [scope] [--format text|json]`

- `scope`（オプション）: `repo`（デフォルト）、`hooks`、`skills`、`commands`、`agents`
- `--format`: 出力スタイル（`text` がデフォルト、自動化用に `json`）

## 決定論的エンジン

常に以下を実行する:

```bash
node scripts/harness-audit.js <scope> --format <text|json>
```

このスクリプトがスコアリングとチェックの情報源です。追加のディメンションやアドホックなポイントを作り出してはなりません。

ルーブリックバージョン: `2026-03-16`

スクリプトは7つの固定カテゴリ（各 `0-10` に正規化）を計算します:

1. ツールカバレッジ
2. コンテキスト効率
3. 品質ゲート
4. メモリ永続性
5. 評価カバレッジ
6. セキュリティガードレール
7. コスト効率

スコアは明示的なファイル/ルールチェックから導出され、同じコミットに対して再現可能です。

## 出力規約

以下を返す:

1. `overall_score` / `max_score`（`repo` の場合は70、スコープ監査の場合はそれより小さい）
2. カテゴリスコアと具体的な所見
3. 正確なファイルパスを含む失敗したチェック
4. 決定論的出力からのトップ3アクション（`top_actions`）
5. 次に適用すべきECCスキルの提案

## チェックリスト

- スクリプト出力をそのまま使用する。手動で再スコアリングしない。
- `--format json` がリクエストされた場合、スクリプトのJSONをそのまま返す。
- テキストがリクエストされた場合、失敗したチェックとトップアクションを要約する。
- `checks[]` と `top_actions[]` からの正確なファイルパスを含める。

## 結果の例

```text
Harness Audit (repo): 66/70
- Tool Coverage: 10/10 (10/10 pts)
- Context Efficiency: 9/10 (9/10 pts)
- Quality Gates: 10/10 (10/10 pts)

Top 3 Actions:
1) [Security Guardrails] Add prompt/tool preflight security guards in hooks/hooks.json. (hooks/hooks.json)
2) [Tool Coverage] Sync commands/harness-audit.md and .opencode/commands/harness-audit.md. (.opencode/commands/harness-audit.md)
3) [Eval Coverage] Increase automated test coverage across scripts/hooks/lib. (tests/)
```

## 引数

$ARGUMENTS:
- `repo|hooks|skills|commands|agents`（オプションのスコープ）
- `--format text|json`（オプションの出力フォーマット）
