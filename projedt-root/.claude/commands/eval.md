# Eval コマンド

評価駆動開発ワークフローを管理します。

## 使い方

`/eval [define|check|report|list] [feature-name]`

## 評価の定義

`/eval define feature-name`

新しい評価定義を作成する:

1. `.claude/evals/feature-name.md` にテンプレートを作成する:

```markdown
## EVAL: feature-name
Created: $(date)

### 機能評価
- [ ] [機能1の説明]
- [ ] [機能2の説明]

### リグレッション評価
- [ ] [既存の動作1が引き続き動作する]
- [ ] [既存の動作2が引き続き動作する]

### 成功基準
- 機能評価のpass@3 > 90%
- リグレッション評価のpass^3 = 100%
```

2. 具体的な基準の入力をユーザーに促す

## 評価のチェック

`/eval check feature-name`

機能の評価を実行する:

1. `.claude/evals/feature-name.md` から評価定義を読み取る
2. 各機能評価について:
   - 基準の検証を試みる
   - PASS/FAILを記録する
   - 試行を `.claude/evals/feature-name.log` に記録する
3. 各リグレッション評価について:
   - 関連するテストを実行する
   - ベースラインと比較する
   - PASS/FAILを記録する
4. 現在のステータスを報告する:

```
EVAL CHECK: feature-name
========================
機能: X/Y 合格
リグレッション: X/Y 合格
ステータス: 進行中 / 準備完了
```

## 評価のレポート

`/eval report feature-name`

包括的な評価レポートを生成する:

```
EVAL REPORT: feature-name
=========================
生成日: $(date)

機能評価
----------------
[eval-1]: PASS (pass@1)
[eval-2]: PASS (pass@2) - 再試行が必要だった
[eval-3]: FAIL - 備考を参照

リグレッション評価
----------------
[test-1]: PASS
[test-2]: PASS
[test-3]: PASS

メトリクス
-------
機能 pass@1: 67%
機能 pass@3: 100%
リグレッション pass^3: 100%

備考
-----
[問題、エッジケース、または所見]

推奨
--------------
[SHIP / 要修正 / ブロック中]
```

## 評価の一覧

`/eval list`

すべての評価定義を表示する:

```
評価定義一覧
================
feature-auth      [3/5 合格] 進行中
feature-search    [5/5 合格] 準備完了
feature-export    [0/4 合格] 未開始
```

## 引数

$ARGUMENTS:
- `define <name>` - 新しい評価定義を作成
- `check <name>` - 評価を実行してチェック
- `report <name>` - 完全なレポートを生成
- `list` - すべての評価を表示
- `clean` - 古い評価ログを削除（最新10回を保持）
