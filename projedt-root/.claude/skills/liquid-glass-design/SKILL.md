---
name: liquid-glass-design
description: iOS 26 Liquid Glassデザインシステム — ブラー、反射、インタラクティブモーフィングを備えた動的ガラスマテリアル（SwiftUI、UIKit、WidgetKit対応）。
---

# Liquid Glassデザインシステム（iOS 26）

AppleのLiquid Glassを実装するためのパターン — 背後のコンテンツをぼかし、周囲のコンテンツから色や光を反射し、タッチやポインターインタラクションに反応する動的マテリアル。SwiftUI、UIKit、WidgetKitの統合をカバーする。

## 有効化するタイミング

- iOS 26+の新しいデザイン言語でアプリを構築・更新する場合
- ガラススタイルのボタン、カード、ツールバー、コンテナを実装する場合
- ガラスエレメント間のモーフィングトランジションを作成する場合
- ウィジェットにLiquid Glassエフェクトを適用する場合
- 既存のブラー/マテリアルエフェクトを新しいLiquid Glass APIに移行する場合

## コアパターン — SwiftUI

### 基本的なグラスエフェクト

任意のビューにLiquid Glassを追加する最もシンプルな方法:

```swift
Text("Hello, World!")
    .font(.title)
    .padding()
    .glassEffect()  // Default: regular variant, capsule shape
```

### シェイプとティントのカスタマイズ

```swift
Text("Hello, World!")
    .font(.title)
    .padding()
    .glassEffect(.regular.tint(.orange).interactive(), in: .rect(cornerRadius: 16.0))
```

主なカスタマイズオプション:
- `.regular` — 標準グラスエフェクト
- `.tint(Color)` — 強調のためにカラーティントを追加
- `.interactive()` — タッチとポインターインタラクションに反応
- シェイプ: `.capsule`（デフォルト）、`.rect(cornerRadius:)`、`.circle`

### グラスボタンスタイル

```swift
Button("Click Me") { /* action */ }
    .buttonStyle(.glass)

Button("Important") { /* action */ }
    .buttonStyle(.glassProminent)
```

### 複数エレメントのためのGlassEffectContainer

パフォーマンスとモーフィングのために、複数のグラスビューは常にコンテナでラップする:

```swift
GlassEffectContainer(spacing: 40.0) {
    HStack(spacing: 40.0) {
        Image(systemName: "scribble.variable")
            .frame(width: 80.0, height: 80.0)
            .font(.system(size: 36))
            .glassEffect()

        Image(systemName: "eraser.fill")
            .frame(width: 80.0, height: 80.0)
            .font(.system(size: 36))
            .glassEffect()
    }
}
```

`spacing` パラメータはマージ距離を制御する — 近いエレメントはグラスシェイプが融合する。

### グラスエフェクトの統合

`glassEffectUnion` で複数のビューを単一のグラスシェイプに結合する:

```swift
@Namespace private var namespace

GlassEffectContainer(spacing: 20.0) {
    HStack(spacing: 20.0) {
        ForEach(symbolSet.indices, id: \.self) { item in
            Image(systemName: symbolSet[item])
                .frame(width: 80.0, height: 80.0)
                .glassEffect()
                .glassEffectUnion(id: item < 2 ? "group1" : "group2", namespace: namespace)
        }
    }
}
```

### モーフィングトランジション

グラスエレメントの表示/非表示時にスムーズなモーフィングを作成する:

```swift
@State private var isExpanded = false
@Namespace private var namespace

GlassEffectContainer(spacing: 40.0) {
    HStack(spacing: 40.0) {
        Image(systemName: "scribble.variable")
            .frame(width: 80.0, height: 80.0)
            .glassEffect()
            .glassEffectID("pencil", in: namespace)

        if isExpanded {
            Image(systemName: "eraser.fill")
                .frame(width: 80.0, height: 80.0)
                .glassEffect()
                .glassEffectID("eraser", in: namespace)
        }
    }
}

Button("Toggle") {
    withAnimation { isExpanded.toggle() }
}
.buttonStyle(.glass)
```

### サイドバー下への水平スクロールの拡張

水平スクロールコンテンツがサイドバーやインスペクターの下に拡張されるようにするには、`ScrollView` コンテンツがコンテナの前端/後端に到達するようにする。レイアウトが端まで拡張されると、システムが自動的にサイドバー下のスクロール動作を処理する — 追加のモディファイアは不要。

## コアパターン — UIKit

### 基本的なUIGlassEffect

```swift
let glassEffect = UIGlassEffect()
glassEffect.tintColor = UIColor.systemBlue.withAlphaComponent(0.3)
glassEffect.isInteractive = true

let visualEffectView = UIVisualEffectView(effect: glassEffect)
visualEffectView.translatesAutoresizingMaskIntoConstraints = false
visualEffectView.layer.cornerRadius = 20
visualEffectView.clipsToBounds = true

view.addSubview(visualEffectView)
NSLayoutConstraint.activate([
    visualEffectView.centerXAnchor.constraint(equalTo: view.centerXAnchor),
    visualEffectView.centerYAnchor.constraint(equalTo: view.centerYAnchor),
    visualEffectView.widthAnchor.constraint(equalToConstant: 200),
    visualEffectView.heightAnchor.constraint(equalToConstant: 120)
])

// Add content to contentView
let label = UILabel()
label.text = "Liquid Glass"
label.translatesAutoresizingMaskIntoConstraints = false
visualEffectView.contentView.addSubview(label)
NSLayoutConstraint.activate([
    label.centerXAnchor.constraint(equalTo: visualEffectView.contentView.centerXAnchor),
    label.centerYAnchor.constraint(equalTo: visualEffectView.contentView.centerYAnchor)
])
```

