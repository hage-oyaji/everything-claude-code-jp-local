---
name: flutter-dart-code-review
description: ライブラリ非依存のFlutter/Dartコードレビューチェックリスト。ウィジェットのベストプラクティス、状態管理パターン（BLoC、Riverpod、Provider、GetX、MobX、Signals）、Dartイディオム、パフォーマンス、アクセシビリティ、セキュリティ、クリーンアーキテクチャをカバー。
origin: ECC
---

# Flutter/Dartコードレビューベストプラクティス

Flutter/Dartアプリケーションをレビューするための包括的でライブラリ非依存のチェックリストです。使用する状態管理ソリューション、ルーティングライブラリ、DIフレームワークに関わらず適用できます。

---

## 1. プロジェクト全般の健全性

- [ ] プロジェクトが一貫したフォルダ構造に従っている（フィーチャーファースト or レイヤーファースト）
- [ ] 適切な関心の分離：UI、ビジネスロジック、データレイヤー
- [ ] ウィジェットにビジネスロジックがない；ウィジェットは純粋にプレゼンテーション用
- [ ] `pubspec.yaml`がクリーン — 未使用の依存関係がなく、バージョンが適切にピン留め
- [ ] `analysis_options.yaml`に厳格なリントセットと厳格なアナライザー設定が含まれている
- [ ] プロダクションコードに`print()`文がない — `dart:developer`の`log()`またはロギングパッケージを使用
- [ ] 生成ファイル（`.g.dart`、`.freezed.dart`、`.gr.dart`）が最新、または`.gitignore`に含まれている
- [ ] プラットフォーム固有のコードが抽象化の背後に分離されている

---

## 2. Dart言語の落とし穴

- [ ] **暗黙的dynamic**：型アノテーション不足による`dynamic` — `strict-casts`、`strict-inference`、`strict-raw-types`を有効化
- [ ] **Null安全の誤用**：適切なnullチェックやDart 3パターンマッチング（`if (value case var v?)`）ではなく過度な`!`（バングオペレーター）
- [ ] **型プロモーション失敗**：ローカル変数のプロモーションが機能する箇所で`this.field`を使用
- [ ] **キャッチ範囲が広すぎる**：`on`句なしの`catch (e)` — 常に例外型を指定
- [ ] **`Error`のキャッチ**：`Error`サブタイプはバグを示すものでありキャッチすべきでない
- [ ] **不要な`async`**：`await`しないのに`async`マークされた関数 — 不要なオーバーヘッド
- [ ] **`late`の過剰使用**：nullableまたはコンストラクタ初期化の方が安全な箇所で`late`を使用；エラーをランタイムに遅延
- [ ] **ループ内の文字列連結**：反復的な文字列構築には`+`ではなく`StringBuffer`を使用
- [ ] **`const`コンテキストでのミュータブル状態**：`const`コンストラクタクラスのフィールドはミュータブルであるべきでない
- [ ] **`Future`戻り値の無視**：`await`を使用するか、意図を示すために明示的に`unawaited()`を呼び出す
- [ ] **`final`が使える箇所での`var`**：ローカルには`final`、コンパイル時定数には`const`を優先
- [ ] **相対インポート**：一貫性のために`package:`インポートを使用
- [ ] **公開されたミュータブルコレクション**：パブリックAPIは生の`List`/`Map`ではなく変更不可ビューを返すべき
- [ ] **Dart 3パターンマッチングの未使用**：冗長な`is`チェックと手動キャストよりswitch式と`if-case`を優先
- [ ] **複数戻り値のための使い捨てクラス**：単一用途DTOの代わりにDart 3レコード`(String, int)`を使用
- [ ] **プロダクションコードでの`print()`**：`dart:developer`の`log()`またはプロジェクトのロギングパッケージを使用；`print()`にはログレベルがなくフィルタリングできない

---

## 3. ウィジェットのベストプラクティス

