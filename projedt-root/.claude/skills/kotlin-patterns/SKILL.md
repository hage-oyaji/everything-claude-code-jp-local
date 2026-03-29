---
name: kotlin-patterns
description: コルーチン、null安全、DSLビルダーを活用した、堅牢で効率的かつ保守可能なKotlinアプリケーション構築のためのイディオマティックなKotlinパターン、ベストプラクティス、規約。
origin: ECC
---

# Kotlin開発パターン

堅牢で効率的かつ保守可能なアプリケーションを構築するためのイディオマティックなKotlinパターンとベストプラクティス。

## 使用するタイミング

- 新しいKotlinコードの作成
- Kotlinコードのレビュー
- 既存Kotlinコードのリファクタリング
- Kotlinモジュールやライブラリの設計
- Gradle Kotlin DSLビルドの設定

## 仕組み

このスキルは7つの主要分野にわたってイディオマティックなKotlinの規約を適用します：型システムとsafe-callオペレーターを使用したnull安全、`val`とデータクラスの`copy()`によるイミュータビリティ、網羅的な型階層のためのsealed classesとinterfaces、コルーチンと`Flow`による構造化された並行処理、継承なしで振る舞いを追加するための拡張関数、`@DslMarker`とラムダレシーバーによるタイプセーフDSLビルダー、ビルド設定のためのGradle Kotlin DSL。

## 使用例

**Elvis演算子によるnull安全：**
```kotlin
fun getUserEmail(userId: String): String {
    val user = userRepository.findById(userId)
    return user?.email ?: "unknown@example.com"
}
```

**網羅的な結果のためのsealed class：**
```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Failure(val error: AppError) : Result<Nothing>()
    data object Loading : Result<Nothing>()
}
```

**async/awaitによる構造化された並行処理：**
```kotlin
suspend fun fetchUserWithPosts(userId: String): UserProfile =
    coroutineScope {
        val user = async { userService.getUser(userId) }
        val posts = async { postService.getUserPosts(userId) }
        UserProfile(user = user.await(), posts = posts.await())
    }
```

## コア原則

### 1. Null安全

Kotlinの型システムはnullableとnon-nullableの型を区別します。これを最大限に活用してください。

```kotlin
// Good: デフォルトでnon-nullable型を使用
fun getUser(id: String): User {
    return userRepository.findById(id)
        ?: throw UserNotFoundException("User $id not found")
}

// Good: safe callとElvis演算子
fun getUserEmail(userId: String): String {
    val user = userRepository.findById(userId)
    return user?.email ?: "unknown@example.com"
}

// Bad: nullable型の強制アンラップ
fun getUserEmail(userId: String): String {
    val user = userRepository.findById(userId)
    return user!!.email // nullの場合NPEをスロー
}
```

### 2. デフォルトでイミュータブル

`var`より`val`を、ミュータブルコレクションよりイミュータブルコレクションを優先。

```kotlin
// Good: イミュータブルデータ
data class User(
    val id: String,
    val name: String,
    val email: String,
)

// Good: copy()による変換
fun updateEmail(user: User, newEmail: String): User =
    user.copy(email = newEmail)

// Good: イミュータブルコレクション
val users: List<User> = listOf(user1, user2)
val filtered = users.filter { it.email.isNotBlank() }

// Bad: ミュータブル状態
var currentUser: User? = null // ミュータブルなグローバル状態を避ける
val mutableUsers = mutableListOf<User>() // 本当に必要でない限り避ける
```

### 3. 式ボディと単一式関数

簡潔で読みやすい関数のために式ボディを使用。

```kotlin
// Good: 式ボディ
fun isAdult(age: Int): Boolean = age >= 18

fun formatFullName(first: String, last: String): String =
    "$first $last".trim()

fun User.displayName(): String =
    name.ifBlank { email.substringBefore('@') }

// Good: 式としてのwhen
fun statusMessage(code: Int): String = when (code) {
    200 -> "OK"
    404 -> "Not Found"
    500 -> "Internal Server Error"
    else -> "Unknown status: $code"
}

// Bad: 不要なブロックボディ
fun isAdult(age: Int): Boolean {
    return age >= 18
}
```

