# 控制器

- [簡介](#introduction)
- [撰寫控制器](#writing-controllers)
    - [基本控制器](#basic-controllers)
    - [單一行為控制器](#single-action-controllers)
- [控制器中介層](#controller-middleware)
- [資源控制器](#resource-controllers)
    - [部分資源路由](#restful-partial-resource-routes)
    - [巢狀資源](#restful-nested-resources)
    - [命名資源路由](#restful-naming-resource-routes)
    - [命名資源路由參數](#restful-naming-resource-route-parameters)
    - [限定資源路由範圍](#restful-scoping-resource-routes)
    - [本地化資源 URI](#restful-localizing-resource-uris)
    - [補充資源控制器](#restful-supplementing-resource-controllers)
    - [單例資源控制器](#singleton-resource-controllers)
- [依賴注入與控制器](#dependency-injection-and-controllers)

<a name="introduction"></a>
## 簡介

與其在路由檔案中將所有請求處理邏輯定義為閉包，您可能希望使用「控制器」類別來組織此行為。控制器可以將相關的請求處理邏輯分組到單一類別中。例如，`UserController` 類別可以處理所有與使用者相關的傳入請求，包括顯示、建立、更新和刪除使用者。預設情況下，控制器儲存在 `app/Http/Controllers` 目錄中。

<a name="writing-controllers"></a>
## 撰寫控制器

<a name="basic-controllers"></a>
### 基本控制器

若要快速產生一個新控制器，您可以執行 `make:controller` Artisan 命令。預設情況下，應用程式中的所有控制器都儲存在 `app/Http/Controllers` 目錄中：

```shell
php artisan make:controller UserController
```

讓我們看看基本控制器的範例。控制器可以有任意數量的公開方法，這些方法將響應傳入的 HTTP 請求：

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

一旦您撰寫了控制器類別和方法，您可以像這樣定義指向控制器方法的路由：

    use App\Http\Controllers\UserController;

    Route::get('/user/{id}', [UserController::class, 'show']);

當傳入請求符合指定的路由 URI 時，`App\Http\Controllers\UserController` 類別上的 `show` 方法將被調用，並且路由參數將被傳遞給該方法。

> [!NOTE]  
> 控制器不**要求**繼承基礎類別。然而，有時繼承一個包含應在所有控制器之間共用方法的基礎控制器類別會很方便。

<a name="single-action-controllers"></a>
### 單一行為控制器

如果一個控制器行為特別複雜，您可能會發現將整個控制器類別專用於該單一行為會很方便。為此，您可以在控制器中定義一個單一的 `__invoke` 方法：

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

在為單一行為控制器註冊路由時，您不需要指定控制器方法。相反地，您只需將控制器的名稱傳遞給路由器：

    use App\Http\Controllers\ProvisionServer;

    Route::post('/server', ProvisionServer::class);

您可以透過使用 `make:controller` Artisan 命令的 `--invokable` 選項來產生一個可調用控制器：

```shell
php artisan make:controller ProvisionServer --invokable
```

> [!NOTE]  
> 控制器 Stub 可以使用 [Stub 發布](/docs/{{version}}/artisan#stub-customization) 進行自訂。

<a name="controller-middleware"></a>
## 控制器中介層

[中介層](/docs/{{version}}/middleware) 可以被分配到控制器在路由檔案中的路由：

    Route::get('/profile', [UserController::class, 'show'])->middleware('auth');

或者，您可能會發現在控制器類別中指定中介層會很方便。為此，您的控制器應實作 `HasMiddleware` 介面，該介面規定控制器應具有靜態的 `middleware` 方法。從這個方法中，您可以返回一個中介層陣列，這些中介層應該應用於控制器的行為：

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

您也可以將控制器中介層定義為閉包，這提供了一種方便的方式來定義行內中介層，而無需撰寫一個完整的中介層類別：

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
> 實作 `Illuminate\Routing\Controllers\HasMiddleware` 的控制器不應繼承 `Illuminate\Routing\Controller`。

<a name="resource-controllers"></a>
## 資源控制器

如果您將應用程式中的每個 Eloquent model 都視為「資源」，那麼對每個資源執行相同的操作集合是相當典型的做法。例如，假設您的應用程式包含一個 `Photo` 模型和一個 `Movie` 模型。使用者很可能可以建立、讀取、更新或刪除這些資源。

由於這個常見的使用情境，Laravel 的資源路由會將典型的建立、讀取、更新和刪除（「CRUD」）路由以單行程式碼的方式指派給控制器。為了快速開始，我們可以使用 `make:controller` Artisan 命令的 `--resource` 選項來快速建立一個控制器以處理這些操作：

```shell
php artisan make:controller PhotoController --resource
```

這個命令會在 `app/Http/Controllers/PhotoController.php` 產生一個控制器。該控制器將包含每個可用資源操作的方法。接下來，您可以註冊一個指向該控制器的資源路由：

    use App\Http\Controllers\PhotoController;

    Route::resource('photos', PhotoController::class);

這個單一的路由宣告會建立多個路由來處理資源上的各種操作。產生的控制器將預先為這些操作預留方法。請記住，您隨時可以執行 `route:list` Artisan 命令來快速概覽應用程式的路由。

您甚至可以透過向 `resources` 方法傳遞一個陣列，一次註冊多個資源控制器：

    Route::resources([
        'photos' => PhotoController::class,
        'posts' => PostController::class,
    ]);


<a name="actions-handled-by-resource-controllers"></a>
#### 資源控制器處理的操作

<div class="overflow-auto">

| 動詞      | URI                    | 操作    | 路由名稱         |
| --------- | ---------------------- | ------- | ---------------- |
| GET       | `/photos`              | index   | photos.index     |
| GET       | `/photos/create`       | create  | photos.create    |
| POST      | `/photos`              | store   | photos.store     |
| GET       | `/photos/{photo}`      | show    | photos.show      |
| GET       | `/photos/{photo}/edit` | edit    | photos.edit      |
| PUT/PATCH | `/photos/{photo}`      | update  | photos.update    |
| DELETE    | `/photos/{photo}`      | destroy | photos.destroy   |

</div>


<a name="customizing-missing-model-behavior"></a>
#### 自訂模型遺失行為

通常，如果找不到隱式綁定的資源模型，將會產生 404 HTTP 回應。然而，您可以在定義資源路由時，呼叫 `missing` 方法來自訂此行為。`missing` 方法接受一個閉包，如果針對資源的任何路由都找不到隱式綁定的模型時，該閉包將會被調用：

    use App\Http\Controllers\PhotoController;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Redirect;

    Route::resource('photos', PhotoController::class)
        ->missing(function (Request $request) {
            return Redirect::route('photos.index');
        });


<a name="soft-deleted-models"></a>
#### 軟刪除模型

通常，隱式模型綁定不會取出已 [軟刪除](/docs/{{version}}/eloquent#soft-deleting) 的模型，而是回傳一個 404 HTTP 回應。然而，您可以在定義資源路由時，調用 `withTrashed` 方法來指示框架允許軟刪除的模型：

    use App\Http\Controllers\PhotoController;

    Route::resource('photos', PhotoController::class)->withTrashed();

呼叫不帶引數的 `withTrashed` 將允許針對 `show`、`edit` 和 `update` 資源路由的軟刪除模型。您可以透過向 `withTrashed` 方法傳遞一個陣列來指定這些路由的一個子集：

    Route::resource('photos', PhotoController::class)->withTrashed(['show']);


<a name="specifying-the-resource-model"></a>
#### 指定資源模型

如果您正在使用 [路由模型綁定](/docs/{{version}}/routing#route-model-binding) 並希望資源控制器的方法能型別提示模型實例，可以在產生控制器時使用 `--model` 選項：

```shell
php artisan make:controller PhotoController --model=Photo --resource
```


<a name="generating-form-requests"></a>
#### 產生表單請求

您可以在產生資源控制器時提供 `--requests` 選項，指示 Artisan 為控制器的儲存和更新方法產生 [表單請求類別](/docs/{{version}}/validation#form-request-validation)：

```shell
php artisan make:controller PhotoController --model=Photo --resource --requests
```


<a name="restful-partial-resource-routes"></a>
### 部分資源路由

宣告資源路由時，您可以指定控制器應處理的操作的一個子集，而不是完整的預設操作集：

    use App\Http\Controllers\PhotoController;

    Route::resource('photos', PhotoController::class)->only([
        'index', 'show'
    ]);

    Route::resource('photos', PhotoController::class)->except([
        'create', 'store', 'update', 'destroy'
    ]);


<a name="api-resource-routes"></a>
#### API 資源路由

在宣告將由 API 消費的資源路由時，您通常會想要排除呈現 HTML 範本的路由，例如 `create` 和 `edit`。為了方便，您可以使用 `apiResource` 方法來自動排除這兩個路由：

    use App\Http\Controllers\PhotoController;

    Route::apiResource('photos', PhotoController::class);

您可以透過向 `apiResources` 方法傳遞一個陣列，一次註冊多個 API 資源控制器：

    use App\Http\Controllers\PhotoController;
    use App\Http\Controllers\PostController;

    Route::apiResources([
        'photos' => PhotoController::class,
        'posts' => PostController::class,
    ]);

為了快速產生一個不包含 `create` 或 `edit` 方法的 API 資源控制器，可以在執行 `make:controller` 命令時使用 `--api` 開關：

```shell
php artisan make:controller PhotoController --api
```

<a name="restful-nested-resources"></a>
### 巢狀資源

有時候您可能需要為巢狀資源定義路由。舉例來說，一個相片資源可能有多個評論，這些評論附屬於該相片。要巢狀化資源控制器，您可以在路由宣告中使用「點」標記法 (dot notation)：

    use App\Http\Controllers\PhotoCommentController;

    Route::resource('photos.comments', PhotoCommentController::class);

此路由會註冊一個巢狀資源，可透過以下 URI 存取：

    /photos/{photo}/comments/{comment}


<a name="scoping-nested-resources"></a>
#### 限定巢狀資源範圍

Laravel 的 [隱式模型綁定](/docs/{{version}}/routing#implicit-model-binding-scoping) 功能可以自動限定巢狀綁定範圍，確保解析的子模型確實屬於父模型。透過在定義巢狀資源時使用 `scoped` 方法，您可以啟用自動範圍限定，並指示 Laravel 應使用哪個欄位來檢索子資源。更多資訊請參閱 [限定資源路由範圍](#restful-scoping-resource-routes) 的文件。


<a name="shallow-nesting"></a>
#### 淺層巢狀

通常，URI 中同時包含父層和子層 ID 並非完全必要，因為子層 ID 本身已經是唯一識別符。當您在 URI 片段中使用唯一識別符，例如自動遞增主鍵來識別模型時，您可以選擇使用「淺層巢狀」(shallow nesting)：

    use App\Http\Controllers\CommentController;

    Route::resource('photos.comments', CommentController::class)->shallow();

此路由定義將會定義以下路由：

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

預設情況下，所有資源控制器的行為都會有一個路由名稱；但是，您可以透過傳入一個包含您所需路由名稱的 `names` 陣列來覆寫這些名稱：

    use App\Http\Controllers\PhotoController;

    Route::resource('photos', PhotoController::class)->names([
        'create' => 'photos.build'
    ]);


<a name="restful-naming-resource-route-parameters"></a>
### 命名資源路由參數

預設情況下，`Route::resource` 會根據資源名稱的「單數」版本來為您的資源路由建立路由參數。您可以透過 `parameters` 方法，輕鬆地為每個資源覆寫此設定。傳入 `parameters` 方法的陣列應該是一個資源名稱與參數名稱的關聯陣列：

    use App\Http\Controllers\AdminUserController;

    Route::resource('users', AdminUserController::class)->parameters([
        'users' => 'admin_user'
    ]);

上述範例會為資源的 `show` 路由產生以下 URI：

    /users/{admin_user}


<a name="restful-scoping-resource-routes"></a>
### 限定資源路由範圍

Laravel 的 [限定範圍隱式模型綁定](/docs/{{version}}/routing#implicit-model-binding-scoping) 功能可以自動限定巢狀綁定範圍，確保解析的子模型確實屬於父模型。透過在定義巢狀資源時使用 `scoped` 方法，您可以啟用自動範圍限定，並指示 Laravel 應使用哪個欄位來檢索子資源：

    use App\Http\Controllers\PhotoCommentController;

    Route::resource('photos.comments', PhotoCommentController::class)->scoped([
        'comment' => 'slug',
    ]);

此路由會註冊一個限定範圍的巢狀資源，可透過以下 URI 存取：

    /photos/{photo}/comments/{comment:slug}

當使用自訂鍵值的隱式綁定作為巢狀路由參數時，Laravel 會自動限定查詢範圍，透過慣例猜測父模型上的關聯名稱來檢索巢狀模型。在此情況下，將會假定 `Photo` 模型具有一個名為 `comments` (路由參數名稱的複數) 的關聯，可用於檢索 `Comment` 模型。


<a name="restful-localizing-resource-uris"></a>
### 本地化資源 URI

預設情況下，`Route::resource` 會使用英文動詞和複數規則來建立資源 URI。如果您需要本地化 `create` 和 `edit` 行為動詞，您可以使用 `Route::resourceVerbs` 方法。這可以在應用程式的 `App\Providers\AppServiceProvider` 中，在 `boot` 方法的開頭完成：

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

Laravel 的複數器支援 [多種不同的語言，您可以根據需求進行配置](/docs/{{version}}/localization#pluralization-language)。一旦動詞和複數語言經過自訂，諸如 `Route::resource('publicacion', PublicacionController::class)` 的資源路由註冊將產生以下 URI：

    /publicacion/crear

    /publicacion/{publicaciones}/editar


<a name="restful-supplementing-resource-controllers"></a>
### 補充資源控制器

如果您需要在預設的資源路由集之外，為資源控制器添加額外路由，您應該在呼叫 `Route::resource` 方法之前定義這些路由；否則，`resource` 方法定義的路由可能會無意中優先於您的補充路由：

    use App\Http\Controller\PhotoController;

    Route::get('/photos/popular', [PhotoController::class, 'popular']);
    Route::resource('photos', PhotoController::class);

> [!NOTE]  
> 請記住讓您的控制器保持專注。如果您發現自己經常需要典型資源行為之外的方法，請考慮將控制器拆分為兩個較小的控制器。

<a name="singleton-resource-controllers"></a>
### 單例資源控制器

有時候，您的應用程式會有一些只能存在單一實例的資源。舉例來說，使用者「個人資料」可以被編輯或更新，但使用者不能擁有一個以上的「個人資料」。同樣地，一張圖片可能只有一個「縮圖」。這些資源被稱為「單例資源」，意味著資源只能存在一個且僅一個實例。在這些情境中，您可以註冊一個「單例」資源控制器：

```php
use App\Http\Controllers\ProfileController;
use Illuminate\Support\Facades\Route;

Route::singleton('profile', ProfileController::class);
```

上述單例資源定義將會註冊以下路由。如您所見，單例資源不會註冊「建立」路由，且註冊的路由也不接受識別碼，因為資源只能存在一個實例：

<div class="overflow-auto">

| HTTP動詞    | URI             | 行為   | 路由名稱       |
| ----------- | --------------- | ------ | -------------- |
| GET         | `/profile`      | show   | profile.show   |
| GET         | `/profile/edit` | edit   | profile.edit   |
| PUT/PATCH   | `/profile`      | update | profile.update |

</div>

單例資源也可以巢狀地定義在標準資源中：

```php
Route::singleton('photos.thumbnail', ThumbnailController::class);
```

在此範例中，`photos` 資源會接收所有 [標準資源路由](#actions-handled-by-resource-controllers)；然而，`thumbnail` 資源將會是一個具有以下路由的單例資源：

<div class="overflow-auto">

| HTTP動詞    | URI                              | 行為   | 路由名稱                |
| ----------- | -------------------------------- | ------ | ----------------------- |
| GET         | `/photos/{photo}/thumbnail`      | show   | photos.thumbnail.show   |
| GET         | `/photos/{photo}/thumbnail/edit` | edit   | photos.thumbnail.edit   |
| PUT/PATCH   | `/photos/{photo}/thumbnail`      | update | photos.thumbnail.update |

</div>


<a name="creatable-singleton-resources"></a>
#### 可建立的單例資源

有時候，您可能希望為單例資源定義建立與儲存路由。為此，您可以在註冊單例資源路由時呼叫 `creatable` 方法：

```php
Route::singleton('photos.thumbnail', ThumbnailController::class)->creatable();
```

在此範例中，將會註冊以下路由。如您所見，可建立的單例資源也會註冊 `DELETE` 路由：

<div class="overflow-auto">

| HTTP動詞    | URI                                | 行為    | 路由名稱                 |
| ----------- | ---------------------------------- | ------- | ------------------------ |
| GET         | `/photos/{photo}/thumbnail/create` | create  | photos.thumbnail.create  |
| POST        | `/photos/{photo}/thumbnail`        | store   | photos.thumbnail.store   |
| GET         | `/photos/{photo}/thumbnail`        | show    | photos.thumbnail.show    |
| GET         | `/photos/{photo}/thumbnail/edit`   | edit    | photos.thumbnail.edit    |
| PUT/PATCH   | `/photos/{photo}/thumbnail`        | update  | photos.thumbnail.update  |
| DELETE      | `/photos/{photo}/thumbnail`        | destroy | photos.thumbnail.destroy |

</div>

如果您希望 Laravel 為單例資源註冊 `DELETE` 路由，但不註冊建立或儲存路由，您可以使用 `destroyable` 方法：

```php
Route::singleton(...)->destroyable();
```


<a name="api-singleton-resources"></a>
#### API 單例資源

`apiSingleton` 方法可用於註冊一個將透過 API 進行操作的單例資源，因此使得 `create` 和 `edit` 路由變得不必要：

```php
Route::apiSingleton('profile', ProfileController::class);
```

當然，API 單例資源也可以是 `creatable` 的，這將會為該資源註冊 `store` 和 `destroy` 路由：

```php
Route::apiSingleton('photos.thumbnail', ProfileController::class)->creatable();
```

<a name="dependency-injection-and-controllers"></a>
## 依賴注入與控制器

<a name="constructor-injection"></a>
#### 建構函式注入

Laravel 的[服務容器](/docs/{{version}}/container)被用於解析所有 Laravel 控制器。因此，你可以在控制器的建構函式中型別提示任何你控制器可能需要的依賴。宣告的依賴將會被自動解析並注入到控制器實例中：

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

除了建構函式注入，你也可以在控制器的方法中型別提示依賴。一個常見的方法注入應用場景是將 `Illuminate\Http\Request` 實例注入到你的控制器方法中：

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

如果你的控制器方法同時期望接收來自路由參數的輸入，請將你的路由參數列在其他依賴之後。例如，如果你的路由定義如下：

    use App\Http\Controllers\UserController;

    Route::put('/user/{id}', [UserController::class, 'update']);

你仍然可以型別提示 `Illuminate\Http\Request`，並透過如下定義你的控制器方法來存取你的 `id` 參數：

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