### ウィジェット分割：
- [ ] `build()`メソッドが約80-100行を超える単一ウィジェットがない
- [ ] ウィジェットがカプセル化と変更方法（リビルド境界）で分割されている
- [ ] ウィジェットを返すプライベート`_build*()`ヘルパーメソッドが別のウィジェットクラスに抽出されている（エレメント再利用、const伝播、フレームワーク最適化を有効化）
- [ ] ミュータブルなローカル状態が不要な場合はStatelessウィジェットを優先
- [ ] 再利用可能な抽出ウィジェットは別ファイルに配置

### constの使用：
- [ ] 可能な限り`const`コンストラクタを使用 — 不要なリビルドを防止
- [ ] 変更されないコレクションにはconstリテラル（`const []`、`const {}`）
- [ ] すべてのフィールドがfinalの場合、コンストラクタを`const`宣言

### Keyの使用：
- [ ] リスト/グリッドで並べ替え時の状態保持に`ValueKey`を使用
- [ ] `GlobalKey`は控えめに使用 — ツリー間での状態アクセスが本当に必要な場合のみ
- [ ] `build()`内での`UniqueKey`は避ける — 毎フレームリビルドを強制
- [ ] 単一の値ではなくデータオブジェクトに基づくアイデンティティには`ObjectKey`を使用

### テーマ＆デザインシステム：
- [ ] 色は`Theme.of(context).colorScheme`から — ハードコードの`Colors.red`やhex値は不可
- [ ] テキストスタイルは`Theme.of(context).textTheme`から — 生のフォントサイズを持つインライン`TextStyle`は不可
- [ ] ダークモード互換性の確認 — 明るい背景の前提を置かない
- [ ] スペーシングとサイジングに一貫したデザイントークンまたは定数を使用、マジックナンバーは不可

### buildメソッドの複雑さ：
- [ ] `build()`内にネットワーク呼び出し、ファイルI/O、重い計算がない
- [ ] `build()`内に`Future.then()`や`async`作業がない
- [ ] `build()`内にサブスクリプション作成（`.listen()`）がない
- [ ] `setState()`は可能な限り小さなサブツリーにローカライズ

---

## 4. 状態管理（ライブラリ非依存）

これらの原則はすべてのFlutter状態管理ソリューション（BLoC、Riverpod、Provider、GetX、MobX、Signals、ValueNotifierなど）に適用されます。

### アーキテクチャ：
- [ ] ビジネスロジックがウィジェットレイヤーの外部にある — 状態管理コンポーネント（BLoC、Notifier、Controller、Store、ViewModelなど）内
- [ ] 状態マネージャーが内部構築ではなくインジェクションで依存関係を受け取る
- [ ] サービスまたはリポジトリレイヤーがデータソースを抽象化 — ウィジェットと状態マネージャーがAPIやデータベースを直接呼び出すべきでない
- [ ] 状態マネージャーが単一責任 — 無関係な関心事を扱う「ゴッド」マネージャーがない
- [ ] コンポーネント間依存関係がソリューションの規約に従う：
  - **Riverpod**：`ref.watch`を介したプロバイダー間依存は想定通り — 循環や過度に絡み合ったチェーンのみをフラグ
  - **BLoC**：BLoC同士が直接依存すべきでない — 共有リポジトリまたはプレゼンテーションレイヤーでの調整を優先
  - その他のソリューション：コンポーネント間通信のドキュメント化された規約に従う

### イミュータビリティ＆値の等価性（イミュータブル状態ソリューション：BLoC、Riverpod、Redux向け）：
- [ ] 状態オブジェクトがイミュータブル — `copyWith()`またはコンストラクタで新しいインスタンスを作成、インプレース変更しない
- [ ] 状態クラスが`==`と`hashCode`を適切に実装（すべてのフィールドが比較に含まれる）
- [ ] プロジェクト全体で一貫したメカニズム — 手動オーバーライド、`Equatable`、`freezed`、Dartレコード、その他
- [ ] 状態オブジェクト内のコレクションが生のミュータブル`List`/`Map`として公開されていない

