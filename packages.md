# 套件開發

- [簡介](#introduction)
    - [關於 Facades 的注意事項](#a-note-on-facades)
- [套件自動發現](#package-discovery)
- [服務提供者](#service-providers)
- [資源](#resources)
    - [設定](#configuration)
    - [資料庫遷移](#migrations)
    - [路由](#routes)
    - [語言檔案](#language-files)
    - [視圖](#views)
    - [視圖元件](#view-components)
    - ["about" Artisan 命令](#about-artisan-command)
- [命令](#commands)
    - [優化命令](#optimize-commands)
- [公開資源](#public-assets)
- [發佈檔案群組](#publishing-file-groups)

<a name="introduction"></a>
## 簡介

套件是為 Laravel 新增功能的主要方式。套件可以是任何東西，從像 [Carbon](https://github.com/briannesbitt/Carbon) 這樣用於處理日期的絕佳工具，到像 Spatie 的 [Laravel Media Library](https://github.com/spatie/laravel-medialibrary) 這樣允許您將檔案與 Eloquent 模型關聯的套件。

套件有不同類型。有些套件是獨立的，這表示它們可以與任何 PHP 框架搭配運作。Carbon 和 Pest 就是獨立套件的範例。這些套件中的任何一個都可以透過在 `composer.json` 檔案中引入來與 Laravel 搭配使用。

另一方面，其他套件則是專為 Laravel 設計。這些套件可能包含路由、控制器、視圖和設定，專門用於增強 Laravel 應用程式。本指南主要涵蓋專為 Laravel 設計的套件開發。

<a name="a-note-on-facades"></a>
### 關於 Facades 的注意事項

編寫 Laravel 應用程式時，使用契約 (contracts) 或 Facades 通常沒有關係，因為兩者都提供實質上同等的測試能力。然而，當編寫套件時，您的套件通常無法存取 Laravel 的所有測試輔助工具。如果您希望能夠像套件安裝在典型的 Laravel 應用程式中一樣編寫您的套件測試，您可以使用 [Orchestral Testbench](https://github.com/orchestral/testbench) 套件。

<a name="package-discovery"></a>
## 套件自動發現

Laravel 應用程式的 `bootstrap/providers.php` 檔案包含應由 Laravel 載入的服務提供者列表。但是，您無需讓使用者手動將您的服務提供者新增到列表中，您可以在套件的 `composer.json` 檔案的 `extra` 區段中定義該提供者，以便 Laravel 自動載入它。除了服務提供者之外，您還可以列出任何您希望註冊的 [Facades](/docs/{{version}}/facades)：

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

一旦您的套件配置為可被自動發現，Laravel 將在安裝時自動註冊其服務提供者和 Facades，為您的套件使用者創造便利的安裝體驗。

<a name="opting-out-of-package-discovery"></a>
#### 停用套件自動發現

如果您是套件的使用者，並且希望停用特定套件的自動發現，您可以在應用程式的 `composer.json` 檔案的 `extra` 區段中列出該套件的名稱：

```json
"extra": {
    "laravel": {
        "dont-discover": [
            "barryvdh/laravel-debugbar"
        ]
    }
},
```

您可以使用應用程式 `dont-discover` 指令中的 `*` 字元來停用所有套件的自動發現：

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

[服務提供者](/docs/{{version}}/providers) 是您的套件與 Laravel 之間的連接點。服務提供者負責將內容綁定到 Laravel 的 [服務容器](/docs/{{version}}/container) 中，並告知 Laravel 從何處載入套件資源，例如視圖、設定和語言檔案。

服務提供者擴充 `Illuminate\Support\ServiceProvider` 類別，並包含兩個方法：`register` 和 `boot`。基礎的 `ServiceProvider` 類別位於 `illuminate/support` Composer 套件中，您應該將其新增到您自己套件的依賴中。要了解有關服務提供者結構和目的的更多資訊，請查閱 [其文件](/docs/{{version}}/providers)。

<a name="resources"></a>
## 資源

<a name="configuration"></a>
### 設定

通常，您會需要將套件的設定檔發佈到應用程式的 `config` 目錄。這會讓套件的使用者可以輕鬆地覆寫您的預設設定選項。若要允許您的設定檔被發佈，請從服務提供者的 `boot` 方法中呼叫 `publishes` 方法：

    /**
     * Bootstrap any package services.
     */
    public function boot(): void
    {
        $this->publishes([
            __DIR__.'/../config/courier.php' => config_path('courier.php'),
        ]);
    }

現在，當您套件的使用者執行 Laravel 的 `vendor:publish` 命令時，您的檔案將會被複製到指定的發佈位置。一旦您的設定被發佈，其值就可以像存取任何其他設定檔一樣被存取：

    $value = config('courier.option');

> [!WARNING]  
> 您不應在設定檔中定義閉包。當使用者執行 `config:cache` Artisan 命令時，它們無法被正確地序列化。

<a name="default-package-configuration"></a>
#### 預設套件設定

您也可以將自己的套件設定檔與應用程式已發佈的副本合併。這將允許您的使用者只定義他們實際想要在設定檔已發佈副本中覆寫的選項。若要合併設定檔值，請在您的服務提供者的 `register` 方法中使用 `mergeConfigFrom` 方法。

`mergeConfigFrom` 方法接受您的套件設定檔路徑作為其第一個引數，以及應用程式設定檔副本的名稱作為其第二個引數：

    /**
     * Register any application services.
     */
    public function register(): void
    {
        $this->mergeConfigFrom(
            __DIR__.'/../config/courier.php', 'courier'
        );
    }

> [!WARNING]  
> 此方法只會合併設定陣列的第一層。如果您的使用者只部分定義了多維度設定陣列，則遺失的選項將不會被合併。

<a name="routes"></a>
### 路由

如果您的套件包含路由，您可以使用 `loadRoutesFrom` 方法來載入它們。此方法會自動判斷應用程式的路由是否已快取，如果路由已被快取，則不會載入您的路由檔案：

    /**
     * Bootstrap any package services.
     */
    public function boot(): void
    {
        $this->loadRoutesFrom(__DIR__.'/../routes/web.php');
    }

<a name="migrations"></a>
### 資料庫遷移

如果您的套件包含[資料庫遷移](/docs/{{version}}/migrations)，您可以使用 `publishesMigrations` 方法來告知 Laravel 給定的目錄或檔案包含遷移。當 Laravel 發佈這些遷移時，它會自動更新其檔案名稱內的時間戳記，以反映目前的日期和時間：

    /**
     * Bootstrap any package services.
     */
    public function boot(): void
    {
        $this->publishesMigrations([
            __DIR__.'/../database/migrations' => database_path('migrations'),
        ]);
    }

<a name="language-files"></a>
### 語言檔案

如果您的套件包含[語言檔案](/docs/{{version}}/localization)，您可以使用 `loadTranslationsFrom` 方法來告知 Laravel 如何載入它們。舉例來說，如果您的套件名稱為 `courier`，您應將以下內容新增到服務提供者的 `boot` 方法中：

    /**
     * Bootstrap any package services.
     */
    public function boot(): void
    {
        $this->loadTranslationsFrom(__DIR__.'/../lang', 'courier');
    }

套件的翻譯行是使用 `package::file.line` 語法慣例來參照的。因此，您可以從 `messages` 檔案中載入 `courier` 套件的 `welcome` 行，像這樣：

    echo trans('courier::messages.welcome');

您可以使用 `loadJsonTranslationsFrom` 方法來為您的套件註冊 JSON 翻譯檔案。此方法接受包含您的套件 JSON 翻譯檔案的目錄路徑：

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
#### 發佈語言檔案

如果您想將套件的語言檔案發佈到應用程式的 `lang/vendor` 目錄，您可以使用服務提供者的 `publishes` 方法。`publishes` 方法接受一個包含套件路徑及其所需發佈位置的陣列。舉例來說，若要發佈 `courier` 套件的語言檔案，您可以執行以下操作：

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

現在，當您套件的使用者執行 Laravel 的 `vendor:publish` Artisan 命令時，您的套件語言檔案將會被發佈到指定的發佈位置。

<a name="views"></a>
### 視圖

若要向 Laravel 註冊您的套件[視圖](/docs/{{version}}/views)，您需要告知 Laravel 視圖位於何處。您可以使用服務提供者的 `loadViewsFrom` 方法來完成此操作。`loadViewsFrom` 方法接受兩個引數：您的視圖模板路徑以及您的套件名稱。舉例來說，如果您的套件名稱為 `courier`，您應將以下內容新增到服務提供者的 `boot` 方法中：

    /**
     * Bootstrap any package services.
     */
    public function boot(): void
    {
        $this->loadViewsFrom(__DIR__.'/../resources/views', 'courier');
    }

套件視圖是使用 `package::view` 語法慣例來參照的。因此，一旦您的視圖路徑在服務提供者中註冊，您可以像這樣從 `courier` 套件中載入 `dashboard` 視圖：

    Route::get('/dashboard', function () {
        return view('courier::dashboard');
    });

<a name="overriding-package-views"></a>
#### 覆寫套件視圖

當您使用 `loadViewsFrom` 方法時，Laravel 實際上會為您的視圖註冊兩個位置：應用程式的 `resources/views/vendor` 目錄以及您指定的目錄。因此，以 `courier` 套件為例，Laravel 會首先檢查開發者是否已將視圖的自訂版本放置在 `resources/views/vendor/courier` 目錄中。然後，如果視圖尚未被自訂，Laravel 將會在您呼叫 `loadViewsFrom` 時指定的套件視圖目錄中搜尋。這使得套件使用者可以輕鬆地自訂／覆寫您的套件視圖。

<a name="publishing-views"></a>
#### 發佈視圖

如果您想讓您的視圖可發佈到應用程式的 `resources/views/vendor` 目錄，您可以使用服務提供者的 `publishes` 方法。`publishes` 方法接受一個包含套件視圖路徑及其所需發佈位置的陣列：

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

現在，當您套件的使用者執行 Laravel 的 `vendor:publish` Artisan 命令時，您的套件視圖將會被複製到指定的發佈位置。

<a name="view-components"></a>
### 視圖元件

如果您正在建構一個使用 Blade 元件的套件，或者將元件放置在非慣例的目錄中，您需要手動註冊您的元件類別及其 HTML 標籤別名，以便 Laravel 知道在哪裡找到該元件。您通常應該在套件服務提供者的 `boot` 方法中註冊您的元件：

    use Illuminate\Support\Facades\Blade;
    use VendorPackage\View\Components\AlertComponent;

    /**
     * Bootstrap your package's services.
     */
    public function boot(): void
    {
        Blade::component('package-alert', AlertComponent::class);
    }

一旦您的元件註冊完成，即可使用其標籤別名進行渲染：

```blade
<x-package-alert/>
```


<a name="autoloading-package-components"></a>
#### 自動載入套件元件

另一種方式是，您可以使用 `componentNamespace` 方法依慣例自動載入元件類別。例如，一個 `Nightshade` 套件可能擁有位於 `Nightshade\Views\Components` 命名空間中的 `Calendar` 和 `ColorPicker` 元件：

    use Illuminate\Support\Facades\Blade;

    /**
     * Bootstrap your package's services.
     */
    public function boot(): void
    {
        Blade::componentNamespace('Nightshade\\Views\\Components', 'nightshade');
    }

這將允許您透過供應商命名空間，使用 `package-name::` 語法來使用套件元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 會透過將元件名稱轉換為 Pascal 大小寫，自動偵測連結到此元件的類別。子目錄也支援使用「點」記號。


<a name="anonymous-components"></a>
#### 匿名元件

如果您的套件包含匿名元件，它們必須放置在套件「視圖」目錄中的 `components` 目錄中 (由 [`loadViewsFrom` 方法](#views) 指定)。然後，您可以透過為元件名稱加上套件的視圖命名空間作為前綴來渲染它們：

```blade
<x-courier::alert />
```


<a name="about-artisan-command"></a>
### "about" Artisan 命令

Laravel 內建的 `about` Artisan 命令提供了應用程式環境和設定的概述。套件可以透過 `AboutCommand` 類別向此命令的輸出推送額外資訊。通常，此資訊可以從您的套件服務提供者的 `boot` 方法中添加：

    use Illuminate\Foundation\Console\AboutCommand;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        AboutCommand::add('My Package', fn () => ['Version' => '1.0.0']);
    }

<a name="commands"></a>
## 命令

若要向 Laravel 註冊套件的 Artisan 命令，您可以使用 `commands` 方法。此方法預期一個命令類別名稱的陣列。一旦命令註冊完成，您便可使用 [Artisan CLI](/docs/{{version}}/artisan) 執行它們：

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


<a name="optimize-commands"></a>
### 優化命令

Laravel 的 [`optimize` 命令](/docs/{{version}}/deployment#optimization) 會快取應用程式的設定、事件、路由和視圖。您可以使用 `optimizes` 方法，註冊套件本身的 Artisan 命令，這些命令應在 `optimize` 和 `optimize:clear` 命令執行時被呼叫：

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


<a name="public-assets"></a>
## 公開資源

您的套件可能包含 JavaScript、CSS 和圖片等資源。若要將這些資源發佈到應用程式的 `public` 目錄，請使用服務提供者的 `publishes` 方法。在此範例中，我們還會加入一個 `public` 資源群組標籤，它可以用來輕鬆發佈相關的資源群組：

    /**
     * Bootstrap any package services.
     */
    public function boot(): void
    {
        $this->publishes([
            __DIR__.'/../public' => public_path('vendor/courier'),
        ], 'public');
    }

現在，當套件使用者執行 `vendor:publish` 命令時，您的資源將會被複製到指定的發佈位置。由於使用者通常需要在每次套件更新時覆寫這些資源，您可以使用 `--force` 旗標：

```shell
php artisan vendor:publish --tag=public --force
```


<a name="publishing-file-groups"></a>
## 發佈檔案群組

您可能希望分開發佈套件的資源和檔案群組。例如，您可能希望讓使用者發佈套件的設定檔，而不需要被迫發佈套件的資源。您可以在從套件的服務提供者呼叫 `publishes` 方法時，透過「標記」它們來實現此目的。例如，讓我們先在套件服務提供者的 `boot` 方法中，使用標籤為 `courier` 套件定義兩個發佈群組 (`courier-config` 和 `courier-migrations`)：

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

現在，您的使用者可以在執行 `vendor:publish` 命令時，透過參考其標籤來分開發佈這些群組：

```shell
php artisan vendor:publish --tag=courier-config
```