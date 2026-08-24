# 控制器 (Controllers)

- [簡介](#introduction)
- [編寫控制器](#writing-controllers)
    - [基本控制器](#basic-controllers)
    - [單一動作控制器](#single-action-controllers)
- [控制器中介層](#controller-middleware)
    - [中介層屬性 (Attributes)](#middleware-attributes)
    - [授權屬性 (Attributes)](#authorization-attributes)
- [資源控制器](#resource-controllers)
    - [部分資源路由](#restful-partial-resource-routes)
    - [巢狀資源](#restful-nested-resources)
    - [命名資源路由](#restful-naming-resource-routes)
    - [命名資源路由參數](#restful-naming-resource-route-parameters)
    - [限定資源路由範圍](#restful-scoping-resource-routes)
    - [本地化資源 URI](#restful-localizing-resource-uris)
    - [擴充資源控制器](#restful-supplementing-resource-controllers)
    - [單例資源控制器](#singleton-resource-controllers)
    - [中介層與資源控制器](#middleware-and-resource-controllers)
- [依賴注入與控制器](#dependency-injection-and-controllers)

<a name="introduction"></a>
## 簡介

除了在路由檔案中將所有請求處理邏輯定義為閉包之外，您可能還希望使用「控制器 (Controller)」類別來組織此行為。控制器可以將相關的請求處理邏輯分組到單一類別中。例如，`UserController` 類別可以處理所有與使用者相關的傳入請求，包括顯示、建立、更新和刪除使用者。在預設情況下，控制器儲存在 `app/Http/Controllers` 目錄中。


<a name="writing-controllers"></a>
## 編寫控制器


<a name="basic-controllers"></a>
### 基本控制器

若要快速產生新的控制器，您可以執行 `make:controller` Artisan 指令。在預設情況下，應用程式的所有控制器都儲存在 `app/Http/Controllers` 目錄中：

```shell
php artisan make:controller UserController
```

讓我們來看一個基本控制器的範例。控制器可以包含任意數量的公開方法，用來回應傳入的 HTTP 請求：

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

編寫好控制器類別和方法後，您可以為該控制器方法定義一條路由，如下所示：

```php
use App\Http\Controllers\UserController;

Route::get('/user/{id}', [UserController::class, 'show']);
```

當傳入的請求與指定的路由 URI 相符時，`App\Http\Controllers\UserController` 類別上的 `show` 方法將會被調用，並將路由參數傳遞給該方法。

> [!NOTE]
> 控制器**並非必須**繼承基礎類別。然而，繼承一個包含所有控制器共享方法的基礎控制器類別，有時會非常方便。


<a name="single-action-controllers"></a>
### 單一動作控制器

如果某個控制器動作特別複雜，您可能會發現將整個控制器類別專門用於該單一動作會很方便。為此，您可以在控制器中定義單一的 `__invoke` 方法：

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

為單一動作控制器註冊路由時，您不需要指定控制器方法。相反地，只需將控制器的名稱傳遞給路由器即可：

```php
use App\Http\Controllers\ProvisionServer;

Route::post('/server', ProvisionServer::class);
```

您可以使用 `make:controller` Artisan 指令的 `--invokable` 選項來產生可調用的控制器：

```shell
php artisan make:controller ProvisionServer --invokable
```

> [!NOTE]
> 控制器 stub 可以使用 [stub 自訂](/docs/{{version}}/artisan#stub-customization) 功能進行自訂。


<a name="controller-middleware"></a>
## 控制器中介層

[中介層](/docs/{{version}}/middleware) 可以在您的路由檔案中指派給控制器的路由：

```php
Route::get('/profile', [UserController::class, 'show'])->middleware('auth');
```

或者，您可能會發現在控制器類別內指定中介層更為方便。若要這樣做，您的控制器應實作 `HasMiddleware` 介面，該介面規定控制器應具有靜態的 `middleware` 方法。在此方法中，您可以回傳應套用至控制器動作的中介層陣列：

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

您也可以將控制器中介層定義為閉包，這提供了一種方便的方式來定義行內中介層，而無需編寫整個中介層類別：

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
### 中介層屬性 (Attributes)

您也可以使用 PHP 屬性 (Attributes) 將中介層指派給控制器：

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

您也可以將中介層屬性放置在個別的控制器方法上。指派給方法的中介層將與類別層級指派的中介層進行合併：

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

若要從控制器或個別控制器方法中排除中介層，請使用 `WithoutMiddleware` 屬性。您可以使用 `only` 和 `except` 引數將類別層級的屬性限制在特定的控制器方法：

```php
<?php

namespace App\Http\Controllers;

use App\Http\Middleware\EnsureTokenIsValid;
use Illuminate\Routing\Attributes\Controllers\WithoutMiddleware;

#[WithoutMiddleware('subscribed', except: ['index'])]
class UserController
{
    #[WithoutMiddleware(EnsureTokenIsValid::class)]
    public function index()
    {
        // ...
    }

    public function show()
    {
        // ...
    }
}
```

類別層級的 `WithoutMiddleware` 屬性會被子控制器繼承。該屬性只能移除路由中介層，不適用於[全域中介層](/docs/{{version}}/middleware#global-middleware)。


<a name="authorization-attributes"></a>
### 授權屬性 (Attributes)

如果您透過 Policy 授權控制器動作，可以使用 `Authorize` 屬性作為 `can` 中介層的便利捷徑：

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

第一個引數是您想要授權的能力 (Ability)。第二個引數是應該傳遞給 Policy 的 Model 類別、路由參數或參數清單。

<a name="resource-controllers"></a>
## 資源控制器

如果你將應用程式中的每個 Eloquent Model 都視為一個「資源 (Resource)」，那麼通常會對應用程式中的每個資源執行相同的操作組合。例如，假設你的應用程式包含一個 `Photo` Model 和一個 `Movie` Model，使用者很可能會對這些資源進行建立、讀取、更新或刪除。

基於這種常見的使用情境，Laravel 的資源路由只需一行程式碼，即可將常見的建立、讀取、更新和刪除 (「CRUD」) 路由分配給控制器。首先，我們可以使用 `make:controller` Artisan 指令的 `--resource` 選項來快速建立一個處理這些動作的控制器：

```shell
php artisan make:controller PhotoController --resource
```

此指令將在 `app/Http/Controllers/PhotoController.php` 產生一個控制器。該控制器將為每個可用的資源操作包含對應的方法。接著，你可以註冊一個指向該控制器的資源路由：

```php
use App\Http\Controllers\PhotoController;

Route::resource('photos', PhotoController::class);
```

這個單一路由宣告建立了多個路由，用來處理對該資源的各種動作。產生的控制器中已經為這些動作預先建立了方法骨架。請記住，你隨時可以透過執行 `route:list` Artisan 指令來快速檢視應用程式的所有路由。

你甚至可以透過傳遞陣列給 `resources` 方法，一次註冊多個資源控制器：

```php
Route::resources([
    'photos' => PhotoController::class,
    'posts' => PostController::class,
]);
```

`softDeletableResources` 方法可以註冊多個都使用 `withTrashed` 方法的資源控制器：

```php
Route::softDeletableResources([
    'photos' => PhotoController::class,
    'posts' => PostController::class,
]);
```


<a name="actions-handled-by-resource-controllers"></a>
#### 資源控制器處理的動作

<div class="overflow-auto">

| Verb      | URI                    | Action  | Route Name     |
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
#### 自訂找不到 Model 時的行為

通常，如果隱式綁定的資源 Model 找不到時，會產生 404 HTTP 回應。不過，你可以在定義資源路由時呼叫 `missing` 方法來自訂此行為。`missing` 方法接受一個閉包，當資源的任何路由找不到隱式綁定的 Model 時，就會叫用該閉包：

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
#### 軟刪除的 Model

通常，隱式 Model 綁定不會取得已被[軟刪除](/docs/{{version}}/eloquent#soft-deleting)的 Model，而是會回傳 404 HTTP 回應。不過，你可以在定義資源路由時叫用 `withTrashed` 方法，指示框架允許軟刪除的 Model：

```php
use App\Http\Controllers\PhotoController;

Route::resource('photos', PhotoController::class)->withTrashed();
```

呼叫不帶引數的 `withTrashed` 將允許 `show`、`edit` 和 `update` 資源路由使用軟刪除的 Model。你也可以透過傳遞陣列給 `withTrashed` 方法來指定這些路由的子集：

```php
Route::resource('photos', PhotoController::class)->withTrashed(['show']);
```


<a name="specifying-the-resource-model"></a>
#### 指定資源 Model

如果你正在使用[路由 Model 綁定](/docs/{{version}}/routing#route-model-binding)，並且希望資源控制器的方法具有 Model 實例的型別提示，可以在產生控制器時使用 `--model` 選項：

```shell
php artisan make:controller PhotoController --model=Photo --resource
```


<a name="generating-form-requests"></a>
#### 產生表單請求

你可以在產生資源控制器時提供 `--requests` 選項，以指示 Artisan 為控制器的 storage 和 update 方法產生[表單請求(Form request)類別](/docs/{{version}}/validation#form-request-validation)：

```shell
php artisan make:controller PhotoController --model=Photo --resource --requests
```


<a name="restful-partial-resource-routes"></a>
### 部分資源路由

宣告資源路由時，你可以指定控制器應處理的動作子集，而不是完整的預設動作集：

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

宣告將由 API 使用的資源路由時，通常會想要排除提供 HTML 模板的路由，例如 `create` 和 `edit`。為了方便起見，你可以使用 `apiResource` 方法自動排除這兩個路由：

```php
use App\Http\Controllers\PhotoController;

Route::apiResource('photos', PhotoController::class);
```

你也可以透過傳遞陣列給 `apiResources` 方法，一次註冊多個 API 資源控制器：

```php
use App\Http\Controllers\PhotoController;
use App\Http\Controllers\PostController;

Route::apiResources([
    'photos' => PhotoController::class,
    'posts' => PostController::class,
]);
```

若要快速產生不包含 `create` 或 `edit` 方法的 API 資源控制器，可以在執行 `make:controller` 指令時使用 `--api` 參數：

```shell
php artisan make:controller PhotoController --api
```

<a name="restful-nested-resources"></a>
### 巢狀資源

有時您可能需要為巢狀資源定義路由。例如，一個相片 (Photo) 資源可能有多個附加到該相片的評論 (Comment)。若要巢狀化資源控制器，您可以在路由宣告中使用「點號 (dot)」標記法：

```php
use App\Http\Controllers\PhotoCommentController;

Route::resource('photos.comments', PhotoCommentController::class);
```

此路由將註冊一個巢狀資源，可透過類似以下的 URI 來存取：

```text
/photos/{photo}/comments/{comment}
```


<a name="scoping-nested-resources"></a>
#### 限定巢狀資源範圍

Laravel 的[隱式模型綁定](/docs/{{version}}/routing#implicit-model-binding-scoping)功能可以自動限定巢狀綁定的範圍，以確保解析出的子模型確實屬於父模型。在定義巢狀資源時使用 `scoped` 方法，您可以啟用自動範圍限定，並指示 Laravel 應透過子資源的哪個欄位來取得模型。有關如何實現此功能的更多資訊，請參閱[限定資源路由範圍](#restful-scoping-resource-routes)的說明文件。


<a name="shallow-nesting"></a>
#### 淺層巢狀

通常，在 URI 中同時包含父層與子層的 ID 並非完全必要，因為子層 ID 本身就已經是唯一識別碼。當在 URI 區段中使用自動遞增主鍵等唯一識別碼來識別您的模型時，您可以選擇使用「淺層巢狀 (Shallow Nesting)」：

```php
use App\Http\Controllers\CommentController;

Route::resource('photos.comments', CommentController::class)->shallow();
```

這個路由定義將會產生以下路由：

<div class="overflow-auto">

| Verb      | URI                               | Action  | Route Name             |
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

預設情況下，所有資源控制器動作都有一個路由名稱；不過，您可以透過傳入帶有您所需路由名稱的 `names` 陣列來覆寫這些名稱：

```php
use App\Http\Controllers\PhotoController;

Route::resource('photos', PhotoController::class)->names([
    'create' => 'photos.build'
]);
```


<a name="restful-naming-resource-route-parameters"></a>
### 命名資源路由參數

預設情況下，`Route::resource` 會根據資源名稱的「單數化」版本來建立資源路由的路由參數。您可以透過 `parameters` 方法針對個別資源輕鬆地覆寫此設定。傳入 `parameters` 方法的陣列應該是資源名稱與參數名稱的關聯陣列：

```php
use App\Http\Controllers\AdminUserController;

Route::resource('users', AdminUserController::class)->parameters([
    'users' => 'admin_user'
]);
```

上述範例會為資源的 `show` 路由產生以下 URI：

```text
/users/{admin_user}
```


<a name="restful-scoping-resource-routes"></a>
### 限定資源路由範圍

Laravel 的[範圍限定隱式模型綁定](/docs/{{version}}/routing#implicit-model-binding-scoping)功能可以自動限定巢狀綁定的範圍，以確保解析出的子模型確實屬於父模型。在定義巢狀資源時使用 `scoped` 方法，您可以啟用自動範圍限定，並指示 Laravel 應透過子資源的哪個欄位來取得模型：

```php
use App\Http\Controllers\PhotoCommentController;

Route::resource('photos.comments', PhotoCommentController::class)->scoped([
    'comment' => 'slug',
]);
```

此路由將註冊一個限定範圍的巢狀資源，可透過類似以下的 URI 來存取：

```text
/photos/{photo}/comments/{comment:slug}
```

當在巢狀路由參數中使用自訂鍵值的隱式綁定時，Laravel 會根據慣例猜測父模型上的關聯名稱，自動限定查詢範圍以透過其父模型取得巢狀模型。在此範例中，系統將假設 `Photo` 模型具有名為 `comments`（路由參數名稱的複數形式）的關聯，可用於取得 `Comment` 模型。


<a name="restful-localizing-resource-uris"></a>
### 本地化資源 URI

預設情況下，`Route::resource` 將使用英文動詞和複數規則建立資源 URI。如果您需要本地化 `create` 和 `edit` 動作動詞，可以使用 `Route::resourceVerbs` 方法。這可以在應用程式的 `App\Providers\AppServiceProvider` 中的 `boot` 方法開頭完成：

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

Laravel 的複數器支援[數種不同的語言，您可以根據需求進行設定](/docs/{{version}}/localization#pluralization-language)。動詞和複數語言自訂完成後，像 `Route::resource('publicacion', PublicacionController::class)` 這樣的資源路由註冊將產生以下 URI：

```text
/publicacion/crear

/publicacion/{publicaciones}/editar
```


<a name="restful-supplementing-resource-controllers"></a>
### 擴充資源控制器

如果您需要在預設的資源路由集合之外，為資源控制器新增其他額外路由，您應該在呼叫 `Route::resource` 方法之前定義這些路由；否則，由 `resource` 方法定義的路由可能會無意中優先於您的擴充路由：

```php
use App\Http\Controller\PhotoController;

Route::get('/photos/popular', [PhotoController::class, 'popular']);
Route::resource('photos', PhotoController::class);
```

> [!NOTE]
> 請記得讓您的控制器保持專注。如果您發現自己經常需要典型資源動作集合之外的方法，請考慮將控制器拆分為兩個較小的控制器。

<a name="singleton-resource-controllers"></a>
### 單例資源控制器

有時，您的應用程式中會有僅存在單一實例的資源。例如，使用者的「個人檔案 (profile)」可以被編輯或更新，但一個使用者不能擁有多個「個人檔案」。同樣地，一張圖片可能只有一個「縮圖 (thumbnail)」。這些資源被稱為「單例資源 (singleton resources)」，意即該資源只存在且僅能存在一個實例。在這些情境下，您可以註冊一個「單例」資源控制器：

```php
use App\Http\Controllers\ProfileController;
use Illuminate\Support\Facades\Route;

Route::singleton('profile', ProfileController::class);
```

上述的單例資源定義將會註冊以下路由。如您所見，單例資源不會註冊「建立 (creation)」路由，且已註冊的路由不接受識別碼，因為該資源僅存在一個實例：

<div class="overflow-auto">

| Verb      | URI             | Action | Route Name     |
| --------- | --------------- | ------ | -------------- |
| GET       | `/profile`      | show   | profile.show   |
| GET       | `/profile/edit` | edit   | profile.edit   |
| PUT/PATCH | `/profile`      | update | profile.update |

</div>

單例資源也可以巢狀定義於標準資源之中：

```php
Route::singleton('photos.thumbnail', ThumbnailController::class);
```

在這個範例中，`photos` 資源將會獲得所有[標準資源路由](#actions-handled-by-resource-controllers)；然而，`thumbnail` 資源則會是一個具有下列路由的單例資源：

<div class="overflow-auto">

| Verb      | URI                              | Action | Route Name              |
| --------- | -------------------------------- | ------ | ----------------------- |
| GET       | `/photos/{photo}/thumbnail`      | show   | photos.thumbnail.show   |
| GET       | `/photos/{photo}/thumbnail/edit` | edit   | photos.thumbnail.edit   |
| PUT/PATCH | `/photos/{photo}/thumbnail`      | update | photos.thumbnail.update |

</div>


<a name="creatable-singleton-resources"></a>
#### 可建立的單例資源

有時，您可能會想要為單例資源定義建立與儲存路由。若要達成此目的，您可以在註冊單例資源路由時呼叫 `creatable` 方法：

```php
Route::singleton('photos.thumbnail', ThumbnailController::class)->creatable();
```

在這個範例中，將會註冊下列路由。如您所見，可建立的單例資源也會註冊一個 `DELETE` 路由：

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

`apiSingleton` 方法可用於註冊將透過 API 進行操作的單例資源，從而使 `create` 與 `edit` 路由變得不必要：

```php
Route::apiSingleton('profile', ProfileController::class);
```

當然，API 單例資源也可以是 `creatable`，這將會為該資源註冊 `store` 與 `destroy` 路由：

```php
Route::apiSingleton('photos.thumbnail', ProfileController::class)->creatable();
```

<a name="middleware-and-resource-controllers"></a>
### 中介層與資源控制器

Laravel 允許您使用 `middleware`、`middlewareFor` 以及 `withoutMiddlewareFor` 方法，將中介層指派給資源路由的所有或特定方法。這些方法提供了精細的控制，以決定將哪個中介層套用至每個資源動作。


#### 套用中介層至所有方法

您可以使用 `middleware` 方法將中介層指派給由資源或單例資源路由所產生的所有路由：

```php
Route::resource('users', UserController::class)
    ->middleware(['auth', 'verified']);

Route::singleton('profile', ProfileController::class)
    ->middleware('auth');
```


#### 套用中介層至特定方法

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

`middlewareFor` 方法也可以與單例以及 API 單例資源控制器搭配使用：

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
#### 建構式注入

Laravel [服務容器](/docs/{{version}}/container)用於解析所有 Laravel 控制器。因此，您可以在控制器的建構式中型別提示控制器可能需要的任何依賴項目。宣告的依賴項目將會被自動解析並注入到控制器實例中：

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

除了建構式注入之外，您也可以在控制器的方法中型別提示依賴項目。方法注入的一個常見使用情境是將 `Illuminate\Http\Request` 實例注入到您的控制器方法中：

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

如果您的控制器方法也預期接收來自路由參數的輸入，請在其他依賴項目之後列出您的路由引數。例如，如果您的路由定義如下：

```php
use App\Http\Controllers\UserController;

Route::put('/user/{id}', [UserController::class, 'update']);
```

您仍然可以透過如下方式定義控制器方法，來型別提示 `Illuminate\Http\Request` 並存取您的 `id` 參數：

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