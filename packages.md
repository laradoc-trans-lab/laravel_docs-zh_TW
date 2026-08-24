# 套件開發

- [介紹](#introduction)
    - [建立套件](#creating-a-package)
    - [關於 Facades 的注意事項](#a-note-on-facades)
- [套件自動偵測](#package-discovery)
- [服務提供者(Service Providers)](#service-providers)
- [資源](#resources)
    - [設定](#configuration)
    - [路由](#routes)
    - [遷移檔](#migrations)
    - [語言檔](#language-files)
    - [視圖](#views)
    - [視圖元件](#view-components)
    - [「About」Artisan 指令](#about-artisan-command)
- [指令](#commands)
    - [最佳化指令](#optimize-commands)
    - [重新載入指令](#reload-commands)
- [公開靜態資源](#public-assets)
- [發布檔案群組](#publishing-file-groups)

<a name="introduction"></a>
## 介紹

套件是為 Laravel 新增功能的主要方式。套件可以是任何東西，從處理日期的好幫手（如 [Carbon](https://github.com/briannesbitt/Carbon)），到讓你將檔案與 Eloquent 模型關聯的套件（如 Spatie 的 [Laravel Media Library](https://github.com/spatie/laravel-medialibrary)）。

套件有不同的類型。有些套件是獨立的 (stand-alone)，這代表它們可以與任何 PHP 框架搭配運作。Carbon 和 Pest 就是獨立套件的範例。這些套件都可以透過在你的 `composer.json` 檔案中引入來與 Laravel 一起使用。

另一方面，有些套件則是專門為 Laravel 設計的。這些套件可能包含專門用來增強 Laravel 應用程式的路由、控制器、視圖和設定檔。本指南主要涵蓋這些專屬於 Laravel 的套件開發。


<a name="creating-a-package"></a>
### 建立套件

開始建立新的 Laravel 套件最簡單的方式是使用官方的 [Laravel package skeleton](https://github.com/laravel/package-skeleton)。此骨架 (skeleton) 提供了建置 Laravel 套件所需的一切，包括服務提供者(Service Providers)、使用 Pest 進行測試、透過 Larastan 進行靜態分析、使用 Pint 進行程式碼格式化，以及用於端到端套件開發的 workbench 應用程式。你可以使用 [Laravel 安裝程式 CLI](/docs/{{version}}/installation#creating-a-laravel-project) 的 `package` 指令來建立新套件：

```shell
laravel package my-package
```

互動式的設定腳本會為你的套件個性化客製骨架，設定你的命名空間、服務提供者，以及僅為你設定需要的功用，例如設定檔、路由、視圖、翻譯檔、遷移檔、靜態資源、指令和 Facade。


<a name="a-note-on-facades"></a>
### 關於 Facades 的注意事項

在撰寫 Laravel 應用程式時，使用契約(Contracts)或 Facade 通常沒有太大的差別，因為兩者本質上都提供相同的可測試性。然而，在撰寫套件時，你的套件通常無法直接使用 Laravel 的所有測試輔助函式。如果你希望能像將套件安裝在一般的 Laravel 應用程式中一樣撰寫套件測試，你可以使用 [Orchestral Testbench](https://github.com/orchestral/testbench) 套件。


<a name="package-discovery"></a>
## 套件自動偵測

Laravel 應用程式的 `bootstrap/providers.php` 檔案包含了由 Laravel 載入的服務提供者清單。然而，你可以直接在你套件的 `composer.json` 檔案中的 `extra` 區塊定義提供者，讓 Laravel 自動載入，而無需要求使用者手動將你的服務提供者新增至清單中。除了服務提供者之外，你也可以列出任何你希望註冊的 [Facade](/docs/{{version}}/facades)：

```json
"extra": {
    "laravel": {
        "providers": [
            "Barryvdh\\Debugbar\\ServiceProvider"
        ],
        "aliases": {
            "Debugbar": "Barryvdh\\Debugbar\\Facade"
        }
    }
},
```

一旦為你的套件設定好自動偵測功能後，Laravel 就會在套件安裝時自動註冊其服務提供者與 Facade，為你的套件使用者提供便利的安裝體驗。


<a name="opting-out-of-package-discovery"></a>
#### 停用套件自動偵測

如果你是套件的使用者，且想要停用特定套件的自動偵測功能，你可以在應用程式的 `composer.json` 檔案中的 `extra` 區塊列出該套件的名稱：

```json
"extra": {
    "laravel": {
        "dont-discover": [
            "barryvdh/laravel-debugbar"
        ]
    }
},
```

你可以使用應用程式的 `dont-discover` 指令中的 `*` 字元來停用所有套件的套件自動偵測：

```json
"extra": {
    "laravel": {
        "dont-discover": [
            "*"
        ]
    }
},
```


<a name="service-providers"></a>
## 服務提供者(Service Providers)

[服務提供者(Service Providers)](/docs/{{version}}/providers) 是你的套件與 Laravel 之間的連結點。服務提供者負責將事物綁定到 Laravel 的[服務容器](/docs/{{version}}/container)中，並告知 Laravel 去哪裡載入套件資源，例如視圖、設定檔和語言檔。

服務提供者繼承了 `Illuminate\Support\ServiceProvider` 類別，並包含兩個方法：`register` 與 `boot`。基礎的 `ServiceProvider` 類別部位於 `illuminate/support` Composer 套件中，你應該將其新增至自己套件的依賴項目中。若要深入瞭解服務提供者的結構與用途，請參閱[其相關文件](/docs/{{version}}/providers)。

<a name="resources"></a>
## 資源


<a name="configuration"></a>
### 設定

通常，您需要將套件的設定檔發布到應用程式的 `config` 目錄中。這將允許您的套件使用者輕鬆覆寫您的預設設定選項。若要允許發布您的設定檔，請在服務提供者的 `boot` 方法中呼叫 `publishes` 方法：

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->publishes([
        __DIR__.'/../config/courier.php' => config_path('courier.php'),
    ]);
}
```

現在，當您套件的使用者執行 Laravel 的 `vendor:publish` 指令時，您的檔案將會被複製到指定的發布位置。一旦您的設定被發布，就可以像任何其他設定檔一樣存取其數值：

```php
$value = config('courier.option');
```

> [!WARNING]
> 您不應該在設定檔中定義閉包 (Closure)。當使用者執行 `config:cache` Artisan 指令時，閉包無法被正確序列化。


<a name="default-package-configuration"></a>
#### 預設套件設定

您也可以將您自己的套件設定檔與應用程式已發布的副本進行合併。這將允許您的使用者僅在已發布的設定檔副本中定義他們實際上想覆寫的選項。若要合併設定檔的值，請在服務提供者的 `register` 方法內使用 `mergeConfigFrom` 方法。

`mergeConfigFrom` 方法接收套件設定檔的路徑作為其第一個引數，並接收應用程式設定檔副本的名稱作為其第二個引數：

```php
/**
 * Register any package services.
 */
public function register(): void
{
    $this->mergeConfigFrom(
        __DIR__.'/../config/courier.php', 'courier'
    );
}
```

> [!WARNING]
> 此方法僅會合併設定陣列的第一層。如果您的使用者僅部分定義了多維設定陣列，則缺失的選項將不會被合併。


<a name="routes"></a>
### 路由

如果您的套件包含路由，您可以使用 `loadRoutesFrom` 方法來載入它們。此方法會自動判斷應用程式的路由是否已被快取，如果路由已經被快取，則不會載入您的路由檔案：

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->loadRoutesFrom(__DIR__.'/../routes/web.php');
}
```


<a name="migrations"></a>
### 遷移檔

如果您的套件包含[資料庫遷移檔](/docs/{{version}}/migrations)，您可以使用 `publishesMigrations` 方法來告知 Laravel 給定的目錄或檔案包含遷移檔。當 Laravel 發布這些遷移檔時，它會自動更新其檔名中的時間戳記，以反映當前的日期與時間：

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->publishesMigrations([
        __DIR__.'/../database/migrations' => database_path('migrations'),
    ]);
}
```


<a name="language-files"></a>
### 語言檔

如果您的套件包含[語言檔](/docs/{{version}}/localization)，您可以使用 `loadTranslationsFrom` 方法來告知 Laravel 如何載入它們。例如，如果您的套件名稱為 `courier`，您應該在服務提供者的 `boot` 方法中加入以下內容：

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->loadTranslationsFrom(__DIR__.'/../lang', 'courier');
}
```

套件翻譯語系行是使用 `package::file.line` 語法慣例來引用的。因此，您可以像這樣從 `messages` 檔案載入 `courier` 套件的 `welcome` 行：

```php
echo trans('courier::messages.welcome');
```

您可以使用 `loadJsonTranslationsFrom` 方法為您的套件註冊 JSON 翻譯檔案。此方法接收包含您套件 JSON 翻譯檔案的目錄路徑：

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->loadJsonTranslationsFrom(__DIR__.'/../lang');
}
```


<a name="publishing-language-files"></a>
#### 發布語言檔

如果您想將套件的語言檔發布到應用程式的 `lang/vendor` 目錄，您可以使用服務提供者的 `publishes` 方法。`publishes` 方法接收一個包含套件路徑及其預期發布位置的陣列。例如，若要發布 `courier` 套件的語言檔，您可以執行以下操作：

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->loadTranslationsFrom(__DIR__.'/../lang', 'courier');

    $this->publishes([
        __DIR__.'/../lang' => $this->app->langPath('vendor/courier'),
    ]);
}
```

現在，當您套件的使用者執行 Laravel 的 `vendor:publish` Artisan 指令時，您的套件語言檔將會被發布到指定的發布位置。


<a name="views"></a>
### 視圖

若要在 Laravel 中註冊您套件的[視圖](/docs/{{version}}/views)，您需要告知 Laravel 這些視圖的位置。您可以使用服務提供者的 `loadViewsFrom` 方法來做到這一點。`loadViewsFrom` 方法接收兩個引數：您的視圖模板路徑以及您的套件名稱。例如，如果您的套件名稱是 `courier`，您可以在服務提供者的 `boot` 方法中加入以下內容：

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->loadViewsFrom(__DIR__.'/../resources/views', 'courier');
}
```

套件視圖是使用 `package::view` 語法慣例來引用的。因此，一旦您的視圖路徑在服務提供者中註冊，您就可以像這樣載入 `courier` 套件的 `dashboard` 視圖：

```php
Route::get('/dashboard', function () {
    return view('courier::dashboard');
});
```


<a name="overriding-package-views"></a>
#### 覆寫套件視圖

當您使用 `loadViewsFrom` 方法時，Laravel 實際上會為您的視圖註冊兩個位置：應用程式的 `resources/views/vendor` 目錄以及您指定的目錄。因此，以 `courier` 套件為例，Laravel 會先檢查開發者是否已在 `resources/views/vendor/courier` 目錄中放置了該視圖的客製化版本。接著，如果該視圖尚未被客製化，Laravel 會搜尋您在呼叫 `loadViewsFrom` 時所指定的套件視圖目錄。這使得套件使用者可以輕鬆地客製化 / 覆寫您套件的視圖。


<a name="publishing-views"></a>
#### 發布視圖

如果您想讓您的視圖可以被發布到應用程式的 `resources/views/vendor` 目錄，您可以使用服務提供者的 `publishes` 方法。`publishes` 方法接收一個包含套件視圖路徑及其預期發布位置的陣列：

```php
/**
 * Bootstrap the package services.
 */
public function boot(): void
{
    $this->loadViewsFrom(__DIR__.'/../resources/views', 'courier');

    $this->publishes([
        __DIR__.'/../resources/views' => resource_path('views/vendor/courier'),
    ]);
}
```

現在，當您套件的使用者執行 Laravel 的 `vendor:publish` Artisan 指令時，您的套件視圖將會被複製到指定的發布位置。

<a name="view-components"></a>
### 視圖元件

若您正在建置使用 Blade 元件的套件，或將元件放置在非傳統的目錄中，您將需要手動註冊元件類別及其 HTML 標籤別名，以便 Laravel 知道在哪裡可以找到該元件。您通常應該在套件服務提供者的 `boot` 方法中註冊您的元件：

```php
use Illuminate\Support\Facades\Blade;
use VendorPackage\View\Components\AlertComponent;

/**
 * Bootstrap your package's services.
 */
public function boot(): void
{
    Blade::component('package-alert', AlertComponent::class);
}
```

元件註冊完成後，即可使用其標籤別名來渲染：

```blade
<x-package-alert/>
```


<a name="autoloading-package-components"></a>
#### 自動載入套件元件

或者，您可以使用 `componentNamespace` 方法按照慣例自動載入元件類別。例如，`Nightshade` 套件可能擁有位於 `Nightshade\Views\Components` 命名空間下的 `Calendar` 與 `ColorPicker` 元件：

```php
use Illuminate\Support\Facades\Blade;

/**
 * Bootstrap your package's services.
 */
public function boot(): void
{
    Blade::componentNamespace('Nightshade\\Views\\Components', 'nightshade');
}
```

這將允許使用 `package-name::` 語法，透過廠商命名空間來使用套件元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 會自動將元件名稱轉換為帕斯卡命名法（PascalCase）來偵測連結至此元件的類別。同時也支援使用「點」標記法來指定子目錄。


<a name="anonymous-components"></a>
#### 匿名元件

如果您的套件包含匿名元件，它們必須放在套件「views」目錄下的 `components` 目錄中（如 [loadViewsFrom 方法](#views) 所指定）。接著，您可以在元件名稱前加上套件的視圖命名空間前綴來渲染它們：

```blade
<x-courier::alert />
```


<a name="about-artisan-command"></a>
### 「About」Artisan 指令

Laravel 內建的 `about` Artisan 指令提供了應用程式環境與設定的概要資訊。套件可以透過 `AboutCommand` 類別將額外資訊推送到此指令的輸出中。通常，這些資訊可以在套件服務提供者的 `boot` 方法中新增：

```php
use Illuminate\Foundation\Console\AboutCommand;

/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    AboutCommand::add('My Package', fn () => ['Version' => '1.0.0']);
}
```

<a name="commands"></a>
## 指令

若要向 Laravel 註冊您套件的 Artisan 指令，可以使用 `commands` 方法。這個方法接收一個指令類別名稱的陣列。指令註冊完成後，您就可以使用 [Artisan CLI](/docs/{{version}}/artisan) 來執行它們：

```php
use Courier\Console\Commands\InstallCommand;
use Courier\Console\Commands\NetworkCommand;

/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    if ($this->app->runningInConsole()) {
        $this->commands([
            InstallCommand::class,
            NetworkCommand::class,
        ]);
    }
}
```


<a name="optimize-commands"></a>
### 最佳化指令

Laravel 的 [optimize 指令](/docs/{{version}}/deployment#optimization)會快取應用程式的設定、事件、路由與視圖。透過使用 `optimizes` 方法，您可以註冊自己套件的 Artisan 指令，使其在執行 `optimize` 與 `optimize:clear` 指令時一併被呼叫：

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    if ($this->app->runningInConsole()) {
        $this->optimizes(
            optimize: 'package:optimize',
            clear: 'package:clear-optimizations',
        );
    }
}
```


<a name="reload-commands"></a>
### 重新載入指令

Laravel 的 [reload 指令](/docs/{{version}}/deployment#reloading-services)會終止任何正在執行的服務，以便系統行程監控器（System Process Monitor）能自動重新啟動它們。透過使用 `reloads` 方法，您可以註冊自己套件的 Artisan 指令，使其在執行 `reload` 指令時一併被呼叫：

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    if ($this->app->runningInConsole()) {
        $this->reloads('package:reload');
    }
}
```


<a name="public-assets"></a>
## 公開靜態資源

您的套件可能包含 JavaScript、CSS 和圖片等靜態資源。若要將這些靜態資源發布至應用程式的 `public` 目錄，請使用服務提供者的 `publishes` 方法。在本例中，我們還會新增一個 `public` 靜態資源群組標籤，可用於輕鬆發布一組相關的靜態資源：

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->publishes([
        __DIR__.'/../public' => public_path('vendor/courier'),
    ], 'public');
}
```

現在，當您套件的使用者執行 `vendor:publish` 指令時，您的靜態資源就會被複製到指定的發布位置。由於使用者通常需要在每次更新套件時覆蓋靜態資源，因此他們可以使用 `--force` 旗標：

```shell
php artisan vendor:publish --tag=public --force
```


<a name="publishing-file-groups"></a>
## 發布檔案群組

您可能希望分別發布套件靜態資源與資源的不同群組。例如，您可能希望允許使用者僅發布套件的設定檔，而不需要強制發布套件的靜態資源。您可以在套件服務提供者呼叫 `publishes` 方法時為它們「加上標籤」來達到這個目的。例如，讓我們在套件服務提供者的 `boot` 方法中，使用標籤為 `courier` 套件定義兩個發布群組（`courier-config` 與 `courier-migrations`）：

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->publishes([
        __DIR__.'/../config/package.php' => config_path('package.php')
    ], 'courier-config');

    $this->publishesMigrations([
        __DIR__.'/../database/migrations/' => database_path('migrations')
    ], 'courier-migrations');
}
```

現在，您的使用者可以在執行 `vendor:publish` 指令時指定標籤，來分別發布這些群組：

```shell
php artisan vendor:publish --tag=courier-config
```

您的使用者也可以使用 `--provider` 旗標來發布套件服務提供者所定義的所有可發布檔案：

```shell
php artisan vendor:publish --provider="Your\Package\ServiceProvider"
```