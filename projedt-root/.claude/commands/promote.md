---
name: promote
description: プロジェクトスコープのインスティンクトをグローバルスコープにプロモート
command: true
---

# プロモートコマンド

continuous-learning-v2でインスティンクトをプロジェクトスコープからグローバルスコープにプロモートします。

## 実装

プラグインのルートパスを使用してインスティンクトCLIを実行します：

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" promote [instinct-id] [--force] [--dry-run]
```

`CLAUDE_PLUGIN_ROOT` が設定されていない場合（手動インストール）：

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py promote [instinct-id] [--force] [--dry-run]
```

## 使い方

```bash
/promote                      # Auto-detect promotion candidates
/promote --dry-run            # Preview auto-promotion candidates
/promote --force              # Promote all qualified candidates without prompt
/promote grep-before-edit     # Promote one specific instinct from current project
```

## 実行内容

1. 現在のプロジェクトを検出
2. `instinct-id` が指定されている場合、そのインスティンクトのみをプロモート（現在のプロジェクトに存在する場合）
3. それ以外の場合、以下の条件を満たすクロスプロジェクト候補を検出：
   - 2つ以上のプロジェクトに存在する
   - 信頼度のしきい値を満たす
4. プロモートされたインスティンクトを `~/.claude/homunculus/instincts/personal/` に `scope: global` で書き込む
