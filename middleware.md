# 中介層

- [簡介](#introduction)
- [定義中介層](#defining-middleware)
- [註冊中介層](#registering-middleware)
    - [全域中介層](#global-middleware)
    - [將中介層分配至路由](#assigning-middleware-to-routes)
    - [中介層群組](#middleware-groups)
    - [中介層別名](#middleware-aliases)
    - [中介層排序](#sorting-middleware)
- [中介層參數](#middleware-parameters)
- [可終止的中介層](#terminable-middleware)

<a name="introduction"></a>
## 簡介

中介層為檢查與篩選進入應用程式的 HTTP 請求提供了一種便捷的機制。例如，Laravel 包含一個用於驗證應用程式使用者是否已通過認證的中介層。如果使用者尚未通過認證，該中介層將會將使用者重新導向至應用程式的登入畫面。然而，如果使用者已通過認證，該中介層將允許請求繼續進入應用程式。

除了認證之外，您還可以編寫額外的中介層來執行各種任務。例如，日誌中介層可能會記錄所有進入應用程式的請求。Laravel 內建了許多中介層，包括用於認證和 CSRF 保護的中介層；然而，所有使用者定義的中介層通常都位於應用程式的 `app/Http/Middleware` 目錄中。


<a name="defining-middleware"></a>
## 定義中介層

若要建立新的中介層，請使用 `make:middleware` Artisan 指令：

```shell
php artisan make:middleware EnsureTokenIsValid
```

此指令會在您的 `app/Http/Middleware` 目錄中放置一個新的 `EnsureTokenIsValid` 類別。在此中介層中，只有當提供的 `token` 輸入與指定值匹配時，我們才允許存取該路由。否則，我們將會將使用者重新導向回 `/home` URI：

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

如您所見，如果給定的 `token` 與我們的私密令牌不匹配，中介層將向客戶端返回一個 HTTP 重新導向；否則，請求將被進一步傳遞到應用程式中。若要將請求傳遞到應用程式更深層（允許中介層「通過」），您應該使用 `$request` 調用 `$next` 回呼函數。

最好將中介層想像成一系列 HTTP 請求在到達應用程式之前必須通過的「層」。每一層都可以檢查請求，甚至完全拒絕該請求。

> [!NOTE]
> 所有中介層都透過 [服務容器(service container)](/docs/{{version}}/container) 解析，因此您可以在中介層的建構子中對任何所需的依賴項進行型別提示 (type-hint)。


<a name="middleware-and-responses"></a>
#### 中介層與回應

當然，中介層可以在將請求傳遞到應用程式深層之前或之後執行任務。例如，以下中介層會在請求被應用程式處理「之前」執行某些任務：

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

然而，此中介層會在請求被應用程式處理「之後」才執行其任務：

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

如果您希望某個中介層在每次進入應用程式的 HTTP 請求期間執行，您可以將其附加到應用程式 `bootstrap/app.php` 檔案中的全域中介層堆疊：

```php
use App\Http\Middleware\EnsureTokenIsValid;

->withMiddleware(function (Middleware $middleware): void {
     $middleware->append(EnsureTokenIsValid::class);
})
```

提供給 `withMiddleware` 閉包的 `$middleware` 物件是一個 `Illuminate\Foundation\Configuration\Middleware` 的實例，負責管理分配給應用程式路由的中介層。`append` 方法會將中介層添加到全域中介層列表的末尾。如果您想將中介層添加到列表的開頭，則應該使用 `prepend` 方法。


<a name="manually-managing-laravels-default-global-middleware"></a>
#### 手動管理 Laravel 的預設全域中介層

如果您想手動管理 Laravel 的全域中介層堆疊，您可以將 Laravel 預設的全域中介層堆疊提供給 `use` 方法。接著，您可以根據需要調整預設的中介層堆疊：

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
### 將中介層分配至路由

如果您想將中介層分配給特定的路由，可以在定義路由時呼叫 `middleware` 方法：

```php
use App\Http\Middleware\EnsureTokenIsValid;

Route::get('/profile', function () {
    // ...
})->middleware(EnsureTokenIsValid::class);
```

您可以透過將中介層名稱的陣列傳遞給 `middleware` 方法，來為該路由分配多個中介層：

```php
Route::get('/', function () {
    // ...
})->middleware([First::class, Second::class]);
```


<a name="excluding-middleware"></a>
#### 排除中介層

在將中介層分配給一組路由時，您可能偶爾需要防止該中介層被應用於群組中的單一路由。您可以使用 `withoutMiddleware` 方法來達成此目的：

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

您也可以從整個路由定義 [群組](/docs/{{version}}/routing#route-groups) 中排除一組特定的中介層：

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

有時您可能想要將多個中介層分在單一鍵值下，以便更輕鬆地將其分配給路由。您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `appendToGroup` 方法來實現：

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

中介層群組可以使用與單一中介層相同的語法分配給路由和控制器動作：

```php
Route::get('/', function () {
    // ...
})->middleware('group-name');

Route::middleware(['group-name'])->group(function () {
    // ...
});
```


<a name="laravels-default-middleware-groups"></a>
#### Laravel 的預設中介層群組

Laravel 包含了預定義的 `web` 和 `api` 中介層群組，其中包含您可能想應用到 Web 和 API 路由的常用中介層。請記住，Laravel 會自動將這些中介層群組應用到對應的 `routes/web.php` 和 `routes/api.php` 檔案：

<div class="overflow-auto">

| `web` 中介層群組                                      |
| --------------------------------------------------------- |
| `Illuminate\Cookie\Middleware\EncryptCookies`             |
| `Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse` |
| `Illuminate\Session\Middleware\StartSession`              |
| `Illuminate\View\Middleware\ShareErrorsFromSession`       |
| `Illuminate\Foundation\Http\Middleware\PreventRequestForgery` |
| `Illuminate\Routing\Middleware\SubstituteBindings`        |

</div>

<div class="overflow-auto">

| `api` 中介層群組                                    |
| -------------------------------------------------- |
| `Illuminate\Routing\Middleware\SubstituteBindings` |

</div>

如果您想將中介層附加 (append) 或前置 (prepend) 到這些群組中，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `web` 和 `api` 方法。`web` 和 `api` 方法是 `appendToGroup` 方法的便捷替代方案：

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

您甚至可以用自定義的中介層替換 Laravel 預設中介層群組中的其中一項：

```php
use App\Http\Middleware\StartCustomSession;
use Illuminate\Session\Middleware\StartSession;

$middleware->web(replace: [
    StartSession::class => StartCustomSession::class,
]);
```

或者，您可以完全移除某個中介層：

```php
$middleware->web(remove: [
    StartSession::class,
]);
```


<a name="manually-managing-laravels-default-middleware-groups"></a>
#### 手動管理 Laravel 的預設中介層群組

如果您想手動管理 Laravel 預設 `web` 和 `api` 中介層群組中的所有中介層，您可以完全重新定義這些群組。下面的範例將使用其預設中介層來定義 `web` 和 `api` 中介層群組，讓您可以根據需要進行自定義：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->group('web', [
        \Illuminate\Cookie\Middleware\EncryptCookies::class,
        \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
        \Illuminate\Session\Middleware\StartSession::class,
        \Illuminate\View\Middleware\ShareErrorsFromSession::class,
        \Illuminate\Foundation\Http\Middleware\PreventRequestForgery::class,
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
> 預設情況下，`web` 和 `api` 中介層群組會由 `bootstrap/app.php` 檔案自動應用到應用程式對應的 `routes/web.php` 和 `routes/api.php` 檔案中。

<a name="middleware-aliases"></a>
### 中介層別名

您可以在應用程式的 `bootstrap/app.php` 檔案中為中介層分配別名。中介層別名允許您為特定的中介層類別定義一個簡短的別名，這對於類別名稱較長的中介層特別有用：

```php
use App\Http\Middleware\EnsureUserIsSubscribed;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->alias([
        'subscribed' => EnsureUserIsSubscribed::class
    ]);
})
```

一旦在應用程式的 `bootstrap/app.php` 檔案中定義了中介層別名，您就可以在將中介層分配至路由時使用該別名：

```php
Route::get('/profile', function () {
    // ...
})->middleware('subscribed');
```

為了方便起見，Laravel 的某些內建中介層預設就已經有了別名。例如，`auth` 中介層就是 `Illuminate\Auth\Middleware\Authenticate` 中介層的別名。以下是預設中介層別名的列表：

<div class="overflow-auto">

| 別名 | 中介層 |
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
### 中介層排序

在極少數情況下，您可能需要中介層以特定的順序執行，但在將其分配至路由時無法控制其順序。在這些情況下，您可以使用應用程式 `bootstrap/app.php` 檔案中的 `priority` 方法來指定中介層的優先權：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->priority([
        \Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests::class,
        \Illuminate\Cookie\Middleware\EncryptCookies::class,
        \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
        \Illuminate\Session\Middleware\StartSession::class,
        \Illuminate\View\Middleware\ShareErrorsFromSession::class,
        \Illuminate\Foundation\Http\Middleware\PreventRequestForgery::class,
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

中介層也可以接收額外的參數。例如，如果您的應用程式需要在執行特定動作前，驗證已認證的使用者是否具有特定的「角色(role)」，您可以建立一個 `EnsureUserHasRole` 中介層，將角色名稱作為額外的引數接收。

額外的中介層參數將在 `$next` 引數之後傳遞給中介層：

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

定義路由時，可以使用 `:` 將中介層名稱與參數分隔開，來指定中介層參數：

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
## 可終止的中介層

有時候中介層需要在 HTTP 回應傳送到瀏覽器之後執行某些工作。如果您在中介層中定義了 `terminate` 方法，且您的網頁伺服器使用 [FastCGI](https://www.php.net/manual/en/install.fpm.php)，則 `terminate` 方法會在回應傳送到瀏覽器後自動被呼叫：

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

`terminate` 方法應該同時接收請求與回應。定義好可終止的中介層後，您應該將其新增至應用程式 `bootstrap/app.php` 檔案中的路由列表或全域中介層中。

在呼叫中介層的 `terminate` 方法時，Laravel 會從 [服務容器](/docs/{{version}}/container) 中解析出一個新的中介層實例。如果您希望在呼叫 `handle` 與 `terminate` 方法時使用同一個中介層實例，請使用容器的 `singleton` 方法將該中介層註冊至容器中。通常這應該在 `AppServiceProvider` 的 `register` 方法中完成：

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