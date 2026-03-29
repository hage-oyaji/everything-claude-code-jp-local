---
description: Kotlinコードの包括的レビュー。イディオマティックなパターン、null安全性、コルーチンの安全性、セキュリティをチェック。kotlin-reviewerエージェントを呼び出します。
---

# Kotlinコードレビュー

このコマンドは **kotlin-reviewer** エージェントを呼び出し、Kotlin固有の包括的なコードレビューを行います。

## このコマンドの動作

1. **Kotlinの変更を特定**: `git diff` で変更された `.kt` および `.kts` ファイルを検出
2. **ビルドと静的解析を実行**: `./gradlew build`、`detekt`、`ktlintCheck` を実行
3. **セキュリティスキャン**: SQLインジェクション、コマンドインジェクション、ハードコードされたシークレットをチェック
4. **Null安全性レビュー**: `!!` の使用、プラットフォーム型の処理、安全でないキャストを分析
5. **コルーチンレビュー**: 構造化された並行性、ディスパッチャの使用、キャンセルをチェック
6. **レポートを生成**: 重要度別に問題を分類

## 使用するタイミング

`/kotlin-review` は以下の場合に使用します：
- Kotlinコードの作成または修正後
- Kotlinの変更をコミットする前
- Kotlinコードを含むプルリクエストのレビュー時
- 新しいKotlinコードベースのオンボーディング時
- イディオマティックなKotlinパターンの学習時

## レビューカテゴリ

### CRITICAL（必須修正）
- SQL/コマンドインジェクションの脆弱性
- 正当な理由のないforce-unwrap `!!`
- プラットフォーム型のnull安全性違反
- GlobalScopeの使用（構造化された並行性の違反）
- ハードコードされた認証情報
- 安全でないデシリアライゼーション

### HIGH（修正推奨）
- 不変で十分な箇所でのミュータブルな状態
- コルーチンコンテキスト内でのブロッキングコール
- 長いループでのキャンセルチェックの欠如
- sealed型での非網羅的な `when`
- 大きな関数（50行超）
- 深いネスト（4レベル超）

### MEDIUM（検討推奨）
- 非イディオマティックなKotlin（Javaスタイルのパターン）
- 末尾カンマの欠如
- スコープ関数の誤用またはネスト
- 大きなコレクションチェーンでのsequenceの欠如
- 冗長な明示的型

## 実行される自動チェック

```bash
# Build check
./gradlew build

# Static analysis
./gradlew detekt

# Formatting check
./gradlew ktlintCheck

# Tests
./gradlew test
```

## 使用例

````text
User: /kotlin-review

Agent:
# Kotlin Code Review Report

## Files Reviewed
- src/main/kotlin/com/example/service/UserService.kt (modified)
- src/main/kotlin/com/example/routes/UserRoutes.kt (modified)

## Static Analysis Results
✓ Build: Successful
✓ detekt: No issues
⚠ ktlint: 2 formatting warnings

## Issues Found

[CRITICAL] Force-Unwrap Null Safety
File: src/main/kotlin/com/example/service/UserService.kt:28
Issue: Using !! on nullable repository result
```kotlin
val user = repository.findById(id)!!  // NPE risk
```
Fix: Use safe call with error handling
```kotlin
val user = repository.findById(id)
    ?: throw UserNotFoundException("User $id not found")
```

[HIGH] GlobalScope Usage
File: src/main/kotlin/com/example/routes/UserRoutes.kt:45
Issue: Using GlobalScope breaks structured concurrency
```kotlin
GlobalScope.launch {
    notificationService.sendWelcome(user)
}
```
Fix: Use the call's coroutine scope
```kotlin
launch {
    notificationService.sendWelcome(user)
}
```

## Summary
- CRITICAL: 1
- HIGH: 1
- MEDIUM: 0

Recommendation: ❌ Block merge until CRITICAL issue is fixed
````

## 承認基準

| ステータス | 条件 |
|--------|-----------|
| ✅ 承認 | CRITICALまたはHIGHの問題なし |
| ⚠️ 警告 | MEDIUMの問題のみ（注意してマージ） |
| ❌ ブロック | CRITICALまたはHIGHの問題あり |

## 他のコマンドとの連携

- `/kotlin-test` を先に使用してテストが通ることを確認
- ビルドエラーが発生した場合は `/kotlin-build` を使用
- コミット前に `/kotlin-review` を使用
- Kotlin固有でない問題には `/code-review` を使用

## 関連

- エージェント: `agents/kotlin-reviewer.md`
- スキル: `skills/kotlin-patterns/`、`skills/kotlin-testing/`
