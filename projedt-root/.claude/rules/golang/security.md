---
paths:
  - "**/*.go"
  - "**/go.mod"
  - "**/go.sum"
---
# Go セキュリティ

> このファイルは [common/security.md](../common/security.md) をGo固有の内容で拡張します。

## シークレット管理

```go
apiKey := os.Getenv("OPENAI_API_KEY")
if apiKey == "" {
    log.Fatal("OPENAI_API_KEY not configured")
}
```

## セキュリティスキャン

- 静的セキュリティ解析に**gosec**を使用する:
  ```bash
  gosec ./...
  ```

## コンテキストとタイムアウト

タイムアウト制御には必ず `context.Context` を使用する:

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
```
