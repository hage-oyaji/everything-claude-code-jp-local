---
paths:
  - "**/*.php"
  - "**/phpunit.xml"
  - "**/phpunit.xml.dist"
  - "**/composer.json"
---
# PHP テスト

> このファイルは [common/testing.md](../common/testing.md) を PHP 固有の内容で拡張します。

## フレームワーク

デフォルトのテストフレームワークとして **PHPUnit** を使用。プロジェクトに **Pest** が設定されている場合は、新規テストに Pest を優先し、フレームワークの混在を避ける。

## カバレッジ

```bash
vendor/bin/phpunit --coverage-text
# or
vendor/bin/pest --coverage
```

CI では **pcov** または **Xdebug** を優先し、カバレッジ閾値は暗黙知ではなく CI で管理。

## テスト構成

- 高速なユニットテストとフレームワーク/データベースインテグレーションテストを分離。
- フィクスチャには大きな手書き配列ではなく、ファクトリ/ビルダーを使用。
- HTTP/コントローラテストはトランスポートとバリデーションに集中。ビジネスルールはサービスレベルのテストに移動。

## Inertia

プロジェクトで Inertia.js を使用している場合、生の JSON アサーションではなく `AssertableInertia` を使った `assertInertia` でコンポーネント名と props を検証。

## 参考

スキル `tdd-workflow` にリポジトリ全体の RED -> GREEN -> REFACTOR ループがあります。
スキル `laravel-tdd` に Laravel 固有のテストパターン（PHPUnit と Pest）があります。
