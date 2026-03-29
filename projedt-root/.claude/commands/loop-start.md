# ループ開始コマンド

安全なデフォルト設定で管理された自律ループパターンを開始します。

## 使い方

`/loop-start [pattern] [--mode safe|fast]`

- `pattern`: `sequential`、`continuous-pr`、`rfc-dag`、`infinite`
- `--mode`:
  - `safe`（デフォルト）: 厳格な品質ゲートとチェックポイント
  - `fast`: 速度重視でゲートを削減

## フロー

1. リポジトリの状態とブランチ戦略を確認。
2. ループパターンとモデルティア戦略を選択。
3. 選択したモードに必要なフック/プロファイルを有効化。
4. ループプランを作成し、`.claude/plans/` にランブックを記述。
5. ループの開始と監視のためのコマンドを表示。

## 必須の安全チェック

- 最初のループイテレーション前にテストが通ることを確認。
- `ECC_HOOK_PROFILE` がグローバルに無効化されていないことを確認。
- ループに明示的な停止条件があることを確認。

## 引数

$ARGUMENTS:
- `<pattern>` オプション (`sequential|continuous-pr|rfc-dag|infinite`)
- `--mode safe|fast` オプション
