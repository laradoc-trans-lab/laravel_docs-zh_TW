# 視圖 (Views)

- [簡介](#introduction)
    - [在 React / Vue 中撰寫視圖](#writing-views-in-react-or-vue)
- [建立與渲染視圖](#creating-and-rendering-views)
    - [巢狀視圖目錄](#nested-view-directories)
    - [建立第一個可用的視圖](#creating-the-first-available-view)
    - [判斷視圖是否存在](#determining-if-a-view-exists)
- [傳遞資料至視圖](#passing-data-to-views)
    - [與所有視圖共用資料](#sharing-data-with-all-views)
- [視圖合成器 (View Composers)](#view-composers)
    - [視圖建立器 (View Creators)](#view-creators)
- [優化視圖](#optimizing-views)

<a name="introduction"></a>
## 簡介

當然，直接從路由和控制器回傳整個 HTML 文件字串是不切實際的。幸運的是，視圖提供了一種方便的方式，將我們所有的 HTML 放在獨立的檔案中。

視圖將您的控制器/應用程式邏輯與您的呈現邏輯分離，並儲存在 `resources/views` 目錄中。使用 Laravel 時，視圖模板通常是使用 [Blade 模板引擎](/docs/{{version}}/blade)撰寫的。一個簡單的視圖可能看起來像這樣：

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
> 正在尋找更多關於如何撰寫 Blade 模板的資訊嗎？請查閱完整的 [Blade 說明文件](/docs/{{version}}/blade)以開始使用。

<a name="writing-views-in-react-or-vue"></a>
### 在 React / Vue 中撰寫視圖

許多開發者不再透過 Blade 以 PHP 撰寫前端模板，而是開始偏好使用 React 或 Vue 撰寫模板。Laravel 透過 [Inertia](https://inertiajs.com/) 讓這一切變得輕鬆，Inertia 是一個讓您的 React / Vue 前端與 Laravel 後端輕鬆連結的函式庫，而無需傳統上建構 SPA 的複雜性。

我們的 [React 和 Vue 應用程式入門套件](/docs/{{version}}/starter-kits)為您下一個由 Inertia 驅動的 Laravel 應用程式提供了絕佳的起點。

<a name="creating-and-rendering-views"></a>
## 建立與渲染視圖

您可以透過在應用程式的 `resources/views` 目錄中放置一個副檔名為 `.blade.php` 的檔案來建立視圖，或者使用 `make:view` Artisan 命令：

```shell
php artisan make:view greeting
```

`.blade.php` 副檔名告知框架該檔案包含一個 [Blade 模板](/docs/{{version}}/blade)。Blade 模板包含 HTML 以及 Blade 指令，讓您可以輕鬆地輸出值、建立「if」陳述式、迭代資料等等。

一旦您建立了視圖，就可以使用全域的 `view` 輔助函式從應用程式的路由或控制器中回傳它：

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

如您所見，傳遞給 `view` 輔助函式的第一個引數對應於 `resources/views` 目錄中視圖檔案的名稱。第二個引數是一個陣列，其中包含應提供給視圖的資料。在本例中，我們傳遞了 `name` 變數，該變數使用 [Blade 語法](/docs/{{version}}/blade)顯示在視圖中。

<a name="nested-view-directories"></a>
### 巢狀視圖目錄

視圖也可以巢狀地放置在 `resources/views` 目錄的子目錄中。可以使用「點」符號來引用巢狀視圖。例如，如果您的視圖儲存在 `resources/views/admin/profile.blade.php`，您可以從應用程式的路由/控制器中回傳它，如下所示：

```php
return view('admin.profile', $data);
```

> [!WARNING]
> 視圖目錄名稱不應包含 `.` 字元。

<a name="creating-the-first-available-view"></a>
### 建立第一個可用的視圖

使用 `View` Facade 的 `first` 方法，您可以建立給定視圖陣列中存在的第一個視圖。這在您的應用程式或套件允許自訂或覆寫視圖時可能很有用：

```php
use Illuminate\Support\Facades\View;

return View::first(['custom.admin', 'admin'], $data);
```

<a name="determining-if-a-view-exists"></a>
### 判斷視圖是否存在

如果您需要判斷視圖是否存在，可以使用 `View` Facade。如果視圖存在，`exists` 方法將回傳 `true`：

```php
use Illuminate\Support\Facades\View;

if (View::exists('admin.profile')) {
    // ...
}
```

<a name="passing-data-to-views"></a>
## 傳遞資料至視圖

如您在前面的範例中所見，您可以將資料陣列傳遞給視圖，以使該資料可供視圖使用：

```php
return view('greetings', ['name' => 'Victoria']);
```

以這種方式傳遞資訊時，資料應該是一個包含鍵/值對的陣列。將資料提供給視圖後，您可以使用資料的鍵在視圖中存取每個值，例如 `<?php echo $name; ?>`。

作為將完整的資料陣列傳遞給 `view` 輔助函式的替代方案，您可以使用 `with` 方法向視圖添加單個資料。`with` 方法回傳視圖物件的實例，以便您可以在回傳視圖之前繼續鏈式呼叫方法：

```php
return view('greeting')
    ->with('name', 'Victoria')
    ->with('occupation', 'Astronaut');
```

<a name="sharing-data-with-all-views"></a>
### 與所有視圖共用資料

有時，您可能需要與應用程式渲染的所有視圖共用資料。您可以使用 `View` Facade 的 `share` 方法來實現。通常，您應該將 `share` 方法的呼叫放在服務提供者 (Service Provider) 的 `boot` 方法中。您可以將它們添加到 `App\Providers\AppServiceProvider` 類別中，或者生成一個單獨的服務提供者來容納它們：

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
## 視圖合成器 (View Composers)

視圖合成器是在視圖渲染時呼叫的回呼函式或類別方法。如果您有希望每次渲染視圖時都綁定到該視圖的資料，視圖合成器可以幫助您將該邏輯組織到一個位置。如果應用程式中的多個路由或控制器回傳相同的視圖，並且該視圖始終需要特定的資料，那麼視圖合成器可能會特別有用。

通常，視圖合成器將在應用程式的 [服務提供者](/docs/{{version}}/providers)之一中註冊。在本例中，我們假設 `App\Providers\AppServiceProvider` 將包含此邏輯。

我們將使用 `View` Facade 的 `composer` 方法來註冊視圖合成器。Laravel 不包含基於類別的視圖合成器的預設目錄，因此您可以隨意組織它們。例如，您可以建立一個 `app/View/Composers` 目錄來容納應用程式的所有視圖合成器：

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

現在我們已經註冊了合成器，每次渲染 `profile` 視圖時，`App\View\Composers\ProfileComposer` 類別的 `compose` 方法都將被執行。讓我們看看合成器類別的範例：

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

如您所見，所有視圖合成器都透過 [服務容器](/docs/{{version}}/container)解析，因此您可以在合成器的建構函式中型別提示您需要的任何依賴項。

<a name="attaching-a-composer-to-multiple-views"></a>
#### 將合成器附加到多個視圖

您可以透過將視圖陣列作為 `composer` 方法的第一個引數傳遞，一次將視圖合成器附加到多個視圖：

```php
use App\Views\Composers\MultiComposer;
use Illuminate\Support\Facades\View;

View::composer(
    ['profile', 'dashboard'],
    MultiComposer::class
);
```

`composer` 方法也接受 `*` 字元作為萬用字元，允許您將合成器附加到所有視圖：

```php
use Illuminate\Support\Facades;
use Illuminate\View\View;

Facades\View::composer('*', function (View $view) {
    // ...
});
```

<a name="view-creators"></a>
### 視圖建立器 (View Creators)

視圖「建立器」與視圖合成器非常相似；但是，它們在視圖實例化後立即執行，而不是等到視圖即將渲染時才執行。要註冊視圖建立器，請使用 `creator` 方法：

```php
use App\View\Creators\ProfileCreator;
use Illuminate\Support\Facades\View;

View::creator('profile', ProfileCreator::class);
```

<a name="optimizing-views"></a>
## 優化視圖

預設情況下，Blade 模板視圖是按需編譯的。當執行渲染視圖的請求時，Laravel 將判斷視圖的編譯版本是否存在。如果檔案存在，Laravel 將判斷未編譯的視圖是否比編譯的視圖更新。如果編譯的視圖不存在，或者未編譯的視圖已被修改，Laravel 將重新編譯視圖。

在請求期間編譯視圖可能會對效能產生輕微的負面影響，因此 Laravel 提供了 `view:cache` Artisan 命令來預編譯應用程式使用的所有視圖。為了提高效能，您可能希望在部署過程中執行此命令：

```shell
php artisan view:cache
```

您可以使用 `view:clear` 命令來清除視圖快取：

```shell
php artisan view:clear
```