### 複数エレメントのためのUIGlassContainerEffect

```swift
let containerEffect = UIGlassContainerEffect()
containerEffect.spacing = 40.0

let containerView = UIVisualEffectView(effect: containerEffect)

let firstGlass = UIVisualEffectView(effect: UIGlassEffect())
let secondGlass = UIVisualEffectView(effect: UIGlassEffect())

containerView.contentView.addSubview(firstGlass)
containerView.contentView.addSubview(secondGlass)
```

### スクロールエッジエフェクト

```swift
scrollView.topEdgeEffect.style = .automatic
scrollView.bottomEdgeEffect.style = .hard
scrollView.leftEdgeEffect.isHidden = true
```

### ツールバーのグラス統合

```swift
let favoriteButton = UIBarButtonItem(image: UIImage(systemName: "heart"), style: .plain, target: self, action: #selector(favoriteAction))
favoriteButton.hidesSharedBackground = true  // Opt out of shared glass background
```

## コアパターン — WidgetKit

### レンダリングモードの検出

```swift
struct MyWidgetView: View {
    @Environment(\.widgetRenderingMode) var renderingMode

    var body: some View {
        if renderingMode == .accented {
            // Tinted mode: white-tinted, themed glass background
        } else {
            // Full color mode: standard appearance
        }
    }
}
```

### ビジュアル階層のためのアクセントグループ

```swift
HStack {
    VStack(alignment: .leading) {
        Text("Title")
            .widgetAccentable()  // Accent group
        Text("Subtitle")
            // Primary group (default)
    }
    Image(systemName: "star.fill")
        .widgetAccentable()  // Accent group
}
```

### アクセントモードでの画像レンダリング

```swift
Image("myImage")
    .widgetAccentedRenderingMode(.monochrome)
```

### コンテナ背景

```swift
VStack { /* content */ }
    .containerBackground(for: .widget) {
        Color.blue.opacity(0.2)
    }
```

## 主要な設計判断

| 判断 | 根拠 |
|------|------|
| GlassEffectContainerでのラッピング | パフォーマンス最適化、グラスエレメント間のモーフィングを有効化 |
| `spacing` パラメータ | マージ距離を制御 — エレメントが融合するために必要な近さを微調整 |
| `@Namespace` + `glassEffectID` | ビュー階層変更時のスムーズなモーフィングトランジションを有効化 |
| `interactive()` モディファイア | タッチ/ポインター反応への明示的なオプトイン — すべてのグラスが反応すべきではない |
| UIKitでのUIGlassContainerEffect | 一貫性のためのSwiftUIと同じコンテナパターン |
| ウィジェットでのアクセントレンダリングモード | ユーザーがティントされたホーム画面を選択した際にシステムがティントグラスを適用 |

## ベストプラクティス

- 複数の兄弟ビューにグラスを適用する場合は**常にGlassEffectContainerを使用する** — モーフィングが有効になり、レンダリングパフォーマンスが向上する
- 他の外観モディファイア（frame、font、padding）の**後に** `.glassEffect()` を適用する
- ユーザーインタラクションに応答するエレメント（ボタン、トグル可能な項目）にのみ **`.interactive()`** を使用する
- コンテナ内の **spacingを慎重に選択する** ことで、グラスエフェクトがいつマージするかを制御する
- スムーズなモーフィングトランジションを有効にするために、ビュー階層を変更する際に **`withAnimation`** を使用する
- **外観をまたいでテストする** — ライトモード、ダークモード、アクセント/ティントモード
- **アクセシビリティコントラストを確保する** — グラス上のテキストは読みやすくなければならない

## 避けるべきアンチパターン

- GlassEffectContainerなしで複数のスタンドアロン `.glassEffect()` ビューを使用する
- 過度にグラスエフェクトをネストする — パフォーマンスと視覚的明瞭さが低下する
- すべてのビューにグラスを適用する — インタラクティブエレメント、ツールバー、カードに限定する
- UIKitで角丸を使用する際に `clipsToBounds = true` を忘れる
- ウィジェットでアクセントレンダリングモードを無視する — ティントされたホーム画面の外観が壊れる
- グラスの背後に不透明な背景を使用する — 半透明エフェクトが無効になる

## 使用するタイミング

- iOS 26の新デザインによるナビゲーションバー、ツールバー、タブバー
- フローティングアクションボタンとカードスタイルのコンテナ
- 視覚的な深さとタッチフィードバックが必要なインタラクティブコントロール
- システムのLiquid Glass外観と統合すべきウィジェット
- 関連するUI状態間のモーフィングトランジション
