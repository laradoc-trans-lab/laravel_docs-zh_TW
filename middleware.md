# Middleware

- [簡介](#introduction)
- [定義 Middleware](#defining-middleware)
- [註冊 Middleware](#registering-middleware)
    - [全域 Middleware](#global-middleware)
    - [將 Middleware 指派給路由](#assigning-middleware-to-routes)
    - [Middleware 群組](#middleware-groups)
    - [Middleware 別名](#middleware-aliases)
    - [排序 Middleware](#sorting-middleware)
- [Middleware 參數](#middleware-parameters)
- [可終止的 Middleware](#terminable-middleware)

<a name="introduction"></a>
## 簡介

Middleware 提供了一種方便的機制，用於檢查和過濾進入應用程式的 HTTP 請求。舉例來說，Laravel 內建了一個 Middleware，用於驗證應用程式的使用者是否已通過身份驗證。如果使用者未通過身份驗證，該 Middleware 會將使用者重新導向到應用程式的登入畫面。然而，如果使用者已通過身份驗證，該 Middleware 將允許請求進一步進入應用程式。

除了身份驗證之外，還可以編寫額外的 Middleware 來執行各種任務。例如，一個日誌記錄 Middleware 可能會記錄所有進入應用程式的請求。Laravel 中包含了多種 Middleware，包括用於身份驗證和 CSRF 保護的 Middleware；然而，所有使用者定義的 Middleware 通常都位於應用程式的 `app/Http/Middleware` 目錄中。

<a name="defining-middleware"></a>
## 定義 Middleware

要建立一個新的 Middleware，請使用 `make:middleware` Artisan 命令：

```shell
php artisan make:middleware EnsureTokenIsValid
```

此命令將在 `app/Http/Middleware` 目錄中建立一個新的 `EnsureTokenIsValid` 類別。在這個 Middleware 中，我們只允許在提供的 `token` 輸入與指定值匹配時才存取路由。否則，我們將把使用者重新導向回 `/home` URI：

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureTokenIsValid
{
    /**
     * Handle an incoming request.
     *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next): Response
    {
        if ($request->input('token') !== 'my-secret-token') {
            return redirect('/home');
        }

        return $next($request);
    }
}
```

如您所見，如果給定的 `token` 與我們的密鑰不匹配，Middleware 將向客戶端返回一個 HTTP 重新導向；否則，請求將進一步傳遞到應用程式中。要將請求更深入地傳遞到應用程式中（允許 Middleware「通過」），您應該使用 `$request` 呼叫 `$next` 回呼。

最好將 Middleware 想像成 HTTP 請求在到達您的應用程式之前必須經過的一系列「層」。每一層都可以檢查請求，甚至完全拒絕它。

> [!NOTE]
> 所有 Middleware 都會透過 [Service Container](/docs/{{version}}/container) 解析，因此您可以在 Middleware 的建構函式中型別提示任何您需要的依賴。

<a name="middleware-and-responses"></a>
#### Middleware 與回應

當然，Middleware 可以在將請求進一步傳遞到應用程式之前或之後執行任務。例如，以下 Middleware 將在請求由應用程式處理**之前**執行某些任務：

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class BeforeMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        // Perform action

        return $next($request);
    }
}
```

然而，這個 Middleware 將在請求由應用程式處理**之後**執行其任務：

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class AfterMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        $response = $next($request);

        // Perform action

        return $response;
    }
}
```

<a name="registering-middleware"></a>
## 註冊 Middleware

<a name="global-middleware"></a>
### 全域 Middleware

如果您希望 Middleware 在應用程式的每個 HTTP 請求期間執行，您可以將其附加到應用程式 `bootstrap/app.php` 檔案中的全域 Middleware 堆疊：

```php
use App\Http\Middleware\EnsureTokenIsValid;

->withMiddleware(function (Middleware $middleware): void {
     $middleware->append(EnsureTokenIsValid::class);
})
```

提供給 `withMiddleware` 閉包的 `$middleware` 物件是 `Illuminate\Foundation\Configuration\Middleware` 的實例，負責管理指派給應用程式路由的 Middleware。`append` 方法將 Middleware 添加到全域 Middleware 列表的末尾。如果您想將 Middleware 添加到列表的開頭，您應該使用 `prepend` 方法。

<a name="manually-managing-laravels-default-global-middleware"></a>
#### 手動管理 Laravel 的預設全域 Middleware

如果您想手動管理 Laravel 的全域 Middleware 堆疊，您可以將 Laravel 預設的全域 Middleware 堆疊提供給 `use` 方法。然後，您可以根據需要調整預設的 Middleware 堆疊：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->use([
        \Illuminate\Foundation\Http\Middleware\InvokeDeferredCallbacks::class,
        // \Illuminate\Http\Middleware\TrustHosts::class,
        \Illuminate\Http\Middleware\TrustProxies::class,
        \Illuminate\Http\Middleware\HandleCors::class,
        \Illuminate\Foundation\Http\Middleware\PreventRequestsDuringMaintenance::class,
        \Illuminate\Http\Middleware\ValidatePostSize::class,
        \Illuminate\Foundation\Http\Middleware\TrimStrings::class,
        \Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull::class,
    ]);
})
```

<a name="assigning-middleware-to-routes"></a>
### 將 Middleware 指派給路由

如果您想將 Middleware 指派給特定的路由，您可以在定義路由時呼叫 `middleware` 方法：

```php
use App\Http\Middleware\EnsureTokenIsValid;

Route::get('/profile', function () {
    // ...
})->middleware(EnsureTokenIsValid::class);
```

您可以透過將 Middleware 名稱陣列傳遞給 `middleware` 方法，為路由指派多個 Middleware：

```php
Route::get('/', function () {
    // ...
})->middleware([First::class, Second::class]);
```

<a name="excluding-middleware"></a>
#### 排除 Middleware

當將 Middleware 指派給一組路由時，您可能偶爾需要阻止 Middleware 應用於該組中的個別路由。您可以使用 `withoutMiddleware` 方法來實現此目的：

```php
use App\Http\Middleware\EnsureTokenIsValid;

Route::middleware([EnsureTokenIsValid::class])->group(function () {
    Route::get('/', function () {
        // ...
    });

    Route::get('/profile', function () {
        // ...
    })->withoutMiddleware([EnsureTokenIsValid::class]);
});
```

您也可以從整個路由定義[群組](/docs/{{version}}/routing#route-groups)中排除給定的一組 Middleware：

```php
use App\Http\Middleware\EnsureTokenIsValid;

Route::withoutMiddleware([EnsureTokenIsValid::class])->group(function () {
    Route::get('/profile', function () {
        // ...
    });
});
```

`withoutMiddleware` 方法只能移除路由 Middleware，不適用於[全域 Middleware](#global-middleware)。

<a name="middleware-groups"></a>
### Middleware 群組

有時您可能希望將多個 Middleware 分組在一個單一的鍵下，以便更容易地將它們指派給路由。您可以使用應用程式 `bootstrap/app.php` 檔案中的 `appendToGroup` 方法來實現此目的：

```php
use App\Http\Middleware\First;
use App\Http\Middleware\Second;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->appendToGroup('group-name', [
        First::class,
        Second::class,
    ]);

    $middleware->prependToGroup('group-name', [
        First::class,
        Second::class,
    ]);
})
```

Middleware 群組可以使用與個別 Middleware 相同的語法指派給路由和控制器動作：

```php
Route::get('/', function () {
    // ...
})->middleware('group-name');

Route::middleware(['group-name'])->group(function () {
    // ...
});
```

<a name="laravels-default-middleware-groups"></a>
#### Laravel 的預設 Middleware 群組

Laravel 包含了預定義的 `web` 和 `api` Middleware 群組，其中包含您可能希望應用於您的 Web 和 API 路由的常用 Middleware。請記住，Laravel 會自動將這些 Middleware 群組應用於相應的 `routes/web.php` 和 `routes/api.php` 檔案：

<div class="overflow-auto">

| `web` Middleware 群組                               |
| --------------------------------------------------------- |
| `Illuminate\Cookie\Middleware\EncryptCookies`             |
| `Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse` |
| `Illuminate\Session\Middleware\StartSession`              |
| `Illuminate\View\Middleware\ShareErrorsFromSession`       |
| `Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` |
| `Illuminate\Routing\Middleware\SubstituteBindings`        |

</div>

<div class="overflow-auto">

| `api` Middleware 群組                              |
| -------------------------------------------------- |
| `Illuminate\Routing\Middleware\SubstituteBindings` |

</div>

如果您想將 Middleware 附加或前置到這些群組，您可以使用應用程式 `bootstrap/app.php` 檔案中的 `web` 和 `api` 方法。`web` 和 `api` 方法是 `appendToGroup` 方法的便捷替代方案：

```php
use App\Http\Middleware\EnsureTokenIsValid;
use App\Http\Middleware\EnsureUserIsSubscribed;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->web(append: [
        EnsureUserIsSubscribed::class,
    ]);

    $middleware->api(prepend: [
        EnsureTokenIsValid::class,
    ]);
})
```

您甚至可以用您自己的自訂 Middleware 替換 Laravel 預設 Middleware 群組中的一個項目：

```php
use App\Http\Middleware\StartCustomSession;
use Illuminate\Session\Middleware\StartSession;

$middleware->web(replace: [
    StartSession::class => StartCustomSession::class,
]);
```

或者，您可以完全移除一個 Middleware：

```php
$middleware->web(remove: [
    StartSession::class,
]);
```

<a name="manually-managing-laravels-default-middleware-groups"></a>
#### 手動管理 Laravel 的預設 Middleware 群組

如果您想手動管理 Laravel 預設 `web` 和 `api` Middleware 群組中的所有 Middleware，您可以完全重新定義這些群組。以下範例將使用其預設 Middleware 定義 `web` 和 `api` Middleware 群組，讓您可以根據需要進行自訂：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->group('web', [
        \Illuminate\Cookie\Middleware\EncryptCookies::class,
        \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
        \Illuminate\Session\Middleware\StartSession::class,
        \Illuminate\View\Middleware\ShareErrorsFromSession::class,
        \Illuminate\Foundation\Http\Middleware\ValidateCsrfToken::class,
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
        // \Illuminate\Session\Middleware\AuthenticateSession::class,
    ]);

    $middleware->group('api', [
        // \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
        // 'throttle:api',
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
    ]);
})
```

> [!NOTE]
> 預設情況下，`web` 和 `api` Middleware 群組會由 `bootstrap/app.php` 檔案自動應用於應用程式相應的 `routes/web.php` 和 `routes/api.php` 檔案。

<a name="middleware-aliases"></a>
### Middleware 別名

您可以在應用程式的 `bootstrap/app.php` 檔案中為 Middleware 指派別名。Middleware 別名允許您為給定的 Middleware 類別定義一個簡短的別名，這對於類別名稱較長的 Middleware 尤其有用：

```php
use App\Http\Middleware\EnsureUserIsSubscribed;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->alias([
        'subscribed' => EnsureUserIsSubscribed::class
    ]);
})
```

一旦 Middleware 別名在應用程式的 `bootstrap/app.php` 檔案中定義，您就可以在將 Middleware 指派給路由時使用該別名：

```php
Route::get('/profile', function () {
    // ...
})->middleware('subscribed');
```

為了方便起見，Laravel 的一些內建 Middleware 預設會被別名化。例如，`auth` Middleware 是 `Illuminate\Auth\Middleware\Authenticate` Middleware 的別名。以下是預設 Middleware 別名的列表：

<
