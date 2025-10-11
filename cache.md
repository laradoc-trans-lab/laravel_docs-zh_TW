# 快取

- [簡介](#introduction)
- [設定](#configuration)
    - [驅動器先決條件](#driver-prerequisites)
- [快取使用](#cache-usage)
    - [取得快取實例](#obtaining-a-cache-instance)
    - [從快取中取得項目](#retrieving-items-from-the-cache)
    - [在快取中儲存項目](#storing-items-in-the-cache)
    - [從快取中移除項目](#removing-items-from-the-cache)
    - [快取輔助函式](#the-cache-helper)
- [原子鎖定](#atomic-locks)
    - [管理鎖定](#managing-locks)
    - [跨程序管理鎖定](#managing-locks-across-processes)
- [新增自訂快取驅動器](#adding-custom-cache-drivers)
    - [撰寫驅動器](#writing-the-driver)
    - [註冊驅動器](#registering-the-driver)
- [事件](#events)

<a name="introduction"></a>
## 簡介

您的應用程式執行的一些資料擷取或處理任務可能會佔用大量 CPU 資源或需要幾秒鐘才能完成。在這種情況下，通常會將擷取的資料快取一段時間，以便在後續對相同資料的請求中快速擷取。快取的資料通常儲存在非常快速的資料儲存區中，例如 [Memcached](https://memcached.org) 或 [Redis](https://redis.io)。

幸運的是，Laravel 為各種快取後端提供了富有表達力且統一的 API，讓您可以利用其極速的資料擷取來加速您的 Web 應用程式。

<a name="configuration"></a>
## 設定

您的應用程式的快取設定檔位於 `config/cache.php`。在此檔案中，您可以指定要在整個應用程式中預設使用的快取儲存區。Laravel 原生支援流行的快取後端，例如 [Memcached](https://memcached.org)、[Redis](https://redis.io)、[DynamoDB](https://aws.amazon.com/dynamodb) 和關聯式資料庫。此外，還提供了基於檔案的快取驅動器，而 `array` 和「null」快取驅動器則為您的自動化測試提供了方便的快取後端。

快取設定檔還包含您可以檢閱的各種其他選項。預設情況下，Laravel 配置為使用 `database` 快取驅動器，它將序列化的快取物件儲存在您的應用程式的資料庫中。

<a name="driver-prerequisites"></a>
### 驅動器先決條件

<a name="prerequisites-database"></a>
#### 資料庫

使用 `database` 快取驅動器時，您需要一個資料庫表來儲存快取資料。通常，這包含在 Laravel 的預設 `0001_01_01_000001_create_cache_table.php` [資料庫遷移](/docs/{{version}}/migrations)中；但是，如果您的應用程式不包含此遷移，您可以使用 `make:cache-table` Artisan 命令來建立它：

```shell
php artisan make:cache-table

php artisan migrate
```

<a name="memcached"></a>
#### Memcached

使用 Memcached 驅動器需要安裝 [Memcached PECL 套件](https://pecl.php.net/package/memcached)。您可以在 `config/cache.php` 設定檔中列出所有 Memcached 伺服器。此檔案已包含 `memcached.servers` 項目以供您開始使用：

    'memcached' => [
        // ...

        'servers' => [
            [
                'host' => env('MEMCACHED_HOST', '127.0.0.1'),
                'port' => env('MEMCACHED_PORT', 11211),
                'weight' => 100,
            ],
        ],
    ],

如有需要，您可以將 `host` 選項設定為 UNIX socket 路徑。如果您這樣做，`port` 選項應設定為 `0`：

    'memcached' => [
        // ...

        'servers' => [
            [
                'host' => '/var/run/memcached/memcached.sock',
                'port' => 0,
                'weight' => 100
            ],
        ],
    ],

<a name="redis"></a>
#### Redis

在使用 Laravel 的 Redis 快取之前，您需要透過 PECL 安裝 PhpRedis PHP 擴充功能，或透過 Composer 安裝 `predis/predis` 套件 (~2.0)。[Laravel Sail](/docs/{{version}}/sail) 已包含此擴充功能。此外，官方 Laravel 部署平台，例如 [Laravel Forge](https://forge.laravel.com) 和 [Laravel Vapor](https://vapor.laravel.com)，預設都安裝了 PhpRedis 擴充功能。

有關配置 Redis 的更多資訊，請查閱其 [Laravel 文件頁面](/docs/{{version}}/redis#configuration)。

<a name="dynamodb"></a>
#### DynamoDB

在使用 [DynamoDB](https://aws.amazon.com/dynamodb) 快取驅動器之前，您必須建立一個 DynamoDB 表格來儲存所有快取資料。通常，此表格應命名為 `cache`。但是，您應該根據 `cache` 設定檔中 `stores.dynamodb.table` 設定值來命名表格。表格名稱也可以透過 `DYNAMODB_CACHE_TABLE` 環境變數設定。

此表格還應具有一個字串分區鍵，其名稱應與您的應用程式 `cache` 設定檔中 `stores.dynamodb.attributes.key` 設定項目的值相對應。預設情況下，分區鍵應命名為 `key`。

通常，DynamoDB 不會主動從表格中移除過期項目。因此，您應該在表格上 [啟用存留時間 (TTL)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)。在配置表格的 TTL 設定時，您應將 TTL 屬性名稱設定為 `expires_at`。

接下來，安裝 AWS SDK，以便您的 Laravel 應用程式可以與 DynamoDB 進行通訊：

```shell
composer require aws/aws-sdk-php
```

此外，您應該確保為 DynamoDB 快取儲存區設定選項提供了值。通常這些選項，例如 `AWS_ACCESS_KEY_ID` 和 `AWS_SECRET_ACCESS_KEY`，應在您的應用程式的 `.env` 設定檔中定義：

```php
'dynamodb' => [
    'driver' => 'dynamodb',
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    'table' => env('DYNAMODB_CACHE_TABLE', 'cache'),
    'endpoint' => env('DYNAMODB_ENDPOINT'),
],
```

<a name="mongodb"></a>
#### MongoDB

如果您正在使用 MongoDB，則官方的 `mongodb/laravel-mongodb` 套件提供了 `mongodb` 快取驅動器，並且可以使用 `mongodb` 資料庫連線進行配置。MongoDB 支援 TTL 索引，可用於自動清除過期的快取項目。

有關配置 MongoDB 的更多資訊，請參閱 MongoDB [快取與鎖定文件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/cache/)。

<a name="cache-usage"></a>
## 快取使用

<a name="obtaining-a-cache-instance"></a>
### 取得快取實例

若要取得快取儲存區實例，您可以使用 `Cache` facade，這也是我們將在整個文件中使用的。`Cache` facade 提供便利、簡潔的存取方式，可存取 Laravel 快取合約的底層實作：

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Support\Facades\Cache;

    class UserController extends Controller
    {
        /**
         * Show a list of all users of the application.
         */
        public function index(): array
        {
            $value = Cache::get('key');

            return [
                // ...
            ];
        }
    }

<a name="accessing-multiple-cache-stores"></a>
#### 存取多個快取儲存區

使用 `Cache` facade，您可以透過 `store` 方法存取各種快取儲存區。傳遞給 `store` 方法的鍵 (key) 應該與 `cache` 設定檔中 `stores` 設定陣列裡列出的其中一個儲存區相對應：

    $value = Cache::store('file')->get('foo');

    Cache::store('redis')->put('bar', 'baz', 600); // 10 Minutes

<a name="retrieving-items-from-the-cache"></a>
### 從快取中取得項目

`Cache` facade 的 `get` 方法用於從快取中取得項目。如果項目在快取中不存在，將會回傳 `null`。如果您希望，可以傳遞第二個參數給 `get` 方法，指定當項目不存在時您希望回傳的預設值：

    $value = Cache::get('key');

    $value = Cache::get('key', 'default');

您甚至可以傳遞一個閉包 (closure) 作為預設值。如果指定的項目在快取中不存在，將會回傳該閉包的結果。傳遞閉包讓您可以延遲從資料庫或其他外部服務中取得預設值：

    $value = Cache::get('key', function () {
        return DB::table(/* ... */)->get();
    });

<a name="determining-item-existence"></a>
#### 判斷項目是否存在

`has` 方法可用於判斷項目是否存在於快取中。如果項目存在但其值為 `null`，此方法也將回傳 `false`：

    if (Cache::has('key')) {
        // ...
    }

<a name="incrementing-decrementing-values"></a>
#### 遞增 / 遞減值

`increment` 和 `decrement` 方法可用於調整快取中整數項目的值。這兩個方法都接受一個可選的第二個參數，指示要遞增或遞減項目值的數量：

    // Initialize the value if it does not exist...
    Cache::add('key', 0, now()->addHours(4));

    // Increment or decrement the value...
    Cache::increment('key');
    Cache::increment('key', $amount);
    Cache::decrement('key');
    Cache::decrement('key', $amount);

<a name="retrieve-store"></a>
#### 取得並儲存

有時您可能希望從快取中取得一個項目，但如果請求的項目不存在，也同時儲存一個預設值。例如，您可能希望從快取中取得所有使用者，或者如果他們不存在，則從資料庫中取得他們並將其添加到快取中。您可以使用 `Cache::remember` 方法來完成此操作：

    $value = Cache::remember('users', $seconds, function () {
        return DB::table('users')->get();
    });

如果項目在快取中不存在，傳遞給 `remember` 方法的閉包將會被執行，並且其結果將被放入快取中。

您可以使用 `rememberForever` 方法從快取中取得項目，或者如果它不存在則永久儲存它：

    $value = Cache::rememberForever('users', function () {
        return DB::table('users')->get();
    });

<a name="swr"></a>
#### 舊資料重新驗證 (Stale While Revalidate)

當使用 `Cache::remember` 方法時，如果快取值已過期，一些使用者可能會遇到回應時間變慢的情況。對於某些類型的資料，在背景重新計算快取值的同時，允許提供部分舊資料會很有用，這可以防止某些使用者在計算快取值時遇到回應時間變慢。這通常被稱為「舊資料重新驗證 (stale-while-revalidate)」模式，而 `Cache::flexible` 方法提供了此模式的實作。

flexible 方法接受一個陣列，該陣列指定快取值被視為「新鮮 (fresh)」的時間長度以及何時變為「舊資料 (stale)」。陣列中的第一個值代表快取被視為新鮮的秒數，而第二個值定義了在重新計算之前可以作為舊資料提供的時間長度。

如果請求是在新鮮期間（第一個值之前）提出，快取將立即回傳而無需重新計算。如果請求是在舊資料期間（兩個值之間）提出，舊資料將提供給使用者，並且會註冊一個 [延遲函式](/docs/{{version}}/helpers#deferred-functions) 以在回應傳送給使用者後重新整理快取值。如果請求是在第二個值之後提出，則快取被視為已過期，並且會立即重新計算值，這可能會導致使用者回應速度變慢：

    $value = Cache::flexible('users', [5, 10], function () {
        return DB::table('users')->get();
    });

<a name="retrieve-delete"></a>
#### 取得並刪除

如果您需要從快取中取得項目，然後刪除該項目，您可以使用 `pull` 方法。與 `get` 方法一樣，如果項目在快取中不存在，將會回傳 `null`：

    $value = Cache::pull('key');

    $value = Cache::pull('key', 'default');

<a name="storing-items-in-the-cache"></a>
### 在快取中儲存項目

您可以使用 `Cache` facade 上的 `put` 方法將項目儲存到快取中：

    Cache::put('key', 'value', $seconds = 10);

如果沒有將儲存時間傳遞給 `put` 方法，該項目將會被無限期儲存：

    Cache::put('key', 'value');

除了將秒數作為整數傳遞之外，您也可以傳遞一個 `DateTime` 實例來表示快取項目的預期過期時間：

    Cache::put('key', 'value', now()->addMinutes(10));

<a name="store-if-not-present"></a>
#### 若不存在則儲存

`add` 方法只會在項目不存在於快取儲存區時才將其添加到快取中。如果項目確實已添加到快取中，該方法將回傳 `true`。否則，該方法將回傳 `false`。`add` 方法是一個原子操作：

    Cache::add('key', 'value', $seconds);

<a name="storing-items-forever"></a>
#### 永久儲存項目

`forever` 方法可用於將項目永久儲存到快取中。由於這些項目不會過期，因此必須使用 `forget` 方法手動將它們從快取中移除：

    Cache::forever('key', 'value');

> [!NOTE]  
> 如果您使用 Memcached 驅動器，當快取達到其大小限制時，儲存為 forever 的項目可能會被移除。

<a name="removing-items-from-the-cache"></a>
### 從快取中移除項目

您可以使用 `forget` 方法從快取中移除項目：

    Cache::forget('key');

您也可以透過提供零或負數的過期秒數來移除項目：

    Cache::put('key', 'value', 0);

    Cache::put('key', 'value', -5);

您可以使用 `flush` 方法清除整個快取：

    Cache::flush();

> [!WARNING]  
> 清除快取不會遵循您配置的快取“前綴” ，並且會從快取中移除所有項目。當清除由其他應用程式共用的快取時，請仔細考慮這一點。

<a name="the-cache-helper"></a>
### 快取輔助函式

除了使用 `Cache` Facade 之外，您也可以使用全域 `cache` 函式來透過快取取得並儲存資料。當 `cache` 函式被單一字串引數呼叫時，它將回傳指定鍵的值：

    $value = cache('key');

如果您提供一個鍵值對的陣列以及一個過期時間給此函式，它會將值在快取中儲存指定的持續時間：

    cache(['key' => 'value'], $seconds);

    cache(['key' => 'value'], now()->addMinutes(10));

當 `cache` 函式被不帶任何引數呼叫時，它會回傳 `Illuminate\Contracts\Cache\Factory` 實作的一個實例，讓您可以呼叫其他的快取方法：

    cache()->remember('users', $seconds, function () {
        return DB::table('users')->get();
    });

> [!NOTE]  
> 當測試全域 `cache` 函式的呼叫時，您可以使用 `Cache::shouldReceive` 方法，就像您在[測試 Facade](/docs/{{version}}/mocking#mocking-facades) 一樣。

<a name="atomic-locks"></a>
## 原子鎖定

> [!WARNING]  
> 若要使用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動器作為應用程式的預設快取驅動器。此外，所有伺服器都必須與同一個中央快取伺服器通訊。

<a name="managing-locks"></a>
### 管理鎖定

原子鎖定允許操作分散式鎖定，而無需擔心競爭條件。例如，Laravel Forge 使用原子鎖定來確保在伺服器上一次只執行一個遠端任務。您可以使用 `Cache::lock` 方法來建立和管理鎖定：

    use Illuminate\Support\Facades\Cache;

    $lock = Cache::lock('foo', 10);

    if ($lock->get()) {
        // Lock acquired for 10 seconds...

        $lock->release();
    }

`get` 方法也接受一個閉包。在閉包執行後，Laravel 會自動釋放鎖定：

    Cache::lock('foo', 10)->get(function () {
        // Lock acquired for 10 seconds and automatically released...
    });

如果您請求時鎖定不可用，您可以指示 Laravel 等待指定的秒數。如果在指定時間限制內無法取得鎖定，將會拋出 `Illuminate\Contracts\Cache\LockTimeoutException`：

    use Illuminate\Contracts\Cache\LockTimeoutException;

    $lock = Cache::lock('foo', 10);

    try {
        $lock->block(5);

        // Lock acquired after waiting a maximum of 5 seconds...
    } catch (LockTimeoutException $e) {
        // Unable to acquire lock...
    } finally {
        $lock->release();
    }

上述範例可以透過將閉包傳遞給 `block` 方法來簡化。當閉包傳遞給此方法時，Laravel 將嘗試在指定秒數內取得鎖定，並在閉包執行後自動釋放鎖定：

    Cache::lock('foo', 10)->block(5, function () {
        // Lock acquired after waiting a maximum of 5 seconds...
    });

<a name="managing-locks-across-processes"></a>
### 跨程序管理鎖定

有時，您可能希望在一個程序中取得鎖定，然後在另一個程序中釋放它。例如，您可以在網頁請求期間取得鎖定，並希望在該請求觸發的佇列任務結束時釋放鎖定。在此情境下，您應該將鎖定的作用域「擁有者憑證」傳遞給佇列任務，以便任務可以使用給定的憑證重新實例化鎖定。

在下面的範例中，如果成功取得鎖定，我們將分派一個佇列任務。此外，我們將透過鎖定的 `owner` 方法將鎖定的擁有者憑證傳遞給佇列任務：

    $podcast = Podcast::find($id);

    $lock = Cache::lock('processing', 120);

    if ($lock->get()) {
        ProcessPodcast::dispatch($podcast, $lock->owner());
    }

在我們應用程式的 `ProcessPodcast` 任務中，我們可以利用擁有者憑證來還原並釋放鎖定：

    Cache::restoreLock('processing', $this->owner)->release();

如果您希望在不考慮目前擁有者的情況下釋放鎖定，可以使用 `forceRelease` 方法：

    Cache::lock('processing')->forceRelease();

<a name="adding-custom-cache-drivers"></a>
## 新增自訂快取驅動器

<a name="writing-the-driver"></a>
### 撰寫驅動器

為了建立我們的自訂快取驅動器，我們首先需要實作 `Illuminate\Contracts\Cache\Store` [合約](/docs/{{version}}/contracts)。因此，MongoDB 快取實作可能看起來像這樣：

    <?php

    namespace App\Extensions;

    use Illuminate\Contracts\Cache\Store;

    class MongoStore implements Store
    {
        public function get($key) {}
        public function many(array $keys) {}
        public function put($key, $value, $seconds) {}
        public function putMany(array $values, $seconds) {}
        public function increment($key, $value = 1) {}
        public function decrement($key, $value = 1) {}
        public function forever($key, $value) {}
        public function forget($key) {}
        public function flush() {}
        public function getPrefix() {}
    }

我們只需要使用 MongoDB 連線來實作這些方法。有關如何實作這些方法的範例，請查看 Laravel 框架原始碼中的 `Illuminate\Cache\MemcachedStore`。一旦我們的實作完成，我們就可以透過呼叫 `Cache` facade 的 `extend` 方法來完成自訂驅動器的註冊：

    Cache::extend('mongo', function (Application $app) {
        return Cache::repository(new MongoStore);
    });

> [!NOTE]  
> 如果您想知道在哪裡放置自訂快取驅動器程式碼，您可以在 `app` 目錄中建立一個 `Extensions` 命名空間。然而，請記住，Laravel 沒有僵化的應用程式結構，您可以根據自己的偏好自由組織應用程式。

<a name="registering-the-driver"></a>
### 註冊驅動器

為了向 Laravel 註冊自訂快取驅動器，我們將使用 `Cache` facade 上的 `extend` 方法。由於其他服務提供者可能會在其 `boot` 方法中嘗試讀取快取值，我們將在 `booting` 回呼中註冊我們的自訂驅動器。透過使用 `booting` 回呼，我們可以確保自訂驅動器在應用程式的服務提供者的 `boot` 方法被呼叫之前，但在所有服務提供者的 `register` 方法被呼叫之後註冊。我們將在應用程式的 `App\Providers\AppServiceProvider` 類別的 `register` 方法中註冊我們的 `booting` 回呼：

    <?php

    namespace App\Providers;

    use App\Extensions\MongoStore;
    use Illuminate\Contracts\Foundation\Application;
    use Illuminate\Support\Facades\Cache;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Register any application services.
         */
        public function register(): void
        {
            $this->app->booting(function () {
                 Cache::extend('mongo', function (Application $app) {
                     return Cache::repository(new MongoStore);
                 });
             });
        }

        /**
         * Bootstrap any application services.
         */
        public function boot(): void
        {
            // ...
        }
    }

傳遞給 `extend` 方法的第一個引數是驅動器的名稱。這將對應於您 `config/cache.php` 設定檔中的 `driver` 選項。第二個引數是一個閉包，它應該返回一個 `Illuminate\Cache\Repository` 實例。該閉包將傳遞一個 `$app` 實例，它是 [服務容器](/docs/{{version}}/container) 的實例。

一旦您的擴充功能已註冊，請更新應用程式 `config/cache.php` 設定檔中的 `CACHE_STORE` 環境變數或 `default` 選項，將其設定為您的擴充功能名稱。

<a name="events"></a>
## 事件

為了在每次快取操作時執行程式碼，您可以監聽由 Cache 分派的各種[事件](/docs/{{version}}/events)：

<div class="overflow-auto">

| 事件名稱 |
| --- |
| `Illuminate\Cache\Events\CacheHit` |
| `Illuminate\Cache\Events\CacheMissed` |
| `Illuminate\Cache\Events\KeyForgotten` |
| `Illuminate\Cache\Events\KeyWritten` |

</div>

為了提高效能，您可以透過在應用程式的 `config/cache.php` 設定檔中，將特定快取儲存區的 `events` 設定選項設為 `false` 來停用快取事件：

```php
'database' => [
    'driver' => 'database',
    // ...
    'events' => false,
],
```