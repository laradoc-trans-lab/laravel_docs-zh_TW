# 路由 (Routing)

- [基礎路由](#basic-routing)
    - [預設路由檔案](#the-default-route-files)
    - [重導向路由](#redirect-routes)
    - [視圖路由](#view-routes)
    - [列出您的路由](#listing-your-routes)
    - [路由自定義](#routing-customization)
- [路由參數](#route-parameters)
    - [必填參數](#required-parameters)
    - [選填參數](#parameters-optional-parameters)
    - [正規表示式限制](#parameters-regular-expression-constraints)
- [具名路由](#named-routes)
- [路由群組](#route-groups)
    - [中介層](#route-group-middleware)
    - [控制器](#route-group-controllers)
    - [子網域路由](#route-group-subdomain-routing)
    - [路由前綴](#route-group-prefixes)
    - [路由名稱前綴](#route-group-name-prefixes)
- [路由模型綁定](#route-model-binding)
    - [隱式綁定](#implicit-binding)
    - [隱式 Enum 綁定](#implicit-enum-binding)
    - [顯式綁定](#explicit-binding)
- [備用路由](#fallback-routes)
- [速率限制](#rate-limiting)
    - [定義速率限制器](#defining-rate-limiters)
    - [將速率限制器附加到路由](#attaching-rate-limiters-to-routes)
- [表單方法欺騙](#form-method-spoofing)
- [存取當前路由](#accessing-the-current-route)
- [跨來源資源共享 (CORS)](#cors)
- [路由快取](#route-caching)

<a name="basic-routing"></a>
## 基礎路由

最基本的 Laravel 路由接受一個 URI 和一個閉包 (Closure)，提供了一種非常簡單且具有表現力的方法來定義路由和行為，而無需複雜的路由設定檔：

```php
use Illuminate\Support\Facades\Route;

Route::get('/greeting', function () {
    return 'Hello World';
});
```


<a name="the-default-route-files"></a>
### 預設路由檔案

所有的 Laravel 路由都定義在 `routes` 目錄下的路由檔案中。Laravel 會根據應用程式 `bootstrap/app.php` 檔案中指定的設定，自動載入這些檔案。`routes/web.php` 檔案定義了用於 Web 介面的路由。這些路由被分配了 `web` [中介層群組 (Middleware Group)](/docs/{{version}}/middleware#laravels-default-middleware-groups)，該群組提供了 Session 狀態和 CSRF 保護等功能。

對於大多數應用程式，您將從在 `routes/web.php` 檔案中定義路由開始。在 `routes/web.php` 中定義的路由可以透過在瀏覽器中輸入定義的路由 URL 來存取。例如，您可以透過在瀏覽器中導航到 `http://example.com/user` 來存取以下路由：

```php
use App\Http\Controllers\UserController;

Route::get('/user', [UserController::class, 'index']);
```


<a name="api-routes"></a>
#### API 路由

如果您的應用程式也提供無狀態 API，您可以使用 `install:api` Artisan 指令啟用 API 路由：

```shell
php artisan install:api
```

`install:api` 指令會安裝 [Laravel Sanctum](/docs/{{version}}/sanctum)，它提供了一個強大且簡單的 API 權杖 (Token) 認證防護，可用於驗證第三方 API 消費者、SPA 或行動應用程式。此外，`install:api` 指令會建立 `routes/api.php` 檔案：

```php
Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```

當然，對於應該公開存取的路由，您可以自由地省略 `auth:sanctum` 中介層。

`routes/api.php` 中的路由是無狀態的，並被分配到 `api` [中介層群組](/docs/{{version}}/middleware#laravels-default-middleware-groups)。此外，這些路由會自動套用 `/api` URI 前綴，因此您不需要在檔案中的每個路由上手動套用它。您可以透過修改應用程式的 `bootstrap/app.php` 檔案來更改前綴：

```php
->withRouting(
    api: __DIR__.'/../routes/api.php',
    apiPrefix: 'api/admin',
    // ...
)
```


<a name="available-router-methods"></a>
#### 可用的路由方法

路由器允許您註冊回應任何 HTTP 動詞的路由：

```php
Route::get($uri, $callback);
Route::post($uri, $callback);
Route::put($uri, $callback);
Route::patch($uri, $callback);
Route::delete($uri, $callback);
Route::options($uri, $callback);
```

有時您可能需要註冊一個回應多個 HTTP 動詞的路由。您可以使用 `match` 方法來做到這一點。或者，您甚至可以使用 `any` 方法註冊一個回應所有 HTTP 動詞的路由：

```php
Route::match(['get', 'post'], '/', function () {
    // ...
});

Route::any('/', function () {
    // ...
});
```

> [!NOTE]
> 當定義多個共享相同 URI 的路由時，使用 `get`、`post`、`put`、`patch`、`delete` 和 `options` 方法的路由應該在定義使用 `any`、`match` 和 `redirect` 方法的路由之前。這可以確保傳入的請求與正確的路由匹配。


<a name="dependency-injection"></a>
#### 依賴注入

您可以在路由的閉包簽章中對路由所需的任何依賴項進行型別提示 (Type-hint)。聲明的依賴項將由 Laravel [服務容器 (Service Container)](/docs/{{version}}/container) 自動解析並注入到閉包中。例如，您可以對 `Illuminate\Http\Request` 類別進行型別提示，以便將當前的 HTTP 請求自動注入到您的路由閉包中：

```php
use Illuminate\Http\Request;

Route::get('/users', function (Request $request) {
    // ...
});
```


<a name="csrf-protection"></a>
#### CSRF 保護

請記住，任何指向在 `web` 路由檔案中定義的 `POST`、`PUT`、`PATCH` 或 `DELETE` 路由的 HTML 表單，都應該包含一個 CSRF 權杖欄位。否則，請求將被拒絕。您可以在 [CSRF 文件](/docs/{{version}}/csrf) 中閱讀有關 CSRF 保護的更多資訊：

```blade
<form method="POST" action="/profile">
    @csrf
    ...
</form>
```


<a name="redirect-routes"></a>
### 重導向路由

如果您正在定義一個重導向到另一個 URI 的路由，您可以使用 `Route::redirect` 方法。此方法提供了一個方便的捷徑，因此您不必為了執行簡單的重導向而定義完整的路由或控制器：

```php
Route::redirect('/here', '/there');
```

預設情況下，`Route::redirect` 回傳 `302` 狀態碼。您可以使用選填的第三個參數來自定義狀態碼：

```php
Route::redirect('/here', '/there', 301);
```

或者，您可以使用 `Route::permanentRedirect` 方法來回傳 `301` 狀態碼：

```php
Route::permanentRedirect('/here', '/there');
```

> [!WARNING]
> 在重導向路由中使用路由參數時，以下參數由 Laravel 保留，不能使用：`destination` 和 `status`。


<a name="view-routes"></a>
### 視圖路由

如果您的路由只需要回傳一個 [視圖 (View)](/docs/{{version}}/views)，您可以使用 `Route::view` 方法。與 `redirect` 方法一樣，此方法提供了一個簡單的捷徑，因此您不必定義完整的路由或控制器。`view` 方法接受 URI 作為其第一個參數，並接受視圖名稱作為其第二個參數。此外，您可以提供一個資料陣列作為選填的第三個參數傳遞給視圖：

```php
Route::view('/welcome', 'welcome');

Route::view('/welcome', 'welcome', ['name' => 'Taylor']);
```

> [!WARNING]
> 在視圖路由中使用路由參數時，以下參數由 Laravel 保留，不能使用：`view`、`data`、`status` 和 `headers`。


<a name="listing-your-routes"></a>
### 列出您的路由

`route:list` Artisan 指令可以輕鬆地提供應用程式定義的所有路由的概覽：

```shell
php artisan route:list
```

預設情況下，分配給每個路由的路由中介層不會顯示在 `route:list` 的輸出中；但是，您可以透過在指令中加入 `-v` 選項來指示 Laravel 顯示路由中介層和中介層群組名稱：

```shell
php artisan route:list -v

# Expand middleware groups...
php artisan route:list -vv
```

您也可以指示 Laravel 僅顯示以給定 URI 開頭的路由：

```shell
php artisan route:list --path=api
```

此外，您可以透過在執行 `route:list` 指令時提供 `--except-vendor` 選項，指示 Laravel 隱藏由第三方套件定義的任何路由：

```shell
php artisan route:list --except-vendor
```

同樣地，您也可以透過在執行 `route:list` 指令時提供 `--only-vendor` 選項，指示 Laravel 僅顯示由第三方套件定義的路由：

```shell
php artisan route:list --only-vendor
```

<a name="routing-customization"></a>
### 路由自定義

預設情況下，您的應用程式路由是由 `bootstrap/app.php` 檔案配置並載入的：

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

然而，有時您可能想要定義一個全新的檔案來包含應用程式路由的子集。為了達成這個目的，您可以提供一個 `then` 閉包給 `withRouting` 方法。在此閉包中，您可以註冊任何應用程式所需的額外路由：

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

或者，您甚至可以透過提供 `using` 閉包給 `withRouting` 方法，來完全掌控路由註冊。當傳遞此引數時，框架將不會註冊任何 HTTP 路由，而您必須負責手動註冊所有路由：

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
### 必填參數

有時您需要在路由中擷取 URI 的片段。例如，您可能需要從 URL 中擷取使用者的 ID。您可以透過定義路由參數來做到這一點：

```php
Route::get('/user/{id}', function (string $id) {
    return 'User '.$id;
});
```

您可以根據路由的需要定義多個路由參數：

```php
Route::get('/posts/{post}/comments/{comment}', function (string $postId, string $commentId) {
    // ...
});
```

路由參數總是包在 `{}` 大括號內，且應由字母字元組成。在路由參數名稱中也可以使用底線 (`_`)。路由參數會根據其順序注入到路由回呼或控制器中 —— 路由回呼或控制器引數的名稱並不重要。


<a name="parameters-and-dependency-injection"></a>
#### 參數與依賴注入

如果您的路由有希望 Laravel 服務容器自動注入到路由回呼中的依賴項，您應該在依賴項之後列出您的路由參數：

```php
use Illuminate\Http\Request;

Route::get('/user/{id}', function (Request $request, string $id) {
    return 'User '.$id;
});
```


<a name="parameters-optional-parameters"></a>
### 選填參數

有時您可能需要指定一個不一定會出現在 URI 中的路由參數。您可以透過在參數名稱後面加上 `?` 標記來達成。請確保為路由對應的變數提供一個預設值：

```php
Route::get('/user/{name?}', function (?string $name = null) {
    return $name;
});

Route::get('/user/{name?}', function (?string $name = 'John') {
    return $name;
});
```


<a name="parameters-regular-expression-constraints"></a>
### 正規表示式限制

您可以使用路由實例上的 `where` 方法來限制路由參數的格式。`where` 方法接受參數名稱和定義如何限制該參數的正規表示式：

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

為了方便起見，一些常用的正規表示式模式具有輔助方法，讓您可以快速地為路由加入模式限制：

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

如果傳入的請求不符合路由模式限制，將會回傳 404 HTTP 回應。


<a name="parameters-global-constraints"></a>
#### 全域限制

如果您希望路由參數始終受到給定正規表示式的限制，可以使用 `pattern` 方法。您應該在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中定義這些模式：

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

一旦定義了模式，它就會自動套用到所有使用該參數名稱的路由：

```php
Route::get('/user/{id}', function (string $id) {
    // Only executed if {id} is numeric...
});
```


<a name="parameters-encoded-forward-slashes"></a>
#### 編碼後的斜線

Laravel 路由元件允許除了 `/` 之外的所有字元出現在路由參數值中。您必須使用 `where` 條件正規表示式明確地允許 `/` 成為佔位符的一部分：

```php
Route::get('/search/{search}', function (string $search) {
    return $search;
})->where('search', '.*');
```

> [!WARNING]
> 編碼後的斜線僅在最後一個路由區段中受支援。


<a name="named-routes"></a>
## 具名路由

具名路由可以方便地為特定路由產生 URL 或重導向。您可以透過在路由定義後鏈結 `name` 方法來為路由指定名稱：

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
> 路由名稱應該始終是唯一的。


<a name="generating-urls-to-named-routes"></a>
#### 為具名路由產生 URL

一旦您為給定的路由分配了名稱，您就可以在透過 Laravel 的 `route` 和 `redirect` 輔助函式產生 URL 或重導向時使用該路由名稱：

```php
// Generating URLs...
$url = route('profile');

// Generating Redirects...
return redirect()->route('profile');

return to_route('profile');
```

如果具名路由定義了參數，您可以將參數作為第二個引數傳遞給 `route` 函式。給定的參數將自動插入到產生的 URL 的正確位置中：

```php
Route::get('/user/{id}/profile', function (string $id) {
    // ...
})->name('profile');

$url = route('profile', ['id' => 1]);
```

如果您在陣列中傳遞額外的參數，這些鍵值對將自動加入到產生的 URL 的查詢字串中：

```php
Route::get('/user/{id}/profile', function (string $id) {
    // ...
})->name('profile');

$url = route('profile', ['id' => 1, 'photos' => 'yes']);

// http://example.com/user/1/profile?photos=yes
```

> [!NOTE]
> 有時，您可能希望為 URL 參數指定請求範圍的預設值，例如當前的語系。要達成此目的，您可以使用 [URL::defaults 方法](/docs/{{version}}/urls#default-values)。


<a name="inspecting-the-current-route"></a>
#### 檢查當前路由

如果您想確定當前請求是否已路由到給定的具名路由，您可以在 Route 實例上使用 `named` 方法。例如，您可以從路由中介層檢查當前路由名稱：

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

路由群組允許您在大量路由中共享路由屬性（例如中介層），而無需在每個單獨的路由上定義這些屬性。

嵌套群組會嘗試智能地將屬性與其父群組「合併」。中介層和 `where` 條件會被合併，而名稱和前綴則會被附加。在適當的情況下，命名空間分隔符與 URI 前綴中的斜線會被自動添加。


<a name="route-group-middleware"></a>
### 中介層

要將 [中介層](/docs/{{version}}/middleware) 分配給群組內的所有路由，您可以在定義群組之前使用 `middleware` 方法。中介層會按照它們在陣列中列出的順序執行：

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

如果一組路由都使用相同的 [控制器](/docs/{{version}}/controllers)，您可以使用 `controller` 方法來為該群組內的所有路由定義共同的控制器。接著，在定義路由時，您只需要提供它們所呼叫的控制器方法即可：

```php
use App\Http\Controllers\OrderController;

Route::controller(OrderController::class)->group(function () {
    Route::get('/orders/{id}', 'show');
    Route::post('/orders', 'store');
});
```


<a name="route-group-subdomain-routing"></a>
### 子網域路由

路由群組也可以用於處理子網域路由。子網域可以像路由 URI 一樣被分配路由參數，讓您可以擷取子網域的一部分用於路由或控制器。子網域可以透過在定義群組之前呼叫 `domain` 方法來指定：

```php
Route::domain('{account}.example.com')->group(function () {
    Route::get('/user/{id}', function (string $account, string $id) {
        // ...
    });
});
```

> [!WARNING]
> 為了確保您的子網域路由是可存取的，您應該在註冊根網域路由之前註冊子網域路由。這將防止具有相同 URI 路徑的根網域路由覆寫子網域路由。


<a name="route-group-prefixes"></a>
### 路由前綴

`prefix` 方法可用於為群組中的每個路由加上指定的 URI 前綴。例如，您可能希望將群組內所有路由的 URI 都加上 `admin` 前綴：

```php
Route::prefix('admin')->group(function () {
    Route::get('/users', function () {
        // Matches The "/admin/users" URL
    });
});
```


<a name="route-group-name-prefixes"></a>
### 路由名稱前綴

`name` 方法可用於為群組中的每個路由名稱加上指定的字串前綴。例如，您可能希望將群組中所有路由的名稱都加上 `admin`。指定的字串會完全按照原樣加到路由名稱的前面，因此我們要確保在前綴中提供結尾的 `.` 字元：

```php
Route::name('admin.')->group(function () {
    Route::get('/users', function () {
        // Route assigned name "admin.users"...
    })->name('users');
});
```

<a name="route-model-binding"></a>
## 路由模型綁定

當將模型 ID 注入到路由或控制器動作時，您通常會查詢資料庫以檢索與該 ID 對應的模型。Laravel 路由模型綁定提供了一種簡便的方法，可以自動將模型實例直接注入到您的路由中。例如，您可以注入與給定 ID 相匹配的完整 `User` 模型實例，而不是注入使用者的 ID。

<a name="implicit-binding"></a>
### 隱式綁定

Laravel 會自動解析在路由或控制器動作中定義的 Eloquent 模型，只要其型別提示 (Type-hinted) 的變數名稱與路由片段名稱相符。例如：

```php
use App\Models\User;

Route::get('/users/{user}', function (User $user) {
    return $user->email;
});
```

由於 `$user` 變數被型別提示為 `App\Models\User` Eloquent 模型，且變數名稱與 `{user}` URI 片段相符，Laravel 將自動注入 ID 與請求 URI 對應值相匹配的模型實例。如果在資料庫中找不到相匹配的模型實例，將自動產生 404 HTTP 回應。

當然，在使用控制器方法時也可以使用隱式綁定。再次提醒，請注意 `{user}` URI 片段與控制器中包含 `App\Models\User` 型別提示的 `$user` 變數相符：

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

通常，隱式模型綁定不會檢索已被[軟刪除](/docs/{{version}}/eloquent#soft-deleting)的模型。但是，您可以透過在路由定義鏈結 `withTrashed` 方法，來指示隱式綁定檢索這些模型：

```php
use App\Models\User;

Route::get('/users/{user}', function (User $user) {
    return $user->email;
})->withTrashed();
```

<a name="customizing-the-default-key-name"></a>
#### 自定義鍵值

有時您可能希望使用 `id` 以外的欄位來解析 Eloquent 模型。為此，您可以在路由參數定義中指定該欄位：

```php
use App\Models\Post;

Route::get('/posts/{post:slug}', function (Post $post) {
    return $post;
});
```

如果您希望模型綁定在檢索給定的模型類別時，始終使用 `id` 以外的資料庫欄位，您可以覆寫 Eloquent 模型上的 `getRouteKeyName` 方法：

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
#### 自定義鍵值與作用域

當在單個路由定義中隱式綁定多個 Eloquent 模型時，您可能希望限定第二個 Eloquent 模型的作用域，使其必須是前一個 Eloquent 模型的子模型。例如，考慮這個為特定使用者透過 slug 檢索部落格文章的路由定義：

```php
use App\Models\Post;
use App\Models\User;

Route::get('/users/{user}/posts/{post:slug}', function (User $user, Post $post) {
    return $post;
});
```

當使用自定義鍵值的隱式綁定作為巢狀路由參數時，Laravel 將自動限定查詢範圍，以便透過其父模型檢索巢狀模型，並使用慣例來猜測父模型上的關聯名稱。在這種情況下，將假設 `User` 模型有一個名為 `posts`（路由參數名稱的複數形式）的關聯，可用於檢索 `Post` 模型。

如果您願意，即使未提供自定義鍵值，也可以指示 Laravel 限定「子」綁定的作用域。為此，您可以在定義路由時呼叫 `scopeBindings` 方法：

```php
use App\Models\Post;
use App\Models\User;

Route::get('/users/{user}/posts/{post}', function (User $user, Post $post) {
    return $post;
})->scopeBindings();
```

或者，您可以指示整個路由定義群組使用限定作用域的綁定：

```php
Route::scopeBindings()->group(function () {
    Route::get('/users/{user}/posts/{post}', function (User $user, Post $post) {
        return $post;
    });
});
```

同樣地，您可以透過呼叫 `withoutScopedBindings` 方法，明確指示 Laravel 不要限定綁定的作用域：

```php
Route::get('/users/{user}/posts/{post:slug}', function (User $user, Post $post) {
    return $post;
})->withoutScopedBindings();
```

<a name="customizing-missing-model-behavior"></a>
#### 自定義缺失模型行為

通常，如果找不到隱式綁定的模型，將產生 404 HTTP 回應。但是，您可以在定義路由時呼叫 `missing` 方法來自定義此行為。`missing` 方法接受一個閉包，如果找不到隱式綁定的模型，該閉包將被叫用：

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

PHP 8.1 引入了對 [Enum](https://www.php.net/manual/en/language.enumerations.backed.php) 的支援。為了輔助此功能，Laravel 允許您在路由定義中對 [string-backed Enum](https://www.php.net/manual/en/language.enumerations.backed.php) 進行型別提示，且僅當該路由片段對應於有效的 Enum 值時，Laravel 才會叫用該路由。否則，將自動回傳 404 HTTP 回應。例如，給定以下 Enum：

```php
<?php

namespace App\Enums;

enum Category: string
{
    case Fruits = 'fruits';
    case People = 'people';
}
```

您可以定義一個僅當 `{category}` 路由片段為 `fruits` 或 `people` 時才會被叫用的路由。否則，Laravel 將回傳 404 HTTP 回應：

```php
use App\Enums\Category;
use Illuminate\Support\Facades\Route;

Route::get('/categories/{category}', function (Category $category) {
    return $category->value;
});
```

<a name="explicit-binding"></a>
### 顯式綁定

您不需要非得使用 Laravel 隱式且基於慣例的模型解析才能使用模型綁定。您也可以顯式定義路由參數如何對應到模型。要註冊顯式綁定，請使用路由器 (router) 的 `model` 方法來指定特定參數的類別。您應該在 `AppServiceProvider` 類別的 `boot` 方法開頭定義您的顯式模型綁定：

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

接著，定義一個包含 `{user}` 參數的路由：

```php
use App\Models\User;

Route::get('/users/{user}', function (User $user) {
    // ...
});
```

由於我們已將所有 `{user}` 參數綁定到 `App\Models\User` 模型，因此該類別的一個實例將被注入到路由中。因此，例如對 `users/1` 的請求將注入資料庫中 ID 為 `1` 的 `User` 實例。

如果在資料庫中找不到相符的模型實例，將自動產生 404 HTTP 回應。


<a name="customizing-the-resolution-logic"></a>
#### 自定義解析邏輯

如果您希望定義自己的模型綁定解析邏輯，可以使用 `Route::bind` 方法。您傳遞給 `bind` 方法的閉包將接收 URI 片段的值，並應回傳應注入到路由中的類別實例。同樣地，此自定義應放在應用程式 `AppServiceProvider` 的 `boot` 方法中：

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

或者，您可以在 Eloquent 模型上覆寫 `resolveRouteBinding` 方法。此方法將接收 URI 片段的值，並應回傳應注入路由的類別實例：

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

如果路由正在使用 [隱式綁定範圍限制 (scoping)](#implicit-model-binding-scoping)，則將使用 `resolveChildRouteBinding` 方法來解析父模型的子綁定：

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
## 備用路由

使用 `Route::fallback` 方法，您可以定義一個當沒有其他路由比對成功時所執行的路由。通常，未處理的請求會透過應用程式的例外處理器自動渲染一個「404」頁面。然而，由於您通常會在 `routes/web.php` 檔案中定義 `fallback` 路由，因此 `web` 中介層群組中的所有中介層都將套用於該路由。您可以根據需要自由地為此路由新增額外的中介層：

```php
Route::fallback(function () {
    // ...
});
```


<a name="rate-limiting"></a>
## 速率限制


<a name="defining-rate-limiters"></a>
### 定義速率限制器

Laravel 包含強大且可自定義的速率限制服務，您可以用來限制特定路由或路由群組的流量。首先，您應該定義符合應用程式需求的速率限制器配置。

速率限制器可以定義在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中：

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
    });
}
```

速率限制器是使用 `RateLimiter` Facade 的 `for` 方法定義的。`for` 方法接受一個速率限制器名稱和一個閉包，該閉包回傳應套用於分配給該速率限制器之路由的限制配置。限制配置是 `Illuminate\Cache\RateLimiting\Limit` 類別的實例。這個類別包含實用的「建構器」方法，讓您可以快速定義限制。速率限制器名稱可以是您想要的任何字串：

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    RateLimiter::for('global', function (Request $request) {
        return Limit::perMinute(1000);
    });
}
```

如果傳入的請求超過了指定的速率限制，Laravel 會自動回傳一個 429 HTTP 狀態碼的回應。如果您想定義自己的速率限制回應，可以使用 `response` 方法：

```php
RateLimiter::for('global', function (Request $request) {
    return Limit::perMinute(1000)->response(function (Request $request, array $headers) {
        return response('Custom response...', 429, $headers);
    });
});
```

由於速率限制器閉包會接收傳入的 HTTP 請求實例，您可以根據傳入的請求或通過身份驗證的使用者動態地建構適當的速率限制：

```php
RateLimiter::for('uploads', function (Request $request) {
    return $request->user()->vipCustomer()
        ? Limit::none()
        : Limit::perHour(10);
});
```


<a name="segmenting-rate-limits"></a>
#### 分段速率限制

有時您可能希望根據某些任意值對速率限制進行分段。例如，您可能希望允許使用者每分鐘每個 IP 位址存取給定路由 100 次。要達成此目的，您可以在建構速率限制時使用 `by` 方法：

```php
RateLimiter::for('uploads', function (Request $request) {
    return $request->user()->vipCustomer()
        ? Limit::none()
        : Limit::perMinute(100)->by($request->ip());
});
```

為了用另一個例子說明此功能，我們可以限制通過身份驗證的使用者 ID 每分鐘存取路由 100 次，或者對訪客限制每個 IP 位址每分鐘存取 10 次：

```php
RateLimiter::for('uploads', function (Request $request) {
    return $request->user()
        ? Limit::perMinute(100)->by($request->user()->id)
        : Limit::perMinute(10)->by($request->ip());
});
```


<a name="multiple-rate-limits"></a>
#### 多個速率限制

如果需要，您可以為指定的速率限制器配置回傳一個速率限制陣列。每個速率限制將根據它們在陣列中的順序對路由進行評估：

```php
RateLimiter::for('login', function (Request $request) {
    return [
        Limit::perMinute(500),
        Limit::perMinute(3)->by($request->input('email')),
    ];
});
```

如果您要分配由相同 `by` 值分段的多個速率限制，應確保每個 `by` 值都是唯一的。達成此目的最簡單方法是為傳遞給 `by` 方法的值加上前綴：

```php
RateLimiter::for('uploads', function (Request $request) {
    return [
        Limit::perMinute(10)->by('minute:'.$request->user()->id),
        Limit::perDay(1000)->by('day:'.$request->user()->id),
    ];
});
```


<a name="response-base-rate-limiting"></a>
#### 基於回應的速率限制

除了對傳入請求進行速率限制外，Laravel 還允許您使用 `after` 方法根據回應進行速率限制。這在您只想將某些回應（例如驗證錯誤、404 回應或其他特定 HTTP 狀態碼）計入速率限制時非常有用。

`after` 方法接受一個接收回應的閉包，如果該回應應計入速率限制，則應回傳 `true`；如果應忽略，則回傳 `false`。這對於防止透過限制連續 404 回應來進行列舉攻擊特別有用，或者允許使用者重試未通過驗證的請求，而不會在應該只限制成功操作的端點上耗盡其速率限制：

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
### 將速率限制器附加到路由

可以使用 `throttle` [中介層](/docs/{{version}}/middleware) 將速率限制器附加到路由或路由群組。throttle 中介層接受您希望分配給該路由的速率限制器名稱：

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

預設情況下，`throttle` 中介層被對應到 `Illuminate\Routing\Middleware\ThrottleRequests` 類別。但是，如果您使用 Redis 作為應用程式的快取驅動器，您可能希望指示 Laravel 使用 Redis 來管理速率限制。要達成此目的，您應該在應用程式的 `bootstrap/app.php` 檔案中使用 `throttleWithRedis` 方法。此方法將 `throttle` 中介層對應到 `Illuminate\Routing\Middleware\ThrottleRequestsWithRedis` 中介層類別：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->throttleWithRedis();
    // ...
})
```


<a name="form-method-spoofing"></a>
## 表單方法欺騙

HTML 表單不支援 `PUT`、`PATCH` 或 `DELETE` 動作。因此，當定義從 HTML 表單呼叫的 `PUT`、`PATCH` 或 `DELETE` 路由時，您需要在表單中加入一個隱藏的 `_method` 欄位。與 `_method` 欄位一起傳送的值將被用作 HTTP 請求方法：

```blade
<form action="/example" method="POST">
    <input type="hidden" name="_method" value="PUT">
    <input type="hidden" name="_token" value="{{ csrf_token() }}">
</form>
```

為了方便起見，您可以使用 `@method` [Blade 指令](/docs/{{version}}/blade) 來產生 `_method` 輸入欄位：

```blade
<form action="/example" method="POST">
    @method('PUT')
    @csrf
</form>
```

<a name="accessing-the-current-route"></a>
## 存取當前路由

您可以使用 `Route` Facade 上的 `current`、`currentRouteName` 與 `currentRouteAction` 方法來存取處理傳入請求的路由相關資訊：

```php
use Illuminate\Support\Facades\Route;

$route = Route::current(); // Illuminate\Routing\Route
$name = Route::currentRouteName(); // string
$action = Route::currentRouteAction(); // string
```

您可以參考 [Route Facade 的底層類別](https://api.laravel.com/docs/{{version}}/Illuminate/Routing/Router.html) 與 [Route 執行個體](https://api.laravel.com/docs/{{version}}/Illuminate/Routing/Route.html) 的 API 文件，以查看 router 與 route 類別中所有可用的方法。


<a name="cors"></a>
## 跨來源資源共享 (CORS)

Laravel 可以根據您設定的數值自動回應 CORS `OPTIONS` HTTP 請求。`OPTIONS` 請求將由 `HandleCors` [中介層](/docs/{{version}}/middleware) 自動處理，該中介層已自動包含在應用程式的全域中介層堆疊中。

有時，您可能需要為您的應用程式自定義 CORS 設定值。您可以使用 `config:publish` Artisan 指令來發布 `cors` 設定檔：

```shell
php artisan config:publish cors
```

此指令將在應用程式的 `config` 目錄中放置一個 `cors.php` 設定檔。

> [!NOTE]
> 有關 CORS 與 CORS 標頭的更多資訊，請參閱 [MDN 關於 CORS 的網路文件](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#The_HTTP_response_headers)。


<a name="route-caching"></a>
## 路由快取

將應用程式部署到正式環境時，您應該利用 Laravel 的路由快取。使用路由快取將大幅減少註冊所有應用程式路由所需的時間。要產生路由快取，請執行 `route:cache` Artisan 指令：

```shell
php artisan route:cache
```

執行此指令後，快取的路由檔案將在每次請求時載入。請記住，如果您新增了任何新路由，則需要產生新的路由快取。因此，您應該只在專案部署期間執行 `route:cache` 指令。

您可以使用 `route:clear` 指令來清除路由快取：

```shell
php artisan route:clear
```