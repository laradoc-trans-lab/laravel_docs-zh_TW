# 視圖 (Views)

- [簡介](#introduction)
    - [在 React / Svelte / Vue 中撰寫視圖](#writing-views-in-react-svelte-or-vue)
- [建立與渲染視圖](#creating-and-rendering-views)
    - [巢狀視圖目錄](#nested-view-directories)
    - [建立第一個可用的視圖](#creating-the-first-available-view)
    - [確認視圖是否存在](#determining-if-a-view-exists)
- [傳遞資料給視圖](#passing-data-to-views)
    - [與所有視圖共享資料](#sharing-data-with-all-views)
- [View Composers](#view-composers)
    - [View Creators](#view-creators)
- [最佳化視圖](#optimizing-views)

<a name="introduction"></a>
## 簡介

當然，直接從路由與控制器回傳完整的 HTML 文件字串是不切實際的。幸好，視圖 (views) 提供了一種方便的方式，將我們所有的 HTML 放在獨立的文件中。

視圖將您的控制器 / 應用程式邏輯與呈現邏輯分離，並儲存在 `resources/views` 目錄中。使用 Laravel 時，視圖模板通常使用 [Blade 模板語言](/docs/{{version}}/blade) 編寫。一個簡單的視圖可能看起來像這樣：

```blade
<!-- View stored in resources/views/greeting.blade.php -->

<html>
    <body>
        <h1>Hello, {{ $name }}</h1>
    </body>
</html>
```

由於此視圖儲存在 `resources/views/greeting.blade.php`，我們可以使用全域的 `view` 輔助函式來回傳它，如下所示：

```php
Route::get('/', function () {
    return view('greeting', ['name' => 'James']);
});
```

> [!NOTE]
> 正在尋找更多關於如何編寫 Blade 模板的資訊嗎？請查看完整的 [Blade 文件](/docs/{{version}}/blade) 來開始。


<a name="writing-views-in-react-svelte-or-vue"></a>
### 在 React / Svelte / Vue 中撰寫視圖

許多開發者不再透過 Blade 在 PHP 中編寫前端模板，而是開始偏好使用 React、Svelte 或 Vue。Laravel 透過 [Inertia](https://inertiajs.com/) 讓這件事變得輕而易舉。Inertia 是一個函式庫，讓您可以輕易地將 React / Svelte / Vue 前端與 Laravel 後端聯繫起來，而無需面對構建 SPA 時常見的複雜性。

我們的 [React, Svelte, and Vue 應用程式入門套件](/docs/{{version}}/starter-kits) 為您下一個由 Inertia 驅動的 Laravel 應用程式提供了一個絕佳的起點。


<a name="creating-and-rendering-views"></a>
## 建立與渲染視圖

您可以透過在應用程式的 `resources/views` 目錄中放置一個副檔名為 `.blade.php` 的檔案，或者使用 `make:view` Artisan 指令來建立視圖：

```shell
php artisan make:view greeting
```

`.blade.php` 副檔名會通知框架該檔案包含 [Blade 模板](/docs/{{version}}/blade)。Blade 模板包含 HTML 以及 Blade 指令，讓您可以輕鬆地印出 (echo) 數值、建立 "if" 語句、對資料進行迭代等等。

建立視圖後，您可以使用全域的 `view` 輔助函式從應用程式的路由或控制器之一回傳它：

```php
Route::get('/', function () {
    return view('greeting', ['name' => 'James']);
});
```

視圖也可以使用 `View` Facade 回傳：

```php
use Illuminate\Support\Facades\View;

return View::make('greeting', ['name' => 'James']);
```

如您所見，傳遞給 `view` 輔助函式的第一個參數對應於 `resources/views` 目錄中視圖檔案的名稱。第二個參數是應提供給視圖使用的資料陣列。在這種情況下，我們傳遞了 `name` 變數，並在視圖中使用 [Blade 語法](/docs/{{version}}/blade) 顯示。


<a name="nested-view-directories"></a>
### 巢狀視圖目錄

視圖也可以巢狀放置在 `resources/views` 目錄的子目錄中。可以使用「點 (dot)」記法來引用巢狀視圖。例如，如果您的視圖儲存在 `resources/views/admin/profile.blade.php`，您可以像這樣從應用程式的路由 / 控制器之一回傳它：

```php
return view('admin.profile', $data);
```

> [!WARNING]
> 視圖目錄名稱不應包含 `.` 字元。


<a name="creating-the-first-available-view"></a>
### 建立第一個可用的視圖

使用 `View` Facade 的 `first` 方法，您可以建立給定視圖陣列中第一個存在的視圖。如果您的應用程式或套件允許自訂或覆寫視圖，這會非常有用：

```php
use Illuminate\Support\Facades\View;

return View::first(['custom.admin', 'admin'], $data);
```


<a name="determining-if-a-view-exists"></a>
### 確認視圖是否存在

如果您需要確定視圖是否存在，可以使用 `View` Facade。如果視圖存在，`exists` 方法將回傳 `true`：

```php
use Illuminate\Support\Facades\View;

if (View::exists('admin.profile')) {
    // ...
}
```


<a name="passing-data-to-views"></a>
## 傳遞資料給視圖

正如您在先前的範例中所看到的，您可以將資料陣列傳遞給視圖，以使該資料在視圖中可用：

```php
return view('greetings', ['name' => 'Victoria']);
```

以這種方式傳遞資訊時，資料應為具有鍵 (key) / 值 (value) 對的陣列。將資料提供給視圖後，您就可以使用資料的鍵在視圖中存取每個值，例如 `<?php echo $name; ?>`。

除了將完整的資料陣列傳遞給 `view` 輔助函式之外，您還可以使用 `with` 方法將單個資料片段新增到視圖中。`with` 方法會回傳視圖物件的實例，因此您可以在回傳視圖之前繼續串接方法：

```php
return view('greeting')
    ->with('name', 'Victoria')
    ->with('occupation', 'Astronaut');
```


<a name="sharing-data-with-all-views"></a>
### 與所有視圖共享資料

有時，您可能需要與應用程式渲染的所有視圖共享資料。您可以使用 `View` Facade 的 `share` 方法來達成此目的。通常，您應該將 `share` 方法的呼叫放在服務提供者 (service provider) 的 `boot` 方法中。您可以自由地將它們新增到 `App\Providers\AppServiceProvider` 類別中，或者產生一個獨立的服務提供者來放置它們：

```php
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
```

<a name="view-composers"></a>
## View Composers

View composers 是在渲染視圖時呼叫的回呼 (Callback) 或類別方法。如果您希望每次渲染視圖時都將資料與該視圖綁定，View composer 可以幫助您將這些邏輯組織到單一位置。如果您的應用程式中的多個路由或控制器傳回相同的視圖且始終需要特定的資料，View composer 就特別有用。

通常，View composer 會在應用程式的其中一個 [服務提供者 (Service Providers)](/docs/{{version}}/providers) 中註冊。在此範例中，我們假設 `App\Providers\AppServiceProvider` 將包含此邏輯。

我們將使用 `View` Facade 的 `composer` 方法來註冊 View composer。Laravel 並未包含類別型 View composer 的預設目錄，因此您可以隨意組織它們。例如，您可以建立一個 `app/View/Composers` 目錄來存放應用程式所有的 View composer：

```php
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
        // Using class-based composers...
        Facades\View::composer('profile', ProfileComposer::class);

        // Using closure-based composers...
        Facades\View::composer('welcome', function (View $view) {
            // ...
        });

        Facades\View::composer('dashboard', function (View $view) {
            // ...
        });
    }
}
```

現在我們已經註冊了 Composer，每當渲染 `profile` 視圖時，都會執行 `App\View\Composers\ProfileComposer` 類別的 `compose` 方法。讓我們來看一個 Composer 類別的範例：

```php
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
```

如您所見，所有的 View composer 都是經由 [服務容器 (Service Container)](/docs/{{version}}/container) 解析的，因此您可以在 Composer 的建構子中對所需的任何依賴項進行型別提示 (Type-hint)。


<a name="attaching-a-composer-to-multiple-views"></a>
#### 將 Composer 附加到多個視圖

您可以透過將視圖陣列作為第一個參數傳遞給 `composer` 方法，一次將 View composer 附加到多個視圖：

```php
use App\Views\Composers\MultiComposer;
use Illuminate\Support\Facades\View;

View::composer(
    ['profile', 'dashboard'],
    MultiComposer::class
);
```

`composer` 方法也接受 `*` 字元作為萬用字元，讓您可以將 Composer 附加到所有視圖：

```php
use Illuminate\Support\Facades;
use Illuminate\View\View;

Facades\View::composer('*', function (View $view) {
    // ...
});
```


<a name="view-creators"></a>
### View Creators

View "creators" 與 View composer 非常相似；然而，它們是在視圖實例化後立即執行，而不是等到視圖即將渲染時。要註冊 View creator，請使用 `creator` 方法：

```php
use App\View\Creators\ProfileCreator;
use Illuminate\Support\Facades\View;

View::creator('profile', ProfileCreator::class);
```


<a name="optimizing-views"></a>
## 最佳化視圖

預設情況下，Blade 模板視圖是按需編譯的。當執行渲染視圖的請求時，Laravel 會判斷是否存在該視圖的編譯版本。如果檔案存在，Laravel 接著會判斷未編譯的視圖是否比已編譯的視圖更晚修改過。如果編譯後的視圖不存在，或者未編譯的視圖已被修改，Laravel 將重新編譯該視圖。

在請求期間編譯視圖可能會對效能產生微小的負面影響，因此 Laravel 提供了 `view:cache` Artisan 指令來預先編譯應用程式使用的所有視圖。為了提高效能，您可能希望將此指令作為部署程序的一部分執行：

```shell
php artisan view:cache
```

您可以使用 `view:clear` 指令來清除視圖快取：

```shell
php artisan view:clear
```