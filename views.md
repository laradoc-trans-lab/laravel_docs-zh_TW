# 視圖

- [簡介](#introduction)
    - [在 React / Vue 中編寫視圖](#writing-views-in-react-or-vue)
- [建立與渲染視圖](#creating-and-rendering-views)
    - [巢狀視圖目錄](#nested-view-directories)
    - [建立第一個可用的視圖](#creating-the-first-available-view)
    - [判斷視圖是否存在](#determining-if-a-view-exists)
- [傳遞資料至視圖](#passing-data-to-views)
    - [與所有視圖共享資料](#sharing-data-with-all-views)
- [視圖合成器](#view-composers)
    - [視圖建立器](#view-creators)
- [優化視圖](#optimizing-views)

<a name="introduction"></a>
## 簡介

當然，直接從路由與控制器回傳完整的 HTML 文件字串是不切實際的。幸好，視圖提供了一個方便的方法，將所有 HTML 放在獨立的檔案中。

視圖將您的控制器 / 應用程式邏輯與您的呈現邏輯分離，並儲存在 `resources/views` 目錄中。使用 Laravel 時，視圖模板通常是使用 [Blade 模板引擎](/docs/{{version}}/blade) 編寫的。一個簡單的視圖可能看起來像這樣：

```blade
<!-- View stored in resources/views/greeting.blade.php -->

<html>
    <body>
        <h1>Hello, {{ $name }}</h1>
    </body>
</html>
```

由於此視圖儲存在 `resources/views/greeting.blade.php`，我們可以像這樣使用全域的 `view` 輔助函式來回傳它：

    Route::get('/', function () {
        return view('greeting', ['name' => 'James']);
    });

> [!NOTE]  
> 正在尋找更多關於如何編寫 Blade 模板的資訊嗎？請查閱完整的 [Blade 文件](/docs/{{version}}/blade) 以開始使用。

<a name="writing-views-in-react-or-vue"></a>
### 在 React / Vue 中編寫視圖

許多開發人員不再透過 Blade 使用 PHP 編寫前端模板，而是開始偏好使用 React 或 Vue 編寫他們的模板。Laravel 透過 [Inertia](https://inertiajs.com/) 讓這過程變得輕鬆，Inertia 是一個函式庫，它能讓您的 React / Vue 前端與您的 Laravel 後端輕鬆連結，而無需處理建構 SPA 的典型複雜性。

我們的 Breeze 和 Jetstream [入門套件](/docs/{{version}}/starter-kits) 為您下一個由 Inertia 驅動的 Laravel 應用程式提供了絕佳的起點。此外，[Laravel Bootcamp](https://bootcamp.laravel.com) 提供了建構由 Inertia 驅動的 Laravel 應用程式的完整示範，包括 Vue 和 React 的範例。

<a name="creating-and-rendering-views"></a>
## 建立與渲染視圖

您可以透過在應用程式的 `resources/views` 目錄中放置一個副檔名為 `.blade.php` 的檔案來建立視圖，或者使用 `make:view` Artisan 命令：

```shell
php artisan make:view greeting
```

`.blade.php` 副檔名會告知框架該檔案包含 [Blade 模板](/docs/{{version}}/blade)。Blade 模板包含 HTML 以及 Blade 指令，讓您可以輕鬆地印出數值、建立「if」語句、疊代資料等等。

建立視圖後，您可以使用全域的 `view` 輔助函式從應用程式的路由或控制器中回傳它：

    Route::get('/', function () {
        return view('greeting', ['name' => 'James']);
    });

視圖也可以使用 `View` 外觀來回傳：

    use Illuminate\Support\Facades\View;

    return View::make('greeting', ['name' => 'James']);

如您所見，傳遞給 `view` 輔助函式的第一個引數對應於 `resources/views` 目錄中的視圖檔案名稱。第二個引數是一個應提供給視圖使用的資料陣列。在本例中，我們傳遞了 `name` 變數，該變數使用 [Blade 語法](/docs/{{version}}/blade) 在視圖中顯示。

<a name="nested-view-directories"></a>
### 巢狀視圖目錄

視圖也可以巢狀地儲存在 `resources/views` 目錄的子目錄中。可以使用「點」表示法來參照巢狀視圖。例如，如果您的視圖儲存在 `resources/views/admin/profile.blade.php`，您可以像這樣從應用程式的路由 / 控制器中回傳它：

    return view('admin.profile', $data);

> [!WARNING]  
> 視圖目錄名稱不應包含 `.` 字元。

<a name="creating-the-first-available-view"></a>
### 建立第一個可用的視圖

使用 `View` 外觀的 `first` 方法，您可以建立在給定視圖陣列中存在的第一個視圖。如果您的應用程式或套件允許視圖被客製化或覆寫，這可能會很有用：

    use Illuminate\Support\Facades\View;

    return View::first(['custom.admin', 'admin'], $data);

<a name="determining-if-a-view-exists"></a>
### 判斷視圖是否存在

如果您需要判斷視圖是否存在，可以使用 `View` 外觀。如果視圖存在，`exists` 方法將回傳 `true`：

    use Illuminate\Support\Facades\View;

    if (View::exists('admin.profile')) {
        // ...
    }

<a name="passing-data-to-views"></a>
## 傳遞資料至視圖

如您在先前範例中所見，您可以將資料陣列傳遞給視圖，使該資料可用於視圖：

    return view('greetings', ['name' => 'Victoria']);

以這種方式傳遞資訊時，資料應該是帶有鍵 / 值對的陣列。將資料提供給視圖後，您可以使用資料的鍵來存取視圖中的每個值，例如 `<?php echo $name; ?>`。

作為將完整的資料陣列傳遞給 `view` 輔助函式的替代方案，您可以使用 `with` 方法將個別資料片段新增到視圖中。`with` 方法會回傳視圖物件的實例，以便您可以在回傳視圖之前繼續鏈式呼叫方法：

    return view('greeting')
        ->with('name', 'Victoria')
        ->with('occupation', 'Astronaut');

<a name="sharing-data-with-all-views"></a>
### 與所有視圖共享資料

有時，您可能需要與應用程式渲染的所有視圖共享資料。您可以使用 `View` 外觀的 `share` 方法來做到這一點。通常，您應該將對 `share` 方法的呼叫放在服務提供者的 `boot` 方法中。您可以將它們添加到 `App\Providers\AppServiceProvider` 類別中，或生成一個單獨的服務提供者來包含它們：

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
## 視圖合成器

視圖合成器是在視圖被渲染時呼叫的回呼或類別方法。如果您有希望在每次視圖被渲染時綁定到視圖的資料，視圖合成器可以幫助您將該邏輯組織到單一位置。如果相同的視圖由應用程式內的多個路由或控制器返回，並且總是需要特定的資料，那麼視圖合成器可能會特別有用。

通常，視圖合成器會在您應用程式的其中一個 [服務提供者](/docs/{{version}}/providers) 中註冊。在此範例中，我們假設 `App\Providers\AppServiceProvider` 將包含此邏輯。

我們將使用 `View` 門面的 `composer` 方法來註冊視圖合成器。Laravel 不包含基於類別的視圖合成器的預設目錄，因此您可以隨心所欲地組織它們。例如，您可以建立 `app/View/Composers` 目錄來存放您應用程式的所有視圖合成器：

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

在我們註冊了合成器之後，`App\View\Composers\ProfileComposer` 類別的 `compose` 方法將會在每次 `profile` 視圖被渲染時執行。讓我們看看合成器類別的範例：

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

如您所見，所有視圖合成器都是透過 [服務容器](/docs/{{version}}/container) 解析的，因此您可以在合成器的建構子中型別提示 (type-hint) 任何您需要的依賴。

<a name="attaching-a-composer-to-multiple-views"></a>
#### 將合成器綁定到多個視圖

您可以透過將視圖陣列作為第一個參數傳遞給 `composer` 方法，一次將視圖合成器綁定到多個視圖：

    use App\Views\Composers\MultiComposer;
    use Illuminate\Support\Facades\View;

    View::composer(
        ['profile', 'dashboard'],
        MultiComposer::class
    );

`composer` 方法也接受 `*` 字元作為萬用字元，讓您可以將合成器綁定到所有視圖：

    use Illuminate\Support\Facades;
    use Illuminate\View\View;

    Facades\View::composer('*', function (View $view) {
        // ...
    });

<a name="view-creators"></a>
### 視圖建立器

視圖「建立器」與視圖合成器非常相似；然而，它們在視圖實例化後立即執行，而不是等待視圖即將渲染。要註冊視圖建立器，請使用 `creator` 方法：

    use App\View\Creators\ProfileCreator;
    use Illuminate\Support\Facades\View;

    View::creator('profile', ProfileCreator::class);

<a name="optimizing-views"></a>
## 優化視圖

預設情況下，Blade 模板視圖是按需編譯的。當執行一個渲染視圖的請求時，Laravel 將判斷視圖的編譯版本是否存在。如果檔案存在，Laravel 將接著判斷未編譯的視圖是否比已編譯的視圖更新。如果編譯的視圖不存在，或者未編譯的視圖已被修改，Laravel 將重新編譯該視圖。

在請求期間編譯視圖可能會對性能產生輕微的負面影響，因此 Laravel 提供了 `view:cache` Artisan 命令，用於預先編譯應用程式使用的所有視圖。為了提高性能，您可能希望在部署過程中執行此命令：

```shell
php artisan view:cache
```

您可以使用 `view:clear` 命令來清除視圖快取：

```shell
php artisan view:clear
```