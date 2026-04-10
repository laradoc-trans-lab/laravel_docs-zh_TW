# 控制器

- [簡介](#introduction)
- [撰寫控制器](#writing-controllers)
    - [基礎控制器](#basic-controllers)
    - [單一動作控制器](#single-action-controllers)
- [控制器中介層](#controller-middleware)
    - [中介層屬性](#middleware-attributes)
    - [授權屬性](#authorization-attributes)
- [資源控制器](#resource-controllers)
    - [部分資源路由](#restful-partial-resource-routes)
    - [巢狀資源](#restful-nested-resources)
    - [命名資源路由](#restful-naming-resource-routes)
    - [命名資源路由參數](#restful-naming-resource-route-parameters)
    - [資源路由範圍限定](#restful-scoping-resource-routes)
    - [資源 URI 本地化](#restful-localizing-resource-uris)
    - [補充資源控制器](#restful-supplementing-resource-controllers)
    - [單例資源控制器](#singleton-resource-controllers)
    - [中介層與資源控制器](#middleware-and-resource-controllers)
- [依賴注入與控制器](#dependency-injection-and-controllers)

<a name="introduction"></a>
## 簡介

與其在路由文件中將所有請求處理邏輯定義為閉包，您可能希望使用「控制器 (controller)」類別來組織這些行為。控制器能將相關的請求處理邏輯分組到單一類別中。例如，一個 `UserController` 類別可能會處理所有與使用者相關的傳入請求，包括顯示、建立、更新和刪除使用者。預設情況下，控制器儲存在 `app/Http/Controllers` 目錄中。


<a name="writing-controllers"></a>
## 撰寫控制器


<a name="basic-controllers"></a>
### 基礎控制器

若要快速產生一個新控制器，您可以執行 `make:controller` Artisan 指令。預設情況下，應用程式的所有控制器都儲存在 `app/Http/Controllers` 目錄中：

```shell
php artisan make:controller UserController
```

讓我們來看看一個基礎控制器的範例。控制器可以擁有任意數量的公開方法來回應傳入的 HTTP 請求：

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

一旦您撰寫了控制器類別與方法，您可以像這樣定義指向該控制器方法的路由：

```php
use App\Http\Controllers\UserController;

Route::get('/user/{id}', [UserController::class, 'show']);
```

當傳入的請求符合指定的路由 URI 時，`App\Http\Controllers\UserController` 類別中的 `show` 方法將被調用，且路由參數將被傳遞給該方法。

> [!NOTE]
> 控制器**並不要求**必須繼承一個基礎類別。然而，有時繼承一個包含所有控制器共用方法的基礎控制器類別會比較方便。


<a name="single-action-controllers"></a>
### 單一動作控制器

如果某個控制器動作特別複雜，您可能會發現將整個控制器類別專用於該單一動作會比較方便。為了實現這一點，您可以在控制器中定義單個 `__invoke` 方法：

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

為單一動作控制器註冊路由時，您不需要指定控制器方法。相反地，您只需將控制器的名稱傳遞給路由器即可：

```php
use App\Http\Controllers\ProvisionServer;

Route::post('/server', ProvisionServer::class);
```

您可以使用 `make:controller` Artisan 指令的 `--invokable` 選項來產生一個可調用 (invokable) 的控制器：

```shell
php artisan make:controller ProvisionServer --invokable
```

> [!NOTE]
> 控制器存根 (stubs) 可以使用 [stub 發布](/docs/{{version}}/artisan#stub-customization) 進行自訂。


<a name="controller-middleware"></a>
## 控制器中介層

[中介層](/docs/{{version}}/middleware) 可以分配給路由文件中的控制器路由：

```php
Route::get('/profile', [UserController::class, 'show'])->middleware('auth');
```

或者，您可能會發現直接在控制器類別中指定中介層比較方便。若要這樣做，您的控制器應該實作 `HasMiddleware` 介面，該介面規定控制器必須有一個靜態的 `middleware` 方法。在此方法中，您可以返回一個應應用於控制器動作的中介層陣列：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Routing\Controllers\HasMiddleware;
use Illuminate\Routing\Controllers\Middleware;

class UserController implements HasMiddleware
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


<a name="middleware-attributes"></a>
### 中介層屬性

您也可以使用 PHP 屬性將中介層分配給控制器：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Routing\Attributes\Controllers\Middleware;

#[Middleware('auth')]
#[Middleware('log', only: ['index'])]
#[Middleware('subscribed', except: ['store'])]
class UserController
{
    // ...
}
```

您也可以將中介層屬性放置在單個控制器方法上。分配給方法的中介層將與類別層級分配的中介層合併：

```php
<?php

namespace App\Http\Controllers;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Routing\Attributes\Controllers\Middleware;

#[Middleware('auth')]
class UserController
{
    #[Middleware('log')]
    #[Middleware('subscribed')]
    public function index()
    {
        // ...
    }

    #[Middleware(static function (Request $request, Closure $next) {
        // ...

        return $next($request);
    })]
    public function store()
    {
        // ...
    }
}
```


<a name="authorization-attributes"></a>
### 授權屬性

如果您透過策略 (policies) 來對控制器動作進行授權，您可以使用 `Authorize` 屬性作為 `can` 中介層的便捷快捷方式：

```php
<?php

namespace App\Http\Controllers;

use App\Models\Comment;
use App\Models\Post;
use Illuminate\Routing\Attributes\Controllers\Authorize;

class CommentController
{
    #[Authorize('create', [Comment::class, 'post'])]
    public function store(Post $post)
    {
        // ...
    }

    #[Authorize('delete', 'comment')]
    public function destroy(Comment $comment)
    {
        // ...
    }
}
```

第一個引數是您希望授權的能力。第二個引數是應傳遞給策略的模型類別、路由參數或多個參數。

<a name="resource-controllers"></a>
## 資源控制器

如果您將應用程式中的每個 Eloquent 模型視為一個「資源」，那麼針對應用程式中的每個資源執行相同的一組動作是很常見的。例如，假設您的應用程式包含一個 `Photo` 模型和一個 `Movie` 模型，使用者很可能需要建立、讀取、更新或刪除這些資源。

由於這種常見的使用情境，Laravel 的資源路由僅需一行程式碼，就能將典型的建立、讀取、更新和刪除 (CRUD) 路由分配給一個控制器。若要開始使用，我們可以使用 `make:controller` Artisan 指令的 `--resource` 選項來快速建立一個處理這些動作的控制器：

```shell
php artisan make:controller PhotoController --resource
```

此指令將在 `app/Http/Controllers/PhotoController.php` 產生一個控制器。該控制器將為每個可用的資源操作包含一個方法。接下來，您可以註冊一個指向該控制器的資源路由：

```php
use App\Http\Controllers\PhotoController;

Route::resource('photos', PhotoController::class);
```

這單一的路由宣告會建立多個路由，用以處理該資源上的各種動作。產生的控制器將已經為每個動作預先定義好方法。請記得，您隨時可以透過執行 `route:list` Artisan 指令來快速概覽應用程式的路由。

您甚至可以透過將陣列傳遞給 `resources` 方法，一次註冊多個資源控制器：

```php
Route::resources([
    'photos' => PhotoController::class,
    'posts' => PostController::class,
]);
```

`softDeletableResources` 方法會註冊多個都使用 `withTrashed` 方法的資源控制器：

```php
Route::softDeletableResources([
    'photos' => PhotoController::class,
    'posts' => PostController::class,
]);
```


<a name="actions-handled-by-resource-controllers"></a>
#### 資源控制器處理的動作

<div class="overflow-auto">

| 動詞      | URI                    | 動作    | 路由名稱      |
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
#### 自訂缺失模型的行為

通常，如果找不到隱式綁定的資源模型，將會產生 404 HTTP 回應。然而，您可以在定義資源路由時呼叫 `missing` 方法來自訂此行為。`missing` 方法接受一個閉包，如果資源的任何路由找不到隱式綁定的模型，該閉包將被呼叫：

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

通常，隱式模型綁定不會檢索已[軟刪除](/docs/{{version}}/eloquent#soft-deleting)的模型，而是會返回 404 HTTP 回應。然而，您可以在定義資源路由時呼叫 `withTrashed` 方法，指示框架允許軟刪除的模型：

```php
use App\Http\Controllers\PhotoController;

Route::resource('photos', PhotoController::class)->withTrashed();
```

不帶引數地呼叫 `withTrashed` 將允許 `show`、`edit` 和 `update` 資源路由使用軟刪除模型。您可以透過將陣列傳遞給 `withTrashed` 方法來指定這些路由的子集：

```php
Route::resource('photos', PhotoController::class)->withTrashed(['show']);
```


<a name="specifying-the-resource-model"></a>
#### 指定資源模型

如果您正在使用[路由模型綁定](/docs/{{version}}/routing#route-model-binding)，並希望資源控制器的方法能型別提示模型實例，您可以在產生控制器時使用 `--model` 選項：

```shell
php artisan make:controller PhotoController --model=Photo --resource
```


<a name="generating-form-requests"></a>
#### 產生表單請求

在產生資源控制器時，您可以提供 `--requests` 選項，以指示 Artisan 為控制器的儲存和更新方法產生[表單請求(Form request)類別](/docs/{{version}}/validation#form-request-validation)：

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

在宣告將由 API 使用的資源路由時，您通常會想要排除呈現 HTML 模板的路由，例如 `create` 和 `edit`。為了方便起見，您可以使用 `apiResource` 方法來自動排除這兩個路由：

```php
use App\Http\Controllers\PhotoController;

Route::apiResource('photos', PhotoController::class);
```

您可以透過將陣列傳遞給 `apiResources` 方法，一次註冊多個 API 資源控制器：

```php
use App\Http\Controllers\PhotoController;
use App\Http\Controllers\PostController;

Route::apiResources([
    'photos' => PhotoController::class,
    'posts' => PostController::class,
]);
```

若要快速產生一個不包含 `create` 或 `edit` 方法的 API 資源控制器，請在執行 `make:controller` 指令時使用 `--api` 選項：

```shell
php artisan make:controller PhotoController --api
```

<a name="restful-nested-resources"></a>
### 巢狀資源

有時候您可能需要定義指向巢狀資源的路由。例如，一個照片資源可能有許多附加在該照片上的留言。要將資源控制器巢狀化，您可以在路由宣告中使用「點」符號表示法：

```php
use App\Http\Controllers\PhotoCommentController;

Route::resource('photos.comments', PhotoCommentController::class);
```

此路由將註冊一個巢狀資源，可以使用如下的 URI 進行訪問：

```text
/photos/{photo}/comments/{comment}
```


<a name="scoping-nested-resources"></a>
#### 巢狀資源範圍限定

Laravel 的 [隱含模型綁定](/docs/{{version}}/routing#implicit-model-binding-scoping) 功能可以自動對巢狀綁定進行範圍限定，以確保解析出的子模型確實屬於父模型。在定義巢狀資源時，透過使用 `scoped` 方法，您可以啟用自動範圍限定，並告知 Laravel 應使用哪個欄位來檢索子資源。關於如何實現此功能的更多資訊，請參閱 [資源路由範圍限定](#restful-scoping-resource-routes) 說明文件。


<a name="shallow-nesting"></a>
#### 淺層巢狀

通常情況下，在 URI 中不一定需要同時包含父級和子級的 ID，因為子級 ID 已經是一個唯一識別碼。當您在 URI 段落中使用自動遞增的主鍵等唯一識別碼來識別模型時，您可以選擇使用「淺層巢狀 (shallow nesting)」：

```php
use App\Http\Controllers\CommentController;

Route::resource('photos.comments', CommentController::class)->shallow();
```

此路由定義將定義以下路由：

<div class="overflow-auto">

| 動詞      | URI                               | 動作    | 路由名稱             |
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

預設情況下，所有資源控制器的動作都有一個路由名稱；然而，您可以透過傳遞一個包含您所需路由名稱的 `names` 陣列來覆寫這些名稱：

```php
use App\Http\Controllers\PhotoController;

Route::resource('photos', PhotoController::class)->names([
    'create' => 'photos.build'
]);
```


<a name="restful-naming-resource-route-parameters"></a>
### 命名資源路由參數

預設情況下，`Route::resource` 會根據資源名稱的「單數形式」來為您的資源路由建立路由參數。您可以使用 `parameters` 方法輕鬆地針對單個資源進行覆寫。傳遞給 `parameters` 方法的陣列應該是一個包含資源名稱與參數名稱的關聯陣列：

```php
use App\Http\Controllers\AdminUserController;

Route::resource('users', AdminUserController::class)->parameters([
    'users' => 'admin_user'
]);
```

上面的範例為資源的 `show` 路由產生了以下 URI：

```text
/users/{admin_user}
```


<a name="restful-scoping-resource-routes"></a>
### 資源路由範圍限定

Laravel 的 [範圍限定隱含模型綁定](/docs/{{version}}/routing#implicit-model-binding-scoping) 功能可以自動對巢狀綁定進行範圍限定，以確保解析出的子模型確實屬於父模型。在定義巢狀資源時，透過使用 `scoped` 方法，您可以啟用自動範圍限定，並告知 Laravel 應使用哪個欄位來檢索子資源：

```php
use App\Http\Controllers\PhotoCommentController;

Route::resource('photos.comments', PhotoCommentController::class)->scoped([
    'comment' => 'slug',
]);
```

此路由將註冊一個範圍限定的巢狀資源，可以使用如下的 URI 進行訪問：

```text
/photos/{photo}/comments/{comment:slug}
```

當使用自定義鍵值的隱含綁定作為巢狀路由參數時，Laravel 會根據慣例推測父模型上的關聯名稱，從而自動將查詢範圍限定在父模型之下以檢索巢狀模型。在此案例中，系統會假設 `Photo` 模型有一個名為 `comments`（路由參數名稱的複數形式）的關聯，可用於檢索 `Comment` 模型。


<a name="restful-localizing-resource-uris"></a>
### 資源 URI 本地化

預設情況下，`Route::resource` 會使用英文動詞和複數規則來建立資源 URI。如果您需要將 `create` 和 `edit` 的動作動詞本地化，可以使用 `Route::resourceVerbs` 方法。這可以在應用程式 `App\Providers\AppServiceProvider` 的 `boot` 方法開頭完成：

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

Laravel 的複數化工具支援 [幾種不同的語言，您可以根據需求進行配置](/docs/{{version}}/localization#pluralization-language)。一旦自定義了動詞和複數化語言，像 `Route::resource('publicacion', PublicacionController::class)` 這樣的資源路由註冊將會產生以下 URI：

```text
/publicacion/crear

/publicacion/{publicaciones}/editar
```


<a name="restful-supplementing-resource-controllers"></a>
### 補充資源控制器

如果您需要為資源控制器添加除預設資源路由集之外的額外路由，您應該在調用 `Route::resource` 方法之前定義這些路由；否則，由 `resource` 方法定義的路由可能會無意中優先於您的補充路由：

```php
use App\Http\Controller\PhotoController;

Route::get('/photos/popular', [PhotoController::class, 'popular']);
Route::resource('photos', PhotoController::class);
```

> [!NOTE]
> 請記得保持控制器的專注。如果您發現自己經常需要在典型的資源動作集之外定義方法，請考慮將您的控制器拆分為兩個較小的控制器。

<a name="singleton-resource-controllers"></a>
### 單例資源控制器

有時候，您的應用程式會有一些資源可能只有單一實例。例如，使用者的「個人設定 (profile)」可以被編輯或更新，但一個使用者不能擁有超過一個「個人設定」。同樣地，一張圖片可能只有一個「縮圖 (thumbnail)」。這些資源被稱為「單例資源 (singleton resources)」，意味著該資源只能存在一個且僅有一個實例。在這些情境下，您可以註冊一個「單例」資源控制器：

```php
use App\Http\Controllers\ProfileController;
use Illuminate\Support\Facades\Route;

Route::singleton('profile', ProfileController::class);
```

上述的單例資源定義將註冊以下路由。如您所見，單例資源不會註冊「建立 (creation)」路由，且註冊的路由不接受識別碼，因為該資源僅能存在一個實例：

<div class="overflow-auto">

| Verb      | URI             | Action | Route Name     |
| --------- | --------------- | ------ | -------------- |
| GET       | `/profile`      | show   | profile.show   |
| GET       | `/profile/edit` | edit   | profile.edit   |
| PUT/PATCH | `/profile`      | update | profile.update |

</div>

單例資源也可以巢狀在標準資源之中：

```php
Route::singleton('photos.thumbnail', ThumbnailController::class);
```

在此範例中，`photos` 資源將接收所有 [標準資源路由](#actions-handled-by-resource-controllers)；然而，`thumbnail` 資源將是一個單例資源，具有以下路由：

<div class="overflow-auto">

| Verb      | URI                              | Action | Route Name              |
| --------- | -------------------------------- | ------ | ----------------------- |
| GET       | `/photos/{photo}/thumbnail`      | show   | photos.thumbnail.show   |
| GET       | `/photos/{photo}/thumbnail/edit` | edit   | photos.thumbnail.edit   |
| PUT/PATCH | `/photos/{photo}/thumbnail`      | update | photos.thumbnail.update |

</div>


<a name="creatable-singleton-resources"></a>
#### 可建立的單例資源

有時候，您可能想要為單例資源定義建立和儲存路由。為了實現這一點，您可以在註冊單例資源路由時呼叫 `creatable` 方法：

```php
Route::singleton('photos.thumbnail', ThumbnailController::class)->creatable();
```

在此範例中，將註冊以下路由。如您所見，可建立的單例資源也會註冊一個 `DELETE` 路由：

<div class="overflow-auto">

| Verb      | URI                                | Action  | Route Name               |
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

`apiSingleton` 方法可用於註冊一個將透過 API 操作的單例資源，因此不需要 `create` 和 `edit` 路由：

```php
Route::apiSingleton('profile', ProfileController::class);
```

當然，API 單例資源也可以是 `creatable`，這將為該資源註冊 `store` 和 `destroy` 路由：

```php
Route::apiSingleton('photos.thumbnail', ProfileController::class)->creatable();
```

<a name="middleware-and-resource-controllers"></a>
### 中介層與資源控制器

Laravel 允許您使用 `middleware`、`middlewareFor` 和 `withoutMiddlewareFor` 方法，將中介層分配給資源路由的所有方法或僅特定方法。這些方法提供了對每個資源動作套用哪些中介層的精細控制。


#### 將中介層套用到所有方法

您可以使用 `middleware` 方法將中介層分配給由資源或單例資源路由所產生的所有路由：

```php
Route::resource('users', UserController::class)
    ->middleware(['auth', 'verified']);

Route::singleton('profile', ProfileController::class)
    ->middleware('auth');
```


#### 將中介層套用到特定方法

您可以使用 `middlewareFor` 方法將中介層分配給特定資源控制器的一個或多個特定方法：

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

`middlewareFor` 方法也可以與單例和 API 單例資源控制器配合使用：

```php
Route::singleton('profile', ProfileController::class)
    ->middlewareFor('show', 'auth');

Route::apiSingleton('profile', ProfileController::class)
    ->middlewareFor(['show', 'update'], 'auth');
```


#### 從特定方法排除中介層

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

Laravel 的 [服務容器 (service container)](/docs/{{version}}/container) 被用來解析所有的 Laravel 控制器。因此，您可以在控制器的建構子中，透過型別提示 (type-hint) 指定控制器可能需要的任何依賴項目。宣告的依賴項目將會被自動解析並注入到控制器實例中：

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

除了建構子注入之外，您也可以在控制器的方法中指定依賴項目的型別提示。方法注入的一個常見使用場景，就是將 `Illuminate\Http\Request` 實例注入到您的控制器方法中：

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

如果您的控制器方法還需要接收來自路由參數的輸入，請將路由引數放在其他依賴項目之後。例如，如果您的路由定義如下：

```php
use App\Http\Controllers\UserController;

Route::put('/user/{id}', [UserController::class, 'update']);
```

您仍然可以透過如下定義控制器方法，來型別提示 `Illuminate\Http\Request` 並存取您的 `id` 參數：

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