### 4. 値オブジェクトのためのデータクラス

主にデータを保持する型にはデータクラスを使用。

```kotlin
// Good: equals/hashCode/copyを持つデータクラス
data class CreateUserRequest(
    val name: String,
    val email: String,
    val role: Role = Role.USER,
)

// Good: 型安全のためのvalue class（ランタイムでゼロオーバーヘッド）
@JvmInline
value class UserId(val value: String) {
    init {
        require(value.isNotBlank()) { "UserId cannot be blank" }
    }
}

@JvmInline
value class Email(val value: String) {
    init {
        require('@' in value) { "Invalid email: $value" }
    }
}

fun getUser(id: UserId): User = userRepository.findById(id)
```

## Sealed ClassesとInterfaces

### 制限された階層のモデリング

```kotlin
// Good: 網羅的whenのためのsealed class
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Failure(val error: AppError) : Result<Nothing>()
    data object Loading : Result<Nothing>()
}

fun <T> Result<T>.getOrNull(): T? = when (this) {
    is Result.Success -> data
    is Result.Failure -> null
    is Result.Loading -> null
}

fun <T> Result<T>.getOrThrow(): T = when (this) {
    is Result.Success -> data
    is Result.Failure -> throw error.toException()
    is Result.Loading -> throw IllegalStateException("Still loading")
}
```

### APIレスポンスのためのSealed Interfaces

```kotlin
sealed interface ApiError {
    val message: String

    data class NotFound(override val message: String) : ApiError
    data class Unauthorized(override val message: String) : ApiError
    data class Validation(
        override val message: String,
        val field: String,
    ) : ApiError
    data class Internal(
        override val message: String,
        val cause: Throwable? = null,
    ) : ApiError
}

fun ApiError.toStatusCode(): Int = when (this) {
    is ApiError.NotFound -> 404
    is ApiError.Unauthorized -> 401
    is ApiError.Validation -> 422
    is ApiError.Internal -> 500
}
```

## スコープ関数

### 使い分け

```kotlin
// let: nullableまたはスコープされた結果の変換
val length: Int? = name?.let { it.trim().length }

// apply: オブジェクトの設定（オブジェクトを返す）
val user = User().apply {
    name = "Alice"
    email = "alice@example.com"
}

// also: 副作用（オブジェクトを返す）
val user = createUser(request).also { logger.info("Created user: ${it.id}") }

// run: レシーバー付きブロックの実行（結果を返す）
val result = connection.run {
    prepareStatement(sql)
    executeQuery()
}

// with: runの非拡張形式
val csv = with(StringBuilder()) {
    appendLine("name,email")
    users.forEach { appendLine("${it.name},${it.email}") }
    toString()
}
```

### アンチパターン

```kotlin
// Bad: スコープ関数のネスト
user?.let { u ->
    u.address?.let { addr ->
        addr.city?.let { city ->
            println(city) // 読みにくい
        }
    }
}

// Good: safe callのチェーン
val city = user?.address?.city
city?.let { println(it) }
```

## 拡張関数

### 継承なしで機能を追加

```kotlin
// Good: ドメイン固有の拡張
fun String.toSlug(): String =
    lowercase()
        .replace(Regex("[^a-z0-9\\s-]"), "")
        .replace(Regex("\\s+"), "-")
        .trim('-')

fun Instant.toLocalDate(zone: ZoneId = ZoneId.systemDefault()): LocalDate =
    atZone(zone).toLocalDate()

// Good: コレクション拡張
fun <T> List<T>.second(): T = this[1]

fun <T> List<T>.secondOrNull(): T? = getOrNull(1)

// Good: スコープされた拡張（グローバル名前空間を汚染しない）
class UserService {
    private fun User.isActive(): Boolean =
        status == Status.ACTIVE && lastLogin.isAfter(Instant.now().minus(30, ChronoUnit.DAYS))

    fun getActiveUsers(): List<User> = userRepository.findAll().filter { it.isActive() }
}
```

