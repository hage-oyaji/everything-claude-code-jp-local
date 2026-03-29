---
paths:
  - "**/*.java"
---
# Java セキュリティ

> このファイルは [common/security.md](../common/security.md) をJava固有の内容で拡張します。

## シークレット管理

- ソースコードにAPIキー、トークン、認証情報をハードコードしない
- 環境変数を使用する: `System.getenv("API_KEY")`
- 本番環境のシークレットにはシークレットマネージャ（Vault、AWS Secrets Manager）を使用する
- シークレットを含むローカル設定ファイルは `.gitignore` に追加する

```java
// BAD
private static final String API_KEY = "sk-abc123...";

// GOOD — 環境変数
String apiKey = System.getenv("PAYMENT_API_KEY");
Objects.requireNonNull(apiKey, "PAYMENT_API_KEY must be set");
```

## SQLインジェクション防止

- 常にパラメータ化クエリを使用する — ユーザー入力をSQLに連結しない
- `PreparedStatement` またはフレームワークのパラメータ化クエリAPIを使用する
- ネイティブクエリで使用する入力はすべて検証・サニタイズする

```java
// BAD — 文字列連結によるSQLインジェクション
Statement stmt = conn.createStatement();
String sql = "SELECT * FROM orders WHERE name = '" + name + "'";
stmt.executeQuery(sql);

// GOOD — パラメータ化クエリのPreparedStatement
PreparedStatement ps = conn.prepareStatement("SELECT * FROM orders WHERE name = ?");
ps.setString(1, name);

// GOOD — JDBCテンプレート
jdbcTemplate.query("SELECT * FROM orders WHERE name = ?", mapper, name);
```

## 入力バリデーション

- 処理前にシステム境界ですべてのユーザー入力を検証する
- バリデーションフレームワーク使用時はDTOにBean Validation（`@NotNull`、`@NotBlank`、`@Size`）を使用する
- ファイルパスやユーザー提供の文字列は使用前にサニタイズする
- バリデーションに失敗した入力は明確なエラーメッセージで拒否する

```java
// プレーンJavaでの手動バリデーション
public Order createOrder(String customerName, BigDecimal amount) {
    if (customerName == null || customerName.isBlank()) {
        throw new IllegalArgumentException("Customer name is required");
    }
    if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) {
        throw new IllegalArgumentException("Amount must be positive");
    }
    return new Order(customerName, amount);
}
```

## 認証と認可

- カスタムの暗号化認証を実装しない — 確立されたライブラリを使用する
- パスワードはbcryptまたはArgon2で保存する、MD5/SHA1は使用しない
- サービス境界で認可チェックを強制する
- ログから機密データを除外する — パスワード、トークン、PIIをログに記録しない

## 依存関係のセキュリティ

- `mvn dependency:tree` または `./gradlew dependencies` を実行して推移的依存関係を監査する
- OWASP Dependency-CheckまたはSnykを使用して既知のCVEをスキャンする
- 依存関係を最新に保つ — DependabotまたはRenovateを設定する

## エラーメッセージ

- APIレスポンスにスタックトレース、内部パス、SQLエラーを公開しない
- ハンドラ境界で例外を安全で汎用的なクライアントメッセージにマッピングする
- 詳細なエラーはサーバーサイドでログに記録し、クライアントには汎用メッセージを返す

```java
// 詳細をログに記録し、汎用メッセージを返す
try {
    return orderService.findById(id);
} catch (OrderNotFoundException ex) {
    log.warn("Order not found: id={}", id);
    return ApiResponse.error("Resource not found");  // generic, no internals
} catch (Exception ex) {
    log.error("Unexpected error processing order id={}", id, ex);
    return ApiResponse.error("Internal server error");  // never expose ex.getMessage()
}
```

## 参考

スキル: `springboot-security` でSpring Securityの認証・認可パターンを参照してください。
スキル: `security-review` で一般的なセキュリティチェックリストを参照してください。
