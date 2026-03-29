---
name: swift-protocol-di-testing
description: テスト可能なSwiftコードのためのプロトコルベースの依存性注入 — 焦点を絞ったプロトコルとSwift Testingを使用してファイルシステム、ネットワーク、外部APIをモック化。
origin: ECC
---

# Swiftプロトコルベースの依存性注入によるテスト

外部依存関係（ファイルシステム、ネットワーク、iCloud）を小さく焦点を絞ったプロトコルの背後に抽象化することで、Swiftコードをテスタブルにするパターン。I/Oなしで決定論的なテストを可能にします。

## アクティベーション条件

- ファイルシステム、ネットワーク、または外部APIにアクセスするSwiftコードを書くとき
- 実際の障害を発生させずにエラーハンドリングパスをテストする必要があるとき
- 環境（アプリ、テスト、SwiftUIプレビュー）間で動作するモジュールを構築するとき
- Swift並行性（アクター、Sendable）を使用したテスタブルなアーキテクチャを設計するとき

## コアパターン

### 1. 小さく焦点を絞ったプロトコルを定義

各プロトコルは正確に1つの外部関心事を処理します。

```swift
// ファイルシステムアクセス
public protocol FileSystemProviding: Sendable {
    func containerURL(for purpose: Purpose) -> URL?
}

// ファイル読み書き操作
public protocol FileAccessorProviding: Sendable {
    func read(from url: URL) throws -> Data
    func write(_ data: Data, to url: URL) throws
    func fileExists(at url: URL) -> Bool
}

// ブックマークストレージ（例: サンドボックスアプリ用）
public protocol BookmarkStorageProviding: Sendable {
    func saveBookmark(_ data: Data, for key: String) throws
    func loadBookmark(for key: String) throws -> Data?
}
```

### 2. デフォルト（プロダクション）実装を作成

```swift
public struct DefaultFileSystemProvider: FileSystemProviding {
    public init() {}

    public func containerURL(for purpose: Purpose) -> URL? {
        FileManager.default.url(forUbiquityContainerIdentifier: nil)
    }
}

public struct DefaultFileAccessor: FileAccessorProviding {
    public init() {}

    public func read(from url: URL) throws -> Data {
        try Data(contentsOf: url)
    }

    public func write(_ data: Data, to url: URL) throws {
        try data.write(to: url, options: .atomic)
    }

    public func fileExists(at url: URL) -> Bool {
        FileManager.default.fileExists(atPath: url.path)
    }
}
```

### 3. テスト用モック実装を作成

```swift
public final class MockFileAccessor: FileAccessorProviding, @unchecked Sendable {
    public var files: [URL: Data] = [:]
    public var readError: Error?
    public var writeError: Error?

    public init() {}

    public func read(from url: URL) throws -> Data {
        if let error = readError { throw error }
        guard let data = files[url] else {
            throw CocoaError(.fileReadNoSuchFile)
        }
        return data
    }

    public func write(_ data: Data, to url: URL) throws {
        if let error = writeError { throw error }
        files[url] = data
    }

    public func fileExists(at url: URL) -> Bool {
        files[url] != nil
    }
}
```

### 4. デフォルトパラメータで依存性を注入

プロダクションコードはデフォルトを使用; テストのみモックを指定。

```swift
public actor SyncManager {
    private let fileSystem: FileSystemProviding
    private let fileAccessor: FileAccessorProviding

    public init(
        fileSystem: FileSystemProviding = DefaultFileSystemProvider(),
        fileAccessor: FileAccessorProviding = DefaultFileAccessor()
    ) {
        self.fileSystem = fileSystem
        self.fileAccessor = fileAccessor
    }

    public func sync() async throws {
        guard let containerURL = fileSystem.containerURL(for: .sync) else {
            throw SyncError.containerNotAvailable
        }
        let data = try fileAccessor.read(
            from: containerURL.appendingPathComponent("data.json")
        )
        // Process data...
    }
}
```

### 5. Swift Testingでテストを記述

```swift
import Testing

@Test("Sync manager handles missing container")
func testMissingContainer() async {
    let mockFileSystem = MockFileSystemProvider(containerURL: nil)
    let manager = SyncManager(fileSystem: mockFileSystem)

    await #expect(throws: SyncError.containerNotAvailable) {
        try await manager.sync()
    }
}

@Test("Sync manager reads data correctly")
func testReadData() async throws {
    let mockFileAccessor = MockFileAccessor()
    mockFileAccessor.files[testURL] = testData

    let manager = SyncManager(fileAccessor: mockFileAccessor)
    let result = try await manager.loadData()

    #expect(result == expectedData)
}

@Test("Sync manager handles read errors gracefully")
func testReadError() async {
    let mockFileAccessor = MockFileAccessor()
    mockFileAccessor.readError = CocoaError(.fileReadCorruptFile)

    let manager = SyncManager(fileAccessor: mockFileAccessor)

    await #expect(throws: SyncError.self) {
        try await manager.sync()
    }
}
```

## ベストプラクティス

- **単一責任**: 各プロトコルは1つの関心事を処理 — 多くのメソッドを持つ「神プロトコル」を作らない
- **Sendable適合**: アクター境界を越えてプロトコルを使用する場合に必要
- **デフォルトパラメータ**: プロダクションコードはデフォルトで実際の実装を使用; テストのみモックを指定する必要がある
- **エラーシミュレーション**: 障害パスをテストするための設定可能なエラープロパティを持つモックを設計
- **境界のみモック**: 外部依存関係（ファイルシステム、ネットワーク、API）のみモック化し、内部型はモックしない

## 避けるべきアンチパターン

- すべての外部アクセスをカバーする単一の大きなプロトコルを作成
- 外部依存関係のない内部型をモック化
- 適切な依存性注入の代わりに`#if DEBUG`条件分岐を使用
- アクターと共に使用する場合に`Sendable`適合を忘れる
- 過剰エンジニアリング: 型に外部依存関係がない場合、プロトコルは不要

## 使用タイミング

- ファイルシステム、ネットワーク、または外部APIに触れるSwiftコード全般
- 実際の環境では発生させにくいエラーハンドリングパスのテスト
- アプリ、テスト、SwiftUIプレビューコンテキストで動作する必要のあるモジュールの構築
- テスタブルなアーキテクチャが必要なSwift並行性（アクター、構造化並行性）を使用するアプリ
