# Views

- [簡介](#introduction)
    - [使用 React / Vue 編寫 View](#writing-views-in-react-or-vue)
- [建立與渲染 View](#creating-and-rendering-views)
    - [巢狀 View 目錄](#nested-view-directories)
    - [建立第一個可用的 View](#creating-the-first-available-view)
    - [判斷 View 是否存在](#determining-if-a-view-exists)
- [傳遞資料至 View](#passing-data-to-views)
    - [與所有 View 共用資料](#sharing-data-with-all-views)
- [View Composers](#view-composers)
    - [View Creators](#view-creators)
- [優化 View](#optimizing-views)

<a name="introduction"></a>
## 簡介

當然，直接從路由與控制器回傳整個 HTML 文件字串並不實際。幸運的是，View 提供了一種方便的方式，將所有 HTML 放在獨立的檔案中。

View 將控制器/應用程式邏輯與呈現邏輯分離，並儲存在 `resources/views` 目錄中。使用 Laravel 時，View 模板通常是使用 [Blade 模板語言](/docs/{{version}}/blade)編寫的。一個簡單的 View 可能會像這樣：

```blade
<!-- View stored in resources/views/greeting.blade.php -->

<html>
    <body>
        <h1>Hello, {{ $name }}</h1>
    </body>
</html>
```

由於此 View 儲存在 `resources/views/greeting.blade.php`，我們可以使用全域的 `view` 輔助函式來回傳它：

    Route::get('/', function () {
        return view('greeting', ['name' => 'James']);
    });

> [!NOTE]  
> 正在尋找更多關於如何編寫 Blade 模板的資訊嗎？請查閱完整的 [Blade 說明文件](/docs/{{version}}/blade)以開始使用。

<a name="writing-views-in-react-or-vue"></a>
### 使用 React / Vue 編寫 View

許多開發者已經開始偏好使用 React 或 Vue 來編寫前端模板，而不是透過 Blade 以 PHP 編寫。多虧了 [Inertia](https://inertiajs.com/)，Laravel 讓這一切變得輕而易舉。Inertia 是一個函式庫，可以輕鬆地將 React / Vue 前端與 Laravel 後端連結起來，而無需傳統上建構 SPA 的複雜性。

我們的 Breeze 和 Jetstream [入門套件](/docs/{{version}}/starter-kits)為您下一個由 Inertia 驅動的 Laravel 應用程式提供了絕佳的起點。此外，[Laravel Bootcamp](https://bootcamp.laravel.com) 提供了建構由 Inertia 驅動的 Laravel 應用程式的完整示範，包括 Vue 和 React 的範例。

<a name="creating-and-rendering-views"></a>
## 建立與渲染 View

您可以透過在應用程式的 `resources/views` 目錄中放置一個副檔名為 `.blade.php` 的檔案來建立 View，或者使用 `make:view` Artisan 命令：

```shell
php artisan make:view greeting
```

`.blade.php` 副檔名會告知框架該檔案包含一個 [Blade 模板](/docs/{{version}}/blade)。Blade 模板包含 HTML 以及 Blade 指令，讓您可以輕鬆地輸出值、建立「if」陳述式、迭代資料等等。

建立 View 後，您可以使用全域的 `view` 輔助函式從應用程式的路由或控制器回傳它：

    Route::get('/', function () {
        return view('greeting', ['name' => 'James']);
    });

View 也可以使用 `View` Facade 回傳：

    use Illuminate\Support\Facades\View;

    return View::make('greeting', ['name' => 'James']);

如您所見，傳遞給 `view` 輔助函式的第一個引數對應於 `resources/views` 目錄中 View 檔案的名稱。第二個引數是一個應提供給 View 的資料陣列。在此範例中，我們傳遞了 `name` 變數，該變數使用 [Blade 語法](/docs/{{version}}/blade)在 View 中顯示。

<a name="nested-view-directories"></a>
### 巢狀 View 目錄

View 也可以巢狀地放置在 `resources/views` 目錄的子目錄中。可以使用「點」符號來引用巢狀 View。例如，如果您的 View 儲存在 `resources/views/admin/profile.blade.php`，您可以從應用程式的路由/控制器中這樣回傳它：

    return view('admin.profile', $data);

> [!WARNING]  
> View 目錄名稱不應包含 `.` 字元。

<a name="creating-the-first-available-view"></a>
### 建立第一個可用的 View

使用 `View` Facade 的 `first` 方法，您可以建立給定 View 陣列中存在的第一個 View。如果您的應用程式或套件允許自訂或覆寫 View，這可能會很有用：

    use Illuminate\Support\Facades\View;

    return View::first(['custom.admin', 'admin'], $data);

<a name="determining-if-a-view-exists"></a>
### 判斷 View 是否存在

如果您需要判斷 View 是否存在，可以使用 `View` Facade。如果 View 存在，`exists` 方法將回傳 `true`：

    use Illuminate\Support\Facades\View;

    if (View::exists('admin.profile')) {
        // ...
    }

<a name="passing-data-to-views"></a>
## 傳遞資料至 View

如您在前面的範例中所見，您可以將資料陣列傳遞給 View，以使該資料可供 View 使用：

    return view('greetings', ['name' => 'Victoria']);

以這種方式傳遞資訊時，資料應該是鍵/值對的陣列。將資料提供給 View 後，您可以使用資料的鍵在 View 中存取每個值，例如 `<?php echo $name; ?>`。

作為將完整資料陣列傳遞給 `view` 輔助函式的替代方案，您可以使用 `with` 方法向 View 添加單獨的資料片段。`with` 方法會回傳 View 物件的實例，以便您可以在回傳 View 之前繼續鏈式呼叫方法：

    return view('greeting')
        ->with('name', 'Victoria')
        ->with('occupation', 'Astronaut');

<a name="sharing-data-with-all-views"></a>
### 與所有 View 共用資料

有時，您可能需要與應用程式渲染的所有 View 共用資料。您可以使用 `View` Facade 的 `share` 方法來實現。通常，您應該將 `share` 方法的呼叫放在 Service Provider 的 `boot` 方法中。您可以將它們添加到 `App\Providers\AppServiceProvider` 類別中，或者生成一個單獨的 Service Provider 來容納它們：

    <?php

    namespace App\Providers;

    use Illuminate\Support\Facades\View;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Register any application services.
         */
        public function register(): void
        {
            // ...
        }

        /**
         * Bootstrap any application services.
         */
        public function boot(): void
        {
            View::share('key', 'value');
        }
    }

<a name="view-composers"></a>
## View Composers

View Composers 是在 View 渲染時呼叫的回呼函式或類別方法。如果您有希望每次渲染 View 時都綁定到 View 的資料，View Composer 可以幫助您將該邏輯組織到一個位置。如果應用程式中的多個路由或控制器回傳相同的 View，並且始終需要特定的資料片段，View Composers 可能會特別有用。

通常，View Composers 會在應用程式的 [Service Provider](/docs/{{version}}/providers) 之一中註冊。在此範例中，我們假設 `App\Providers\AppServiceProvider` 將包含此邏輯。

我們將使用 `View` Facade 的 `composer` 方法來註冊 View Composer。Laravel 不包含基於類別的 View Composers 的預設目錄，因此您可以隨意組織它們。例如，您可以建立一個 `app/View/Composers` 目錄來容納應用程式的所有 View Composers：

    <?php

    namespace App\Providers;

    use App\View\Composers\ProfileComposer;
    use Illuminate\Support\Facades;
    use Illuminate\Support\ServiceProvider;
    use Illuminate\View\View;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Register any application services.
         */
        public function register(): void
        {
            // ...
        }

        /**
         * Bootstrap any application services.
         */
        public function boot(): void
        {
            // Using class based composers...
            Facades\View::composer('profile', ProfileComposer::class);

            // Using closure based composers...
            Facades\View::composer('welcome', function (View $view) {
                // ...
            });

            Facades\View::composer('dashboard', function (View $view) {
                // ...
            });
        }
    }

現在我們已經註冊了 Composer，`App\View\Composers\ProfileComposer` 類別的 `compose` 方法將在每次渲染 `profile` View 時執行。讓我們看看 Composer 類別的範例：

    <?php

    namespace App\View\Composers;

    use App\Repositories\UserRepository;
    use Illuminate\View\View;

    class ProfileComposer
    {
        /**
         * Create a new profile composer.
         */
        public function __construct(
            protected UserRepository $users,
        ) {}

        /**
         * Bind data to the view.
         */
        public function compose(View $view): void
        {
            $view->with('count', $this->users->count());
        }
    }

如您所見，所有 View Composers 都透過 [Service Container](/docs/{{version}}/container) 解析，因此您可以在 Composer 的建構函式中型別提示您需要的任何依賴項。

<a name="attaching-a-composer-to-multiple-views"></a>
#### 將 Composer 附加到多個 View

您可以透過將 View 陣列作為第一個引數傳遞給 `composer` 方法，一次將 View Composer 附加到多個 View：

    use App\Views\Composers\MultiComposer;
    use Illuminate\Support\Facades\View;

    View::composer(
        ['profile', 'dashboard'],
        MultiComposer::class
    );

`composer` 方法也接受 `*` 字元作為萬用字元，允許您將 Composer 附加到所有 View：

    use Illuminate\Support\Facades;
    use Illuminate\View\View;

    Facades\View::composer('*', function (View $view) {
        // ...
    });

<a name="view-creators"></a>
### View Creators

View 「Creators」與 View Composers 非常相似；但是，它們是在 View 實例化後立即執行，而不是等到 View 即將渲染時才執行。要註冊 View Creator，請使用 `creator` 方法：

    use App\View\Creators\ProfileCreator;
    use Illuminate\Support\Facades\View;

    View::creator('profile', ProfileCreator::class);

<a name="optimizing-views"></a>
## 優化 View

預設情況下，Blade 模板 View 是按需編譯的。當執行渲染 View 的請求時，Laravel 將判斷 View 的編譯版本是否存在。如果檔案存在，Laravel 將判斷未編譯的 View 是否比編譯的 View 更晚修改。如果編譯的 View 不存在，或者未編譯的 View 已被修改，Laravel 將重新編譯 View。

在請求期間編譯 View 可能會對效能產生輕微的負面影響，因此 Laravel 提供了 `view:cache` Artisan 命令來預編譯應用程式使用的所有 View。為了提高效能，您可能希望在部署過程中執行此命令：

```shell
php artisan view:cache
```

您可以使用 `view:clear` 命令來清除 View 快取：

```shell
php artisan view:clear
```
