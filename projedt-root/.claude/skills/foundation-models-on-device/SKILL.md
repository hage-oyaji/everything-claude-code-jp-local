---
name: foundation-models-on-device
description: Apple FoundationModelsフレームワークによるオンデバイスLLM — テキスト生成、@Generableによるガイド生成、ツール呼び出し、iOS 26+でのスナップショットストリーミング。
---

# FoundationModels：オンデバイスLLM（iOS 26）

FoundationModelsフレームワークを使用してAppleのオンデバイス言語モデルをアプリに統合するパターン。テキスト生成、`@Generable`による構造化出力、カスタムツール呼び出し、スナップショットストリーミングをカバーし、すべてプライバシーとオフラインサポートのためにオンデバイスで実行されます。

## アクティベーション条件

- Apple Intelligenceを使用したオンデバイスのAI搭載機能の構築
- クラウド依存なしのテキスト生成または要約
- 自然言語入力からの構造化データ抽出
- ドメイン固有のAIアクション用カスタムツール呼び出しの実装
- リアルタイムUI更新のための構造化レスポンスストリーミング
- プライバシー保護AI（データがデバイスから出ない）が必要な場合

## コアパターン — 可用性チェック

セッション作成前に常にモデルの可用性を確認：

```swift
struct GenerativeView: View {
    private var model = SystemLanguageModel.default

    var body: some View {
        switch model.availability {
        case .available:
            ContentView()
        case .unavailable(.deviceNotEligible):
            Text("Device not eligible for Apple Intelligence")
        case .unavailable(.appleIntelligenceNotEnabled):
            Text("Please enable Apple Intelligence in Settings")
        case .unavailable(.modelNotReady):
            Text("Model is downloading or not ready")
        case .unavailable(let other):
            Text("Model unavailable: \(other)")
        }
    }
}
```

## コアパターン — 基本セッション

```swift
// Single-turn: create a new session each time
let session = LanguageModelSession()
let response = try await session.respond(to: "What's a good month to visit Paris?")
print(response.content)

// Multi-turn: reuse session for conversation context
let session = LanguageModelSession(instructions: """
    You are a cooking assistant.
    Provide recipe suggestions based on ingredients.
    Keep suggestions brief and practical.
    """)

let first = try await session.respond(to: "I have chicken and rice")
let followUp = try await session.respond(to: "What about a vegetarian option?")
```

instructionsのポイント：
- モデルの役割を定義（「あなたはメンターです」）
- 何をすべきか指定（「カレンダーイベントの抽出を手伝ってください」）
- スタイルの好みを設定（「できるだけ簡潔に返答してください」）
- 安全措置を追加（「危険なリクエストには『お手伝いできません』と返答してください」）

## コアパターン — @Generableによるガイド生成

生の文字列の代わりに構造化されたSwift型を生成：

### 1. Generable型の定義

```swift
@Generable(description: "Basic profile information about a cat")
struct CatProfile {
    var name: String

    @Guide(description: "The age of the cat", .range(0...20))
    var age: Int

    @Guide(description: "A one sentence profile about the cat's personality")
    var profile: String
}
```

### 2. 構造化出力のリクエスト

```swift
let response = try await session.respond(
    to: "Generate a cute rescue cat",
    generating: CatProfile.self
)

// Access structured fields directly
print("Name: \(response.content.name)")
print("Age: \(response.content.age)")
print("Profile: \(response.content.profile)")
```

### サポートされる@Guide制約

- `.range(0...20)` — 数値範囲
- `.count(3)` — 配列要素数
- `description:` — 生成のセマンティックガイダンス

## コアパターン — ツール呼び出し

ドメイン固有のタスクにカスタムコードを呼び出す：

### 1. ツールの定義

```swift
struct RecipeSearchTool: Tool {
    let name = "recipe_search"
    let description = "Search for recipes matching a given term and return a list of results."

    @Generable
    struct Arguments {
        var searchTerm: String
        var numberOfResults: Int
    }

    func call(arguments: Arguments) async throws -> ToolOutput {
        let recipes = await searchRecipes(
            term: arguments.searchTerm,
            limit: arguments.numberOfResults
        )
        return .string(recipes.map { "- \($0.name): \($0.description)" }.joined(separator: "\n"))
    }
}
```

