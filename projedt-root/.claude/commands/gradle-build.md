---
description: AndroidおよびKMPプロジェクトのGradleビルドエラーを修正する
---

# Gradle ビルド修正

AndroidおよびKotlin Multiplatformプロジェクトのビルドおよびコンパイルエラーを段階的に修正します。

## ステップ1: ビルド設定の検出

プロジェクトタイプを特定し、適切なビルドを実行する:

| 判定基準 | ビルドコマンド |
|-----------|---------------|
| `build.gradle.kts` + `composeApp/`（KMP） | `./gradlew composeApp:compileKotlinMetadata 2>&1` |
| `build.gradle.kts` + `app/`（Android） | `./gradlew app:compileDebugKotlin 2>&1` |
| `settings.gradle.kts` にモジュールあり | `./gradlew assemble 2>&1` |
| Detektが設定済み | `./gradlew detekt 2>&1` |

`gradle.properties` と `local.properties` の設定も確認する。

## ステップ2: エラーの解析とグループ化

1. ビルドコマンドを実行し、出力をキャプチャする
2. Kotlinコンパイルエラーと Gradle設定エラーを分離する
3. モジュールとファイルパスでグループ化する
4. ソート: 設定エラーを先に、次に依存順序でコンパイルエラー

## ステップ3: 修正ループ

各エラーに対して:

1. **ファイルを読み取る** — エラー行周辺の完全なコンテキスト
2. **診断する** — 一般的なカテゴリ:
   - インポートの不足または未解決の参照
   - 型の不一致または互換性のない型
   - `build.gradle.kts` の依存関係の不足
   - expect/actualの不一致（KMP）
   - Composeコンパイラエラー
3. **最小限に修正する** — エラーを解決する最小限の変更
4. **ビルドを再実行する** — 修正を検証し、新しいエラーを確認する
5. **続行する** — 次のエラーに移動

## ステップ4: ガードレール

以下の場合はユーザーに確認する:
- 修正が解決するよりも多くのエラーを引き起こす場合
- 同じエラーが3回試行しても解消しない場合
- 新しい依存関係の追加やモジュール構造の変更が必要な場合
- Gradleの同期自体が失敗する場合（設定フェーズエラー）
- 生成されたコード（Room、SQLDelight、KSP）のエラーの場合

## ステップ5: サマリー

レポート:
- 修正したエラー（モジュール、ファイル、説明）
- 残存エラー
- 新たに発生したエラー（ゼロであるべき）
- 次のステップの提案

## よくあるGradle/KMPの修正

| エラー | 修正 |
|-------|-----|
| `commonMain` の未解決参照 | 依存関係が `commonMain.dependencies {}` にあるか確認 |
| actualのないexpect宣言 | 各プラットフォームソースセットに `actual` 実装を追加 |
| Composeコンパイラのバージョン不一致 | `libs.versions.toml` でKotlinとComposeコンパイラのバージョンを揃える |
| 重複クラス | `./gradlew dependencies` で競合する依存関係を確認 |
| KSPエラー | `./gradlew kspCommonMainKotlinMetadata` を実行して再生成 |
| 設定キャッシュの問題 | シリアライズ不可能なタスク入力を確認 |
