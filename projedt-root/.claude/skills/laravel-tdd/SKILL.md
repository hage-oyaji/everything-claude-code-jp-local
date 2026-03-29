---
name: laravel-tdd
description: PHPUnitとPestによるLaravelのテスト駆動開発、ファクトリー、データベーステスト、フェイク、カバレッジ目標。
origin: ECC
---

# Laravel TDDワークフロー

PHPUnitとPestを使用した80%以上のカバレッジ（ユニット + フィーチャー）によるLaravelアプリケーションのテスト駆動開発。

## 使用タイミング

- Laravelでの新機能やエンドポイント
- バグ修正やリファクタリング
- Eloquentモデル、ポリシー、ジョブ、通知のテスト
- プロジェクトが既にPHPUnitを標準化していない限り、新しいテストにはPestを優先

## 仕組み

### Red-Green-Refactorサイクル

1) 失敗するテストを書く
2) テストを通過する最小限の変更を実装
3) テストをグリーンに保ちながらリファクタリング

### テストレイヤー

- **ユニット**: 純粋なPHPクラス、値オブジェクト、サービス
- **フィーチャー**: HTTPエンドポイント、認証、バリデーション、ポリシー
- **インテグレーション**: データベース + キュー + 外部境界

スコープに基づいてレイヤーを選択:

- 純粋なビジネスロジックとサービスには**ユニット**テストを使用。
- HTTP、認証、バリデーション、レスポンス形状には**フィーチャー**テストを使用。
- DB/キュー/外部サービスを一緒に検証する場合は**インテグレーション**テストを使用。

### データベース戦略

- ほとんどのフィーチャー/インテグレーションテストには`RefreshDatabase`（テスト実行ごとに1回マイグレーションを実行し、サポートされている場合は各テストをトランザクションでラップ。インメモリデータベースではテストごとに再マイグレーションする場合あり）
- スキーマが既にマイグレーション済みでテストごとのロールバックのみが必要な場合は`DatabaseTransactions`
- テストごとに完全なmigrate/freshが必要でそのコストを許容できる場合は`DatabaseMigrations`

データベースに触れるテストにはデフォルトで`RefreshDatabase`を使用: トランザクションサポートのあるデータベースでは、テスト実行ごとに1回マイグレーションを実行し（静的フラグを使用）、各テストをトランザクションでラップ。`:memory:` SQLiteやトランザクションのないコネクションでは、各テスト前にマイグレーション。スキーマが既にマイグレーション済みでテストごとのロールバックのみが必要な場合は`DatabaseTransactions`を使用。

### テスティングフレームワークの選択

- 利用可能な場合は新しいテストに**Pest**をデフォルトで使用。
- プロジェクトが既にPHPUnitを標準化しているか、PHPUnit固有のツーリングが必要な場合にのみ**PHPUnit**を使用。

## 事例

### PHPUnitの例

```php
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class ProjectControllerTest extends TestCase
{
    use RefreshDatabase;

    public function test_owner_can_create_project(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)->postJson('/api/projects', [
            'name' => 'New Project',
        ]);

        $response->assertCreated();
        $this->assertDatabaseHas('projects', ['name' => 'New Project']);
    }
}
```

### フィーチャーテストの例（HTTPレイヤー）

```php
use App\Models\Project;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class ProjectIndexTest extends TestCase
{
    use RefreshDatabase;

    public function test_projects_index_returns_paginated_results(): void
    {
        $user = User::factory()->create();
        Project::factory()->count(3)->for($user)->create();

        $response = $this->actingAs($user)->getJson('/api/projects');

        $response->assertOk();
        $response->assertJsonStructure(['success', 'data', 'error', 'meta']);
    }
}
```

### Pestの例

