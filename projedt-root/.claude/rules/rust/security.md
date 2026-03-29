---
paths:
  - "**/*.rs"
---
# Rust セキュリティ

> このファイルは [common/security.md](../common/security.md) を Rust 固有の内容で拡張します。

## シークレット管理

- API キー、トークン、認証情報をソースコードにハードコードしない
- 環境変数を使用: `std::env::var("API_KEY")`
- 必要なシークレットが起動時に欠けている場合はフェイルファスト
- `.env` ファイルは `.gitignore` に含める

```rust
// BAD
const API_KEY: &str = "sk-abc123...";

// GOOD — environment variable with early validation
fn load_api_key() -> anyhow::Result<String> {
    std::env::var("PAYMENT_API_KEY")
        .context("PAYMENT_API_KEY must be set")
}
```

## SQL インジェクション防止

- 常にパラメータ化クエリを使用 — ユーザー入力を SQL 文字列にフォーマットしない
- バインドパラメータ付きのクエリビルダーまたは ORM（sqlx、diesel、sea-orm）を使用

```rust
// BAD — SQL injection via format string
let query = format!("SELECT * FROM users WHERE name = '{name}'");
sqlx::query(&query).fetch_one(&pool).await?;

// GOOD — parameterized query with sqlx
// Placeholder syntax varies by backend: Postgres: $1  |  MySQL: ?  |  SQLite: $1
sqlx::query("SELECT * FROM users WHERE name = $1")
    .bind(&name)
    .fetch_one(&pool)
    .await?;
```

## 入力バリデーション

- すべてのユーザー入力をシステム境界で処理前にバリデーション
- 型システムで不変条件を強制（newtype パターン）
- バリデーションではなくパース — 境界で非構造化データを型付き構造体に変換
- 無効な入力は明確なエラーメッセージで拒否

```rust
// Parse, don't validate — invalid states are unrepresentable
pub struct Email(String);

impl Email {
    pub fn parse(input: &str) -> Result<Self, ValidationError> {
        let trimmed = input.trim();
        let at_pos = trimmed.find('@')
            .filter(|&p| p > 0 && p < trimmed.len() - 1)
            .ok_or_else(|| ValidationError::InvalidEmail(input.to_string()))?;
        let domain = &trimmed[at_pos + 1..];
        if trimmed.len() > 254 || !domain.contains('.') {
            return Err(ValidationError::InvalidEmail(input.to_string()));
        }
        // For production use, prefer a validated email crate (e.g., `email_address`)
        Ok(Self(trimmed.to_string()))
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}
```

## Unsafe コード

- `unsafe` ブロックを最小限に — 安全な抽象化を優先
- すべての `unsafe` ブロックに不変条件を説明する `// SAFETY:` コメントが必須
- 便宜のために借用チェッカーをバイパスする目的で `unsafe` を使わない
- レビュー時にすべての `unsafe` コードを監査 — 正当な理由なしのレッドフラグ
- C ライブラリのラッパーは安全な FFI ラッパーを優先

```rust
// GOOD — safety comment documents ALL required invariants
let widget: &Widget = {
    // SAFETY: `ptr` is non-null, aligned, points to an initialized Widget,
    // and no mutable references or mutations exist for its lifetime.
    unsafe { &*ptr }
};

// BAD — no safety justification
unsafe { &*ptr }
```

## 依存関係のセキュリティ

- `cargo audit` で依存関係の既知の CVE をスキャン
- `cargo deny check` でライセンスとアドバイザリのコンプライアンスを確認
- `cargo tree` で推移的依存関係を監査
- 依存関係を最新に保つ — Dependabot または Renovate を設定
- 依存関係数を最小限に — 新しいクレートの追加前に評価

```bash
# Security audit
cargo audit

# Deny advisories, duplicate versions, and restricted licenses
cargo deny check

# Inspect dependency tree
cargo tree
cargo tree -d  # Show duplicates only
```

## エラーメッセージ

- API レスポンスに内部パス、スタックトレース、データベースエラーを公開しない
- 詳細なエラーはサーバーサイドでログ出力。クライアントには汎用メッセージを返す
- 構造化されたサーバーサイドロギングには `tracing` または `log` を使用

```rust
// Map errors to appropriate status codes and generic messages
// (Example uses axum; adapt the response type to your framework)
match order_service.find_by_id(id) {
    Ok(order) => Ok((StatusCode::OK, Json(order))),
    Err(ServiceError::NotFound(_)) => {
        tracing::info!(order_id = id, "order not found");
        Err((StatusCode::NOT_FOUND, "Resource not found"))
    }
    Err(e) => {
        tracing::error!(order_id = id, error = %e, "unexpected error");
        Err((StatusCode::INTERNAL_SERVER_ERROR, "Internal server error"))
    }
}
```

## 参考

スキル `rust-patterns` に unsafe コードのガイドラインと所有権パターンがあります。
スキル `security-review` に一般的なセキュリティチェックリストがあります。
