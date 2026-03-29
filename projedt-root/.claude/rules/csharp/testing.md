---
paths:
  - "**/*.cs"
  - "**/*.csx"
  - "**/*.csproj"
---
# C# テスト

> このファイルは [common/testing.md](../common/testing.md) をC#固有の内容で拡張します。

## テストフレームワーク

- ユニットテストと統合テストには **xUnit** を優先する
- 可読性の高いアサーションには **FluentAssertions** を使用する
- 依存関係のモックには **Moq** または **NSubstitute** を使用する
- 実際のインフラストラクチャが必要な統合テストには **Testcontainers** を使用する

## テストの構成

- `tests/` 配下に `src/` の構造をミラーリングする
- ユニット、統合、エンドツーエンドのカバレッジを明確に分離する
- テスト名は実装の詳細ではなく動作で命名する

```csharp
public sealed class OrderServiceTests
{
    [Fact]
    public async Task FindByIdAsync_ReturnsOrder_WhenOrderExists()
    {
        // Arrange
        // Act
        // Assert
    }
}
```

## ASP.NET Core統合テスト

- API統合カバレッジには `WebApplicationFactory<TEntryPoint>` を使用する
- ミドルウェアをバイパスせず、HTTP経由で認証、バリデーション、シリアライゼーションをテストする

## カバレッジ

- 行カバレッジ80%以上を目標にする
- ドメインロジック、バリデーション、認証、失敗パスにカバレッジを集中させる
- CIでカバレッジ収集を有効にして `dotnet test` を実行する
