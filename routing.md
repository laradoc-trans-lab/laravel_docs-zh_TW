# 路由

- [基本路由](#basic-routing)
    - [預設路由檔案](#the-default-route-files)
    - [轉址路由](#redirect-routes)
    - [視圖路由](#view-routes)
    - [列出您的路由](#listing-your-routes)
    - [自訂路由](#routing-customization)
- [路由參數](#route-parameters)
    - [必填參數](#required-parameters)
    - [選擇性參數](#parameters-optional-parameters)
    - [正則表達式條件限制](#parameters-regular-expression-constraints)
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
- [表單 HTTP 方法偽裝](#form-method-spoofing)
- [取得當前路由](#accessing-the-current-route)
- [跨來源資源共享 (CORS)](#cors)
- [路由快取](#route-caching)

<a name="basic-routing"></a>
## 基本路由

最基本的 Laravel 路由接受一個 URI 和一個閉包 (Closure)，提供了一種非常簡單且語意明確的方式來定義路由與行為，無需複雜的路由設定檔：

```php
use Illuminate\Support\Facades\Route;

Route::get('/greeting', function () {
    return 'Hello World';
});
```


<a name="the-default-route-files"></a>
### 預設路由檔案

所有的 Laravel 路由都定義在 `routes` 目錄下的路由檔案中。Laravel 會根據您應用程式的 `bootstrap/app.php` 檔案中所指定的設定，自動載入這些檔案。`routes/web.php` 檔案定義了用於網頁介面的路由。這些路由被分配給 `web` [中介層群組](/docs/{{version}}/middleware#laravels-default-middleware-groups)，該群組提供了 Session 狀態和 CSRF 保護等功能。

對於大多數應用程式，您將從在 `routes/web.php` 檔案中定義路由開始。定義在 `routes/web.php` 中的路由可以透過在瀏覽器中輸入對應的路由 URL 來存取。例如，您可以透過在瀏覽器中瀏覽 `http://example.com/user` 來存取以下路由：

```php
use App\Http\Controllers\UserController;

Route::get('/user', [UserController::class, 'index']);
```


<a name="api-routes"></a>
#### API 路由

如果您的應用程式也將提供無狀態 (stateless) 的 API，您可以使用 `install:api` Artisan 命令來啟用 API 路由：

```shell
php artisan install:api
```

`install:api` 命令會安裝 [Laravel Sanctum](/docs/{{version}}/sanctum)，它提供了一個強大且簡單的 API token 認證防護，可用於驗證第三方 API 消費者、單頁應用程式 (SPA) 或行動裝置應用程式。此外，`install:api` 命令還會建立 `routes/api.php` 檔案：

```php
Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```

當然，對於應該可以公開存取的路由，您可以自由地省略 `auth:sanctum` 中介層。

在 `routes/api.php` 中的路由是無狀態的，並被分配給 `api` [中介層群組](/docs/{{version}}/middleware#laravels-default-middleware-groups)。此外，`/api` URI 前綴會自動套用到這些路由上，因此您不需要手動將其套用到檔案中的每個路由。您可以透過修改應用程式的 `bootstrap/app.php` 檔案來變更此前綴：

```php
->withRouting(
    api: __DIR__.'/../routes/api.php',
    apiPrefix: 'api/admin',
    // ...
)
```


<a name="available-router-methods"></a>
#### 可用的路由器方法

路由器允許您註冊可回應任何 HTTP 動作 (Verb) 的路由：

```php
Route::get($uri, $callback);
Route::post($uri, $callback);
Route::put($uri, $callback);
Route::patch($uri, $callback);
Route::delete($uri, $callback);
Route::options($uri, $callback);
```

有時候您可能需要註冊一個能回應多種 HTTP 動作的路由。您可以透過 `match` 方法來做到這一點。甚至，您可以使用 `any` 方法註冊一個能回應所有 HTTP 動作的路由：

```php
Route::match(['get', 'post'], '/', function () {
    // ...
});

Route::any('/', function () {
    // ...
});
```

> [!NOTE]
> 當定義多個共享相同 URI 的路由時，使用 `get`、`post`、`put`、`patch`、`delete` 及 `options` 方法的路由應定義在 `any`、`match` 與 `redirect` 方法的路由之前。這可以確保傳入的請求能比對到正確的路由。


<a name="dependency-injection"></a>
#### 依賴注入

您可以在路由的閉包回呼函式簽名中型別提示 (Type-hint) 路由所需的任何依賴項。宣告的依賴項將由 Laravel [服務容器](/docs/{{version}}/container)自動解析並注入到回呼函式中。例如，您可以型別提示 `Illuminate\Http\Request` 類別，讓當前的 HTTP 請求自動注入到您的路由閉包中：

```php
use Illuminate\Http\Request;

Route::get('/users', function (Request $request) {
    // ...
});
```


<a name="csrf-protection"></a>
#### CSRF 保護

請記住，任何指向在 `web` 路由檔案中定義的 `POST`、`PUT`、`PATCH` 或 `DELETE` 路由的 HTML 表單，都應該包含一個 CSRF Token 欄位。否則，該請求將被拒絕。您可以在 [CSRF 文件](/docs/{{version}}/csrf)中閱讀更多關於 CSRF 保護的資訊：

```blade
<form method="POST" action="/profile">
    @csrf
    ...
</form>
```


<a name="redirect-routes"></a>
### 轉址路由

如果您要定義一個會轉址到另一個 URI 的路由，可以使用 `Route::redirect` 方法。這個方法提供了一個方便的捷徑，讓您不必為了執行簡單的轉址而定義完整的路由或控制器：

```php
Route::redirect('/here', '/there');
```

預設情況下，`Route::redirect` 會回傳 `302` 狀態碼。您可以透過選擇性的第三個參數自訂狀態碼：

```php
Route::redirect('/here', '/there', 301);
```

或者，您可以透過 `Route::permanentRedirect` 方法來回傳 `301` 狀態碼：

```php
Route::permanentRedirect('/here', '/there');
```

> [!WARNING]
> 在轉址路由中使用路由參數時，以下參數名稱為 Laravel 的保留字且無法使用：`destination` 和 `status`。


<a name="view-routes"></a>
### 視圖路由

如果您的路由只需要回傳一個[視圖](/docs/{{version}}/views)，可以使用 `Route::view` 方法。就像 `redirect` 方法一樣，這個方法提供了一個簡單的捷徑，讓您不必定義完整的路由或控制器。`view` 方法的第一個引數接受 URI，第二個引數接受視圖名稱。此外，您還可以提供一個陣列作為選擇性的第三個引數，將資料傳遞給視圖：

```php
Route::view('/welcome', 'welcome');

Route::view('/welcome', 'welcome', ['name' => 'Taylor']);
```

> [!WARNING]
> 在視圖路由中使用路由參數時，以下參數名稱為 Laravel 的保留字且無法使用：`view`、`data`、`status` 和 `headers`。


<a name="listing-your-routes"></a>
### 列出您的路由

`route:list` Artisan 命令可以輕鬆地提供應用程式所定義的所有路由總覽：

```shell
php artisan route:list
```

預設情況下，指派給每個路由的路由中介層不會顯示在 `route:list` 的輸出中；不過，您可以透過在命令中加上 `-v` 選項，指示 Laravel 顯示路由中介層與中介層群組名稱：

```shell
php artisan route:list -v

# Expand middleware groups...
php artisan route:list -vv
```

您也可以指示 Laravel 僅顯示以指定 URI 開頭的路由：

```shell
php artisan route:list --path=api
```

此外，您可以透過在執行 `route:list` 命令時提供 `--except-vendor` 選項，指示 Laravel 隱藏第三方套件定義的任何路由：

```shell
php artisan route:list --except-vendor
```

同樣地，您也可以透過在執行 `route:list` 命令時提供 `--only-vendor` 選項，指示 Laravel 僅顯示第三方套件所定義的路由：

```shell
php artisan route:list --only-vendor
```

<a name="routing-customization"></a>
### 自訂路由

預設情況下，您應用程式的路由是由 `bootstrap/app.php` 檔案進行設定與載入：

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

然而，有時您可能希望定義一個全新的檔案來包含您應用程式的一部分路由。若要做到這一點，您可以傳遞一個 `then` 閉包給 `withRouting` 方法。在此閉包中，您可以註冊應用程式所需的任何額外路由：

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

或者，您甚至可以透過提供一個 `using` 閉包給 `withRouting` 方法來完全掌控路由的註冊。傳遞此引數時，框架將不會註冊任何 HTTP 路由，且您必須負責手動註冊所有路由：

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

有時候您會需要擷取路由中 URI 的區段。例如，您可能需要從 URL 中擷取使用者的 ID。您可以透過定義路由參數來達成：

```php
Route::get('/user/{id}', function (string $id) {
    return 'User '.$id;
});
```

您可以根據路由需求定義任意數量的路由參數：

```php
Route::get('/posts/{post}/comments/{comment}', function (string $postId, string $commentId) {
    // ...
});
```

路由參數總是包圍在 `{}` 大括號之內，且應該由英文字母組成。路由參數名稱中也可以使用底線 (`_`)。路由參數會根據它們的順序被注入到路由回呼 (Callback) 或控制器中——路由回呼或控制器引數的名稱並不重要。

<a name="parameters-and-dependency-injection"></a>
#### 參數與依賴注入

如果您的路由有希望 Laravel 服務容器自動注入到路由回呼中的依賴項目，您應該將路由參數列在依賴項目之後：

```php
use Illuminate\Http\Request;

Route::get('/user/{id}', function (Request $request, string $id) {
    return 'User '.$id;
});
```

<a name="parameters-optional-parameters"></a>
### 選擇性參數

有時候您可能需要指定一個不一定會出現在 URI 中的路由參數。您可以透過在參數名稱後面加上 `?` 記號來做到這一點。請確保給予路由對應的變數一個預設值：

```php
Route::get('/user/{name?}', function (?string $name = null) {
    return $name;
});

Route::get('/user/{name?}', function (?string $name = 'John') {
    return $name;
});
```

<a name="parameters-regular-expression-constraints"></a>
### 正則表達式條件限制

您可以使用路由實例上的 `where` 方法來限制路由參數的格式。`where` 方法接受參數名稱以及定義該參數應如何受限的正則表達式：

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

為了方便起見，一些常用的正則表達式模式擁有輔助方法，讓您可以快速地為路由新增模式條件限制：

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

如果傳入的請求不符合路由模式條件限制，將會回傳 404 HTTP 回應。

<a name="parameters-global-constraints"></a>
#### 全域條件限制

如果您希望路由參數總是受到給定正則表達式的限制，您可以使用 `pattern` 方法。您應該在應用程式的 `App\Providers\AppServiceProvider` 類別中的 `boot` 方法裡定義這些模式：

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

模式定義好之後，它會自動套用到所有使用該參數名稱的路由：

```php
Route::get('/user/{id}', function (string $id) {
    // Only executed if {id} is numeric...
});
```

<a name="parameters-encoded-forward-slashes"></a>
#### 編碼後的斜線 (Forward Slashes)

Laravel 路由元件允許除了 `/` 之外的所有字元出現在路由參數值中。您必須使用 `where` 條件正則表達式明確允許 `/` 成為預留位置的一部分：

```php
Route::get('/search/{search}', function (string $search) {
    return $search;
})->where('search', '.*');
```

> [!WARNING]
> 編碼後的斜線僅在最後一個路由區段中受到支援。

<a name="named-routes"></a>
## 具名路由

具名路由方便為特定路由生成 URL 或轉址。您可以在路由定義後鏈結 `name` 方法來為路由指定名稱：

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
> 路由名稱應始終保持唯一。

<a name="generating-urls-to-named-routes"></a>
#### 生成具名路由的 URL

一旦您為給定的路由指派了名稱，您可以在透過 Laravel 的 `route` 和 `redirect` 輔助函數生成 URL 或轉址時使用該路由名稱：

```php
// Generating URLs...
$url = route('profile');

// Generating Redirects...
return redirect()->route('profile');

return to_route('profile');
```

如果具名路由有定義參數，您可以將參數作為第二個引數傳遞給 `route` 函數。給定的參數將會自動插入到生成 URL 的正確位置：

```php
Route::get('/user/{id}/profile', function (string $id) {
    // ...
})->name('profile');

$url = route('profile', ['id' => 1]);
```

如果您在陣列中傳遞額外的參數，這些鍵 / 值對將會自動新增到生成的 URL 查詢字串中：

```php
Route::get('/user/{id}/profile', function (string $id) {
    // ...
})->name('profile');

$url = route('profile', ['id' => 1, 'photos' => 'yes']);

// http://example.com/user/1/profile?photos=yes
```

> [!NOTE]
> 有時候，您可能希望為 URL 參數指定適用於整個請求的預設值，例如目前的語系。若要做到這一點，您可以使用 [URL::defaults 方法](/docs/{{version}}/urls#default-values)。

<a name="inspecting-the-current-route"></a>
#### 檢視當前路由

如果您想確定當前請求是否被路由到給定的具名路由，您可以使用 Route 實例上的 `named` 方法。例如，您可以從路由中介層中檢查當前的路由名稱：

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

路由群組允許您在大量路由之間共享路由屬性（例如中介層），而無需在每個單獨的路由上定義這些屬性。

巢狀群組會嘗試智慧地與其父群組「合併」屬性。中介層與 `where` 條件會被合併，而名稱和前綴則會被附加在後面。命名空間分隔符號與 URI 前綴中的斜線會在適當的地方自動新增。


<a name="route-group-middleware"></a>
### 中介層

若要將[中介層](/docs/{{version}}/middleware)指派給群組內的所有路由，可以在定義群組前使用 `middleware` 方法。中介層會依照它們在陣列中列出的順序執行：

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

如果一組路由都使用相同的[控制器](/docs/{{version}}/controllers)，您可以使用 `controller` 方法為該群組內的所有路由定義共同的控制器。接著在定義路由時，您只需要提供它們所呼叫的控制器方法：

```php
use App\Http\Controllers\OrderController;

Route::controller(OrderController::class)->group(function () {
    Route::get('/orders/{id}', 'show');
    Route::post('/orders', 'store');
});
```


<a name="route-group-subdomain-routing"></a>
### 子網域路由

路由群組也可以用來處理子網域路由。子網域可以像路由 URI 一樣被指派路由參數，讓您可以擷取子網域的一部分以在路由或控制器中使用。可以在定義群組前呼叫 `domain` 方法來指定子網域：

```php
Route::domain('{account}.example.com')->group(function () {
    Route::get('/user/{id}', function (string $account, string $id) {
        // ...
    });
});
```


<a name="route-group-prefixes"></a>
### 路由前綴

`prefix` 方法可用於為群組中的每個路由加上指定的 URI 前綴。例如，您可能希望將群組內的所有路由 URI 都加上 `admin` 前綴：

```php
Route::prefix('admin')->group(function () {
    Route::get('/users', function () {
        // Matches The "/admin/users" URL
    });
});
```


<a name="route-group-name-prefixes"></a>
### 路由名稱前綴

`name` 方法可用於為群組中的每個路由名稱加上指定的字串前綴。例如，您可能希望將群組中所有路由的名稱都加上 `admin` 前綴。給定的字串會完全按照指定方式作為路由名稱的前綴，因此我們需要確保在字串結尾提供 `.` 字元：

```php
Route::name('admin.')->group(function () {
    Route::get('/users', function () {
        // Route assigned name "admin.users"...
    })->name('users');
});
```

<a name="route-model-binding"></a>
## 路由模型綁定

當將模型 ID 注入到路由或控制器動作時，您通常會查詢資料庫以取得該 ID 對應的模型。Laravel 的路由模型綁定提供了一種便利的方法，可以直接將模型實例自動注入到您的路由中。例如，您可以注入與給定 ID 相符的整個 `User` 模型實例，而不是僅注入使用者的 ID。


<a name="implicit-binding"></a>
### 隱式綁定

Laravel 會自動解析定義在路由或控制器動作中的 Eloquent 模型，只要其型態提示（Type-hinted）的變數名稱與路由片段名稱相符即可。例如：

```php
use App\Models\User;

Route::get('/users/{user}', function (User $user) {
    return $user->email;
});
```

由於 `$user` 變數被型態提示為 `App\Models\User` Eloquent 模型，且變數名稱與 `{user}` URI 片段相符，因此 Laravel 會自動注入 ID 與請求 URI 中對應值相符的模型實例。如果在資料庫中找不到相符的模型實例，系統將會自動產生 404 HTTP 回應。

當然，使用控制器方法時也可以進行隱式綁定。同樣地，請注意 `{user}` URI 片段與控制器中帶有 `App\Models\User` 型態提示的 `$user` 變數相符：

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
#### 軟刪除的模型

通常，隱式模型綁定不會取得已被[軟刪除](/docs/{{version}}/eloquent#soft-deleting)的模型。然而，您可以透過在路由定義鏈結 `withTrashed` 方法，來指示隱式綁定取得這些模型：

```php
use App\Models\User;

Route::get('/users/{user}', function (User $user) {
    return $user->email;
})->withTrashed();
```


<a name="customizing-the-default-key-name"></a>
#### 自訂鍵名

有時候您可能希望使用 `id` 以外的欄位來解析 Eloquent 模型。為此，您可以在路由參數定義中指定該欄位：

```php
use App\Models\Post;

Route::get('/posts/{post:slug}', function (Post $post) {
    return $post;
});
```

如果您希望在取得給定的模型類別時，模型綁定總是使用 `id` 以外的資料庫欄位，您可以在 Eloquent 模型上套用 `RouteKey` 屬性（Attribute）：

```php
use Illuminate\Database\Eloquent\Attributes\RouteKey;
use Illuminate\Database\Eloquent\Model;

#[RouteKey('slug')]
class Post extends Model
{
    // ...
}
```


<a name="implicit-model-binding-scoping"></a>
#### 自訂鍵與作用域

當在單一路由定義中隱式綁定多個 Eloquent 模型時，您可能希望對第二個 Eloquent 模型進行作用域限制，使其必須是前一個 Eloquent 模型的子模型。例如，考慮以下這段透過 slug 替特定使用者取得網誌文章的路由定義：

```php
use App\Models\Post;
use App\Models\User;

Route::get('/users/{user}/posts/{post:slug}', function (User $user, Post $post) {
    return $post;
});
```

當使用帶有自訂鍵的隱式綁定作為巢狀路由參數時，Laravel 將根據慣例推測父模型上的關聯名稱，自動將查詢範圍限制為透過其父模型來取得該巢狀模型。在這種情況下，系統會假設 `User` 模型有一個名為 `posts`（路由參數名稱的複數形式）的關聯，用來取得 `Post` 模型。

如果您願意，即使沒有提供自訂鍵，您也可以指示 Laravel 限制「子」綁定的作用域。為此，您可以在定義路由時呼叫 `scopeBindings` 方法：

```php
use App\Models\Post;
use App\Models\User;

Route::get('/users/{user}/posts/{post}', function (User $user, Post $post) {
    return $post;
})->scopeBindings();
```

或者，您可以指示一整個路由定義群組都使用作用域綁定：

```php
Route::scopeBindings()->group(function () {
    Route::get('/users/{user}/posts/{post}', function (User $user, Post $post) {
        return $post;
    });
});
```

同樣地，您可以呼叫 `withoutScopedBindings` 方法，明確地指示 Laravel 不要限制綁定的作用域：

```php
Route::get('/users/{user}/posts/{post:slug}', function (User $user, Post $post) {
    return $post;
})->withoutScopedBindings();
```


<a name="customizing-missing-model-behavior"></a>
#### 自訂找不到模型時的行為

通常，如果找不到隱式綁定的模型，系統會自動產生 404 HTTP 回應。不過，您可以在定義路由時呼叫 `missing` 方法來自訂此行為。`missing` 方法接受一個閉包（Closure），當找不到隱式綁定的模型時，該閉包將會被呼叫：

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

PHP 8.1 引入了對 [Enum](https://www.php.net/manual/en/language.enumerations.backed.php) 的支援。為了配合此功能，Laravel 允許您在路由定義中為 [以字串為底層值的 Enum (String-backed Enum)](https://www.php.net/manual/en/language.enumerations.backed.php) 設定型態提示，且只有當該路由片段對應到有效的 Enum 值時，Laravel 才會執行該路由。否則，系統將會自動回傳 404 HTTP 回應。例如，給定以下 Enum：

```php
<?php

namespace App\Enums;

enum Category: string
{
    case Fruits = 'fruits';
    case People = 'people';
}
```

您可以定義一個只有當 `{category}` 路由片段為 `fruits` 或 `people` 時才會執行的路由。否則，Laravel 將會回傳 404 HTTP 回應：

```php
use App\Enums\Category;
use Illuminate\Support\Facades\Route;

Route::get('/categories/{category}', function (Category $category) {
    return $category->value;
});
```

<a name="explicit-binding"></a>
### 顯式綁定

您並非一定要使用 Laravel 的隱式、基於慣例的模型解析才能進行模型綁定。您也可以顯式地定義路由參數如何對應到模型。若要註冊顯式綁定，請使用路由器的 `model` 方法來指定特定參數所對應的類別。您應該在 `AppServiceProvider` 類別的 `boot` 方法開頭定義您的顯式模型綁定：

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

由於我們已經將所有 `{user}` 參數綁定到 `App\Models\User` 模型，因此該類別的實例將會被注入到路由中。例如，發送到 `users/1` 的請求將會注入資料庫中 ID 為 `1` 的 `User` 實例。

如果在資料庫中找不到匹配的模型實例，系統將會自動產生 404 HTTP 回應。


<a name="customizing-the-resolution-logic"></a>
#### 自訂解析邏輯

如果您希望定義自己的模型綁定解析邏輯，您可以使用 `Route::bind` 方法。傳遞給 `bind` 方法的閉包將會接收 URI 片段的值，並應傳回要注入到路由中的類別實例。同樣地，這項自訂應放置在應用程式 `AppServiceProvider` 的 `boot` 方法中：

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

或者，您也可以在 Eloquent 模型上覆寫 `resolveRouteBinding` 方法。此方法將接收 URI 片段的值，並應傳回要注入到路由中的類別實例：

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

如果路由使用了[隱式綁定作用域](#implicit-model-binding-scoping)，則會使用 `resolveChildRouteBinding` 方法來解析父模型的子綁定：

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

使用 `Route::fallback` 方法，您可以定義一個當沒有其他路由符合傳入請求時所要執行的路由。通常，未處理的請求會透過應用程式的例外處理器自動轉譯「404」頁面。然而，由於您通常會在 `routes/web.php` 檔案中定義 `fallback` 路由，因此 `web` 中介層群組中的所有中介層都會套用到該路由。您可以根據需要自由地為此路由新增其他中介層：

```php
Route::fallback(function () {
    // ...
});
```

<a name="rate-limiting"></a>
## 速率限制

<a name="defining-rate-limiters"></a>
### 定義速率限制器

Laravel 包含了強大且可自訂的速率限制服務，您可以使用它們來限制特定路由或路由群組的流量大小。首先，您應該定義符合應用程式需求的速率限制器設定。

速率限制器可以在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中定義：

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

速率限制器是使用 `RateLimiter` Facade 的 `for` 方法來定義的。`for` 方法接收速率限制器名稱以及一個閉包，該閉包應回傳套用到被指派至該速率限制器之路由的限制設定。限制設定是 `Illuminate\Cache\RateLimiting\Limit` 類別的實例。該類別包含了便利的「建構器」方法，讓您可以快速定義限制。速率限制器名稱可以是您想要的任何字串：

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

若傳入的請求超過了指定的速率限制，Laravel 會自動回傳 HTTP 狀態碼 429 的回應。如果您想自訂觸發速率限制時回傳的回應，可以使用 `response` 方法：

```php
RateLimiter::for('global', function (Request $request) {
    return Limit::perMinute(1000)->response(function (Request $request, array $headers) {
        return response('Custom response...', 429, $headers);
    });
});
```

由於速率限制器的回呼會接收傳入的 HTTP 請求實例，因此您可以根據傳入的請求或已通過認證的使用者動態建構適當的速率限制：

```php
RateLimiter::for('uploads', function (Request $request) {
    return $request->user()?->vipCustomer()
        ? Limit::none()
        : Limit::perHour(10);
});
```

<a name="segmenting-rate-limits"></a>
#### 分段速率限制

有時您可能希望依據某些任意值來分段速率限制。例如，您可能希望允許使用者每個 IP 地址每分鐘存取給定路由 100 次。若要達成此目的，可以在建構速率限制時使用 `by` 方法：

```php
RateLimiter::for('uploads', function (Request $request) {
    return $request->user()->vipCustomer()
        ? Limit::none()
        : Limit::perMinute(100)->by($request->ip());
});
```

為了用另一個範例說明此功能，我們可以將路由的存取限制為：已驗證的使用者 ID 每分鐘 100 次，訪客則是每個 IP 地址每分鐘 10 次：

```php
RateLimiter::for('uploads', function (Request $request) {
    return $request->user()
        ? Limit::perMinute(100)->by($request->user()->id)
        : Limit::perMinute(10)->by($request->ip());
});
```

<a name="multiple-rate-limits"></a>
#### 多重速率限制

如果需要，您可以為給定的速率限制器設定回傳一個速率限制陣列。每個速率限制都會依據它們在陣列中的放置順序在路由上進行評估：

```php
RateLimiter::for('login', function (Request $request) {
    return [
        Limit::perMinute(500),
        Limit::perMinute(3)->by($request->input('email')),
    ];
});
```

如果您要指派由相同的 `by` 值所分段的多個速率限制，您應該確保每個 `by` 值都是唯一的。達成此目的最簡單的方法是為傳給 `by` 方法的值加上前綴：

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

除了對傳入請求進行速率限制之外，Laravel 還允許您使用 `after` 方法基於回應來進行速率限制。當您只想將特定的回應計算在速率限制內時（例如驗證錯誤、404 回應或其他特定的 HTTP 狀態碼），這非常有用。

`after` 方法接收一個閉包，該閉包會取得回應，如果該回應應該被計入速率限制，則應回傳 `true`，如果應該忽略則回傳 `false`。這對於透過限制連續 404 回應來防止列舉攻擊特別有用，或者允許使用者重試驗證失敗的請求，而不會在應該只限制成功操作的端點上耗盡其速率限制額度：

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

可以使用 `throttle` [中介層](/docs/{{version}}/middleware) 將速率限制器附加到路由或路由群組。throttle 中介層接收您希望指派給該路由的速率限制器名稱：

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
#### 使用 Redis 進行限制

預設情況下，`throttle` 中介層會映射到 `Illuminate\Routing\Middleware\ThrottleRequests` 類別。然而，如果您使用 Redis 作為應用程式的快取驅動程式，您可能會希望指示 Laravel 使用 Redis 來管理速率限制。為此，您應該在應用程式的 `bootstrap/app.php` 檔案中使用 `throttleWithRedis` 方法。此方法會將 `throttle` 中介層映射到 `Illuminate\Routing\Middleware\ThrottleRequestsWithRedis` 中介層類別：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->throttleWithRedis();
    // ...
})
```

<a name="form-method-spoofing"></a>
## 表單 HTTP 方法偽裝

HTML 表單不支援 `PUT`、`PATCH` 或 `DELETE` 動作。因此，當定義由 HTML 表單呼叫的 `PUT`、`PATCH` 或 `DELETE` 路由時，您需要向表單新增一個隱藏的 `_method` 欄位。與 `_method` 欄位一起發送的值將作為 HTTP 請求方法使用：

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
## 取得當前路由

您可以使用 `Route` Facade 上的 `current`、`currentRouteName` 與 `currentRouteAction` 方法來存取處理傳入請求的路由資訊：

```php
use Illuminate\Support\Facades\Route;

$route = Route::current(); // Illuminate\Routing\Route
$name = Route::currentRouteName(); // string
$action = Route::currentRouteAction(); // string
```

您可以參考 [Route Facade 底層類別](https://api.laravel.com/docs/{{version}}/Illuminate/Routing/Router.html)與 [Route 實例](https://api.laravel.com/docs/{{version}}/Illuminate/Routing/Route.html)的 API 文件，以瞭解路由控制器（Router）與路由類別中所有可用的方法。


<a name="cors"></a>
## 跨來源資源共享 (CORS)

Laravel 可以根據您設定的數值自動回應 CORS 的 `OPTIONS` HTTP 請求。`OPTIONS` 請求會自動由包含在應用程式全域中介層堆疊中的 `HandleCors` [中介層](/docs/{{version}}/middleware)處理。

有時，您可能需要自訂應用程式的 CORS 設定值。您可以透過使用 `config:publish` Artisan 指令發布 `cors` 設定檔來達到此目的：

```shell
php artisan config:publish cors
```

此指令將會在應用程式的 `config` 目錄中放置一個 `cors.php` 設定檔。

> [!NOTE]
> 關於 CORS 以及 CORS 標頭的更多資訊，請參考 [MDN 關於 CORS 的 Web 文件](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#The_HTTP_response_headers)。


<a name="route-caching"></a>
## 路由快取

將應用程式部署到正式環境時，您應該善加利用 Laravel 的路由快取。使用路由快取可以大幅減少註冊應用程式所有路由所需的時間。要產生路由快取，請執行 `route:cache` Artisan 指令：

```shell
php artisan route:cache
```

執行此指令後，系統將會在每次請求時載入快取的路由檔案。請記住，若您新增了任何新路由，就必須重新產生全新的路由快取。因此，您應該只在專案部署期間執行 `route:cache` 指令。

您可以使用 `route:clear` 指令來清除路由快取：

```shell
php artisan route:clear
```