### リアクティビティ規律（リアクティブミューテーションソリューション：MobX、GetX、Signals向け）：
- [ ] 状態がソリューションのリアクティブAPI（MobXの`@action`、Signalsの`.value`、GetXの`.obs`）を通じてのみ変更される — 直接フィールド変更は変更追跡をバイパス
- [ ] 派生値は冗長に保存されるのではなくソリューションのcomputedメカニズムを使用
- [ ] リアクションとディスポーザーが適切にクリーンアップされる（MobXの`ReactionDisposer`、Signalsのエフェクトクリーンアップ）

### 状態の形設計：
- [ ] 相互排他的な状態がsealedタイプ、ユニオンバリアント、またはソリューション組み込みの非同期状態タイプ（例：Riverpodの`AsyncValue`）を使用 — ブーリアンフラグ（`isLoading`、`isError`、`hasData`）ではない
- [ ] すべての非同期操作がローディング、成功、エラーを明確な状態としてモデル化
- [ ] すべての状態バリアントがUIで網羅的に処理されている — サイレントに無視されるケースがない
- [ ] エラー状態が表示用のエラー情報を持つ；ローディング状態が古いデータを持たない
- [ ] Nullableデータがローディングインジケーターとして使用されていない — 状態は明示的

```dart
// BAD — boolean flag soup allows impossible states
class UserState {
  bool isLoading = false;
  bool hasError = false; // isLoading && hasError is representable!
  User? user;
}

// GOOD (immutable approach) — sealed types make impossible states unrepresentable
sealed class UserState {}
class UserInitial extends UserState {}
class UserLoading extends UserState {}
class UserLoaded extends UserState {
  final User user;
  const UserLoaded(this.user);
}
class UserError extends UserState {
  final String message;
  const UserError(this.message);
}

// GOOD (reactive approach) — observable enum + data, mutations via reactivity API
// enum UserStatus { initial, loading, loaded, error }
// Use your solution's observable/signal to wrap status and data separately
```

### リビルド最適化：
- [ ] 状態コンシューマーウィジェット（Builder、Consumer、Observer、Obx、Watchなど）を可能な限り狭くスコープ
- [ ] セレクターを使用して特定フィールド変更時のみリビルド — 全状態変更ではなく
- [ ] `const`ウィジェットでツリーを通じたリビルド伝播を停止
- [ ] computed/派生状態が冗長に保存されるのではなくリアクティブに計算

### サブスクリプション＆破棄：
- [ ] すべての手動サブスクリプション（`.listen()`）が`dispose()` / `close()`でキャンセル
- [ ] ストリームコントローラーが不要になったら閉じる
- [ ] タイマーが破棄ライフサイクルでキャンセル
- [ ] 手動サブスクリプションよりフレームワーク管理のライフサイクルを優先（`.listen()`より宣言的ビルダー）
- [ ] 非同期コールバックでの`setState`前に`mounted`チェック
- [ ] `await`後に`context.mounted`（Flutter 3.7+）を確認せずに`BuildContext`を使用しない — 古いコンテキストはクラッシュの原因
- [ ] 非同期ギャップ後のナビゲーション、ダイアログ、スキャフォールドメッセージはウィジェットがまだマウントされていることを確認してから
- [ ] `BuildContext`をシングルトン、状態マネージャー、静的フィールドに保存しない

### ローカル vs グローバル状態：
- [ ] 一時的なUI状態（チェックボックス、スライダー、アニメーション）はローカル状態を使用（`setState`、`ValueNotifier`）
- [ ] 共有状態は必要な範囲まで持ち上げる — 過度にグローバル化しない
- [ ] フィーチャースコープの状態はフィーチャーがアクティブでなくなった時に適切に破棄

---

## 5. パフォーマンス

### 不要なリビルド：
- [ ] ルートウィジェットレベルで`setState()`を呼び出さない — 状態変更をローカライズ
- [ ] `const`ウィジェットでリビルド伝播を停止
- [ ] 独立して再ペイントする複雑なサブツリー周りに`RepaintBoundary`を使用
- [ ] アニメーションから独立したサブツリーに`AnimatedBuilder`のchildパラメータを使用

