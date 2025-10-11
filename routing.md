# 路由

- [基礎路由](#basic-routing)
    - [預設路由檔案](#the-default-route-files)
    - [重新導向路由](#redirect-routes)
    - [視圖路由](#view-routes)
    - [列出路由](#listing-your-routes)
    - [路由自訂](#routing-customization)
- [路由參數](#route-parameters)
    - [必要參數](#required-parameters)
    - [選用參數](#parameters-optional-parameters)
    - [正規表達式限制](#parameters-regular-expression-constraints)
- [命名路由](#named-routes)
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
- [備援路由](#fallback-routes)
- [速率限制](#rate-limiting)
    - [定義速率限制器](#defining-rate-limiters)
    - [將速率限制器附加至路由](#attaching-rate-limiters-to-routes)
- [表單方法偽裝](#form-method-spoofing)
- [存取目前的路由](#accessing-the-current-route)
- [跨來源資源共享 (CORS)](#cors)
- [路由快取](#route-caching)

<a name="basic-routing"></a>
## 基礎路由

最基礎的 Laravel 路由接受一個 URI 與一個閉包，提供一種非常簡潔且富有表達力的方式來定義路由與行為，而無需複雜的路由設定檔：

    use Illuminate\Support\Facades\Route;

    Route::get('/greeting', function () {
        return 'Hello World';
    });


<a name="the-default-route-files"></a>
### 預設路由檔案

所有 Laravel 路由都定義在您的路由檔案中，這些檔案位於 `routes` 目錄。Laravel 會依據應用程式 `bootstrap/app.php` 檔案中指定的設定，自動載入這些檔案。`routes/web.php` 檔案定義了您的網頁介面路由。這些路由會被指派 `web` [中介層群組](/docs/{{version}}/middleware#laravels-default-middleware-groups)，該群組提供例如 Session 狀態與 CSRF 保護等功能。

對於大多數應用程式而言，您會從 `routes/web.php` 檔案中定義路由開始。`routes/web.php` 中定義的路由可透過在瀏覽器中輸入其定義的路由 URL 來存取。舉例來說，您可以在瀏覽器中前往 `http://example.com/user` 來存取下列路由：

    use App\Http\Controllers\UserController;

    Route::get('/user', [UserController::class, 'index']);


<a name="api-routes"></a>
#### API 路由

如果您的應用程式也將提供無狀態的 API，您可以使用 `install:api` Artisan 指令來啟用 API 路由：

```shell
php artisan install:api
```

此 `install:api` 指令會安裝 [Laravel Sanctum](/docs/{{version}}/sanctum)，它提供了一個強大且簡單的 API 權杖驗證守衛，可用於驗證第三方 API 消費者、SPA 或行動應用程式。此外，`install:api` 指令還會建立 `routes/api.php` 檔案：

    Route::get('/user', function (Request $request) {
        return $request->user();
    })->middleware('auth:sanctum');

`routes/api.php` 中的路由是無狀態的，並被指派給 `api` [中介層群組](/docs/{{version}}/middleware#laravels-default-middleware-groups)。此外，`/api` URI 前綴會自動套用至這些路由，因此您無需手動將其套用至檔案中的每個路由。您可以透過修改應用程式的 `bootstrap/app.php` 檔案來變更前綴：

    ->withRouting(
        api: __DIR__.'/../routes/api.php',
        apiPrefix: 'api/admin',
        // ...
    )


<a name="available-router-methods"></a>
#### 可用的路由方法

路由器允許您註冊可響應任何 HTTP 動詞的路由：

    Route::get($uri, $callback);
    Route::post($uri, $callback);
    Route::put($uri, $callback);
    Route::patch($uri, $callback);
    Route::delete($uri, $callback);
    Route::options($uri, $callback);

有時候您可能需要註冊一個響應多個 HTTP 動詞的路由。您可以使用 `match` 方法來實現。或者，您甚至可以使用 `any` 方法註冊一個響應所有 HTTP 動詞的路由：

    Route::match(['get', 'post'], '/', function () {
        // ...
    });

    Route::any('/', function () {
        // ...
    });

> [!NOTE]  
> 當定義多個共用相同 URI 的路由時，使用 `get`、`post`、`put`、`patch`、`delete` 和 `options` 方法的路由應在使用 `any`、`match` 和 `redirect` 方法的路由之前定義。這確保傳入的請求與正確的路由匹配。


<a name="dependency-injection"></a>
#### 依賴注入

您可以在路由的閉包簽章中，型別提示路由所需的任何依賴。Laravel [服務容器](/docs/{{version}}/container)會自動解析宣告的依賴並將其注入到閉包中。舉例來說，您可以型別提示 `Illuminate\Http\Request` 類別，以便將目前的 HTTP 請求自動注入到路由閉包中：

    use Illuminate\Http\Request;

    Route::get('/users', function (Request $request) {
        // ...
    });


<a name="csrf-protection"></a>
#### CSRF 保護

請記住，任何指向 `POST`、`PUT`、`PATCH` 或 `DELETE` 路由（定義在 `web` 路由檔案中）的 HTML 表單，都應包含一個 CSRF 權杖欄位。否則，請求將會被拒絕。您可以閱讀 [CSRF 文件](/docs/{{version}}/csrf)以了解更多關於 CSRF 保護的資訊：

    <form method="POST" action="/profile">
        @csrf
        ...
    </form>


<a name="redirect-routes"></a>
### 重新導向路由

如果您正在定義一個重新導向到另一個 URI 的路由，您可以使用 `Route::redirect` 方法。這個方法提供了一個方便的捷徑，讓您不必為執行簡單的重新導向而定義完整的路由或控制器：

    Route::redirect('/here', '/there');

預設情況下，`Route::redirect` 會回傳 `302` 狀態碼。您可以使用可選的第三個參數來客製化狀態碼：

    Route::redirect('/here', '/there', 301);

或者，您可以使用 `Route::permanentRedirect` 方法來回傳 `301` 狀態碼：

    Route::permanentRedirect('/here', '/there');

> [!WARNING]  
> 當在重新導向路由中使用路由參數時，以下參數為 Laravel 保留，不能使用：`destination` 和 `status`。


<a name="view-routes"></a>
### 視圖路由

如果您的路由只需回傳一個 [視圖](/docs/{{version}}/views)，您可以使用 `Route::view` 方法。與 `redirect` 方法一樣，此方法提供了一個簡單的捷徑，讓您不必定義完整的路由或控制器。`view` 方法接受 URI 作為第一個參數，視圖名稱作為第二個參數。此外，您還可以提供一個包含資料的陣列作為可選的第三個參數，以傳遞給視圖：

    Route::view('/welcome', 'welcome');

    Route::view('/welcome', 'welcome', ['name' => 'Taylor']);

> [!WARNING]  
> 當在視圖路由中使用路由參數時，以下參數為 Laravel 保留，不能使用：`view`、`data`、`status` 和 `headers`。


<a name="listing-your-routes"></a>
### 列出路由

Artisan 指令 `route:list` 可以輕鬆地提供應用程式中所有已定義路由的總覽：

```shell
php artisan route:list
```

預設情況下，指派給每個路由的路由中介層不會顯示在 `route:list` 的輸出中；但是，您可以透過向指令新增 `-v` 選項來指示 Laravel 顯示路由中介層和中介層群組名稱：

```shell
php artisan route:list -v

# Expand middleware groups...
php artisan route:list -vv
```

您也可以指示 Laravel 僅顯示以指定 URI 開頭的路由：

```shell
php artisan route:list --path=api
```

此外，您可以透過在執行 `route:list` 指令時提供 `--except-vendor` 選項，來指示 Laravel 隱藏任何由第三方套件定義的路由：

```shell
php artisan route:list --except-vendor
```

同樣地，您也可以透過在執行 `route:list` 指令時提供 `--only-vendor` 選項，來指示 Laravel 僅顯示由第三方套件定義的路由：

```shell
php artisan route:list --only-vendor
```

<a name="routing-customization"></a>
### 路由自訂

預設情況下，應用程式的路由是透過 `bootstrap/app.php` 檔案進行配置和載入的：

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

然而，有時您可能希望定義一個全新的檔案來包含應用程式路由的子集。為此，您可以為 `withRouting` 方法提供一個 `then` 閉包。在此閉包內，您可以註冊應用程式所需的任何額外路由：

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

有時，您需要在路由中擷取 URI 的片段。例如，您可能需要從 URL 擷取使用者的 ID。您可以透過定義路由參數來完成此操作：

    Route::get('/user/{id}', function (string $id) {
        return 'User '.$id;
    });

您可以根據路由需求定義任意數量的路由參數：

    Route::get('/posts/{post}/comments/{comment}', function (string $postId, string $commentId) {
        // ...
    });

路由參數總是使用 `{}` 大括號包圍，並且應由字母字元組成。底線符號 (`_`) 在路由參數名稱中也是可接受的。路由參數會依照其順序注入路由回呼或控制器中 —— 路由回呼或控制器參數的名稱並不重要。

<a name="parameters-and-dependency-injection"></a>
#### 參數與依賴注入

如果您的路由有您希望 Laravel 服務容器自動注入到路由回呼中的依賴，您應該將路由參數列在您的依賴之後：

    use Illuminate\Http\Request;

    Route::get('/user/{id}', function (Request $request, string $id) {
        return 'User '.$id;
    });

<a name="parameters-optional-parameters"></a>
### 選用參數

有時您可能需要指定一個不一定存在於 URI 中的路由參數。您可以透過在參數名稱後加上 `?` 符號來實現。請務必為路由的對應變數賦予一個預設值：

    Route::get('/user/{name?}', function (?string $name = null) {
        return $name;
    });

    Route::get('/user/{name?}', function (?string $name = 'John') {
        return $name;
    });

<a name="parameters-regular-expression-constraints"></a>
### 正規表達式限制

您可以使用路由實例上的 `where` 方法來約束路由參數的格式。`where` 方法接受參數名稱和一個定義參數應如何約束的正規表達式：

    Route::get('/user/{name}', function (string $name) {
        // ...
    })->where('name', '[A-Za-z]+');

    Route::get('/user/{id}', function (string $id) {
        // ...
    })->where('id', '[0-9]+');

    Route::get('/user/{id}/{name}', function (string $id, string $name) {
        // ...
    })->where(['id' => '[0-9]+', 'name' => '[a-z]+']);

為了方便起見，一些常用的正規表達式模式擁有輔助方法，讓您可以快速為路由新增模式約束：

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

如果傳入的請求不符合路由模式約束，將會回傳一個 404 HTTP 回應。

<a name="parameters-global-constraints"></a>
#### 全域約束

如果您希望路由參數始終由給定的正規表達式約束，可以使用 `pattern` 方法。您應該在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中定義這些模式：

    use Illuminate\Support\Facades\Route;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Route::pattern('id', '[0-9]+');
    }

一旦模式被定義，它會自動應用於所有使用該參數名稱的路由：

    Route::get('/user/{id}', function (string $id) {
        // Only executed if {id} is numeric...
    });

<a name="parameters-encoded-forward-slashes"></a>
#### 編碼斜線

Laravel 路由元件允許所有字元（除了 `/`）存在於路由參數值中。您必須明確允許 `/` 透過 `where` 條件正規表達式成為佔位符的一部分：

    Route::get('/search/{search}', function (string $search) {
        return $search;
    })->where('search', '.*');

> [!WARNING]  
> 編碼斜線僅支援在路由的最後一個片段中使用。

<a name="named-routes"></a>
## 命名路由

命名路由允許方便地產生特定路由的 URL 或重新導向。您可以透過將 `name` 方法鏈接到路由定義上來為路由指定名稱：

    Route::get('/user/profile', function () {
        // ...
    })->name('profile');

您也可以為控制器動作指定路由名稱：

    Route::get(
        '/user/profile',
        [UserProfileController::class, 'show']
    )->name('profile');

> [!WARNING]  
> 路由名稱應始終是唯一的。

<a name="generating-urls-to-named-routes"></a>
#### 產生命名路由的 URL

一旦您為指定路由分配了名稱，您就可以透過 Laravel 的 `route` 和 `redirect` 輔助函式在產生 URL 或重新導向時使用路由的名稱：

    // Generating URLs...
    $url = route('profile');

    // Generating Redirects...
    return redirect()->route('profile');

    return to_route('profile');

如果命名路由定義了參數，您可以將這些參數作為第二個引數傳遞給 `route` 函式。給定的參數將自動插入到產生的 URL 的正確位置：

    Route::get('/user/{id}/profile', function (string $id) {
        // ...
    })->name('profile');

    $url = route('profile', ['id' => 1]);

如果您在陣列中傳遞額外的參數，這些鍵/值對將自動新增到產生的 URL 的查詢字串：

    Route::get('/user/{id}/profile', function (string $id) {
        // ...
    })->name('profile');

    $url = route('profile', ['id' => 1, 'photos' => 'yes']);

    // /user/1/profile?photos=yes

> [!NOTE]  
> 有時，您可能希望為 URL 參數指定請求範圍的預設值，例如目前的語言環境。要實現此目的，您可以使用 [`URL::defaults` 方法](/docs/{{version}}/urls#default-values)。

<a name="inspecting-the-current-route"></a>
#### 檢查目前的路由

如果您想判斷目前的請求是否被路由到給定的命名路由，您可以使用 Route 實例上的 `named` 方法。例如，您可以從路由中介層檢查目前的路由名稱：

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

<a name="route-groups"></a>
## 路由群組

路由群組允許您在多個路由之間共用路由屬性（例如中介層），而無需在每個個別路由上定義這些屬性。

巢狀群組會嘗試智慧地將屬性與其父群組「合併」。中介層與 `where` 條件會被合併，而名稱和前綴則會被附加。命名空間分隔符號和 URI 前綴中的斜線會自動在適當位置添加。

<a name="route-group-middleware"></a>
### 中介層

若要為群組內的所有路由指派 [中介層](/docs/{{version}}/middleware)，您可以在定義群組之前使用 `middleware` 方法。中介層會按照其在陣列中的列出順序執行：

    Route::middleware(['first', 'second'])->group(function () {
        Route::get('/', function () {
            // Uses first & second middleware...
        });

        Route::get('/user/profile', function () {
            // Uses first & second middleware...
        });
    });

<a name="route-group-controllers"></a>
### 控制器

如果一組路由都使用相同的 [控制器](/docs/{{version}}/controllers)，您可以使用 `controller` 方法來為群組內的所有路由定義共同的控制器。然後，在定義路由時，您只需要提供它們所呼叫的控制器方法即可：

    use App\Http\Controllers\OrderController;

    Route::controller(OrderController::class)->group(function () {
        Route::get('/orders/{id}', 'show');
        Route::post('/orders', 'store');
    });

<a name="route-group-subdomain-routing"></a>
### 子網域路由

路由群組也可用於處理子網域路由。子網域可以像路由 URI 一樣被指派路由參數，讓您可以擷取子網域的一部分以用於路由或控制器。您可以在定義群組之前呼叫 `domain` 方法來指定子網域：

    Route::domain('{account}.example.com')->group(function () {
        Route::get('/user/{id}', function (string $account, string $id) {
            // ...
        });
    });

> [!WARNING]  
> 為了確保您的子網域路由可被存取，您應該在註冊根網域路由之前註冊子網域路由。這將防止根網域路由覆寫具有相同 URI 路徑的子網域路由。

<a name="route-group-prefixes"></a>
### 路由前綴

`prefix` 方法可用來為群組中的每個路由加上給定的 URI 前綴。例如，您可能希望將群組內所有路由的 URI 前綴設定為 `admin`：

    Route::prefix('admin')->group(function () {
        Route::get('/users', function () {
            // Matches The "/admin/users" URL
        });
    });

<a name="route-group-name-prefixes"></a>
### 路由名稱前綴

`name` 方法可用來為群組中的每個路由名稱加上給定的字串前綴。例如，您可能希望將群組內所有路由的名稱前綴設定為 `admin`。所給定的字串會完全按照指定的方式作為路由名稱的前綴，因此我們務必在字串中提供尾隨的 `.` 字元：

    Route::name('admin.')->group(function () {
        Route::get('/users', function () {
            // Route assigned name "admin.users"...
        })->name('users');
    });

<a name="route-model-binding"></a>
## 路由模型綁定

當將模型 ID 注入到路由或控制器動作時，您通常會查詢資料庫以取得對應的那個模型。Laravel 路由模型綁定提供了一種便捷方式，可將模型實例直接自動注入到路由中。例如，您無需注入使用者的 ID，而是可以注入與給定 ID 相符的整個 `User` 模型實例。

<a name="implicit-binding"></a>
### 隱式綁定

Laravel 會自動解析定義在路由或控制器動作中、且其型別提示的變數名稱與路由區段名稱相符的 Eloquent 模型。例如：

    use App\Models\User;

    Route::get('/users/{user}', function (User $user) {
        return $user->email;
    });

由於 `$user` 變數被型別提示為 `App\Models\User` Eloquent 模型，且變數名稱與 `{user}` URI 區段相符，Laravel 將會自動注入與請求 URI 中對應值相符的 ID 的模型實例。如果資料庫中找不到相符的模型實例，則會自動生成一個 404 HTTP 回應。

當然，隱式綁定也適用於控制器方法。再次注意，`{user}` URI 區段與控制器中包含 `App\Models\User` 型別提示的 `$user` 變數相符：

    use App\Http\Controllers\UserController;
    use App\Models\User;

    // Route definition...
    Route::get('/users/{user}', [UserController::class, 'show']);

    // Controller method definition...
    public function show(User $user)
    {
        return view('user.profile', ['user' => $user]);
    }

<a name="implicit-soft-deleted-models"></a>
#### 軟刪除模型

隱式模型綁定通常不會取得已 [軟刪除](/docs/{{version}}/eloquent#soft-deleting) 的模型。不過，您可以透過在路由定義上鏈接 `withTrashed` 方法來指示隱式綁定取得這些模型：

    use App\Models\User;

    Route::get('/users/{user}', function (User $user) {
        return $user->email;
    })->withTrashed();

<a name="customizing-the-default-key-name"></a>
#### 自訂鍵名

有時您可能希望使用 `id` 以外的欄位來解析 Eloquent 模型。為此，您可以在路由參數定義中指定欄位：

    use App\Models\Post;

    Route::get('/posts/{post:slug}', function (Post $post) {
        return $post;
    });

如果您希望模型綁定在取得給定模型類別時，始終使用 `id` 以外的資料庫欄位，您可以在 Eloquent 模型上覆寫 `getRouteKeyName` 方法：

    /**
     * Get the route key for the model.
     */
    public function getRouteKeyName(): string
    {
        return 'slug';
    }

<a name="implicit-model-binding-scoping"></a>
#### 自訂鍵與範圍綁定

當在單一路由定義中隱式綁定多個 Eloquent 模型時，您可能希望將第二個 Eloquent 模型範圍綁定為前一個 Eloquent 模型的子項。例如，考慮這個路由定義，它會為特定使用者透過 slug 取得一篇部落格文章：

    use App\Models\Post;
    use App\Models\User;

    Route::get('/users/{user}/posts/{post:slug}', function (User $user, Post $post) {
        return $post;
    });

當將自訂鍵的隱式綁定用作巢狀路由參數時，Laravel 會自動範圍綁定查詢，透過其父項來取得巢狀模型，並利用慣例來猜測父項上的關聯名稱。在此情況下，會假設 `User` 模型有一個名為 `posts` 的關聯 (路由參數名稱的複數形式)，可用於取得 `Post` 模型。

如果您願意，即使未提供自訂鍵，您也可以指示 Laravel 範圍綁定「子」綁定。為此，您可以在定義路由時呼叫 `scopeBindings` 方法：

    use App\Models\Post;
    use App\Models\User;

    Route::get('/users/{user}/posts/{post}', function (User $user, Post $post) {
        return $post;
    })->scopeBindings();

或者，您可以指示一整個路由定義群組使用範圍綁定：

    Route::scopeBindings()->group(function () {
        Route::get('/users/{user}/posts/{post}', function (User $user, Post $post) {
            return $post;
        });
    });

同樣地，您可以透過呼叫 `withoutScopedBindings` 方法，明確指示 Laravel 不要範圍綁定：

    Route::get('/users/{user}/posts/{post:slug}', function (User $user, Post $post) {
        return $post;
    })->withoutScopedBindings();

<a name="customizing-missing-model-behavior"></a>
#### 自訂模型缺失行為

通常，如果找不到隱式綁定的模型，將會生成 404 HTTP 回應。不過，您可以在定義路由時呼叫 `missing` 方法來自訂此行為。`missing` 方法接受一個閉包，如果找不到隱式綁定的模型，該閉包將會被呼叫：

    use App\Http\Controllers\LocationsController;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Redirect;

    Route::get('/locations/{location:slug}', [LocationsController::class, 'show'])
        ->name('locations.view')
        ->missing(function (Request $request) {
            return Redirect::route('locations.index');
        });

<a name="implicit-enum-binding"></a>
### 隱式 Enum 綁定

PHP 8.1 引入了對 [Enums](https://www.php.net/manual/en/language.enumerations.backed.php) 的支援。為了補充此功能，Laravel 允許您在路由定義上型別提示 [字串支援的 Enum](https://www.php.net/manual/en/language.enumerations.backed.php)，並且 Laravel 將僅在該路由區段對應到有效的 Enum 值時才呼叫該路由。否則，將會自動返回 404 HTTP 回應。例如，給定以下 Enum：

```php
<?php

namespace App\Enums;

enum Category: string
{
    case Fruits = 'fruits';
    case People = 'people';
}
```

您可以定義一個路由，該路由將僅在 `{category}` 路由區段為 `fruits` 或 `people` 時被呼叫。否則，Laravel 將會返回 404 HTTP 回應：

```php
use App\Enums\Category;
use Illuminate\Support\Facades\Route;

Route::get('/categories/{category}', function (Category $category) {
    return $category->value;
});
```

<a name="explicit-binding"></a>
### 顯式綁定

您不必使用 Laravel 隱式、基於慣例的模型解析來使用模型綁定。您也可以明確定義路由參數如何對應到模型。要註冊顯式綁定，請使用路由器的 `model` 方法來指定給定參數的類別。您應該在應用程式 `AppServiceProvider` 類別的 `boot` 方法開頭定義您的顯式模型綁定：

    use App\Models\User;
    use Illuminate\Support\Facades\Route;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Route::model('user', User::class);
    }

接著，定義一個包含 `{user}` 參數的路由：

    use App\Models\User;

    Route::get('/users/{user}', function (User $user) {
        // ...
    });

由於我們已將所有 `{user}` 參數綁定到 `App\Models\User` 模型，因此該類別的實例將會被注入到路由中。舉例來說，對 `users/1` 的請求將會從資料庫中注入 ID 為 `1` 的 `User` 實例。

如果在資料庫中找不到匹配的模型實例，將會自動產生 404 HTTP 回應。

<a name="customizing-the-resolution-logic"></a>
#### 自訂解析邏輯

如果您希望定義自己的模型綁定解析邏輯，您可以使用 `Route::bind` 方法。傳遞給 `bind` 方法的閉包將會接收 URI 片段的值，並應回傳應注入到路由中的類別實例。同樣地，此自訂應在應用程式 `AppServiceProvider` 的 `boot` 方法中進行：

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

或者，您可以覆寫 Eloquent 模型的 `resolveRouteBinding` 方法。此方法將接收 URI 片段的值，並應回傳應注入到路由中的類別實例：

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

如果路由正在使用 [隱式綁定作用域](#implicit-model-binding-scoping)，則將使用 `resolveChildRouteBinding` 方法來解析父模型的子綁定：

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

<a name="fallback-routes"></a>
## 備援路由

使用 `Route::fallback` 方法，您可以定義一個路由，當沒有其他路由與傳入的請求匹配時，該路由將會被執行。通常，未處理的請求會透過應用程式的例外處理器自動渲染「404」頁面。然而，由於您通常會在 `routes/web.php` 檔案中定義 `fallback` 路由，因此 `web` 中介層群組中的所有中介層都將應用於該路由。您可以根據需要自由地為此路由添加額外的中介層：

    Route::fallback(function () {
        // ...
    });


<a name="rate-limiting"></a>
## 速率限制


<a name="defining-rate-limiters"></a>
### 定義速率限制器

Laravel 包含功能強大且可自訂的速率限制服務，您可以用來限制給定路由或路由群組的流量。首先，您應該定義符合應用程式需求的速率限制器配置。

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

速率限制器是使用 `RateLimiter` Facade 的 `for` 方法定義的。 `for` 方法接受一個速率限制器名稱和一個閉包，該閉包會回傳應套用於分配給該速率限制器的路由的限制配置。限制配置是 `Illuminate\Cache\RateLimiting\Limit` 類別的實例。此類別包含有用的「建構器」方法，讓您可以快速定義限制。速率限制器名稱可以是您想要的任何字串：

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

如果傳入的請求超過指定的速率限制，Laravel 將會自動回傳帶有 429 HTTP 狀態碼的回應。如果您想定義自己的回應，該回應應由速率限制回傳，則可以使用 `response` 方法：

    RateLimiter::for('global', function (Request $request) {
        return Limit::perMinute(1000)->response(function (Request $request, array $headers) {
            return response('Custom response...', 429, $headers);
        });
    });

由於速率限制器回呼函式會接收傳入的 HTTP 請求實例，因此您可以根據傳入的請求或經過驗證的使用者動態地建立適當的速率限制：

    RateLimiter::for('uploads', function (Request $request) {
        return $request->user()->vipCustomer()
            ? Limit::none()
            : Limit::perMinute(100);
    });


<a name="segmenting-rate-limits"></a>
#### 區分速率限制

有時您可能希望按任意值區分速率限制。例如，您可能希望允許使用者每分鐘每 IP 位址存取給定路由 100 次。為此，您可以在建構速率限制時使用 `by` 方法：

    RateLimiter::for('uploads', function (Request $request) {
        return $request->user()->vipCustomer()
            ? Limit::none()
            : Limit::perMinute(100)->by($request->ip());
    });

為了用另一個範例來說明此功能，我們可以將路由的存取限制為每分鐘每經過驗證的使用者 ID 100 次，或每分鐘每 IP 位址 10 次（針對訪客）：

    RateLimiter::for('uploads', function (Request $request) {
        return $request->user()
            ? Limit::perMinute(100)->by($request->user()->id)
            : Limit::perMinute(10)->by($request->ip());
    });


<a name="multiple-rate-limits"></a>
#### 多重速率限制

如有需要，您可以為給定的速率限制器配置回傳一個速率限制陣列。每個速率限制將根據它們在陣列中的順序為路由進行評估：

    RateLimiter::for('login', function (Request $request) {
        return [
            Limit::perMinute(500),
            Limit::perMinute(3)->by($request->input('email')),
        ];
    });

如果您正在分配多個以相同 `by` 值區分的速率限制，您應該確保每個 `by` 值都是唯一的。最簡單的方法是為 `by` 方法提供的值加上前綴：

    RateLimiter::for('uploads', function (Request $request) {
        return [
            Limit::perMinute(10)->by('minute:'.$request->user()->id),
            Limit::perDay(1000)->by('day:'.$request->user()->id),
        ];
    });


<a name="attaching-rate-limiters-to-routes"></a>
### 將速率限制器附加至路由

速率限制器可以使用 `throttle` [中介層](/docs/{{version}}/middleware) 附加到路由或路由群組。 throttle 中介層接受您希望分配給路由的速率限制器名稱：

    Route::middleware(['throttle:uploads'])->group(function () {
        Route::post('/audio', function () {
            // ...
        });

        Route::post('/video', function () {
            // ...
        });
    });


<a name="throttling-with-redis"></a>
#### 使用 Redis 進行速率限制

預設情況下，`throttle` 中介層會映射到 `Illuminate\Routing\Middleware\ThrottleRequests` 類別。但是，如果您使用 Redis 作為應用程式的快取驅動程式，您可能希望指示 Laravel 使用 Redis 來管理速率限制。為此，您應該在應用程式的 `bootstrap/app.php` 檔案中使用 `throttleWithRedis` 方法。此方法會將 `throttle` 中介層映射到 `Illuminate\Routing\Middleware\ThrottleRequestsWithRedis` 中介層類別：

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->throttleWithRedis();
        // ...
    })


<a name="form-method-spoofing"></a>
## 表單方法偽裝

HTML 表單不支援 `PUT`、`PATCH` 或 `DELETE` 動作。因此，當定義從 HTML 表單呼叫的 `PUT`、`PATCH` 或 `DELETE` 路由時，您需要向表單添加一個隱藏的 `_method` 欄位。與 `_method` 欄位一起傳送的值將用作 HTTP 請求方法：

    <form action="/example" method="POST">
        <input type="hidden" name="_method" value="PUT">
        <input type="hidden" name="_token" value="{{ csrf_token() }}">
    </form>

為方便起見，您可以使用 `@method` [Blade 指令](/docs/{{version}}/blade) 來生成 `_method` 輸入欄位：

    <form action="/example" method="POST">
        @method('PUT')
        @csrf
    </form>


<a name="accessing-the-current-route"></a>
## 存取目前的路由

您可以使用 `Route` Facade 上的 `current`、`currentRouteName` 和 `currentRouteAction` 方法來存取有關處理傳入請求的路由資訊：

    use Illuminate\Support\Facades\Route;

    $route = Route::current(); // Illuminate\Routing\Route
    $name = Route::currentRouteName(); // string
    $action = Route::currentRouteAction(); // string

您可以參考 [Route facade 的底層類別](https://laravel.com/api/{{version}}/Illuminate/Routing/Router.html) 和 [Route 實例](https://laravel.com/api/{{version}}/Illuminate/Routing/Route.html) 的 API 文件，以查閱路由器和路由類別中所有可用的方法。

<a name="cors"></a>
## 跨來源資源共享 (CORS)

Laravel 可以自動回應 CORS `OPTIONS` HTTP 請求，並使用你設定的值。`OPTIONS` 請求將會由 `HandleCors` [中介層](/docs/{{version}}/middleware) 自動處理，這個中介層會自動包含在應用程式的全域中介層堆疊中。

有時候，你可能需要自訂應用程式的 CORS 設定值。你可以透過發布 `cors` 設定檔來完成，使用 `config:publish` Artisan 指令：

```shell
php artisan config:publish cors
```

這個指令會將 `cors.php` 設定檔放置到你的應用程式 `config` 目錄中。

> [!NOTE]  
> 關於 CORS 和 CORS 標頭的更多資訊，請查閱 [MDN 網頁上的 CORS 文件](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#The_HTTP_response_headers)。

<a name="route-caching"></a>
## 路由快取

當將應用程式部署到生產環境時，你應該利用 Laravel 的路由快取。使用路由快取將會大幅減少註冊應用程式所有路由所需的時間。要產生路由快取，請執行 `route:cache` Artisan 指令：

```shell
php artisan route:cache
```

執行此指令後，你的快取路由檔將會在每次請求時載入。請記住，如果你新增任何路由，你將需要重新產生路由快取。因此，你應該只在專案部署期間執行 `route:cache` 指令。

你可以使用 `route:clear` 指令來清除路由快取：

```shell
php artisan route:clear
```