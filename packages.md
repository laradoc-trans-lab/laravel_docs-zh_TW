# 套件開發

- [介紹](#introduction)
    - [關於 Facades 的說明](#a-note-on-facades)
- [套件探索](#package-discovery)
- [服務提供者](#service-providers)
- [資源](#resources)
    - [配置](#configuration)
    - [路由](#routes)
    - [遷移](#migrations)
    - [語系檔案](#language-files)
    - [視圖](#views)
    - [視圖元件](#view-components)
    - [「About」Artisan 指令](#about-artisan-command)
- [指令](#commands)
    - [優化指令](#optimize-commands)
- [公用資源](#public-assets)
- [發佈檔案群組](#publishing-file-groups)

<a name="introduction"></a>
## 介紹

套件是向 Laravel 應用程式增加功能的主要方式。套件可能是任何東西，從像 [Carbon](https://github.com/briannesbitt/Carbon) 這樣好用的日期處理工具，到像 Spatie 的 [Laravel Media Library](https://github.com/spatie/laravel-medialibrary) 這樣允許你將檔案與 Eloquent 模型關聯的套件。

套件有不同的類型。有些套件是獨立的，這表示它們可以與任何 PHP 框架配合使用。Carbon 和 Pest 是獨立套件的範例。這些套件中的任何一個都可以透過在你的 `composer.json` 檔案中引入它們來與 Laravel 搭配使用。

另一方面，其他套件是專門為 Laravel 而設計的。這些套件可能包含路由、控制器、視圖和配置，專門用於增強 Laravel 應用程式。本指南主要涵蓋專為 Laravel 設計的套件開發。

<a name="a-note-on-facades"></a>
### 關於 Facades 的說明

當撰寫 Laravel 應用程式時，你使用契約 (contracts) 還是 Facades 通常並不重要，因為兩者都提供本質上相同的可測試性。然而，當撰寫套件時，你的套件通常無法存取所有 Laravel 的測試輔助工具。如果你希望能夠像套件安裝在典型的 Laravel 應用程式中一樣撰寫套件測試，你可以使用 [Orchestral Testbench](https://github.com/orchestral/testbench) 套件。

<a name="package-discovery"></a>
## 套件探索

Laravel 應用程式的 `bootstrap/providers.php` 檔案包含了 Laravel 應該載入的服務提供者清單。然而，你不需要要求使用者手動將你的服務提供者加入清單，你可以在套件的 `composer.json` 檔案的 `extra` 部分定義服務提供者，這樣 Laravel 就會自動載入它。除了服務提供者之外，你也可以列出任何你希望被註冊的 [Facades](/docs/{{version}}/facades)：

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

一旦你的套件被配置為可探索，Laravel 將在安裝時自動註冊其服務提供者和 Facades，為你的套件使用者創造便利的安裝體驗。

<a name="opting-out-of-package-discovery"></a>
#### 停用套件探索

如果你是套件的使用者，並且希望為某個套件停用套件探索，你可以在應用程式的 `composer.json` 檔案的 `extra` 部分列出該套件的名稱：

```json
"extra": {
    "laravel": {
        "dont-discover": [
            "barryvdh/laravel-debugbar"
        ]
    }
},
```

你可以使用應用程式 `dont-discover` 指令中的 `*` 字元，來停用所有套件的套件探索：

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
## 服務提供者

[服務提供者](/docs/{{version}}/providers) 是你的套件與 Laravel 之間的連接點。服務提供者負責將內容綁定到 Laravel 的 [服務容器](/docs/{{version}}/container) 中，並告知 Laravel 從何處載入套件資源，例如視圖、配置和語系檔案。

服務提供者擴展了 `Illuminate\Support\ServiceProvider` 類別，並包含兩個方法：`register` 和 `boot`。基礎 `ServiceProvider` 類別位於 `illuminate/support` Composer 套件中，你應該將其添加到你自己的套件依賴項中。要了解更多關於服務提供者結構和目的的資訊，請查看 [其文件](/docs/{{version}}/providers)。

<a name="resources"></a>
## 資源

<a name="configuration"></a>
### 配置

通常，你需要將你的套件配置檔發佈到應用程式的 `config` 目錄。這將允許你的套件使用者能夠輕鬆地覆寫你的預設配置選項。為了讓你的配置檔可以被發佈，請在你的服務提供者的 `boot` 方法中呼叫 `publishes` 方法：

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

現在，當你的套件使用者執行 Laravel 的 `vendor:publish` 指令時，你的檔案將會被複製到指定的發佈位置。一旦你的配置已經發佈，它的值就可以像其他任何配置檔一樣被存取：

```php
$value = config('courier.option');
```

> [!WARNING]
> 你不應該在你的配置檔中定義閉包。當使用者執行 `config:cache` Artisan 指令時，它們無法正確地被序列化。

<a name="default-package-configuration"></a>
#### 預設套件配置

你也可以將你自己的套件配置檔與應用程式已發佈的副本合併。這將允許你的使用者只定義他們實際想要覆寫的選項在已發佈的配置檔副本中。為了合併配置檔的值，請在你的服務提供者的 `register` 方法中使用 `mergeConfigFrom` 方法。

`mergeConfigFrom` 方法接受你的套件配置檔的路徑作為它的第一個引數，以及應用程式配置檔副本的名稱作為它的第二個引數：

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
> 這個方法只會合併配置陣列的第一層。如果你的使用者只部分定義了多維度的配置陣列，則缺少的選項將不會被合併。

<a name="routes"></a>
### 路由

如果你的套件包含路由，你可以使用 `loadRoutesFrom` 方法來載入它們。這個方法會自動判斷應用程式的路由是否已被快取，如果路由已被快取，它就不會載入你的路由檔案：

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
### 遷移

如果你的套件包含 [資料庫遷移](/docs/{{version}}/migrations)，你可以使用 `publishesMigrations` 方法來告知 Laravel 給定的目錄或檔案包含遷移。當 Laravel 發佈遷移時，它會自動更新其檔名中的時間戳記以反映目前的日期和時間：

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
### 語系檔案

如果你的套件包含 [語系檔案](/docs/{{version}}/localization)，你可以使用 `loadTranslationsFrom` 方法來告知 Laravel 如何載入它們。例如，如果你的套件名為 `courier`，你應該將以下內容新增到你的服務提供者的 `boot` 方法中：

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->loadTranslationsFrom(__DIR__.'/../lang', 'courier');
}
```

套件的翻譯行是使用 `package::file.line` 語法慣例來引用的。因此，你可以像這樣從 `messages` 檔案中載入 `courier` 套件的 `welcome` 行：

```php
echo trans('courier::messages.welcome');
```

你可以為你的套件註冊 JSON 翻譯檔案，使用 `loadJsonTranslationsFrom` 方法。這個方法接受包含你的套件的 JSON 翻譯檔案的目錄路徑：

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
#### 發佈語系檔案

如果你想要發佈你的套件的語系檔案到應用程式的 `lang/vendor` 目錄，你可以使用服務提供者的 `publishes` 方法。`publishes` 方法接受一個套件路徑及其所需發佈位置的陣列。例如，要發佈 `courier` 套件的語系檔案，你可以這樣做：

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

現在，當你的套件使用者執行 Laravel 的 `vendor:publish` Artisan 指令時，你的套件的語系檔案將會被發佈到指定的發佈位置。

<a name="views"></a>
### 視圖

為了向 Laravel 註冊你的套件的 [視圖](/docs/{{version}}/views)，你需要告知 Laravel 視圖所在的位置。你可以使用服務提供者的 `loadViewsFrom` 方法來做到這一點。`loadViewsFrom` 方法接受兩個引數：你的視圖範本的路徑和你的套件名稱。例如，如果你的套件名為 `courier`，你應該將以下內容新增到你的服務提供者的 `boot` 方法中：

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->loadViewsFrom(__DIR__.'/../resources/views', 'courier');
}
```

套件視圖是使用 `package::view` 語法慣例來引用的。因此，一旦你的視圖路徑在服務提供者中註冊，你就可以像這樣從 `courier` 套件中載入 `dashboard` 視圖：

```php
Route::get('/dashboard', function () {
    return view('courier::dashboard');
});
```

<a name="overriding-package-views"></a>
#### 覆寫套件視圖

當你使用 `loadViewsFrom` 方法時，Laravel 實際上會為你的視圖註冊兩個位置：應用程式的 `resources/views/vendor` 目錄和你指定的目錄。因此，以 `courier` 套件為例，Laravel 會首先檢查開發者是否已將視圖的自訂版本放置在 `resources/views/vendor/courier` 目錄中。然後，如果視圖尚未被自訂，Laravel 將會在你在呼叫 `loadViewsFrom` 時指定的套件視圖目錄中搜尋。這使得套件使用者可以輕鬆地自訂／覆寫你的套件視圖。

<a name="publishing-views"></a>
#### 發佈視圖

如果你想要讓你的視圖可供發佈到應用程式的 `resources/views/vendor` 目錄，你可以使用服務提供者的 `publishes` 方法。`publishes` 方法接受一個套件視圖路徑及其所需發佈位置的陣列：

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

現在，當你的套件使用者執行 Laravel 的 `vendor:publish` Artisan 指令時，你的套件視圖將會被複製到指定的發佈位置。

<a name="view-components"></a>
### 視圖元件

如果您正在建構一個利用 Blade 元件的套件，或將元件放置在非傳統目錄中，您需要手動註冊您的元件類別及其 HTML 標籤別名，以便 Laravel 知道在哪裡找到該元件。您通常應在套件服務提供者的 `boot` 方法中註冊您的元件：

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

一旦您的元件註冊完成，即可使用其標籤別名進行渲染：

```blade
<x-package-alert/>
```


<a name="autoloading-package-components"></a>
#### 自動載入套件元件

或者，您可以使用 `componentNamespace` 方法來依照慣例自動載入元件類別。例如，一個 `Nightshade` 套件可能會有 `Calendar` 和 `ColorPicker` 元件，這些元件位於 `Nightshade\Views\Components` 命名空間中：

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

這將允許使用 `package-name::` 語法，透過其供應商命名空間來使用套件元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 將透過將元件名稱轉換為 Pascal 命名法 (pascal-casing) 來自動偵測與此元件連結的類別。子目錄也支援使用「點」符號 (dot notation)。


<a name="anonymous-components"></a>
#### 匿名元件

如果您的套件包含匿名元件，它們必須放置在套件「視圖」目錄的 `components` 目錄中（如 [loadViewsFrom 方法](#views)所指定）。然後，您可以透過在元件名稱前加上套件的視圖命名空間來渲染它們：

```blade
<x-courier::alert />
```


<a name="about-artisan-command"></a>
### 「About」Artisan 指令

Laravel 內建的 `about` Artisan 指令提供了應用程式環境和配置的摘要。套件可以透過 `AboutCommand` 類別向此指令的輸出推送額外資訊。通常，這些資訊可以從套件服務提供者的 `boot` 方法中新增：

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

若要向 Laravel 註冊套件的 Artisan 指令，可以使用 `commands` 方法。此方法需要一個指令類別名稱的陣列。指令註冊完成後，您可以使用 [Artisan CLI](/docs/{{version}}/artisan) 執行這些指令：

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
### 優化指令

Laravel 的 [優化指令](/docs/{{version}}/deployment#optimization) 會快取應用程式的配置、事件、路由和視圖。透過使用 `optimizes` 方法，您可以註冊自己套件的 Artisan 指令，這些指令應在 `optimize` 和 `optimize:clear` 指令執行時被呼叫：

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

<a name="public-assets"></a>
## 公用資源

您的套件可能包含 JavaScript、CSS 和圖片等資源 (assets)。若要將這些資源發佈到應用程式的 `public` 目錄，請使用服務提供者的 `publishes` 方法。在此範例中，我們還會新增一個 `public` 資源群組標籤，它可以用來輕鬆發佈相關的資源群組：

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

現在，當您套件的使用者執行 `vendor:publish` 指令時，您的資源將會被複製到指定的發佈位置。由於使用者通常需要在每次套件更新時覆寫這些資源，因此您可以使用 `--force` 旗標：

```shell
php artisan vendor:publish --tag=public --force
```

<a name="publishing-file-groups"></a>
## 發佈檔案群組

您可能希望分開發佈套件的資源和資產群組。例如，您可能希望允許使用者發佈套件的配置檔，而無需強制發佈套件的資產。您可以在從套件的服務提供者呼叫 `publishes` 方法時，透過「標記」它們來實現此目的。例如，讓我們在套件服務提供者的 `boot` 方法中，使用標籤為 `courier` 套件定義兩個發佈群組 (`courier-config` 和 `courier-migrations`)：

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

現在，您的使用者可以在執行 `vendor:publish` 指令時，透過參考其標籤來分開發佈這些群組：

```shell
php artisan vendor:publish --tag=courier-config
```

您的使用者也可以使用 `--provider` 旗標來發佈由套件服務提供者定義的所有可發佈檔案：

```shell
php artisan vendor:publish --provider="Your\Package\ServiceProvider"
```