### build()内の高コスト操作：
- [ ] `build()`内で大きなコレクションのソート、フィルタリング、マッピングをしない — 状態管理レイヤーで計算
- [ ] `build()`内で正規表現コンパイルをしない
- [ ] `MediaQuery.of(context)`の使用を具体的に（例：`MediaQuery.sizeOf(context)`）

### 画像最適化：
- [ ] ネットワーク画像がキャッシュを使用（プロジェクトに適したキャッシュソリューション）
- [ ] ターゲットデバイスに適した画像解像度（サムネイルに4K画像をロードしない）
- [ ] 表示サイズでデコードするための`Image.asset`と`cacheWidth`/`cacheHeight`
- [ ] ネットワーク画像にプレースホルダーとエラーウィジェットを提供

### 遅延読み込み：
- [ ] 大きなリストや動的リストには`ListView(children: [...])`ではなく`ListView.builder` / `GridView.builder`を使用（小さな静的リストにはconcreteコンストラクタで可）
- [ ] 大きなデータセットにはページネーションを実装
- [ ] Webビルドの重いライブラリには遅延読み込み（`deferred as`）を使用

### その他：
- [ ] アニメーションで`Opacity`ウィジェットを避ける — `AnimatedOpacity`または`FadeTransition`を使用
- [ ] アニメーション中のクリッピングを避ける — 画像を事前クリップ
- [ ] ウィジェットで`operator ==`をオーバーライドしない — 代わりに`const`コンストラクタを使用
- [ ] 内在的寸法ウィジェット（`IntrinsicHeight`、`IntrinsicWidth`）は控えめに使用（追加レイアウトパス）

---

## 6. テスト

### テストタイプと期待：
- [ ] **ユニットテスト**：すべてのビジネスロジックをカバー（状態マネージャー、リポジトリ、ユーティリティ関数）
- [ ] **ウィジェットテスト**：個々のウィジェットの動作、インタラクション、ビジュアル出力をカバー
- [ ] **インテグレーションテスト**：クリティカルなユーザーフローをエンドツーエンドでカバー
- [ ] **ゴールデンテスト**：デザインクリティカルなUIコンポーネントのピクセルパーフェクト比較

### カバレッジ目標：
- [ ] ビジネスロジックで80%以上のラインカバレッジを目指す
- [ ] すべての状態遷移に対応するテスト（loading → success、loading → error、retryなど）
- [ ] エッジケースのテスト：空状態、エラー状態、ローディング状態、境界値

### テスト分離：
- [ ] 外部依存関係（APIクライアント、データベース、サービス）がモックまたはフェイク化
- [ ] 各テストファイルが正確に1つのクラス/ユニットをテスト
- [ ] テストが実装の詳細ではなく動作を検証
- [ ] スタブは各テストに必要な動作のみを定義（最小スタビング）
- [ ] テストケース間で共有ミュータブル状態がない

### ウィジェットテスト品質：
- [ ] 非同期操作で`pumpWidget`と`pump`が正しく使用されている
- [ ] `find.byType`、`find.text`、`find.byKey`が適切に使用されている
- [ ] タイミングに依存するフレーキーテストがない — `pumpAndSettle`または明示的な`pump(Duration)`を使用
- [ ] テストがCIで実行され、失敗がマージをブロック

---

## 7. アクセシビリティ

### セマンティックウィジェット：
- [ ] 自動ラベルが不十分な場合にスクリーンリーダーラベルを提供する`Semantics`ウィジェットの使用
- [ ] 純粋に装飾的な要素には`ExcludeSemantics`を使用
- [ ] 関連ウィジェットを単一のアクセシブル要素に結合する`MergeSemantics`を使用
- [ ] 画像に`semanticLabel`プロパティを設定

### スクリーンリーダーサポート：
- [ ] すべてのインタラクティブ要素がフォーカス可能で意味のある説明がある
- [ ] フォーカス順序が論理的（視覚的な読み順に従う）

### ビジュアルアクセシビリティ：
- [ ] テキストと背景のコントラスト比 >= 4.5:1
- [ ] タップ可能なターゲットが少なくとも48x48ピクセル
- [ ] 色が状態の唯一の指標でない（アイコン/テキストを併用）
- [ ] テキストがシステムフォントサイズ設定に合わせてスケール

