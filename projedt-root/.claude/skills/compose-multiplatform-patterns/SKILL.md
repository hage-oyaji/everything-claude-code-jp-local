---
name: compose-multiplatform-patterns
description: KMPプロジェクトのためのCompose MultiplatformおよびJetpack Composeパターン — ステート管理、ナビゲーション、テーマ設定、パフォーマンス、プラットフォーム固有UI。
origin: ECC
---

# Compose Multiplatformパターン

Compose MultiplatformとJetpack Composeを使用して、Android、iOS、デスクトップ、Web間で共有UIを構築するためのパターン。ステート管理、ナビゲーション、テーマ設定、パフォーマンスを網羅。

## 発動条件

- Compose UI（Jetpack ComposeまたはCompose Multiplatform）を構築するとき
- ViewModelとComposeステートでUIステートを管理するとき
- KMPまたはAndroidプロジェクトでナビゲーションを実装するとき
- 再利用可能なComposableとデザインシステムを設計するとき
- リコンポジションとレンダリングパフォーマンスを最適化するとき

## ステート管理

### ViewModel + 単一ステートオブジェクト

画面ステートには単一のデータクラスを使用する。`StateFlow`として公開し、Composeで収集する:

```kotlin
data class ItemListState(
    val items: List<Item> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null,
    val searchQuery: String = ""
)

class ItemListViewModel(
    private val getItems: GetItemsUseCase
) : ViewModel() {
    private val _state = MutableStateFlow(ItemListState())
    val state: StateFlow<ItemListState> = _state.asStateFlow()

    fun onSearch(query: String) {
        _state.update { it.copy(searchQuery = query) }
        loadItems(query)
    }

    private fun loadItems(query: String) {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            getItems(query).fold(
                onSuccess = { items -> _state.update { it.copy(items = items, isLoading = false) } },
                onFailure = { e -> _state.update { it.copy(error = e.message, isLoading = false) } }
            )
        }
    }
}
```

### Composeでのステート収集

```kotlin
@Composable
fun ItemListScreen(viewModel: ItemListViewModel = koinViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    ItemListContent(
        state = state,
        onSearch = viewModel::onSearch
    )
}

@Composable
private fun ItemListContent(
    state: ItemListState,
    onSearch: (String) -> Unit
) {
    // Stateless composable — easy to preview and test
}
```

### イベントシンクパターン

複雑な画面では、複数のコールバックラムダの代わりにイベント用のsealed interfaceを使用する:

```kotlin
sealed interface ItemListEvent {
    data class Search(val query: String) : ItemListEvent
    data class Delete(val itemId: String) : ItemListEvent
    data object Refresh : ItemListEvent
}

// In ViewModel
fun onEvent(event: ItemListEvent) {
    when (event) {
        is ItemListEvent.Search -> onSearch(event.query)
        is ItemListEvent.Delete -> deleteItem(event.itemId)
        is ItemListEvent.Refresh -> loadItems(_state.value.searchQuery)
    }
}

// In Composable — single lambda instead of many
ItemListContent(
    state = state,
    onEvent = viewModel::onEvent
)
```

## ナビゲーション

### 型安全ナビゲーション（Compose Navigation 2.8+）

ルートを`@Serializable`オブジェクトとして定義する:

```kotlin
@Serializable data object HomeRoute
@Serializable data class DetailRoute(val id: String)
@Serializable data object SettingsRoute

@Composable
fun AppNavHost(navController: NavHostController = rememberNavController()) {
    NavHost(navController, startDestination = HomeRoute) {
        composable<HomeRoute> {
            HomeScreen(onNavigateToDetail = { id -> navController.navigate(DetailRoute(id)) })
        }
        composable<DetailRoute> { backStackEntry ->
            val route = backStackEntry.toRoute<DetailRoute>()
            DetailScreen(id = route.id)
        }
        composable<SettingsRoute> { SettingsScreen() }
    }
}
```

### ダイアログとボトムシートナビゲーション

命令的なshow/hideの代わりに`dialog()`とオーバーレイパターンを使用する:

