# 控制器

- [簡介](#introduction)
- [撰寫控制器](#writing-controllers)
    - [基本控制器](#basic-controllers)
    - [單一動作控制器](#single-action-controllers)
- [控制器 Middleware](#controller-middleware)
- [資源控制器](#resource-controllers)
    - [部分資源路由](#restful-partial-resource-routes)
    - [巢狀資源](#restful-nested-resources)
    - [命名資源路由](#restful-naming-resource-routes)
    - [命名資源路由參數](#restful-naming-resource-route-parameters)
    - [範圍化資源路由](#restful-scoping-resource-routes)
    - [本地化資源 URI](#restful-localizing-resource-uris)
    - [補充資源控制器](#restful-supplementing-resource-controllers)
    - [單例資源控制器](#singleton-resource-controllers)
    - [Middleware 與資源控制器](#middleware-and-resource-controllers)
- [依賴注入與控制器](#dependency-injection-and-controllers)

<a name="introduction"></a>
## 簡介

除了在路由檔案中將所有請求處理邏輯定義為閉包之外，您可能希望使用「控制器」類別來組織此行為。控制器可以將相關的請求處理邏輯分組到單一類別中。例如，`UserController` 類別可以處理所有與使用者相關的傳入請求，包括顯示、建立、更新和刪除使用者。預設情況下，控制器儲存在 `app/Http/Controllers` 目錄中。

<a name="writing-controllers"></a>
## 撰寫控制器

<a name="basic-controllers"></a>
### 基本控制器

若要快速產生新的控制器，您可以執行 `make:controller` Artisan 命令。預設情況下，應用程式的所有控制器都儲存在 `app/Http/Controllers` 目錄中：

```shell
php artisan make:controller UserController
```

讓我們看看一個基本控制器的範例。控制器可以有任意數量的公開方法，這些方法將回應傳入的 HTTP 請求：

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\View\View;

class UserController extends Controller
{
    /**
     * Show the profile for a given user.
     */
    public function show(string $id): View
    {
        return view('user.profile', [
            'user' => User::findOrFail($id)
        ]);
    }
}
```

一旦您撰寫了控制器類別和方法，您可以像這樣定義指向控制器方法的路由：

```php
use App\Http\Controllers\UserController;

Route::get('/user/{id}', [UserController::class, 'show']);
```

當傳入的請求與指定的路由 URI 匹配時，`App\Http\Controllers\UserController` 類別上的 `show` 方法將被呼叫，並且路由參數將傳遞給該方法。

> [!NOTE]
> 控制器**不要求**繼承基礎類別。然而，有時繼承包含應在所有控制器之間共用的方法的基礎控制器類別會很方便。

<a name="single-action-controllers"></a>
### 單一動作控制器

如果控制器動作特別複雜，您可能會發現將整個控制器類別專用於該單一動作很方便。為此，您可以在控制器中定義一個單一的 `__invoke` 方法：

```php
<?php

namespace App\Http\Controllers;

class ProvisionServer extends Controller
{
    /**
     * Provision a new web server.
     */
    public function __invoke()
    {
        // ...
    }
}
```

為單一動作控制器註冊路由時，您不需要指定控制器方法。相反，您可以直接將控制器的名稱傳遞給路由器：

```php
use App\Http\Controllers\ProvisionServer;

Route::post('/server', ProvisionServer::class);
```

您可以使用 `make:controller` Artisan 命令的 `--invokable` 選項來產生一個可呼叫的控制器：

```shell
php artisan make:controller ProvisionServer --invokable
```

> [!NOTE]
> 控制器 Stub 可以使用[Stub 發佈](/docs/{{version}}/artisan#stub-customization)進行自訂。

<a name="controller-middleware"></a>
## 控制器 Middleware

[Middleware](/docs/{{version}}/middleware) 可以指派給路由檔案中的控制器路由：

```php
Route::get('/profile', [UserController::class, 'show'])->middleware('auth');
```

或者，您可能會發現在控制器類別中指定 Middleware 很方便。為此，您的控制器應實作 `HasMiddleware` 介面，該介面規定控制器應具有靜態 `middleware` 方法。從此方法中，您可以回傳一個應套用至控制器動作的 Middleware 陣列：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Routing\Controllers\HasMiddleware;
use Illuminate\Routing\Controllers\Middleware;

class UserController extends Controller implements HasMiddleware
{
    /**
     * Get the middleware that should be assigned to the controller.
     */
    public static function middleware(): array
    {
        return [
            'auth',
            new Middleware('log', only: ['index']),
            new Middleware('subscribed', except: ['store']),
        ];
    }

    // ...
}
```

您也可以將控制器 Middleware 定義為閉包，這提供了一種方便的方式來定義內聯 Middleware，而無需撰寫整個 Middleware 類別：

```php
use Closure;
use Illuminate\Http\Request;

/**
 * Get the middleware that should be assigned to the controller.
 */
public static function middleware(): array
{
    return [
        function (Request $request, Closure $next) {
            return $next($request);
        },
    ];
}
```

<a name="resource-controllers"></a>
## 資源控制器

如果您將應用程式中的每個 Eloquent 模型視為一個「資源」，那麼對應用程式中的每個資源執行相同的動作集是很常見的。例如，假設您的應用程式包含一個 `Photo` 模型和一個 `Movie` 模型。使用者很可能可以建立、讀取、更新或刪除這些資源。

由於這種常見的用例，Laravel 資源路由透過一行程式碼將典型的建立、讀取、更新和刪除 (「CRUD」) 路由指派給控制器。首先，我們可以使用 `make:controller` Artisan 命令的 `--resource` 選項快速建立一個控制器來處理這些動作：

```shell
php artisan make:controller PhotoController --resource
```

此命令將在 `app/Http/Controllers/PhotoController.php` 產生一個控制器。該控制器將包含每個可用資源操作的方法。接下來，您可以註冊一個指向該控制器的資源路由：

```php
use App\Http\Controllers\PhotoController;

Route::resource('photos', PhotoController::class);
```

這個單一的路由宣告會建立多個路由來處理資源上的各種動作。產生的控制器將已經為每個動作提供了 Stub 方法。請記住，您始終可以透過執行 `route:list` Artisan 命令來快速概覽應用程式的路由。

您甚至可以透過將陣列傳遞給 `resources` 方法來一次註冊多個資源控制器：

```php
Route::resources([
    'photos' => PhotoController::class,
    'posts' => PostController::class,
]);
```

`softDeletableResources` 方法註冊了許多資源控制器，這些控制器都使用 `withTrashed` 方法：

```php
Route::softDeletableResources([
    'photos' => PhotoController::class,
    'posts' => PostController::class,
]);
```

<a name="actions-handled-by-resource-controllers"></a>
#### 資源控制器處理的動作

<div class="overflow-auto">

| 動詞      | URI                    | 動作    | 路由名稱       |
| --------- | ---------------------- | ------- | -------------- |
| GET       | `/photos`              | index   | photos.index   |
| GET       | `/photos/create`       | create  | photos.create  |
| POST      | `/photos`              | store   | photos.store   |
| GET       | `/photos/{photo}`      | show    | photos.show    |
| GET       | `/photos/{photo}/edit` | edit    | photos.edit    |
| PUT/PATCH | `/photos/{photo}`      | update  | photos.update  |
| DELETE    | `/photos/{photo}`      | destroy | photos.destroy |

</div>

<a name="customizing-missing-model-behavior"></a>
#### 自訂遺失模型行為

通常，如果找不到隱式綁定的資源模型，將會產生 404 HTTP 回應。但是，您可以在定義資源路由時呼叫 `missing` 方法來自訂此行為。`missing` 方法接受一個閉包，如果找不到任何資源路由的隱式綁定模型，則會呼叫該閉包：

```php
use App\Http\Controllers\PhotoController;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Redirect;

Route::resource('photos', PhotoController::class)
    ->missing(function (Request $request) {
        return Redirect::route('photos.index');
    });
```

<a name="soft-deleted-models"></a>
#### 軟刪除模型

通常，隱式模型綁定不會檢索已[軟刪除](/docs/{{version}}/eloquent#soft-deleting)的模型，而是會回傳 404 HTTP 回應。但是，您可以在定義資源路由時呼叫 `withTrashed` 方法，指示框架允許軟刪除模型：

```php
use App\Http\Controllers\PhotoController;

Route::resource('photos', PhotoController::class)->withTrashed();
```

不帶參數呼叫 `withTrashed` 將允許 `show`、`edit` 和 `update` 資源路由的軟刪除模型。您可以透過將陣列傳遞給 `withTrashed` 方法來指定這些路由的子集：

```php
Route::resource('photos', PhotoController::class)->withTrashed(['show']);
```

<a name="specifying-the-resource-model"></a>
#### 指定資源模型

如果您正在使用[路由模型綁定](/docs/{{version}}/routing#route-model-binding)並希望資源控制器的方法型別提示模型實例，您可以在產生控制器時使用 `--model` 選項：

```shell
php artisan make:controller PhotoController --model=Photo --resource
```

<a name="generating-form-requests"></a>
#### 產生表單請求

您可以在產生資源控制器時提供 `--requests` 選項，指示 Artisan 為控制器的儲存和更新方法產生[表單請求類別](/docs/{{version}}/validation#form-request-validation)：

```shell
php artisan make:controller PhotoController --model=Photo --resource --requests
```

<a name="restful-partial-resource-routes"></a>
### 部分資源路由

宣告資源路由時，您可以指定控制器應處理的動作子集，而不是完整的預設動作集：

```php
use App\Http\Controllers\PhotoController;

Route::resource('photos', PhotoController::class)->only([
    'index', 'show'
]);

Route::resource('photos', PhotoController::class)->except([
    'create', 'store', 'update', 'destroy'
]);
```

<a name="api-resource-routes"></a>
#### API 資源路由

宣告將由 API 消費的資源路由時，您通常會希望排除呈現 HTML 範本的路由，例如 `create` 和 `edit`。為方便起見，您可以使用 `apiResource` 方法自動排除這兩個路由：

```php
use App\Http\Controllers\PhotoController;

Route::apiResource('photos', PhotoController::class);
```

您可以透過將陣列傳遞給 `apiResources` 方法來一次註冊多個 API 資源控制器：

```php
use App\Http\Controllers\PhotoController;
use App\Http\Controllers\PostController;

Route::apiResources([
    'photos' => PhotoController::class,
    'posts' => PostController::class,
]);
```

若要快速產生不包含 `create` 或 `edit` 方法的 API 資源控制器，請在執行 `make:controller` 命令時使用 `--api` 開關：

```shell
php artisan make:controller PhotoController --api
```

<a name="restful-nested-resources"></a>
### 巢狀資源

有時您可能需要定義指向巢狀資源的路由。例如，照片資源可能有多個可以附加到照片的評論。若要巢狀化資源控制器，您可以在路由宣告中使用「點」表示法：

```php
use App\Http\Controllers\PhotoCommentController;

Route::resource('photos.comments', PhotoCommentController::class);
```

此路由將註冊一個巢狀資源，可以使用以下 URI 存取：

```text
/photos/{photo}/comments/{comment}
```

<a name="scoping-nested-resources"></a>
#### 範圍化巢狀資源

Laravel 的[隱式模型綁定](/docs/{{version}}/routing#implicit-model-binding-scoping)功能可以自動範圍化巢狀綁定，以確認解析的子模型屬於父模型。透過在定義巢狀資源時使用 `scoped` 方法，您可以啟用自動範圍化，並指示 Laravel 子資源應透過哪個欄位檢索。有關如何實現此目的的更多資訊，請參閱[範圍化資源路由](#restful-scoping-resource-routes)的說明文件。

<a name="shallow-nesting"></a>
#### 淺層巢狀

通常，URI 中同時包含父 ID 和子 ID 並非完全必要，因為子 ID 已經是唯一的識別碼。當使用唯一識別碼（例如自動遞增主鍵）來識別 URI 片段中的模型時，您可以選擇使用「淺層巢狀」：

```php
use App\Http\Controllers\CommentController;

Route::resource('photos.comments', CommentController::class)->shallow();
```

此路由定義將定義以下路由：

<div class="overflow-auto">

| 動詞      | URI                               | 動作    | 路由名稱               |
| --------- | --------------------------------- | ------- | ---------------------- |
| GET       | `/photos/{photo}/comments`        | index   | photos.comments.index  |
| GET       | `/photos/{photo}/comments/create` | create  | photos.comments.create |
| POST      | `/photos/{photo}/comments`        | store   | photos.comments.store  |
| GET       | `/comments/{comment}`             | show    | comments.show          |
| GET       | `/comments/{comment}/edit`        | edit    | comments.edit          |
| PUT/PATCH | `/comments/{comment}`             | update  | comments.update        |
| DELETE    | `/comments/{comment}`             | destroy | comments.destroy       |

</div>

<a name="restful-naming-resource-routes"></a>
### 命名資源路由

預設情況下，所有資源控制器動作都有一個路由名稱；但是，您可以透過傳遞一個包含所需路由名稱的 `names` 陣列來覆寫這些名稱：

```php
use App\Http\Controllers\PhotoController;

Route::resource('photos', PhotoController::class)->names([
    'create' => 'photos.build'
]);
```

<a name="restful-naming-resource-route-parameters"></a>
### 命名資源路由參數

預設情況下，`Route::resource` 將根據資源名稱的「單數化」版本為您的資源路由建立路由參數。您可以使用 `parameters` 方法輕鬆地針對每個資源覆寫此設定。傳遞給 `parameters` 方法的陣列應該是資源名稱和參數名稱的關聯陣列：

```php
use App\Http\Controllers\AdminUserController;

Route::resource('users', AdminUserController::class)->parameters([
    'users' => 'admin_user'
]);
```

上面的範例為資源的 `show` 路由產生以下 URI：

```text
/users/{admin_user}
```

<a name="restful-scoping-resource-routes"></a>
### 範圍化資源路由

Laravel 的[範圍化隱式模型綁定](/docs/{{version}}/routing#implicit-model-binding-scoping)功能可以自動範圍化巢狀綁定，以確認解析的子模型屬於父模型。透過在定義巢狀資源時使用 `scoped` 方法，您可以啟用自動範圍化，並指示 Laravel 子資源應透過哪個欄位檢索：

```php
use App\Http\Controllers\PhotoCommentController;

Route::resource('photos.comments', PhotoCommentController::class)->scoped([
    'comment' => 'slug',
]);
```

此路由將註冊一個範圍化的巢狀資源，可以使用以下 URI 存取：

```text
/photos/{photo}/comments/{comment:slug}
```

當使用自訂鍵的隱式綁定作為巢狀路由參數時，Laravel 將自動範圍化查詢，透過其父級檢索巢狀模型，並使用慣例來猜測父級上的關係名稱。在此情況下，將假定 `Photo` 模型具有名為 `comments` (路由參數名稱的複數) 的關係，可用於檢索 `Comment` 模型。

<a name="restful-localizing-resource-uris"></a>
### 本地化資源 URI

預設情況下，`Route::resource` 將使用英文動詞和複數規則建立資源 URI。如果您需要本地化 `create` 和 `edit` 動作動詞，您可以使用 `Route::resourceVerbs` 方法。這可以在應用程式的 `App\Providers\AppServiceProvider` 中的 `boot` 方法開頭完成：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Route::resourceVerbs([
        'create' => 'crear',
        'edit' => 'editar',
    ]);
}
```

Laravel 的複數化器支援[多種不同的語言，您可以根據需要進行設定](/docs/{{version}}/localization#pluralization-language)。一旦動詞和複數化語言被自訂，像 `Route::resource('publicacion', PublicacionController::class)` 這樣的資源路由註冊將產生以下 URI：

```text
/publicacion/crear

/publicacion/{publicaciones}/editar
```

<a name="restful-supplementing-resource-controllers"></a>
### 補充資源控制器

如果您需要為資源控制器添加預設資源路由集之外的其他路由，您應該在呼叫 `Route::resource` 方法之前定義這些路由；否則，`resource` 方法定義的路由可能會無意中優先於您的補充路由：

```php
use App\Http\Controller\PhotoController;

Route::get('/photos/popular', [PhotoController::class, 'popular']);
Route::resource('photos', PhotoController::class);
```

> [!NOTE]
> 請記住讓您的控制器保持專注。如果您發現自己經常需要典型資源動作集之外的方法，請考慮將控制器拆分為兩個較小的控制器。

<a name="singleton-resource-controllers"></a>
### 單例資源控制器

有時，您的應用程式將擁有只能有一個實例的資源。例如，使用者的「個人資料」可以編輯或更新，但使用者不能擁有多個「個人資料」。同樣，圖像可能只有一個「縮圖」。這些資源稱為「單例資源」，表示資源只能存在一個實例。在這些情況下，您可以註冊一個「單例」資源控制器：

```php
use App\Http\Controllers\ProfileController;
use Illuminate\Support\Facades\Route;

Route::singleton('profile', ProfileController::class);
```

上述單例資源定義將註冊以下路由。如您所見，單例資源不會註冊「建立」路由，並且註冊的路由不接受識別碼，因為資源只能存在一個實例：

<div class="overflow-auto">

| 動詞      | URI             | 動作   | 路由名稱       |
| --------- | --------------- | ------ | -------------- |
| GET       | `/profile`      | show   | profile.show   |
| GET       | `/profile/edit` | edit   | profile.edit   |
| PUT/PATCH | `/profile`      | update | profile.update |

</div>

單例資源也可以巢狀在標準資源中：

```php
Route::singleton('photos.thumbnail', ThumbnailController::class);
```

在此範例中，`photos` 資源將接收所有[標準資源路由](#actions-handled-by-resource-controllers)；但是，`thumbnail` 資源將是具有以下路由的單例資源：

<div class="overflow-auto">

| 動詞      | URI                              | 動作   | 路由名稱                |
| --------- | -------------------------------- | ------ | ----------------------- |
| GET       | `/photos/{photo}/thumbnail`      | show   | photos.thumbnail.show   |
| GET       | `/photos/{photo}/thumbnail/edit` | edit   | photos.thumbnail.edit   |
| PUT/PATCH | `/photos/{photo}/thumbnail`      | update | photos.thumbnail.update |

</div>

<a name="creatable-singleton-resources"></a>
#### 可建立的單例資源

有時，您可能希望為單例資源定義建立和儲存路由。為此，您可以在註冊單例資源路由時呼叫 `creatable` 方法：

```php
Route::singleton('photos.thumbnail', ThumbnailController::class)->creatable();
```

在此範例中，將註冊以下路由。如您所見，可建立的單例資源也將註冊 `DELETE` 路由：

<div class="overflow-auto">

| 動詞      | URI                                | 動作    | 路由名稱                 |
| --------- | ---------------------------------- | ------- | ------------------------ |
| GET       | `/photos/{photo}/thumbnail/create` | create  | photos.thumbnail.create  |
| POST      | `/photos/{photo}/thumbnail`        | store   | photos.thumbnail.store   |
| GET       | `/photos/{photo}/thumbnail`        | show    | photos.thumbnail.show    |
| GET       | `/photos/{photo}/thumbnail/edit`   | edit    | photos.thumbnail.edit    |
| PUT/PATCH | `/photos/{photo}/thumbnail`        | update  | photos.thumbnail.update  |
| DELETE    | `/photos/{photo}/thumbnail`        | destroy | photos.thumbnail.destroy |

</div>

如果您希望 Laravel 為單例資源註冊 `DELETE` 路由，但不註冊建立或儲存路由，您可以使用 `destroyable` 方法：

```php
Route::singleton(...)->destroyable();
```

<a name="api-singleton-resources"></a>
#### API 單例資源

`apiSingleton` 方法可用於註冊將透過 API 操作的單例資源，因此 `create` 和 `edit` 路由變得不必要：

```php
Route::apiSingleton('profile', ProfileController::class);
```

當然，API 單例資源也可以是 `creatable`，這將為資源註冊 `store` 和 `destroy` 路由：

```php
Route::apiSingleton('photos.thumbnail', ProfileController::class)->creatable();
```
<a name="middleware-and-resource-controllers"></a>
### Middleware 與資源控制器

Laravel 允許您使用 `middleware`、`middlewareFor` 和 `withoutMiddlewareFor` 方法將 Middleware 指派給資源路由的所有方法或僅特定方法。這些方法提供了對每個資源動作套用哪些 Middleware 的精細控制。

#### 將 Middleware 套用至所有方法

您可以使用 `middleware` 方法將 Middleware 指派給資源或單例資源路由產生的所有路由：

```php
Route::resource('users', UserController::class)
    ->middleware(['auth', 'verified']);

Route::singleton('profile', ProfileController::class)
    ->middleware('auth');
```

#### 將 Middleware 套用至特定方法

您可以使用 `middlewareFor` 方法將 Middleware 指派給給定資源控制器的一個或多個特定方法：

```php
Route::resource('users', UserController::class)
    ->middlewareFor('show', 'auth');

Route::apiResource('users', UserController::class)
    ->middlewareFor(['show', 'update'], 'auth');

Route::resource('users', UserController::class)
    ->middlewareFor('show', 'auth')
    ->middlewareFor('update', 'auth');

Route::apiResource('users', UserController::class)
    ->middlewareFor(['show', 'update'], ['auth', 'verified']);
```

`middlewareFor` 方法也可以與單例和 API 單例資源控制器結合使用：

```php
Route::singleton('profile', ProfileController::class)
    ->middlewareFor('show', 'auth');

Route::apiSingleton('profile', ProfileController::class)
    ->middlewareFor(['show', 'update'], 'auth');
```

#### 從特定方法中排除 Middleware

您可以使用 `withoutMiddlewareFor` 方法從資源控制器的特定方法中排除 Middleware：

```php
Route::middleware(['auth', 'verified', 'subscribed'])->group(function () {
    Route::resource('users', UserController::class)
        ->withoutMiddlewareFor('index', ['auth', 'verified'])
        ->withoutMiddlewareFor(['create', 'store'], 'verified')
        ->withoutMiddlewareFor('destroy', 'subscribed');
});
```

<a name="dependency-injection-and-controllers"></a>
## 依賴注入與控制器

<a name="constructor-injection"></a>
#### 建構子注入

Laravel [服務容器](/docs/{{version}}/container)用於解析所有 Laravel 控制器。因此，您可以在控制器的建構子中型別提示控制器可能需要的任何依賴。宣告的依賴將自動解析並注入到控制器實例中：

```php
<?php

namespace App\Http\Controllers;

use App\Repositories\UserRepository;

class UserController extends Controller
{
    /**
     * Create a new controller instance.
     */
    public function __construct(
        protected UserRepository $users,
    ) {}
}
```

<a name="method-injection"></a>
#### 方法注入

除了建構子注入之外，您還可以在控制器的方法上型別提示依賴。方法注入的一個常見用例是將 `Illuminate\Http\Request` 實例注入到控制器方法中：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class UserController extends Controller
{
    /**
     * Store a new user.
     */
    public function store(Request $request): RedirectResponse
    {
        $name = $request->name;

        // Store the user...

        return redirect('/users');
    }
}
```

如果您的控制器方法也期望來自路由參數的輸入，請在其他依賴之後列出您的路由參數。例如，如果您的路由定義如下：

```php
use App\Http\Controllers\UserController;

Route::put('/user/{id}', [UserController::class, 'update']);
```

您仍然可以型別提示 `Illuminate\Http\Request` 並透過以下方式定義控制器方法來存取您的 `id` 參數：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class UserController extends Controller
{
    /**
     * Update the given user.
     */
    public function update(Request $request, string $id): RedirectResponse
    {
        // Update the user...

        return redirect('/users');
    }
}
```

