---
paths:
  - "**/*.java"
  - "**/pom.xml"
  - "**/build.gradle"
  - "**/build.gradle.kts"
---
# Java フック

> このファイルは [common/hooks.md](../common/hooks.md) をJava固有の内容で拡張します。

## PostToolUseフック

`~/.claude/settings.json` で設定する:

- **google-java-format**: 編集後に `.java` ファイルを自動フォーマット
- **checkstyle**: Javaファイル編集後にスタイルチェックを実行
- **./mvnw compile** または **./gradlew compileJava**: 変更後にコンパイルを検証