### 2. ツール付きセッションの作成

```swift
let session = LanguageModelSession(tools: [RecipeSearchTool()])
let response = try await session.respond(to: "Find me some pasta recipes")
```

### 3. ツールエラーの処理

```swift
do {
    let answer = try await session.respond(to: "Find a recipe for tomato soup.")
} catch let error as LanguageModelSession.ToolCallError {
    print(error.tool.name)
    if case .databaseIsEmpty = error.underlyingError as? RecipeSearchToolError {
        // Handle specific tool error
    }
}
```

## コアパターン — スナップショットストリーミング

`PartiallyGenerated`型を使用してリアルタイムUIのための構造化レスポンスをストリーミング：

```swift
@Generable
struct TripIdeas {
    @Guide(description: "Ideas for upcoming trips")
    var ideas: [String]
}

let stream = session.streamResponse(
    to: "What are some exciting trip ideas?",
    generating: TripIdeas.self
)

for try await partial in stream {
    // partial: TripIdeas.PartiallyGenerated (all properties Optional)
    print(partial)
}
```

### SwiftUI統合

```swift
@State private var partialResult: TripIdeas.PartiallyGenerated?
@State private var errorMessage: String?

var body: some View {
    List {
        ForEach(partialResult?.ideas ?? [], id: \.self) { idea in
            Text(idea)
        }
    }
    .overlay {
        if let errorMessage { Text(errorMessage).foregroundStyle(.red) }
    }
    .task {
        do {
            let stream = session.streamResponse(to: prompt, generating: TripIdeas.self)
            for try await partial in stream {
                partialResult = partial
            }
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}
```

## 主要な設計判断

| 判断 | 根拠 |
|----------|-----------|
| オンデバイス実行 | プライバシー — データがデバイスから出ない；オフラインで動作 |
| 4,096トークン制限 | オンデバイスモデルの制約；大きなデータはセッション間でチャンク化 |
| スナップショットストリーミング（デルタではなく） | 構造化出力にフレンドリー；各スナップショットは完全な部分状態 |
| `@Generable`マクロ | 構造化生成のコンパイル時安全性；`PartiallyGenerated`型を自動生成 |
| セッションあたり単一リクエスト | `isResponding`が同時リクエストを防止；必要に応じて複数セッションを作成 |
| `response.content`（`.output`ではなく） | 正しいAPI — 常に`.content`プロパティで結果にアクセス |

## ベストプラクティス

- **セッション作成前に常に`model.availability`を確認** — すべての非可用性ケースを処理
- **`instructions`を使用**してモデルの動作をガイド — プロンプトより優先される
- **新しいリクエスト送信前に`isResponding`を確認** — セッションは一度に1つのリクエストを処理
- **結果には`response.content`でアクセス** — `.output`ではない
- **大きな入力をチャンクに分割** — 4,096トークン制限はinstructions + prompt + outputの合計に適用
- **構造化出力には`@Generable`を使用** — 生の文字列パースより強い保証
- **`GenerationOptions(temperature:)`で創造性を調整**（高い=より創造的）
- **Instrumentsで監視** — Xcode Instrumentsを使用してリクエストパフォーマンスをプロファイリング

## 避けるべきアンチパターン

- `model.availability`を確認せずにセッションを作成
- 4,096トークンのコンテキストウィンドウを超える入力の送信
- 単一セッションでの同時リクエストの試行
- レスポンスデータへのアクセスに`.content`ではなく`.output`を使用
- `@Generable`構造化出力が使える場面での生の文字列レスポンスのパース
- 単一のプロンプトに複雑なマルチステップロジックを構築 — 複数のフォーカスされたプロンプトに分割
- モデルが常に利用可能であると仮定 — デバイスの適格性と設定は異なる

## 使用すべきケース

- プライバシーに配慮したアプリのためのオンデバイステキスト生成
- ユーザー入力からの構造化データ抽出（フォーム、自然言語コマンド）
- オフラインで動作する必要があるAI支援機能
- 生成コンテンツを段階的に表示するストリーミングUI
- ツール呼び出しによるドメイン固有のAIアクション（検索、計算、ルックアップ）
