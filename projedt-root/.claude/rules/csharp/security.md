---
paths:
  - "**/*.cs"
  - "**/*.csx"
  - "**/*.csproj"
  - "**/appsettings*.json"
---
# C# セキュリティ

> このファイルは [common/security.md](../common/security.md) をC#固有の内容で拡張します。

## シークレット管理

- ソースコードにAPIキー、トークン、接続文字列をハードコードしない
- ローカル開発には環境変数やユーザーシークレットを使用し、本番環境ではシークレットマネージャを使用する
- `appsettings.*.json` に実際の認証情報を含めない

```csharp
// BAD
const string ApiKey = "sk-live-123";

// GOOD
var apiKey = builder.Configuration["OpenAI:ApiKey"]
    ?? throw new InvalidOperationException("OpenAI:ApiKey is not configured.");
```

## SQLインジェクション防止

- ADO.NET、Dapper、またはEF Coreでは必ずパラメータ化クエリを使用する
- ユーザー入力をSQL文字列に連結しない
- 動的クエリ構成を使用する前にソートフィールドとフィルタ演算子を検証する

```csharp
const string sql = "SELECT * FROM Orders WHERE CustomerId = @customerId";
await connection.QueryAsync<Order>(sql, new { customerId });
```

## 入力バリデーション

- アプリケーション境界でDTOを検証する
- データアノテーション、FluentValidation、または明示的なガード句を使用する
- ビジネスロジックを実行する前に無効なモデル状態を拒否する

## 認証と認可

- カスタムトークン解析よりもフレームワークの認証ハンドラを優先する
- エンドポイントまたはハンドラ境界で認可ポリシーを適用する
- 生のトークン、パスワード、PIIをログに記録しない

## エラーハンドリング

- 安全なクライアント向けメッセージを返す
- 詳細な例外は構造化コンテキストとともにサーバーサイドでログを記録する
- APIレスポンスにスタックトレース、SQLテキスト、ファイルシステムパスを公開しない

## 参考

スキル: `security-review` でより広範なアプリケーションセキュリティレビューチェックリストを参照してください。
