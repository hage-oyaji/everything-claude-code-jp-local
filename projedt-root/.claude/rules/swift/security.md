---
paths:
  - "**/*.swift"
  - "**/Package.swift"
---
# Swift セキュリティ

> このファイルは [common/security.md](../common/security.md) を Swift 固有の内容で拡張します。

## シークレット管理

- 機密データ（トークン、パスワード、キー）には **Keychain Services** を使用 — `UserDefaults` は使わない
- ビルド時シークレットには環境変数または `.xcconfig` ファイルを使用
- ソースにシークレットをハードコードしない — デコンパイルツールで容易に抽出される

```swift
let apiKey = ProcessInfo.processInfo.environment["API_KEY"]
guard let apiKey, !apiKey.isEmpty else {
    fatalError("API_KEY not configured")
}
```

## トランスポートセキュリティ

- App Transport Security（ATS）はデフォルトで適用 — 無効にしない
- 重要なエンドポイントには証明書ピンニングを使用
- すべてのサーバー証明書をバリデーション

## 入力バリデーション

- インジェクション防止のため、表示前にすべてのユーザー入力をサニタイズ
- 強制アンラップではなくバリデーション付きの `URL(string:)` を使用
- 外部ソース（API、ディープリンク、ペーストボード）からのデータを処理前にバリデーション
