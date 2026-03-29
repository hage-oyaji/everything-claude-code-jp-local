---
paths:
  - "**/*.go"
  - "**/go.mod"
  - "**/go.sum"
---
# Go フック

> このファイルは [common/hooks.md](../common/hooks.md) をGo固有の内容で拡張します。

## PostToolUseフック

`~/.claude/settings.json` で設定する:

- **gofmt/goimports**: 編集後に `.go` ファイルを自動フォーマット
- **go vet**: `.go` ファイル編集後に静的解析を実行
- **staticcheck**: 変更されたパッケージに対して拡張静的チェックを実行
