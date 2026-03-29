---
paths:
  - "**/*.swift"
  - "**/Package.swift"
---
# Swift コーディングスタイル

> このファイルは [common/coding-style.md](../common/coding-style.md) を Swift 固有の内容で拡張します。

## フォーマット

- 自動フォーマットには **SwiftFormat**、スタイル適用には **SwiftLint**
- Xcode 16+ には代替として `swift-format` がバンドル

## イミュータビリティ

- `var` よりも `let` を優先 — すべて `let` で定義し、コンパイラが要求する場合のみ `var` に変更
- デフォルトで値セマンティクスの `struct` を使用。同一性または参照セマンティクスが必要な場合のみ `class` を使用

## 命名規則

[Apple API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/) に従う：

- 使用箇所での明確さ — 不要な単語を省略
- メソッドとプロパティは型ではなく役割にちなんで命名
- グローバル定数よりも `static let` を使用

## エラーハンドリング

型付き throws（Swift 6+）とパターンマッチングを使用：

```swift
func load(id: String) throws(LoadError) -> Item {
    guard let data = try? read(from: path) else {
        throw .fileNotFound(id)
    }
    return try decode(data)
}
```

## 並行処理

Swift 6 の厳格な並行性チェックを有効化。以下を優先：

- 分離境界を越えるデータには `Sendable` 値型
- 共有ミュータブル状態にはアクター
- 非構造化 `Task {}` よりも構造化された並行処理（`async let`、`TaskGroup`）
