---
paths:
  - "**/*.kt"
  - "**/*.kts"
---
# Kotlin セキュリティ

> このファイルは [common/security.md](../common/security.md) を Kotlin および Android/KMP 固有の内容で拡張します。

## シークレット管理

- API キー、トークン、認証情報をソースコードにハードコードしない
- ローカル開発用シークレットには `local.properties`（git-ignored）を使用
- リリースビルドには CI シークレットから生成される `BuildConfig` フィールドを使用
- ランタイムのシークレット保存には `EncryptedSharedPreferences`（Android）または Keychain（iOS）を使用

```kotlin
// BAD
val apiKey = "sk-abc123..."

// GOOD — from BuildConfig (generated at build time)
val apiKey = BuildConfig.API_KEY

// GOOD — from secure storage at runtime
val token = secureStorage.get("auth_token")
```

## ネットワークセキュリティ

- HTTPS のみ使用 — `network_security_config.xml` でクリアテキストをブロックするよう設定
- 機密エンドポイントには OkHttp の `CertificatePinner` または Ktor 相当で証明書ピンニングを実施
- すべての HTTP クライアントにタイムアウトを設定 — デフォルト（無限の可能性あり）のまま放置しない
- すべてのサーバーレスポンスを使用前にバリデーションおよびサニタイズ

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <base-config cleartextTrafficPermitted="false" />
</network-security-config>
```

## 入力バリデーション

- すべてのユーザー入力を処理または API 送信前にバリデーション
- Room/SQLDelight ではパラメータ化クエリを使用 — ユーザー入力を SQL に連結しない
- パストラバーサル防止のためユーザー入力からのファイルパスをサニタイズ

```kotlin
// BAD — SQL injection
@Query("SELECT * FROM items WHERE name = '$input'")

// GOOD — parameterized
@Query("SELECT * FROM items WHERE name = :input")
fun findByName(input: String): List<ItemEntity>
```

## データ保護

- Android では機密なキーバリューデータに `EncryptedSharedPreferences` を使用
- 明示的なフィールド名で `@Serializable` を使用 — 内部プロパティ名を漏洩させない
- 不要になった機密データはメモリからクリア
- シリアライズされたクラスには `@Keep` または ProGuard ルールを使用し、名前マングリングを防止

## 認証

- トークンはセキュアストレージに保存。プレーンな SharedPreferences には保存しない
- 適切な 401/403 ハンドリングによるトークンリフレッシュを実装
- ログアウト時にすべての認証状態をクリア（トークン、キャッシュされたユーザーデータ、Cookie）
- 機密操作にはバイオメトリクス認証（`BiometricPrompt`）を使用

## ProGuard / R8

- すべてのシリアライズモデルの Keep ルール（`@Serializable`、Gson、Moshi）
- リフレクションベースのライブラリの Keep ルール（Koin、Retrofit）
- リリースビルドをテスト — 難読化はシリアライゼーションを暗黙的に破壊する可能性がある

## WebView セキュリティ

- 明示的に必要でない限り JavaScript を無効化: `settings.javaScriptEnabled = false`
- WebView で読み込む前に URL をバリデーション
- 機密データにアクセスする `@JavascriptInterface` メソッドを公開しない
- ナビゲーション制御に `WebViewClient.shouldOverrideUrlLoading()` を使用
