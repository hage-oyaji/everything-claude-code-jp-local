# ドキュメントの更新

コードベースと同期したドキュメントを、信頼できるソースファイルから生成します。

## ステップ1：信頼できるソースの特定

| ソース | 生成内容 |
|--------|-----------|
| `package.json`のscripts | 利用可能なコマンドリファレンス |
| `.env.example` | 環境変数ドキュメント |
| `openapi.yaml` / ルートファイル | APIエンドポイントリファレンス |
| ソースコードのエクスポート | パブリックAPIドキュメント |
| `Dockerfile` / `docker-compose.yml` | インフラセットアップドキュメント |

## ステップ2：スクリプトリファレンスの生成

1. `package.json`（または`Makefile`、`Cargo.toml`、`pyproject.toml`）を読み取る
2. すべてのスクリプト/コマンドとその説明を抽出
3. リファレンステーブルを生成：

```markdown
| Command | Description |
|---------|-------------|
| `npm run dev` | Start development server with hot reload |
| `npm run build` | Production build with type checking |
| `npm test` | Run test suite with coverage |
```

## ステップ3：環境変数ドキュメントの生成

1. `.env.example`（または`.env.template`、`.env.sample`）を読み取る
2. すべての変数とその目的を抽出
3. 必須とオプションに分類
4. 期待される形式と有効な値を記録

```markdown
| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `DATABASE_URL` | Yes | PostgreSQL connection string | `postgres://user:pass@host:5432/db` |
| `LOG_LEVEL` | No | Logging verbosity (default: info) | `debug`, `info`, `warn`, `error` |
```

## ステップ4：コントリビューティングガイドの更新

`docs/CONTRIBUTING.md`を生成または更新します：
- 開発環境セットアップ（前提条件、インストール手順）
- 利用可能なスクリプトとその目的
- テスト手順（実行方法、新しいテストの記述方法）
- コードスタイルの強制（リンター、フォーマッター、pre-commitフック）
- PRサブミッションチェックリスト

## ステップ5：ランブックの更新

`docs/RUNBOOK.md`を生成または更新します：
- デプロイ手順（ステップバイステップ）
- ヘルスチェックエンドポイントと監視
- よくある問題とその修正
- ロールバック手順
- アラートとエスカレーションパス

## ステップ6：陳腐化チェック

1. 90日以上変更されていないドキュメントファイルを検出
2. 最近のソースコード変更と相互参照
3. 手動レビューが必要な可能性のある古いドキュメントにフラグを立てる

## ステップ7：サマリーの表示

```
Documentation Update
──────────────────────────────
Updated:  docs/CONTRIBUTING.md (scripts table)
Updated:  docs/ENV.md (3 new variables)
Flagged:  docs/DEPLOY.md (142 days stale)
Skipped:  docs/API.md (no changes detected)
──────────────────────────────
```

## ルール

- **信頼できるソースは1つ**：常にコードから生成し、生成されたセクションを手動で編集しない
- **手動セクションを保持**：生成されたセクションのみ更新。手書きの文章はそのまま残す
- **生成されたコンテンツをマーク**：生成されたセクションの周りに`<!-- AUTO-GENERATED -->`マーカーを使用
- **求められていないドキュメントを作成しない**：コマンドが明示的に要求した場合にのみ新しいドキュメントファイルを作成
