---
name: android-clean-architecture
description: AndroidおよびKotlin Multiplatformプロジェクトのためのクリーンアーキテクチャパターン — モジュール構成、依存関係ルール、UseCase、Repository、データレイヤーパターン
origin: ECC
---

# Android クリーンアーキテクチャ

AndroidおよびKMPプロジェクトのためのクリーンアーキテクチャパターン。モジュール境界、依存性逆転、UseCase/Repositoryパターン、Room・SQLDelight・Ktorを使用したデータレイヤー設計をカバーします。

## 起動条件

- AndroidまたはKMPプロジェクトのモジュール構成を設計する場合
- UseCase、Repository、DataSourceを実装する場合
- レイヤー間（domain、data、presentation）のデータフローを設計する場合
- KoinまたはHiltで依存性注入をセットアップする場合
- レイヤードアーキテクチャでRoom、SQLDelight、Ktorを使用する場合

## モジュール構成

### 推奨レイアウト

```
project/
├── app/                  # Android entry point, DI wiring, Application class
├── core/                 # Shared utilities, base classes, error types
├── domain/               # UseCases, domain models, repository interfaces (pure Kotlin)
├── data/                 # Repository implementations, DataSources, DB, network
├── presentation/         # Screens, ViewModels, UI models, navigation
├── design-system/        # Reusable Compose components, theme, typography
└── feature/              # Feature modules (optional, for larger projects)
    ├── auth/
    ├── settings/
    └── profile/
```

### 依存関係ルール

```
app → presentation, domain, data, core
presentation → domain, design-system, core
data → domain, core
domain → core (or no dependencies)
core → (nothing)
```

**重要**: `domain`は`data`、`presentation`、またはいかなるフレームワークにも依存してはいけません。純粋なKotlinのみを含みます。

## ドメインレイヤー

### UseCaseパターン

各UseCaseは1つのビジネスオペレーションを表します。クリーンな呼び出しサイトのために`operator fun invoke`を使用します：

```kotlin
class GetItemsByCategoryUseCase(
    private val repository: ItemRepository
) {
    suspend operator fun invoke(category: String): Result<List<Item>> {
        return repository.getItemsByCategory(category)
    }
}

// Flow-based UseCase for reactive streams
class ObserveUserProgressUseCase(
    private val repository: UserRepository
) {
    operator fun invoke(userId: String): Flow<UserProgress> {
        return repository.observeProgress(userId)
    }
}
```

### ドメインモデル

ドメインモデルはプレーンなKotlinデータクラスです — フレームワークアノテーションなし：

```kotlin
data class Item(
    val id: String,
    val title: String,
    val description: String,
    val tags: List<String>,
    val status: Status,
    val category: String
)

enum class Status { DRAFT, ACTIVE, ARCHIVED }
```

### リポジトリインターフェース

domainで定義し、dataで実装：

```kotlin
interface ItemRepository {
    suspend fun getItemsByCategory(category: String): Result<List<Item>>
    suspend fun saveItem(item: Item): Result<Unit>
    fun observeItems(): Flow<List<Item>>
}
```

## データレイヤー

### リポジトリ実装

ローカルとリモートのデータソース間を調整します：

```kotlin
class ItemRepositoryImpl(
    private val localDataSource: ItemLocalDataSource,
    private val remoteDataSource: ItemRemoteDataSource
) : ItemRepository {

    override suspend fun getItemsByCategory(category: String): Result<List<Item>> {
        return runCatching {
            val remote = remoteDataSource.fetchItems(category)
            localDataSource.insertItems(remote.map { it.toEntity() })
            localDataSource.getItemsByCategory(category).map { it.toDomain() }
        }
    }

    override suspend fun saveItem(item: Item): Result<Unit> {
        return runCatching {
            localDataSource.insertItems(listOf(item.toEntity()))
        }
    }

    override fun observeItems(): Flow<List<Item>> {
        return localDataSource.observeAll().map { entities ->
            entities.map { it.toDomain() }
        }
    }
}
```

### マッパーパターン

マッパーはデータモデルの近くに拡張関数として配置します：

```kotlin
// In data layer
fun ItemEntity.toDomain() = Item(
    id = id,
    title = title,
    description = description,
    tags = tags.split("|"),
    status = Status.valueOf(status),
    category = category
)

fun ItemDto.toEntity() = ItemEntity(
    id = id,
    title = title,
    description = description,
    tags = tags.joinToString("|"),
    status = status,
    category = category
)
```

### Roomデータベース（Android）

