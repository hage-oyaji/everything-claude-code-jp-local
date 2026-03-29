---
name: instinct-export
description: プロジェクト/グローバルスコープからインスティンクトをファイルにエクスポートする
command: /instinct-export
---

# インスティンクトエクスポートコマンド

インスティンクトを共有可能なフォーマットにエクスポートします。以下の用途に最適です:
- チームメイトとの共有
- 新しいマシンへの移行
- プロジェクト規約への貢献

## 使い方

```
/instinct-export                           # 個人の全インスティンクトをエクスポート
/instinct-export --domain testing          # テスト関連のインスティンクトのみエクスポート
/instinct-export --min-confidence 0.7      # 高信頼度のインスティンクトのみエクスポート
/instinct-export --output team-instincts.yaml
/instinct-export --scope project --output project-instincts.yaml
```

## 実行内容

1. 現在のプロジェクトコンテキストを検出する
2. 選択されたスコープでインスティンクトを読み込む:
   - `project`: 現在のプロジェクトのみ
   - `global`: グローバルのみ
   - `all`: プロジェクト + グローバルをマージ（デフォルト）
3. フィルターを適用する（`--domain`、`--min-confidence`）
4. YAMLスタイルのエクスポートをファイルに書き込む（出力パスが未指定の場合は標準出力）

## 出力フォーマット

YAMLファイルを生成する:

```yaml
# Instincts Export
# Generated: 2025-01-22
# Source: personal
# Count: 12 instincts

---
id: prefer-functional-style
trigger: "when writing new functions"
confidence: 0.8
domain: code-style
source: session-observation
scope: project
project_id: a1b2c3d4e5f6
project_name: my-app
---

# Prefer Functional Style

## Action
Use functional patterns over classes.
```

## フラグ

- `--domain <name>`: 指定されたドメインのみエクスポート
- `--min-confidence <n>`: 最小信頼度しきい値
- `--output <file>`: 出力ファイルパス（省略時は標準出力に出力）
- `--scope <project|global|all>`: エクスポートスコープ（デフォルト: `all`）
