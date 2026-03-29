---
name: java-coding-standards
description: "Spring Bootサービスのためのコーディング規約: 命名規則、イミュータビリティ、Optional使用法、ストリーム、例外、ジェネリクス、プロジェクト構成。"
origin: ECC
---

# Javaコーディング規約

Spring Bootサービスにおける可読性と保守性の高いJava（17以上）コードの規約。

## アクティベーション条件

- Spring BootプロジェクトでJavaコードを書くまたはレビューする時
- 命名規則、イミュータビリティ、例外処理の規約を適用する時
- records、sealed classes、パターンマッチング（Java 17以上）を扱う時
- Optional、ストリーム、ジェネリクスの使用をレビューする時
- パッケージ構成とプロジェクトレイアウトを設計する時

## コア原則

- 巧妙さより明確さを優先
- デフォルトでイミュータブル。共有ミュータブル状態を最小化
- 意味のある例外で早期失敗
- 一貫した命名規則とパッケージ構成

## 命名規則

```java
// ✅ Classes/Records: PascalCase
public class MarketService {}
public record Money(BigDecimal amount, Currency currency) {}

// ✅ Methods/fields: camelCase
private final MarketRepository marketRepository;
public Market findBySlug(String slug) {}

// ✅ Constants: UPPER_SNAKE_CASE
private static final int MAX_PAGE_SIZE = 100;
```

## イミュータビリティ

```java
// ✅ recordsとfinalフィールドを優先
public record MarketDto(Long id, String name, MarketStatus status) {}

public class Market {
  private final Long id;
  private final String name;
  // getterのみ、setterなし
}
```

## Optionalの使い方

```java
// ✅ find*メソッドからOptionalを返す
Optional<Market> market = marketRepository.findBySlug(slug);

// ✅ get()の代わりにmap/flatMapを使用
return market
    .map(MarketResponse::from)
    .orElseThrow(() -> new EntityNotFoundException("Market not found"));
```

## ストリームのベストプラクティス

```java
// ✅ 変換にストリームを使用し、パイプラインは短く保つ
List<String> names = markets.stream()
    .map(Market::name)
    .filter(Objects::nonNull)
    .toList();

// ❌ 複雑なネストされたストリームは避ける。明確さのためループを優先
```

## 例外処理

- ドメインエラーには非チェック例外を使用。技術的例外はコンテキスト付きでラップ
- ドメイン固有の例外を作成（例: `MarketNotFoundException`）
- 再スローやロギングの集中処理でない限り、広範な`catch (Exception ex)`を避ける

```java
throw new MarketNotFoundException(slug);
```

## ジェネリクスと型安全性

- raw型を避ける。ジェネリックパラメータを宣言
- 再利用可能なユーティリティには境界付きジェネリクスを優先

```java
public <T extends Identifiable> Map<Long, T> indexById(Collection<T> items) { ... }
```

## プロジェクト構成（Maven/Gradle）

```
src/main/java/com/example/app/
  config/
  controller/
  service/
  repository/
  domain/
  dto/
  util/
src/main/resources/
  application.yml
src/test/java/... (mainを反映)
```

## フォーマットとスタイル

- 2スペースまたは4スペースを一貫して使用（プロジェクト標準に従う）
- ファイルごとに1つのpublicトップレベル型
- メソッドは短く焦点を絞る。ヘルパーを抽出
- メンバーの順序：定数、フィールド、コンストラクタ、publicメソッド、protected、private

## 避けるべきコードスメル

- 長いパラメータリスト → DTO/ビルダーを使用
- 深いネスト → 早期リターン
- マジックナンバー → 名前付き定数
- static ミュータブル状態 → 依存性注入を優先
- サイレントcatchブロック → ログを記録して対処、または再スロー

## ロギング

```java
private static final Logger log = LoggerFactory.getLogger(MarketService.class);
log.info("fetch_market slug={}", slug);
log.error("failed_fetch_market slug={}", slug, ex);
```

## Null処理

- 不可避な場合のみ`@Nullable`を受け入れる。それ以外は`@NonNull`を使用
- 入力にはBean Validation（`@NotNull`、`@NotBlank`）を使用

## テストの期待事項

- JUnit 5 + AssertJで流暢なアサーション
- モックにはMockito。可能な限りパーシャルモックを避ける
- 決定論的なテストを優先。隠れたsleepを使用しない

**留意事項**: コードは意図的で、型付けされ、観測可能であるべき。必要性が証明されない限り、マイクロ最適化より保守性を優先。
