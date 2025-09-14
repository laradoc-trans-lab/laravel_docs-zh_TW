# 套件開發

- [簡介](#introduction)
    - [關於 Facades 的說明](#a-note-on-facades)
- [套件自動探索](#package-discovery)
- [服務提供者](#service-providers)
- [資源](#resources)
    - [設定](#configuration)
    - [路由](#routes)
    - [資料庫遷移](#migrations)
    - [語系檔](#language-files)
    - [視圖](#views)
    - [視圖元件](#view-components)
    - ["About" Artisan 命令](#about-artisan-command)
- [命令](#commands)
    - [最佳化命令](#optimize-commands)
- [公開資產](#public-assets)
- [發佈檔案群組](#publishing-file-groups)

<a name="introduction"></a>
## 簡介

套件是為 Laravel 增加功能的主要方式。套件可以是任何東西，從像 [Carbon](https://github.com/briannesbitt/Carbon) 這樣處理日期的絕佳工具，到像 Spatie 的 [Laravel Media Library](https://github.com/spatie/laravel-medialalibrary) 這樣允許您將檔案與 Eloquent 模型關聯的套件。

套件有不同的類型。有些套件是獨立的，這表示它們可以與任何 PHP 框架一起使用。Carbon 和 Pest 就是獨立套件的範例。這些套件中的任何一個都可以透過在您的 `composer.json` 檔案中引入它們來與 Laravel 一起使用。

另一方面，其他套件是專門為 Laravel 設計的。這些套件可能包含專門用於增強 Laravel 應用程式的路由、控制器、視圖和設定。本指南主要涵蓋這些 Laravel 專用套件的開發。

<a name="a-note-on-facades"></a>
### 關於 Facades 的說明

在撰寫 Laravel 應用程式時，通常使用 Contracts 或 Facades 並沒有太大區別，因為兩者都提供了本質上相同的可測試性。然而，在撰寫套件時，您的套件通常無法存取所有 Laravel 的測試輔助工具。如果您希望能夠像套件安裝在典型的 Laravel 應用程式中一樣撰寫套件測試，您可以使用 [Orchestral Testbench](https://github.com/orchestral/testbench) 套件。

<a name="package-discovery"></a>
## 套件自動探索

Laravel 應用程式的 `bootstrap/providers.php` 檔案包含 Laravel 應該載入的服務提供者列表。然而，您不必要求使用者手動將您的服務提供者新增到列表中，您可以將提供者定義在套件的 `composer.json` 檔案的 `extra` 區塊中，以便 Laravel 自動載入它。除了服務提供者之外，您還可以列出任何您希望註冊的 [Facades](/docs/{{version}}/facades)：

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

一旦您的套件配置為可自動探索，Laravel 將在安裝時自動註冊其服務提供者和 Facades，為您的套件使用者提供便利的安裝體驗。

<a name="opting-out-of-package-discovery"></a>
#### 停用套件自動探索

如果您是套件的使用者，並且希望停用某個套件的自動探索功能，您可以在應用程式的 `composer.json` 檔案的 `extra` 區塊中列出該套件的名稱：

```json
"extra": {
    "laravel": {
        "dont-discover": [
            "barryvdh/laravel-debugbar"
        ]
    }
},
```

您可以使用應用程式 `dont-discover` 指令中的 `*` 字元來停用所有套件的自動探索功能：

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

[服務提供者](/docs/{{version}}/providers) 是您的套件與 Laravel 之間的連接點。服務提供者負責將內容綁定到 Laravel 的 [Service Container](/docs/{{version}}/container) 中，並告知 Laravel 從何處載入套件資源，例如視圖、設定和語系檔。

服務提供者會擴展 `Illuminate\Support\ServiceProvider` 類別，並包含兩個方法：`register` 和 `boot`。基礎的 `ServiceProvider` 類別位於 `illuminate/support` Composer 套件中，您應該將其新增到您自己的套件的依賴項中。要了解有關服務提供者結構和目的的更多資訊，請查閱 [它們的說明文件](/docs/{{version}}/providers)。

<a name="resources"></a>
## 資源

<a name="configuration"></a>
### 設定

通常，您需要將套件的設定檔發佈到應用程式的 `config` 目錄。這將允許您的套件使用者輕鬆覆寫您的預設設定選項。為了允許您的設定檔被發佈，請從服務提供者的 `boot` 方法中呼叫 `publishes` 方法：

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

現在，當您的套件使用者執行 Laravel 的 `vendor:publish` 命令時，您的檔案將被複製到指定的發佈位置。一旦您的設定檔被發佈，其值就可以像任何其他設定檔一樣被存取：

```php
$value = config('courier.option');
```

> [!WARNING]
> 您不應該在設定檔中定義閉包。當使用者執行 `config:cache` Artisan 命令時，它們無法正確序列化。

<a name="default-package-configuration"></a>
#### 預設套件設定

您也可以將您自己的套件設定檔與應用程式已發佈的副本合併。這將允許您的使用者僅定義他們實際希望在設定檔的已發佈副本中覆寫的選項。要合併設定檔值，請在服務提供者的 `register` 方法中使用 `mergeConfigFrom` 方法。

`mergeConfigFrom` 方法接受您的套件設定檔的路徑作為第一個參數，以及應用程式設定檔副本的名稱作為第二個參數：

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
> 此方法僅合併設定陣列的第一層。如果您的使用者部分定義了多維設定陣列，則遺失的選項將不會被合併。

<a name="routes"></a>
### 路由

如果您的套件包含路由，您可以使用 `loadRoutesFrom` 方法載入它們。此方法將自動判斷應用程式的路由是否已快取，如果路由已快取，則不會載入您的路由檔案：

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->loadRoutesFrom(__DIR__.'/../routes/web.php');
}
}
```

<a name="migrations"></a>
### 資料庫遷移

如果您的套件包含 [資料庫遷移](/docs/{{version}}/migrations)，您可以使用 `publishesMigrations` 方法來告知 Laravel 給定的目錄或檔案包含遷移。當 Laravel 發佈遷移時，它會自動更新其檔案名稱中的時間戳記以反映當前的日期和時間：

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
### 語系檔

如果您的套件包含 [語系檔](/docs/{{version}}/localization)，您可以使用 `loadTranslationsFrom` 方法來告知 Laravel 如何載入它們。例如，如果您的套件名為 `courier`，您應該將以下內容新增到服務提供者的 `boot` 方法中：

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->loadTranslationsFrom(__DIR__.'/../lang', 'courier');
}
```

套件翻譯行使用 `package::file.line` 語法慣例引用。因此，您可以像這樣從 `messages` 檔案中載入 `courier` 套件的 `welcome` 行：

```php
echo trans('courier::messages.welcome');
```

您可以使用 `loadJsonTranslationsFrom` 方法為您的套件註冊 JSON 翻譯檔。此方法接受包含您的套件 JSON 翻譯檔的目錄路徑：

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
#### 發佈語系檔

如果您希望將套件的語系檔發佈到應用程式的 `lang/vendor` 目錄，您可以使用服務提供者的 `publishes` 方法。`publishes` 方法接受一個套件路徑陣列及其所需的發佈位置。例如，要發佈 `courier` 套件的語系檔，您可以執行以下操作：

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

現在，當您的套件使用者執行 Laravel 的 `vendor:publish` Artisan 命令時，您的套件語系檔將被發佈到指定的發佈位置。

<a name="views"></a>
### 視圖

要向 Laravel 註冊您的套件 [視圖](/docs/{{version}}/views)，您需要告訴 Laravel 視圖的位置。您可以使用服務提供者的 `loadViewsFrom` 方法來完成此操作。`loadViewsFrom` 方法接受兩個參數：您的視圖模板路徑和您的套件名稱。例如，如果您的套件名稱是 `courier`，您將以下內容新增到服務提供者的 `boot` 方法中：

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->loadViewsFrom(__DIR__.'/../resources/views', 'courier');
}
```

套件視圖使用 `package::view` 語法慣例引用。因此，一旦您的視圖路徑在服務提供者中註冊，您就可以像這樣從 `courier` 套件中載入 `dashboard` 視圖：

```php
Route::get('/dashboard', function () {
    return view('courier::dashboard');
});
```

<a name="overriding-package-views"></a>
#### 覆寫套件視圖

當您使用 `loadViewsFrom` 方法時，Laravel 實際上會為您的視圖註冊兩個位置：應用程式的 `resources/views/vendor` 目錄和您指定的目錄。因此，以 `courier` 套件為例，Laravel 將首先檢查開發人員是否已將視圖的自訂版本放置在 `resources/views/vendor/courier` 目錄中。然後，如果視圖尚未自訂，Laravel 將搜尋您在呼叫 `loadViewsFrom` 時指定的套件視圖目錄。這使得套件使用者可以輕鬆自訂/覆寫您的套件視圖。

<a name="publishing-views"></a>
#### 發佈視圖

如果您希望將您的視圖發佈到應用程式的 `resources/views/vendor` 目錄，您可以使用服務提供者的 `publishes` 方法。`publishes` 方法接受一個套件視圖路徑陣列及其所需的發佈位置：

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

現在，當您的套件使用者執行 Laravel 的 `vendor:publish` Artisan 命令時，您的套件視圖將被複製到指定的發佈位置。

<a name="view-components"></a>
### 視圖元件

如果您正在建構一個使用 Blade 元件或將元件放置在非傳統目錄中的套件，您將需要手動註冊您的元件類別及其 HTML 標籤別名，以便 Laravel 知道在哪裡找到該元件。您通常應該在套件服務提供者的 `boot` 方法中註冊您的元件：

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

一旦您的元件被註冊，就可以使用其標籤別名來渲染：

```blade
<x-package-alert/>
```

<a name="autoloading-package-components"></a>
#### 自動載入套件元件

或者，您可以使用 `componentNamespace` 方法透過慣例自動載入元件類別。例如，一個 `Nightshade` 套件可能包含位於 `Nightshade\Views\Components` 命名空間中的 `Calendar` 和 `ColorPicker` 元件：

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

這將允許使用 `package-name::` 語法透過其供應商命名空間來使用套件元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 將透過將元件名稱轉換為 PascalCase 來自動偵測與此元件連結的類別。子目錄也支援使用「點」表示法。

<a name="anonymous-components"></a>
#### 匿名元件

如果您的套件包含匿名元件，它們必須放置在套件「視圖」目錄（如 [loadViewsFrom 方法](#views) 所指定）的 `components` 目錄中。然後，您可以透過在元件名稱前加上套件的視圖命名空間來渲染它們：

```blade
<x-courier::alert />
```

<a name="about-artisan-command"></a>
### "About" Artisan 命令

Laravel 內建的 `about` Artisan 命令提供了應用程式環境和設定的概要。套件可以透過 `AboutCommand` 類別向此命令的輸出推送額外資訊。通常，此資訊可以從您的套件服務提供者的 `boot` 方法中新增：

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
## 命令

要向 Laravel 註冊您的套件 Artisan 命令，您可以使用 `commands` 方法。此方法需要一個命令類別名稱陣列。一旦命令被註冊，您就可以使用 [Artisan CLI](/docs/{{version}}/artisan) 執行它們：

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
### 最佳化命令

Laravel 的 [最佳化命令](/docs/{{version}}/deployment#optimization) 會快取應用程式的設定、事件、路由和視圖。使用 `optimizes` 方法，您可以註冊您自己的套件 Artisan 命令，這些命令應該在執行 `optimize` 和 `optimize:clear` 命令時被呼叫：

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
## 公開資產

您的套件可能包含 JavaScript、CSS 和圖片等資產。要將這些資產發佈到應用程式的 `public` 目錄，請使用服務提供者的 `publishes` 方法。在此範例中，我們還將新增一個 `public` 資產群組標籤，該標籤可用於輕鬆發佈相關資產群組：

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

現在，當您的套件使用者執行 `vendor:publish` 命令時，您的資產將被複製到指定的發佈位置。由於使用者通常需要在每次套件更新時覆寫資產，您可以使用 `--force` 旗標：

```shell
php artisan vendor:publish --tag=public --force
```

<a name="publishing-file-groups"></a>
## 發佈檔案群組

您可能希望單獨發佈套件資產和資源的群組。例如，您可能希望允許您的使用者發佈您的套件設定檔，而無需強制發佈您的套件資產。您可以透過在套件服務提供者的 `publishes` 方法中呼叫時「標記」它們來實現此目的。例如，讓我們在套件服務提供者的 `boot` 方法中使用標籤為 `courier` 套件定義兩個發佈群組（`courier-config` 和 `courier-migrations`）：

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

現在，您的使用者可以透過在執行 `vendor:publish` 命令時引用其標籤來單獨發佈這些群組：

```shell
php artisan vendor:publish --tag=courier-config
```

您的使用者也可以使用 `--provider` 旗標發佈您的套件服務提供者定義的所有可發佈檔案：

```shell
php artisan vendor:publish --provider="Your\Package\ServiceProvider"
```

