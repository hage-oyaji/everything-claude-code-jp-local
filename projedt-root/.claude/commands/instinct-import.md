---
name: instinct-import
description: ファイルまたはURLからインスティンクトをプロジェクト/グローバルスコープにインポート
command: true
---

# インスティンクトインポートコマンド

## 実装

プラグインのルートパスを使用してインスティンクトCLIを実行します：

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" import <file-or-url> [--dry-run] [--force] [--min-confidence 0.7] [--scope project|global]
```

`CLAUDE_PLUGIN_ROOT` が設定されていない場合（手動インストール）：

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py import <file-or-url>
```

ローカルファイルパスまたはHTTP(S) URLからインスティンクトをインポートします。

## 使い方

```
/instinct-import team-instincts.yaml
/instinct-import https://github.com/org/repo/instincts.yaml
/instinct-import team-instincts.yaml --dry-run
/instinct-import team-instincts.yaml --scope global --force
```

## 実行内容

1. インスティンクトファイルを取得（ローカルパスまたはURL）
2. フォーマットを解析・検証
3. 既存のインスティンクトとの重複をチェック
4. 新しいインスティンクトをマージまたは追加
5. 継承インスティンクトディレクトリに保存：
   - プロジェクトスコープ: `~/.claude/homunculus/projects/<project-id>/instincts/inherited/`
   - グローバルスコープ: `~/.claude/homunculus/instincts/inherited/`

## インポートプロセス

```
📥 Importing instincts from: team-instincts.yaml
================================================

Found 12 instincts to import.

Analyzing conflicts...

## New Instincts (8)
These will be added:
  ✓ use-zod-validation (confidence: 0.7)
  ✓ prefer-named-exports (confidence: 0.65)
  ✓ test-async-functions (confidence: 0.8)
  ...

## Duplicate Instincts (3)
Already have similar instincts:
  ⚠️ prefer-functional-style
     Local: 0.8 confidence, 12 observations
     Import: 0.7 confidence
     → Keep local (higher confidence)

  ⚠️ test-first-workflow
     Local: 0.75 confidence
     Import: 0.9 confidence
     → Update to import (higher confidence)

Import 8 new, update 1?
```

## マージ動作

既存のIDを持つインスティンクトをインポートする場合：
- 信頼度が高いインポートは更新候補になる
- 信頼度が同等以下のインポートはスキップされる
- `--force` を使用しない限り、ユーザーに確認を求める

## ソーストラッキング

インポートされたインスティンクトには以下のマークが付けられます：
```yaml
source: inherited
scope: project
imported_from: "team-instincts.yaml"
project_id: "a1b2c3d4e5f6"
project_name: "my-project"
```

## フラグ

- `--dry-run`: インポートせずにプレビュー
- `--force`: 確認プロンプトをスキップ
- `--min-confidence <n>`: しきい値を超えるインスティンクトのみインポート
- `--scope <project|global>`: ターゲットスコープを選択（デフォルト: `project`）

## 出力

インポート後：
```
✅ Import complete!

Added: 8 instincts
Updated: 1 instinct
Skipped: 3 instincts (equal/higher confidence already exists)

New instincts saved to: ~/.claude/homunculus/instincts/inherited/

Run /instinct-status to see all instincts.
```
