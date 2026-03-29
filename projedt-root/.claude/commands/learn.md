# /learn - 再利用可能なパターンの抽出

現在のセッションを分析し、スキルとして保存する価値のあるパターンを抽出します。

## トリガー

セッション中に非自明な問題を解決した時点で `/learn` を実行してください。

## 抽出対象

以下を探します：

1. **エラー解決パターン**
   - どのようなエラーが発生したか？
   - 根本原因は何だったか？
   - 何が修正したか？
   - 類似のエラーに再利用できるか？

2. **デバッグテクニック**
   - 非自明なデバッグステップ
   - 効果的だったツールの組み合わせ
   - 診断パターン

3. **ワークアラウンド**
   - ライブラリの癖
   - APIの制限
   - バージョン固有の修正

4. **プロジェクト固有のパターン**
   - 発見されたコードベースの規約
   - 行われたアーキテクチャの決定
   - 統合パターン

## 出力フォーマット

`~/.claude/skills/learned/[pattern-name].md` にスキルファイルを作成：

```markdown
# [Descriptive Pattern Name]

**Extracted:** [Date]
**Context:** [Brief description of when this applies]

## Problem
[What problem this solves - be specific]

## Solution
[The pattern/technique/workaround]

## Example
[Code example if applicable]

## When to Use
[Trigger conditions - what should activate this skill]
```

## プロセス

1. セッションを確認し、抽出可能なパターンを探す
2. 最も価値のある/再利用可能なインサイトを特定
3. スキルファイルをドラフト
4. 保存前にユーザーに確認を求める
5. `~/.claude/skills/learned/` に保存

## 注意事項

- 些末な修正（タイポ、単純な構文エラー）は抽出しない
- 一回限りの問題（特定のAPI障害など）は抽出しない
- 将来のセッションで時間を節約するパターンに焦点を当てる
- スキルは焦点を絞る — 1スキル1パターン
