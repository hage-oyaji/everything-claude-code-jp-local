---
description: Kotlin/Gradleのビルドエラー、コンパイラ警告、依存関係の問題をインクリメンタルに修正。最小限の外科的修正のためにkotlin-build-resolverエージェントを呼び出します。
---

# Kotlinビルドと修正

このコマンドは **kotlin-build-resolver** エージェントを呼び出し、最小限の変更でKotlinビルドエラーをインクリメンタルに修正します。

## このコマンドの動作

1. **診断を実行**: `./gradlew build`、`detekt`、`ktlintCheck` を実行
2. **エラーを解析**: ファイルごとにグループ化し、重要度順にソート
3. **インクリメンタルに修正**: エラーを1つずつ修正
4. **各修正を検証**: 変更後にビルドを再実行
5. **サマリーを報告**: 修正済みと残りの問題を表示

## 使用するタイミング

`/kotlin-build` は以下の場合に使用します：
- `./gradlew build` がエラーで失敗した場合
- Kotlinコンパイラがエラーを報告した場合
- `./gradlew detekt` が違反を報告した場合
- Gradleの依存関係解決が失敗した場合
- ビルドを壊す変更をプルした後

## 実行される診断コマンド

```bash
# Primary build check
./gradlew build 2>&1

# Static analysis
./gradlew detekt 2>&1 || echo "detekt not configured"
./gradlew ktlintCheck 2>&1 || echo "ktlint not configured"

# Dependency issues
./gradlew dependencies --configuration runtimeClasspath 2>&1 | head -100

# Optional deep refresh when caches or dependency metadata are suspect
./gradlew build --refresh-dependencies
```

## セッション例

````text
User: /kotlin-build

Agent:
# Kotlin Build Resolution

## Initial Diagnostics

```bash
$ ./gradlew build
e: src/main/kotlin/com/example/service/UserService.kt:25:15 Unresolved reference: UserRepository
e: src/main/kotlin/com/example/routes/UserRoutes.kt:42:9 Type mismatch: inferred type is String but Int was expected
e: src/main/kotlin/com/example/routes/UserRoutes.kt:58:5 'when' expression must be exhaustive
```

Errors found: 3

## Fix 1: Unresolved Reference

File: src/main/kotlin/com/example/service/UserService.kt:25
Error: Unresolved reference: UserRepository
Cause: Missing import

```kotlin
// Added import
import com.example.repository.UserRepository
```

```bash
$ ./gradlew build
# 2 errors remaining
```

## Fix 2: Type Mismatch

File: src/main/kotlin/com/example/routes/UserRoutes.kt:42
Error: Type mismatch: inferred type is String but Int was expected

```kotlin
// Changed
val count = call.parameters["count"]
// To
val count = call.parameters["count"]?.toIntOrNull()
    ?: return@get call.respond(HttpStatusCode.BadRequest, "Invalid count")
```

```bash
$ ./gradlew build
# 1 error remaining
```

## Fix 3: Non-Exhaustive When

File: src/main/kotlin/com/example/routes/UserRoutes.kt:58
Error: 'when' expression must be exhaustive

```kotlin
// Added missing branch
when (user.role) {
    Role.ADMIN -> handleAdmin(user)
    Role.USER -> handleUser(user)
    Role.MODERATOR -> handleModerator(user) // Added
}
```

```bash
$ ./gradlew build
# Build successful!
```

## Final Verification

```bash
$ ./gradlew detekt
# No issues

$ ./gradlew test
# All tests passed
```

## Summary

| Metric | Count |
|--------|-------|
| Build errors fixed | 3 |
| Detekt issues fixed | 0 |
| Files modified | 2 |
| Remaining issues | 0 |

Build Status: ✅ SUCCESS
````

## よくあるエラーと修正

| エラー | 一般的な修正 |
|-------|-------------|
| `Unresolved reference: X` | インポートまたは依存関係を追加 |
| `Type mismatch` | 型変換または代入を修正 |
| `'when' must be exhaustive` | 不足しているsealed classの分岐を追加 |
| `Suspend function can only be called from coroutine` | `suspend` 修飾子を追加 |
| `Smart cast impossible` | ローカル `val` または `let` を使用 |
| `None of the following candidates is applicable` | 引数の型を修正 |
| `Could not resolve dependency` | バージョンを修正またはリポジトリを追加 |

## 修正戦略

1. **ビルドエラーを最優先** - コードがコンパイルできること
2. **Detekt違反を次に** - コード品質の問題を修正
3. **ktlint警告を最後に** - フォーマットを修正
4. **1つずつ修正** - 各変更を検証
5. **最小限の変更** - リファクタリングせず、修正のみ

## 停止条件

エージェントは以下の場合に停止して報告します：
- 3回の試行後も同じエラーが続く
- 修正によりエラーが増える
- アーキテクチャの変更が必要
- 外部依存関係が不足

## 関連コマンド

- `/kotlin-test` - ビルド成功後にテストを実行
- `/kotlin-review` - コード品質をレビュー
- `/verify` - フル検証ループ

## 関連

- エージェント: `agents/kotlin-build-resolver.md`
- スキル: `skills/kotlin-patterns/`
