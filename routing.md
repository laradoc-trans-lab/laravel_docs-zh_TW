# 路由 (Routing)

- [基本路由](#basic-routing)
    - [預設路由檔案](#the-default-route-files)
    - [重新導向路由](#redirect-routes)
    - [視圖路由](#view-routes)
    - [列出您的路由](#listing-your-routes)
    - [路由客製化](#routing-customization)
- [路由參數](#route-parameters)
    - [必要參數](#required-parameters)
    - [選用參數](#parameters-optional-parameters)
    - [正規表達式限制](#parameters-regular-expression-constraints)
- [具名路由](#named-routes)
- [路由群組](#route-groups)
    - [Middleware](#route-group-middleware)
    - [控制器](#route-group-controllers)
    - [子網域路由](#route-group-subdomain-routing)
    - [路由前綴](#route-group-prefixes)
    - [路由名稱前綴](#route-group-name-prefixes)
- [路由模型綁定](#route-model-binding)
    - [隱式綁定](#implicit-binding)
    - [隱式 Enum 綁定](#implicit-enum-binding)
    - [顯式綁定](#explicit-binding)
- [備援路由](#fallback-routes)
- [速率限制](#rate-limiting)
    - [定義速率限制器](#defining-rate-limiters)
    - [將速率限制器附加到路由](#attaching-rate-limiters-to-routes)
- [表單方法偽造](#form-method-spoofing)
- [存取目前路由](#accessing-the-current-route)
- [跨來源資源共用 (CORS)](#cors)
- [路由快取](#route-caching)

<a name="basic-routing"></a>
## 基本路由

最基本的 Laravel 路由接受一個 URI 和一個閉包 (closure)，提供了一種非常簡單且富有表達力的方式來定義路由和行為，而無需複雜的路由設定檔：

```php
use Illuminate\Support\Facades\Route;

Route::get('/greeting', function () {
    return 'Hello World';
});
```

<a name="the-default-route-files"></a>
### 預設路由檔案

所有 Laravel 路由都定義在您的路由檔案中，這些檔案位於 `routes` 目錄。這些檔案會透過應用程式的 `bootstrap/app.php` 檔案中指定的設定，由 Laravel 自動載入。`routes/web.php` 檔案定義了用於您的網頁介面的路由。這些路由會被指派 `web` [Middleware 群組](/docs/{{version}}/middleware#laravels-default-middleware-groups)，該群組提供了諸如 Session 狀態和 CSRF 保護等功能。

對於大多數應用程式，您將從在 `routes/web.php` 檔案中定義路由開始。在 `routes/web.php` 中定義的路由可以透過在瀏覽器中輸入定義路由的 URL 來存取。例如，您可以透過在瀏覽器中導航到 `http://example.com/user` 來存取以下路由：

```php
use App\Http\Controllers\UserController;

Route::get('/user', [UserController::class, 'index']);
```

<a name="api-routes"></a>
#### API 路由

如果您的應用程式也將提供無狀態的 API，您可以使用 `install:api` Artisan 命令來啟用 API 路由：

```shell
php artisan install:api
```

`install:api` 命令會安裝 [Laravel Sanctum](/docs/{{version}}/sanctum)，它提供了一個強大而簡單的 API token 驗證守衛，可用於驗證第三方 API 消費者、SPA 或行動應用程式。此外，`install:api` 命令會建立 `routes/api.php` 檔案：

```php
Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```

`routes/api.php` 中的路由是無狀態的，並被指派給 `api` [Middleware 群組](/docs/{{version}}/middleware#laravels-default-middleware-groups)。此外，`/api` URI 前綴會自動應用於這些路由，因此您無需手動將其應用於檔案中的每個路由。您可以透過修改應用程式的 `bootstrap/app.php` 檔案來更改前綴：

```php
->withRouting(
    api: __DIR__.'/../routes/api.php',
    apiPrefix: 'api/admin',
    // ...
)
```

<a name="available-router-methods"></a>
#### 可用的路由方法

路由允許您註冊回應任何 HTTP 動詞的路由：

```php
Route::get($uri, $callback);
Route::post($uri, $callback);
Route::put($uri, $callback);
Route::patch($uri, $callback);
Route::delete($uri, $callback);
Route::options($uri, $callback);
```

有時您可能需要註冊一個回應多個 HTTP 動詞的路由。您可以使用 `match` 方法來實現。或者，您甚至可以使用 `any` 方法註冊一個回應所有 HTTP 動詞的路由：

```php
Route::match(['get', 'post'], '/', function () {
    // ...
});

Route::any('/', function () {
    // ...
});
```

> [!NOTE]
> 當定義多個共用相同 URI 的路由時，使用 `get`、`post`、`put`、`patch`、`delete` 和 `options` 方法的路由應在使用 `any`、`match` 和 `redirect` 方法的路由之前定義。這確保傳入的請求與正確的路由匹配。

<a name="dependency-injection"></a>
#### 依賴注入 (Dependency Injection)

您可以在路由的閉包簽名中型別提示路由所需的任何依賴。宣告的依賴將由 Laravel [Service Container](/docs/{{version}}/container) 自動解析並注入到閉包中。例如，您可以型別提示 `Illuminate\Http\Request` 類別，以將目前的 HTTP 請求自動注入到您的路由閉包中：

```php
use Illuminate\Http\Request;

Route::get('/users', function (Request $request) {
    // ...
});
```

<a name="csrf-protection"></a>
#### CSRF 保護

請記住，任何指向 `web` 路由檔案中定義的 `POST`、`PUT`、`PATCH` 或 `DELETE` 路由的 HTML 表單都應包含一個 CSRF token 欄位。否則，請求將被拒絕。您可以在 [CSRF 說明文件](/docs/{{version}}/csrf) 中閱讀更多關於 CSRF 保護的資訊：

```blade
<form method="POST" action="/profile">
    <!-- GEMINI_TRANSLATION_SUCCESS -->
# CSRF 保護

- [簡介](#csrf-introduction)
- [防止 CSRF 請求](#preventing-csrf-requests)
    - [排除特定 URI](#csrf-excluding-uris)
- [X-CSRF-Token](#csrf-x-csrf-token)
- [X-XSRF-Token](#csrf-x-xsrf-token)

<a name="csrf-introduction"></a>
## 簡介

跨網站請求偽造 (Cross-site request forgeries) 是一種惡意攻擊，攻擊者會代表已通過身份驗證的使用者執行未經授權的指令。幸運的是，Laravel 讓保護您的應用程式免受 [跨網站請求偽造](https://en.wikipedia.org/wiki/Cross-site_request_forgery) (CSRF) 攻擊變得輕而易舉。

<a name="csrf-explanation"></a>
#### 漏洞說明

如果您不熟悉跨網站請求偽造，讓我們討論一個此漏洞如何被利用的範例。想像您的應用程式有一個 `/user/email` 路由，它接受 `POST` 請求來更改已通過身份驗證的使用者電子郵件地址。此路由很可能預期 `email` 輸入欄位包含使用者想要開始使用的電子郵件地址。

如果沒有 CSRF 保護，惡意網站可以建立一個指向您應用程式 `/user/email` 路由的 HTML 表單，並提交惡意使用者自己的電子郵件地址：

```blade
<form action="https://your-application.com/user/email" method="POST">
    <input type="email" value="malicious-email@example.com">
</form>

<script>
    document.forms[0].submit();
</script>
```

如果惡意網站在頁面載入時自動提交表單，惡意使用者只需誘騙您應用程式中毫無戒心的使用者造訪他們的網站，他們的電子郵件地址就會在您的應用程式中被更改。

為了防止此漏洞，我們需要檢查每個傳入的 `POST`、`PUT`、`PATCH` 或 `DELETE` 請求，以尋找惡意應用程式無法存取的秘密 Session 值。

<a name="preventing-csrf-requests"></a>
## 防止 CSRF 請求

Laravel 會為應用程式管理的每個活躍 [使用者 Session](/docs/{{version}}/session) 自動產生一個 CSRF「token」。此 token 用於驗證已通過身份驗證的使用者確實是向應用程式發出請求的人。由於此 token 儲存在使用者的 Session 中，並且每次 Session 重新產生時都會更改，因此惡意應用程式無法存取它。

目前 Session 的 CSRF token 可以透過請求的 Session 或 `csrf_token` 輔助函式來存取：

```php
use Illuminate\Http\Request;

Route::get('/token', function (Request $request) {
    $token = $request->session()->token();

    $token = csrf_token();

    // ...
});
```

每當您在應用程式中定義「POST」、「PUT」、「PATCH」或「DELETE」HTML 表單時，您都應該在表單中包含一個隱藏的 CSRF `_token` 欄位，以便 CSRF 保護 Middleware 可以驗證請求。為方便起見，您可以使用 ` @csrf` Blade 指令來產生隱藏的 token 輸入欄位：

```blade
<form method="POST" action="/profile">
    @csrf

    <!-- Equivalent to... -->
    <input type="hidden" name="_token" value="{{ csrf_token() }}" />
</form>
```

預設包含在 `web` Middleware 群組中的 `Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` [Middleware](/docs/{{version}}/middleware) 會自動驗證請求輸入中的 token 是否與 Session 中儲存的 token 相符。當這兩個 token 相符時，我們就知道是已通過身份驗證的使用者發起了請求。

<a name="csrf-tokens-and-spas"></a>
### CSRF Token 與 SPA

如果您正在建構一個使用 Laravel 作為 API 後端的 SPA，您應該查閱 [Laravel Sanctum 說明文件](/docs/{{version}}/sanctum)，以獲取有關使用您的 API 進行身份驗證和防範 CSRF 漏洞的資訊。

<a name="csrf-excluding-uris"></a>
### 從 CSRF 保護中排除 URI

有時您可能希望從 CSRF 保護中排除一組 URI。例如，如果您正在使用 [Stripe](https://stripe.com) 處理付款並利用他們的 webhook 系統，您將需要從 CSRF 保護中排除您的 Stripe webhook 處理程式路由，因為 Stripe 不會知道要向您的路由發送什麼 CSRF token。

通常，您應該將這些類型的路由放置在 Laravel 應用於 `routes/web.php` 檔案中所有路由的 `web` Middleware 群組之外。但是，您也可以透過在應用程式的 `bootstrap/app.php` 檔案中向 `validateCsrfTokens` 方法提供其 URI 來排除特定路由：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->validateCsrfTokens(except: [
        'stripe/*',
        'http://example.com/foo/bar',
        'http://example.com/foo/*',
    ]);
})
```

> [!NOTE]
> 為方便起見，當 [執行測試](/docs/{{version}}/testing) 時，CSRF Middleware 會自動對所有路由停用。

<a name="csrf-x-csrf-token"></a>
## X-CSRF-TOKEN

除了檢查作為 POST 參數的 CSRF token 之外，預設包含在 `web` Middleware 群組中的 `Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` Middleware 也會檢查 `X-CSRF-TOKEN` 請求標頭。例如，您可以將 token 儲存在 HTML `meta` 標籤中：

```blade
<meta name="csrf-token" content="{{ csrf_token() }}">
```

然後，您可以指示像 jQuery 這樣的函式庫自動將 token 添加到所有請求標頭中。這為您使用傳統 JavaScript 技術的 AJAX 應用程式提供了簡單、方便的 CSRF 保護：

```js
$.ajaxSetup({
    headers: {
        'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
    }
});
```

<a name="csrf-x-xsrf-token"></a>
## X-XSRF-TOKEN

Laravel 會將目前的 CSRF token 儲存在一個加密的 `XSRF-TOKEN` Cookie 中，該 Cookie 會隨框架產生的每個回應一起包含。您可以使用 Cookie 值來設定 `X-XSRF-TOKEN` 請求標頭。

此 Cookie 主要作為開發人員的便利性而發送，因為某些 JavaScript 框架和函式庫，例如 Angular 和 Axios，會自動將其值放置在同源請求的 `X-XSRF-TOKEN` 標頭中。

> [!NOTE]
> 預設情況下，`resources/js/bootstrap.js` 檔案包含 Axios HTTP 函式庫，它會自動為您發送 `X-XSRF-TOKEN` 標頭。
    ...
</form>
```

<a name="redirect-routes"></a>
### 重新導向路由

如果您正在定義一個重新導向到另一個 URI 的路由，您可以使用 `Route::redirect` 方法。此方法提供了一個方便的捷徑，讓您無需為執行簡單的重新導向而定義完整的路由或控制器：

```php
Route::redirect('/here', '/there');
```

預設情況下，`Route::redirect` 會回傳 `302` 狀態碼。您可以使用選用的第三個參數來自訂狀態碼：

```php
Route::redirect('/here', '/there', 301);
```

或者，您可以使用 `Route::permanentRedirect` 方法來回傳 `301` 狀態碼：

```php
Route::permanentRedirect('/here', '/there');
```

> [!WARNING]
> 在重新導向路由中使用路由參數時，以下參數由 Laravel 保留，不能使用：`destination` 和 `status`。

<a name="view-routes"></a>
### 視圖路由

如果您的路由只需要回傳一個 [視圖](/docs/{{version}}/views)，您可以使用 `Route::view` 方法。與 `redirect` 方法一樣，此方法提供了一個簡單的捷徑，讓您無需定義完整的路由或控制器。`view` 方法接受一個 URI 作為其第一個參數，一個視圖名稱作為其第二個參數。此外，您可以提供一個資料陣列作為選用的第三個參數傳遞給視圖：

```php
Route::view('/welcome', 'welcome');

Route::view('/welcome', 'welcome', ['name' => 'Taylor']);
```

> [!WARNING]
> 在視圖路由中使用路由參數時，以下參數由 Laravel 保留，不能使用：`view`、`data`、`status` 和 `headers`。

<a name="listing-your-routes"></a>
### 列出您的路由

`route:list` Artisan 命令可以輕鬆提供應用程式定義的所有路由的概覽：

```shell
php artisan route:list
```

預設情況下，指派給每個路由的路由 Middleware 不會顯示在 `route:list` 輸出中；但是，您可以透過向命令添加 `-v` 選項來指示 Laravel 顯示路由 Middleware 和 Middleware 群組名稱：

```shell
php artisan route:list -v

# 展開 Middleware 群組...
php artisan route:list -vv
```

您還可以指示 Laravel 只顯示以給定 URI 開頭的路由：

```shell
php artisan route:list --path=api
```

此外，您可以在執行 `route:list` 命令時提供 `--except-vendor` 選項，指示 Laravel 隱藏由第三方套件定義的任何路由：

```shell
php artisan route:list --except-vendor
```

同樣地，您也可以在執行 `route:list` 命令時提供 `--only-vendor` 選項，指示 Laravel 只顯示由第三方套件定義的路由：

```shell
php artisan route:list --only-vendor
```

<a name="routing-customization"></a>
### 路由客製化

預設情況下，應用程式的路由由 `bootstrap/app.php` 檔案設定和載入：

```php
<?php

use Illuminate\Foundation\Application;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )->create();
```

然而，有時您可能希望定義一個全新的檔案來包含應用程式路由的子集。為此，您可以向 `withRouting` 方法提供一個 `then` 閉包。在此閉包中，您可以註冊應用程式所需的任何額外路由：

```php
use Illuminate\Support\Facades\Route;

->withRouting(
    web: __DIR__.'/../routes/web.php',
    commands: __DIR__.'/../routes/console.php',
    health: '/up',
    then: function () {
        Route::middleware('api')
            ->prefix('webhooks')
            ->name('webhooks.')
            ->group(base_path('routes/webhooks.php'));
    },
)
```

或者，您甚至可以透過向 `withRouting` 方法提供一個 `using` 閉包來完全控制路由註冊。當傳遞此參數時，框架將不會註冊任何 HTTP 路由，您將負責手動註冊所有路由：

```php
use Illuminate\Support\Facades\Route;

->withRouting(
    commands: __DIR__.'/../routes/console.php',
    using: function () {
        Route::middleware('api')
            ->prefix('api')
            ->group(base_path('routes/api.php'));

        Route::middleware('web')
            ->group(base_path('routes/web.php'));
    },
)
```

<a name="route-parameters"></a>
## 路由參數

<a name="required-parameters"></a>
### 必要參數

有時您需要在路由中擷取 URI 的片段。例如，您可能需要從 URL 中擷取使用者的 ID。您可以透過定義路由參數來實現：

```php
Route::get('/user/{id}', function (string $id) {
    return 'User '.$id;
});
```

您可以根據路由的需要定義任意數量的路由參數：

```php
Route::get('/posts/{post}/comments/{comment}', function (string $postId, string $commentId) {
    // ...
});
```

路由參數始終包含在 `{}` 大括號中，並且應由字母字元組成。底線 (`_`) 在路由參數名稱中也是可接受的。路由參數根據其順序注入到路由閉包/控制器中——路由閉包/控制器引數的名稱並不重要。

<a name="parameters-and-dependency-injection"></a>
#### 參數與依賴注入

如果您的路由具有您希望 Laravel Service Container 自動注入到路由閉包中的依賴，您應該在依賴之後列出您的路由參數：

```php
use Illuminate\Http\Request;

Route::get('/user/{id}', function (Request $request, string $id) {
    return 'User '.$id;
});
```

<a name="parameters-optional-parameters"></a>
### 選用參數

有時您可能需要指定一個在 URI 中可能不總是存在的路由參數。您可以透過在參數名稱後放置一個 `?` 標記來實現。請務必為路由的對應變數提供一個預設值：

```php
Route::get('/user/{name?}', function (?string $name = null) {
    return $name;
});

Route::get('/user/{name?}', function (?string $name = 'John') {
    return $name;
});
```

<a name="parameters-regular-expression-constraints"></a>
### 正規表達式限制

您可以使用路由實例上的 `where` 方法來限制路由參數的格式。`where` 方法接受參數的名稱和定義參數應如何限制的正規表達式：

```php
Route::get('/user/{name}', function (string $name) {
    // ...
})->where('name', '[A-Za-z]+');

Route::get('/user/{id}', function (string $id) {
    // ...
})->where('id', '[0-9]+');

Route::get('/user/{id}/{name}', function (string $id, string $name) {
    // ...
})->where(['id' => '[0-9]+', 'name' => '[a-z]+']);
```

為方便起見，一些常用的正規表達式模式具有輔助方法，可讓您快速為路由添加模式限制：

```php
Route::get('/user/{id}/{name}', function (string $id, string $name) {
    // ...
})->whereNumber('id')->whereAlpha('name');

Route::get('/user/{name}', function (string $name) {
    // ...
})->whereAlphaNumeric('name');

Route::get('/user/{id}', function (string $id) {
    // ...
})->whereUuid('id');

Route::get('/user/{id}', function (string $id) {
    // ...
})->whereUlid('id');

Route::get('/category/{category}', function (string $category) {
    // ...
})->whereIn('category', ['movie', 'song', 'painting']);

Route::get('/category/{category}', function (string $category) {
    // ...
})->whereIn('category', CategoryEnum::cases());
```

如果傳入的請求不符合路由模式限制，將回傳 404 HTTP 回應。

<a name="parameters-global-constraints"></a>
#### 全域限制

如果您希望路由參數始終受給定正規表達式限制，您可以使用 `pattern` 方法。您應該在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中定義這些模式：

```php
use Illuminate\Support\Facades\Route;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Route::pattern('id', '[0-9]+');
}
```

一旦定義了模式，它就會自動應用於所有使用該參數名稱的路由：

```php
Route::get('/user/{id}', function (string $id) {
    // 僅在 {id} 為數字時執行...
});
```

<a name="parameters-encoded-forward-slashes"></a>
#### 編碼的正斜線

Laravel 路由元件允許所有字元（除了 `/`）存在於路由參數值中。您必須使用 `where` 條件正規表達式明確允許 `/` 成為佔位符的一部分：

```php
Route::get('/search/{search}', function (string $search) {
    return $search;
})->where('search', '.*');
```

> [!WARNING]
> 編碼的正斜線僅在最後一個路由片段中受支援。

<a name="named-routes"></a>
## 具名路由

具名路由允許方便地為特定路由產生 URL 或重新導向。您可以透過將 `name` 方法鏈接到路由定義來為路由指定名稱：

```php
Route::get('/user/profile', function () {
    // ...
})->name('profile');
```

您也可以為控制器動作指定路由名稱：

```php
Route::get(
    '/user/profile',
    [UserProfileController::class, 'show']
)->name('profile');
```

> [!WARNING]
> 路由名稱應始終是唯一的。

<a name="generating-urls-to-named-routes"></a>
#### 產生具名路由的 URL

一旦您為給定路由分配了名稱，您就可以在透過 Laravel 的 `route` 和 `redirect` 輔助函式產生 URL 或重新導向時使用路由的名稱：

```php
// 產生 URL...
$url = route('profile');

// 產生重新導向...
return redirect()->route('profile');

return to_route('profile');
```

如果具名路由定義了參數，您可以將參數作為第二個引數傳遞給 `route` 函式。給定的參數將自動以正確的位置插入到產生的 URL 中：

```php
Route::get('/user/{id}/profile', function (string $id) {
    // ...
})->name('profile');

$url = route('profile', ['id' => 1]);
```

如果您在陣列中傳遞額外的參數，這些鍵/值對將自動添加到產生的 URL 的查詢字串中：

```php
Route::get('/user/{id}/profile', function (string $id) {
    // ...
})->name('profile');

$url = route('profile', ['id' => 1, 'photos' => 'yes']);

// /user/1/profile?photos=yes
```

> [!NOTE]
> 有時，您可能希望為 URL 參數指定請求範圍的預設值，例如目前的語系。為此，您可以使用 [URL::defaults 方法](/docs/{{version}}/urls#default-values)。

<a name="inspecting-the-current-route"></a>
#### 檢查目前路由

如果您想確定目前的請求是否路由到給定的具名路由，您可以使用 Route 實例上的 `named` 方法。例如，您可以從路由 Middleware 中檢查目前的路由名稱：

```php
use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

/**
 * Handle an incoming request.
 *
 * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
 */
public function handle(Request $request, Closure $next): Response
{
    if ($request->route()->named('profile')) {
        // ...
    }

    return $next($request);
}
```

<a name="route-groups"></a>
## 路由群組

路由群組允許您在大量路由之間共用路由屬性，例如 Middleware，而無需在每個單獨的路由上定義這些屬性。

巢狀群組會嘗試智慧地將屬性與其父群組「合併」。Middleware 和 `where` 條件會合併，而名稱和前綴會附加。命名空間分隔符和 URI 前綴中的斜線會自動在適當的位置添加。

<a name="route-group-middleware"></a>
### Middleware

要將 [Middleware](/docs/{{version}}/middleware) 指派給群組中的所有路由，您可以在定義群組之前使用 `middleware` 方法。Middleware 會按照它們在陣列中列出的順序執行：

```php
Route::middleware(['first', 'second'])->group(function () {
    Route::get('/', function () {
        // 使用 first & second Middleware...
    });

    Route::get('/user/profile', function () {
        // 使用 first & second Middleware...
    });
});
```

<a name="route-group-controllers"></a>
### 控制器

如果一組路由都使用相同的 [控制器](/docs/{{version}}/controllers)，您可以使用 `controller` 方法為群組中的所有路由定義共同的控制器。然後，在定義路由時，您只需要提供它們呼叫的控制器方法：

```php
use App\Http\Controllers\OrderController;

Route::controller(OrderController::class)->group(function () {
    Route::get('/orders/{id}', 'show');
    Route::post('/orders', 'store');
});
```

<a name="route-group-subdomain-routing"></a>
### 子網域路由

路由群組也可以用於處理子網域路由。子網域可以像路由 URI 一樣指派路由參數，允許您擷取子網域的一部分以用於您的路由或控制器。子網域可以透過在定義群組之前呼叫 `domain` 方法來指定：

```php
Route::domain('{account}.example.com')->group(function () {
    Route::get('/user/{id}', function (string $account, string $id) {
        // ...
    });
});
```

> [!WARNING]
> 為了確保您的子網域路由可達，您應該在註冊根網域路由之前註冊子網域路由。這將防止根網域路由覆寫具有相同 URI 路徑的子網域路由。

<a name="route-group-prefixes"></a>
### 路由前綴

`prefix` 方法可用於為群組中的每個路由加上給定的 URI 前綴。例如，您可能希望為群組中的所有路由 URI 加上 `admin` 前綴：

```php
Route::prefix('admin')->group(function () {
    Route::get('/users', function () {
        // 匹配 "/admin/users" URL
    });
});
```

<a name="route-group-name-prefixes"></a>
### 路由名稱前綴

`name` 方法可用於為群組中的每個路由名稱加上給定的字串前綴。例如，您可能希望為群組中所有路由的名稱加上 `admin` 前綴。給定的字串會完全按照指定的方式作為路由名稱的前綴，因此我們將確保在前綴中提供尾隨的 `.` 字元：

```php
Route::name('admin.')->group(function () {
    Route::get('/users', function () {
        // 路由被指派名稱 "admin.users"...
    })->name('users');
});
```

<a name="route-model-binding"></a>
## 路由模型綁定

當將模型 ID 注入到路由或控制器動作時，您通常會查詢資料庫以檢索與該 ID 對應的模型。Laravel 路由模型綁定提供了一種方便的方式，可以將模型實例直接自動注入到您的路由中。例如，您可以注入整個與給定 ID 匹配的 `User` 模型實例，而不是注入使用者的 ID。

<a name="implicit-binding"></a>
### 隱式綁定

Laravel 會自動解析在路由或控制器動作中定義的 Eloquent 模型，這些模型的型別提示變數名稱與路由片段名稱匹配。例如：

```php
use App\Models\User;

Route::get('/users/{user}', function (User $user) {
    return $user->email;
});
```

由於 `$user` 變數被型別提示為 `App\Models\User` Eloquent 模型，並且變數名稱與 `{user}` URI 片段匹配，Laravel 將自動注入具有與請求 URI 中對應值匹配的 ID 的模型實例。如果在資料庫中找不到匹配的模型實例，將自動產生 404 HTTP 回應。

當然，在使用控制器方法時也可以進行隱式綁定。再次注意 `{user}` URI 片段與控制器中包含 `App\Models\User` 型別提示的 `$user` 變數匹配：

```php
use App\Http\Controllers\UserController;
use App\Models\User;

// 路由定義...
Route::get('/users/{user}', [UserController::class, 'show']);

// 控制器方法定義...
public function show(User $user)
{
    return view('user.profile', ['user' => $user]);
}
```

<a name="implicit-soft-deleted-models"></a>
#### 軟刪除模型

通常，隱式模型綁定不會檢索已 [軟刪除](/docs/{{version}}/eloquent#soft-deleting) 的模型。但是，您可以透過將 `withTrashed` 方法鏈接到路由的定義來指示隱式綁定檢索這些模型：

```php
use App\Models\User;

Route::get('/users/{user}', function (User $user) {
    return $user->email;
})->withTrashed();
```

<a name="customizing-the-default-key-name"></a>
#### 自訂預設鍵名

有時您可能希望使用 `id` 以外的欄位來解析 Eloquent 模型。為此，您可以在路由參數定義中指定欄位：

```php
use App\Models\Post;

Route::get('/posts/{post:slug}', function (Post $post) {
    return $post;
});
```

如果您希望模型綁定在檢索給定模型類別時始終使用 `id` 以外的資料庫欄位，您可以覆寫 Eloquent 模型上的 `getRouteKeyName` 方法：

```php
/**
 * Get the route key for the model.
 */
public function getRouteKeyName(): string
{
    return 'slug';
}
```

<a name="implicit-model-binding-scoping"></a>
#### 自訂鍵與範圍設定

當在單一路由定義中隱式綁定多個 Eloquent 模型時，您可能希望對第二個 Eloquent 模型進行範圍設定，使其必須是前一個 Eloquent 模型的子級。例如，考慮這個路由定義，它為特定使用者透過 slug 檢索部落格文章：

```php
use App\Models\Post;
use App\Models\User;

Route::get('/users/{user}/posts/{post:slug}', function (User $user, Post $post) {
    return $post;
});
```

當使用自訂鍵的隱式綁定作為巢狀路由參數時，Laravel 將自動透過其父級對查詢進行範圍設定，以檢索巢狀模型，並使用慣例來猜測父級上的關聯名稱。在這種情況下，將假定 `User` 模型具有名為 `posts` (路由參數名稱的複數形式) 的關聯，可用於檢索 `Post` 模型。

如果您願意，您可以指示 Laravel 即使未提供自訂鍵也對「子」綁定進行範圍設定。為此，您可以在定義路由時呼叫 `scopeBindings` 方法：

```php
use App\Models\Post;
use App\Models\User;

Route::get('/users/{user}/posts/{post}', function (User $user, Post $post) {
    return $post;
})->scopeBindings();
```

或者，您可以指示整個路由定義群組使用範圍綁定：

```php
Route::scopeBindings()->group(function () {
    Route::get('/users/{user}/posts/{post}', function (User $user, Post $post) {
        return $post;
    });
});
```

同樣地，您可以透過呼叫 `withoutScopedBindings` 方法明確指示 Laravel 不對綁定進行範圍設定：

```php
Route::get('/users/{user}/posts/{post:slug}', function (User $user, Post $post) {
    return $post;
})->withoutScopedBindings();
```

<a name="customizing-missing-model-behavior"></a>
#### 自訂模型遺失行為

通常，如果找不到隱式綁定的模型，將會產生 404 HTTP 回應。但是，您可以在定義路由時呼叫 `missing` 方法來自訂此行為。`missing` 方法接受一個閉包，如果找不到隱式綁定的模型，該閉包將被呼叫：

```php
use App\Http\Controllers\LocationsController;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Redirect;

Route::get('/locations/{location:slug}', [LocationsController::class, 'show'])
    ->name('locations.view')
    ->missing(function (Request $request) {
        return Redirect::route('locations.index');
    });
```

<a name="implicit-enum-binding"></a>
### 隱式 Enum 綁定

PHP 8.1 引入了對 [Enums](https://www.php.net/manual/en/language.enumerations.backed.php) 的支援。為了補充此功能，Laravel 允許您在路由定義上型別提示 [string-backed Enum](https://www.php.net/manual/en/language.enumerations.backed.php)，並且 Laravel 只會在該路由片段對應於有效的 Enum 值時呼叫路由。否則，將自動回傳 404 HTTP 回應。例如，給定以下 Enum：

```php
<?php

namespace App\Enums;

enum Category: string
{
    case Fruits = 'fruits';
    case People = 'people';
}
```

您可以定義一個路由，該路由只會在 `{category}` 路由片段為 `fruits` 或 `people` 時被呼叫。否則，Laravel 將回傳 404 HTTP 回應：

```php
use App\Enums\Category;
use Illuminate\Support\Facades\Route;

Route::get('/categories/{category}', function (Category $category) {
    return $category->value;
});
```

<a name="explicit-binding"></a>
### 顯式綁定

您不需要使用 Laravel 的隱式、基於慣例的模型解析來使用模型綁定。您也可以明確定義路由參數如何對應到模型。要註冊顯式綁定，請使用路由器的 `model` 方法為給定參數指定類別。您應該在 `AppServiceProvider` 類別的 `boot` 方法開頭定義您的顯式模型綁定：

```php
use App\Models\User;
use Illuminate\Support\Facades\Route;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Route::model('user', User::class);
}
```

接下來，定義一個包含 `{user}` 參數的路由：

```php
use App\Models\User;

Route::get('/users/{user}', function (User $user) {
    // ...
});
```

由於我們已將所有 `{user}` 參數綁定到 `App\Models\User` 模型，因此該類別的實例將被注入到路由中。因此，例如，對 `users/1` 的請求將注入資料庫中 ID 為 `1` 的 `User` 實例。

如果在資料庫中找不到匹配的模型實例，將自動產生 404 HTTP 回應。

<a name="customizing-the-resolution-logic"></a>
#### 自訂解析邏輯

如果您希望定義自己的模型綁定解析邏輯，您可以使用 `Route::bind` 方法。您傳遞給 `bind` 方法的閉包將接收 URI 片段的值，並應回傳應注入到路由中的類別實例。同樣，此客製化應在應用程式的 `AppServiceProvider` 的 `boot` 方法中進行：

```php
use App\Models\User;
use Illuminate\Support\Facades\Route;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Route::bind('user', function (string $value) {
        return User::where('name', $value)->firstOrFail();
    });
}
```

或者，您可以覆寫 Eloquent 模型上的 `resolveRouteBinding` 方法。此方法將接收 URI 片段的值，並應回傳應注入到路由中的類別實例：

```php
/**
 * Retrieve the model for a bound value.
 *
 * @param  mixed  $value
 * @param  string|null  $field
 * @return \Illuminate\Database\Eloquent\Model|null
 */
public function resolveRouteBinding($value, $field = null)
{
    return $this->where('name', $value)->firstOrFail();
}
```

如果路由正在使用 [隱式綁定範圍設定](#implicit-model-binding-scoping)，則 `resolveChildRouteBinding` 方法將用於解析父模型的子綁定：

```php
/**
 * Retrieve the child model for a bound value.
 *
 * @param  string  $childType
 * @param  mixed  $value
 * @param  string|null  $field
 * @return \Illuminate\Database\Eloquent\Model|null
 */
public function resolveChildRouteBinding($childType, $value, $field)
{
    return parent::resolveChildRouteBinding($childType, $value, $field);
}
```

<a name="fallback-routes"></a>
## 備援路由

使用 `Route::fallback` 方法，您可以定義一個路由，當沒有其他路由匹配傳入請求時，該路由將被執行。通常，未處理的請求將透過應用程式的例外處理器自動呈現「404」頁面。然而，由於您通常會在 `routes/web.php` 檔案中定義 `fallback` 路由，因此 `web` Middleware 群組中的所有 Middleware 都將應用於該路由。您可以根據需要為此路由添加額外的 Middleware：

```php
Route::fallback(function () {
    // ...
});
```

<a name="rate-limiting"></a>
## 速率限制

<a name="defining-rate-limiters"></a>
### 定義速率限制器

Laravel 包含強大且可自訂的速率限制服務，您可以使用這些服務來限制給定路由或路由群組的流量。首先，您應該定義符合應用程式需求的速率限制器設定。

速率限制器可以在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中定義：

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

/**
 * Bootstrap any application services.
 */
protected function boot(): void
{
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
    });
}
```

速率限制器是使用 `RateLimiter` Facade 的 `for` 方法定義的。`for` 方法接受一個速率限制器名稱和一個閉包，該閉包回傳應應用於指派給該速率限制器的路由的限制設定。限制設定是 `Illuminate\Cache\RateLimiting\Limit` 類別的實例。此類別包含有用的「建構器」方法，可讓您快速定義限制。速率限制器名稱可以是您想要的任何字串：

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

/**
 * Bootstrap any application services.
 */
protected function boot(): void
{
    RateLimiter::for('global', function (Request $request) {
        return Limit::perMinute(1000);
    });
}
```

如果傳入的請求超過指定的速率限制，Laravel 將自動回傳一個 HTTP 狀態碼為 429 的回應。如果您想定義自己的速率限制器應回傳的回應，您可以使用 `response` 方法：

```php
RateLimiter::for('global', function (Request $request) {
    return Limit::perMinute(1000)->response(function (Request $request, array $headers) {
        return response('Custom response...', 429, $headers);
    });
});
```

由於速率限制器閉包會接收傳入的 HTTP 請求實例，您可以根據傳入請求或已驗證的使用者動態建構適當的速率限制：

```php
RateLimiter::for('uploads', function (Request $request) {
    return $request->user()->vipCustomer()
        ? Limit::none()
        : Limit::perHour(10);
});
```

<a name="segmenting-rate-limits"></a>
#### 分段速率限制

有時您可能希望按某些任意值對速率限制進行分段。例如，您可能希望允許使用者每分鐘每個 IP 位址存取給定路由 100 次。為此，您可以在建構速率限制時使用 `by` 方法：

```php
RateLimiter::for('uploads', function (Request $request) {
    return $request->user()->vipCustomer()
        ? Limit::none()
        : Limit::perMinute(100)->by($request->ip());
});
```

為了使用另一個範例來說明此功能，我們可以將路由的存取限制為每分鐘每個已驗證使用者 ID 100 次，或每分鐘每個 IP 位址 10 次（針對訪客）：

```php
RateLimiter::for('uploads', function (Request $request) {
    return $request->user()
        ? Limit::perMinute(100)->by($request->user()->id)
        : Limit::perMinute(10)->by($request->ip());
});
```

<a name="multiple-rate-limits"></a>
#### 多重速率限制

如果需要，您可以為給定的速率限制器設定回傳一個速率限制陣列。每個速率限制將根據它們在陣列中的順序對路由進行評估：

```php
RateLimiter::for('login', function (Request $request) {
    return [
        Limit::perMinute(500),
        Limit::perMinute(3)->by($request->input('email')),
    ];
});
```

如果您要指派多個由相同 `by` 值分段的速率限制，您應該確保每個 `by` 值都是唯一的。實現此目的最簡單的方法是為傳遞給 `by` 方法的值加上前綴：

```php
RateLimiter::for('uploads', function (Request $request) {
    return [
        Limit::perMinute(10)->by('minute:'.$request->user()->id),
        Limit::perDay(1000)->by('day:'.$request->user()->id),
    ];
});
```

<a name="attaching-rate-limiters-to-routes"></a>
### 將速率限制器附加到路由

速率限制器可以使用 `throttle` [Middleware](/docs/{{version}}/middleware) 附加到路由或路由群組。`throttle` Middleware 接受您希望指派給路由的速率限制器名稱：

```php
Route::middleware(['throttle:uploads'])->group(function () {
    Route::post('/audio', function () {
        // ...
    });

    Route::post('/video', function () {
        // ...
    });
});
```

<a name="throttling-with-redis"></a>
#### 使用 Redis 進行節流

預設情況下，`throttle` Middleware 會映射到 `Illuminate\Routing\Middleware\ThrottleRequests` 類別。但是，如果您使用 Redis 作為應用程式的快取驅動程式，您可能希望指示 Laravel 使用 Redis 來管理速率限制。為此，您應該在應用程式的 `bootstrap/app.php` 檔案中使用 `throttleWithRedis` 方法。此方法將 `throttle` Middleware 映射到 `Illuminate\Routing\Middleware\ThrottleRequestsWithRedis` Middleware 類別：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->throttleWithRedis();
    // ...
})
```

<a name="form-method-spoofing"></a>
## 表單方法偽造

HTML 表單不支援 `PUT`、`PATCH` 或 `DELETE` 動作。因此，當定義從 HTML 表單呼叫的 `PUT`、`PATCH` 或 `DELETE` 路由時，您需要在表單中添加一個隱藏的 `_method` 欄位。隨 `_method` 欄位發送的值將用作 HTTP 請求方法：

```blade
<form action="/example" method="POST">
    <input type="hidden" name="_method" value="PUT">
    <input type="hidden" name="_token" value="{{ csrf_token() }}">
</form>
```

為方便起見，您可以使用 ` @method` [Blade 指令](/docs/{{version}}/blade) 來產生 `_method` 輸入欄位：

```blade
<form action="/example" method="POST">
    @method('PUT')
    <!-- GEMINI_TRANSLATION_SUCCESS -->
# CSRF 保護

- [簡介](#csrf-introduction)
- [防止 CSRF 請求](#preventing-csrf-requests)
    - [排除特定 URI](#csrf-excluding-uris)
- [X-CSRF-Token](#csrf-x-csrf-token)
- [X-XSRF-Token](#csrf-x-xsrf-token)

<a name="csrf-introduction"></a>
## 簡介

跨網站請求偽造 (Cross-site request forgeries) 是一種惡意攻擊，攻擊者會代表已通過身份驗證的使用者執行未經授權的指令。幸運的是，Laravel 讓保護您的應用程式免受 [跨網站請求偽造](https://en.wikipedia.org/wiki/Cross-site_request_forgery) (CSRF) 攻擊變得輕而易舉。

<a name="csrf-explanation"></a>
#### 漏洞說明

如果您不熟悉跨網站請求偽造，讓我們討論一個此漏洞如何被利用的範例。想像您的應用程式有一個 `/user/email` 路由，它接受 `POST` 請求來更改已通過身份驗證的使用者電子郵件地址。此路由很可能預期 `email` 輸入欄位包含使用者想要開始使用的電子郵件地址。

如果沒有 CSRF 保護，惡意網站可以建立一個指向您應用程式 `/user/email` 路由的 HTML 表單，並提交惡意使用者自己的電子郵件地址：

```blade
<form action="https://your-application.com/user/email" method="POST">
    <input type="email" value="malicious-email@example.com">
</form>

<script>
    document.forms[0].submit();
</script>
```

如果惡意網站在頁面載入時自動提交表單，惡意使用者只需誘騙您應用程式中毫無戒心的使用者造訪他們的網站，他們的電子郵件地址就會在您的應用程式中被更改。

為了防止此漏洞，我們需要檢查每個傳入的 `POST`、`PUT`、`PATCH` 或 `DELETE` 請求，以尋找惡意應用程式無法存取的秘密 Session 值。

<a name="preventing-csrf-requests"></a>
## 防止 CSRF 請求

Laravel 會為應用程式管理的每個活躍 [使用者 Session](/docs/{{version}}/session) 自動產生一個 CSRF「token」。此 token 用於驗證已通過身份驗證的使用者確實是向應用程式發出請求的人。由於此 token 儲存在使用者的 Session 中，並且每次 Session 重新產生時都會更改，因此惡意應用程式無法存取它。

目前 Session 的 CSRF token 可以透過請求的 Session 或 `csrf_token` 輔助函式來存取：

```php
use Illuminate\Http\Request;

Route::get('/token', function (Request $request) {
    $token = $request->session()->token();

    $token = csrf_token();

    // ...
});
```

每當您在應用程式中定義「POST」、「PUT」、「PATCH」或「DELETE」HTML 表單時，您都應該在表單中包含一個隱藏的 CSRF `_token` 欄位，以便 CSRF 保護 Middleware 可以驗證請求。為方便起見，您可以使用 ` @csrf` Blade 指令來產生隱藏的 token 輸入欄位：

```blade
<form method="POST" action="/profile">
    @csrf

    <!-- Equivalent to... -->
    <input type="hidden" name="_token" value="{{ csrf_token() }}" />
</form>
```

預設包含在 `web` Middleware 群組中的 `Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` [Middleware](/docs/{{version}}/middleware) 會自動驗證請求輸入中的 token 是否與 Session 中儲存的 token 相符。當這兩個 token 相符時，我們就知道是已通過身份驗證的使用者發起了請求。

<a name="csrf-tokens-and-spas"></a>
### CSRF Token 與 SPA

如果您正在建構一個使用 Laravel 作為 API 後端的 SPA，您應該查閱 [Laravel Sanctum 說明文件](/docs/{{version}}/sanctum)，以獲取有關使用您的 API 進行身份驗證和防範 CSRF 漏洞的資訊。

<a name="csrf-excluding-uris"></a>
### 從 CSRF 保護中排除 URI

有時您可能希望從 CSRF 保護中排除一組 URI。例如，如果您正在使用 [Stripe](https://stripe.com) 處理付款並利用他們的 webhook 系統，您將需要從 CSRF 保護中排除您的 Stripe webhook 處理程式路由，因為 Stripe 不會知道要向您的路由發送什麼 CSRF token。

通常，您應該將這些類型的路由放置在 Laravel 應用於 `routes/web.php` 檔案中所有路由的 `web` Middleware 群組之外。但是，您也可以透過在應用程式的 `bootstrap/app.php` 檔案中向 `validateCsrfTokens` 方法提供其 URI 來排除特定路由：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->validateCsrfTokens(except: [
        'stripe/*',
        'http://example.com/foo/bar',
        'http://example.com/foo/*',
    ]);
})
```

> [!NOTE]
> 為方便起見，當 [執行測試](/docs/{{version}}/testing) 時，CSRF Middleware 會自動對所有路由停用。

<a name="csrf-x-csrf-token"></a>
## X-CSRF-TOKEN

除了檢查作為 POST 參數的 CSRF token 之外，預設包含在 `web` Middleware 群組中的 `Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` Middleware 也會檢查 `X-CSRF-TOKEN` 請求標頭。例如，您可以將 token 儲存在 HTML `meta` 標籤中：

```blade
<meta name="csrf-token" content="{{ csrf_token() }}">
```

然後，您可以指示像 jQuery 這樣的函式庫自動將 token 添加到所有請求標頭中。這為您使用傳統 JavaScript 技術的 AJAX 應用程式提供了簡單、方便的 CSRF 保護：

```js
$.ajaxSetup({
    headers: {
        'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
    }
});
```

<a name="csrf-x-xsrf-token"></a>
## X-XSRF-TOKEN

Laravel 會將目前的 CSRF token 儲存在一個加密的 `XSRF-TOKEN` Cookie 中，該 Cookie 會隨框架產生的每個回應一起包含。您可以使用 Cookie 值來設定 `X-XSRF-TOKEN` 請求標頭。

此 Cookie 主要作為開發人員的便利性而發送，因為某些 JavaScript 框架和函式庫，例如 Angular 和 Axios，會自動將其值放置在同源請求的 `X-XSRF-TOKEN` 標頭中。

> [!NOTE]
> 預設情況下，`resources/js/bootstrap.js` 檔案包含 Axios HTTP 函式庫，它會自動為您發送 `X-XSRF-TOKEN` 標頭。
</form>
```

<a name="accessing-the-current-route"></a>
## 存取目前路由

您可以使用 `Route` Facade 上的 `current`、`currentRouteName` 和 `currentRouteAction` 方法來存取有關處理傳入請求的路由資訊：

```php
use Illuminate\Support\Facades\Route;

$route = Route::current(); // Illuminate\Routing\Route
$name = Route::currentRouteName(); // string
$action = Route::currentRouteAction(); // string
```

您可以參考 [Route Facade 的底層類別](https://api.laravel.com/docs/{{version}}/Illuminate/Routing/Router.html) 和 [Route 實例](https://api.laravel.com/docs/{{version}}/Illuminate/Routing/Route.html) 的 API 說明文件，以檢閱路由器和路由類別上所有可用的方法。

<a name="cors"></a>
## 跨來源資源共用 (CORS)

Laravel 可以自動回應 CORS `OPTIONS` HTTP 請求，並帶有您設定的值。`OPTIONS` 請求將由自動包含在應用程式全域 Middleware 堆疊中的 `HandleCors` [Middleware](/docs/{{version}}/middleware) 自動處理。

有時，您可能需要自訂應用程式的 CORS 設定值。您可以透過使用 `config:publish` Artisan 命令發布 `cors` 設定檔來實現：

```shell
php artisan config:publish cors
```

此命令將在應用程式的 `config` 目錄中放置一個 `cors.php` 設定檔。

> [!NOTE]
> 有關 CORS 和 CORS 標頭的更多資訊，請查閱 [MDN 網頁上關於 CORS 的說明文件](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#The_HTTP_response_headers)。

<a name="route-caching"></a>
## 路由快取

當將您的應用程式部署到生產環境時，您應該利用 Laravel 的路由快取。使用路由快取將大幅減少註冊應用程式所有路由所需的時間。要產生路由快取，請執行 `route:cache` Artisan 命令：

```shell
php artisan route:cache
```

執行此命令後，您的快取路由檔案將在每個請求上載入。請記住，如果您添加任何新路由，您將需要產生一個新的路由快取。因此，您應該只在專案部署期間執行 `route:cache` 命令。

您可以使用 `route:clear` 命令來清除路由快取：

```shell
php artisan route:clear
```
