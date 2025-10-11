# Facades

- [介紹](#introduction)
- [何時使用 Facades](#when-to-use-facades)
    - [Facades vs. 依賴注入](#facades-vs-dependency-injection)
    - [Facades vs. 輔助函式](#facades-vs-helper-functions)
- [Facades 的運作方式](#how-facades-work)
- [即時 Facades](#real-time-facades)
- [Facade 類別參考](#facade-class-reference)

<a name="introduction"></a>
## 介紹

在整個 Laravel 文件中，您將看到透過「facades」與 Laravel 功能互動的程式碼範例。Facades 為應用程式[服務容器](/docs/{{version}}/container)中可用的類別提供了「靜態」介面。Laravel 內建了許多 facades，這些 facades 提供了對幾乎所有 Laravel 功能的存取。

Laravel 的 facades 作為服務容器中底層類別的「靜態代理」，提供了簡潔、富有表現力的語法優勢，同時比傳統的靜態方法具有更高的可測試性和靈活性。如果您不完全理解 facades 的運作方式，這也沒關係 —— 順其自然，繼續學習 Laravel 即可。

所有 Laravel 的 facades 都定義在 `Illuminate\Support\Facades` 命名空間中。因此，我們可以輕鬆地像這樣存取 facade：

    use Illuminate\Support\Facades\Cache;
    use Illuminate\Support\Facades\Route;

    Route::get('/cache', function () {
        return Cache::get('key');
    });

在整個 Laravel 文件中，許多範例都將使用 facades 來展示框架的各種功能。


<a name="helper-functions"></a>
#### 輔助函式

為了輔助 facades，Laravel 提供了多種全域「輔助函式」，讓與常見 Laravel 功能互動變得更加容易。您可能會遇到的一些常見輔助函式包括 `view`、`response`、`url`、`config` 等。Laravel 提供的每個輔助函式都隨其對應的功能一同記錄；不過，完整列表可在專用的[輔助函式文件](/docs/{{version}}/helpers)中找到。

例如，我們可以使用 `response` 函式，而不是使用 `Illuminate\Support\Facades\Response` facade 來產生 JSON 回應。因為輔助函式是全域可用的，所以您無需匯入任何類別即可使用它們：

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

facades 有許多優點。它們提供了簡潔、易記的語法，讓您無需記住必須手動注入或設定的冗長類別名稱，即可使用 Laravel 的功能。此外，由於它們獨特的 PHP 動態方法用法，facades 易於測試。

然而，在使用 facades 時必須小心。facades 的主要危險是類別「職責膨脹」。由於 facades 易於使用且不需要注入，因此您的類別可能會不斷增長並在單一類別中使用許多 facades。使用依賴注入可以透過大型建構函式提供的視覺回饋來緩解這種潛在問題，提示您的類別變得過於龐大。因此，在使用 facades 時，請特別注意類別的大小，以確保其職責範圍保持狹窄。如果您的類別變得過大，請考慮將其拆分為多個較小的類別。


<a name="facades-vs-dependency-injection"></a>
### Facades vs. 依賴注入

依賴注入的主要優點之一是能夠替換被注入類別的實作。這在測試期間非常有用，因為您可以注入模擬 (mock) 或替代 (stub) 物件，並斷言各種方法已在替代物件上被呼叫。

通常，要模擬或替代真正的靜態類別方法是不可能的。然而，由於 facades 使用動態方法將方法呼叫代理到從服務容器解析出的物件，因此我們實際上可以像測試被注入的類別實例一樣測試 facades。例如，考慮以下路由：

    use Illuminate\Support\Facades\Cache;

    Route::get('/cache', function () {
        return Cache::get('key');
    });

使用 Laravel 的 facade 測試方法，我們可以編寫以下測試來驗證 `Cache::get` 方法是否以我們預期的引數被呼叫：

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
### Facades vs. 輔助函式

除了 facades 之外，Laravel 還包含各種「輔助函式」，這些函式可以執行常見任務，例如產生視圖、觸發事件、分派任務或傳送 HTTP 回應。許多這些輔助函式執行與其對應 facade 相同的功能。例如，這個 facade 呼叫和輔助函式呼叫是等效的：

    return Illuminate\Support\Facades\View::make('profile');

    return view('profile');

facades 和輔助函式之間沒有絕對的實際差異。當使用輔助函式時，您仍然可以像測試其對應的 facade 一樣測試它們。例如，考慮以下路由：

    Route::get('/cache', function () {
        return cache('key');
    });

`cache` 輔助函式將呼叫 `Cache` facade 底層類別上的 `get` 方法。因此，即使我們使用輔助函式，我們也可以編寫以下測試來驗證該方法是否以我們預期的引數被呼叫：

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

在 Laravel 應用程式中，Facade 是一個類別，它提供從容器中存取物件的方法。其背後的運作機制在 `Facade` 類別中。Laravel 的 Facades，以及任何您建立的自訂 Facades，都會擴展基礎的 `Illuminate\Support\Facades\Facade` 類別。

`Facade` 基礎類別會利用 `__callStatic()` 魔術方法，將您 Facade 的呼叫委派給從容器中解析的物件。在下面的範例中，對 Laravel 的快取系統發出了一個呼叫。乍看這段程式碼，可能會認為靜態的 `get` 方法是在 `Cache` 類別上被呼叫：

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

注意，在檔案的頂部我們「引入」了 `Cache` Facade。這個 Facade 作為一個代理，用於存取 `Illuminate\Contracts\Cache\Factory` 介面的底層實作。任何我們使用此 Facade 進行的呼叫，都將傳遞給 Laravel 快取服務的底層實例。

如果我們查看那個 `Illuminate\Support\Facades\Cache` 類別，您會發現裡面並沒有靜態方法 `get`：

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

相反地，`Cache` Facade 擴展了基礎的 `Facade` 類別，並定義了 `getFacadeAccessor()` 方法。這個方法的職責是回傳一個服務容器綁定的名稱。當使用者在 `Cache` Facade 上參照任何靜態方法時，Laravel 會從 [服務容器](/docs/{{version}}/container) 中解析 `cache` 綁定，並對該物件執行所請求的方法 (在本例中為 `get`)。

<a name="real-time-facades"></a>
## 即時 Facades

使用即時 Facades，您可以將應用程式中的任何類別視為 Facade。為了說明其用法，讓我們先看看一段不使用即時 Facades 的程式碼。例如，假設我們的 `Podcast` Model 有一個 `publish` 方法。然而，為了發布 podcast，我們需要注入一個 `Publisher` 實例：

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

將 publisher 實作注入到方法中，使我們能夠輕鬆地獨立測試該方法，因為我們可以模擬注入的 publisher。然而，這要求我們每次呼叫 `publish` 方法時都必須傳遞一個 publisher 實例。使用即時 Facades，我們可以保持相同的可測試性，同時無需明確傳遞 `Publisher` 實例。要生成即時 Facade，請在引入類別的命名空間前加上 `Facades`：

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

當使用即時 Facade 時，publisher 實作將根據 `Facades` 前綴之後的介面或類別名稱部分，從服務容器中解析出來。在測試時，我們可以使用 Laravel 內建的 Facade 測試輔助工具來模擬此方法呼叫：

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

下方列出了每個 Facade 及其底層類別。這是一個實用的工具，能幫助您快速深入查閱特定 Facade 根的 API 文件。在適用情況下，也包含了 [服務容器繫結](/docs/{{version}}/container) 鍵。

<div class="overflow-auto">

| Facade | 類別 | 服務容器繫結 |
| --- | --- | --- |
| App | [Illuminate\Foundation\Application](https://laravel.com/api/{{version}}/Illuminate/Foundation/Application.html) | `app` |
| Artisan | [Illuminate\Contracts\Console\Kernel](https://laravel.com/api/{{version}}/Illuminate/Contracts/Console/Kernel.html) | `artisan` |
| Auth (Instance) | [Illuminate\Contracts\Auth\Guard](https://laravel.com/api/{{version}}/Illuminate/Contracts/Auth/Guard.html) | `auth.driver` |
| Auth | [Illuminate\Auth\AuthManager](https://laravel.com/api/{{version}}/Illuminate/Auth/AuthManager.html) | `auth` |
| Blade | [Illuminate\View\Compilers\BladeCompiler](https://laravel.com/api/{{version}}/Illuminate/View/Compilers/BladeCompiler.html) | `blade.compiler` |
| Broadcast (Instance) | [Illuminate\Contracts\Broadcasting\Broadcaster](https://laravel.com/api/{{version}}/Illuminate/Contracts/Broadcasting/Broadcaster.html) | &nbsp; |
| Broadcast | [Illuminate\Contracts\Broadcasting\Factory](https://laravel.com/api/{{version}}/Illuminate/Contracts/Broadcasting/Factory.html) | &nbsp; |
| Bus | [Illuminate\Contracts\Bus\Dispatcher](https://laravel.com/api/{{version}}/Illuminate\Contracts\Bus\Dispatcher.html) | &nbsp; |
| Cache (Instance) | [Illuminate\Cache\Repository](https://laravel.com/api/{{version}}/Illuminate/Cache/Repository.html) | `cache.store` |
| Cache | [Illuminate\Cache\CacheManager](https://laravel.com/api/{{version}}/Illuminate/Cache/CacheManager.html) | `cache` |
| Config | [Illuminate\Config\Repository](https://laravel.com/api/{{version}}/Illuminate/Config/Repository.html) | `config` |
| Context | [Illuminate\Log\Context\Repository](https://laravel.com/api/{{version}}/Illuminate/Log/Context/Repository.html) | &nbsp; |
| Cookie | [Illuminate\Cookie\CookieJar](https://laravel.com/api/{{version}}/Illuminate/Cookie/CookieJar.html) | `cookie` |
| Crypt | [Illuminate\Encryption\Encrypter](https://laravel.com/api/{{version}}/Illuminate/Encryption/Encrypter.html) | `encrypter` |
| Date | [Illuminate\Support\DateFactory](https://laravel.com/api/{{version}}/Illuminate/Support/DateFactory.html) | `date` |
| DB (Instance) | [Illuminate\Database\Connection](https://laravel.com/api/{{version}}/Illuminate/Database/Connection.html) | `db.connection` |
| DB | [Illuminate\Database\DatabaseManager](https://laravel.com/api/{{version}}/Illuminate/Database/DatabaseManager.html) | `db` |
| Event | [Illuminate\Events\Dispatcher](https://laravel.com/api/{{version}}/Illuminate/Events/Dispatcher.html) | `events` |
| Exceptions (Instance) | [Illuminate\Contracts\Debug\ExceptionHandler](https://laravel.com/api/{{version}}/Illuminate/Contracts/Debug/ExceptionHandler.html) | &nbsp; |
| Exceptions | [Illuminate\Foundation\Exceptions\Handler](https://laravel.com/api/{{version}}/Illuminate/Foundation/Exceptions/Handler.html) | &nbsp; |
| File | [Illuminate\Filesystem\Filesystem](https://laravel.com/api/{{version}}/Illuminate/Filesystem/Filesystem.html) | `files` |
| Gate | [Illuminate\Contracts\Auth\Access\Gate](https://laravel.com/api/{{version}}/Illuminate/Contracts/Auth/Access/Gate.html) | &nbsp; |
| Hash | [Illuminate\Contracts\Hashing\Hasher](https://laravel.com/api/{{version}}/Illuminate/Contracts/Hashing/Hasher.html) | `hash` |
| Http | [Illuminate\Http\Client\Factory](https://laravel.com/api/{{version}}/Illuminate/Http/Client/Factory.html) | &nbsp; |
| Lang | [Illuminate\Translation\Translator](https://laravel.com/api/{{version}}/Illuminate/Translation/Translator.html) | `translator` |
| Log | [Illuminate\Log\LogManager](https://laravel.com/api/{{version}}/Illuminate/Log/LogManager.html) | `log` |
| Mail | [Illuminate\Mail\Mailer](https://laravel.com/api/{{version}}/Illuminate/Mail/Mailer.html) | `mailer` |
| Notification | [Illuminate\Notifications\ChannelManager](https://laravel.com/api/{{version}}/Illuminate/Notifications/ChannelManager.html) | &nbsp; |
| Password (Instance) | [Illuminate\Auth\Passwords\PasswordBroker](https://laravel.com/api/{{version}}/Illuminate/Auth/Passwords/PasswordBroker.html) | `auth.password.broker` |
| Password | [Illuminate\Auth\Passwords\PasswordBrokerManager](https://laravel.com/api/{{version}}/Illuminate/Auth/Passwords/PasswordBrokerManager.html) | `auth.password` |
| Pipeline (Instance) | [Illuminate\Pipeline\Pipeline](https://laravel.com/api/{{version}}/Illuminate/Pipeline/Pipeline.html) | &nbsp; |
| Process | [Illuminate\Process\Factory](https://laravel.com/api/{{version}}/Illuminate/Process/Factory.html) | &nbsp; |
| Queue (Base Class) | [Illuminate\Queue\Queue](https://laravel.com/api/{{version}}/Illuminate/Queue/Queue.html) | &nbsp; |
| Queue (Instance) | [Illuminate\Contracts\Queue\Queue](https://laravel.com/api/{{version}}/Illuminate/Contracts/Queue/Queue.html) | `queue.connection` |
| Queue | [Illuminate\Queue\QueueManager](https://laravel.com/api/{{version}}/Illuminate/Queue/QueueManager.html) | `queue` |
| RateLimiter | [Illuminate\Cache\RateLimiter](https://laravel.com/api/{{version}}/Illuminate/Cache/RateLimiter.html) | &nbsp; |
| Redirect | [Illuminate\Routing\Redirector](https://laravel.com/api/{{version}}/Illuminate/Routing/Redirector.html) | `redirect` |
| Redis (Instance) | [Illuminate\Redis\Connections\Connection](https://laravel.com/api/{{version}}/Illuminate/Redis/Connections/Connection.html) | `redis.connection` |
| Redis | [Illuminate\Redis\RedisManager](https://laravel.com/api/{{version}}/Illuminate/Redis/RedisManager.html) | `redis` |
| Request | [Illuminate\Http\Request](https://laravel.com/api/{{version}}/Illuminate/Http/Request.html) | `request` |
| Response (Instance) | [Illuminate\Http\Response](https://laravel.com/api/{{version}}/Illuminate/Http/Response.html) | &nbsp; |
| Response | [Illuminate\Contracts\Routing\ResponseFactory](https://laravel.com/api/{{version}}/Illuminate/Contracts/Routing/ResponseFactory.html) | &nbsp; |
| Route | [Illuminate\Routing\Router](https://laravel.com/api/{{version}}/Illuminate/Routing/Router.html) | `router` |
| Schedule | [Illuminate\Console\Scheduling\Schedule](https://laravel.com/api/{{version}}/Illuminate/Console/Scheduling/Schedule.html) | &nbsp; |
| Schema | [Illuminate\Database\Schema\Builder](https://laravel.com/api/{{version}}/Illuminate/Database/Schema/Builder.html) | &nbsp; |
| Session (Instance) | [Illuminate\Session\Store](https://laravel.com/api/{{version}}/Illuminate/Session/Store.html) | `session.store` |
| Session | [Illuminate\Session\SessionManager](https://laravel.com/api/{{version}}/Illuminate/Session/SessionManager.html) | `session` |
| Storage (Instance) | [Illuminate\Contracts\Filesystem\Filesystem](https://laravel.com/api/{{version}}/Illuminate/Contracts/Filesystem/Filesystem.html) | `filesystem.disk` |
| Storage | [Illuminate\Filesystem\FilesystemManager](https://laravel.com/api/{{version}}/Illuminate/Filesystem/FilesystemManager.html) | `filesystem` |
| URL | [Illuminate\Routing\UrlGenerator](https://laravel.com/api/{{version}}/Illuminate/Routing/UrlGenerator.html) | `url` |
| Validator (Instance) | [Illuminate\Validation\Validator](https://laravel.com/api/{{version}}/Illuminate/Validation/Validator.html) | &nbsp; |
| Validator | [Illuminate\Validation\Factory](https://laravel.com/api/{{version}}/Illuminate/Validation/Factory.html) | `validator` |
| View (Instance) | [Illuminate\View\View](https://laravel.com/api/{{version}}/Illuminate/View/View.html) | &nbsp; |
| View | [Illuminate\View\Factory](https://laravel.com/api/{{version}}/Illuminate/View/Factory.html) | `view` |
| Vite | [Illuminate\Foundation\Vite](https://laravel.com/api/{{version}}/Illuminate/Foundation/Vite.html) | &nbsp; |

</div>