---
name: kotlin-coroutines-flows
description: AndroidおよびKMPのためのKotlinコルーチンとFlowパターン — 構造化された並行処理、Flowオペレーター、StateFlow、エラーハンドリング、テスト。
origin: ECC
---

# Kotlinコルーチン & Flow

AndroidおよびKotlin Multiplatformプロジェクトにおける構造化された並行処理、Flowベースのリアクティブストリーム、コルーチンテストのパターン。

## アクティベーション条件

- Kotlinコルーチンによる非同期コードの作成
- Flow、StateFlow、SharedFlowによるリアクティブデータの使用
- 並行処理の実装（並列ロード、デバウンス、リトライ）
- コルーチンとFlowのテスト
- コルーチンスコープとキャンセルの管理

## 構造化された並行処理

### スコープ階層

```
Application
  └── viewModelScope (ViewModel)
        └── coroutineScope { } (構造化された子)
              ├── async { } (並行タスク)
              └── async { } (並行タスク)
```

常に構造化された並行処理を使用 — 決して`GlobalScope`を使わない：

```kotlin
// BAD
GlobalScope.launch { fetchData() }

// GOOD — ViewModelのライフサイクルにスコープ
viewModelScope.launch { fetchData() }

// GOOD — Composableのライフサイクルにスコープ
LaunchedEffect(key) { fetchData() }
```

### 並列分解

並列処理には`coroutineScope` + `async`を使用：

```kotlin
suspend fun loadDashboard(): Dashboard = coroutineScope {
    val items = async { itemRepository.getRecent() }
    val stats = async { statsRepository.getToday() }
    val profile = async { userRepository.getCurrent() }
    Dashboard(
        items = items.await(),
        stats = stats.await(),
        profile = profile.await()
    )
}
```

### SupervisorScope

子の失敗が兄弟をキャンセルしてはいけない場合に`supervisorScope`を使用：

```kotlin
suspend fun syncAll() = supervisorScope {
    launch { syncItems() }       // ここでの失敗はsyncStatsをキャンセルしない
    launch { syncStats() }
    launch { syncSettings() }
}
```

## Flowパターン

### Cold Flow — ワンショットからストリームへの変換

```kotlin
fun observeItems(): Flow<List<Item>> = flow {
    // データベースが変更されるたびに再発行
    itemDao.observeAll()
        .map { entities -> entities.map { it.toDomain() } }
        .collect { emit(it) }
}
```

### UI状態のためのStateFlow

```kotlin
class DashboardViewModel(
    observeProgress: ObserveUserProgressUseCase
) : ViewModel() {
    val progress: StateFlow<UserProgress> = observeProgress()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = UserProgress.EMPTY
        )
}
```

`WhileSubscribed(5_000)`は最後のサブスクライバーが離れた後5秒間アップストリームをアクティブに保つ — コンフィグレーション変更時に再起動なしで生き残る。

### 複数のFlowの結合

```kotlin
val uiState: StateFlow<HomeState> = combine(
    itemRepository.observeItems(),
    settingsRepository.observeTheme(),
    userRepository.observeProfile()
) { items, theme, profile ->
    HomeState(items = items, theme = theme, profile = profile)
}.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), HomeState())
```

### Flowオペレーター

```kotlin
// 検索入力のデバウンス
searchQuery
    .debounce(300)
    .distinctUntilChanged()
    .flatMapLatest { query -> repository.search(query) }
    .catch { emit(emptyList()) }
    .collect { results -> _state.update { it.copy(results = results) } }

// 指数バックオフ付きリトライ
fun fetchWithRetry(): Flow<Data> = flow { emit(api.fetch()) }
    .retryWhen { cause, attempt ->
        if (cause is IOException && attempt < 3) {
            delay(1000L * (1 shl attempt.toInt()))
            true
        } else {
            false
        }
    }
```

### ワンタイムイベント用SharedFlow

