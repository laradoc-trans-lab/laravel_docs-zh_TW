# 套件開發

- [簡介](#introduction)
    - [關於 Facades 的注意事項](#a-note-on-facades)
- [套件探索](#package-discovery)
- [服務提供者](#service-providers)
- [資源](#resources)
    - [設定](#configuration)
    - [路由](#routes)
    - [遷移](#migrations)
    - [語言檔案](#language-files)
    - [視圖](#views)
    - [視圖元件](#view-components)
    - ["About" Artisan 指令](#about-artisan-command)
- [指令](#commands)
    - [最佳化指令](#optimize-commands)
    - [重新載入指令](#reload-commands)
    - [公開資產](#public-assets)
- [發布檔案群組](#publishing-file-groups)

<a name="introduction"></a>
## 簡介

套件是為 Laravel 新增功能的主要方式。套件可以是任何形式，從處理日期的好工具如 [Carbon](https://github.com/briannesbitt/Carbon)，到允許您將檔案與 Eloquent 模型關聯的套件如 Spatie 的 [Laravel Media Library](https://github.com/spatie/laravel-medialibrary)。

套件有不同的類型。有些套件是獨立的，這意味著它們可以與任何 PHP 框架搭配使用。Carbon 和 Pest 就是獨立套件的範例。任何這類套件都可以透過在您的 `composer.json` 檔案中要求 (Require) 它們來與 Laravel 一起使用。

另一方面，有些套件則是專門為 Laravel 使用而設計的。這些套件可能含有專門用於增強 Laravel 應用程式的路由、控制器、視圖和設定。本指南主要涵蓋這些 Laravel 專用套件的開發。


<a name="a-note-on-facades"></a>
### 關於 Facades 的注意事項

在撰寫 Laravel 應用程式時，通常使用契約 (Contracts) 或 facades 並無大礙，因為兩者基本上都提供相同等級的可測試性。然而，在撰寫套件時，您的套件通常無法存取 Laravel 所有的測試輔助程式。如果您希望能夠像將套件安裝在典型的 Laravel 應用程式中一樣撰寫套件測試，您可以使用 [Orchestral Testbench](https://github.com/orchestral/testbench) 套件。


<a name="package-discovery"></a>
## 套件探索

Laravel 應用程式的 `bootstrap/providers.php` 檔案包含了應該由 Laravel 載入的服務提供者列表。然而，您可以將服務提供者定義在套件 `composer.json` 檔案的 `extra` 區塊中，讓 Laravel 自動載入，而不需要要求使用者手動將您的服務提供者新增到列表中。除了服務提供者之外，您也可以列出任何您想要註冊的 [facades](/docs/{{version}}/facades)：

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

一旦您的套件配置好套件探索後，Laravel 將在安裝時自動註冊其服務提供者和 facades，為您的套件使用者創造便利的安裝體驗。


<a name="opting-out-of-package-discovery"></a>
#### 選擇退出套件探索

如果您是套件的使用者，並希望停用某個套件的套件探索功能，您可以在應用程式的 `composer.json` 檔案中的 `extra` 區塊列出該套件名稱：

```json
"extra": {
    "laravel": {
        "dont-discover": [
            "barryvdh/laravel-debugbar"
        ]
    }
},
```

您可以使用應用程式 `dont-discover` 指令內的 `*` 字元來停用所有套件的套件探索功能：

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

[服務提供者](/docs/{{version}}/providers)是您的套件與 Laravel 之間的連接點。服務提供者負責將事物綁定到 Laravel 的 [服務容器](/docs/{{version}}/container)，並通知 Laravel 從何處載入套件資源，例如視圖、設定和語言檔案。

服務提供者繼承自 `Illuminate\Support\ServiceProvider` 類別，並包含兩個方法：`register` 和 `boot`。基礎的 `ServiceProvider` 類別位於 `illuminate/support` Composer 套件中，您應該將其新增到您自己套件的依賴項目中。要深入瞭解服務提供者的結構和目的，請參閱[其文件](/docs/{{version}}/providers)。

<a name="resources"></a>
## 資源

<a name="configuration"></a>
### 設定

通常，您需要將套件的設定檔發布到應用程式的 `config` 目錄中。這將允許套件的使用者輕鬆地覆寫您的預設設定選項。要允許您的設定檔被發布，請在服務提供者的 `boot` 方法中呼叫 `publishes` 方法：

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

現在，當套件的使用者執行 Laravel 的 `vendor:publish` 指令時，您的檔案將被複製到指定的發布位置。一旦您的設定被發布，就可以像存取其他設定檔一樣存取其值：

```php
$value = config('courier.option');
```

> [!WARNING]
> 您不應該在設定檔中定義閉包 (Closures)。當使用者執行 `config:cache` Artisan 指令時，它們無法被正確地序列化。

<a name="default-package-configuration"></a>
#### 預設套件設定

您也可以將自己的套件設定檔與應用程式已發布的複本合併。這將允許您的使用者僅定義他們實際想要在已發布的設定檔複本中覆寫的選項。要合併設定檔的值，請在服務提供者的 `register` 方法中使用 `mergeConfigFrom` 方法。

`mergeConfigFrom` 方法接受套件設定檔的路徑作為其第一個參數，並接受應用程式設定檔複本的名稱作為其第二個參數：

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
> 此方法僅合併設定陣列的第一層。如果您的使用者部分定義了多維設定陣列，則缺失的選項將不會被合併。

<a name="routes"></a>
### 路由

如果您的套件包含路由，您可以使用 `loadRoutesFrom` 方法載入它們。此方法會自動判斷應用程式的路由是否已被快取，如果路由已經快取，則不會載入您的路由檔案：

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

如果您的套件包含 [資料庫遷移](/docs/{{version}}/migrations)，您可以使用 `publishesMigrations` 方法來告知 Laravel 給定的目錄或檔案包含遷移。當 Laravel 發布遷移時，它會自動更新檔名中的時間戳記，以反映目前的日期和時間：

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
### 語言檔案

如果您的套件包含 [語言檔案](/docs/{{version}}/localization)，您可以使用 `loadTranslationsFrom` 方法告知 Laravel 如何載入它們。例如，如果您的套件名稱為 `courier`，您應該在服務提供者的 `boot` 方法中加入以下內容：

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->loadTranslationsFrom(__DIR__.'/../lang', 'courier');
}
```

套件翻譯行是使用 `package::file.line` 語法慣例來引用的。因此，您可以像這樣從 `messages` 檔案中載入 `courier` 套件的 `welcome` 行：

```php
echo trans('courier::messages.welcome');
```

您可以使用 `loadJsonTranslationsFrom` 方法為您的套件註冊 JSON 翻譯檔案。此方法接受包含套件 JSON 翻譯檔案的目錄路徑：

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
#### 發布語言檔案

如果您想將套件的語言檔案發布到應用程式的 `lang/vendor` 目錄，您可以使用服務提供者的 `publishes` 方法。`publishes` 方法接受一個包含套件路徑及其所需發布位置的陣列。例如，要發布 `courier` 套件的語言檔案，您可以執行以下操作：

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

現在，當套件的使用者執行 Laravel 的 `vendor:publish` Artisan 指令時，您的套件語言檔案將被發布到指定的發布位置。

<a name="views"></a>
### 視圖

要向 Laravel 註冊您的套件 [視圖](/docs/{{version}}/views)，您需要告知 Laravel 視圖的位置。您可以使用服務提供者的 `loadViewsFrom` 方法來執行此操作。`loadViewsFrom` 方法接受兩個參數：視圖模板的路徑和您的套件名稱。例如，如果您的套件名稱為 `courier`，您應該在服務提供者的 `boot` 方法中加入以下內容：

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->loadViewsFrom(__DIR__.'/../resources/views', 'courier');
}
```

套件視圖是使用 `package::view` 語法慣例來引用的。因此，一旦您的視圖路徑在服務提供者中註冊，您就可以像這樣從 `courier` 套件載入 `dashboard` 視圖：

```php
Route::get('/dashboard', function () {
    return view('courier::dashboard');
});
```

<a name="overriding-package-views"></a>
#### 覆寫套件視圖

當您使用 `loadViewsFrom` 方法時，Laravel 實際上為您的視圖註冊了兩個位置：應用程式的 `resources/views/vendor` 目錄和您指定的目錄。因此，以 `courier` 套件為例，Laravel 首先會檢查開發者是否在 `resources/views/vendor/courier` 目錄中放置了自定義版本的視圖。接著，如果視圖沒有被自定義，Laravel 將在您呼叫 `loadViewsFrom` 時指定的套件視圖目錄中搜尋。這使得套件使用者能夠輕鬆地自定義或覆寫您的套件視圖。

<a name="publishing-views"></a>
#### 發布視圖

如果您想讓您的視圖可以發布到應用程式的 `resources/views/vendor` 目錄，您可以使用服務提供者的 `publishes` 方法。`publishes` 方法接受一個包含套件視圖路徑及其所需發布位置的陣列：

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

現在，當套件的使用者執行 Laravel 的 `vendor:publish` Artisan 指令時，您的套件視圖將被複製到指定的發布位置。

<a name="view-components"></a>
### 視圖元件

如果您正在開發一個使用 Blade 元件或將元件放置在非傳統目錄中的套件，您需要手動註冊您的元件類別及其 HTML 標籤別名，以便 Laravel 知道在哪裡可以找到該元件。您通常應該在套件服務提供者的 `boot` 方法中註冊您的元件：

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

或者，您可以使用 `componentNamespace` 方法按照慣例自動載入元件類別。例如，一個 `Nightshade` 套件可能擁有位於 `Nightshade\Views\Components` 命名空間下的 `Calendar` 和 `ColorPicker` 元件：

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

這將允許使用 `package-name::` 語法，透過開發者命名空間來使用套件元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 會藉由將元件名稱轉換為大駝峰式命名 (Pascal-case) 來自動偵測連結到該元件的類別。也支援使用「點 (dot)」標記法來指定子目錄。

<a name="anonymous-components"></a>
#### 匿名元件

如果您的套件包含匿名元件，則必須將它們放在套件「views」目錄下的 `components` 目錄中（如 [loadViewsFrom 方法](#views) 中所述）。接著，您可以透過在元件名稱前加上套件的視圖命名空間來渲染它們：

```blade
<x-courier::alert />
```

<a name="about-artisan-command"></a>
### "About" Artisan 指令

Laravel 內建的 `about` Artisan 指令提供了應用程式環境和設定的摘要。套件可以透過 `AboutCommand` 類別將額外資訊推送到此指令的輸出中。通常，可以從套件服務提供者的 `boot` 方法中加入此資訊：

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

若要向 Laravel 註冊您的套件 Artisan 指令，您可以使用 `commands` 方法。此方法需要一個包含指令類別名稱的陣列。指令註冊完成後，您可以使用 [Artisan CLI](/docs/{{version}}/artisan) 來執行它們：

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

Laravel 的 [最佳化指令](/docs/{{version}}/deployment#optimization) 會快取應用程式的設定、事件、路由和視圖。透過使用 `optimizes` 方法，您可以註冊套件專屬的 Artisan 指令，這些指令會在執行 `optimize` 和 `optimize:clear` 指令時被呼叫：

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

Laravel 的 [重新載入指令](/docs/{{version}}/deployment#reloading-services) 會終止任何正在運行的服務，以便系統程序監控器 (Process Monitor) 能自動重新啟動它們。透過使用 `reloads` 方法，您可以註冊套件專屬的 Artisan 指令，這些指令會在執行 `reload` 指令時被呼叫：

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
## 公開資產

您的套件可能會包含 JavaScript、CSS 和圖片等資產。若要將這些資產發布到應用程式的 `public` 目錄，請使用服務提供者的 `publishes` 方法。在此範例中，我們還會加入一個 `public` 資產群組標籤，這可以用來輕鬆發布相關聯的資產群組：

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

現在，當套件的使用者執行 `vendor:publish` 指令時，您的資產將會被複製到指定的發布位置。由於使用者通常需要在每次套件更新時覆寫資產，因此他們可以使用 `--force` 旗標：

```shell
php artisan vendor:publish --tag=public --force
```


<a name="publishing-file-groups"></a>
## 發布檔案群組

您可能希望分別發布套件資產和資源群組。例如，您可能希望讓使用者發布套件的設定檔，而不必強迫他們也發布套件的資產。您可以透過在套件服務提供者的 `publishes` 方法中對其進行「標記 (Tagging)」來達成。例如，讓我們在套件服務提供者的 `boot` 方法中使用標記為 `courier` 套件定義兩個發布群組 (`courier-config` 和 `courier-migrations`)：

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

現在，您的使用者可以透過在執行 `vendor:publish` 指令時引用標籤來分別發布這些群組：

```shell
php artisan vendor:publish --tag=courier-config
```

使用者也可以使用 `--provider` 旗標來發布由您套件服務提供者定義的所有可發布檔案：

```shell
php artisan vendor:publish --provider="Your\Package\ServiceProvider"
```