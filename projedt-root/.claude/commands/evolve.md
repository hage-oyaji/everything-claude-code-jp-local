---
name: evolve
description: インスティンクトを分析し、進化した構造を提案または生成する
command: true
---

# Evolve コマンド

## 実装

プラグインルートパスを使用してインスティンクトCLIを実行する:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" evolve [--generate]
```

または `CLAUDE_PLUGIN_ROOT` が設定されていない場合（手動インストール）:

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py evolve [--generate]
```

インスティンクトを分析し、関連するものをより高次の構造にクラスタリングする:
- **コマンド**: インスティンクトがユーザーが呼び出すアクションを記述している場合
- **スキル**: インスティンクトが自動的にトリガーされる動作を記述している場合
- **エージェント**: インスティンクトが複雑なマルチステッププロセスを記述している場合

## 使い方

```
/evolve                    # 全インスティンクトを分析し、進化を提案
/evolve --generate         # 加えてevolved/{skills,commands,agents}配下にファイルを生成
```

## 進化ルール

### → コマンド（ユーザー呼び出し型）
インスティンクトがユーザーが明示的に要求するアクションを記述している場合:
- 「ユーザーが...を要求した時」に関する複数のインスティンクト
- 「新しいXを作成する時」のようなトリガーを持つインスティンクト
- 繰り返し可能なシーケンスに従うインスティンクト

例:
- `new-table-step1`: 「データベーステーブルを追加する時、マイグレーションを作成」
- `new-table-step2`: 「データベーステーブルを追加する時、スキーマを更新」
- `new-table-step3`: 「データベーステーブルを追加する時、型を再生成」

→ 作成: **new-table** コマンド

### → スキル（自動トリガー型）
インスティンクトが自動的に発生すべき動作を記述している場合:
- パターンマッチングトリガー
- エラーハンドリングのレスポンス
- コーディングスタイルの強制

例:
- `prefer-functional`: 「関数を書く時、関数型スタイルを優先」
- `use-immutable`: 「状態を変更する時、イミュータブルパターンを使用」
- `avoid-classes`: 「モジュールを設計する時、クラスベースの設計を避ける」

→ 作成: `functional-patterns` スキル

### → エージェント（深さ/分離が必要な場合）
インスティンクトが分離の恩恵を受ける複雑なマルチステッププロセスを記述している場合:
- デバッグワークフロー
- リファクタリングシーケンス
- リサーチタスク

例:
- `debug-step1`: 「デバッグ時、まずログを確認」
- `debug-step2`: 「デバッグ時、失敗コンポーネントを分離」
- `debug-step3`: 「デバッグ時、最小再現を作成」
- `debug-step4`: 「デバッグ時、テストで修正を検証」

→ 作成: **debugger** エージェント

## 実行内容

1. 現在のプロジェクトコンテキストを検出する
2. プロジェクト + グローバルのインスティンクトを読み取る（IDの競合ではプロジェクトが優先）
3. インスティンクトをトリガー/ドメインパターンでグループ化する
4. 以下を特定する:
   - スキル候補（2つ以上のインスティンクトを持つトリガークラスター）
   - コマンド候補（高信頼度のワークフローインスティンクト）
   - エージェント候補（より大きな高信頼度のクラスター）
5. 該当する場合、昇格候補（プロジェクト -> グローバル）を表示する
6. `--generate` が指定された場合、以下にファイルを書き込む:
   - プロジェクトスコープ: `~/.claude/homunculus/projects/<project-id>/evolved/`
   - グローバルフォールバック: `~/.claude/homunculus/evolved/`

## 出力フォーマット

```
============================================================
  EVOLVE ANALYSIS - 12 instincts
  Project: my-app (a1b2c3d4e5f6)
  Project-scoped: 8 | Global: 4
============================================================

High confidence instincts (>=80%): 5

## SKILL CANDIDATES
1. Cluster: "adding tests"
   Instincts: 3
   Avg confidence: 82%
   Domains: testing
   Scopes: project

## COMMAND CANDIDATES (2)
  /adding-tests
    From: test-first-workflow [project]
    Confidence: 84%

## AGENT CANDIDATES (1)
  adding-tests-agent
    Covers 3 instincts
    Avg confidence: 82%
```

## フラグ

- `--generate`: 分析出力に加えて進化したファイルを生成する

## 生成されるファイルのフォーマット

### コマンド
```markdown
---
name: new-table
description: Create a new database table with migration, schema update, and type generation
command: /new-table
evolved_from:
  - new-table-migration
  - update-schema
  - regenerate-types
---

# New Table Command

[クラスタリングされたインスティンクトに基づいて生成されたコンテンツ]

## Steps
1. ...
2. ...
```

### スキル
```markdown
---
name: functional-patterns
description: Enforce functional programming patterns
evolved_from:
  - prefer-functional
  - use-immutable
  - avoid-classes
---

# Functional Patterns Skill

[クラスタリングされたインスティンクトに基づいて生成されたコンテンツ]
```

### エージェント
```markdown
---
name: debugger
description: Systematic debugging agent
model: sonnet
evolved_from:
  - debug-check-logs
  - debug-isolate
  - debug-reproduce
---

# Debugger Agent

[クラスタリングされたインスティンクトに基づいて生成されたコンテンツ]
```
