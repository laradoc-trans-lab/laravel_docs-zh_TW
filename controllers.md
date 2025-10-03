# 控制器

- [簡介](#introduction)
- [撰寫控制器](#writing-controllers)
    - [基本控制器](#basic-controllers)
    - [單一動作控制器](#single-action-controllers)
- [控制器中介層](#controller-middleware)
- [資源控制器](#resource-controllers)
    - [部分資源路由](#restful-partial-resource-routes)
    - [巢狀資源](#restful-nested-resources)
    - [命名資源路由](#restful-naming-resource-routes)
    - [命名資源路由參數](#restful-naming-resource-route-parameters)
    - [範圍化資源路由](#restful-scoping-resource-routes)
    - [在地化資源 URI](#restful-localizing-resource-uris)
    - [補充資源控制器](#restful-supplementing-resource-controllers)
    - [單例資源控制器](#singleton-resource-controllers)
    - [中介層與資源控制器](#middleware-and-resource-controllers)
- [依賴注入與控制器](#dependency-injection-and-controllers)

<a name="introduction"></a>
## 簡介

與其將所有請求處理邏輯定義為路由檔案中的閉包，您可能會希望使用「控制器」類別來組織這些行為。控制器可以將相關的請求處理邏輯分組到單一類別中。舉例來說，`UserController` 類別可能處理所有與使用者相關的傳入請求，包含顯示、建立、更新和刪除使用者。依預設，控制器儲存於 `app/Http/Controllers` 目錄中。

<a name="writing-controllers"></a>
## 撰寫控制器

<a name="basic-controllers"></a>
### 基本控制器

若要快速產生新的控制器，您可以執行 `make:controller` Artisan 指令。依預設，應用程式的所有控制器都儲存於 `app/Http/Controllers` 目錄中：

```shell
php artisan make:controller UserController
```

我們來看一個基本控制器的範例。控制器可以有任意數量的公用方法，這些方法將回應傳入的 HTTP 請求：

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

一旦您撰寫了控制器類別和方法，您可以像這樣為該控制器方法定義路由：

```php
use App\Http\Controllers\UserController;

Route::get('/user/{id}', [UserController::class, 'show']);
```

當傳入請求符合指定的路由 URI 時，`App\Http\Controllers\UserController` 類別上的 `show` 方法將會被呼叫，並且路由參數會傳遞給該方法。

> [!NOTE]
> 控制器**不一定**要繼承基礎類別。不過，繼承一個包含應該在所有控制器中共享的方法的基礎控制器類別，有時會很方便。

<a name="single-action-controllers"></a>
### 單一動作控制器

如果控制器動作特別複雜，您可能會覺得將整個控制器類別專用於該單一動作會更方便。為此，您可以在控制器中定義一個單一的 `__invoke` 方法：

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

當註冊單一動作控制器的路由時，您不需要指定控制器方法。相反地，您只需將控制器的名稱傳遞給路由器即可：

```php
use App\Http\Controllers\ProvisionServer;

Route::post('/server', ProvisionServer::class);
```

您可以透過使用 `make:controller` Artisan 指令的 `--invokable` 選項來產生可呼叫的控制器：

```shell
php artisan make:controller ProvisionServer --invokable
```

> [!NOTE]
> 控制器 Stub 可透過 [Stub 發佈](/docs/{{version}}/artisan#stub-customization)進行客製化。

<a name="controller-middleware"></a>
## 控制器中介層

[中介層](/docs/{{version}}/middleware)可以分配給路由檔案中的控制器路由：

```php
Route::get('/profile', [UserController::class, 'show'])->middleware('auth');
```

或者，您可能會覺得在控制器類別中指定中介層很方便。為此，您的控制器應該實作 `HasMiddleware` 介面，這規定了控制器應該有一個靜態的 `middleware` 方法。從這個方法中，您可以回傳一個應該應用於控制器動作的中介層陣列：

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

您也可以將控制器中介層定義為閉包，這提供了一種方便的方式來定義行內中介層，而無需撰寫整個中介層類別：

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

如果您將應用程式中的每個 Eloquent 模型視為一個「資源」，通常會對應用程式中的每個資源執行相同的動作集。例如，假設您的應用程式包含 `Photo` 模型和 `Movie` 模型。使用者很可能可以建立、讀取、更新或刪除這些資源。

由於這種常見的使用情境，Laravel 資源路由會使用單行程式碼將典型的建立、讀取、更新和刪除（「CRUD」）路由指派給控制器。首先，我們可以使用 `make:controller` Artisan 命令的 `--resource` 選項來快速建立一個控制器以處理這些動作：

```shell
php artisan make:controller PhotoController --resource
```

此命令將在 `app/Http/Controllers/PhotoController.php` 產生一個控制器。該控制器將包含每個可用資源操作的方法。接下來，您可以註冊一個指向該控制器的資源路由：

```php
use App\Http\Controllers\PhotoController;

Route::resource('photos', PhotoController::class);
```

這個單一的路由宣告會建立多個路由來處理資源上的各種動作。產生的控制器將已為這些動作預留方法。請記住，您隨時可以執行 `route:list` Artisan 命令來快速概覽應用程式的路由。

您甚至可以透過向 `resources` 方法傳遞陣列來一次註冊多個資源控制器：

```php
Route::resources([
    'photos' => PhotoController::class,
    'posts' => PostController::class,
]);
```

`softDeletableResources` 方法會註冊多個資源控制器，這些控制器都使用 `withTrashed` 方法：

```php
Route::softDeletableResources([
    'photos' => PhotoController::class,
    'posts' => PostController::class,
]);
```


<a name="actions-handled-by-resource-controllers"></a>
#### 資源控制器處理的動作

<div class="overflow-auto">

| 動詞      | URI                    | 動作  | 路由名稱     |
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
#### 自訂模型遺失行為

通常，如果找不到隱式繫結的資源模型，將會產生 404 HTTP 回應。然而，您可以在定義資源路由時呼叫 `missing` 方法來自訂此行為。`missing` 方法接受一個閉包，如果資源的任何路由找不到隱式繫結的模型，此閉包將會被呼叫：

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

通常，隱式模型繫結不會檢索已[軟刪除](/docs/{{version}}/eloquent#soft-deleting)的模型，而是會回傳 404 HTTP 回應。然而，您可以在定義資源路由時呼叫 `withTrashed` 方法，指示框架允許軟刪除的模型：

```php
use App\Http\Controllers\PhotoController;

Route::resource('photos', PhotoController::class)->withTrashed();
```

不帶任何參數呼叫 `withTrashed` 將允許 `show`、`edit` 和 `update` 資源路由的軟刪除模型。您可以透過向 `withTrashed` 方法傳遞陣列來指定這些路由的一個子集：

```php
Route::resource('photos', PhotoController::class)->withTrashed(['show']);
```


<a name="specifying-the-resource-model"></a>
#### 指定資源模型

如果您正在使用[路由模型繫結](/docs/{{version}}/routing#route-model-binding)，並希望資源控制器的方法能類型提示一個模型實例，您可以在產生控制器時使用 `--model` 選項：

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

在宣告資源路由時，您可以指定控制器應處理的動作子集，而不是完整的預設動作集：

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

宣告將由 API 消耗的資源路由時，您通常會想要排除呈現 HTML 模板的路由，例如 `create` 和 `edit`。為了方便起見，您可以使用 `apiResource` 方法自動排除這兩個路由：

```php
use App\Http\Controllers\PhotoController;

Route::apiResource('photos', PhotoController::class);
```

您可以透過向 `apiResources` 方法傳遞陣列來一次註冊多個 API 資源控制器：

```php
use App\Http\Controllers\PhotoController;
use App\Http\Controllers\PostController;

Route::apiResources([
    'photos' => PhotoController::class,
    'posts' => PostController::class,
]);
```

若要快速產生一個不包含 `create` 或 `edit` 方法的 API 資源控制器，請在執行 `make:controller` 命令時使用 `--api` 開關：

```shell
php artisan make:controller PhotoController --api
```

<a name="restful-nested-resources"></a>
### 巢狀資源

有時您可能需要為巢狀資源定義路由。例如，一個相片資源可能有多個評論，這些評論可以附加到相片。要巢狀化資源控制器，您可以在路由宣告中使用「點符號 (dot notation)」：

```php
use App\Http\Controllers\PhotoCommentController;

Route::resource('photos.comments', PhotoCommentController::class);
```

此路由將註冊一個巢狀資源，可透過以下 URI 存取：

```text
/photos/{photo}/comments/{comment}
```

<a name="scoping-nested-resources"></a>
#### 範圍化巢狀資源

Laravel 的[隱式模型綁定](/docs/{{version}}/routing#implicit-model-binding-scoping)功能可以自動範圍化巢狀綁定，以確保解析的子模型屬於父模型。透過在定義巢狀資源時使用 `scoped` 方法，您可以啟用自動範圍化，並指示 Laravel 應透過哪個欄位來檢索子資源。有關如何實現此目的的更多資訊，請參閱[範圍化資源路由](#restful-scoping-resource-routes)的說明文件。

<a name="shallow-nesting"></a>
#### 淺層巢狀

通常，URI 中同時包含父 ID 和子 ID 並非完全必要，因為子 ID 本身就是一個唯一識別碼。當使用唯一識別碼（例如自動遞增主鍵）來識別 URI 片段中的模型時，您可以選擇使用「淺層巢狀」：

```php
use App\Http\Controllers\CommentController;

Route::resource('photos.comments', CommentController::class)->shallow();
```

此路由定義將定義以下路由：

<div class="overflow-auto">

| 動詞      | URI                               | 動作  | 路由名稱             |
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

預設情況下，所有資源控制器動作都具有路由名稱；但是，您可以透過傳入一個包含您所需路由名稱的 `names` 陣列來覆寫這些名稱：

```php
use App\Http\Controllers\PhotoController;

Route::resource('photos', PhotoController::class)->names([
    'create' => 'photos.build'
]);
```

<a name="restful-naming-resource-route-parameters"></a>
### 命名資源路由參數

預設情況下，`Route::resource` 將根據資源名稱的「單數化」版本來為您的資源路由建立路由參數。您可以透過 `parameters` 方法，輕鬆地針對每個資源覆寫此設定。傳入 `parameters` 方法的陣列應該是資源名稱和參數名稱的關聯式陣列：

```php
use App\Http\Controllers\AdminUserController;

Route::resource('users', AdminUserController::class)->parameters([
    'users' => 'admin_user'
]);
```

上述範例為資源的 `show` 路由產生以下 URI：

```text
/users/{admin_user}
```

<a name="restful-scoping-resource-routes"></a>
### 範圍化資源路由

Laravel 的[範圍化隱式模型綁定](/docs/{{version}}/routing#implicit-model-binding-scoping)功能可以自動範圍化巢狀綁定，以確保解析的子模型屬於父模型。透過在定義巢狀資源時使用 `scoped` 方法，您可以啟用自動範圍化，並指示 Laravel 應透過哪個欄位來檢索子資源：

```php
use App\Http\Controllers\PhotoCommentController;

Route::resource('photos.comments', PhotoCommentController::class)->scoped([
    'comment' => 'slug',
]);
```

此路由將註冊一個範圍化的巢狀資源，可透過以下 URI 存取：

```text
/photos/{photo}/comments/{comment:slug}
```

當使用自訂鍵名的隱式綁定作為巢狀路由參數時，Laravel 將自動範圍化查詢，以透過其父級檢索巢狀模型，並使用慣例來猜測父級上的關聯名稱。在此情況下，將假定 `Photo` 模型具有一個名為 `comments` (路由參數名稱的複數形式) 的關聯，該關聯可用於檢索 `Comment` 模型。

<a name="restful-localizing-resource-uris"></a>
### 在地化資源 URI

預設情況下，`Route::resource` 將使用英文動詞和複數規則建立資源 URI。如果您需要在地化 `create` 和 `edit` 動作動詞，您可以使用 `Route::resourceVerbs` 方法。這可以在應用程式的 `App\Providers\AppServiceProvider` 中，於 `boot` 方法的開頭處完成：

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

Laravel 的複數器支援[多種不同的語言，您可以根據需求進行配置](/docs/{{version}}/localization#pluralization-language)。一旦動詞和複數語言被自訂，像 `Route::resource('publicacion', PublicacionController::class)` 這樣的資源路由註冊將產生以下 URI：

```text
/publicacion/crear

/publicacion/{publicaciones}/editar
```

<a name="restful-supplementing-resource-controllers"></a>
### 補充資源控制器

如果您需要為資源控制器新增超出預設資源路由集合的額外路由，您應該在呼叫 `Route::resource` 方法之前定義這些路由；否則，`resource` 方法所定義的路由可能會無意中優先於您的補充路由：

```php
use App\Http\Controller\PhotoController;

Route::get('/photos/popular', [PhotoController::class, 'popular']);
Route::resource('photos', PhotoController::class);
```

> [!NOTE]
> 請記住讓您的控制器保持專注。如果您發現自己經常需要典型資源動作集之外的方法，請考慮將控制器拆分為兩個較小的控制器。

<a name="singleton-resource-controllers"></a>
### 單例資源控制器

有時候，您的應用程式會有一些資源，這些資源可能只會有單一實例。例如，使用者的「個人資料 (profile)」可以被編輯或更新，但一個使用者不能擁有多個「個人資料 (profile)」。同樣地，一張圖片可能會有單一的「縮圖 (thumbnail)」。這些資源被稱為「單例資源」，意指該資源只能存在一個且僅有一個實例。在這些情境下，您可以註冊一個「單例資源控制器」：

```php
use App\Http\Controllers\ProfileController;
use Illuminate\Support\Facades\Route;

Route::singleton('profile', ProfileController::class);
```

上述單例資源定義將會註冊以下路由。如您所見，「建立」路由不會為單例資源註冊，且已註冊的路由不接受識別碼，因為該資源只能存在一個實例：

<div class="overflow-auto">

| 動詞      | URI             | 動作   | 路由名稱       |
| --------- | --------------- | ------ | -------------- |
| GET       | `/profile`      | show   | profile.show   |
| GET       | `/profile/edit` | edit   | profile.edit   |
| PUT/PATCH | `/profile`      | update | profile.update |

</div>

單例資源也可以巢狀於標準資源中：

```php
Route::singleton('photos.thumbnail', ThumbnailController::class);
```

在此範例中，`photos` 資源將會收到所有 [標準資源路由](#actions-handled-by-resource-controllers)；然而，`thumbnail` 資源將會是一個單例資源，並具有以下路由：

<div class="overflow-auto">

| 動詞      | URI                              | 動作   | 路由名稱                |
| --------- | -------------------------------- | ------ | ----------------------- |
| GET       | `/photos/{photo}/thumbnail`      | show   | photos.thumbnail.show   |
| GET       | `/photos/{photo}/thumbnail/edit` | edit   | photos.thumbnail.edit   |
| PUT/PATCH | `/photos/{photo}/thumbnail`      | update | photos.thumbnail.update |

</div>


<a name="creatable-singleton-resources"></a>
#### 可建立的單例資源

有時，您可能希望為單例資源定義建立與儲存路由。為此，您可以在註冊單例資源路由時呼叫 `creatable` 方法：

```php
Route::singleton('photos.thumbnail', ThumbnailController::class)->creatable();
```

在此範例中，將會註冊以下路由。如您所見，`DELETE` 路由也將會為可建立的單例資源註冊：

<div class="overflow-auto">

| 動詞      | URI                                | 動作   | 路由名稱                 |
| --------- | ---------------------------------- | ------- | ------------------------ |
| GET       | `/photos/{photo}/thumbnail/create` | create  | photos.thumbnail.create  |
| POST      | `/photos/{photo}/thumbnail`        | store   | photos.thumbnail.store   |
| GET       | `/photos/{photo}/thumbnail`        | show    | photos.thumbnail.show    |
| GET       | `/photos/{photo}/thumbnail/edit`   | edit    | photos.thumbnail.edit    |
| PUT/PATCH | `/photos/{photo}/thumbnail`        | update  | photos.thumbnail.update  |
| DELETE    | `/photos/{photo}/thumbnail`        | destroy | photos.thumbnail.destroy |

</div>

如果您希望 Laravel 為單例資源註冊 `DELETE` 路由，但不註冊建立或儲存路由，您可以利用 `destroyable` 方法：

```php
Route::singleton(...)->destroyable();
```


<a name="api-singleton-resources"></a>
#### API 單例資源

`apiSingleton` 方法可用於註冊一個透過 API 進行操作的單例資源，因此 `create` 和 `edit` 路由將變得不必要：

```php
Route::apiSingleton('profile', ProfileController::class);
```

當然，API 單例資源也可以是 `creatable`，這將為該資源註冊 `store` 和 `destroy` 路由：

```php
Route::apiSingleton('photos.thumbnail', ProfileController::class)->creatable();
```

<a name="middleware-and-resource-controllers"></a>
### 中介層與資源控制器

Laravel 允許您使用 `middleware`、`middlewareFor` 和 `withoutMiddlewareFor` 方法，將中介層指派給所有或僅特定資源路由的方法。這些方法提供了對每個資源動作應用哪些中介層的細緻控制。


#### 將中介層應用於所有方法

您可以使用 `middleware` 方法將中介層指派給由資源或單例資源路由產生的所有路由：

```php
Route::resource('users', UserController::class)
    ->middleware(['auth', 'verified']);

Route::singleton('profile', ProfileController::class)
    ->middleware('auth');
```


#### 將中介層應用於特定方法

您可以使用 `middlewareFor` 方法將中介層指派給指定資源控制器的一個或多個特定方法：

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


#### 從特定方法中排除中介層

您可以使用 `withoutMiddlewareFor` 方法從資源控制器的特定方法中排除中介層：

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

Laravel 的 [服務容器](/docs/{{version}}/container) 用來解析所有 Laravel 控制器。因此，您可以在控制器的建構子中型別提示任何所需的依賴。這些宣告的依賴會被自動解析並注入到控制器實例中：

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

除了建構子注入之外，您也可以在控制器的方法上型別提示依賴。方法注入的一個常見用例是將 `Illuminate\Http\Request` 實例注入到您的控制器方法中：

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

如果您的控制器方法也預期從路由參數接收輸入，請在其他依賴之後列出您的路由引數。例如，如果您的路由定義如下：

```php
use App\Http\Controllers\UserController;

Route::put('/user/{id}', [UserController::class, 'update']);
```

您仍然可以透過以下方式定義控制器方法，來型別提示 `Illuminate\Http\Request` 並存取您的 `id` 參數：

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