---
name: laravel-verification
description: Laravelプロジェクトの検証ループ — 環境チェック、リンティング、静的解析、カバレッジ付きテスト、セキュリティスキャン、デプロイメント準備。
origin: ECC
---

# Laravel検証ループ

PR作成前、大きな変更後、デプロイ前に実行。

## 使用タイミング

- LaravelプロジェクトのPR作成前
- 大規模リファクタリングや依存関係アップグレード後
- ステージングまたは本番へのデプロイ前検証
- 完全なリント -> テスト -> セキュリティ -> デプロイ準備パイプラインの実行

## 仕組み

- 環境チェックからデプロイメント準備まで順次フェーズを実行し、各レイヤーが前のレイヤーの上に構築されるようにする。
- 環境とComposerのチェックが他のすべてをゲートする。失敗した場合は即座に停止。
- リンティング/静的解析はフルテストとカバレッジの実行前にクリーンであるべき。
- セキュリティとマイグレーションのレビューはテスト後に行い、データやリリースステップの前に動作を検証。
- ビルド/デプロイ準備とキュー/スケジューラーチェックは最終ゲート。いずれかの失敗でリリースをブロック。

## フェーズ1: 環境チェック

```bash
php -v
composer --version
php artisan --version
```

- `.env`が存在し、必要なキーが存在することを確認
- 本番環境では`APP_DEBUG=false`であることを確認
- `APP_ENV`がターゲットデプロイメント（`production`、`staging`）と一致することを確認

Laravel Sailをローカルで使用する場合:

```bash
./vendor/bin/sail php -v
./vendor/bin/sail artisan --version
```

## フェーズ1.5: Composerとオートロード

```bash
composer validate
composer dump-autoload -o
```

## フェーズ2: リンティングと静的解析

```bash
vendor/bin/pint --test
vendor/bin/phpstan analyse
```

プロジェクトがPHPStanの代わりにPsalmを使用する場合:

```bash
vendor/bin/psalm
```

## フェーズ3: テストとカバレッジ

```bash
php artisan test
```

カバレッジ（CI）:

```bash
XDEBUG_MODE=coverage php artisan test --coverage
```

CIの例（フォーマット -> 静的解析 -> テスト）:

```bash
vendor/bin/pint --test
vendor/bin/phpstan analyse
XDEBUG_MODE=coverage php artisan test --coverage
```

## フェーズ4: セキュリティと依存関係チェック

```bash
composer audit
```

## フェーズ5: データベースとマイグレーション

```bash
php artisan migrate --pretend
php artisan migrate:status
```

- 破壊的なマイグレーションを慎重にレビュー
- マイグレーションファイル名が`Y_m_d_His_*`形式（例: `2025_03_14_154210_create_orders_table.php`）に従い、変更内容を明確に記述していることを確認
- ロールバックが可能であることを確認
- `down()`メソッドを検証し、明示的なバックアップなしに不可逆的なデータ損失を避ける

## フェーズ6: ビルドとデプロイメント準備

```bash
php artisan optimize:clear
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

- 本番設定でキャッシュウォームアップが成功することを確認
- キューワーカーとスケジューラーが設定されていることを検証
- ターゲット環境で`storage/`と`bootstrap/cache/`が書き込み可能であることを確認

## フェーズ7: キューとスケジューラーチェック

```bash
php artisan schedule:list
php artisan queue:failed
```

Horizonを使用する場合:

```bash
php artisan horizon:status
```

`queue:monitor`が利用可能な場合、ジョブを処理せずにバックログを確認:

```bash
php artisan queue:monitor default --max=100
```

アクティブな検証（ステージングのみ）: 専用キューにno-opジョブをディスパッチし、単一のワーカーを実行して処理する（非`sync`キュー接続が設定されていることを確認）。

```bash
php artisan tinker --execute="dispatch((new App\\Jobs\\QueueHealthcheck())->onQueue('healthcheck'))"
php artisan queue:work --once --queue=healthcheck
```

ジョブが期待される副作用（ログエントリ、ヘルスチェックテーブルの行、メトリクス）を生成したことを検証。

テストジョブの処理が安全な非本番環境でのみ実行してください。

## 事例

最小限のフロー:

```bash
php -v
composer --version
php artisan --version
composer validate
vendor/bin/pint --test
vendor/bin/phpstan analyse
php artisan test
composer audit
php artisan migrate --pretend
php artisan config:cache
php artisan queue:failed
```

CIスタイルのパイプライン:

```bash
composer validate
composer dump-autoload -o
vendor/bin/pint --test
vendor/bin/phpstan analyse
XDEBUG_MODE=coverage php artisan test --coverage
composer audit
php artisan migrate --pretend
php artisan optimize:clear
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan schedule:list
```
