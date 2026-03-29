---
paths:
  - "**/*.cs"
  - "**/*.csx"
  - "**/*.csproj"
  - "**/*.sln"
  - "**/Directory.Build.props"
  - "**/Directory.Build.targets"
---
# C# フック

> このファイルは [common/hooks.md](../common/hooks.md) をC#固有の内容で拡張します。

## PostToolUseフック

`~/.claude/settings.json` で設定する:

- **dotnet format**: 編集されたC#ファイルの自動フォーマットとアナライザ修正の適用
- **dotnet build**: 編集後にソリューションまたはプロジェクトがコンパイルされることを確認
- **dotnet test --no-build**: 動作変更後に最も関連するテストプロジェクトを再実行

## Stopフック

- 広範なC#変更を含むセッション終了前に最終的な `dotnet build` を実行する
- シークレットがコミットされないよう、変更された `appsettings*.json` ファイルについて警告する
