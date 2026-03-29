---
paths:
  - "**/*.cs"
  - "**/*.csx"
---
# C# コーディングスタイル

> このファイルは [common/coding-style.md](../common/coding-style.md) をC#固有の内容で拡張します。

## 標準

- 現行の.NET規約に従い、null許容参照型を有効にする
- publicおよびinternalのAPIには明示的なアクセス修飾子を付ける
- ファイルは定義する主要な型に合わせる

## 型とモデル

- イミュータブルな値型モデルには `record` または `record struct` を優先する
- IDとライフサイクルを持つエンティティには `class` を使用する
- サービス境界と抽象化には `interface` を使用する
- アプリケーションコードでは `dynamic` を避け、ジェネリクスまたは明示的なモデルを使用する

```csharp
public sealed record UserDto(Guid Id, string Email);

public interface IUserRepository
{
    Task<UserDto?> FindByIdAsync(Guid id, CancellationToken cancellationToken);
}
```

## イミュータビリティ

- 共有状態には `init` セッター、コンストラクタパラメータ、イミュータブルコレクションを優先する
- 更新後の状態を生成する際に入力モデルをインプレースで変更しない

```csharp
public sealed record UserProfile(string Name, string Email);

public static UserProfile Rename(UserProfile profile, string name) =>
    profile with { Name = name };
```

## 非同期とエラーハンドリング

- `.Result` や `.Wait()` のようなブロッキング呼び出しよりも `async`/`await` を優先する
- publicな非同期APIには `CancellationToken` を渡す
- 具体的な例外をスローし、構造化プロパティでログを記録する

```csharp
public async Task<Order> LoadOrderAsync(
    Guid orderId,
    CancellationToken cancellationToken)
{
    try
    {
        return await repository.FindAsync(orderId, cancellationToken)
            ?? throw new InvalidOperationException($"Order {orderId} was not found.");
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Failed to load order {OrderId}", orderId);
        throw;
    }
}
```

## フォーマット

- フォーマットとアナライザ修正には `dotnet format` を使用する
- `using` ディレクティブを整理し、未使用のインポートを削除する
- 式形式のメンバは可読性を保てる場合のみ使用する
