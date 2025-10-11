# 中介層

- [簡介](#introduction)
- [定義中介層](#defining-middleware)
- [註冊中介層](#registering-middleware)
    - [全域中介層](#global-middleware)
    - [將中介層指派給路由](#assigning-middleware-to-routes)
    - [中介層群組](#middleware-groups)
    - [中介層別名](#middleware-aliases)
    - [排序中介層](#sorting-middleware)
- [中介層參數](#middleware-parameters)
- [可終止的中介層](#terminable-middleware)

<a name="introduction"></a>
## 簡介

中介層提供了一個便利的機制，用於檢查和過濾進入您應用程式的 HTTP 請求。例如，Laravel 包含一個中介層，用於驗證您應用程式的使用者是否已通過身份驗證。如果使用者未經身份驗證，該中介層會將使用者重導向到您應用程式的登入畫面。然而，如果使用者已通過身份驗證，中介層將允許請求進一步進入應用程式。

除了身份驗證之外，還可以編寫額外的中介層來執行各種任務。例如，一個日誌中介層可能會記錄所有進入您應用程式的請求。Laravel 包含了多種中介層，包括用於身份驗證和 CSRF 防護的中介層；然而，所有使用者定義的中介層通常都位於您應用程式的 `app/Http/Middleware` 目錄中。


<a name="defining-middleware"></a>
## 定義中介層

要建立一個新的中介層，請使用 `make:middleware` Artisan 命令：

```shell
php artisan make:middleware EnsureTokenIsValid
```

這個命令會在您的 `app/Http/Middleware` 目錄中建立一個新的 `EnsureTokenIsValid` 類別。在這個中介層中，我們只允許在提供的 `token` 輸入與指定值匹配時，才允許訪問該路由。否則，我們會將使用者重導向回 `/home` URI：

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

如您所見，如果給定的 `token` 不符合我們的密鑰，中介層將會向客戶端返回一個 HTTP 重導向；否則，請求將會進一步傳遞到應用程式中。要將請求傳遞到應用程式更深層（允許中介層「通過」），您應該使用 `$request` 呼叫 `$next` 回呼函式。

最好將中介層想像成一系列的「層」，HTTP 請求必須先通過這些層，才能到達您的應用程式。每一層都可以檢查請求，甚至完全拒絕它。

> [!NOTE]  
> 所有中介層都透過 [service container](/docs/{{version}}/container) 解析，因此您可以在中介層的建構函式中型別提示任何所需的依賴項。


<a name="middleware-and-responses"></a>
#### 中介層與回應

當然，中介層可以在將請求傳遞到應用程式更深層之前或之後執行任務。例如，以下中介層將會在請求被應用程式處理**之前**執行一些任務：

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

然而，這個中介層將會在請求被應用程式處理**之後**執行其任務：

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

<a name="registering-middleware"></a>
## 註冊中介層

<a name="global-middleware"></a>
### 全域中介層

若您想讓中介層在應用程式的每個 HTTP 請求期間執行，您可以將它附加到應用程式 `bootstrap/app.php` 檔案中的全域中介層堆疊：

    use App\Http\Middleware\EnsureTokenIsValid;

    ->withMiddleware(function (Middleware $middleware) {
         $middleware->append(EnsureTokenIsValid::class);
    })

提供給 `withMiddleware` 閉包的 `$middleware` 物件是 `Illuminate\Foundation\Configuration\Middleware` 的實例，負責管理指派給應用程式路由的中介層。`append` 方法會將中介層新增到全域中介層列表的末尾。若您想將中介層新增到列表的開頭，則應使用 `prepend` 方法。

<a name="manually-managing-laravels-default-global-middleware"></a>
#### 手動管理 Laravel 預設的全域中介層

若您想手動管理 Laravel 的全域中介層堆疊，您可以將 Laravel 預設的全域中介層堆疊提供給 `use` 方法。然後，您可以視需要調整預設的中介層堆疊：

    ->withMiddleware(function (Middleware $middleware) {
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

<a name="assigning-middleware-to-routes"></a>
### 將中介層指派給路由

若您想將中介層指派給特定路由，您可以在定義路由時呼叫 `middleware` 方法：

    use App\Http\Middleware\EnsureTokenIsValid;

    Route::get('/profile', function () {
        // ...
    })->middleware(EnsureTokenIsValid::class);

您可以透過向 `middleware` 方法傳遞中介層名稱陣列，將多個中介層指派給路由：

    Route::get('/', function () {
        // ...
    })->middleware([First::class, Second::class]);

<a name="excluding-middleware"></a>
#### 排除中介層

當將中介層指派給路由群組時，您可能偶爾需要阻止該中介層應用於群組中的個別路由。您可以使用 `withoutMiddleware` 方法來達成此目的：

    use App\Http\Middleware\EnsureTokenIsValid;

    Route::middleware([EnsureTokenIsValid::class])->group(function () {
        Route::get('/', function () {
            // ...
        });

        Route::get('/profile', function () {
            // ...
        })->withoutMiddleware([EnsureTokenIsValid::class]);
    });

您也可以從整個 [路由群組](/docs/{{version}}/routing#route-groups) 定義中排除給定的一組中介層：

    use App\Http\Middleware\EnsureTokenIsValid;

    Route::withoutMiddleware([EnsureTokenIsValid::class])->group(function () {
        Route::get('/profile', function () {
            // ...
        });
    });

`withoutMiddleware` 方法只能移除路由中介層，不適用於 [全域中介層](#global-middleware)。

<a name="middleware-groups"></a>
### 中介層群組

有時您可能希望將幾個中介層分組在單一鍵下，以使其更容易指派給路由。您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `appendToGroup` 方法來達成此目的：

    use App\Http\Middleware\First;
    use App\Http\Middleware\Second;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->appendToGroup('group-name', [
            First::class,
            Second::class,
        ]);

        $middleware->prependToGroup('group-name', [
            First::class,
            Second::class,
        ]);
    })

中介層群組可以使用與個別中介層相同的語法指派給路由和控制器動作：

    Route::get('/', function () {
        // ...
    })->middleware('group-name');

    Route::middleware(['group-name'])->group(function () {
        // ...
    });

<a name="laravels-default-middleware-groups"></a>
#### Laravel 預設的中介層群組

Laravel 包含預定義的 `web` 和 `api` 中介層群組，其中包含您可能想應用於 Web 和 API 路由的常用中介層。請記住，Laravel 會自動將這些中介層群組應用於相對應的 `routes/web.php` 和 `routes/api.php` 檔案：

<div class="overflow-auto">

| `web` 中介層群組 |
| --- |
| `Illuminate\Cookie\Middleware\EncryptCookies` |
| `Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse` |
| `Illuminate\Session\Middleware\StartSession` |
| `Illuminate\View\Middleware\ShareErrorsFromSession` |
| `Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` |
| `Illuminate\Routing\Middleware\SubstituteBindings` |

</div>

<div class="overflow-auto">

| `api` 中介層群組 |
| --- |
| `Illuminate\Routing\Middleware\SubstituteBindings` |

</div>

若您想在這些群組中附加 (append) 或前置 (prepend) 中介層，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `web` 和 `api` 方法。`web` 和 `api` 方法是 `appendToGroup` 方法的便捷替代方案：

    use App\Http\Middleware\EnsureTokenIsValid;
    use App\Http\Middleware\EnsureUserIsSubscribed;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->web(append: [
            EnsureUserIsSubscribed::class,
        ]);

        $middleware->api(prepend: [
            EnsureTokenIsValid::class,
        ]);
    })

您甚至可以將 Laravel 預設中介層群組的其中一個項目替換為您自己的自訂中介層：

    use App\Http\Middleware\StartCustomSession;
    use Illuminate\Session\Middleware\StartSession;

    $middleware->web(replace: [
        StartSession::class => StartCustomSession::class,
    ]);

或者，您可以完全移除一個中介層：

    $middleware->web(remove: [
        StartSession::class,
    ]);

<a name="manually-managing-laravels-default-middleware-groups"></a>
#### 手動管理 Laravel 預設的中介層群組

若您想手動管理 Laravel 預設 `web` 和 `api` 中介層群組中的所有中介層，您可以完全重新定義這些群組。下面的範例將定義 `web` 和 `api` 中介層群組及其預設中介層，讓您可以視需要自訂它們：

    ->withMiddleware(function (Middleware $middleware) {
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

> [!NOTE]  
> 預設情況下，`web` 和 `api` 中介層群組會透過 `bootstrap/app.php` 檔案自動應用於您的應用程式相對應的 `routes/web.php` 和 `routes/api.php` 檔案。

<a name="middleware-aliases"></a>
### 中介層別名

您可以在應用程式的 `bootstrap/app.php` 檔案中，為中介層指派別名。中介層別名讓您可以為特定中介層類別定義一個簡短的別名，這對於名稱冗長的類別特別有用：

    use App\Http\Middleware\EnsureUserIsSubscribed;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->alias([
            'subscribed' => EnsureUserIsSubscribed::class
        ]);
    })

一旦中介層別名在應用程式的 `bootstrap/app.php` 檔案中定義後，您就可以在將中介層指派給路由時使用該別名：

    Route::get('/profile', function () {
        // ...
    })->middleware('subscribed');

為方便起見，Laravel 的一些內建中介層預設已設定別名。例如，`auth` 中介層是 `Illuminate\Auth\Middleware\Authenticate` 中介層的別名。以下是預設中介層別名的列表：

<div class="overflow-auto">

| 別名 | 中介層 |
| --- | --- |
| `auth` | `Illuminate\Auth\Middleware\Authenticate` |
| `auth.basic` | `Illuminate\Auth\Middleware\AuthenticateWithBasicAuth` |
| `auth.session` | `Illuminate\Session\Middleware\AuthenticateSession` |
| `cache.headers` | `Illuminate\Http\Middleware\SetCacheHeaders` |
| `can` | `Illuminate\Auth\Middleware\Authorize` |
| `guest` | `Illuminate\Auth\Middleware\RedirectIfAuthenticated` |
| `password.confirm` | `Illuminate\Auth\Middleware\RequirePassword` |
| `precognitive` | `Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests` |
| `signed` | `Illuminate\Routing\Middleware\ValidateSignature` |
| `subscribed` | `\Spark\Http\Middleware\VerifyBillableIsSubscribed` |
| `throttle` | `Illuminate\Routing\Middleware\ThrottleRequests` or `Illuminate\Routing\Middleware\ThrottleRequestsWithRedis` |
| `verified` | `Illuminate\Auth\Middleware\EnsureEmailIsVerified` |

</div>

<a name="sorting-middleware"></a>
### 排序中介層

在極少數情況下，您可能會需要中介層以特定順序執行，但在將它們指派給路由時，卻無法控制它們的順序。在這些情況下，您可以使用應用程式的 `bootstrap/app.php` 檔案中的 `priority` 方法來指定中介層的優先順序：

    ->withMiddleware(function (Middleware $middleware) {
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

<a name="middleware-parameters"></a>
## 中介層參數

中介層也能接收額外參數。例如，若你的應用程式需要在執行特定動作前，驗證已驗證使用者是否具有某個「角色」，你可以建立一個 `EnsureUserHasRole` 中介層，該中介層會接收一個角色名稱作為額外引數。

額外的中介層參數將在 `$next` 引數之後傳遞給中介層：

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

中介層參數可在定義路由時指定，透過使用 `:` 分隔中介層名稱和參數：

    use App\Http\Middleware\EnsureUserHasRole;

    Route::put('/post/{id}', function (string $id) {
        // ...
    })->middleware(EnsureUserHasRole::class.':editor');

多個參數可使用逗號分隔：

    Route::put('/post/{id}', function (string $id) {
        // ...
    })->middleware(EnsureUserHasRole::class.':editor,publisher');


<a name="terminable-middleware"></a>
## 可終止的中介層

有時候，中介層可能需要在 HTTP 回應已發送給瀏覽器之後執行某些工作。若你在中介層上定義了 `terminate` 方法，且你的網頁伺服器正在使用 FastCGI，則 `terminate` 方法將在回應發送給瀏覽器之後自動呼叫：

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

`terminate` 方法應同時接收請求和回應。一旦你定義了一個可終止的中介層，就應該將其新增到應用程式 `bootstrap/app.php` 檔案中的路由或全域中介層清單。

當呼叫中介層的 `terminate` 方法時，Laravel 會從 [服務容器](/docs/{{version}}/container) 中解析一個全新的中介層實例。若你希望在 `handle` 和 `terminate` 方法被呼叫時使用相同的中介層實例，請使用容器的 `singleton` 方法在容器中註冊該中介層。通常這應該在 `AppServiceProvider` 的 `register` 方法中完成：

    use App\Http\Middleware\TerminatingMiddleware;

    /**
     * Register any application services.
     */
    public function register(): void
    {
        $this->app->singleton(TerminatingMiddleware::class);
    }