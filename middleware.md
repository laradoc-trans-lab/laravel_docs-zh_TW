# 中介層

- [簡介](#introduction)
- [定義中介層](#defining-middleware)
- [註冊中介層](#registering-middleware)
    - [全域中介層](#global-middleware)
    - [指派中介層給路由](#assigning-middleware-to-routes)
    - [中介層群組](#middleware-groups)
    - [中介層別名](#middleware-aliases)
    - [排序中介層](#sorting-middleware)
- [中介層參數](#middleware-parameters)
- [可終結的中介層](#terminable-middleware)

<a name="introduction"></a>
## 簡介

中介層提供了一種方便的機制，用於檢查與過濾進入應用程式的 HTTP 請求。例如，Laravel 包含了一個中介層，用來驗證應用程式的使用者是否已通過認證。如果使用者未通過認證，該中介層會將使用者重新導向至應用程式的登入畫面。然而，如果使用者已通過認證，該中介層則會允許請求繼續進入應用程式。

除了認證之外，您還可以撰寫額外之中介層來執行各種不同的任務。例如，記錄日誌的中介層可以記錄所有進入應用程式的請求。Laravel 內建了多種中介層，包括用於認證與 CSRF 保護的中介層；不過，所有由使用者自訂的中介層通常都會放置在應用程式的 `app/Http/Middleware` 目錄中。


<a name="defining-middleware"></a>
## 定義中介層

若要建立新的中介層，請使用 `make:middleware` Artisan 指令：

```shell
php artisan make:middleware EnsureTokenIsValid
```

此指令會在您的 `app/Http/Middleware` 目錄中放置一個新的 `EnsureTokenIsValid` 類別。在這個中介層中，只有當傳入的 `token` 輸入值符合特定數值時，我們才允許存取該路由。否則，我們將把使用者重新導向回 `/home` URI：

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

如您所見，如果給定的 `token` 與我們的私密令牌不符，中介層將向用戶端傳回 HTTP 重新導向；否則，請求將被進一步傳遞到應用程式中。若要將請求進一步傳遞至應用程式更深處（允許中介層「通過」），您應該傳入 `$request` 並呼叫 `$next` 回呼函式。

最好將中介層想像為 HTTP 請求在到達您的應用程式之前必須穿過的一系列「層」。每一層都可以檢查請求，甚至可以完全拒絕請求。

> [!NOTE]
> 所有中介層都是透過[服務容器](/docs/{{version}}/container)來解析的，因此您可以在中介層的建構函式中，為所需的任何依賴進行型別提示 (Type-hint)。


<a name="middleware-and-responses"></a>
#### 中介層與回應

當然，中介層可以在將請求進一步傳遞給應用程式之前或之後執行任務。例如，以下中介層會在應用程式處理請求**之前**執行某些任務：

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

然而，這個中介層會在應用程式處理請求**之後**執行其任務：

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

如果你希望中介層在進入應用程式的每個 HTTP 請求期間都執行，可以將其附加到應用程式 `bootstrap/app.php` 檔案中的全域中介層堆疊：

```php
use App\Http\Middleware\EnsureTokenIsValid;

->withMiddleware(function (Middleware $middleware): void {
     $middleware->append(EnsureTokenIsValid::class);
})
```

傳遞給 `withMiddleware` 閉包的 `$middleware` 物件是 `Illuminate\Foundation\Configuration\Middleware` 的實例，負責管理指派給應用程式路由的中介層。`append` 方法會將該中介層新增到全域中介層清單的末端。若你想將中介層新增到清單開頭，應該使用 `prepend` 方法。

<a name="manually-managing-laravels-default-global-middleware"></a>
#### 手動管理 Laravel 的預設全域中介層

如果你想手動管理 Laravel 的全域中介層堆疊，可以將 Laravel 的預設全域中介層堆疊傳遞給 `use` 方法。接著，你便可以根據需要調整預設的中介層堆疊：

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
### 指派中介層給路由

如果你想將中介層指派給特定路由，可以在定義路由時呼叫 `middleware` 方法：

```php
use App\Http\Middleware\EnsureTokenIsValid;

Route::get('/profile', function () {
    // ...
})->middleware(EnsureTokenIsValid::class);
```

你可以透過將中介層名稱陣列傳遞給 `middleware` 方法，來將多個中介層指派給該路由：

```php
Route::get('/', function () {
    // ...
})->middleware([First::class, Second::class]);
```

<a name="excluding-middleware"></a>
#### 排除中介層

當指派中介層給一組路由群組時，你偶爾可能會需要防止該中介層套用到群組內的特定個別路由。你可以使用 `withoutMiddleware` 方法來達成此目的：

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

你也可以從整個路由定義[群組](/docs/{{version}}/routing#route-groups)中排除一組指定的中介層：

```php
use App\Http\Middleware\EnsureTokenIsValid;

Route::withoutMiddleware([EnsureTokenIsValid::class])->group(function () {
    Route::get('/profile', function () {
        // ...
    });
});
```

`withoutMiddleware` 方法只能移除路由中介層，不適用於[全域中介層](#global-middleware)。

<a name="middleware-groups"></a>
### 中介層群組

有時你可能希望將幾個中介層歸類在單一鍵名之下，以便更容易指派給路由。你可以透過應用程式 `bootstrap/app.php` 檔案中的 `appendToGroup` 方法來達成：

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

中介層群組可以使用與單一中介層相同的語法，指派給路由和控制器動作：

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

Laravel 包含了預先定義的 `web` 與 `api` 中介層群組，其中含有你可能會想要套用到 Web 與 API 路由的常見中介層。請記住，Laravel 會自動將這些中介層群組套用到相對應的 `routes/web.php` 與 `routes/api.php` 檔案：

<div class="overflow-auto">

| `web` 中介層群組 |
| --------------------------------------------------------- |
| `Illuminate\Cookie\Middleware\EncryptCookies`             |
| `Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse` |
| `Illuminate\Session\Middleware\StartSession`              |
| `Illuminate\View\Middleware\ShareErrorsFromSession`       |
| `Illuminate\Foundation\Http\Middleware\PreventRequestForgery` |
| `Illuminate\Routing\Middleware\SubstituteBindings`        |

</div>

<div class="overflow-auto">

| `api` 中介層群組 |
| -------------------------------------------------- |
| `Illuminate\Routing\Middleware\SubstituteBindings` |

</div>

如果你想要在這些群組中附加或預置中介層，可以使用應用程式 `bootstrap/app.php` 檔案中的 `web` 與 `api` 方法。`web` 與 `api` 方法是 `appendToGroup` 方法的便利替代方案：

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

你甚至可以使用自己的自訂中介層來替換 Laravel 預設中介層群組中的某個項目：

```php
use App\Http\Middleware\StartCustomSession;
use Illuminate\Session\Middleware\StartSession;

$middleware->web(replace: [
    StartSession::class => StartCustomSession::class,
]);
```

或者，你可以完全移除某個中介層：

```php
$middleware->web(remove: [
    StartSession::class,
]);
```

<a name="manually-managing-laravels-default-middleware-groups"></a>
#### 手動管理 Laravel 的預設中介層群組

如果你想手動管理 Laravel 預設 `web` 與 `api` 中介層群組內的所有中介層，你可以完全重新定義這些群組。下面的範例將使用其預設中介層來定義 `web` 與 `api` 中介層群組，讓你能夠根據需要進行自訂：

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
> 預設情況下，`bootstrap/app.php` 檔案會自動將 `web` 與 `api` 中介層群組套用到應用程式相對應的 `routes/web.php` 與 `routes/api.php` 檔案中。

<a name="middleware-aliases"></a>
### 中介層別名

您可以在應用程式的 `bootstrap/app.php` 檔案中為中介層指派別名。中介層別名讓您可以為指定的中介層類別定義簡短的別名，這對於類別名稱較長的中介層特別有用：

```php
use App\Http\Middleware\EnsureUserIsSubscribed;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->alias([
        'subscribed' => EnsureUserIsSubscribed::class
    ]);
})
```

一旦在應用程式的 `bootstrap/app.php` 檔案中定義了中介層別名，您就可以在將中介層指派給路由時使用該別名：

```php
Route::get('/profile', function () {
    // ...
})->middleware('subscribed');
```

為方便起見，Laravel 的某些內建中介層預設已設定別名。例如，`auth` 中介層就是 `Illuminate\Auth\Middleware\Authenticate` 中介層的別名。以下是預設的中介層別名列表：

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
| `throttle`         | `Illuminate\Routing\Middleware\ThrottleRequests` 或 `Illuminate\Routing\Middleware\ThrottleRequestsWithRedis` |
| `verified`         | `Illuminate\Auth\Middleware\EnsureEmailIsVerified`                                                            |

</div>

<a name="sorting-middleware"></a>
### 排序中介層

在極少數情況下，您可能需要中介層按特定順序執行，但在指派給路由時卻無法控制它們的順序。在這些情況下，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `priority` 方法來指定中介層的優先順序：

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

如果您想在不替換現有優先順序清單的情況下將中介層新增至其中，可以使用 `prependToPriorityList` 或 `appendToPriorityList` 方法。`prependToPriorityList` 方法會在另一個中介層之前插入指定的中介層，而 `appendToPriorityList` 方法則會在另一個中介層之後插入：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->prependToPriorityList(
        before: \Illuminate\Routing\Middleware\SubstituteBindings::class,
        prepend: \App\Http\Middleware\EnsureTokenIsValid::class,
    );

    $middleware->appendToPriorityList(
        after: \Illuminate\Routing\Middleware\SubstituteBindings::class,
        append: \App\Http\Middleware\EnsureUserIsSubscribed::class,
    );
})
```

`before` 和 `after` 引數也可以是中介層類別的陣列。

<a name="middleware-parameters"></a>
## 中介層參數

中介層也可以接收額外的參數。例如，若您的應用程式需要在執行特定動作前，驗證已認證的使用者是否具有指定的「角色」，您可以建立一個接收角色名稱作為額外引數的 `EnsureUserHasRole` 中介層。

額外的中介層參數會在 `$next` 引數之後傳遞給中介層：

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

在中介層名稱與參數之間加上 `:`，即可在定義路由時指定中介層參數：

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
## 可終結的中介層

有時候，中介層可能需要在 HTTP 回應傳送到瀏覽器之後執行一些工作。如果您在中介層上定義了 `terminate` 方法，且您的 Web 伺服器使用的是 [FastCGI](https://www.php.net/manual/en/install.fpm.php)，則 `terminate` 方法將會在回應傳送到瀏覽器後自動被呼叫：

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

`terminate` 方法應該同時接收請求與回應。定義好可終結的中介層後，您應該將其加入應用程式 `bootstrap/app.php` 檔案中的路由或全域中介層列表中。

當呼叫中介層的 `terminate` 方法時，Laravel 會從[服務容器](/docs/{{version}}/container)解析出該中介層的全新執行個體。如果您希望在呼叫 `handle` 與 `terminate` 方法時使用相同的控制執行個體，請使用容器的 `singleton` 方法將中介層註冊至容器中。這通常應該在 `AppServiceProvider` 的 `register` 方法中進行：

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