---
name: continuous-agent-loop
description: 品質ゲート、評価、リカバリー制御を備えた継続的自律エージェントループのパターン。
origin: ECC
---

# 継続的エージェントループ

v1.8+の正式なループスキル名。`autonomous-loops`を後継しつつ、1リリース分の互換性を維持。

## ループ選択フロー

```text
Start
  |
  +-- Need strict CI/PR control? -- yes --> continuous-pr
  |                                    
  +-- Need RFC decomposition? -- yes --> rfc-dag
  |
  +-- Need exploratory parallel generation? -- yes --> infinite
  |
  +-- default --> sequential
```

## 組み合わせパターン

推奨プロダクションスタック:
1. RFC分解 (`ralphinho-rfc-pipeline`)
2. 品質ゲート (`plankton-code-quality` + `/quality-gate`)
3. 評価ループ (`eval-harness`)
4. セッション永続化 (`nanoclaw-repl`)

## 失敗モード

- 測定可能な進捗なしのループチャーン
- 同じ根本原因での繰り返しリトライ
- マージキューの停滞
- 上限のないエスカレーションによるコストドリフト

## リカバリー

- ループを凍結する
- `/harness-audit`を実行する
- スコープを失敗しているユニットに縮小する
- 明示的な受け入れ基準を付けてリプレイする