## コルーチン

### 構造化された並行処理

```kotlin
// Good: coroutineScopeによる構造化された並行処理
suspend fun fetchUserWithPosts(userId: String): UserProfile =
    coroutineScope {
        val userDeferred = async { userService.getUser(userId) }
        val postsDeferred = async { postService.getUserPosts(userId) }

        UserProfile(
            user = userDeferred.await(),
            posts = postsDeferred.await(),
        )
    }

// Good: 子が独立して失敗できる場合のsupervisorScope
suspend fun fetchDashboard(userId: String): Dashboard =
    supervisorScope {
        val user = async { userService.getUser(userId) }
        val notifications = async { notificationService.getRecent(userId) }
        val recommendations = async { recommendationService.getFor(userId) }

        Dashboard(
            user = user.await(),
            notifications = try {
                notifications.await()
            } catch (e: CancellationException) {
                throw e
            } catch (e: Exception) {
                emptyList()
            },
            recommendations = try {
                recommendations.await()
            } catch (e: CancellationException) {
                throw e
            } catch (e: Exception) {
                emptyList()
            },
        )
    }
```

### リアクティブストリームのためのFlow

```kotlin
// Good: 適切なエラーハンドリングを持つcold flow
fun observeUsers(): Flow<List<User>> = flow {
    while (currentCoroutineContext().isActive) {
        val users = userRepository.findAll()
        emit(users)
        delay(5.seconds)
    }
}.catch { e ->
    logger.error("Error observing users", e)
    emit(emptyList())
}

// Good: Flowオペレーター
fun searchUsers(query: Flow<String>): Flow<List<User>> =
    query
        .debounce(300.milliseconds)
        .distinctUntilChanged()
        .filter { it.length >= 2 }
        .mapLatest { q -> userRepository.search(q) }
        .catch { emit(emptyList()) }
```

### キャンセルとクリーンアップ

```kotlin
// Good: キャンセルを尊重
suspend fun processItems(items: List<Item>) {
    items.forEach { item ->
        ensureActive() // 重い処理の前にキャンセルを確認
        processItem(item)
    }
}

// Good: try/finallyによるクリーンアップ
suspend fun acquireAndProcess() {
    val resource = acquireResource()
    try {
        resource.process()
    } finally {
        withContext(NonCancellable) {
            resource.release() // キャンセル時も常にリリース
        }
    }
}
```

## デリゲーション

### プロパティデリゲーション

```kotlin
// 遅延初期化
val expensiveData: List<User> by lazy {
    userRepository.findAll()
}

// 監視可能なプロパティ
var name: String by Delegates.observable("initial") { _, old, new ->
    logger.info("Name changed from '$old' to '$new'")
}

// Map-backedプロパティ
class Config(private val map: Map<String, Any?>) {
    val host: String by map
    val port: Int by map
    val debug: Boolean by map
}

val config = Config(mapOf("host" to "localhost", "port" to 8080, "debug" to true))
```

### インターフェースデリゲーション

```kotlin
// Good: インターフェース実装のデリゲート
class LoggingUserRepository(
    private val delegate: UserRepository,
    private val logger: Logger,
) : UserRepository by delegate {
    // ロギングを追加したいものだけをオーバーライド
    override suspend fun findById(id: String): User? {
        logger.info("Finding user by id: $id")
        return delegate.findById(id).also {
            logger.info("Found user: ${it?.name ?: "null"}")
        }
    }
}
```

## DSLビルダー

### タイプセーフビルダー

```kotlin
// Good: @DslMarker付きDSL
@DslMarker
annotation class HtmlDsl

@HtmlDsl
class HTML {
    private val children = mutableListOf<Element>()

    fun head(init: Head.() -> Unit) {
        children += Head().apply(init)
    }

    fun body(init: Body.() -> Unit) {
        children += Body().apply(init)
    }

    override fun toString(): String = children.joinToString("\n")
}

fun html(init: HTML.() -> Unit): HTML = HTML().apply(init)

// Usage
val page = html {
    head { title("My Page") }
    body {
        h1("Welcome")
        p("Hello, World!")
    }
}
```