```kotlin
NavHost(navController, startDestination = HomeRoute) {
    composable<HomeRoute> { /* ... */ }
    dialog<ConfirmDeleteRoute> { backStackEntry ->
        val route = backStackEntry.toRoute<ConfirmDeleteRoute>()
        ConfirmDeleteDialog(
            itemId = route.itemId,
            onConfirm = { navController.popBackStack() },
            onDismiss = { navController.popBackStack() }
        )
    }
}
```

## Composable設計

### スロットベースAPI

柔軟性のためにスロットパラメータを持つComposableを設計する:

```kotlin
@Composable
fun AppCard(
    modifier: Modifier = Modifier,
    header: @Composable () -> Unit = {},
    content: @Composable ColumnScope.() -> Unit,
    actions: @Composable RowScope.() -> Unit = {}
) {
    Card(modifier = modifier) {
        Column {
            header()
            Column(content = content)
            Row(horizontalArrangement = Arrangement.End, content = actions)
        }
    }
}
```

### Modifierの順序

Modifierの順序は重要 — 以下の順序で適用する:

```kotlin
Text(
    text = "Hello",
    modifier = Modifier
        .padding(16.dp)          // 1. Layout (padding, size)
        .clip(RoundedCornerShape(8.dp))  // 2. Shape
        .background(Color.White) // 3. Drawing (background, border)
        .clickable { }           // 4. Interaction
)
```

## KMPプラットフォーム固有UI

### expect/actualによるプラットフォームComposable

```kotlin
// commonMain
@Composable
expect fun PlatformStatusBar(darkIcons: Boolean)

// androidMain
@Composable
actual fun PlatformStatusBar(darkIcons: Boolean) {
    val systemUiController = rememberSystemUiController()
    SideEffect { systemUiController.setStatusBarColor(Color.Transparent, darkIcons) }
}

// iosMain
@Composable
actual fun PlatformStatusBar(darkIcons: Boolean) {
    // iOS handles this via UIKit interop or Info.plist
}
```

## パフォーマンス

### スキップ可能なリコンポジションのための安定型

すべてのプロパティが安定している場合、クラスに`@Stable`または`@Immutable`をマークする:

```kotlin
@Immutable
data class ItemUiModel(
    val id: String,
    val title: String,
    val description: String,
    val progress: Float
)
```

### `key()`とLazyリストの正しい使用

```kotlin
LazyColumn {
    items(
        items = items,
        key = { it.id }  // Stable keys enable item reuse and animations
    ) { item ->
        ItemRow(item = item)
    }
}
```

### `derivedStateOf`による読み取りの遅延

```kotlin
val listState = rememberLazyListState()
val showScrollToTop by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 5 }
}
```

### リコンポジションでのアロケーション回避

```kotlin
// BAD — new lambda and list every recomposition
items.filter { it.isActive }.forEach { ActiveItem(it, onClick = { handle(it) }) }

// GOOD — key each item so callbacks stay attached to the right row
val activeItems = remember(items) { items.filter { it.isActive } }
activeItems.forEach { item ->
    key(item.id) {
        ActiveItem(item, onClick = { handle(item) })
    }
}
```

## テーマ設定

### Material 3ダイナミックテーマ

```kotlin
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            if (darkTheme) dynamicDarkColorScheme(LocalContext.current)
            else dynamicLightColorScheme(LocalContext.current)
        }
        darkTheme -> darkColorScheme()
        else -> lightColorScheme()
    }

    MaterialTheme(colorScheme = colorScheme, content = content)
}
```

## 避けるべきアンチパターン

- `MutableStateFlow`と`collectAsStateWithLifecycle`の方がライフサイクル的に安全なのに、ViewModelで`mutableStateOf`を使用すること
- `NavController`をComposableの深い階層に渡すこと — 代わりにラムダコールバックを渡す
- `@Composable`関数内での重い計算 — ViewModelまたは`remember {}`に移動する
- ViewModel initの代替として`LaunchedEffect(Unit)`を使用すること — 一部のセットアップではConfiguration変更時に再実行される
- Composableパラメータで新しいオブジェクトインスタンスを作成すること — 不要なリコンポジションを引き起こす

## 参考

スキル`android-clean-architecture`でモジュール構造とレイヤリングを参照。
スキル`kotlin-coroutines-flows`でコルーチンとFlowパターンを参照。
