---
paths:
  - "**/*.kt"
  - "**/*.kts"
---
# Kotlin テスト

> このファイルは [common/testing.md](../common/testing.md) を Kotlin および Android/KMP 固有の内容で拡張します。

## テストフレームワーク

- マルチプラットフォーム（KMP）には **kotlin.test** — `@Test`、`assertEquals`、`assertTrue`
- Android 固有のテストには **JUnit 4/5**
- Flow と StateFlow のテストには **Turbine**
- コルーチンテストには **kotlinx-coroutines-test**（`runTest`、`TestDispatcher`）

## Turbine による ViewModel テスト

```kotlin
@Test
fun `loading state emitted then data`() = runTest {
    val repo = FakeItemRepository()
    repo.addItem(testItem)
    val viewModel = ItemListViewModel(GetItemsUseCase(repo))

    viewModel.state.test {
        assertEquals(ItemListState(), awaitItem())     // initial state
        viewModel.onEvent(ItemListEvent.Load)
        assertTrue(awaitItem().isLoading)               // loading
        assertEquals(listOf(testItem), awaitItem().items) // loaded
    }
}
```

## モックよりフェイクを優先

モッキングフレームワークよりも手書きのフェイクを優先：

```kotlin
class FakeItemRepository : ItemRepository {
    private val items = mutableListOf<Item>()
    var fetchError: Throwable? = null

    override suspend fun getAll(): Result<List<Item>> {
        fetchError?.let { return Result.failure(it) }
        return Result.success(items.toList())
    }

    override fun observeAll(): Flow<List<Item>> = flowOf(items.toList())

    fun addItem(item: Item) { items.add(item) }
}
```

## コルーチンテスト

```kotlin
@Test
fun `parallel operations complete`() = runTest {
    val repo = FakeRepository()
    val result = loadDashboard(repo)
    advanceUntilIdle()
    assertNotNull(result.items)
    assertNotNull(result.stats)
}
```

`runTest` を使用 — 仮想時間を自動的に進め、`TestScope` を提供します。

## Ktor MockEngine

```kotlin
val mockEngine = MockEngine { request ->
    when (request.url.encodedPath) {
        "/api/items" -> respond(
            content = Json.encodeToString(testItems),
            headers = headersOf(HttpHeaders.ContentType, ContentType.Application.Json.toString())
        )
        else -> respondError(HttpStatusCode.NotFound)
    }
}

val client = HttpClient(mockEngine) {
    install(ContentNegotiation) { json() }
}
```

## Room/SQLDelight テスト

- Room: インメモリテストには `Room.inMemoryDatabaseBuilder()` を使用
- SQLDelight: JVM テストには `JdbcSqliteDriver(JdbcSqliteDriver.IN_MEMORY)` を使用

```kotlin
@Test
fun `insert and query items`() = runTest {
    val driver = JdbcSqliteDriver(JdbcSqliteDriver.IN_MEMORY)
    Database.Schema.create(driver)
    val db = Database(driver)

    db.itemQueries.insert("1", "Sample Item", "description")
    val items = db.itemQueries.getAll().executeAsList()
    assertEquals(1, items.size)
}
```

## テスト命名

バッククォートで囲んだ説明的な名前を使用：

```kotlin
@Test
fun `search with empty query returns all items`() = runTest { }

@Test
fun `delete item emits updated list without deleted item`() = runTest { }
```

## テスト構成

```
src/
├── commonTest/kotlin/     # 共有テスト（ViewModel、UseCase、Repository）
├── androidUnitTest/kotlin/ # Android ユニットテスト（JUnit）
├── androidInstrumentedTest/kotlin/  # インストルメンテッドテスト（Room、UI）
└── iosTest/kotlin/        # iOS 固有テスト
```

最低限のテストカバレッジ: 各フィーチャーの ViewModel + UseCase。
