# 服務提供者

- [簡介](#introduction)
- [編寫服務提供者](#writing-service-providers)
    - [Register 方法](#the-register-method)
    - [Boot 方法](#the-boot-method)
- [註冊服務提供者](#registering-providers)
- [延遲服務提供者](#deferred-providers)

<a name="introduction"></a>
## 簡介

服務提供者 (Service providers) 是所有 Laravel 應用程式啟動引導的核心。您的應用程式以及所有 Laravel 的核心服務，都是透過服務提供者來啟動引導的。

但是，「啟動引導 (bootstrapped)」是什麼意思呢？一般來說，我們指的是**註冊 (registering)** 各種事物，包括註冊服務容器綁定 (service container bindings)、事件監聽器 (event listeners)、中介層 (middleware)，甚至是路由 (routes)。服務提供者是配置您應用程式的核心所在。

Laravel 內部使用數十個服務提供者來啟動引導其核心服務，例如郵件發送器 (mailer)、佇列 (queue)、快取 (cache) 等。其中許多服務提供者是「延遲」服務提供者，這表示它們不會在每次請求時都載入，而只在實際需要其所提供的服務時才載入。

所有使用者定義的服務提供者都註冊在 `bootstrap/providers.php` 檔案中。在接下來的說明文件中，您將學習如何編寫自己的服務提供者並將它們註冊到您的 Laravel 應用程式中。

> [!NOTE]
> 如果您想進一步了解 Laravel 如何處理請求以及其內部運作方式，請查閱我們關於 Laravel [請求生命週期](/docs/{{version}}/lifecycle)的說明文件。

<a name="writing-service-providers"></a>
## 編寫服務提供者

所有的服務提供者都繼承 `Illuminate\Support\ServiceProvider` 類別。大多數服務提供者都包含一個 `register` 方法和一個 `boot` 方法。在 `register` 方法中，您應該**只將事物綁定到 [服務容器](/docs/{{version}}/container) 中**。您絕不應該在 `register` 方法中嘗試註冊任何事件監聽器、路由或任何其他功能。

Artisan CLI 可以透過 `make:provider` 命令產生一個新的服務提供者。Laravel 會自動在您應用程式的 `bootstrap/providers.php` 檔案中註冊您的新服務提供者：

```shell
php artisan make:provider RiakServiceProvider
```

<a name="the-register-method"></a>
### Register 方法

如前所述，在 `register` 方法中，您應該只將事物綁定到 [服務容器](/docs/{{version}}/container) 中。您絕不應該在 `register` 方法中嘗試註冊任何事件監聽器、路由或任何其他功能。否則，您可能會不小心使用了由尚未載入的服務提供者所提供的服務。

讓我們看看一個基本的服務提供者。在您的任何服務提供者方法中，您始終可以存取 `$app` 屬性，它提供了對服務容器的存取權限：

```php
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
```

這個服務提供者只定義了一個 `register` 方法，並使用該方法在服務容器中定義 `App\Services\Riak\Connection` 的實作。如果您還不熟悉 Laravel 的服務容器，請查閱 [其說明文件](/docs/{{version}}/container)。

<a name="the-bindings-and-singletons-properties"></a>
#### `bindings` 與 `singletons` 屬性

如果您的服務提供者註冊了許多簡單的綁定，您可能會希望使用 `bindings` 和 `singletons` 屬性，而不是手動註冊每個容器綁定。當框架載入服務提供者時，它會自動檢查這些屬性並註冊其綁定：

```php
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
```

<a name="the-boot-method"></a>
### Boot 方法

那麼，如果我們需要在服務提供者中註冊 [視圖組件 (view composer)](/docs/{{version}}/views#view-composers) 怎麼辦？這應該在 `boot` 方法中完成。**此方法會在所有其他服務提供者註冊完成後呼叫**，這表示您可以存取框架已註冊的所有其他服務：

```php
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
```

<a name="boot-method-dependency-injection"></a>
#### Boot 方法依賴注入

您可以為服務提供者的 `boot` 方法型別提示依賴。 [服務容器](/docs/{{version}}/container) 會自動注入您需要的任何依賴：

```php
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
```

<a name="registering-providers"></a>
## 註冊服務提供者

所有的服務提供者都註冊在 `bootstrap/providers.php` 設定檔中。此檔案會回傳一個包含您應用程式服務提供者類別名稱的陣列：

```php
<?php

return [
    App\Providers\AppServiceProvider::class,
];
```

當您呼叫 `make:provider` Artisan 命令時，Laravel 會自動將產生的服務提供者新增到 `bootstrap/providers.php` 檔案中。但是，如果您是手動建立服務提供者類別，則應手動將服務提供者類別新增到陣列中：

```php
<?php

return [
    App\Providers\AppServiceProvider::class,
    App\Providers\ComposerServiceProvider::class, // [tl! add]
];
```

<a name="deferred-providers"></a>
## 延遲服務提供者

如果您的提供者**只**在 [服務容器](/docs/{{version}}/container) 中註冊綁定，您可以選擇延遲其註冊，直到實際需要其中一個註冊的綁定時才進行。延遲載入此類提供者將提高您應用程式的效能，因為它不會在每次請求時都從檔案系統中載入。

Laravel 會編譯並儲存由延遲服務提供者提供的所有服務列表，以及其服務提供者類別的名稱。然後，只有當您嘗試解析其中一個服務時，Laravel 才會載入該服務提供者。

要延遲提供者的載入，請實作 `\Illuminate\Contracts\Support\DeferrableProvider` 介面並定義一個 `provides` 方法。`provides` 方法應回傳由提供者註冊的服務容器綁定：

```php
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
```