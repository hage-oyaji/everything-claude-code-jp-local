---
name: jpa-patterns
description: Spring BootにおけるJPA/Hibernateパターン — エンティティ設計、リレーションシップ、クエリ最適化、トランザクション、監査、インデックス、ページネーション、コネクションプーリング。
origin: ECC
---

# JPA/Hibernateパターン

Spring Bootにおけるデータモデリング、リポジトリ、パフォーマンスチューニングに使用。

## アクティベーション条件

- JPAエンティティとテーブルマッピングの設計
- リレーションシップの定義（@OneToMany、@ManyToOne、@ManyToMany）
- クエリの最適化（N+1防止、フェッチ戦略、プロジェクション）
- トランザクション、監査、論理削除の設定
- ページネーション、ソート、カスタムリポジトリメソッドのセットアップ
- コネクションプーリング（HikariCP）や二次キャッシュのチューニング

## エンティティ設計

```java
@Entity
@Table(name = "markets", indexes = {
  @Index(name = "idx_markets_slug", columnList = "slug", unique = true)
})
@EntityListeners(AuditingEntityListener.class)
public class MarketEntity {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, length = 200)
  private String name;

  @Column(nullable = false, unique = true, length = 120)
  private String slug;

  @Enumerated(EnumType.STRING)
  private MarketStatus status = MarketStatus.ACTIVE;

  @CreatedDate private Instant createdAt;
  @LastModifiedDate private Instant updatedAt;
}
```

監査の有効化：
```java
@Configuration
@EnableJpaAuditing
class JpaConfig {}
```

## リレーションシップとN+1防止

```java
@OneToMany(mappedBy = "market", cascade = CascadeType.ALL, orphanRemoval = true)
private List<PositionEntity> positions = new ArrayList<>();
```

- デフォルトで遅延ロード。必要な場合はクエリで`JOIN FETCH`を使用
- コレクションでの`EAGER`は避ける。読み取りパスにはDTOプロジェクションを使用

```java
@Query("select m from MarketEntity m left join fetch m.positions where m.id = :id")
Optional<MarketEntity> findWithPositions(@Param("id") Long id);
```

## リポジトリパターン

```java
public interface MarketRepository extends JpaRepository<MarketEntity, Long> {
  Optional<MarketEntity> findBySlug(String slug);

  @Query("select m from MarketEntity m where m.status = :status")
  Page<MarketEntity> findByStatus(@Param("status") MarketStatus status, Pageable pageable);
}
```

- 軽量クエリにはプロジェクションを使用：
```java
public interface MarketSummary {
  Long getId();
  String getName();
  MarketStatus getStatus();
}
Page<MarketSummary> findAllBy(Pageable pageable);
```

## トランザクション

- サービスメソッドに`@Transactional`をアノテーション
- 読み取りパスの最適化に`@Transactional(readOnly = true)`を使用
- プロパゲーションを慎重に選択。長時間実行トランザクションを避ける

```java
@Transactional
public Market updateStatus(Long id, MarketStatus status) {
  MarketEntity entity = repo.findById(id)
      .orElseThrow(() -> new EntityNotFoundException("Market"));
  entity.setStatus(status);
  return Market.from(entity);
}
```

## ページネーション

```java
PageRequest page = PageRequest.of(pageNumber, pageSize, Sort.by("createdAt").descending());
Page<MarketEntity> markets = repo.findByStatus(MarketStatus.ACTIVE, page);
```

カーソルベースのページネーションには、JPQLに`id > :lastId`を含めてソートと組み合わせる。

## インデックスとパフォーマンス

- 頻繁なフィルターにインデックスを追加（`status`、`slug`、外部キー）
- クエリパターンに合わせた複合インデックスを使用（`status, created_at`）
- `select *`を避ける。必要なカラムのみプロジェクション
- `saveAll`と`hibernate.jdbc.batch_size`でバッチ書き込み

## コネクションプーリング（HikariCP）

推奨プロパティ：
```
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.validation-timeout=5000
```

PostgreSQLのLOB処理には以下を追加：
```
spring.jpa.properties.hibernate.jdbc.lob.non_contextual_creation=true
```

## キャッシング

- 一次キャッシュはEntityManagerごと。トランザクションをまたいでエンティティを保持しない
- 読み取り頻度の高いエンティティには二次キャッシュを慎重に検討。エビクション戦略を検証

## マイグレーション

- FlywayまたはLiquibaseを使用。本番でHibernateの自動DDLに依存しない
- マイグレーションはべき等で追加的に保つ。計画なしにカラムを削除しない

## データアクセスのテスト

- 本番環境を反映するため`@DataJpaTest`とTestcontainersの使用を推奨
- ログでSQLの効率をアサート：`logging.level.org.hibernate.SQL=DEBUG`と`logging.level.org.hibernate.orm.jdbc.bind=TRACE`でパラメータ値を確認

**留意事項**: エンティティはスリムに、クエリは意図的に、トランザクションは短く。フェッチ戦略とプロジェクションでN+1を防止し、読み取り/書き込みパスに合わせてインデックスを設定。
