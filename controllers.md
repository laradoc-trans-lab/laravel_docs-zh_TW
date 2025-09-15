# Controllers

- [簡介](#introduction)
- [撰寫 Controller](#writing-controllers)
    - [基本 Controller](#basic-controllers)
    - [單一動作 Controller](#single-action-controllers)
- [Controller Middleware](#controller-middleware)
- [資源 Controller](#resource-controllers)
    - [部分資源路由](#restful-partial-resource-routes)
    - [巢狀資源](#restful-nested-resources)
    - [命名資源路由](#restful-naming-resource-routes)
    - [命名資源路由參數](#restful-naming-resource-route-parameters)
    - [範圍化資源路由](#restful-scoping-resource-routes)
    - [本地化資源 URI](#restful-localizing-resource-uris)
    - [補充資源 Controller](#restful-supplementing-resource-controllers)
    - [單例資源 Controller](#singleton-resource-controllers)
- [依賴注入與 Controller](#dependency-injection-and-controllers)

<a name="introduction"></a>
## 簡介

與其將所有請求處理邏輯定義為路由檔案中的閉包，您可能會希望使用「Controller」類別來組織這些行為。Controller 可以將相關的請求處理邏輯分組到單一類別中。例如，一個 `UserController` 類別可能會處理所有與使用者相關的傳入請求，包括顯示、建立、更新和刪除使用者。預設情況下，Controller 儲存在 `app/Http/Controllers` 目錄中。

<a name="writing-controllers"></a>
## 撰寫 Controller

<a name="basic-controllers"></a>
### 基本 Controller

若要快速產生一個新的 Controller，您可以執行 `make:controller` Artisan 命令。預設情況下，應用程式的所有 Controller 都儲存在 `app/Http/Controllers` 目錄中：

```shell
php artisan make:controller UserController
```

讓我們看看一個基本 Controller 的範例。一個 Controller 可以有多個公開方法，這些方法將回應傳入的 HTTP 請求：

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

一旦您撰寫了 Controller 類別和方法，您可以像這樣為 Controller 方法定義路由：

    use App\Http\Controllers\UserController;

    Route::get('/user/{id}', [UserController::class, 'show']);

當傳入請求與指定的路由 URI 匹配時，`App\Http\Controllers\UserController` 類別上的 `show` 方法將會被呼叫，並且路由參數將會傳遞給該方法。

> [!NOTE]  
> Controller **不一定**需要繼承基礎類別。然而，有時繼承一個包含應在所有 Controller 中共用方法的基礎 Controller 類別會很方便。

<a name="single-action-controllers"></a>
### 單一動作 Controller

如果一個 Controller 動作特別複雜，您可能會覺得將整個 Controller 類別專用於該單一動作會很方便。為此，您可以在 Controller 中定義一個單一的 `__invoke` 方法：

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

為單一動作 Controller 註冊路由時，您不需要指定 Controller 方法。相反地，您可以直接將 Controller 的名稱傳遞給路由器：

    use App\Http\Controllers\ProvisionServer;

    Route::post('/server', ProvisionServer::class);

您可以使用 `make:controller` Artisan 命令的 `--invokable` 選項來產生一個可呼叫的 Controller：

```shell
php artisan make:controller ProvisionServer --invokable
```

> [!NOTE]  
> Controller 樣板可以使用 [樣板發佈](/docs/{{version}}/artisan#stub-customization) 進行自訂。

<a name="controller-middleware"></a>
## Controller Middleware

[Middleware](/docs/{{version}}/middleware) 可以指派給路由檔案中的 Controller 路由：

    Route::get('/profile', [UserController::class, 'show'])->middleware('auth');

或者，您可能會覺得在 Controller 類別中指定 Middleware 更方便。為此，您的 Controller 應該實作 `HasMiddleware` 介面，這表示 Controller 應該有一個靜態的 `middleware` 方法。從這個方法中，您可以回傳一個 Middleware 陣列，這些 Middleware 應該應用於 Controller 的動作：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
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

您也可以將 Controller Middleware 定義為閉包，這提供了一種方便的方式來定義行內 Middleware，而無需撰寫整個 Middleware 類別：

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

> [!WARNING]  
> 實作 `Illuminate\Routing\Controllers\HasMiddleware` 的 Controller 不應繼承 `Illuminate\Routing\Controller`。

<a name="resource-controllers"></a>
## 資源 Controller

如果您將應用程式中的每個 Eloquent 模型視為一個「資源」，那麼對應用程式中的每個資源執行相同的動作集是很常見的。例如，假設您的應用程式包含一個 `Photo` 模型和一個 `Movie` 模型。使用者很可能可以建立、讀取、更新或刪除這些資源。

由於這種常見的使用情境，Laravel 資源路由只需一行程式碼即可將典型的建立、讀取、更新和刪除 (「CRUD」) 路由指派給 Controller。首先，我們可以使用 `make:controller` Artisan 命令的 `--resource` 選項來快速建立一個 Controller 來處理這些動作：

```shell
php artisan make:controller PhotoController --resource
```

此命令將在 `app/Http/Controllers/PhotoController.php` 產生一個 Controller。該 Controller 將包含每個可用資源操作的方法。接下來，您可以註冊一個指向該 Controller 的資源路由：

    use App\Http\Controllers\PhotoController;

    Route::resource('photos', PhotoController::class);

這個單一的路由宣告會建立多個路由來處理資源上的各種動作。產生的 Controller 將已經為每個這些動作預留了方法。請記住，您始終可以透過執行 `route:list` Artisan 命令來快速概覽應用程式的路由。

您甚至可以透過將陣列傳遞給 `resources` 方法來一次註冊多個資源 Controller：

    Route::resources([
        'photos' => PhotoController::class,
        'posts' => PostController::class,
    ]);

<a name="actions-handled-by-resource-controllers"></a>
#### 資源 Controller 處理的動作

<div class="overflow-auto">

| Verb      | URI                    | Action  | Route Name     |
| --------- | ---------------------- | ------- | -------------- |
| GET       | `/photos`              | index   | photos.index   |
| GET       | `/photos/create`       | create  | photos.create  |
| POST       | `/photos`              | store   | photos.store   |
| GET       | `/photos/{photo}`      | show    | photos.show    |
| GET       | `/photos/{photo}/edit` | edit    | photos.edit    |
| PUT/PATCH | `/photos/{photo}`      | update  | photos.update  |
| DELETE    | `/photos/{photo}`      | destroy | photos.destroy |

</div>

<a name="customizing-missing-model-behavior"></a>
#### 自訂遺失模型行為

通常，如果找不到隱式綁定的資源模型，將會產生 404 HTTP 回應。但是，您可以在定義資源路由時呼叫 `missing` 方法來自訂此行為。`missing` 方法接受一個閉包，如果找不到任何資源路由的隱式綁定模型，該閉包將會被呼叫：

    use App\Http\Controllers\PhotoController;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Redirect;

    Route::resource('photos', PhotoController::class)
        ->missing(function (Request $request) {
            return Redirect::route('photos.index');
        });

<a name="soft-deleted-models"></a>
#### 軟刪除模型

通常，隱式模型綁定不會檢索已 [軟刪除](/docs/{{version}}/eloquent#soft-deleting) 的模型，而是會回傳 404 HTTP 回應。但是，您可以在定義資源路由時呼叫 `withTrashed` 方法，指示框架允許軟刪除模型：

    use App\Http\Controllers\PhotoController;

    Route::resource('photos', PhotoController::class)->withTrashed();

不帶任何參數呼叫 `withTrashed` 將允許 `show`、`edit` 和 `update` 資源路由的軟刪除模型。您可以透過將陣列傳遞給 `withTrashed` 方法來指定這些路由的子集：

    Route::resource('photos', PhotoController::class)->withTrashed(['show']);

<a name="specifying-the-resource-model"></a>
#### 指定資源模型

如果您正在使用 [路由模型綁定](/docs/{{version}}/routing#route-model-binding) 並希望資源 Controller 的方法能夠型別提示模型實例，您可以在產生 Controller 時使用 `--model` 選項：

```shell
php artisan make:controller PhotoController --model=Photo --resource
```

<a name="generating-form-requests"></a>
#### 產生表單請求

您可以在產生資源 Controller 時提供 `--requests` 選項，以指示 Artisan 為 Controller 的儲存和更新方法產生 [表單請求類別](/docs/{{version}}/validation#form-request-validation)：

```shell
php artisan make:controller PhotoController --model=Photo --resource --requests
```

<a name="restful-partial-resource-routes"></a>
### 部分資源路由

宣告資源路由時，您可以指定 Controller 應該處理的動作子集，而不是完整的預設動作集：

    use App\Http\Controllers\PhotoController;

    Route::resource('photos', PhotoController::class)->only([
        'index', 'show'
    ]);

    Route::resource('photos', PhotoController::class)->except([
        'create', 'store', 'update', 'destroy'
    ]);

<a name="api-resource-routes"></a>
#### API 資源路由

當宣告將由 API 消費的資源路由時，您通常會希望排除呈現 HTML 樣板的路由，例如 `create` 和 `edit`。為了方便起見，您可以使用 `apiResource` 方法自動排除這兩個路由：

    use App\Http\Controllers\PhotoController;

    Route::apiResource('photos', PhotoController::class);

您可以透過將陣列傳遞給 `apiResources` 方法來一次註冊多個 API 資源 Controller：

    use App\Http\Controllers\PhotoController;
    use App\Http\Controllers\PostController;

    Route::apiResources([
        'photos' => PhotoController::class,
        'posts' => PostController::class,
    ]);

若要快速產生一個不包含 `create` 或 `edit` 方法的 API 資源 Controller，請在執行 `make:controller` 命令時使用 `--api` 開關：

```shell
php artisan make:controller PhotoController --api
```

<a name="restful-nested-resources"></a>
### 巢狀資源

有時您可能需要定義巢狀資源的路由。例如，一個照片資源可能有多個評論可以附加到該照片。若要巢狀化資源 Controller，您可以在路由宣告中使用「點」符號：

    use App\Http\Controllers\PhotoCommentController;

    Route::resource('photos.comments', PhotoCommentController::class);

此路由將註冊一個巢狀資源，可以透過以下 URI 存取：

    /photos/{photo}/comments/{comment}

<a name="scoping-nested-resources"></a>
#### 範圍化巢狀資源

Laravel 的 [隱式模型綁定](/docs/{{version}}/routing#implicit-model-binding-scoping) 功能可以自動範圍化巢狀綁定，以確認解析的子模型屬於父模型。透過在定義巢狀資源時使用 `scoped` 方法，您可以啟用自動範圍化，並指示 Laravel 應該透過哪個欄位檢索子資源。有關如何實現此目的的更多資訊，請參閱 [範圍化資源路由](#restful-scoping-resource-routes) 的文件。

<a name="shallow-nesting"></a>
#### 淺層巢狀

通常，URI 中同時包含父 ID 和子 ID 並非完全必要，因為子 ID 已經是唯一的識別碼。當使用唯一識別碼 (例如自動遞增的主鍵) 來識別 URI 片段中的模型時，您可以選擇使用「淺層巢狀」：

    use App\Http\Controllers\CommentController;

    Route::resource('photos.comments', CommentController::class)->shallow();

此路由定義將定義以下路由：

<div class="overflow-auto">

| Verb      | URI                               | Action  | Route Name             |
| --------- | --------------------------------- | ------- | ---------------------- |
| GET       | `/photos/{photo}/comments`        | index   | photos.comments.index  |
| GET       | `/photos/{photo}/comments/create` | create  | photos.comments.create |
| POST       | `/photos/{photo}/comments`        | store   | photos.comments.store  |
| GET       | `/comments/{comment}`             | show    | comments.show          |
| GET       | `/comments/{comment}/edit`        | edit    | comments.edit          |
| PUT/PATCH | `/comments/{comment}`             | update  | comments.update        |
| DELETE    | `/comments/{comment}`             | destroy | comments.destroy       |

</div>

<a name="restful-naming-resource-routes"></a>
### 命名資源路由

預設情況下，所有資源 Controller 動作都有一個路由名稱；但是，您可以透過傳遞一個包含您所需路由名稱的 `names` 陣列來覆寫這些名稱：

    use App\Http\Controllers\PhotoController;

    Route::resource('photos', PhotoController::class)->names([
        'create' => 'photos.build'
    ]);

<a name="restful-naming-resource-route-parameters"></a>
### 命名資源路由參數

預設情況下，`Route::resource` 將根據資源名稱的「單數化」版本為您的資源路由建立路由參數。您可以使用 `parameters` 方法輕鬆地為每個資源覆寫此設定。傳遞給 `parameters` 方法的陣列應該是資源名稱和參數名稱的關聯陣列：

    use App\Http\Controllers\AdminUserController;

    Route::resource('users', AdminUserController::class)->parameters([
        'users' => 'admin_user'
    ]);

上面的範例為資源的 `show` 路由產生以下 URI：

    /users/{admin_user}

<a name="restful-scoping-resource-routes"></a>
### 範圍化資源路由

Laravel 的 [範圍化隱式模型綁定](/docs/{{version}}/routing#implicit-model-binding-scoping) 功能可以自動範圍化巢狀綁定，以確認解析的子模型屬於父模型。透過在定義巢狀資源時使用 `scoped` 方法，您可以啟用自動範圍化，並指示 Laravel 應該透過哪個欄位檢索子資源：

    use App\Http\Controllers\PhotoCommentController;

    Route::resource('photos.comments', PhotoCommentController::class)->scoped([
        'comment' => 'slug',
    ]);

此路由將註冊一個範圍化的巢狀資源，可以透過以下 URI 存取：

    /photos/{photo}/comments/{comment:slug}

當使用自訂鍵的隱式綁定作為巢狀路由參數時，Laravel 將自動範圍化查詢，透過其父級檢索巢狀模型，並使用慣例來猜測父級上的關聯名稱。在這種情況下，將假定 `Photo` 模型具有一個名為 `comments` (路由參數名稱的複數) 的關聯，該關聯可用於檢索 `Comment` 模型。

<a name="restful-localizing-resource-uris"></a>
### 本地化資源 URI

預設情況下，`Route::resource` 將使用英文動詞和複數規則建立資源 URI。如果您需要本地化 `create` 和 `edit` 動作動詞，您可以使用 `Route::resourceVerbs` 方法。這可以在應用程式的 `App\Providers\AppServiceProvider` 中的 `boot` 方法開頭完成：

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

Laravel 的複數化器支援 [多種不同的語言，您可以根據您的需求進行配置](/docs/{{version}}/localization#pluralization-language)。一旦動詞和複數化語言被自訂，像 `Route::resource('publicacion', PublicacionController::class)` 這樣的資源路由註冊將產生以下 URI：

    /publicacion/crear

    /publicacion/{publicaciones}/editar

<a name="restful-supplementing-resource-controllers"></a>
### 補充資源 Controller

如果您需要為資源 Controller 添加預設資源路由集之外的額外路由，您應該在呼叫 `Route::resource` 方法之前定義這些路由；否則，由 `resource` 方法定義的路由可能會無意中優先於您的補充路由：

    use App\Http\Controller\PhotoController;

    Route::get('/photos/popular', [PhotoController::class, 'popular']);
    Route::resource('photos', PhotoController::class);

> [!NOTE]  
> 請記住讓您的 Controller 保持專注。如果您發現自己經常需要典型資源動作集之外的方法，請考慮將您的 Controller 拆分為兩個較小的 Controller。

<a name="singleton-resource-controllers"></a>
### 單例資源 Controller

有時，您的應用程式會有一些資源可能只有一個實例。例如，使用者的「個人資料」可以被編輯或更新，但使用者不能擁有多個「個人資料」。同樣地，一張圖片可能只有一個「縮圖」。這些資源稱為「單例資源」，表示該資源只能存在一個實例。在這些情境中，您可以註冊一個「單例」資源 Controller：

```php
use App\Http\Controllers\ProfileController;
use Illuminate\Support\Facades\Route;

Route::singleton('profile', ProfileController::class);
```

上述單例資源定義將註冊以下路由。如您所見，單例資源不會註冊「建立」路由，並且註冊的路由不接受識別碼，因為該資源只能存在一個實例：

<div class="overflow-auto">

| Verb      | URI             | Action | Route Name     |
| --------- | --------------- | ------ | -------------- |
| GET       | `/profile`      | show   | profile.show   |
| GET       | `/profile/edit` | edit   | profile.edit   |
| PUT/PATCH | `/profile`      | update | profile.update |

</div>

單例資源也可以巢狀在標準資源中：

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

有時，您可能希望為單例資源定義建立和儲存路由。為此，您可以在註冊單例資源路由時呼叫 `creatable` 方法：

```php
Route::singleton('photos.thumbnail', ThumbnailController::class)->creatable();
```

在此範例中，將註冊以下路由。如您所見，可建立的單例資源也將註冊一個 `DELETE` 路由：

<div class="overflow-auto">

| Verb      | URI                                | Action  | Route Name               |
| --------- | ---------------------------------- | ------- | ------------------------ |
| GET       | `/photos/{photo}/thumbnail/create` | create  | photos.thumbnail.create  |
| POST       | `/photos/{photo}/thumbnail`        | store   | photos.thumbnail.store   |
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

<a name="dependency-injection-and-controllers"></a>
## 依賴注入與 Controller

<a name="constructor-injection"></a>
#### 建構子注入

Laravel [服務容器](/docs/{{version}}/container) 用於解析所有 Laravel Controller。因此，您可以在 Controller 的建構子中型別提示 Controller 可能需要的任何依賴。宣告的依賴將自動被解析並注入到 Controller 實例中：

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

<a name="method-injection"></a>
#### 方法注入

除了建構子注入之外，您還可以在 Controller 的方法上型別提示依賴。方法注入的一個常見用例是將 `Illuminate\Http\Request` 實例注入到您的 Controller 方法中：

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

如果您的 Controller 方法也期望來自路由參數的輸入，請在其他依賴之後列出您的路由參數。例如，如果您的路由定義如下：

    use App\Http\Controllers\UserController;

    Route::put('/user/{id}', [UserController::class, 'update']);

您仍然可以型別提示 `Illuminate\Http\Request` 並透過以下方式定義您的 Controller 方法來存取您的 `id` 參數：

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

