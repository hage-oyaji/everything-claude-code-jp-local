---
paths:
  - "**/*.go"
  - "**/go.mod"
  - "**/go.sum"
---
# Go コーディングスタイル

> このファイルは [common/coding-style.md](../common/coding-style.md) をGo固有の内容で拡張します。

## フォーマット

- **gofmt** と **goimports** は必須 — スタイルの議論は不要

## 設計原則

- インターフェースを受け取り、構造体を返す
- インターフェースは小さく保つ（1〜3メソッド）

## エラーハンドリング

エラーには必ずコンテキストを付加してラップする:

```go
if err != nil {
    return fmt.Errorf("failed to create user: %w", err)
}
```

## 参考

スキル: `golang-patterns` で包括的なGoのイディオムとパターンを参照してください。
