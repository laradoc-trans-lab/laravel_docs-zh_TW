# Service Providers

- [簡介](#introduction)
- [撰寫 Service Providers](#writing-service-providers)
    - [Register 方法](#the-register-method)
    - [Boot 方法](#the-boot-method)
- [註冊 Providers](#registering-providers)
- [延遲載入的 Providers](#deferred-providers)

<a name="introduction"></a>
## 簡介

Service Providers 是所有 Laravel 應用程式啟動的核心。您的應用程式以及所有 Laravel 的核心服務，都是透過 Service Providers 啟動的。

那麼，「啟動」是什麼意思呢？一般來說，我們指的是**註冊**各種事物，包括註冊 Service Container 綁定、事件監聽器、Middleware，甚至是路由。Service Providers 是設定應用程式的核心位置。

Laravel 內部使用了數十個 Service Providers 來啟動其核心服務，例如郵件、佇列、快取等。其中許多 Providers 都是「延遲載入的 Providers」，這表示它們不會在每個請求時都載入，而只會在實際需要其提供的服務時才載入。

所有使用者定義的 Service Providers 都註冊在 `bootstrap/providers.php` 檔案中。在接下來的文件中，您將學習如何撰寫自己的 Service Providers 並將其註冊到您的 Laravel 應用程式中。

> [!NOTE]  
> 如果您想了解更多關於 Laravel 如何處理請求以及其內部運作方式，請查閱我們關於 Laravel [請求生命週期](/docs/{{version}}/lifecycle)的說明文件。

<a name="writing-service-providers"></a>
## 撰寫 Service Providers

所有的 Service Providers 都會繼承 `Illuminate\Support\ServiceProvider` 類別。大多數 Service Providers 都包含 `register` 和 `boot` 方法。在 `register` 方法中，您應該**只將事物綁定到 [Service Container](/docs/{{version}}/container) 中**。您絕不應該嘗試在 `register` 方法中註冊任何事件監聽器、路由或任何其他功能。

Artisan CLI 可以透過 `make:provider` 命令產生一個新的 Provider。Laravel 會自動將您新的 Provider 註冊到應用程式的 `bootstrap/providers.php` 檔案中：

```shell
php artisan make:provider RiakServiceProvider
```

<a name="the-register-method"></a>
### Register 方法

如前所述，在 `register` 方法中，您應該只將事物綁定到 [Service Container](/docs/{{version}}/container) 中。您絕不應該嘗試在 `register` 方法中註冊任何事件監聽器、路由或任何其他功能。否則，您可能會不小心使用了由尚未載入的 Service Provider 所提供的服務。

讓我們看看一個基本的 Service Provider。在您的任何 Service Provider 方法中，您都可以存取 `$app` 屬性，該屬性提供了對 Service Container 的存取：

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

這個 Service Provider 只定義了一個 `register` 方法，並使用該方法在 Service Container 中定義了 `App\Services\Riak\Connection` 的實作。如果您還不熟悉 Laravel 的 Service Container，請查閱[其說明文件](/docs/{{version}}/container)。

<a name="the-bindings-and-singletons-properties"></a>
#### `bindings` 與 `singletons` 屬性

如果您的 Service Provider 註冊了許多簡單的綁定，您可能希望使用 `bindings` 和 `singletons` 屬性，而不是手動註冊每個容器綁定。當框架載入 Service Provider 時，它會自動檢查這些屬性並註冊其綁定：

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
### Boot 方法

那麼，如果我們需要在 Service Provider 中註冊一個 [View Composer](/docs/{{version}}/views#view-composers) 該怎麼辦？這應該在 `boot` 方法中完成。**此方法會在所有其他 Service Providers 都註冊完畢後呼叫**，這表示您可以存取框架已註冊的所有其他服務：

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
#### Boot 方法的依賴注入

您可以為 Service Provider 的 `boot` 方法型別提示依賴。 [Service Container](/docs/{{version}}/container) 會自動注入您需要的任何依賴：

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
## 註冊 Providers

所有的 Service Providers 都註冊在 `bootstrap/providers.php` 設定檔中。此檔案會回傳一個包含應用程式 Service Providers 類別名稱的陣列：

    <?php

    return [
        App\Providers\AppServiceProvider::class,
    ];

當您呼叫 `make:provider` Artisan 命令時，Laravel 會自動將產生的 Provider 加入到 `bootstrap/providers.php` 檔案中。但是，如果您是手動建立 Provider 類別，則應該手動將 Provider 類別加入到陣列中：

    <?php

    return [
        App\Providers\AppServiceProvider::class,
        App\Providers\ComposerServiceProvider::class, // [tl! add]
    ];

<a name="deferred-providers"></a>
## 延遲載入的 Providers

如果您的 Provider **只**在 [Service Container](/docs/{{version}}/container) 中註冊綁定，您可以選擇延遲其註冊，直到實際需要其中一個已註冊的綁定時才載入。延遲載入此類 Provider 將提高應用程式的效能，因為它不會在每個請求時都從檔案系統載入。

Laravel 會編譯並儲存所有由延遲載入的 Service Providers 所提供的服務列表，以及其 Service Provider 類別的名稱。然後，只有當您嘗試解析其中一個服務時，Laravel 才會載入該 Service Provider。

要延遲載入 Provider，請實作 `\Illuminate\Contracts\Support\DeferrableProvider` 介面並定義 `provides` 方法。`provides` 方法應該回傳由 Provider 註冊的 Service Container 綁定：

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
