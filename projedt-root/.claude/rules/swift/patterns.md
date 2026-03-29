---
paths:
  - "**/*.swift"
  - "**/Package.swift"
---
# Swift パターン

> このファイルは [common/patterns.md](../common/patterns.md) を Swift 固有の内容で拡張します。

## プロトコル指向設計

小さく焦点を絞ったプロトコルを定義。共有デフォルトにはプロトコル拡張を使用：

```swift
protocol Repository: Sendable {
    associatedtype Item: Identifiable & Sendable
    func find(by id: Item.ID) async throws -> Item?
    func save(_ item: Item) async throws
}
```

## 値型

- データ転送オブジェクトとモデルには構造体を使用
- 個別の状態をモデル化するには関連値付き列挙型を使用：

```swift
enum LoadState<T: Sendable>: Sendable {
    case idle
    case loading
    case loaded(T)
    case failed(Error)
}
```

## アクターパターン

ロックやディスパッチキューの代わりに、共有ミュータブル状態にはアクターを使用：

```swift
actor Cache<Key: Hashable & Sendable, Value: Sendable> {
    private var storage: [Key: Value] = [:]

    func get(_ key: Key) -> Value? { storage[key] }
    func set(_ key: Key, value: Value) { storage[key] = value }
}
```

## 依存性注入

デフォルトパラメータ付きのプロトコルを注入 — プロダクションはデフォルトを使用、テストはモックを注入：

```swift
struct UserService {
    private let repository: any UserRepository

    init(repository: any UserRepository = DefaultUserRepository()) {
        self.repository = repository
    }
}
```

## 参考

スキル `swift-actor-persistence` にアクターベースの永続化パターンがあります。
スキル `swift-protocol-di-testing` にプロトコルベースの DI とテストがあります。