```php
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

use function Pest\Laravel\actingAs;
use function Pest\Laravel\assertDatabaseHas;

uses(RefreshDatabase::class);

test('owner can create project', function () {
    $user = User::factory()->create();

    $response = actingAs($user)->postJson('/api/projects', [
        'name' => 'New Project',
    ]);

    $response->assertCreated();
    assertDatabaseHas('projects', ['name' => 'New Project']);
});
```

### フィーチャーテストPestの例（HTTPレイヤー）

```php
use App\Models\Project;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

use function Pest\Laravel\actingAs;

uses(RefreshDatabase::class);

test('projects index returns paginated results', function () {
    $user = User::factory()->create();
    Project::factory()->count(3)->for($user)->create();

    $response = actingAs($user)->getJson('/api/projects');

    $response->assertOk();
    $response->assertJsonStructure(['success', 'data', 'error', 'meta']);
});
```

### ファクトリーとステート

- テストデータにはファクトリーを使用
- エッジケース用にステートを定義（archived、admin、trial）

```php
$user = User::factory()->state(['role' => 'admin'])->create();
```

### データベーステスト

- クリーンな状態のために`RefreshDatabase`を使用
- テストを分離し決定的に保つ
- 手動クエリより`assertDatabaseHas`を優先

### 永続化テストの例

```php
use App\Models\Project;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class ProjectRepositoryTest extends TestCase
{
    use RefreshDatabase;

    public function test_project_can_be_retrieved_by_slug(): void
    {
        $project = Project::factory()->create(['slug' => 'alpha']);

        $found = Project::query()->where('slug', 'alpha')->firstOrFail();

        $this->assertSame($project->id, $found->id);
    }
}
```

### 副作用のためのフェイク

- ジョブには`Bus::fake()`
- キューイングされた処理には`Queue::fake()`
- 通知には`Mail::fake()`と`Notification::fake()`
- ドメインイベントには`Event::fake()`

```php
use Illuminate\Support\Facades\Queue;

Queue::fake();

dispatch(new SendOrderConfirmation($order->id));

Queue::assertPushed(SendOrderConfirmation::class);
```

```php
use Illuminate\Support\Facades\Notification;

Notification::fake();

$user->notify(new InvoiceReady($invoice));

Notification::assertSentTo($user, InvoiceReady::class);
```

### 認証テスト（Sanctum）

```php
use Laravel\Sanctum\Sanctum;

Sanctum::actingAs($user);

$response = $this->getJson('/api/projects');
$response->assertOk();
```

### HTTPと外部サービス

- 外部APIを分離するために`Http::fake()`を使用
- `Http::assertSent()`でアウトバウンドペイロードをアサート

### カバレッジ目標

- ユニット + フィーチャーテストで80%以上のカバレッジを適用
- CIでは`pcov`または`XDEBUG_MODE=coverage`を使用

### テストコマンド

- `php artisan test`
- `vendor/bin/phpunit`
- `vendor/bin/pest`

### テスト設定

- `phpunit.xml`で`DB_CONNECTION=sqlite`と`DB_DATABASE=:memory:`を設定して高速テスト
- 開発/本番データに触れないようテスト用に別の環境を保持

### 認可テスト

```php
use Illuminate\Support\Facades\Gate;

$this->assertTrue(Gate::forUser($user)->allows('update', $project));
$this->assertFalse(Gate::forUser($otherUser)->allows('update', $project));
```

### Inertiaフィーチャーテスト

Inertia.jsを使用する場合、Inertiaテストヘルパーでコンポーネント名とpropsをアサート。

```php
use App\Models\User;
use Inertia\Testing\AssertableInertia;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class DashboardInertiaTest extends TestCase
{
    use RefreshDatabase;

    public function test_dashboard_inertia_props(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)->get('/dashboard');

        $response->assertOk();
        $response->assertInertia(fn (AssertableInertia $page) => $page
            ->component('Dashboard')
            ->where('user.id', $user->id)
            ->has('projects')
        );
    }
}
```

Inertiaレスポンスに合わせてテストを保つため、生のJSONアサーションより`assertInertia`を優先してください。
