# 路由

- [基本路由](#basic-routing)
    - [預設路由檔案](#the-default-route-files)
    - [重新導向路由](#redirect-routes)
    - [視圖路由](#view-routes)
    - [列出路由](#listing-your-routes)
    - [路由自訂](#routing-customization)
- [路由參數](#route-parameters)
    - [必要參數](#required-parameters)
    - [選用參數](#parameters-optional-parameters)
    - [正規表達式約束](#parameters-regular-expression-constraints)
- [命名路由](#named-routes)
- [路由群組](#route-groups)
    - [中介層](#route-group-middleware)
    - [控制器](#route-group-controllers)
    - [子網域路由](#route-group-subdomain-routing)
    - [路由前綴](#route-group-prefixes)
    - [路由名稱前綴](#route-group-name-prefixes)
- [路由模型繫結](#route-model-binding)
    - [隱式繫結](#implicit-binding)
    - [隱式列舉繫結](#implicit-enum-binding)
    - [顯式繫結](#explicit-binding)
- [備援路由](#fallback-routes)
- [流量限制](#rate-limiting)
    - [定義流量限制器](#defining-rate-limiters)
    - [將流量限制器附加到路由](#attaching-rate-limiters-to-routes)
- [表單方法偽造](#form-method-spoofing)
- [存取目前路由](#accessing-the-current-route)
- [跨來源資源共用 (CORS)](#cors)
- [路由快取](#route-caching)

<a name="basic-routing"></a>
## 基本路由

最基本的 Laravel 路由接受一個 URI 和一個閉包，提供一種非常簡單且富有表達力的方法來定義路由與行為，而無需複雜的路由設定檔：

```php
use Illuminate\Support\Facades\Route;

Route::get('/greeting', function () {
    return 'Hello World';
});
```


<a name="the-default-route-files"></a>
### 預設路由檔案

所有 Laravel 路由都定義在你的路由檔案中，這些檔案位於 `routes` 目錄。Laravel 會使用你的應用程式 `bootstrap/app.php` 檔案中指定的設定，自動載入這些檔案。`routes/web.php` 檔案定義了用於你的網路介面的路由。這些路由被指派給 `web` [中介層群組](/docs/{{version}}/middleware#laravels-default-middleware-groups)，它提供了諸如會話狀態 (session state) 和 CSRF 保護等功能。

對於大多數應用程式，你會從在 `routes/web.php` 檔案中定義路由開始。定義在 `routes/web.php` 中的路由可以透過在瀏覽器中輸入路由的 URL 來存取。例如，你可以透過在瀏覽器中導航到 `http://example.com/user` 來存取以下路由：

```php
use App\Http\Controllers\UserController;

Route::get('/user', [UserController::class, 'index']);
```


<a name="api-routes"></a>
#### API 路由

如果你的應用程式也將提供無狀態的 API，你可以使用 `install:api` Artisan 命令來啟用 API 路由：

```shell
php artisan install:api
```

`install:api` 命令會安裝 [Laravel Sanctum](/docs/{{version}}/sanctum)，它提供一個強大而簡單的 API token 驗證守衛，可用於驗證第三方 API 消費者、SPA 或行動應用程式。此外，`install:api` 命令會建立 `routes/api.php` 檔案：

```php
Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```

`routes/api.php` 中的路由是無狀態的，並被指派給 `api` [中介層群組](/docs/{{version}}/middleware#laravels-default-middleware-groups)。此外，`/api` URI 前綴會自動應用於這些路由，因此你無需手動將其應用於檔案中的每個路由。你可以透過修改應用程式的 `bootstrap/app.php` 檔案來變更前綴：

```php
->withRouting(
    api: __DIR__.'/../routes/api.php',
    apiPrefix: 'api/admin',
    // ...
)
```


<a name="available-router-methods"></a>
#### 可用的路由器方法

路由器允許你註冊響應任何 HTTP 動詞的路由：

```php
Route::get($uri, $callback);
Route::post($uri, $callback);
Route::put($uri, $callback);
Route::patch($uri, $callback);
Route::delete($uri, $callback);
Route::options($uri, $callback);
```

有時你可能需要註冊一個響應多個 HTTP 動詞的路由。你可以使用 `match` 方法來做到這一點。或者，你甚至可以使用 `any` 方法註冊一個響應所有 HTTP 動詞的路由：

```php
Route::match(['get', 'post'], '/', function () {
    // ...
});

Route::any('/', function () {
    // ...
});
```

> [!NOTE]
> 當定義多個共用相同 URI 的路由時，使用 `get`、`post`、`put`、`patch`、`delete` 和 `options` 方法的路由應在使用 `any`、`match` 和 `redirect` 方法的路由之前定義。這可確保傳入的請求與正確的路由匹配。


<a name="dependency-injection"></a>
#### 依賴注入

你可以在路由的回呼函式簽章中類型提示路由所需的任何依賴。聲明的依賴將會被 Laravel [服務容器](/docs/{{version}}/container)自動解析並注入到回呼函式中。例如，你可以類型提示 `Illuminate\Http\Request` 類別，以便將當前 HTTP 請求自動注入到你的路由回呼函式中：

```php
use Illuminate\Http\Request;

Route::get('/users', function (Request $request) {
    // ...
});
```


<a name="csrf-protection"></a>
#### CSRF 保護

請記住，任何指向 `web` 路由檔案中定義的 `POST`、`PUT`、`PATCH` 或 `DELETE` 路由的 HTML 表單都應包含一個 CSRF 令牌欄位。否則，請求將被拒絕。你可以在 [CSRF 文件](/docs/{{version}}/csrf)中閱讀更多關於 CSRF 保護的資訊：

```blade
<form method="POST" action="/profile">
    @csrf
    ...
</form>
```


<a name="redirect-routes"></a>
### 重新導向路由

如果你正在定義一個重新導向到另一個 URI 的路由，你可以使用 `Route::redirect` 方法。此方法提供了一個方便的捷徑，讓你無需為執行簡單的重新導向而定義完整的路由或控制器：

```php
Route::redirect('/here', '/there');
```

預設情況下，`Route::redirect` 會回傳 `302` 狀態碼。你可以使用選用的第三個參數來自訂狀態碼：

```php
Route::redirect('/here', '/there', 301);
```

或者，你可以使用 `Route::permanentRedirect` 方法來回傳 `301` 狀態碼：

```php
Route::permanentRedirect('/here', '/there');
```

> [!WARNING]
> 在重新導向路由中使用路由參數時，以下參數由 Laravel 保留且不能使用：`destination` 和 `status`。


<a name="view-routes"></a>
### 視圖路由

如果你的路由只需要回傳一個 [視圖](/docs/{{version}}/views)，你可以使用 `Route::view` 方法。與 `redirect` 方法一樣，此方法提供了一個簡單的捷徑，讓你無需定義完整的路由或控制器。`view` 方法接受一個 URI 作為它的第一個參數，以及一個視圖名稱作為它的第二個參數。此外，你可以提供一個資料陣列作為選用的第三個參數傳遞給視圖：

```php
Route::view('/welcome', 'welcome');

Route::view('/welcome', 'welcome', ['name' => 'Taylor']);
```

> [!WARNING]
> 在視圖路由中使用路由參數時，以下參數由 Laravel 保留且不能使用：`view`、`data`、`status` 和 `headers`。


<a name="listing-your-routes"></a>
### 列出路由

`route:list` Artisan 命令可以輕鬆地提供應用程式中所有已定義路由的總覽：

```shell
php artisan route:list
```

預設情況下，分配給每個路由的路由中介層將不會顯示在 `route:list` 輸出中；但是，你可以透過向命令新增 `-v` 選項來指示 Laravel 顯示路由中介層和中介層群組名稱：

```shell
php artisan route:list -v

# Expand middleware groups...
php artisan route:list -vv
```

您還可以指示 Laravel 僅顯示以指定 URI 開頭的路由：

```shell
php artisan route:list --path=api
```

此外，您可以透過在執行 `route:list` 指令時提供 `--except-vendor` 選項，指示 Laravel 隱藏任何由第三方套件定義的路由：

```shell
php artisan route:list --except-vendor
```

同樣地，您也可以透過在執行 `route:list` 指令時提供 `--only-vendor` 選項，指示 Laravel 僅顯示由第三方套件定義的路由：

```shell
php artisan route:list --only-vendor
```


<a name="routing-customization"></a>
### 路由自訂

預設情況下，您的應用程式路由由 `bootstrap/app.php` 檔案配置和載入：

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

然而，有時候您可能需要定義一個全新的檔案來包含應用程式路由的一個子集。為了實現這一點，您可以為 `withRouting` 方法提供一個 `then` 閉包。在此閉包中，您可以註冊應用程式所需的任何額外路由：

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

或者，您甚至可以透過為 `withRouting` 方法提供一個 `using` 閉包來完全控制路由註冊。當傳遞此參數時，框架將不會註冊任何 HTTP 路由，您將負責手動註冊所有路由：

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

有時候您需要捕獲 URI 的片段在您的路由中。例如，您可能需要從 URL 中捕獲使用者的 ID。您可以透過定義路由參數來實現：

```php
Route::get('/user/{id}', function (string $id) {
    return 'User '.$id;
});
```

您可以根據路由的需求定義任意數量的路由參數：

```php
Route::get('/posts/{post}/comments/{comment}', function (string $postId, string $commentId) {
    // ...
});
```

路由參數總是包含在 `{}` 大括號中，並且應該由字母組成。在路由參數名稱中也接受底線 (`_`)。路由參數是根據它們的順序注入到路由回呼 / 控制器中的，路由回呼 / 控制器參數的名稱並不重要。


<a name="parameters-and-dependency-injection"></a>
#### 參數與依賴注入

如果您的路由有依賴項，您希望 Laravel 服務容器自動注入到您的路由回呼中，您應該在依賴項之後列出您的路由參數：

```php
use Illuminate\Http\Request;

Route::get('/user/{id}', function (Request $request, string $id) {
    return 'User '.$id;
});
```


<a name="parameters-optional-parameters"></a>
### 選用參數

有時候您可能需要指定一個路由參數，該參數可能不會總是出現在 URI 中。您可以透過在參數名稱後加上 `?` 標記來實現。請確保為路由的對應變數提供一個預設值：

```php
Route::get('/user/{name?}', function (?string $name = null) {
    return $name;
});

Route::get('/user/{name?}', function (?string $name = 'John') {
    return $name;
});
```


<a name="parameters-regular-expression-constraints"></a>
### 正規表達式約束

您可以限制路由參數的格式，使用路由實例上的 `where` 方法。`where` 方法接受參數的名稱和一個正規表達式，定義參數應如何被約束：

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

為了方便，一些常用的正規表達式模式提供了輔助方法，讓您能夠快速地為路由添加模式約束：

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

如果傳入的請求不符合路由模式約束，將會返回一個 404 HTTP 回應。


<a name="parameters-global-constraints"></a>
#### 全域約束

如果您希望路由參數總是受限於給定的正規表達式，您可以使用 `pattern` 方法。您應該在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中定義這些模式：

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

一旦模式被定義，它會自動應用於所有使用該參數名稱的路由：

```php
Route::get('/user/{id}', function (string $id) {
    // Only executed if {id} is numeric...
});
```


<a name="parameters-encoded-forward-slashes"></a>
#### 編碼的正斜線

Laravel 路由組件允許所有字元，除了 `/` 之外，存在於路由參數值中。您必須明確允許 `/` 成為您佔位符的一部分，使用 `where` 條件正規表達式：

```php
Route::get('/search/{search}', function (string $search) {
    return $search;
})->where('search', '.*');
```

> [!WARNING]
> 編碼的正斜線僅在最後一個路由段中受支援。

<a name="named-routes"></a>
## 命名路由

命名路由允許方便地產生特定路由的 URL 或重新導向。您可以透過在路由定義上鏈接 `name` 方法來為路由指定名稱：

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
#### 產生命名路由的 URL

一旦您為給定路由賦予名稱，您就可以在透過 Laravel 的 `route` 和 `redirect` 輔助函式產生 URL 或重新導向時使用該路由的名稱：

```php
// Generating URLs...
$url = route('profile');

// Generating Redirects...
return redirect()->route('profile');

return to_route('profile');
```

如果命名路由定義了參數，您可以將參數作為 `route` 函式的第二個引數傳入。給定的參數將自動插入到產生的 URL 中正確的位置：

```php
Route::get('/user/{id}/profile', function (string $id) {
    // ...
})->name('profile');

$url = route('profile', ['id' => 1]);
```

如果您在陣列中傳遞額外參數，這些鍵值對將自動添加到產生的 URL 的查詢字串中：

```php
Route::get('/user/{id}/profile', function (string $id) {
    // ...
})->name('profile');

$url = route('profile', ['id' => 1, 'photos' => 'yes']);

// /user/1/profile?photos=yes
```

> [!NOTE]
> 有時，您可能希望為 URL 參數指定請求範圍的預設值，例如當前語系。為此，您可以使用 [URL::defaults 方法](/docs/{{version}}/urls#default-values)。


<a name="inspecting-the-current-route"></a>
#### 檢查目前路由

如果您想判斷目前的請求是否被路由到給定的命名路由，您可以在 Route 實例上使用 `named` 方法。例如，您可以從路由中介層中檢查當前路由名稱：

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

路由群組允許您在大量路由之間共用路由屬性，例如中介層，而無需在每個獨立路由上定義這些屬性。

巢狀群組會嘗試智慧地將屬性與其父群組「合併」。中介層和 `where` 條件會被合併，而名稱和前綴則會被附加。命名空間分隔符和 URI 前綴中的斜線會自動在適當的位置添加。


<a name="route-group-middleware"></a>
### 中介層

要為群組中的所有路由分配 [中介層](/docs/{{version}}/middleware)，您可以在定義群組之前使用 `middleware` 方法。中介層按照它們在陣列中列出的順序執行：

```php
Route::middleware(['first', 'second'])->group(function () {
    Route::get('/', function () {
        // Uses first & second middleware...
    });

    Route::get('/user/profile', function () {
        // Uses first & second middleware...
    });
});
```


<a name="route-group-controllers"></a>
### 控制器

如果一組路由都使用相同的 [控制器](/docs/{{version}}/controllers)，您可以使用 `controller` 方法為群組中的所有路由定義共同的控制器。然後，在定義路由時，您只需提供它們調用的控制器方法：

```php
use App\Http\Controllers\OrderController;

Route::controller(OrderController::class)->group(function () {
    Route::get('/orders/{id}', 'show');
    Route::post('/orders', 'store');
});
```


<a name="route-group-subdomain-routing"></a>
### 子網域路由

路由群組也可用於處理子網域路由。子網域可以像路由 URI 一樣分配路由參數，允許您擷取子網域的一部分以用於路由或控制器。可以在定義群組之前呼叫 `domain` 方法來指定子網域：

```php
Route::domain('{account}.example.com')->group(function () {
    Route::get('/user/{id}', function (string $account, string $id) {
        // ...
    });
});
```

> [!WARNING]
> 為確保您的子網域路由可達，您應該在註冊根網域路由之前註冊子網域路由。這將防止根網域路由覆寫具有相同 URI 路徑的子網域路由。


<a name="route-group-prefixes"></a>
### 路由前綴

`prefix` 方法可用於為群組中的每個路由加上給定的 URI 前綴。例如，您可能希望為群組中所有路由 URI 加上 `admin` 前綴：

```php
Route::prefix('admin')->group(function () {
    Route::get('/users', function () {
        // Matches The "/admin/users" URL
    });
});
```


<a name="route-group-name-prefixes"></a>
### 路由名稱前綴

`name` 方法可用於為群組中的每個路由名稱加上給定的字串前綴。例如，您可能希望為群組中所有路由的名稱加上 `admin` 前綴。給定的字串會完全按照指定的方式作為路由名稱的前綴，因此我們務必在該前綴中提供尾隨的 `.` 字元。

```php
Route::name('admin.')->group(function () {
    Route::get('/users', function () {
        // Route assigned name "admin.users"...
    })->name('users');
});
```

<a name="route-model-binding"></a>
## 路由模型繫結

當將模型 ID 注入到路由或控制器行為時，您通常會查詢資料庫以擷取與該 ID 對應的模型。Laravel 路由模型繫結提供了一種便利的方式，可以將模型實例直接自動注入到您的路由中。例如，您無需注入使用者的 ID，而是可以注入符合該 ID 的整個 `User` 模型實例。


<a name="implicit-binding"></a>
### 隱式繫結

Laravel 會自動解析在路由或控制器行為中定義的 Eloquent 模型，其型別提示的變數名稱符合路由區段名稱。例如：

```php
use App\Models\User;

Route::get('/users/{user}', function (User $user) {
    return $user->email;
});
```

由於 `$user` 變數被型別提示為 `App\Models\User` Eloquent 模型，且變數名稱符合 `{user}` URI 區段，Laravel 將自動注入 ID 符合請求 URI 中對應值的模型實例。如果在資料庫中未找到符合的模型實例，將自動產生 404 HTTP 回應。

當然，使用控制器方法時也適用隱式繫結。再次注意，`{user}` URI 區段符合控制器中包含 `App\Models\User` 型別提示的 `$user` 變數：

```php
use App\Http\Controllers\UserController;
use App\Models\User;

// Route definition...
Route::get('/users/{user}', [UserController::class, 'show']);

// Controller method definition...
public function show(User $user)
{
    return view('user.profile', ['user' => $user]);
}
```


<a name="implicit-soft-deleted-models"></a>
#### 軟刪除模型

通常，隱式模型繫結不會擷取已 [軟刪除](/docs/{{version}}/eloquent#soft-deleting) 的模型。但是，您可以透過將 `withTrashed` 方法鏈接到您的路由定義上，來指示隱式繫結擷取這些模型：

```php
use App\Models\User;

Route::get('/users/{user}', function (User $user) {
    return $user->email;
})->withTrashed();
```


<a name="customizing-the-default-key-name"></a>
#### 自訂鍵

有時您可能希望使用 `id` 以外的欄位來解析 Eloquent 模型。為此，您可以在路由參數定義中指定欄位：

```php
use App\Models\Post;

Route::get('/posts/{post:slug}', function (Post $post) {
    return $post;
});
```

如果您希望模型繫結在擷取給定的模型類別時，總是使用 `id` 以外的資料庫欄位，您可以覆寫 Eloquent 模型上的 `getRouteKeyName` 方法：

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
#### 自訂鍵與範圍限定

當在單一路由定義中隱式繫結多個 Eloquent 模型時，您可能希望範圍限定第二個 Eloquent 模型，使其必須是前一個 Eloquent 模型的子級。例如，考慮這個透過 slug 擷取特定使用者部落格文章的路由定義：

```php
use App\Models\Post;
use App\Models\User;

Route::get('/users/{user}/posts/{post:slug}', function (User $user, Post $post) {
    return $post;
});
```

當使用自訂鍵的隱式繫結作為巢狀路由參數時，Laravel 將自動範圍限定查詢，透過其父級使用慣例猜測父級上的關聯名稱來擷取巢狀模型。在此情況下，將會假設 `User` 模型有一個名為 `posts` 的關聯 (路由參數名稱的複數形式)，可用於擷取 `Post` 模型。

如果您願意，您可以指示 Laravel 範圍限定「子級」繫結，即使未提供自訂鍵。為此，您可以在定義路由時調用 `scopeBindings` 方法：

```php
use App\Models\Post;
use App\Models\User;

Route::get('/users/{user}/posts/{post}', function (User $user, Post $post) {
    return $post;
})->scopeBindings();
```

或者，您可以指示整個路由定義群組使用範圍限定的繫結：

```php
Route::scopeBindings()->group(function () {
    Route::get('/users/{user}/posts/{post}', function (User $user, Post $post) {
        return $post;
    });
});
```

同樣地，您可以透過調用 `withoutScopedBindings` 方法，明確指示 Laravel 不要範圍限定繫結：

```php
Route::get('/users/{user}/posts/{post:slug}', function (User $user, Post $post) {
    return $post;
})->withoutScopedBindings();
```


<a name="customizing-missing-model-behavior"></a>
#### 自訂遺失模型的行為

通常，如果未找到隱式繫結的模型，將會產生 404 HTTP 回應。但是，您可以在定義路由時呼叫 `missing` 方法來自訂此行為。`missing` 方法接受一個閉包，當找不到隱式繫結的模型時將會調用該閉包：

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
### 隱式列舉繫結

PHP 8.1 引入了對 [Enums](https://www.php.net/manual/en/language.enumerations.backed.php) 的支援。為了補充此功能，Laravel 允許您在路由定義上型別提示 [字符串支援的 Enum](https://www.php.net/manual/en/language.enumerations.backed.php)，並且 Laravel 僅在該路由區段對應到有效的 Enum 值時才會調用路由。否則，將自動返回 404 HTTP 回應。例如，給定以下 Enum：

```php
<?php

namespace App\Enums;

enum Category: string
{
    case Fruits = 'fruits';
    case People = 'people';
}
```

您可以定義一個路由，該路由僅在 `{category}` 路由區段為 `fruits` 或 `people` 時才會被調用。否則，Laravel 將返回 404 HTTP 回應：

```php
use App\Enums\Category;
use Illuminate\Support\Facades\Route;

Route::get('/categories/{category}', function (Category $category) {
    return $category->value;
});
```


<a name="explicit-binding"></a>
### 顯式繫結

您不需要使用 Laravel 的隱式、基於慣例的模型解析才能使用模型繫結。您也可以明確定義路由參數如何對應到模型。要註冊顯式繫結，請使用路由器的 `model` 方法為給定參數指定類別。您應該在 `AppServiceProvider` 類別的 `boot` 方法開頭定義您的顯式模型繫結：

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

由於我們已將所有 `{user}` 參數繫結到 `App\Models\User` 模型，因此該類別的實例將被注入到路由中。舉例來說，對 `users/1` 的請求將會注入資料庫中 ID 為 `1` 的 `User` 實例。

如果在資料庫中未找到符合的模型實例，將自動產生 404 HTTP 回應。


<a name="customizing-the-resolution-logic"></a>
#### 自訂解析邏輯

如果您希望定義自己的模型繫結解析邏輯，您可以使用 `Route::bind` 方法。您傳遞給 `bind` 方法的閉包將接收 URI 區段的值，並且應該返回應注入到路由中的類別實例。同樣地，此自訂應在您的應用程式 `AppServiceProvider` 的 `boot` 方法中進行：

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

或者，您可以覆寫 Eloquent 模型上的 `resolveRouteBinding` 方法。此方法將接收 URI 區段的值，並且應該返回應注入到路由中的類別實例：

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

如果路由正在使用 [隱式繫結範圍限定](#implicit-model-binding-scoping)，則 `resolveChildRouteBinding` 方法將用於解析父級模型的子級繫結：

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

透過使用 `Route::fallback` 方法，您可以定義一個路由，當沒有其他路由與傳入的請求匹配時，該路由將會被執行。通常，未處理的請求將透過您應用程式的例外處理器自動渲染「404」頁面。然而，由於您通常會在 `routes/web.php` 檔案中定義 `fallback` 路由，因此 `web` 中介層群組中的所有中介層都將應用於該路由。您可以根據需要為此路由新增額外的中介層：

```php
Route::fallback(function () {
    // ...
});
```

<a name="rate-limiting"></a>
## 流量限制

<a name="defining-rate-limiters"></a>
### 定義流量限制器

Laravel 包含強大且可自訂的流量限制服務，您可以使用這些服務來限制特定路由或路由群組的流量。首先，您應該定義符合您應用程式需求的流量限制器配置。

流量限制器可以在您應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中定義：

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

流量限制器是透過使用 `RateLimiter` Facade 的 `for` 方法來定義的。`for` 方法接受一個流量限制器名稱和一個閉包 (closure)，該閉包返回應應用於分配給該流量限制器的路由的限制配置。限制配置是 `Illuminate\Cache\RateLimiting\Limit` 類別的實例。此類別包含有用的「建構器」方法，以便您快速定義限制。流量限制器名稱可以是您想要的任何字串：

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

如果傳入的請求超過指定的流量限制，Laravel 將自動返回帶有 429 HTTP 狀態碼的回應。如果您想定義自己的回應，該回應應由流量限制器返回，您可以使用 `response` 方法：

```php
RateLimiter::for('global', function (Request $request) {
    return Limit::perMinute(1000)->response(function (Request $request, array $headers) {
        return response('Custom response...', 429, $headers);
    });
});
```

由於流量限制器回呼 (callbacks) 會接收傳入的 HTTP 請求實例，您可以根據傳入的請求或已驗證的使用者動態地建立適當的流量限制：

```php
RateLimiter::for('uploads', function (Request $request) {
    return $request->user()->vipCustomer()
        ? Limit::none()
        : Limit::perHour(10);
});
```

<a name="segmenting-rate-limits"></a>
#### 區分流量限制

有時您可能希望透過某個任意值來區分流量限制。例如，您可能希望允許使用者每 IP 位址每分鐘存取指定路由 100 次。為此，您可以在建立流量限制時使用 `by` 方法：

```php
RateLimiter::for('uploads', function (Request $request) {
    return $request->user()->vipCustomer()
        ? Limit::none()
        : Limit::perMinute(100)->by($request->ip());
});
```

為了透過另一個範例來說明此功能，我們可以限制路由的存取，對於已驗證的使用者 ID，每分鐘 100 次；對於訪客，每分鐘 10 次，每 IP 位址：

```php
RateLimiter::for('uploads', function (Request $request) {
    return $request->user()
        ? Limit::perMinute(100)->by($request->user()->id)
        : Limit::perMinute(10)->by($request->ip());
});
```

<a name="multiple-rate-limits"></a>
#### 多重流量限制

如有需要，您可以為給定的流量限制器配置返回一個流量限制陣列。每個流量限制將根據它們在陣列中的順序對路由進行評估：

```php
RateLimiter::for('login', function (Request $request) {
    return [
        Limit::perMinute(500),
        Limit::perMinute(3)->by($request->input('email')),
    ];
});
```

如果您正在分配多個由相同 `by` 值區分的流量限制，您應該確保每個 `by` 值都是唯一的。實現這一點的最簡單方法是為傳遞給 `by` 方法的值加上前綴：

```php
RateLimiter::for('uploads', function (Request $request) {
    return [
        Limit::perMinute(10)->by('minute:'.$request->user()->id),
        Limit::perDay(1000)->by('day:'.$request->user()->id),
    ];
});
```

<a name="response-base-rate-limiting"></a>
#### 基於回應的流量限制

除了限制傳入的請求之外，Laravel 還允許您使用 `after` 方法根據回應來限制流量。當您只想將某些回應計入流量限制時，這會很有用，例如驗證錯誤、404 回應或其他特定的 HTTP 狀態碼。

`after` 方法接受一個閉包，該閉包接收回應，如果回應應計入流量限制，則應返回 `true`，如果應忽略，則返回 `false`。這對於透過限制連續的 404 回應來防止列舉攻擊，或者允許使用者重試驗證失敗的請求而不會耗盡其在應僅限制成功操作的端點上的流量限制特別有用：

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;
use Symfony\Component\HttpFoundation\Response;

RateLimiter::for('resource-not-found', function (Request $request) {
    return Limit::perMinute(10)
        ->by($request->user()?->id ?: $request->ip())
        ->after(function (Response $response) {
            // Only count 404 responses toward the rate limit to prevent enumeration...
            return $response->status() === 404;
        });
});
```

<a name="attaching-rate-limiters-to-routes"></a>
### 將流量限制器附加到路由

流量限制器可以使用 `throttle` [中介層](/docs/{{version}}/middleware) 附加到路由或路由群組。這個 throttle 中介層接受您希望分配給路由的流量限制器名稱：

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
#### 使用 Redis 進行流量限制

預設情況下，`throttle` 中介層被映射到 `Illuminate\Routing\Middleware\ThrottleRequests` 類別。然而，如果您使用 Redis 作為應用程式的快取驅動程式，您可能希望指示 Laravel 使用 Redis 來管理流量限制。為此，您應該在應用程式的 `bootstrap/app.php` 檔案中使用 `throttleWithRedis` 方法。此方法將 `throttle` 中介層映射到 `Illuminate\Routing\Middleware\ThrottleRequestsWithRedis` 中介層類別：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->throttleWithRedis();
    // ...
})
```

<a name="form-method-spoofing"></a>
## 表單方法偽造

HTML 表單不支援 `PUT`、`PATCH` 或 `DELETE` 動作。因此，當定義從 HTML 表單呼叫的 `PUT`、`PATCH` 或 `DELETE` 路由時，您需要向表單新增一個隱藏的 `_method` 欄位。隨 `_method` 欄位發送的值將用作 HTTP 請求方法：

```blade
<form action="/example" method="POST">
    <input type="hidden" name="_method" value="PUT">
    <input type="hidden" name="_token" value="{{ csrf_token() }}">
</form>
```

為方便起見，您可以使用 `@method` [Blade 指令](/docs/{{version}}/blade) 來產生 `_method` 輸入欄位：

```blade
<form action="/example" method="POST">
    @method('PUT')
    @csrf
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

您可以參考 [Route facade 的基礎類別](https://api.laravel.com/docs/{{version}}/Illuminate/Routing/Router.html) 和 [Route 實例](https://api.laravel.com/docs/{{version}}/Illuminate/Routing/Route.html) 的 API 文件，以查閱路由器和路由類別上所有可用的方法。

<a name="cors"></a>
## 跨來源資源共用 (CORS)

Laravel 可以自動回應您設定的 CORS `OPTIONS` HTTP 請求。`OPTIONS` 請求將由您的應用程式全域中介層堆疊中自動包含的 `HandleCors` [中介層](/docs/{{version}}/middleware) 自動處理。

有時，您可能需要自訂應用程式的 CORS 設定值。您可以透過 `config:publish` Artisan 命令發佈 `cors` 設定檔來實現此目的：

```shell
php artisan config:publish cors
```

此命令將在您的應用程式 `config` 目錄中放置一個 `cors.php` 設定檔。

> [!NOTE]
> 有關 CORS 和 CORS 標頭的更多資訊，請查閱 [MDN 關於 CORS 的網路文件](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#The_HTTP_response_headers)。

<a name="route-caching"></a>
## 路由快取

將應用程式部署到生產環境時，您應該利用 Laravel 的路由快取。使用路由快取將大幅減少註冊所有應用程式路由所需的時間。要產生路由快取，請執行 `route:cache` Artisan 命令：

```shell
php artisan route:cache
```

執行此命令後，您的快取路由檔案將在每個請求時載入。請記住，如果您新增任何新路由，則需要重新產生路由快取。因此，您應該僅在專案部署期間執行 `route:cache` 命令。

您可以使用 `route:clear` 命令來清除路由快取：

```shell
php artisan route:clear
```