```kotlin
@Entity(tableName = "items")
data class ItemEntity(
    @PrimaryKey val id: String,
    val title: String,
    val description: String,
    val tags: String,
    val status: String,
    val category: String
)

@Dao
interface ItemDao {
    @Query("SELECT * FROM items WHERE category = :category")
    suspend fun getByCategory(category: String): List<ItemEntity>

    @Upsert
    suspend fun upsert(items: List<ItemEntity>)

    @Query("SELECT * FROM items")
    fun observeAll(): Flow<List<ItemEntity>>
}
```

### SQLDelight（KMP）

```sql
-- Item.sq
CREATE TABLE ItemEntity (
    id TEXT NOT NULL PRIMARY KEY,
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    tags TEXT NOT NULL,
    status TEXT NOT NULL,
    category TEXT NOT NULL
);

getByCategory:
SELECT * FROM ItemEntity WHERE category = ?;

upsert:
INSERT OR REPLACE INTO ItemEntity (id, title, description, tags, status, category)
VALUES (?, ?, ?, ?, ?, ?);

observeAll:
SELECT * FROM ItemEntity;
```

### Ktorネットワーククライアント（KMP）

```kotlin
class ItemRemoteDataSource(private val client: HttpClient) {

    suspend fun fetchItems(category: String): List<ItemDto> {
        return client.get("api/items") {
            parameter("category", category)
        }.body()
    }
}

// HttpClient setup with content negotiation
val httpClient = HttpClient {
    install(ContentNegotiation) { json(Json { ignoreUnknownKeys = true }) }
    install(Logging) { level = LogLevel.HEADERS }
    defaultRequest { url("https://api.example.com/") }
}
```

## 依存性注入

### Koin（KMP対応）

```kotlin
// Domain module
val domainModule = module {
    factory { GetItemsByCategoryUseCase(get()) }
    factory { ObserveUserProgressUseCase(get()) }
}

// Data module
val dataModule = module {
    single<ItemRepository> { ItemRepositoryImpl(get(), get()) }
    single { ItemLocalDataSource(get()) }
    single { ItemRemoteDataSource(get()) }
}

// Presentation module
val presentationModule = module {
    viewModelOf(::ItemListViewModel)
    viewModelOf(::DashboardViewModel)
}
```

### Hilt（Android専用）

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    abstract fun bindItemRepository(impl: ItemRepositoryImpl): ItemRepository
}

@HiltViewModel
class ItemListViewModel @Inject constructor(
    private val getItems: GetItemsByCategoryUseCase
) : ViewModel()
```

## エラーハンドリング

### Result/Tryパターン

エラー伝播には`Result<T>`またはカスタムsealedタイプを使用します：

```kotlin
sealed interface Try<out T> {
    data class Success<T>(val value: T) : Try<T>
    data class Failure(val error: AppError) : Try<Nothing>
}

sealed interface AppError {
    data class Network(val message: String) : AppError
    data class Database(val message: String) : AppError
    data object Unauthorized : AppError
}

// In ViewModel — map to UI state
viewModelScope.launch {
    when (val result = getItems(category)) {
        is Try.Success -> _state.update { it.copy(items = result.value, isLoading = false) }
        is Try.Failure -> _state.update { it.copy(error = result.error.toMessage(), isLoading = false) }
    }
}
```

## コンベンションプラグイン（Gradle）

KMPプロジェクトでは、ビルドファイルの重複を減らすためにコンベンションプラグインを使用します：

```kotlin
// build-logic/src/main/kotlin/kmp-library.gradle.kts
plugins {
    id("org.jetbrains.kotlin.multiplatform")
}

kotlin {
    androidTarget()
    iosX64(); iosArm64(); iosSimulatorArm64()
    sourceSets {
        commonMain.dependencies { /* shared deps */ }
        commonTest.dependencies { implementation(kotlin("test")) }
    }
}
```

モジュールで適用：

```kotlin
// domain/build.gradle.kts
plugins { id("kmp-library") }
```

## 避けるべきアンチパターン

- `domain`でAndroidフレームワーククラスをインポートする — 純粋なKotlinに保つ
- データベースエンティティやDTOをUIレイヤーに公開する — 常にドメインモデルにマッピングする
- ViewModelにビジネスロジックを置く — UseCaseに抽出する
- `GlobalScope`や非構造化コルーチンを使用する — `viewModelScope`または構造化された並行処理を使用する
- 肥大化したリポジトリ実装 — 焦点を絞ったDataSourceに分割する
- 循環モジュール依存 — AがBに依存するなら、BはAに依存してはならない

## 参考

UIパターンについてはスキル`compose-multiplatform-patterns`を参照。
非同期パターンについてはスキル`kotlin-coroutines-flows`を参照。
