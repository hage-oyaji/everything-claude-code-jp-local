---
name: laravel-patterns
description: Laravelアーキテクチャパターン — ルーティング/コントローラー、Eloquent ORM、サービスレイヤー、キュー、イベント、キャッシュ、APIリソースを含む本番アプリ向け。
origin: ECC
---

# Laravel開発パターン

スケーラブルでメンテナブルなアプリケーションのためのプロダクショングレードのLaravelアーキテクチャパターン。

## 使用タイミング

- Laravel WebアプリケーションまたはAPIの構築
- コントローラー、サービス、ドメインロジックの構造化
- Eloquentモデルとリレーションシップの操作
- リソースとページネーションによるAPI設計
- キュー、イベント、キャッシュ、バックグラウンドジョブの追加

## 仕組み

- 明確な境界（コントローラー -> サービス/アクション -> モデル）でアプリを構造化。
- ルーティングを予測可能に保つために明示的バインディングとスコープドバインディングを使用。アクセスコントロールには認可を適用。
- 型付きモデル、キャスト、スコープを活用してドメインロジックの一貫性を保つ。
- IO負荷の高い処理はキューに、高コストな読み取りはキャッシュに。
- 設定は`config/*`に集約し、環境を明示的に保つ。

## 事例

### プロジェクト構造

明確なレイヤー境界（HTTP、サービス/アクション、モデル）を持つ慣例的なLaravelレイアウトを使用。

### 推奨レイアウト

```
app/
├── Actions/            # Single-purpose use cases
├── Console/
├── Events/
├── Exceptions/
├── Http/
│   ├── Controllers/
│   ├── Middleware/
│   ├── Requests/       # Form request validation
│   └── Resources/      # API resources
├── Jobs/
├── Models/
├── Policies/
├── Providers/
├── Services/           # Coordinating domain services
└── Support/
config/
database/
├── factories/
├── migrations/
└── seeders/
resources/
├── views/
└── lang/
routes/
├── api.php
├── web.php
└── console.php
```

### コントローラー -> サービス -> アクション

コントローラーは薄く保つ。オーケストレーションはサービスに、単一目的のロジックはアクションに。

```php
final class CreateOrderAction
{
    public function __construct(private OrderRepository $orders) {}

    public function handle(CreateOrderData $data): Order
    {
        return $this->orders->create($data);
    }
}

final class OrdersController extends Controller
{
    public function __construct(private CreateOrderAction $createOrder) {}

    public function store(StoreOrderRequest $request): JsonResponse
    {
        $order = $this->createOrder->handle($request->toDto());

        return response()->json([
            'success' => true,
            'data' => OrderResource::make($order),
            'error' => null,
            'meta' => null,
        ], 201);
    }
}
```

### ルーティングとコントローラー

明確さのためにルートモデルバインディングとリソースコントローラーを優先。

```php
use Illuminate\Support\Facades\Route;

Route::middleware('auth:sanctum')->group(function () {
    Route::apiResource('projects', ProjectController::class);
});
```

### ルートモデルバインディング（スコープド）

クロステナントアクセスを防止するためにスコープドバインディングを使用。

```php
Route::scopeBindings()->group(function () {
    Route::get('/accounts/{account}/projects/{project}', [ProjectController::class, 'show']);
});
```

### ネストされたルートとバインディング名

- 二重ネストを避けるためにプレフィックスとパスの一貫性を保つ（例: `conversation`と`conversations`）。
- バインドされたモデルに一致する単一のパラメータ名を使用（例: `Conversation`に対して`{conversation}`）。
- ネスト時に親子関係を強制するためにスコープドバインディングを優先。

```php
use App\Http\Controllers\Api\ConversationController;
use App\Http\Controllers\Api\MessageController;
use Illuminate\Support\Facades\Route;

Route::middleware('auth:sanctum')->prefix('conversations')->group(function () {
    Route::post('/', [ConversationController::class, 'store'])->name('conversations.store');

    Route::scopeBindings()->group(function () {
        Route::get('/{conversation}', [ConversationController::class, 'show'])
            ->name('conversations.show');

        Route::post('/{conversation}/messages', [MessageController::class, 'store'])
            ->name('conversation-messages.store');

        Route::get('/{conversation}/messages/{message}', [MessageController::class, 'show'])
            ->name('conversation-messages.show');
    });
});
```

パラメータを異なるモデルクラスに解決したい場合は、明示的バインディングを定義。カスタムバインディングロジックには`Route::bind()`を使用するか、モデルに`resolveRouteBinding()`を実装。

```php
use App\Models\AiConversation;
use Illuminate\Support\Facades\Route;

Route::model('conversation', AiConversation::class);
```

### サービスコンテナバインディング

明確な依存関係配線のためにサービスプロバイダーでインターフェースを実装にバインド。

```php
use App\Repositories\EloquentOrderRepository;
use App\Repositories\OrderRepository;
use Illuminate\Support\ServiceProvider;

final class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(OrderRepository::class, EloquentOrderRepository::class);
    }
}
```

