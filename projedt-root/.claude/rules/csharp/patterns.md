---
paths:
  - "**/*.cs"
  - "**/*.csx"
---
# C# パターン

> このファイルは [common/patterns.md](../common/patterns.md) をC#固有の内容で拡張します。

## APIレスポンスパターン

```csharp
public sealed record ApiResponse<T>(
    bool Success,
    T? Data = default,
    string? Error = null,
    object? Meta = null);
```

## リポジトリパターン

```csharp
public interface IRepository<T>
{
    Task<IReadOnlyList<T>> FindAllAsync(CancellationToken cancellationToken);
    Task<T?> FindByIdAsync(Guid id, CancellationToken cancellationToken);
    Task<T> CreateAsync(T entity, CancellationToken cancellationToken);
    Task<T> UpdateAsync(T entity, CancellationToken cancellationToken);
    Task DeleteAsync(Guid id, CancellationToken cancellationToken);
}
```

## Optionsパターン

コードベース全体で生の文字列を読み取る代わりに、設定には強く型付けされたOptionsを使用する。

```csharp
public sealed class PaymentsOptions
{
    public const string SectionName = "Payments";
    public required string BaseUrl { get; init; }
    public required string ApiKeySecretName { get; init; }
}
```

## 依存性注入

- サービス境界ではインターフェースに依存する
- コンストラクタは焦点を絞る。サービスの依存関係が多すぎる場合は責務を分割する
- ライフタイムは意図を持って登録する: ステートレス/共有サービスにはsingleton、リクエストデータにはscoped、軽量な純粋ワーカーにはtransient
