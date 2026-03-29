---
name: swift-concurrency-6-2
description: Swift 6.2 Approachable Concurrency — デフォルトでシングルスレッド、明示的なバックグラウンドオフロードに@concurrent、メインアクター型の分離適合。
---

# Swift 6.2 Approachable Concurrency

コードがデフォルトでシングルスレッドで実行され、並行性が明示的に導入されるSwift 6.2の並行性モデルを採用するためのパターン。パフォーマンスを犠牲にすることなく、一般的なデータ競合エラーを排除します。

## アクティベーション条件

- Swift 5.xまたは6.0/6.1プロジェクトをSwift 6.2に移行するとき
- データ競合安全性のコンパイラエラーを解決するとき
- MainActorベースのアプリアーキテクチャを設計するとき
- CPU集約的な作業をバックグラウンドスレッドにオフロードするとき
- MainActor分離型のプロトコル適合を実装するとき
- Xcode 26でApproachable Concurrencyビルド設定を有効化するとき

## コア問題: 暗黙的なバックグラウンドオフロード

Swift 6.1以前では、非同期関数が暗黙的にバックグラウンドスレッドにオフロードされ、一見安全に見えるコードでもデータ競合エラーが発生する可能性がありました:

```swift
// Swift 6.1: ERROR
@MainActor
final class StickerModel {
    let photoProcessor = PhotoProcessor()

    func extractSticker(_ item: PhotosPickerItem) async throws -> Sticker? {
        guard let data = try await item.loadTransferable(type: Data.self) else { return nil }

        // Error: Sending 'self.photoProcessor' risks causing data races
        return await photoProcessor.extractSticker(data: data, with: item.itemIdentifier)
    }
}
```

Swift 6.2はこれを修正: 非同期関数はデフォルトで呼び出し元のアクターに留まります。

```swift
// Swift 6.2: OK — asyncはMainActorに留まり、データ競合なし
@MainActor
final class StickerModel {
    let photoProcessor = PhotoProcessor()

    func extractSticker(_ item: PhotosPickerItem) async throws -> Sticker? {
        guard let data = try await item.loadTransferable(type: Data.self) else { return nil }
        return await photoProcessor.extractSticker(data: data, with: item.itemIdentifier)
    }
}
```

## コアパターン — 分離適合

MainActor型が非分離プロトコルに安全に適合できるようになりました:

```swift
protocol Exportable {
    func export()
}

// Swift 6.1: ERROR — メインアクター分離コードへの越境
// Swift 6.2: OK（分離適合）
extension StickerModel: @MainActor Exportable {
    func export() {
        photoProcessor.exportAsPNG()
    }
}
```

コンパイラは適合がメインアクター上でのみ使用されることを保証します:

```swift
// OK — ImageExporterも@MainActor
@MainActor
struct ImageExporter {
    var items: [any Exportable]

    mutating func add(_ item: StickerModel) {
        items.append(item)  // Safe: same actor isolation
    }
}

// ERROR — nonisolatedコンテキストはMainActor適合を使用できない
nonisolated struct ImageExporter {
    var items: [any Exportable]

    mutating func add(_ item: StickerModel) {
        items.append(item)  // Error: Main actor-isolated conformance cannot be used here
    }
}
```

## コアパターン — グローバル変数とスタティック変数

グローバル/スタティック状態をMainActorで保護:

```swift
// Swift 6.1: ERROR — non-Sendable型が共有可変状態を持つ可能性
final class StickerLibrary {
    static let shared: StickerLibrary = .init()  // Error
}

// 修正: @MainActorでアノテーション
@MainActor
final class StickerLibrary {
    static let shared: StickerLibrary = .init()  // OK
}
```

### MainActorデフォルト推論モード

Swift 6.2ではMainActorがデフォルトで推論されるモードが導入されました — 手動アノテーション不要:

```swift
// MainActorデフォルト推論が有効な場合:
final class StickerLibrary {
    static let shared: StickerLibrary = .init()  // 暗黙的に@MainActor
}

final class StickerModel {
    let photoProcessor: PhotoProcessor
    var selection: [PhotosPickerItem]  // 暗黙的に@MainActor
}

extension StickerModel: Exportable {  // 暗黙的に@MainActor適合
    func export() {
        photoProcessor.exportAsPNG()
    }
}
```

このモードはオプトインで、アプリ、スクリプト、その他の実行可能ターゲットに推奨されます。

## コアパターン — バックグラウンド作業のための@concurrent

