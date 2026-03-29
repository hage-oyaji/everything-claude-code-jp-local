---
name: swift-actor-persistence
description: Swiftのアクターを使用したスレッドセーフなデータ永続化 — インメモリキャッシュとファイルバックドストレージにより、設計段階でデータ競合を排除。
origin: ECC
---

# Swiftアクターによるスレッドセーフな永続化

Swiftアクターを使用してスレッドセーフなデータ永続化レイヤーを構築するパターン。インメモリキャッシングとファイルバックドストレージを組み合わせ、アクターモデルを活用してコンパイル時にデータ競合を排除します。

## 発動タイミング

- Swift 5.5以降でデータ永続化レイヤーを構築するとき
- 共有可変状態へのスレッドセーフなアクセスが必要なとき
- 手動の同期処理（ロック、DispatchQueues）を排除したいとき
- ローカルストレージを持つオフラインファーストアプリを構築するとき

## コアパターン

### アクターベースのリポジトリ

アクターモデルはシリアライズされたアクセスを保証します — データ競合なし、コンパイラによる強制。

```swift
public actor LocalRepository<T: Codable & Identifiable> where T.ID == String {
    private var cache: [String: T] = [:]
    private let fileURL: URL

    public init(directory: URL = .documentsDirectory, filename: String = "data.json") {
        self.fileURL = directory.appendingPathComponent(filename)
        // Synchronous load during init (actor isolation not yet active)
        self.cache = Self.loadSynchronously(from: fileURL)
    }

    // MARK: - Public API

    public func save(_ item: T) throws {
        cache[item.id] = item
        try persistToFile()
    }

    public func delete(_ id: String) throws {
        cache[id] = nil
        try persistToFile()
    }

    public func find(by id: String) -> T? {
        cache[id]
    }

    public func loadAll() -> [T] {
        Array(cache.values)
    }

    // MARK: - Private

    private func persistToFile() throws {
        let data = try JSONEncoder().encode(Array(cache.values))
        try data.write(to: fileURL, options: .atomic)
    }

    private static func loadSynchronously(from url: URL) -> [String: T] {
        guard let data = try? Data(contentsOf: url),
              let items = try? JSONDecoder().decode([T].self, from: data) else {
            return [:]
        }
        return Dictionary(uniqueKeysWithValues: items.map { ($0.id, $0) })
    }
}
```

### 使用方法

アクターの分離により、すべての呼び出しは自動的にasyncになります：

```swift
let repository = LocalRepository<Question>()

// Read — fast O(1) lookup from in-memory cache
let question = await repository.find(by: "q-001")
let allQuestions = await repository.loadAll()

// Write — updates cache and persists to file atomically
try await repository.save(newQuestion)
try await repository.delete("q-001")
```

### @Observable ViewModelとの組み合わせ

```swift
@Observable
final class QuestionListViewModel {
    private(set) var questions: [Question] = []
    private let repository: LocalRepository<Question>

    init(repository: LocalRepository<Question> = LocalRepository()) {
        self.repository = repository
    }

    func load() async {
        questions = await repository.loadAll()
    }

    func add(_ question: Question) async throws {
        try await repository.save(question)
        questions = await repository.loadAll()
    }
}
```

## 主要な設計判断

| 判断 | 根拠 |
|----------|-----------|
| アクター（クラス + ロックではなく） | コンパイラ強制のスレッドセーフティ、手動同期不要 |
| インメモリキャッシュ + ファイル永続化 | キャッシュからの高速読み取り、ディスクへの永続書き込み |
| 同期的なinitロード | 非同期初期化の複雑さを回避 |
| IDをキーとするDictionary | 識別子によるO(1)ルックアップ |
| `Codable & Identifiable`のジェネリック | 任意のモデル型で再利用可能 |
| アトミックファイル書き込み（`.atomic`） | クラッシュ時の部分書き込みを防止 |

## ベストプラクティス

- アクター境界を越えるすべてのデータに**`Sendable`型を使用**
- **アクターのパブリックAPIを最小限に保つ** — 永続化の詳細ではなくドメイン操作のみを公開
- **`.atomic`書き込みを使用** — アプリが書き込み中にクラッシュした場合のデータ破損を防止
- **`init`で同期的にロード** — 非同期イニシャライザはローカルファイルにとって最小限のメリットで複雑さを増す
- **`@Observable`** ViewModelと組み合わせてリアクティブなUI更新

## 避けるべきアンチパターン

- 新しいSwift Concurrencyコードで`DispatchQueue`や`NSLock`をアクターの代わりに使用
- 内部キャッシュディクショナリを外部の呼び出し元に公開
- バリデーションなしでファイルURLを設定可能にする
- すべてのアクターメソッド呼び出しが`await`であることを忘れる — 呼び出し元は非同期コンテキストを処理する必要がある
- アクターの分離を回避するために`nonisolated`を使用（目的を無効にする）

## 使用タイミング

- iOS/macOSアプリのローカルデータストレージ（ユーザーデータ、設定、キャッシュコンテンツ）
- 後でサーバーに同期するオフラインファーストアーキテクチャ
- アプリの複数の部分が同時にアクセスする共有可変状態
- レガシーの`DispatchQueue`ベースのスレッドセーフティをモダンなSwift Concurrencyに置き換え