### 設定DSL

```kotlin
data class ServerConfig(
    val host: String = "0.0.0.0",
    val port: Int = 8080,
    val ssl: SslConfig? = null,
    val database: DatabaseConfig? = null,
)

data class SslConfig(val certPath: String, val keyPath: String)
data class DatabaseConfig(val url: String, val maxPoolSize: Int = 10)

class ServerConfigBuilder {
    var host: String = "0.0.0.0"
    var port: Int = 8080
    private var ssl: SslConfig? = null
    private var database: DatabaseConfig? = null

    fun ssl(certPath: String, keyPath: String) {
        ssl = SslConfig(certPath, keyPath)
    }

    fun database(url: String, maxPoolSize: Int = 10) {
        database = DatabaseConfig(url, maxPoolSize)
    }

    fun build(): ServerConfig = ServerConfig(host, port, ssl, database)
}

fun serverConfig(init: ServerConfigBuilder.() -> Unit): ServerConfig =
    ServerConfigBuilder().apply(init).build()

// Usage
val config = serverConfig {
    host = "0.0.0.0"
    port = 443
    ssl("/certs/cert.pem", "/certs/key.pem")
    database("jdbc:postgresql://localhost:5432/mydb", maxPoolSize = 20)
}
```

## 遅延評価のためのシーケンス

```kotlin
// Good: 複数の操作を持つ大きなコレクションにはシーケンスを使用
val result = users.asSequence()
    .filter { it.isActive }
    .map { it.email }
    .filter { it.endsWith("@company.com") }
    .take(10)
    .toList()

// Good: 無限シーケンスの生成
val fibonacci: Sequence<Long> = sequence {
    var a = 0L
    var b = 1L
    while (true) {
        yield(a)
        val next = a + b
        a = b
        b = next
    }
}

val first20 = fibonacci.take(20).toList()
```

## Gradle Kotlin DSL

### build.gradle.kts設定

```kotlin
// Check for latest versions: https://kotlinlang.org/docs/releases.html
plugins {
    kotlin("jvm") version "2.3.10"
    kotlin("plugin.serialization") version "2.3.10"
    id("io.ktor.plugin") version "3.4.0"
    id("org.jetbrains.kotlinx.kover") version "0.9.7"
    id("io.gitlab.arturbosch.detekt") version "1.23.8"
}

group = "com.example"
version = "1.0.0"

kotlin {
    jvmToolchain(21)
}

dependencies {
    // Ktor
    implementation("io.ktor:ktor-server-core:3.4.0")
    implementation("io.ktor:ktor-server-netty:3.4.0")
    implementation("io.ktor:ktor-server-content-negotiation:3.4.0")
    implementation("io.ktor:ktor-serialization-kotlinx-json:3.4.0")

    // Exposed
    implementation("org.jetbrains.exposed:exposed-core:1.0.0")
    implementation("org.jetbrains.exposed:exposed-dao:1.0.0")
    implementation("org.jetbrains.exposed:exposed-jdbc:1.0.0")
    implementation("org.jetbrains.exposed:exposed-kotlin-datetime:1.0.0")

    // Koin
    implementation("io.insert-koin:koin-ktor:4.2.0")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.10.2")

    // Testing
    testImplementation("io.kotest:kotest-runner-junit5:6.1.4")
    testImplementation("io.kotest:kotest-assertions-core:6.1.4")
    testImplementation("io.kotest:kotest-property:6.1.4")
    testImplementation("io.mockk:mockk:1.14.9")
    testImplementation("io.ktor:ktor-server-test-host:3.4.0")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.10.2")
}

tasks.withType<Test> {
    useJUnitPlatform()
}

detekt {
    config.setFrom(files("config/detekt/detekt.yml"))
    buildUponDefaultConfig = true
}
```

## エラーハンドリングパターン

### ドメイン操作のためのResult型

