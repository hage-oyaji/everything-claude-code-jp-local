---
name: laravel-security
description: Laravelセキュリティベストプラクティス — 認証/認可、バリデーション、CSRF、マスアサインメント、ファイルアップロード、シークレット、レート制限、セキュアなデプロイメント。
origin: ECC
---

# Laravelセキュリティベストプラクティス

一般的な脆弱性から保護するためのLaravelアプリケーション向け包括的なセキュリティガイダンス。

## アクティベーション条件

- 認証または認可の追加
- ユーザー入力とファイルアップロードの処理
- 新しいAPIエンドポイントの構築
- シークレットと環境設定の管理
- 本番デプロイメントの強化

## 仕組み

- ミドルウェアがベースライン保護を提供（`VerifyCsrfToken`によるCSRF、`SecurityHeaders`によるセキュリティヘッダー）。
- ガードとポリシーがアクセスコントロールを適用（`auth:sanctum`、`$this->authorize`、ポリシーミドルウェア）。
- フォームリクエストがサービスに到達する前に入力を検証・整形（`UploadInvoiceRequest`）。
- レート制限が認証コントロールと併せて不正利用保護を追加（`RateLimiter::for('login')`）。
- データの安全性は暗号化キャスト、マスアサインメントガード、署名付きURL（`URL::temporarySignedRoute` + `signed`ミドルウェア）から。

## コアセキュリティ設定

- 本番では`APP_DEBUG=false`
- `APP_KEY`は必ず設定し、漏洩時にはローテート
- `SESSION_SECURE_COOKIE=true`と`SESSION_SAME_SITE=lax`（機密アプリでは`strict`）を設定
- 正しいHTTPS検出のためにトラステッドプロキシを設定

## セッションとCookieの強化

- JavaScriptアクセスを防止するために`SESSION_HTTP_ONLY=true`を設定
- 高リスクフローには`SESSION_SAME_SITE=strict`を使用
- ログインと権限変更時にセッションを再生成

## 認証とトークン

- API認証にはLaravel SanctumまたはPassportを使用
- 機密データには短寿命トークンとリフレッシュフローを優先
- ログアウトと漏洩アカウントでトークンを取り消し

ルート保護の例:

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;

Route::middleware('auth:sanctum')->get('/me', function (Request $request) {
    return $request->user();
});
```

## パスワードセキュリティ

- `Hash::make()`でパスワードをハッシュし、平文を保存しない
- リセットフローにはLaravelのパスワードブローカーを使用

```php
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\Rules\Password;

$validated = $request->validate([
    'password' => ['required', 'string', Password::min(12)->letters()->mixedCase()->numbers()->symbols()],
]);

$user->update(['password' => Hash::make($validated['password'])]);
```

## 認可: ポリシーとゲート

- モデルレベルの認可にはポリシーを使用
- コントローラーとサービスで認可を適用

```php
$this->authorize('update', $project);
```

ルートレベルの適用にはポリシーミドルウェアを使用:

```php
use Illuminate\Support\Facades\Route;

Route::put('/projects/{project}', [ProjectController::class, 'update'])
    ->middleware(['auth:sanctum', 'can:update,project']);
```

## バリデーションとデータサニタイズ

- 常にフォームリクエストで入力を検証
- 厳密なバリデーションルールと型チェックを使用
- 派生フィールドにリクエストペイロードを信頼しない

## マスアサインメント保護

- `$fillable`または`$guarded`を使用し、`Model::unguard()`を避ける
- DTOまたは明示的な属性マッピングを優先

## SQLインジェクション防止

- Eloquentまたはクエリビルダーのパラメータバインディングを使用
- 厳密に必要でない限り生SQLを避ける

```php
DB::select('select * from users where email = ?', [$email]);
```

## XSS防止

- Bladeはデフォルトで出力をエスケープ（`{{ }}`）
- `{!! !!}`は信頼でき、サニタイズ済みのHTMLにのみ使用
- リッチテキストは専用ライブラリでサニタイズ

## CSRF保護

- `VerifyCsrfToken`ミドルウェアを有効に保つ
- フォームには`@csrf`を含め、SPAリクエストにはXSRFトークンを送信

SanctumによるSPA認証では、ステートフルリクエストが設定されていることを確認:

```php
// config/sanctum.php
'stateful' => explode(',', env('SANCTUM_STATEFUL_DOMAINS', 'localhost')),
```

## ファイルアップロードの安全性

- ファイルサイズ、MIMEタイプ、拡張子を検証
- 可能な場合はパブリックパスの外部にアップロードを保存
- 必要に応じてマルウェアスキャン

```php
final class UploadInvoiceRequest extends FormRequest
{
    public function authorize(): bool
    {
        return (bool) $this->user()?->can('upload-invoice');
    }