### Eloquentモデルパターン

### モデル設定

```php
final class Project extends Model
{
    use HasFactory;

    protected $fillable = ['name', 'owner_id', 'status'];

    protected $casts = [
        'status' => ProjectStatus::class,
        'archived_at' => 'datetime',
    ];

    public function owner(): BelongsTo
    {
        return $this->belongsTo(User::class, 'owner_id');
    }

    public function scopeActive(Builder $query): Builder
    {
        return $query->whereNull('archived_at');
    }
}
```

### カスタムキャストと値オブジェクト

厳密な型付けにはenumまたは値オブジェクトを使用。

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

protected $casts = [
    'status' => ProjectStatus::class,
];
```

```php
protected function budgetCents(): Attribute
{
    return Attribute::make(
        get: fn (int $value) => Money::fromCents($value),
        set: fn (Money $money) => $money->toCents(),
    );
}
```

### N+1を避けるためのイーガーローディング

```php
$orders = Order::query()
    ->with(['customer', 'items.product'])
    ->latest()
    ->paginate(25);
```

### 複雑なフィルターのためのクエリオブジェクト

```php
final class ProjectQuery
{
    public function __construct(private Builder $query) {}

    public function ownedBy(int $userId): self
    {
        $query = clone $this->query;

        return new self($query->where('owner_id', $userId));
    }

    public function active(): self
    {
        $query = clone $this->query;

        return new self($query->whereNull('archived_at'));
    }

    public function builder(): Builder
    {
        return $this->query;
    }
}
```

### グローバルスコープとソフトデリート

デフォルトフィルタリングにはグローバルスコープを、復元可能なレコードには`SoftDeletes`を使用。
レイヤード動作を意図しない限り、同じフィルターにグローバルスコープと名前付きスコープの両方を使用しない。

```php
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Builder;

final class Project extends Model
{
    use SoftDeletes;

    protected static function booted(): void
    {
        static::addGlobalScope('active', function (Builder $builder): void {
            $builder->whereNull('archived_at');
        });
    }
}
```

### 再利用可能なフィルターのためのクエリスコープ

```php
use Illuminate\Database\Eloquent\Builder;

final class Project extends Model
{
    public function scopeOwnedBy(Builder $query, int $userId): Builder
    {
        return $query->where('owner_id', $userId);
    }
}

// In service, repository etc.
$projects = Project::ownedBy($user->id)->get();
```

### 複数ステップ更新のためのトランザクション

```php
use Illuminate\Support\Facades\DB;

DB::transaction(function (): void {
    $order->update(['status' => 'paid']);
    $order->items()->update(['paid_at' => now()]);
});
```

### マイグレーション

### 命名規則

- ファイル名はタイムスタンプを使用: `YYYY_MM_DD_HHMMSS_create_users_table.php`
- マイグレーションは無名クラスを使用（名前付きクラスなし）。ファイル名が意図を伝える
- テーブル名はデフォルトで`snake_case`の複数形

### マイグレーション例

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('orders', function (Blueprint $table): void {
            $table->id();
            $table->foreignId('customer_id')->constrained()->cascadeOnDelete();
            $table->string('status', 32)->index();
            $table->unsignedInteger('total_cents');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('orders');
    }
};
```

### フォームリクエストとバリデーション

バリデーションはフォームリクエストに保持し、入力をDTOに変換。

```php
use App\Models\Order;

final class StoreOrderRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()?->can('create', Order::class) ?? false;
    }

    public function rules(): array
    {
        return [
            'customer_id' => ['required', 'integer', 'exists:customers,id'],
            'items' => ['required', 'array', 'min:1'],
            'items.*.sku' => ['required', 'string'],
            'items.*.quantity' => ['required', 'integer', 'min:1'],
        ];
    }

    public function toDto(): CreateOrderData
    {
        return new CreateOrderData(
            customerId: (int) $this->validated('customer_id'),
            items: $this->validated('items'),
        );
    }
}
```

### APIリソース

リソースとページネーションでAPIレスポンスの一貫性を保つ。

```php
$projects = Project::query()->active()->paginate(25);

return response()->json([
    'success' => true,
    'data' => ProjectResource::collection($projects->items()),
    'error' => null,
    'meta' => [
        'page' => $projects->currentPage(),
        'per_page' => $projects->perPage(),
        'total' => $projects->total(),
    ],
]);
```

### イベント、ジョブ、キュー

- 副作用（メール、アナリティクス）にはドメインイベントを発行
- 遅い処理（レポート、エクスポート、Webhook）にはキューイングされたジョブを使用
- リトライとバックオフを持つべき等なハンドラーを優先

### キャッシュ

- 読み取り負荷の高いエンドポイントと高コストなクエリをキャッシュ
- モデルイベント（作成/更新/削除）でキャッシュを無効化
- 関連データのキャッシュにはタグを使用して簡単に無効化

### 設定と環境

- シークレットは`.env`に、設定は`config/*.php`に保持
- 環境ごとの設定オーバーライドと本番での`config:cache`を使用
