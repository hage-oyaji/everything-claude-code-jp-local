---
paths:
  - "**/*.kt"
  - "**/*.kts"
---
# Kotlin コーディングスタイル

> このファイルは [common/coding-style.md](../common/coding-style.md) を Kotlin 固有の内容で拡張します。

## フォーマット

- スタイル適用には **ktlint** または **Detekt** を使用
- 公式 Kotlin コードスタイル（`gradle.properties` に `kotlin.code.style=official`）

## イミュータビリティ

- `var` よりも `val` を優先 — デフォルトで `val` を使い、ミューテーションが必要な場合のみ `var` を使用
- 値型には `data class` を使用。パブリック API ではイミュータブルコレクション（`List`、`Map`、`Set`）を使用
- 状態更新にはコピーオンライト: `state.copy(field = newValue)`

## 命名規則

Kotlin の規約に従う：
- 関数とプロパティには `camelCase`
- クラス、インターフェース、オブジェクト、型エイリアスには `PascalCase`
- 定数には `SCREAMING_SNAKE_CASE`（`const val` または `@JvmStatic`）
- インターフェースには `I` ではなく動作を表すプレフィックス: `IClickable` ではなく `Clickable`

## Null 安全

- `!!` は絶対に使わない — `?.`、`?:`、`requireNotNull()`、`checkNotNull()` を優先
- スコープ付き null 安全操作には `?.let {}` を使用
- 正当に結果がない可能性のある関数からは nullable 型を返す

```kotlin
// BAD
val name = user!!.name

// GOOD
val name = user?.name ?: "Unknown"
val name = requireNotNull(user) { "User must be set before accessing name" }.name
```

## シールド型

閉じた状態階層をモデル化するにはシールドクラス/インターフェースを使用：

```kotlin
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val message: String) : UiState<Nothing>
}
```

シールド型では必ず網羅的な `when` を使用 — `else` ブランチは使わない。

## 拡張関数

ユーティリティ操作には拡張関数を使用するが、発見しやすさを保つ：
- レシーバー型にちなんだファイル名に配置（`StringExt.kt`、`FlowExt.kt`）
- スコープを限定 — `Any` や過度に汎用的な型への拡張は追加しない

## スコープ関数

適切なスコープ関数を使用：
- `let` — null チェック + 変換: `user?.let { greet(it) }`
- `run` — レシーバーを使って結果を計算: `service.run { fetch(config) }`
- `apply` — オブジェクトの設定: `builder.apply { timeout = 30 }`
- `also` — 副作用: `result.also { log(it) }`
- スコープ関数の深いネストは避ける（最大2レベル）

## エラーハンドリング

- `Result<T>` またはカスタムシールド型を使用
- throwable コードのラップには `runCatching {}` を使用
- `CancellationException` は絶対にキャッチしない — 常に再スローする
- 制御フローに `try-catch` を使わない

```kotlin
// BAD — using exceptions for control flow
val user = try { repository.getUser(id) } catch (e: NotFoundException) { null }

// GOOD — nullable return
val user: User? = repository.findUser(id)
```
