---
name: context-budget
description: Claude Codeセッション全体のエージェント、スキル、MCPサーバー、ルールにわたるコンテキストウィンドウ消費を監査。肥大化や冗長コンポーネントを特定し、優先順位付けされたトークン節約推奨事項を生成。
origin: ECC
---

# コンテキストバジェット

Claude Codeセッションでロードされるすべてのコンポーネントのトークンオーバーヘッドを分析し、コンテキストスペースを回復するための実行可能な最適化を提示する。

## 使用するとき

- セッションパフォーマンスが遅く感じたり、出力品質が低下しているとき
- 最近多くのスキル、エージェント、MCPサーバーを追加したとき
- 実際にどのくらいのコンテキスト余裕があるか知りたいとき
- さらにコンポーネントを追加する予定があり、空きがあるか確認する必要があるとき
- `/context-budget`コマンドを実行するとき（このスキルがバックエンド）

## 動作方法

### フェーズ1: インベントリ

すべてのコンポーネントディレクトリをスキャンし、トークン消費を推定する:

**エージェント** (`agents/*.md`)
- ファイルごとの行数とトークン数をカウント（単語数 × 1.3）
- `description`フロントマターの長さを抽出
- フラグ: 200行超のファイル（重い）、30語超のdescription（肥大化したフロントマター）

**スキル** (`skills/*/SKILL.md`)
- SKILL.mdごとのトークン数をカウント
- フラグ: 400行超のファイル
- `.agents/skills/`内の重複コピーを確認 — 同一コピーは二重カウントを避けるためスキップ

**ルール** (`rules/**/*.md`)
- ファイルごとのトークン数をカウント
- フラグ: 100行超のファイル
- 同じ言語モジュール内のルールファイル間のコンテンツ重複を検出

**MCPサーバー** (`.mcp.json`またはアクティブなMCP設定)
- 設定済みサーバー数と合計ツール数をカウント
- スキーマオーバーヘッドをツールあたり約500トークンとして推定
- フラグ: 20超のツールを持つサーバー、シンプルなCLIコマンドをラップするサーバー（`gh`、`git`、`npm`、`supabase`、`vercel`）

**CLAUDE.md**（プロジェクト + ユーザーレベル）
- CLAUDE.mdチェーン内のファイルごとのトークン数をカウント
- フラグ: 合計300行超

### フェーズ2: 分類

すべてのコンポーネントをバケットに分類する:

| バケット | 基準 | アクション |
|--------|----------|--------|
| **常に必要** | CLAUDE.mdで参照されている、アクティブなコマンドのバックエンド、現在のプロジェクトタイプに一致 | 保持 |
| **時々必要** | ドメイン固有（例: 言語パターン）、CLAUDE.mdで参照されていない | オンデマンドアクティベーションを検討 |
| **まれに必要** | コマンド参照なし、コンテンツが重複、明確なプロジェクトマッチなし | 削除またはレイジーロード |

### フェーズ3: 問題検出

以下の問題パターンを特定する:

- **肥大化したエージェントdescription** — フロントマターのdescriptionが30語超で、すべてのTask toolの呼び出しにロードされる
- **重いエージェント** — 200行超のファイルが、すべてのスポーン時にTask toolコンテキストを膨張させる
- **冗長コンポーネント** — エージェントロジックを複製するスキル、CLAUDE.mdを複製するルール
- **MCPの過剰サブスクリプション** — 10超のサーバー、または無料で利用可能なCLIツールをラップするサーバー
- **CLAUDE.mdの肥大化** — 冗長な説明、古くなったセクション、ルールであるべき指示

### フェーズ4: レポート

コンテキストバジェットレポートを生成する:

```
Context Budget Report
═══════════════════════════════════════

Total estimated overhead: ~XX,XXX tokens
Context model: Claude Sonnet (200K window)
Effective available context: ~XXX,XXX tokens (XX%)

Component Breakdown:
┌─────────────────┬────────┬───────────┐
│ Component       │ Count  │ Tokens    │
├─────────────────┼────────┼───────────┤
│ Agents          │ N      │ ~X,XXX    │
│ Skills          │ N      │ ~X,XXX    │
│ Rules           │ N      │ ~X,XXX    │
│ MCP tools       │ N      │ ~XX,XXX   │
│ CLAUDE.md       │ N      │ ~X,XXX    │
└─────────────────┴────────┴───────────┘

⚠ Issues Found (N):
[ranked by token savings]

Top 3 Optimizations:
1. [action] → save ~X,XXX tokens
2. [action] → save ~X,XXX tokens
3. [action] → save ~X,XXX tokens

Potential savings: ~XX,XXX tokens (XX% of current overhead)
```

詳細モードでは、さらにファイルごとのトークン数、最も重いファイルの行ごとの内訳、重複コンポーネント間の具体的な冗長行、ツールごとのスキーマサイズ推定付きMCPツールリストを出力する。

## 例

**基本監査**
```
User: /context-budget
Skill: Scans setup → 16 agents (12,400 tokens), 28 skills (6,200), 87 MCP tools (43,500), 2 CLAUDE.md (1,200)
       Flags: 3 heavy agents, 14 MCP servers (3 CLI-replaceable)
       Top saving: remove 3 MCP servers → -27,500 tokens (47% overhead reduction)
```

**詳細モード**
```
User: /context-budget --verbose
Skill: Full report + per-file breakdown showing planner.md (213 lines, 1,840 tokens),
       MCP tool list with per-tool sizes, duplicated rule lines side by side
```

**拡張前チェック**
```
User: I want to add 5 more MCP servers, do I have room?
Skill: Current overhead 33% → adding 5 servers (~50 tools) would add ~25,000 tokens → pushes to 45% overhead
       Recommendation: remove 2 CLI-replaceable servers first to stay under 40%
```

## ベストプラクティス

- **トークン推定**: 散文には`単語数 × 1.3`を使用、コード中心のファイルには`文字数 / 4`を使用
- **MCPが最大のレバー**: 各ツールスキーマは約500トークンのコスト; 30ツールのサーバーはすべてのスキルの合計よりも高コスト
- **エージェントdescriptionは常にロードされる**: エージェントが呼び出されなくても、そのdescriptionフィールドはすべてのTask toolコンテキストに存在する
- **デバッグには詳細モード**: オーバーヘッドの原因となる正確なファイルを特定する必要がある場合に使用、通常の監査には不要
- **変更後に監査**: エージェント、スキル、MCPサーバーを追加した後に実行し、肥大化を早期に検出する