### インタラクションアクセシビリティ：
- [ ] no-opの`onPressed`コールバックがない — すべてのボタンが何かを行うか無効化
- [ ] エラーフィールドが修正を提案
- [ ] ユーザーがデータ入力中にコンテキストが予期せず変更されない

---

## 8. プラットフォーム固有の考慮事項

### iOS/Androidの違い：
- [ ] 適切な箇所でプラットフォーム適応ウィジェットを使用
- [ ] バックナビゲーションが正しく処理される（Androidバックボタン、iOSスワイプバック）
- [ ] ステータスバーとセーフエリアが`SafeArea`ウィジェットで処理
- [ ] プラットフォーム固有のパーミッションが`AndroidManifest.xml`と`Info.plist`で宣言

### レスポンシブデザイン：
- [ ] レスポンシブレイアウトに`LayoutBuilder`または`MediaQuery`を使用
- [ ] ブレークポイントが一貫して定義（phone、tablet、desktop）
- [ ] 小さい画面でテキストがオーバーフローしない — `Flexible`、`Expanded`、`FittedBox`を使用
- [ ] ランドスケープ向きがテスト済みまたは明示的にロック
- [ ] Web固有：マウス/キーボードインタラクションがサポートされ、ホバー状態が存在

---

## 9. セキュリティ

### セキュアストレージ：
- [ ] 機密データ（トークン、クレデンシャル）がプラットフォームセキュアストレージを使用して保存（iOSのKeychain、AndroidのEncryptedSharedPreferences）
- [ ] シークレットをプレーンテキストストレージに保存しない
- [ ] 機密操作に対する生体認証ゲーティングを検討

### APIキー管理：
- [ ] APIキーがDartソースにハードコードされていない — `--dart-define`、VCSから除外された`.env`ファイル、またはコンパイル時設定を使用
- [ ] シークレットがgitにコミットされていない — `.gitignore`を確認
- [ ] 真に秘密の鍵にはバックエンドプロキシを使用（クライアントはサーバーシークレットを保持すべきでない）

### 入力バリデーション：
- [ ] すべてのユーザー入力がAPIに送信前にバリデーション
- [ ] フォームバリデーションが適切なバリデーションパターンを使用
- [ ] ユーザー入力の生SQLや文字列補間がない
- [ ] ディープリンクURLがナビゲーション前にバリデーションおよびサニタイズ

### ネットワークセキュリティ：
- [ ] すべてのAPI呼び出しでHTTPSを強制
- [ ] 高セキュリティアプリに対する証明書ピンニングの検討
- [ ] 認証トークンが適切にリフレッシュおよび失効
- [ ] 機密データがログまたはプリントされない

---

## 10. パッケージ/依存関係レビュー

### pub.devパッケージの評価：
- [ ] **pubポイントスコア**を確認（130+/160を目標）
- [ ] コミュニティシグナルとして**いいね**と**人気度**を確認
- [ ] pub.devでパブリッシャーが**認証済み**であることを確認
- [ ] 最終公開日を確認 — 古いパッケージ（1年以上）はリスク
- [ ] オープンイシューとメンテナーの応答時間をレビュー
- [ ] ライセンス互換性を確認
- [ ] ターゲットのプラットフォームサポートを確認

### バージョン制約：
- [ ] 依存関係にキャレット構文（`^1.2.3`）を使用 — 互換性のあるアップデートを許可
- [ ] 正確なバージョンピンニングは絶対に必要な場合のみ
- [ ] `flutter pub outdated`を定期的に実行して古い依存関係を追跡
- [ ] プロダクション`pubspec.yaml`にdependency overridesがない — 一時的な修正のみコメント/イシューリンク付きで
- [ ] 推移的依存関係の数を最小化 — 各依存関係はアタックサーフェス

