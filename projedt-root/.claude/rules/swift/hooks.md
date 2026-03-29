---
paths:
  - "**/*.swift"
  - "**/Package.swift"
---
# Swift フック

> このファイルは [common/hooks.md](../common/hooks.md) を Swift 固有の内容で拡張します。

## PostToolUse フック

`~/.claude/settings.json` で設定：

- **SwiftFormat**: 編集後に `.swift` ファイルを自動フォーマット
- **SwiftLint**: `.swift` ファイル編集後にリントチェックを実行
- **swift build**: 編集後に変更されたパッケージの型チェック

## 警告

`print()` 文をフラグ — プロダクションコードでは代わりに `os.Logger` または構造化ロギングを使用。
