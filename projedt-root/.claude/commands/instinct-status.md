---
name: instinct-status
description: 学習済みインスティンクト（プロジェクト＋グローバル）を信頼度とともに表示
command: true
---

# インスティンクトステータスコマンド

現在のプロジェクトの学習済みインスティンクトとグローバルインスティンクトを、ドメインごとにグループ化して表示します。

## 実装

プラグインのルートパスを使用してインスティンクトCLIを実行します：

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" status
```

`CLAUDE_PLUGIN_ROOT` が設定されていない場合（手動インストール）：

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py status
```

## 使い方

```
/instinct-status
```

## 実行内容

1. 現在のプロジェクトコンテキストを検出（gitリモート/パスハッシュ）
2. `~/.claude/homunculus/projects/<project-id>/instincts/` からプロジェクトインスティンクトを読み込み
3. `~/.claude/homunculus/instincts/` からグローバルインスティンクトを読み込み
4. 優先ルールに従ってマージ（IDが衝突した場合はプロジェクトがグローバルを上書き）
5. ドメインごとにグループ化し、信頼度バーと観測統計とともに表示

## 出力フォーマット

```
============================================================
  INSTINCT STATUS - 12 total
============================================================

  Project: my-app (a1b2c3d4e5f6)
  Project instincts: 8
  Global instincts:  4

## PROJECT-SCOPED (my-app)
  ### WORKFLOW (3)
    ███████░░░  70%  grep-before-edit [project]
              trigger: when modifying code

## GLOBAL (apply to all projects)
  ### SECURITY (2)
    █████████░  85%  validate-user-input [global]
              trigger: when handling user input
```
