# Facades

- [簡介](#introduction)
- [何時使用 Facades](#when-to-use-facades)
    - [Facades 與依賴注入的比較](#facades-vs-dependency-injection)
    - [Facades 與輔助函式的比較](#facades-vs-helper-functions)
- [Facades 的運作方式](#how-facades-work)
- [即時 Facades](#real-time-facades)
- [Facade 類別參考](#facade-class-reference)

<a name="introduction"></a>
## 簡介

在整個 Laravel 說明文件中，您會看到透過「facades」與 Laravel 功能互動的程式碼範例。Facades 為應用程式 [Service Container](/docs/{{version}}/container) 中可用的類別提供了「靜態」介面。Laravel 內建了許多 Facades，提供了對幾乎所有 Laravel 功能的存取。

Laravel Facades 作為 Service Container 中底層類別的「靜態代理」，提供了簡潔、富有表達力的語法優勢，同時比傳統靜態方法保持了更好的可測試性和靈活性。如果您不完全理解 Facades 的運作方式，這完全沒問題 —— 只要順其自然，繼續學習 Laravel 即可。

所有 Laravel 的 Facades 都定義在 `Illuminate\Support\Facades` 命名空間中。因此，我們可以輕鬆地存取 Facade，如下所示：

    use Illuminate\Support\Facades\Cache;
    use Illuminate\Support\Facades\Route;

    Route::get('/cache', function () {
        return Cache::get('key');
    });

在整個 Laravel 說明文件中，許多範例都會使用 Facades 來展示框架的各種功能。

<a name="helper-functions"></a>
#### 輔助函式

為了輔助 Facades，Laravel 提供了各種全域「輔助函式」，讓與常見 Laravel 功能的互動變得更加容易。您可能會用到的一些常見輔助函式包括 `view`、`response`、`url`、`config` 等。Laravel 提供的每個輔助函式都在其對應的功能說明文件中進行了說明；然而，完整的列表可在專用的 [輔助函式說明文件](/docs/{{version}}/helpers) 中找到。

例如，我們可以使用 `response` 函式來產生 JSON 回應，而不是使用 `Illuminate\Support\Facades\Response` Facade。由於輔助函式是全域可用的，您無需匯入任何類別即可使用它們：

    use Illuminate\Support\Facades\Response;

    Route::get('/users', function () {
        return Response::json([
            // ...
        ]);
    });

    Route::get('/users', function () {
        return response()->json([
            // ...
        ]);
    });

<a name="when-to-use-facades"></a>
## 何時使用 Facades

Facades 有許多優點。它們提供了簡潔、易記的語法，讓您無需記住必須手動注入或設定的冗長類別名稱即可使用 Laravel 的功能。此外，由於它們獨特地使用了 PHP 的動態方法，因此易於測試。

然而，使用 Facades 時必須謹慎。Facades 的主要危險是類別「職責蔓延」。由於 Facades 非常易於使用且不需要注入，因此很容易讓您的類別不斷增長並在單一類別中使用許多 Facades。使用依賴注入時，大型建構函式提供的視覺回饋會提醒您類別變得過於龐大，從而減輕了這種潛在問題。因此，在使用 Facades 時，請特別注意類別的大小，以確保其職責範圍保持狹窄。如果您的類別變得過於龐大，請考慮將其拆分為多個較小的類別。

<a name="facades-vs-dependency-injection"></a>
### Facades 與依賴注入的比較

依賴注入的主要優點之一是能夠替換被注入類別的實作。這在測試期間非常有用，因為您可以注入一個 Mock 或 Stub，並斷言 Stub 上呼叫了各種方法。

通常，不可能 Mock 或 Stub 一個真正的靜態類別方法。然而，由於 Facades 使用動態方法將方法呼叫代理到從 Service Container 解析的物件，我們實際上可以像測試注入的類別實例一樣測試 Facades。例如，給定以下路由：

    use Illuminate\Support\Facades\Cache;

    Route::get('/cache', function () {
        return Cache::get('key');
    });

使用 Laravel 的 Facade 測試方法，我們可以編寫以下測試來驗證 `Cache::get` 方法是否以我們預期的參數呼叫：

```php tab=Pest
use Illuminate\Support\Facades\Cache;

test('basic example', function () {
    Cache::shouldReceive('get')
        ->with('key')
        ->andReturn('value');

    $response = $this->get('/cache');

    $response->assertSee('value');
});
```

```php tab=PHPUnit
use Illuminate\Support\Facades\Cache;

/**
 * A basic functional test example.
 */
public function test_basic_example(): void
{
    Cache::shouldReceive('get')
        ->with('key')
        ->andReturn('value');

    $response = $this->get('/cache');

    $response->assertSee('value');
}
```

<a name="facades-vs-helper-functions"></a>
### Facades 與輔助函式的比較

除了 Facades 之外，Laravel 還包含各種「輔助」函式，可以執行常見任務，例如產生視圖、觸發事件、分派任務或傳送 HTTP 回應。許多這些輔助函式執行與對應 Facade 相同的功能。例如，以下 Facade 呼叫和輔助函式呼叫是等效的：

    return Illuminate\Support\Facades\View::make('profile');

    return view('profile');

Facades 和輔助函式之間沒有任何實際差異。使用輔助函式時，您仍然可以像測試對應的 Facade 一樣測試它們。例如，給定以下路由：

    Route::get('/cache', function () {
        return cache('key');
    });

`cache` 輔助函式將呼叫 `Cache` Facade 底層類別上的 `get` 方法。因此，即使我們使用輔助函式，我們也可以編寫以下測試來驗證該方法是否以我們預期的參數呼叫：

    use Illuminate\Support\Facades\Cache;

    /**
     * A basic functional test example.
     */
    public function test_basic_example(): void
    {
        Cache::shouldReceive('get')
            ->with('key')
            ->andReturn('value');

        $response = $this->get('/cache');

        $response->assertSee('value');
    }

<a name="how-facades-work"></a>
## Facades 的運作方式

在 Laravel 應用程式中，Facade 是一個提供從 Container 存取物件的類別。實現此功能的機制位於 `Facade` 類別中。Laravel 的 Facades 以及您建立的任何自訂 Facades 都將擴展基礎 `Illuminate\Support\Facades\Facade` 類別。

`Facade` 基礎類別利用 `__callStatic()` 魔術方法將您的 Facade 的呼叫延遲到從 Container 解析的物件。在下面的範例中，對 Laravel 快取系統進行了呼叫。透過查看此程式碼，人們可能會認為正在呼叫 `Cache` 類別上的靜態 `get` 方法：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Support\Facades\Cache;
    use Illuminate\View\View;

    class UserController extends Controller
    {
        /**
         * Show the profile for the given user.
         */
        public function showProfile(string $id): View
        {
            $user = Cache::get('user:'.$id);

            return view('profile', ['user' => $user]);
        }
    }

請注意，在檔案頂部附近我們正在「匯入」`Cache` Facade。此 Facade 作為存取 `Illuminate\Contracts\Cache\Factory` 介面底層實作的代理。我們使用 Facade 進行的任何呼叫都將傳遞給 Laravel 快取服務的底層實例。

如果我們查看 `Illuminate\Support\Facades\Cache` 類別，您會發現沒有靜態方法 `get`：

    class Cache extends Facade
    {
        /**
         * Get the registered name of the component.
         */
        protected static function getFacadeAccessor(): string
        {
            return 'cache';
        }
    }

相反，`Cache` Facade 擴展了基礎 `Facade` 類別並定義了方法 `getFacadeAccessor()`。此方法的職責是返回 Service Container 綁定的名稱。當使用者引用 `Cache` Facade 上的任何靜態方法時，Laravel 會從 [Service Container](/docs/{{version}}/container) 解析 `cache` 綁定，並對該物件執行請求的方法（在本例中為 `get`）。

<a name="real-time-facades"></a>
## 即時 Facades

使用即時 Facades，您可以將應用程式中的任何類別視為 Facade。為了說明如何使用此功能，讓我們先檢查一些不使用即時 Facades 的程式碼。例如，假設我們的 `Podcast` Model 有一個 `publish` 方法。然而，為了發布 Podcast，我們需要注入一個 `Publisher` 實例：

    <?php

    namespace App\Models;

    use App\Contracts\Publisher;
    use Illuminate\Database\Eloquent\Model;

    class Podcast extends Model
    {
        /**
         * Publish the podcast.
         */
        public function publish(Publisher $publisher): void
        {
            $this->update(['publishing' => now()]);

            $publisher->publish($this);
        }
    }

將 Publisher 實作注入到方法中，使我們能夠輕鬆地單獨測試該方法，因為我們可以 Mock 注入的 Publisher。然而，這要求我們每次呼叫 `publish` 方法時都必須傳遞一個 Publisher 實例。使用即時 Facades，我們可以保持相同的可測試性，同時無需明確傳遞 `Publisher` 實例。要產生即時 Facade，請在匯入類別的命名空間前加上 `Facades`：

    <?php

    namespace App\Models;

    use App\Contracts\Publisher; // [tl! remove]
    use Facades\App\Contracts\Publisher; // [tl! add]
    use Illuminate\Database\Eloquent\Model;

    class Podcast extends Model
    {
        /**
         * Publish the podcast.
         */
        public function publish(Publisher $publisher): void // [tl! remove]
        public function publish(): void // [tl! add]
        {
            $this->update(['publishing' => now()]);

            $publisher->publish($this); // [tl! remove]
            Publisher::publish($this); // [tl! add]
        }
    }

當使用即時 Facade 時，Publisher 實作將從 Service Container 中解析，使用介面或類別名稱中出現在 `Facades` 前綴之後的部分。在測試時，我們可以使用 Laravel 內建的 Facade 測試輔助函式來 Mock 此方法呼叫：

```php tab=Pest
<?php

use App\Models\Podcast;
use Facades\App\Contracts\Publisher;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

test('podcast can be published', function () {
    $podcast = Podcast::factory()->create();

    Publisher::shouldReceive('publish')->once()->with($podcast);

    $podcast->publish();
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Models\Podcast;
use Facades\App\Contracts\Publisher;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class PodcastTest extends TestCase
{
    use RefreshDatabase;

    /**
     * A test example.
     */
    public function test_podcast_can_be_published(): void
    {
        $podcast = Podcast::factory()->create();

        Publisher::shouldReceive('publish')->once()->with($podcast);

        $podcast->publish();
    }
}
```

<a name="facade-class-reference"></a>
## Facade 類別參考

您將在下方找到每個 Facade 及其底層類別。這是一個有用的工具，可以快速深入了解給定 Facade 根的 API 說明文件。Service Container 綁定鍵（如果適用）也包含在內。

<div class="overflow-auto">

| Facade | 類別 | Service Container 綁定 |
| --- | --- | --- |
| App | [Illuminate\Foundation\Application](https://laravel.com/api/{{version}}/Illuminate/Foundation/Application.html) | `app` |
| Artisan | [Illuminate\Contracts\Console\Kernel](https://laravel.com/api/{{version}}/Illuminate/Contracts/Console/Kernel.html) | `artisan` |
| Auth (實例) | [Illuminate\Contracts\Auth\Guard](https://laravel.com/api/{{version}}/Illuminate/Contracts/Auth/Guard.html) | `auth.driver` |
| Auth | [Illuminate\Auth\AuthManager](https://laravel.com/api/{{version}}/Illuminate/Auth/AuthManager.html) | `auth` |
| Blade | [Illuminate\View\Compilers\BladeCompiler](https://laravel.com/api/{{version}}/Illuminate/View/Compilers/BladeCompiler.html) | `blade.compiler` |
| Broadcast (實例) | [Illuminate\Contracts\Broadcasting\Broadcaster](https://laravel.com/api/{{version}}/Illuminate/Contracts/Broadcasting/Broadcaster.html) | &nbsp; |
| Broadcast | [Illuminate\Contracts\Broadcasting\Factory](https://laravel.com/api/{{version}}/Illuminate/Contracts/Broadcasting/Factory.html) | &nbsp; |
| Bus | [Illuminate\Contracts\Bus\Dispatcher](https://laravel.com/api/{{version}}/Illuminate/Contracts/Bus/Dispatcher.html) | &nbsp; |
| Cache (實例) | [Illuminate\Cache\Repository](https://laravel.com/api/{{version}}/Illuminate/Cache/Repository.html) | `cache.store` |
| Cache | [Illuminate\Cache\CacheManager](https://laravel.com/api/{{version}}/Illuminate/Cache/CacheManager.html) | `cache` |
| Config | [Illuminate\Config\Repository](https://laravel.com/api/{{version}}/Illuminate/Config/Repository.html) | `config` |
| Context | [Illuminate\Log\Context\Repository](https://laravel.com/api/{{version}}/Illuminate/Log/Context\Repository.html) | &nbsp; |
| Cookie | [Illuminate\Cookie\CookieJar](https://laravel.com/api/{{version}}/Illuminate/Cookie/CookieJar.html) | `cookie` |
| Crypt | [Illuminate\Encryption\Encrypter](https://laravel.com/api/{{version}}/Illuminate/Encryption/Encrypter.html) | `encrypter` |
| Date | [Illuminate\Support\DateFactory](https://laravel.com/api/{{version}}/Illuminate/Support/DateFactory.html) | `date` |
| DB (實例) | [Illuminate\Database\Connection](https://laravel.com/api/{{version}}/Illuminate/Database/Connection.html) | `db.connection` |
| DB | [Illuminate\Database\DatabaseManager](https://laravel.com/api/{{version}}/Illuminate/Database/DatabaseManager.html) | `db` |
| Event | [Illuminate\Events\Dispatcher](https://laravel.com/api/{{version}}/Illuminate/Events/Dispatcher.html) | `events` |
| Exceptions (實例) | [Illuminate\Contracts\Debug\ExceptionHandler](https://laravel.com/api/{{version}}/Illuminate/Contracts/Debug/ExceptionHandler.html) | &nbsp; |
| Exceptions | [Illuminate\Foundation\Exceptions\Handler](https://laravel.com/api/{{version}}/Illuminate/Foundation/Exceptions/Handler.html) | &nbsp; |
| File | [Illuminate\Filesystem\Filesystem](https://laravel.com/api/{{version}}/Illuminate/Filesystem/Filesystem.html) | `files` |
| Gate | [Illuminate\Contracts\Auth\Access\Gate](https://laravel.com/api/{{version}}/Illuminate/Contracts/Auth/Access/Gate.html) | &nbsp; |
| Hash | [Illuminate\Contracts\Hashing\Hasher](https://laravel.com/api/{{version}}/Illuminate/Contracts/Hashing/Hasher.html) | `hash` |
| Http | [Illuminate\Http\Client\Factory](https://laravel.com/api/{{version}}/Illuminate/Http/Client/Factory.html) | &nbsp; |
| Lang | [Illuminate\Translation\Translator](https://laravel.com/api/{{version}}/Illuminate/Translation/Translator.html) | `translator` |
| Log | [Illuminate\Log\LogManager](https://laravel.com/api/{{version}}/Illuminate/Log/LogManager.html) | `log` |
| Mail | [Illuminate\Mail\Mailer](https://laravel.com/api/{{version}}/Illuminate/Mail/Mailer.html) | `mailer` |
| Notification | [Illuminate\Notifications\ChannelManager](https://laravel.com/api/{{version}}/Illuminate/Notifications/ChannelManager.html) | &nbsp; |
| Password (實例) | [Illuminate\Auth\Passwords\PasswordBroker](https://laravel.com/api/{{version}}/Illuminate/Auth/Passwords/PasswordBroker.html) | `auth.password.broker` |
| Password | [Illuminate\Auth\Passwords\PasswordBrokerManager](https://laravel.com/api/{{version}}/Illuminate/Auth/Passwords/PasswordBrokerManager.html) | `auth.password` |
| Pipeline (實例) | [Illuminate\Pipeline\Pipeline](https://laravel.com/api/{{version}}/Illuminate/Pipeline/Pipeline.html) | &nbsp; |
| Process | [Illuminate\Process\Factory](https://laravel.com/api/{{version}}/Illuminate/Process/Factory.html) | &nbsp; |
| Queue (基礎類別) | [Illuminate\Queue\Queue](https://laravel.com/api/{{version}}/Illuminate/Queue/Queue.html) | &nbsp; |
| Queue (實例) | [Illuminate\Contracts\Queue\Queue](https://laravel.com/api/{{version}}/Illuminate/Contracts/Queue/Queue.html) | `queue.connection` |
| Queue | [Illuminate\Queue\QueueManager](https://laravel.com/api/{{version}}/Illuminate/Queue/QueueManager.html) | `queue` |
| RateLimiter | [Illuminate\Cache\RateLimiter](https://laravel.com/api/{{version}}/Illuminate/Cache/RateLimiter.html) | &nbsp; |
| Redirect | [Illuminate\Routing\Redirector](https://laravel.com/api/{{version}}/Illuminate/Routing/Redirector.html) | `redirect` |
| Redis (實例) | [Illuminate\Redis\Connections\Connection](https://laravel.com/api/{{version}}/Illuminate/Redis/Connections/Connection.html) | `redis.connection` |
| Redis | [Illuminate\Redis\RedisManager](https://laravel.com/api/{{version}}/Illuminate/Redis/RedisManager.html) | `redis` |
| Request | [Illuminate\Http\Request](https://laravel.com/api/{{version}}/Illuminate/Http/Request.html) | `request` |
| Response (實例) | [Illuminate\Http\Response](https://laravel.com/api/{{version}}/Illuminate/Http/Response.html) | &nbsp; |
| Response | [Illuminate\Contracts\Routing\ResponseFactory](https://laravel.com/api/{{version}}/Illuminate/Contracts/Routing/ResponseFactory.html) | &nbsp; |
| Route | [Illuminate\Routing\Router](https://laravel.com/api/{{version}}/Illuminate/Routing/Router.html) | `router` |
| Schedule | [Illuminate\Console\Scheduling\Schedule](https://laravel.com/api/{{version}}/Illuminate/Console/Scheduling/Schedule.html) | &nbsp; |
| Schema | [Illuminate\Database\Schema\Builder](https://laravel.com/api/{{version}}/Illuminate/Database/Schema/Builder.html) | &nbsp; |
| Session (實例) | [Illuminate\Session\Store](https://laravel.com/api/{{version}}/Illuminate/Session/Store.html) | `session.store` |
| Session | [Illuminate\Session\SessionManager](https://laravel.com/api/{{version}}/Illuminate/Session/SessionManager.html) | `session` |
| Storage (實例) | [Illuminate\Contracts\Filesystem\Filesystem](https://laravel.com/api/{{version}}/Illuminate/Contracts/Filesystem/Filesystem.html) | `filesystem.disk` |
| Storage | [Illuminate\Filesystem\FilesystemManager](https://laravel.com/api/{{version}}/Illuminate/Filesystem/FilesystemManager.html) | `filesystem` |
| URL | [Illuminate\Routing\UrlGenerator](https://laravel.com/api/{{version}}/Illuminate/Routing/UrlGenerator.html) | `url` |
| Validator (實例) | [Illuminate\Validation\Validator](https://laravel.com/api/{{version}}/Illuminate/Validation/Validator.html) | &nbsp; |
| Validator | [Illuminate\Validation\Factory](https://laravel.com/api/{{version}}/Illuminate/Validation/Factory.html) | `validator` |
| View (實例) | [Illuminate\View\View](https://laravel.com/api/{{version}}/Illuminate/View/View.html) | &nbsp; |
| View | [Illuminate\View\Factory](https://laravel.com/api/{{version}}/Illuminate/View/Factory.html) | `view` |
| Vite | [Illuminate\Foundation\Vite](https://laravel.com/api/{{version}}/Illuminate/Foundation/Vite.html) | &nbsp; |

</div>
