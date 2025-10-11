# 服務提供者

- [簡介](#introduction)
- [撰寫服務提供者](#writing-service-providers)
    - [註冊方法](#the-register-method)
    - [啟動方法](#the-boot-method)
- [註冊服務提供者](#registering-providers)
- [延遲服務提供者](#deferred-providers)

<a name="introduction"></a>
## 簡介

服務提供者是所有 Laravel 應用程式啟動的中心。您的應用程式以及所有 Laravel 的核心服務，都是透過服務提供者來啟動。

但，「啟動」是什麼意思？一般來說，我們指的是**註冊**各種事物，包括註冊服務容器繫結、事件監聽器、中介層，甚至是路由。服務提供者是設定應用程式的中心。

Laravel 內部使用數十個服務提供者來啟動其核心服務，例如郵件、佇列、快取等。許多服務提供者是「延遲」提供者，這表示它們不會在每個請求時都載入，而只會在實際需要它們提供的服務時才載入。

所有使用者定義的服務提供者都註冊在 `bootstrap/providers.php` 檔案中。在接下來的文件中，您將學習如何撰寫自己的服務提供者並將其註冊到您的 Laravel 應用程式中。

> [!NOTE]  
> 如果您想了解更多關於 Laravel 如何處理請求以及其內部運作方式，請查閱我們關於 Laravel [請求生命週期](/docs/{{version}}/lifecycle)的文件。

<a name="writing-service-providers"></a>
## 撰寫服務提供者

所有服務提供者都繼承自 `Illuminate\Support\ServiceProvider` 類別。大多數服務提供者包含一個 `register` 方法和一個 `boot` 方法。在 `register` 方法中，您應該**只將事物繫結到 [服務容器](/docs/{{version}}/container) 中**。您不應該嘗試在 `register` 方法中註冊任何事件監聽器、路由或任何其他功能。

Artisan CLI 可以透過 `make:provider` 命令產生新的提供者。Laravel 會自動將您新的提供者註冊到應用程式的 `bootstrap/providers.php` 檔案中：

```shell
php artisan make:provider RiakServiceProvider
```

<a name="the-register-method"></a>
### 註冊方法

如前所述，在 `register` 方法中，您應該只將事物繫結到 [服務容器](/docs/{{version}}/container) 中。您不應該嘗試在 `register` 方法中註冊任何事件監聽器、路由或任何其他功能。否則，您可能會不小心使用了尚未載入的服務提供者所提供的服務。

讓我們看看一個基本的服務提供者。在您的任何服務提供者方法中，您都可以存取 `$app` 屬性，該屬性提供對服務容器的存取權限：

    <?php

    namespace App\Providers;

    use App\Services\Riak\Connection;
    use Illuminate\Contracts\Foundation\Application;
    use Illuminate\Support\ServiceProvider;

    class RiakServiceProvider extends ServiceProvider
    {
        /**
         * Register any application services.
         */
        public function register(): void
        {
            $this->app->singleton(Connection::class, function (Application $app) {
                return new Connection(config('riak'));
            });
        }
    }

這個服務提供者只定義了一個 `register` 方法，並使用該方法在服務容器中定義 `App\Services\Riak\Connection` 的實作。如果您還不熟悉 Laravel 的服務容器，請查閱 [其文件](/docs/{{version}}/container)。

<a name="the-bindings-and-singletons-properties"></a>
#### `bindings` 和 `singletons` 屬性

如果您的服務提供者註冊了許多簡單的繫結，您可能希望使用 `bindings` 和 `singletons` 屬性，而不是手動註冊每個容器繫結。當框架載入服務提供者時，它會自動檢查這些屬性並註冊它們的繫結：

    <?php

    namespace App\Providers;

    use App\Contracts\DowntimeNotifier;
    use App\Contracts\ServerProvider;
    use App\Services\DigitalOceanServerProvider;
    use App\Services\PingdomDowntimeNotifier;
    use App\Services\ServerToolsProvider;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * All of the container bindings that should be registered.
         *
         * @var array
         */
        public $bindings = [
            ServerProvider::class => DigitalOceanServerProvider::class,
        ];

        /**
         * All of the container singletons that should be registered.
         *
         * @var array
         */
        public $singletons = [
            DowntimeNotifier::class => PingdomDowntimeNotifier::class,
            ServerProvider::class => ServerToolsProvider::class,
        ];
    }

<a name="the-boot-method"></a>
### 啟動方法

那麼，如果我們需要在服務提供者中註冊一個 [視圖 Composer](/docs/{{version}}/views#view-composers) 呢？這應該在 `boot` 方法中完成。**這個方法會在所有其他服務提供者註冊完畢後呼叫**，這表示您可以存取框架已註冊的所有其他服務：

    <?php

    namespace App\Providers;

    use Illuminate\Support\Facades\View;
    use Illuminate\Support\ServiceProvider;

    class ComposerServiceProvider extends ServiceProvider
    {
        /**
         * Bootstrap any application services.
         */
        public function boot(): void
        {
            View::composer('view', function () {
                // ...
            });
        }
    }

<a name="boot-method-dependency-injection"></a>
#### 啟動方法依賴注入

您可以在服務提供者的 `boot` 方法中對依賴項進行型別提示。[服務容器](/docs/{{version}}/container) 會自動注入您需要的任何依賴項：

    use Illuminate\Contracts\Routing\ResponseFactory;

    /**
     * Bootstrap any application services.
     */
    public function boot(ResponseFactory $response): void
    {
        $response->macro('serialized', function (mixed $value) {
            // ...
        });
    }

<a name="registering-providers"></a>
## 註冊服務提供者

所有服務提供者都註冊在 `bootstrap/providers.php` 設定檔中。這個檔案會回傳一個包含您的應用程式服務提供者類別名稱的陣列：

    <?php

    return [
        App\Providers\AppServiceProvider::class,
    ];

當您呼叫 `make:provider` Artisan 命令時，Laravel 會自動將生成的提供者新增到 `bootstrap/providers.php` 檔案中。然而，如果您是手動建立提供者類別，您應該手動將提供者類別新增到陣列中：

    <?php

    return [
        App\Providers\AppServiceProvider::class,
        App\Providers\ComposerServiceProvider::class, // [tl! add]
    ];

<a name="deferred-providers"></a>
## 延遲服務提供者

如果您的服務提供者**僅僅**在 [服務容器](/docs/{{version}}/container) 中註冊綁定，您可以選擇延遲其註冊，直到其中一個註冊的綁定實際被需要時才執行。延遲載入這類型的服務提供者將提升應用程式的效能，因為它不會在每個請求時都從檔案系統中載入。

Laravel 會編譯並儲存一份由延遲服務提供者所提供的所有服務清單，連同其服務提供者類別名稱。接著，只有當您嘗試解析其中一個服務時，Laravel 才會載入該服務提供者。

若要延遲載入服務提供者，請實作 `\Illuminate\Contracts\Support\DeferrableProvider` 介面並定義一個 `provides` 方法。此 `provides` 方法應該回傳該服務提供者所註冊的服務容器綁定：

    <?php

    namespace App\Providers;

    use App\Services\Riak\Connection;
    use Illuminate\Contracts\Foundation\Application;
    use Illuminate\Contracts\Support\DeferrableProvider;
    use Illuminate\Support\ServiceProvider;

    class RiakServiceProvider extends ServiceProvider implements DeferrableProvider
    {
        /**
         * Register any application services.
         */
        public function register(): void
        {
            $this->app->singleton(Connection::class, function (Application $app) {
                return new Connection($app['config']['riak']);
            });
        }

        /**
         * Get the services provided by the provider.
         *
         * @return array<int, string>
         */
        public function provides(): array
        {
            return [Connection::class];
        }
    }