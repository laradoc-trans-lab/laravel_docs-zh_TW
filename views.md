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

當然，直接從您的路由和控制器傳回完整的 HTML 文件字串是不切實際的。幸運的是，視圖提供了一種方便的方式，將所有 HTML 放在單獨的檔案中。

視圖將您的控制器 / 應用程式邏輯與您的呈現邏輯分離，並儲存在 `resources/views` 目錄中。使用 Laravel 時，視圖模板通常是使用 [Blade 模板語言](/docs/{{version}}/blade)編寫的。一個簡單的視圖可能看起來像這樣：

```blade
<!-- View stored in resources/views/greeting.blade.php -->

<html>
    <body>
        <h1>Hello, {{ $name }}</h1>
    </body>
</html>
```

由於此視圖儲存在 `resources/views/greeting.blade.php`，我們可以使用全域 `view` 輔助函式來傳回它，如下所示：

```php
Route::get('/', function () {
    return view('greeting', ['name' => 'James']);
});
```

> [!NOTE]
> 正在尋找有關如何編寫 Blade 模板的更多資訊？請參閱完整的 [Blade 說明文件](/docs/{{version}}/blade)以開始使用。


<a name="writing-views-in-react-or-vue"></a>
### 在 React / Vue 中編寫視圖

許多開發人員已開始偏好使用 React 或 Vue 編寫其前端模板，而非透過 Blade 在 PHP 中編寫。多虧了 [Inertia](https://inertiajs.com/)，Laravel 使這一切變得輕而易舉，Inertia 是一個函式庫，可讓您輕鬆將 React / Vue 前端與 Laravel 後端繫結，而無需建立 SPA 的典型複雜性。

我們的 [React 和 Vue 應用程式入門套件](/docs/{{version}}/starter-kits)為您的下一個由 Inertia 驅動的 Laravel 應用程式提供了絕佳的起點。


<a name="creating-and-rendering-views"></a>
## 建立與渲染視圖

您可以透過將副檔名為 `.blade.php` 的檔案放在應用程式的 `resources/views` 目錄中，或使用 `make:view` Artisan 命令來建立視圖：

```shell
php artisan make:view greeting
```

`.blade.php` 副檔名會告知框架該檔案包含一個 [Blade 模板](/docs/{{version}}/blade)。Blade 模板包含 HTML 以及 Blade 指令，可讓您輕鬆印出值、建立「if」判斷式、迭代資料等等。

建立視圖後，您可以從應用程式的路由或控制器之一使用全域 `view` 輔助函式來傳回它：

```php
Route::get('/', function () {
    return view('greeting', ['name' => 'James']);
});
```

視圖也可以使用 `View` Facade 傳回：

```php
use Illuminate\Support\Facades\View;

return View::make('greeting', ['name' => 'James']);
```

如您所見，傳遞給 `view` 輔助函式的第一個引數對應於 `resources/views` 目錄中視圖檔案的名稱。第二個引數是一個應提供給視圖的資料陣列。在此案例中，我們傳遞了 `name` 變數，該變數使用 [Blade 語法](/docs/{{version}}/blade)顯示在視圖中。


<a name="nested-view-directories"></a>
### 巢狀視圖目錄

視圖也可以巢狀地儲存在 `resources/views` 目錄的子目錄中。「點」符號可用於參照巢狀視圖。例如，如果您的視圖儲存在 `resources/views/admin/profile.blade.php`，您可以從應用程式的路由 / 控制器之一傳回它，如下所示：

```php
return view('admin.profile', $data);
```

> [!WARNING]
> 視圖目錄名稱不應包含 `.` 字元。


<a name="creating-the-first-available-view"></a>
### 建立第一個可用的視圖

使用 `View` Facade 的 `first` 方法，您可以建立給定視圖陣列中存在的第一個視圖。如果您的應用程式或套件允許自訂或覆寫視圖，這可能會很有用：

```php
use Illuminate\Support\Facades\View;

return View::first(['custom.admin', 'admin'], $data);
```


<a name="determining-if-a-view-exists"></a>
### 判斷視圖是否存在

如果您需要判斷視圖是否存在，可以使用 `View` Facade。`exists` 方法將在視圖存在時傳回 `true`：

```php
use Illuminate\Support\Facades\View;

if (View::exists('admin.profile')) {
    // ...
}
```


<a name="passing-data-to-views"></a>
## 傳遞資料至視圖

正如您在前面的範例中看到的，您可以將資料陣列傳遞給視圖，以使該資料在視圖中可用：

```php
return view('greetings', ['name' => 'Victoria']);
```

以這種方式傳遞資訊時，資料應為鍵值對陣列。將資料提供給視圖後，您可以使用資料的鍵存取視圖中的每個值，例如 `<?php echo $name; ?>`。

作為將完整資料陣列傳遞給 `view` 輔助函式的一種替代方案，您可以使用 `with` 方法向視圖添加個別資料。`with` 方法傳回一個視圖物件實例，以便您可以在傳回視圖之前繼續鏈接方法：

```php
return view('greeting')
    ->with('name', 'Victoria')
    ->with('occupation', 'Astronaut');
```


<a name="sharing-data-with-all-views"></a>
### 與所有視圖共享資料

有時，您可能需要與應用程式渲染的所有視圖共享資料。您可以使用 `View` Facade 的 `share` 方法來做到這一點。通常，您應該將 `share` 方法的呼叫放在服務提供者的 `boot` 方法中。您可以將它們添加到 `App\Providers\AppServiceProvider` 類別中，或產生一個單獨的服務提供者來包含它們：

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
## 視圖合成器

視圖合成器是當視圖被渲染時呼叫的回呼或類別方法。如果你想將資料綁定到每次渲染的視圖上，視圖合成器可以幫助你將該邏輯組織到單一位置。如果應用程式中的多個路由或控制器回傳相同的視圖，並且該視圖總是需要特定資料，視圖合成器將會特別有用。

通常，視圖合成器會註冊在應用程式的其中一個[服務提供者](/docs/{{version}}/providers)中。在此範例中，我們假設 `App\Providers\AppServiceProvider` 將包含此邏輯。

我們將使用 `View` Facade 的 `composer` 方法來註冊視圖合成器。Laravel 不包含基於類別的視圖合成器的預設目錄，因此你可以自由組織它們。例如，你可以建立一個 `app/View/Composers` 目錄來存放應用程式的所有視圖合成器：

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

現在我們已經註冊了合成器，每當 `profile` 視圖被渲染時，`App\View\Composers\ProfileComposer` 類別的 `compose` 方法都會被執行。讓我們看看這個合成器類別的範例：

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

如你所見，所有視圖合成器都透過[服務容器](/docs/{{version}}/container)解析，因此你可以在合成器的建構子中型別提示任何所需的依賴。

<a name="attaching-a-composer-to-multiple-views"></a>
#### 將合成器附加到多個視圖

你可以透過將視圖陣列作為 `composer` 方法的第一個引數傳遞，一次將視圖合成器附加到多個視圖：

```php
use App\Views\Composers\MultiComposer;
use Illuminate\Support\Facades\View;

View::composer(
    ['profile', 'dashboard'],
    MultiComposer::class
);
```

`composer` 方法也接受 `*` 字元作為萬用字元，允許你將合成器附加到所有視圖：

```php
use Illuminate\Support\Facades;
use Illuminate\View\View;

Facades\View::composer('*', function (View $view) {
    // ...
});
```

<a name="view-creators"></a>
### 視圖建立器

視圖「建立器」與視圖合成器非常相似；然而，它們是在視圖被實例化後立即執行，而不是等到視圖即將渲染時才執行。要註冊視圖建立器，請使用 `creator` 方法：

```php
use App\View\Creators\ProfileCreator;
use Illuminate\Support\Facades\View;

View::creator('profile', ProfileCreator::class);
```

<a name="optimizing-views"></a>
## 優化視圖

預設情況下，Blade 模板視圖是按需編譯的。當執行渲染視圖的請求時，Laravel 會判斷是否存在視圖的編譯版本。如果該檔案存在，Laravel 將進一步判斷未編譯的視圖是否比已編譯的視圖更新。如果編譯後的視圖不存在，或者未編譯的視圖已被修改，Laravel 將重新編譯該視圖。

在請求期間編譯視圖可能會對效能產生輕微的負面影響，因此 Laravel 提供了 `view:cache` Artisan 命令來預編譯應用程式使用的所有視圖。為了提高效能，你可能希望在部署過程中執行此命令：

```shell
php artisan view:cache
```

你可以使用 `view:clear` 命令來清除視圖快取：

```shell
php artisan view:clear
```