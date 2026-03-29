# コードマップの更新

コードベースの構造を分析し、トークン効率の良いアーキテクチャドキュメントを生成します。

## ステップ1：プロジェクト構造のスキャン

1. プロジェクトタイプを特定（モノレポ、単一アプリ、ライブラリ、マイクロサービス）
2. すべてのソースディレクトリを検出（src/、lib/、app/、packages/）
3. エントリーポイントをマッピング（main.ts、index.ts、app.py、main.goなど）

## ステップ2：コードマップの生成

`docs/CODEMAPS/`（または`.reports/codemaps/`）にコードマップを作成または更新します：

| ファイル | 内容 |
|------|----------|
| `architecture.md` | 高レベルのシステム図、サービス境界、データフロー |
| `backend.md` | APIルート、ミドルウェアチェーン、サービス→リポジトリマッピング |
| `frontend.md` | ページツリー、コンポーネント階層、状態管理フロー |
| `data.md` | データベーステーブル、リレーションシップ、マイグレーション履歴 |
| `dependencies.md` | 外部サービス、サードパーティ統合、共有ライブラリ |

### コードマップフォーマット

各コードマップはトークン効率が良い — AIコンテキスト消費に最適化されたものにします：

```markdown
# Backend Architecture

## Routes
POST /api/users → UserController.create → UserService.create → UserRepo.insert
GET  /api/users/:id → UserController.get → UserService.findById → UserRepo.findById

## Key Files
src/services/user.ts (business logic, 120 lines)
src/repos/user.ts (database access, 80 lines)

## Dependencies
- PostgreSQL (primary data store)
- Redis (session cache, rate limiting)
- Stripe (payment processing)
```

## ステップ3：差分検出

1. 以前のコードマップが存在する場合、差分パーセンテージを計算
2. 変更が30%を超える場合、差分を表示してユーザーの承認を要求してから上書き
3. 変更が30%以下の場合、その場で更新

## ステップ4：メタデータの追加

各コードマップに鮮度ヘッダーを追加します：

```markdown
<!-- Generated: 2026-02-11 | Files scanned: 142 | Token estimate: ~800 -->
```

## ステップ5：分析レポートの保存

`.reports/codemap-diff.txt`にサマリーを書き込みます：
- 前回のスキャン以降に追加/削除/変更されたファイル
- 検出された新しい依存関係
- アーキテクチャの変更（新しいルート、新しいサービスなど）
- 90日以上更新されていないドキュメントの陳腐化警告

## ヒント

- 実装の詳細ではなく**高レベルの構造**に注力する
- 完全なコードブロックよりも**ファイルパスと関数シグネチャ**を優先する
- 効率的なコンテキストロードのため各コードマップを**1000トークン未満**に抑える
- 冗長な説明の代わりにASCII図でデータフローを表現する
- 大きな機能追加やリファクタリングセッションの後に実行する