実際の並列処理が必要な場合、`@concurrent`で明示的にオフロード:

> **重要:** この例にはApproachable Concurrencyビルド設定 — SE-0466（MainActorデフォルト分離）とSE-0461（NonisolatedNonsendingByDefault）が必要です。これらが有効な場合、`extractSticker`は呼び出し元のアクターに留まり、可変状態アクセスが安全になります。**これらの設定なしでは、このコードにはデータ競合があります** — コンパイラがフラグを立てます。

```swift
nonisolated final class PhotoProcessor {
    private var cachedStickers: [String: Sticker] = [:]

    func extractSticker(data: Data, with id: String) async -> Sticker {
        if let sticker = cachedStickers[id] {
            return sticker
        }

        let sticker = await Self.extractSubject(from: data)
        cachedStickers[id] = sticker
        return sticker
    }

    // 高コスト処理をコンカレントスレッドプールにオフロード
    @concurrent
    static func extractSubject(from data: Data) async -> Sticker { /* ... */ }
}

// 呼び出し元はawaitが必要
let processor = PhotoProcessor()
processedPhotos[item.id] = await processor.extractSticker(data: data, with: item.id)
```

`@concurrent`を使用するには:
1. 含まれる型を`nonisolated`としてマーク
2. 関数に`@concurrent`を追加
3. まだ非同期でなければ`async`を追加
4. 呼び出し元で`await`を追加

## 主要な設計判断

| 判断 | 理由 |
|----------|-----------|
| デフォルトでシングルスレッド | 最も自然なコードがデータ競合フリー; 並行性はオプトイン |
| 非同期は呼び出し元のアクターに留まる | データ競合エラーを引き起こす暗黙的なオフロードを排除 |
| 分離適合 | MainActor型が安全でないワークアラウンドなしでプロトコルに適合可能 |
| `@concurrent`の明示的オプトイン | バックグラウンド実行は偶発的ではなく意図的なパフォーマンス選択 |
| MainActorデフォルト推論 | アプリターゲットのボイラープレート`@MainActor`アノテーションを削減 |
| オプトイン採用 | 非破壊的な移行パス — 機能を段階的に有効化 |

## 移行ステップ

1. **Xcodeで有効化**: Build Settings > Swift Compiler > Concurrencyセクション
2. **SPMで有効化**: パッケージマニフェストの`SwiftSettings` APIを使用
3. **移行ツールの使用**: swift.org/migrationの自動コード変更
4. **MainActorデフォルトから開始**: アプリターゲットで推論モードを有効化
5. **必要な箇所に`@concurrent`を追加**: まずプロファイリングし、ホットパスをオフロード
6. **徹底的にテスト**: データ競合の問題がコンパイル時エラーになる

## ベストプラクティス

- **MainActorから始める** — まずシングルスレッドのコードを書き、後で最適化
- **`@concurrent`はCPU集約的な作業にのみ使用** — 画像処理、圧縮、複雑な計算
- **MainActor推論モードを有効化** — 主にシングルスレッドのアプリターゲットに
- **オフロード前にプロファイリング** — Instrumentsを使用して実際のボトルネックを特定
- **グローバルをMainActorで保護** — グローバル/スタティックの可変状態にはアクター分離が必要
- **分離適合を使用** — `nonisolated`ワークアラウンドや`@Sendable`ラッパーの代わりに
- **段階的に移行** — ビルド設定で機能を1つずつ有効化

## 避けるべきアンチパターン

- すべてのasync関数に`@concurrent`を適用する（ほとんどはバックグラウンド実行を必要としない）
- 分離を理解せずにコンパイラエラーを抑制するために`nonisolated`を使用
- アクターが同じ安全性を提供するのにレガシーの`DispatchQueue`パターンを維持
- 並行性関連のFoundation Modelsコードで`model.availability`チェックをスキップ
- コンパイラと戦う — データ競合を報告する場合、コードに実際の並行性の問題がある
- すべてのasyncコードがバックグラウンドで実行されると想定する（Swift 6.2のデフォルト: 呼び出し元のアクターに留まる）

## 使用タイミング

- すべての新しいSwift 6.2以降のプロジェクト（Approachable Concurrencyが推奨デフォルト）
- Swift 5.xまたは6.0/6.1の並行性からの既存アプリの移行
- Xcode 26採用時のデータ競合安全性コンパイラエラーの解決
- MainActor中心のアプリアーキテクチャの構築（ほとんどのUIアプリ）
- パフォーマンス最適化 — 特定の重い計算をバックグラウンドにオフロード
