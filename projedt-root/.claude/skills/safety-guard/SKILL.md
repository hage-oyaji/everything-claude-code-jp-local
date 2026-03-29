# Safety Guard — 破壊的操作の防止

## 使用タイミング

- プロダクションシステムで作業するとき
- エージェントが自律的に実行されているとき（フルオートモード）
- 編集を特定のディレクトリに制限したいとき
- センシティブな操作中（マイグレーション、デプロイ、データ変更）

## 仕組み

3つの保護モード：

### モード 1: Carefulモード

破壊的コマンドを実行前にインターセプトして警告します：

```
Watched patterns:
- rm -rf (especially /, ~, or project root)
- git push --force
- git reset --hard
- git checkout . (discard all changes)
- DROP TABLE / DROP DATABASE
- docker system prune
- kubectl delete
- chmod 777
- sudo rm
- npm publish (accidental publishes)
- Any command with --no-verify
```

検出時：コマンドの内容を表示し、確認を求め、より安全な代替手段を提案します。

### モード 2: Freezeモード

ファイル編集を特定のディレクトリツリーにロックします：

```
/safety-guard freeze src/components/
```

`src/components/`外へのWrite/Editは説明付きでブロックされます。エージェントを特定の領域に集中させ、無関係なコードに触れさせたくない場合に有用です。

### モード 3: Guardモード（Careful + Freeze統合）

両方の保護が有効。自律エージェントのための最大の安全性。

```
/safety-guard guard --dir src/api/ --allow-read-all
```

エージェントはすべてを読み取れますが、書き込みは`src/api/`のみ。破壊的コマンドはどこでもブロックされます。

### ロック解除

```
/safety-guard off
```

## 実装

PreToolUseフックを使用してBash、Write、Edit、MultiEditツールコールをインターセプトします。実行を許可する前に、コマンド/パスをアクティブなルールと照合チェックします。

## 統合

- `codex -a never`セッションでデフォルトで有効化
- ECC 2.0のオブザーバビリティリスクスコアリングと組み合わせ
- すべてのブロックされたアクションを`~/.claude/safety-guard.log`に記録
