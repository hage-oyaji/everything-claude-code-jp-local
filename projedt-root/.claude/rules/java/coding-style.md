---
paths:
  - "**/*.java"
---
# Java コーディングスタイル

> このファイルは [common/coding-style.md](../common/coding-style.md) をJava固有の内容で拡張します。

## フォーマット

- フォーマット適用には **google-java-format** または **Checkstyle**（GoogleまたはSunスタイル）を使用する
- 1ファイルにつき1つのpublicトップレベル型
- 一貫したインデント: 2または4スペース（プロジェクト標準に合わせる）
- メンバの順序: 定数、フィールド、コンストラクタ、publicメソッド、protected、private

## イミュータビリティ

- 値型には `record` を優先する（Java 16+）
- フィールドはデフォルトで `final` にする — ミュータブルな状態は必要な場合のみ使用する
- publicなAPIからは防御的コピーを返す: `List.copyOf()`、`Map.copyOf()`、`Set.copyOf()`
- コピーオンライト: 既存のオブジェクトを変更するのではなく、新しいインスタンスを返す

```java
// GOOD — イミュータブルな値型
public record OrderSummary(Long id, String customerName, BigDecimal total) {}

// GOOD — finalフィールド、セッターなし
public class Order {
    private final Long id;
    private final List<LineItem> items;

    public List<LineItem> getItems() {
        return List.copyOf(items);
    }
}
```

## 命名

標準のJava規約に従う:
- クラス、インターフェース、レコード、列挙型は `PascalCase`
- メソッド、フィールド、パラメータ、ローカル変数は `camelCase`
- `static final` 定数は `SCREAMING_SNAKE_CASE`
- パッケージ: すべて小文字、逆ドメイン（`com.example.app.service`）

## モダンJava機能

明確性が向上する場合はモダンな言語機能を使用する:
- DTOと値型には**レコード**（Java 16+）
- 閉じた型階層には**シールドクラス**（Java 17+）
- `instanceof` の**パターンマッチング** — 明示的キャスト不要（Java 16+）
- 複数行文字列には**テキストブロック** — SQL、JSONテンプレート（Java 15+）
- アロー構文の**switch式**（Java 14+）
- **switchでのパターンマッチング** — シールド型の網羅的ハンドリング（Java 21+）

```java
// パターンマッチング instanceof
if (shape instanceof Circle c) {
    return Math.PI * c.radius() * c.radius();
}

// シールド型階層
public sealed interface PaymentMethod permits CreditCard, BankTransfer, Wallet {}

// switch式
String label = switch (status) {
    case ACTIVE -> "Active";
    case SUSPENDED -> "Suspended";
    case CLOSED -> "Closed";
};
```

## Optionalの使い方

- 結果がない可能性のあるfinderメソッドからは `Optional<T>` を返す
- `map()`、`flatMap()`、`orElseThrow()` を使用する — `isPresent()` なしで `get()` を呼ばない
- フィールド型やメソッドパラメータに `Optional` を使用しない

```java
// GOOD
return repository.findById(id)
    .map(ResponseDto::from)
    .orElseThrow(() -> new OrderNotFoundException(id));

// BAD — パラメータとしてのOptional
public void process(Optional<String> name) {}
```

## エラーハンドリング

- ドメインエラーには非チェック例外を優先する
- `RuntimeException` を拡張したドメイン固有の例外を作成する
- トップレベルハンドラ以外では広範な `catch (Exception e)` を避ける
- 例外メッセージにコンテキストを含める

```java
public class OrderNotFoundException extends RuntimeException {
    public OrderNotFoundException(Long id) {
        super("Order not found: id=" + id);
    }
}
```

## ストリーム

- 変換にはストリームを使用する。パイプラインは短く保つ（最大3〜4操作）
- 可読性がある場合はメソッド参照を優先する: `.map(Order::getTotal)`
- ストリーム操作で副作用を避ける
- 複雑なロジックの場合は、複雑なストリームパイプラインよりもループを優先する

## 参考

スキル: `java-coding-standards` で完全なコーディング標準と例を参照してください。
スキル: `jpa-patterns` でJPA/Hibernateエンティティ設計パターンを参照してください。