### モノレポ固有（melos/workspace）：
- [ ] 内部パッケージがパブリックAPIからのみインポート — `package:other/src/internal.dart`はなし（Dartパッケージのカプセル化を破壊）
- [ ] 内部パッケージの依存関係がハードコードの`path: ../../`相対文字列ではなくワークスペース解決を使用
- [ ] すべてのサブパッケージがルートの`analysis_options.yaml`を共有または継承

---

## 11. ナビゲーションとルーティング

### 一般原則（あらゆるルーティングソリューションに適用）：
- [ ] 1つのルーティングアプローチを一貫して使用 — 命令的`Navigator.push`と宣言的ルーターを混在させない
- [ ] ルート引数が型付き — `Map<String, dynamic>`や`Object?`キャストはなし
- [ ] ルートパスが定数、列挙型、または生成で定義 — コードに散在するマジック文字列はなし
- [ ] 認証ガード/リダイレクトが一元化 — 個々の画面で重複しない
- [ ] AndroidとiOSの両方でディープリンクが設定
- [ ] ディープリンクURLがナビゲーション前にバリデーションおよびサニタイズ
- [ ] ナビゲーション状態がテスト可能 — ルート変更がテストで検証可能
- [ ] すべてのプラットフォームでバック動作が正しい

---

## 12. エラーハンドリング

### フレームワークエラーハンドリング：
- [ ] フレームワークエラー（build、layout、paint）をキャプチャするために`FlutterError.onError`がオーバーライド
- [ ] Flutterでキャッチされない非同期エラー用に`PlatformDispatcher.instance.onError`を設定
- [ ] リリースモード用に`ErrorWidget.builder`がカスタマイズ（赤い画面ではなくユーザーフレンドリー）
- [ ] `runApp`周りのグローバルエラーキャプチャラッパー（例：`runZonedGuarded`、Sentry/Crashlyticsラッパー）

### エラー報告：
- [ ] エラー報告サービスが統合（Firebase Crashlytics、Sentry、または同等品）
- [ ] 非致命的エラーがスタックトレース付きで報告
- [ ] 状態管理エラーオブザーバーがエラー報告に接続（例：BlocObserver、ProviderObserver、またはソリューション同等品）
- [ ] デバッグ用にユーザー識別可能情報（ユーザーID）がエラーレポートに添付

### グレースフルデグラデーション：
- [ ] APIエラーがクラッシュではなくユーザーフレンドリーなエラーUIに
- [ ] 一時的なネットワーク障害に対するリトライメカニズム
- [ ] オフライン状態がグレースフルに処理
- [ ] 状態管理のエラー状態が表示用のエラー情報を保持
- [ ] 生の例外（ネットワーク、パース）がUIに到達する前にユーザーフレンドリーでローカライズされたメッセージにマッピング — 生の例外文字列をユーザーに表示しない

---

## 13. 国際化（l10n）

### セットアップ：
- [ ] ローカライゼーションソリューションが設定（Flutterの組み込みARB/l10n、easy_localization、または同等品）
- [ ] サポートされるロケールがアプリ設定で宣言

### コンテンツ：
- [ ] すべてのユーザー表示文字列がローカライゼーションシステムを使用 — ウィジェットにハードコード文字列がない
- [ ] テンプレートファイルに翻訳者向けの説明/コンテキストが含まれる
- [ ] 複数形、性別、選択にICUメッセージ構文を使用
- [ ] プレースホルダーが型付きで定義
- [ ] ロケール間で欠落キーがない

### コードレビュー：
- [ ] ローカライゼーションアクセサーがプロジェクト全体で一貫して使用
- [ ] 日付、時間、数値、通貨のフォーマットがロケール対応
- [ ] アラビア語、ヘブライ語などを対象とする場合、テキスト方向（RTL）がサポート
- [ ] ローカライズされたテキストに文字列連結を使用しない — パラメータ化メッセージを使用

---

## 14. 依存性注入

