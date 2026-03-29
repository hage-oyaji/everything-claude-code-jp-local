# Checkpoint コマンド

ワークフローのチェックポイントを作成または検証します。

## 使い方

`/checkpoint [create|verify|list] [name]`

## チェックポイントの作成

チェックポイントを作成する場合:

1. `/verify quick` を実行して現在の状態がクリーンであることを確認する
2. チェックポイント名でgit stashまたはコミットを作成する
3. `.claude/checkpoints.log` にチェックポイントを記録する:

```bash
echo "$(date +%Y-%m-%d-%H:%M) | $CHECKPOINT_NAME | $(git rev-parse --short HEAD)" >> .claude/checkpoints.log
```

4. チェックポイント作成を報告する

## チェックポイントの検証

チェックポイントに対して検証する場合:

1. ログからチェックポイントを読み取る
2. 現在の状態をチェックポイントと比較する:
   - チェックポイント以降に追加されたファイル
   - チェックポイント以降に変更されたファイル
   - 現在のテスト合格率 vs チェックポイント時
   - 現在のカバレッジ vs チェックポイント時

3. レポート:
```
CHECKPOINT COMPARISON: $NAME
============================
Files changed: X
Tests: +Y passed / -Z failed
Coverage: +X% / -Y%
Build: [PASS/FAIL]
```

## チェックポイントの一覧

すべてのチェックポイントを以下の情報とともに表示する:
- 名前
- タイムスタンプ
- Git SHA
- ステータス（current, behind, ahead）

## ワークフロー

典型的なチェックポイントフロー:

```
[開始] --> /checkpoint create "feature-start"
   |
[実装] --> /checkpoint create "core-done"
   |
[テスト] --> /checkpoint verify "core-done"
   |
[リファクタリング] --> /checkpoint create "refactor-done"
   |
[PR] --> /checkpoint verify "feature-start"
```

## 引数

$ARGUMENTS:
- `create <name>` - 名前付きチェックポイントを作成
- `verify <name>` - 名前付きチェックポイントに対して検証
- `list` - すべてのチェックポイントを表示
- `clear` - 古いチェックポイントを削除（最新5件を保持）
