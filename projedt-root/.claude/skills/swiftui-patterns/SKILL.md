---
name: swiftui-patterns
description: SwiftUIアーキテクチャパターン、@Observableによる状態管理、ビュー合成、ナビゲーション、パフォーマンス最適化、モダンなiOS/macOS UIのベストプラクティス。
---

# SwiftUIパターン

Appleプラットフォーム上で宣言的かつ高性能なUIを構築するためのモダンなSwiftUIパターン。Observationフレームワーク、ビュー合成、型安全なナビゲーション、パフォーマンス最適化をカバー。

## アクティベーション条件

- SwiftUIビューの構築と状態管理（`@State`、`@Observable`、`@Binding`）
- `NavigationStack`によるナビゲーションフローの設計
- ビューモデルとデータフローの構造化
- リストや複雑なレイアウトのレンダリングパフォーマンス最適化
- SwiftUIにおけるEnvironment値と依存性注入の操作

## 状態管理

### プロパティラッパーの選択

適合する最もシンプルなラッパーを選択:

| ラッパー | ユースケース |
|---------|----------|
| `@State` | ビューローカルの値型（トグル、フォームフィールド、シート表示） |
| `@Binding` | 親の`@State`への双方向参照 |
| `@Observable`クラス + `@State` | 複数プロパティを持つ所有モデル |
| `@Observable`クラス（ラッパーなし） | 親から渡される読み取り専用参照 |
| `@Bindable` | `@Observable`プロパティへの双方向バインディング |
| `@Environment` | `.environment()`で注入される共有依存関係 |

### @Observable ViewModel

`@Observable`を使用（`ObservableObject`ではなく）— プロパティレベルの変更を追跡するため、SwiftUIは変更されたプロパティを読み取るビューのみを再レンダリング:

```swift
@Observable
final class ItemListViewModel {
    private(set) var items: [Item] = []
    private(set) var isLoading = false
    var searchText = ""

    private let repository: any ItemRepository

    init(repository: any ItemRepository = DefaultItemRepository()) {
        self.repository = repository
    }

    func load() async {
        isLoading = true
        defer { isLoading = false }
        items = (try? await repository.fetchAll()) ?? []
    }
}
```

### ViewModelを使用するビュー

```swift
struct ItemListView: View {
    @State private var viewModel: ItemListViewModel

    init(viewModel: ItemListViewModel = ItemListViewModel()) {
        _viewModel = State(initialValue: viewModel)
    }

    var body: some View {
        List(viewModel.items) { item in
            ItemRow(item: item)
        }
        .searchable(text: $viewModel.searchText)
        .overlay { if viewModel.isLoading { ProgressView() } }
        .task { await viewModel.load() }
    }
}
```

### Environmentインジェクション

`@EnvironmentObject`を`@Environment`に置き換え:

```swift
// 注入
ContentView()
    .environment(authManager)

// 使用
struct ProfileView: View {
    @Environment(AuthManager.self) private var auth

    var body: some View {
        Text(auth.currentUser?.name ?? "Guest")
    }
}
```

## ビュー合成

### 無効化を制限するためにサブビューを抽出

ビューを小さく焦点を絞った構造体に分割。状態が変更されると、その状態を読み取るサブビューのみが再レンダリング:

```swift
struct OrderView: View {
    @State private var viewModel = OrderViewModel()

    var body: some View {
        VStack {
            OrderHeader(title: viewModel.title)
            OrderItemList(items: viewModel.items)
            OrderTotal(total: viewModel.total)
        }
    }
}
```

### 再利用可能なスタイリングのためのViewModifier

```swift
struct CardModifier: ViewModifier {
    func body(content: Content) -> some View {
        content
            .padding()
            .background(.regularMaterial)
            .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}

extension View {
    func cardStyle() -> some View {
        modifier(CardModifier())
    }
}
```

## ナビゲーション

### 型安全なNavigationStack

プログラム的で型安全なルーティングのために`NavigationStack`と`NavigationPath`を使用:

```swift
@Observable
final class Router {
    var path = NavigationPath()

    func navigate(to destination: Destination) {
        path.append(destination)
    }

    func popToRoot() {
        path = NavigationPath()
    }
}

enum Destination: Hashable {
    case detail(Item.ID)
    case settings
    case profile(User.ID)
}

struct RootView: View {
    @State private var router = Router()

    var body: some View {
        NavigationStack(path: $router.path) {
            HomeView()
                .navigationDestination(for: Destination.self) { dest in
                    switch dest {
                    case .detail(let id): ItemDetailView(itemID: id)
                    case .settings: SettingsView()
                    case .profile(let id): ProfileView(userID: id)
                    }
                }
        }
        .environment(router)
    }
}
```

## パフォーマンス

### 大規模コレクションにはLazyコンテナを使用

`LazyVStack`と`LazyHStack`は表示時にのみビューを作成:

```swift
ScrollView {
    LazyVStack(spacing: 8) {
        ForEach(items) { item in
            ItemRow(item: item)
        }
    }
}
```

### 安定した識別子

`ForEach`では常に安定したユニークなIDを使用 — 配列インデックスの使用を避ける:

```swift
// Identifiable適合または明示的なidを使用
ForEach(items, id: \.stableID) { item in
    ItemRow(item: item)
}
```

### body内での高コスト処理を避ける

- `body`内でI/O、ネットワーク呼び出し、重い計算を決して行わない
- 非同期作業には`.task {}`を使用 — ビューが消えると自動的にキャンセル
- スクロールビュー内での`.sensoryFeedback()`や`.geometryGroup()`の使用は控えめに
- リスト内の`.shadow()`、`.blur()`、`.mask()`を最小限に — オフスクリーンレンダリングを発生させる

### Equatable適合

高コストなbodyを持つビューでは、不要な再レンダリングをスキップするために`Equatable`に適合:

```swift
struct ExpensiveChartView: View, Equatable {
    let dataPoints: [DataPoint] // DataPointはEquatableに適合必要

    static func == (lhs: Self, rhs: Self) -> Bool {
        lhs.dataPoints == rhs.dataPoints
    }

    var body: some View {
        // 複雑なチャートレンダリング
    }
}
```

## プレビュー

高速なイテレーションのためにインラインモックデータと`#Preview`マクロを使用:

```swift
#Preview("Empty state") {
    ItemListView(viewModel: ItemListViewModel(repository: EmptyMockRepository()))
}

#Preview("Loaded") {
    ItemListView(viewModel: ItemListViewModel(repository: PopulatedMockRepository()))
}
```

## 避けるべきアンチパターン

- 新しいコードで`ObservableObject` / `@Published` / `@StateObject` / `@EnvironmentObject`を使用 — `@Observable`に移行
- `body`や`init`に直接非同期処理を入れる — `.task {}`または明示的なロードメソッドを使用
- データを所有しない子ビュー内で`@State`としてビューモデルを作成 — 代わりに親から渡す
- `AnyView`型消去を使用 — 条件付きビューには`@ViewBuilder`や`Group`を推奨
- アクターとの間でデータを渡す際の`Sendable`要件を無視

## 参考

スキル`swift-actor-persistence`でアクターベースの永続化パターンを参照。
スキル`swift-protocol-di-testing`でプロトコルベースのDIとSwift Testingによるテストを参照。