```kotlin
// Good: KotlinのResultまたはカスタムsealed classを使用
suspend fun createUser(request: CreateUserRequest): Result<User> = runCatching {
    require(request.name.isNotBlank()) { "Name cannot be blank" }
    require('@' in request.email) { "Invalid email format" }

    val user = User(
        id = UserId(UUID.randomUUID().toString()),
        name = request.name,
        email = Email(request.email),
    )
    userRepository.save(user)
    user
}

// Good: 結果のチェーン
val displayName = createUser(request)
    .map { it.name }
    .getOrElse { "Unknown" }
```

### require、check、error

```kotlin
// Good: 明確なメッセージ付き前提条件
fun withdraw(account: Account, amount: Money): Account {
    require(amount.value > 0) { "Amount must be positive: $amount" }
    check(account.balance >= amount) { "Insufficient balance: ${account.balance} < $amount" }

    return account.copy(balance = account.balance - amount)
}
```

## コレクション操作

### イディオマティックなコレクション処理

```kotlin
// Good: チェーンされた操作
val activeAdminEmails: List<String> = users
    .filter { it.role == Role.ADMIN && it.isActive }
    .sortedBy { it.name }
    .map { it.email }

// Good: グルーピングと集約
val usersByRole: Map<Role, List<User>> = users.groupBy { it.role }

val oldestByRole: Map<Role, User?> = users.groupBy { it.role }
    .mapValues { (_, users) -> users.minByOrNull { it.createdAt } }

// Good: マップ作成のためのassociate
val usersById: Map<UserId, User> = users.associateBy { it.id }

// Good: 分割のためのpartition
val (active, inactive) = users.partition { it.isActive }
```

## クイックリファレンス: Kotlinイディオム

| イディオム | 説明 |
|-------|------|
| `val` over `var` | イミュータブル変数を優先 |
| `data class` | equals/hashCode/copyを持つ値オブジェクト |
| `sealed class/interface` | 制限された型階層 |
| `value class` | ゼロオーバーヘッドのタイプセーフラッパー |
| 式`when` | 網羅的パターンマッチング |
| Safe call `?.` | null安全なメンバーアクセス |
| Elvis `?:` | nullableのデフォルト値 |
| `let`/`apply`/`also`/`run`/`with` | クリーンコードのためのスコープ関数 |
| 拡張関数 | 継承なしで振る舞いを追加 |
| `copy()` | データクラスのイミュータブル更新 |
| `require`/`check` | 前提条件アサーション |
| コルーチン`async`/`await` | 構造化された並行実行 |
| `Flow` | コールドリアクティブストリーム |
| `sequence` | 遅延評価 |
| デリゲーション`by` | 継承なしの実装再利用 |

## 避けるべきアンチパターン

```kotlin
// Bad: nullable型の強制アンラップ
val name = user!!.name

// Bad: Javaからのプラットフォーム型リーク
fun getLength(s: String) = s.length // Safe
fun getLength(s: String?) = s?.length ?: 0 // Javaからのnullを処理

// Bad: ミュータブルデータクラス
data class MutableUser(var name: String, var email: String)

// Bad: 制御フローのための例外使用
try {
    val user = findUser(id)
} catch (e: NotFoundException) {
    // 予期されるケースに例外を使用しない
}

// Good: nullable戻り値またはResultを使用
val user: User? = findUserOrNull(id)

// Bad: コルーチンスコープを無視
GlobalScope.launch { /* GlobalScopeを避ける */ }

// Good: 構造化された並行処理を使用
coroutineScope {
    launch { /* 適切にスコープされた */ }
}

// Bad: スコープ関数の深いネスト
user?.let { u ->
    u.address?.let { a ->
        a.city?.let { c -> process(c) }
    }
}

// Good: 直接的なnull安全チェーン
user?.address?.city?.let { process(it) }
```

**留意事項**: Kotlinコードは簡潔でありながら読みやすくあるべきです。安全性のために型システムを活用し、イミュータビリティを優先し、並行処理にはコルーチンを使用してください。迷ったらコンパイラに助けてもらいましょう。