```kotlin
class ItemListViewModel : ViewModel() {
    private val _effects = MutableSharedFlow<Effect>()
    val effects: SharedFlow<Effect> = _effects.asSharedFlow()

    sealed interface Effect {
        data class ShowSnackbar(val message: String) : Effect
        data class NavigateTo(val route: String) : Effect
    }

    private fun deleteItem(id: String) {
        viewModelScope.launch {
            repository.delete(id)
            _effects.emit(Effect.ShowSnackbar("Item deleted"))
        }
    }
}

// Composableで収集
LaunchedEffect(Unit) {
    viewModel.effects.collect { effect ->
        when (effect) {
            is Effect.ShowSnackbar -> snackbarHostState.showSnackbar(effect.message)
            is Effect.NavigateTo -> navController.navigate(effect.route)
        }
    }
}
```

## ディスパッチャー

```kotlin
// CPU集約的な処理
withContext(Dispatchers.Default) { parseJson(largePayload) }

// IO バウンドな処理
withContext(Dispatchers.IO) { database.query() }

// メインスレッド（UI） — viewModelScopeのデフォルト
withContext(Dispatchers.Main) { updateUi() }
```

KMPでは`Dispatchers.Default`と`Dispatchers.Main`を使用（全プラットフォームで利用可能）。`Dispatchers.IO`はJVM/Androidのみ — 他のプラットフォームでは`Dispatchers.Default`を使用するか、DIで提供。

## キャンセル

### 協調的キャンセル

長時間ループはキャンセルをチェックする必要がある：

```kotlin
suspend fun processItems(items: List<Item>) = coroutineScope {
    for (item in items) {
        ensureActive()  // キャンセルされた場合CancellationExceptionをスロー
        process(item)
    }
}
```

### try/finallyによるクリーンアップ

```kotlin
viewModelScope.launch {
    try {
        _state.update { it.copy(isLoading = true) }
        val data = repository.fetch()
        _state.update { it.copy(data = data) }
    } finally {
        _state.update { it.copy(isLoading = false) }  // キャンセル時も常に実行
    }
}
```

## テスト

### TurbineによるStateFlowテスト

```kotlin
@Test
fun `search updates item list`() = runTest {
    val fakeRepository = FakeItemRepository().apply { emit(testItems) }
    val viewModel = ItemListViewModel(GetItemsUseCase(fakeRepository))

    viewModel.state.test {
        assertEquals(ItemListState(), awaitItem())  // 初期値

        viewModel.onSearch("query")
        val loading = awaitItem()
        assertTrue(loading.isLoading)

        val loaded = awaitItem()
        assertFalse(loaded.isLoading)
        assertEquals(1, loaded.items.size)
    }
}
```

### TestDispatcherによるテスト

```kotlin
@Test
fun `parallel load completes correctly`() = runTest {
    val viewModel = DashboardViewModel(
        itemRepo = FakeItemRepo(),
        statsRepo = FakeStatsRepo()
    )

    viewModel.load()
    advanceUntilIdle()

    val state = viewModel.state.value
    assertNotNull(state.items)
    assertNotNull(state.stats)
}
```

### Flowのフェイク化

```kotlin
class FakeItemRepository : ItemRepository {
    private val _items = MutableStateFlow<List<Item>>(emptyList())

    override fun observeItems(): Flow<List<Item>> = _items

    fun emit(items: List<Item>) { _items.value = items }

    override suspend fun getItemsByCategory(category: String): Result<List<Item>> {
        return Result.success(_items.value.filter { it.category == category })
    }
}
```

## 避けるべきアンチパターン

- `GlobalScope`の使用 — コルーチンがリークし、構造化されたキャンセルがない
- スコープなしの`init {}`でのFlow収集 — `viewModelScope.launch`を使用
- ミュータブルコレクションでの`MutableStateFlow`使用 — 常にイミュータブルコピーを使用：`_state.update { it.copy(list = it.list + newItem) }`
- `CancellationException`のキャッチ — 適切なキャンセルのために伝播させる
- `flowOn(Dispatchers.Main)`での収集 — 収集ディスパッチャーは呼び出し元のディスパッチャー
- `@Composable`内で`remember`なしの`Flow`作成 — 再コンポジションのたびにFlowが再作成される

## 参考

Flowの UI での利用については、スキル `compose-multiplatform-patterns` を参照。
コルーチンがレイヤーのどこに位置するかは、スキル `android-clean-architecture` を参照。
