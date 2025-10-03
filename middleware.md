# 中介層 (Middleware)

- [介紹](#introduction)
- [定義中介層](#defining-middleware)
- [註冊中介層](#registering-middleware)
    - [全域中介層](#global-middleware)
    - [將中介層指派給路由](#assigning-middleware-to-routes)
    - [中介層群組](#middleware-groups)
    - [中介層別名](#middleware-aliases)
    - [排序中介層](#sorting-middleware)
- [中介層參數](#middleware-parameters)
- [可終止中介層](#terminable-middleware)

<a name="introduction"></a>
## 介紹

中介層 (Middleware) 提供了一個方便的機制，用於檢查與過濾進入您應用程式的 HTTP 請求。例如，Laravel 包含了一個中介層，用於驗證您應用程式的使用者是否已驗證。如果使用者未經驗證，中介層會將使用者重新導向到您應用程式的登入畫面。然而，如果使用者已驗證，中介層將允許請求繼續深入應用程式。

除了驗證之外，還可以編寫額外的中介層來執行各種任務。例如，一個記錄中介層可能會記錄所有進入您應用程式的請求。Laravel 中包含了多種中介層，包括用於驗證和 CSRF 防護的中介層；然而，所有使用者定義的中介層通常位於您應用程式的 `app/Http/Middleware` 目錄中。

<a name="defining-middleware"></a>
## 定義中介層

要建立一個新的中介層，請使用 `make:middleware` Artisan 命令：

```shell
php artisan make:middleware EnsureTokenIsValid
```

此命令將在您的 `app/Http/Middleware` 目錄中放置一個新的 `EnsureTokenIsValid` 類別。在此中介層中，我們只允許在提供的 `token` 輸入符合指定值時存取路由。否則，我們將把使用者重新導向回 `/home` URI：

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

如您所見，如果給定的 `token` 不符合我們的秘密 token，中介層將向用戶端返回一個 HTTP 重新導向；否則，請求將會進一步傳遞到應用程式中。要將請求傳遞到應用程式深層 (允許中介層「通過」)，您應該使用 `$request` 呼叫 `$next` 回呼。

最好將中介層想像成一系列 HTTP 請求在抵達您的應用程式之前必須經過的「層」。每一層都可以檢查請求，甚至完全拒絕它。

> [!NOTE]
> 所有中介層皆透過 [服務容器](/docs/{{version}}/container) 解析，因此您可以在中介層的建構式中型別提示所需的任何依賴。

<a name="middleware-and-responses"></a>
#### 中介層與回應

當然，中介層可以在將請求傳遞到應用程式深層之前或之後執行任務。例如，以下中介層將在請求由應用程式處理**之前**執行某些任務：

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

然而，這個中介層將在請求由應用程式處理**之後**執行其任務：

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
## 註冊中介層

<a name="global-middleware"></a>
### 全域中介層

如果您希望中介層在應用程式的每個 HTTP 請求期間執行，您可以將其附加到應用程式的 `bootstrap/app.php` 檔案中的全域中介層堆疊：

```php
use App\Http\Middleware\EnsureTokenIsValid;

->withMiddleware(function (Middleware $middleware): void {
     $middleware->append(EnsureTokenIsValid::class);
})
```

提供給 `withMiddleware` 閉包的 `$middleware` 物件是 `Illuminate\Foundation\Configuration\Middleware` 的一個實例，負責管理指派給應用程式路由的中介層。`append` 方法會將中介層新增到全域中介層清單的末尾。如果您想將中介層新增到清單的開頭，則應使用 `prepend` 方法。

<a name="manually-managing-laravels-default-global-middleware"></a>
#### 手動管理 Laravel 預設的全域中介層

如果您想手動管理 Laravel 的全域中介層堆疊，您可以將 Laravel 預設的全域中介層堆疊提供給 `use` 方法。然後，您可以視需要調整預設的中介層堆疊：

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
### 將中介層指派給路由

如果您想將中介層指派給特定路由，您可以在定義路由時呼叫 `middleware` 方法：

```php
use App\Http\Middleware\EnsureTokenIsValid;

Route::get('/profile', function () {
    // ...
})->middleware(EnsureTokenIsValid::class);
```

您可以透過傳遞一個中介層名稱陣列給 `middleware` 方法，將多個中介層指派給路由：

```php
Route::get('/', function () {
    // ...
})->middleware([First::class, Second::class]);
```

<a name="excluding-middleware"></a>
#### 排除中介層

當將中介層指派給一組路由時，您有時可能需要防止該中介層應用於群組內的個別路由。您可以透過使用 `withoutMiddleware` 方法來達成此目的：

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

您也可以從整個 [群組](/docs/{{version}}/routing#route-groups) 的路由定義中排除一組給定的中介層：

```php
use App\Http\Middleware\EnsureTokenIsValid;

Route::withoutMiddleware([EnsureTokenIsValid::class])->group(function () {
    Route::get('/profile', function () {
        // ...
    });
});
```

`withoutMiddleware` 方法只能移除路由中介層，不適用於 [全域中介層](#global-middleware)。

<a name="middleware-groups"></a>
### 中介層群組

有時您可能希望將多個中介層分組在單一鍵名下，以便更容易將它們指派給路由。您可以透過應用程式的 `bootstrap/app.php` 檔案中的 `appendToGroup` 方法來達成此目的：

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

中介層群組可以使用與個別中介層相同的語法指派給路由和控制器動作：

```php
Route::get('/', function () {
    // ...
})->middleware('group-name');

Route::middleware(['group-name'])->group(function () {
    // ...
});
```

<a name="laravels-default-middleware-groups"></a>
#### Laravel 預設的中介層群組

Laravel 包含預定義的 `web` 和 `api` 中介層群組，這些群組包含您可能希望應用於 web 和 API 路由的常見中介層。請記住，Laravel 會自動將這些中介層群組應用於對應的 `routes/web.php` 和 `routes/api.php` 檔案：

<div class="overflow-auto">

| `web` 中介層群組                                          |
| --------------------------------------------------------- |
| `Illuminate\Cookie\Middleware\EncryptCookies`             |
| `Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse` |
| `Illuminate\Session\Middleware\StartSession`              |
| `Illuminate\View\Middleware\ShareErrorsFromSession`       |
| `Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` |
| `Illuminate\Routing\Middleware\SubstituteBindings`        |

</div>

<div class="overflow-auto">

| `api` 中介層群組                             |
| ------------------------------------------ |
| `Illuminate\Routing\Middleware\SubstituteBindings` |

</div>

如果您想將中介層附加或前置到這些群組，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `web` 和 `api` 方法。`web` 和 `api` 方法是 `appendToGroup` 方法的方便替代方案：

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

您甚至可以將 Laravel 預設中介層群組中的一個項目替換為您自己的客製化中介層：

```php
use App\Http\Middleware\StartCustomSession;
use Illuminate\Session\Middleware\StartSession;

$middleware->web(replace: [
    StartSession::class => StartCustomSession::class,
]);
```

或者，您可以完全移除一個中介層：

```php
$middleware->web(remove: [
    StartSession::class,
]);
```

<a name="manually-managing-laravels-default-middleware-groups"></a>
#### 手動管理 Laravel 預設的中介層群組

如果您想手動管理 Laravel 預設 `web` 和 `api` 中介層群組中的所有中介層，您可以完全重新定義這些群組。下面的範例將使用其預設中介層定義 `web` 和 `api` 中介層群組，讓您能夠視需要進行客製化：

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
> 預設情況下，`web` 和 `api` 中介層群組會透過 `bootstrap/app.php` 檔案自動應用於您的應用程式對應的 `routes/web.php` 和 `routes/api.php` 檔案。

<a name="middleware-aliases"></a>
### 中介層別名

您可以為應用程式的`bootstrap/app.php`檔案中的中介層設定別名。中介層別名允許您為特定的中介層類別定義一個簡短的別名，這對於類別名稱很長的中介層特別有用：

```php
use App\Http\Middleware\EnsureUserIsSubscribed;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->alias([
        'subscribed' => EnsureUserIsSubscribed::class
    ]);
})
```

一旦中介層別名已在您的應用程式的`bootstrap/app.php`檔案中定義，您便可以在將中介層指派給路由時使用該別名：

```php
Route::get('/profile', function () {
    // ...
})->middleware('subscribed');
```

為了方便起見，Laravel 的一些內建中介層預設是設定別名的。例如，`auth` 中介層是 `Illuminate\Auth\Middleware\Authenticate` 中介層的別名。以下是預設中介層別名的清單：

<div class="overflow-auto">

| 別名              | 中介層                                                                                                    |
| ------------------ | ------------------------------------------------------------------------------------------------------------- |
| `auth`             | `Illuminate\Auth\Middleware\Authenticate`                                                                     |
| `auth.basic`       | `Illuminate\Auth\Middleware\AuthenticateWithBasicAuth`                                                        |
| `auth.session`     | `Illuminate\Session\Middleware\AuthenticateSession`                                                           |
| `cache.headers`    | `Illuminate\Http\Middleware\SetCacheHeaders`                                                                  |
| `can`              | `Illuminate\Auth\Middleware\Authorize`                                                                        |
| `guest`            | `Illuminate\Auth\Middleware\RedirectIfAuthenticated`                                                          |
| `password.confirm` | `Illuminate\Auth\Middleware\RequirePassword`                                                                  |
| `precognitive`     | `Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests`                                            |
| `signed`           | `Illuminate\Routing\Middleware\ValidateSignature`                                                             |
| `subscribed`       | `\Spark\Http\Middleware\VerifyBillableIsSubscribed`                                                           |
| `throttle`         | `Illuminate\Routing\Middleware\ThrottleRequests` or `Illuminate\Routing\Middleware\ThrottleRequestsWithRedis` |
| `verified`         | `Illuminate\Auth\Middleware\EnsureEmailIsVerified`                                                            |

</div>

<a name="sorting-middleware"></a>
### 排序中介層

在少數情況下，您可能需要中介層以特定的順序執行，但在將它們指派給路由時無法控制其順序。在這些情況下，您可以在應用程式的`bootstrap/app.php`檔案中使用`priority`方法來指定中介層的優先權：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->priority([
        \Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests::class,
        \Illuminate\Cookie\Middleware\EncryptCookies::class,
        \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
        \Illuminate\Session\Middleware\StartSession::class,
        \Illuminate\View\Middleware\ShareErrorsFromSession::class,
        \Illuminate\Foundation\Http\Middleware\ValidateCsrfToken::class,
        \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
        \Illuminate\Routing\Middleware\ThrottleRequests::class,
        \Illuminate\Routing\Middleware\ThrottleRequestsWithRedis::class,
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
        \Illuminate\Contracts\Auth\Middleware\AuthenticatesRequests::class,
        \Illuminate\Auth\Middleware\Authorize::class,
    ]);
})
```

<a name="middleware-parameters"></a>
## 中介層參數

中介層也能接收額外的參數。例如，如果你的應用程式需要在執行特定動作前，驗證已驗證使用者是否具有特定「角色」，你可以建立一個 `EnsureUserHasRole` middleware，它會接收一個角色名稱作為額外引數。

額外的 middleware 參數會在 `$next` 引數之後傳遞給 middleware：

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureUserHasRole
{
    /**
     * Handle an incoming request.
     *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next, string $role): Response
    {
        if (! $request->user()->hasRole($role)) {
            // Redirect...
        }

        return $next($request);
    }
}
```

可以在定義路由時指定 middleware 參數，方式是使用 `:` 分隔 middleware 名稱和參數：

```php
use App\Http\Middleware\EnsureUserHasRole;

Route::put('/post/{id}', function (string $id) {
    // ...
})->middleware(EnsureUserHasRole::class.':editor');
```

多個參數可以使用逗號分隔：

```php
Route::put('/post/{id}', function (string $id) {
    // ...
})->middleware(EnsureUserHasRole::class.':editor,publisher');
```

<a name="terminable-middleware"></a>
## 可終止中介層

有時 middleware 可能需要在 HTTP 回應傳送至瀏覽器後執行某些工作。如果你在 middleware 上定義了 `terminate` 方法，且你的網頁伺服器正在使用 [FastCGI](https://www.php.net/manual/en/install.fpm.php)，則 `terminate` 方法將在回應傳送至瀏覽器後自動被呼叫：

```php
<?php

namespace Illuminate\Session\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class TerminatingMiddleware
{
    /**
     * Handle an incoming request.
     *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next): Response
    {
        return $next($request);
    }

    /**
     * Handle tasks after the response has been sent to the browser.
     */
    public function terminate(Request $request, Response $response): void
    {
        // ...
    }
}
```

`terminate` 方法應同時接收請求 (request) 和回應 (response)。一旦你定義了可終止的 middleware，就應該將其加入到應用程式的 `bootstrap/app.php` 檔案中，作為路由或全域 middleware 的列表。

當呼叫 middleware 上的 `terminate` 方法時，Laravel 將從 [服務容器](/docs/{{version}}/container) 中解析出一個全新的 middleware 實例。如果你希望在呼叫 `handle` 和 `terminate` 方法時使用相同的 middleware 實例，請使用容器的 `singleton` 方法註冊該 middleware。通常這應該在你的 `AppServiceProvider` 的 `register` 方法中完成：

```php
use App\Http\Middleware\TerminatingMiddleware;

/**
 * Register any application services.
 */
public function register(): void
{
    $this->app->singleton(TerminatingMiddleware::class);
}
```