### 原則（あらゆるDIアプローチに適用）：
- [ ] レイヤー境界でクラスが具象実装ではなく抽象（インターフェース）に依存
- [ ] 依存関係がコンストラクタ、DIフレームワーク、またはプロバイダーグラフを介して外部から提供 — 内部で作成されない
- [ ] 登録がライフタイムを区別：シングルトン vs ファクトリー vs 遅延シングルトン
- [ ] 環境固有のバインディング（dev/staging/prod）がランタイム`if`チェックではなく設定を使用
- [ ] DIグラフに循環依存がない
- [ ] サービスロケーター呼び出し（使用する場合）がビジネスロジック全体に散在していない

---

## 15. 静的解析

### 設定：
- [ ] `analysis_options.yaml`が存在し厳格な設定が有効
- [ ] 厳格なアナライザー設定：`strict-casts: true`、`strict-inference: true`、`strict-raw-types: true`
- [ ] 包括的なリントルールセットが含まれている（very_good_analysis、flutter_lints、またはカスタム厳格ルール）
- [ ] モノレポのすべてのサブパッケージがルートのanalysis optionsを継承または共有

### 強制：
- [ ] コミット済みコードに未解決のアナライザー警告がない
- [ ] リント抑制（`// ignore:`）が理由を説明するコメント付きで正当化
- [ ] `flutter analyze`がCIで実行され、失敗がマージをブロック

### リントパッケージに関わらず確認すべき主要ルール：
- [ ] `prefer_const_constructors` — ウィジェットツリーでのパフォーマンス
- [ ] `avoid_print` — 適切なロギングを使用
- [ ] `unawaited_futures` — fire-and-forgetの非同期バグを防止
- [ ] `prefer_final_locals` — 変数レベルのイミュータビリティ
- [ ] `always_declare_return_types` — 明示的なコントラクト
- [ ] `avoid_catches_without_on_clauses` — 具体的なエラーハンドリング
- [ ] `always_use_package_imports` — 一貫したインポートスタイル

---

## 状態管理クイックリファレンス

以下の表は、普遍的な原則を一般的なソリューションでの実装にマッピングしています。プロジェクトが使用するソリューションに合わせてレビュールールを適応させてください。

| 原則 | BLoC/Cubit | Riverpod | Provider | GetX | MobX | Signals | 組み込み |
|-----------|-----------|----------|----------|------|------|---------|----------|
| 状態コンテナ | `Bloc`/`Cubit` | `Notifier`/`AsyncNotifier` | `ChangeNotifier` | `GetxController` | `Store` | `signal()` | `StatefulWidget` |
| UIコンシューマー | `BlocBuilder` | `ConsumerWidget` | `Consumer` | `Obx`/`GetBuilder` | `Observer` | `Watch` | `setState` |
| セレクター | `BlocSelector`/`buildWhen` | `ref.watch(p.select(...))` | `Selector` | N/A | computed | `computed()` | N/A |
| サイドエフェクト | `BlocListener` | `ref.listen` | `Consumer` callback | `ever()`/`once()` | `reaction` | `effect()` | callbacks |
| 破棄 | `BlocProvider`経由で自動 | `.autoDispose` | `Provider`経由で自動 | `onClose()` | `ReactionDisposer` | 手動 | `dispose()` |
| テスト | `blocTest()` | `ProviderContainer` | `ChangeNotifier`直接 | テストで`Get.put` | store直接 | signal直接 | widget test |

---

## ソース

- [Effective Dart: Style](https://dart.dev/effective-dart/style)
- [Effective Dart: Usage](https://dart.dev/effective-dart/usage)
- [Effective Dart: Design](https://dart.dev/effective-dart/design)
- [Flutter Performance Best Practices](https://docs.flutter.dev/perf/best-practices)
- [Flutter Testing Overview](https://docs.flutter.dev/testing/overview)
- [Flutter Accessibility](https://docs.flutter.dev/ui/accessibility-and-internationalization/accessibility)
- [Flutter Internationalization](https://docs.flutter.dev/ui/accessibility-and-internationalization/internationalization)
- [Flutter Navigation and Routing](https://docs.flutter.dev/ui/navigation)
- [Flutter Error Handling](https://docs.flutter.dev/testing/errors)
- [Flutter State Management Options](https://docs.flutter.dev/data-and-backend/state-mgmt/options)