    public function rules(): array
    {
        return [
            'invoice' => ['required', 'file', 'mimes:pdf', 'max:5120'],
        ];
    }
}
```

```php
$path = $request->file('invoice')->store(
    'invoices',
    config('filesystems.private_disk', 'local') // set this to a non-public disk
);
```

## レート制限

- 認証と書き込みエンドポイントに`throttle`ミドルウェアを適用
- ログイン、パスワードリセット、OTPにはより厳格な制限を使用

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

RateLimiter::for('login', function (Request $request) {
    return [
        Limit::perMinute(5)->by($request->ip()),
        Limit::perMinute(5)->by(strtolower((string) $request->input('email'))),
    ];
});
```

## シークレットと認証情報

- ソース管理にシークレットをコミットしない
- 環境変数とシークレットマネージャーを使用
- 漏洩後にキーをローテートし、セッションを無効化

## 暗号化属性

保存時の機密カラムには暗号化キャストを使用。

```php
protected $casts = [
    'api_token' => 'encrypted',
];
```

## セキュリティヘッダー

- 適切な場所にCSP、HSTS、フレーム保護を追加
- HTTPSリダイレクトを適用するためにトラステッドプロキシ設定を使用

ヘッダーを設定するミドルウェアの例:

```php
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

final class SecurityHeaders
{
    public function handle(Request $request, \Closure $next): Response
    {
        $response = $next($request);

        $response->headers->add([
            'Content-Security-Policy' => "default-src 'self'",
            'Strict-Transport-Security' => 'max-age=31536000', // add includeSubDomains/preload only when all subdomains are HTTPS
            'X-Frame-Options' => 'DENY',
            'X-Content-Type-Options' => 'nosniff',
            'Referrer-Policy' => 'no-referrer',
        ]);

        return $response;
    }
}
```

## CORSとAPIの公開

- `config/cors.php`でオリジンを制限
- 認証付きルートにはワイルドカードオリジンを避ける

```php
// config/cors.php
return [
    'paths' => ['api/*', 'sanctum/csrf-cookie'],
    'allowed_methods' => ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
    'allowed_origins' => ['https://app.example.com'],
    'allowed_headers' => [
        'Content-Type',
        'Authorization',
        'X-Requested-With',
        'X-XSRF-TOKEN',
        'X-CSRF-TOKEN',
    ],
    'supports_credentials' => true,
];
```

## ロギングとPII

- パスワード、トークン、完全なカード情報をログに記録しない
- 構造化ログで機密フィールドをリダクト

```php
use Illuminate\Support\Facades\Log;

Log::info('User updated profile', [
    'user_id' => $user->id,
    'email' => '[REDACTED]',
    'token' => '[REDACTED]',
]);
```

## 依存関係のセキュリティ

- `composer audit`を定期的に実行
- 依存関係を注意深くピン留めし、CVE発生時には迅速に更新

## 署名付きURL

一時的な改ざん防止リンクには署名付きルートを使用。

```php
use Illuminate\Support\Facades\URL;

$url = URL::temporarySignedRoute(
    'downloads.invoice',
    now()->addMinutes(15),
    ['invoice' => $invoice->id]
);
```

```php
use Illuminate\Support\Facades\Route;

Route::get('/invoices/{invoice}/download', [InvoiceController::class, 'download'])
    ->name('downloads.invoice')
    ->middleware('signed');
```
