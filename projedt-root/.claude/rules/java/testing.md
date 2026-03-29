---
paths:
  - "**/*.java"
---
# Java テスト

> このファイルは [common/testing.md](../common/testing.md) をJava固有の内容で拡張します。

## テストフレームワーク

- **JUnit 5**（`@Test`、`@ParameterizedTest`、`@Nested`、`@DisplayName`）
- 流暢なアサーションには **AssertJ**（`assertThat(result).isEqualTo(expected)`）
- 依存関係のモックには **Mockito**
- データベースやサービスが必要な統合テストには **Testcontainers**

## テストの構成

```
src/test/java/com/example/app/
  service/           # サービス層のユニットテスト
  controller/        # Web層 / APIテスト
  repository/        # データアクセステスト
  integration/       # レイヤー横断の統合テスト
```

`src/test/java` で `src/main/java` のパッケージ構造をミラーリングする。

## ユニットテストパターン

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    private OrderService orderService;

    @BeforeEach
    void setUp() {
        orderService = new OrderService(orderRepository);
    }

    @Test
    @DisplayName("findById returns order when exists")
    void findById_existingOrder_returnsOrder() {
        var order = new Order(1L, "Alice", BigDecimal.TEN);
        when(orderRepository.findById(1L)).thenReturn(Optional.of(order));

        var result = orderService.findById(1L);

        assertThat(result.customerName()).isEqualTo("Alice");
        verify(orderRepository).findById(1L);
    }

    @Test
    @DisplayName("findById throws when order not found")
    void findById_missingOrder_throws() {
        when(orderRepository.findById(99L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> orderService.findById(99L))
            .isInstanceOf(OrderNotFoundException.class)
            .hasMessageContaining("99");
    }
}
```

## パラメータ化テスト

```java
@ParameterizedTest
@CsvSource({
    "100.00, 10, 90.00",
    "50.00, 0, 50.00",
    "200.00, 25, 150.00"
})
@DisplayName("discount applied correctly")
void applyDiscount(BigDecimal price, int pct, BigDecimal expected) {
    assertThat(PricingUtils.discount(price, pct)).isEqualByComparingTo(expected);
}
```

## 統合テスト

実際のデータベース統合にはTestcontainersを使用する:

```java
@Testcontainers
class OrderRepositoryIT {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    private OrderRepository repository;

    @BeforeEach
    void setUp() {
        var dataSource = new PGSimpleDataSource();
        dataSource.setUrl(postgres.getJdbcUrl());
        dataSource.setUser(postgres.getUsername());
        dataSource.setPassword(postgres.getPassword());
        repository = new JdbcOrderRepository(dataSource);
    }

    @Test
    void save_and_findById() {
        var saved = repository.save(new Order(null, "Bob", BigDecimal.ONE));
        var found = repository.findById(saved.getId());
        assertThat(found).isPresent();
    }
}
```

Spring Bootの統合テストについては、スキル: `springboot-tdd` を参照してください。

## テスト命名

`@DisplayName` を使った説明的な名前を付ける:
- メソッド名: `methodName_scenario_expectedBehavior()`
- レポート用: `@DisplayName("human-readable description")`

## カバレッジ

- 行カバレッジ80%以上を目標にする
- カバレッジレポートには JaCoCo を使用する
- サービスとドメインロジックに集中する — トリビアルなgetter/設定クラスは省略可

## 参考

スキル: `springboot-tdd` でMockMvcとTestcontainersを使用したSpring Boot TDDパターンを参照してください。
スキル: `java-coding-standards` でテストに関する期待事項を参照してください。
