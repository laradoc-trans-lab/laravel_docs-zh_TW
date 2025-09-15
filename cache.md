# 快取 (Cache)

- [簡介](#introduction)
- [設定](#configuration)
    - [驅動程式先決條件](#driver-prerequisites)
- [快取使用](#cache-usage)
    - [取得快取實例](#obtaining-a-cache-instance)
    - [從快取中取回項目](#retrieving-items-from-the-cache)
    - [在快取中儲存項目](#storing-items-in-the-cache)
    - [從快取中移除項目](#removing-items-from-the-cache)
    - [快取輔助函式](#the-cache-helper)
- [原子鎖 (Atomic Locks)](#atomic-locks)
    - [管理鎖定](#managing-locks)
    - [跨程序管理鎖定](#managing-locks-across-processes)
- [新增自訂快取驅動程式](#adding-custom-cache-drivers)
    - [撰寫驅動程式](#writing-the-driver)
    - [註冊驅動程式](#registering-the-driver)
- [事件](#events)

<a name="introduction"></a>
## 簡介

應用程式執行的一些資料取回或處理任務可能非常耗費 CPU 資源，或需要數秒才能完成。在這種情況下，通常會將取回的資料快取一段時間，以便在後續對相同資料的請求中能快速取回。快取資料通常儲存在非常快速的資料儲存區中，例如 [Memcached](https://memcached.org) 或 [Redis](https://redis.io)。

幸運的是，Laravel 為各種快取後端提供了表達性強且統一的 API，讓您能夠利用它們極快的資料取回速度，加速您的網路應用程式。

<a name="configuration"></a>
## 設定

您的應用程式快取設定檔位於 `config/cache.php`。在此檔案中，您可以指定應用程式預設要使用的快取儲存區。Laravel 預設支援流行的快取後端，例如 [Memcached](https://memcached.org)、[Redis](https://redis.io)、[DynamoDB](https://aws.amazon.com/dynamodb) 和關聯式資料庫。此外，還提供了基於檔案的快取驅動程式，而 `array` 和「null」快取驅動程式則為您的自動化測試提供了方便的快取後端。

快取設定檔還包含您可以檢閱的各種其他選項。預設情況下，Laravel 配置為使用 `database` 快取驅動程式，該驅動程式將序列化的快取物件儲存在應用程式的資料庫中。

<a name="driver-prerequisites"></a>
### 驅動程式先決條件

<a name="prerequisites-database"></a>
#### Database

使用 `database` 快取驅動程式時，您需要一個資料庫資料表來存放快取資料。通常，這會包含在 Laravel 預設的 `0001_01_01_000001_create_cache_table.php` [資料庫遷移](/docs/{{version}}/migrations)中；但是，如果您的應用程式不包含此遷移，您可以使用 `make:cache-table` Artisan 命令來建立它：

```shell
php artisan make:cache-table

php artisan migrate
```

<a name="memcached"></a>
#### Memcached

使用 Memcached 驅動程式需要安裝 [Memcached PECL 套件](https://pecl.php.net/package/memcached)。您可以在 `config/cache.php` 設定檔中列出所有 Memcached 伺服器。此檔案已包含一個 `memcached.servers` 項目供您開始使用：

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

如果需要，您可以將 `host` 選項設定為 UNIX socket 路徑。如果您這樣做，`port` 選項應設定為 `0`：

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

在 Laravel 中使用 Redis 快取之前，您需要透過 PECL 安裝 PhpRedis PHP 擴充功能，或透過 Composer 安裝 `predis/predis` 套件 (~2.0)。[Laravel Sail](/docs/{{version}}/sail) 已包含此擴充功能。此外，官方 Laravel 部署平台，例如 [Laravel Forge](https://forge.laravel.com) 和 [Laravel Vapor](https://vapor.laravel.com)，預設已安裝 PhpRedis 擴充功能。

有關配置 Redis 的更多資訊，請參閱其 [Laravel 說明文件頁面](/docs/{{version}}/redis#configuration)。

<a name="dynamodb"></a>
#### DynamoDB

在使用 [DynamoDB](https://aws.amazon.com/dynamodb) 快取驅動程式之前，您必須建立一個 DynamoDB 資料表來儲存所有快取資料。通常，此資料表應命名為 `cache`。但是，您應該根據 `cache` 設定檔中 `stores.dynamodb.table` 設定值來命名資料表。資料表名稱也可以透過 `DYNAMODB_CACHE_TABLE` 環境變數設定。

此資料表還應具有一個字串分割區鍵，其名稱應與應用程式 `cache` 設定檔中 `stores.dynamodb.attributes.key` 設定項目的值相對應。預設情況下，分割區鍵應命名為 `key`。

通常，DynamoDB 不會主動從資料表中移除過期項目。因此，您應該在資料表上[啟用 Time to Live (TTL)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)。配置資料表的 TTL 設定時，您應該將 TTL 屬性名稱設定為 `expires_at`。

接下來，安裝 AWS SDK，以便您的 Laravel 應用程式可以與 DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

此外，您應該確保為 DynamoDB 快取儲存區設定選項提供了值。通常，這些選項，例如 `AWS_ACCESS_KEY_ID` 和 `AWS_SECRET_ACCESS_KEY`，應在應用程式的 `.env` 設定檔中定義：

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

如果您正在使用 MongoDB，官方的 `mongodb/laravel-mongodb` 套件提供了 `mongodb` 快取驅動程式，並且可以使用 `mongodb` 資料庫連線進行配置。MongoDB 支援 TTL 索引，可用於自動清除過期的快取項目。

有關配置 MongoDB 的更多資訊，請參閱 MongoDB 的 [Cache and Locks 說明文件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/cache/)。

<a name="cache-usage"></a>
## 快取使用

<a name="obtaining-a-cache-instance"></a>
### 取得快取實例

要取得快取儲存區實例，您可以使用 `Cache` Facade，這也是我們在整個說明文件中將使用的。`Cache` Facade 提供了方便、簡潔的方式來存取 Laravel 快取契約的底層實作：

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

使用 `Cache` Facade，您可以透過 `store` 方法存取各種快取儲存區。傳遞給 `store` 方法的鍵應與 `cache` 設定檔中 `stores` 設定陣列中列出的其中一個儲存區相對應：

    $value = Cache::store('file')->get('foo');

    Cache::store('redis')->put('bar', 'baz', 600); // 10 分鐘

<a name="retrieving-items-from-the-cache"></a>
### 從快取中取回項目

`Cache` Facade 的 `get` 方法用於從快取中取回項目。如果項目不存在於快取中，將返回 `null`。如果您願意，可以向 `get` 方法傳遞第二個參數，指定如果項目不存在時您希望返回的預設值：

    $value = Cache::get('key');

    $value = Cache::get('key', 'default');

您甚至可以傳遞一個閉包作為預設值。如果指定的項目不存在於快取中，將返回該閉包的結果。傳遞閉包允許您延遲從資料庫或其他外部服務取回預設值：

    $value = Cache::get('key', function () {
        return DB::table(/* ... */)->get();
    });

<a name="determining-item-existence"></a>
#### 判斷項目是否存在

`has` 方法可用於判斷項目是否存在於快取中。如果項目存在但其值為 `null`，此方法也將返回 `false`：

    if (Cache::has('key')) {
        // ...
    }

<a name="incrementing-decrementing-values"></a>
#### 遞增 / 遞減值

`increment` 和 `decrement` 方法可用於調整快取中整數項目的值。這兩個方法都接受一個可選的第二個參數，指示要遞增或遞減項目值的數量：

    // 如果值不存在，則初始化...
    Cache::add('key', 0, now()->addHours(4));

    // 遞增或遞減值...
    Cache::increment('key');
    Cache::increment('key', $amount);
    Cache::decrement('key');
    Cache::decrement('key', $amount);

<a name="retrieve-store"></a>
#### 取回並儲存

有時您可能希望從快取中取回一個項目，但如果請求的項目不存在，也儲存一個預設值。例如，您可能希望從快取中取回所有使用者，或者如果他們不存在，則從資料庫中取回他們並將他們新增到快取中。您可以使用 `Cache::remember` 方法來執行此操作：

    $value = Cache::remember('users', $seconds, function () {
        return DB::table('users')->get();
    });

如果項目不存在於快取中，傳遞給 `remember` 方法的閉包將會執行，其結果將會放入快取中。

您可以使用 `rememberForever` 方法從快取中取回項目，或者如果它不存在則永久儲存它：

    $value = Cache::rememberForever('users', function () {
        return DB::table('users')->get();
    });

<a name="swr"></a>
#### 陳舊時重新驗證 (Stale While Revalidate)

當使用 `Cache::remember` 方法時，如果快取值已過期，某些使用者可能會遇到緩慢的響應時間。對於某些類型的資料，允許在背景重新計算快取值時提供部分陳舊的資料會很有用，這可以防止某些使用者在計算快取值時遇到緩慢的響應時間。這通常被稱為「stale-while-revalidate」模式，而 `Cache::flexible` 方法提供了此模式的實作。

`flexible` 方法接受一個陣列，該陣列指定快取值被視為「新鮮」的時間長度以及何時變為「陳舊」。陣列中的第一個值表示快取被視為新鮮的秒數，而第二個值定義了在需要重新計算之前可以作為陳舊資料提供的時間長度。

如果在新鮮期內（第一個值之前）發出請求，則快取會立即返回而無需重新計算。如果在陳舊期內（兩個值之間）發出請求，則會向使用者提供陳舊值，並註冊一個[延遲函式](/docs/{{version}}/helpers#deferred-functions)，以便在響應發送給使用者後刷新快取值。如果在第二個值之後發出請求，則快取被視為已過期，並且會立即重新計算值，這可能會導致使用者響應速度變慢：

    $value = Cache::flexible('users', [5, 10], function () {
        return DB::table('users')->get();
    });

<a name="retrieve-delete"></a>
#### 取回並刪除

如果您需要從快取中取回一個項目，然後刪除該項目，您可以使用 `pull` 方法。與 `get` 方法一樣，如果項目不存在於快取中，將返回 `null`：

    $value = Cache::pull('key');

    $value = Cache::pull('key', 'default');

<a name="storing-items-in-the-cache"></a>
### 在快取中儲存項目

您可以使用 `Cache` Facade 上的 `put` 方法將項目儲存在快取中：

    Cache::put('key', 'value', $seconds = 10);

如果沒有將儲存時間傳遞給 `put` 方法，該項目將會永久儲存：

    Cache::put('key', 'value');

除了將秒數作為整數傳遞之外，您還可以傳遞一個 `DateTime` 實例，表示快取項目的預期過期時間：

    Cache::put('key', 'value', now()->addMinutes(10));

<a name="store-if-not-present"></a>
#### 如果不存在則儲存

`add` 方法只會在快取儲存區中不存在該項目時才將其新增到快取中。如果該項目確實新增到快取中，該方法將返回 `true`。否則，該方法將返回 `false`。`add` 方法是一個原子操作：

    Cache::add('key', 'value', $seconds);

<a name="storing-items-forever"></a>
#### 永久儲存項目

`forever` 方法可用於將項目永久儲存在快取中。由於這些項目不會過期，因此必須使用 `forget` 方法手動從快取中移除：

    Cache::forever('key', 'value');

> [!NOTE]  
> 如果您正在使用 Memcached 驅動程式，當快取達到其大小限制時，永久儲存的項目可能會被移除。

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
> 清除快取不會遵守您配置的快取「prefix」，並且會從快取中移除所有項目。在清除由其他應用程式共享的快取時，請仔細考慮這一點。

<a name="the-cache-helper"></a>
### 快取輔助函式

除了使用 `Cache` Facade 之外，您還可以使用全域 `cache` 函式透過快取取回和儲存資料。當 `cache` 函式以單一字串參數呼叫時，它將返回給定鍵的值：

    $value = cache('key');

如果您向函式提供一個鍵/值對陣列和一個過期時間，它將在指定持續時間內將值儲存在快取中：

    cache(['key' => 'value'], $seconds);

    cache(['key' => 'value'], now()->addMinutes(10));

當 `cache` 函式在沒有任何參數的情況下呼叫時，它會返回 `Illuminate\Contracts\Cache\Factory` 實作的一個實例，允許您呼叫其他快取方法：

    cache()->remember('users', $seconds, function () {
        return DB::table('users')->get();
    });

> [!NOTE]  
> 當測試對全域 `cache` 函式的呼叫時，您可以使用 `Cache::shouldReceive` 方法，就像您[測試 Facade](/docs/{{version}}/mocking#mocking-facades) 一樣。

<a name="atomic-locks"></a>
## 原子鎖 (Atomic Locks)

> [!WARNING]  
> 要使用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動程式作為應用程式的預設快取驅動程式。此外，所有伺服器都必須與相同的中央快取伺服器通訊。

<a name="managing-locks"></a>
### 管理鎖定

原子鎖允許操作分散式鎖定，而無需擔心競爭條件。例如，[Laravel Forge](https://forge.laravel.com) 使用原子鎖來確保在伺服器上一次只執行一個遠端任務。您可以使用 `Cache::lock` 方法建立和管理鎖定：

    use Illuminate\Support\Facades\Cache;

    $lock = Cache::lock('foo', 10);

    if ($lock->get()) {
        // 鎖定已取得 10 秒...

        $lock->release();
    }

`get` 方法也接受一個閉包。閉包執行後，Laravel 將自動釋放鎖定：

    Cache::lock('foo', 10)->get(function () {
        // 鎖定已取得 10 秒並自動釋放...
    });

如果您在請求鎖定時無法取得鎖定，您可以指示 Laravel 等待指定的秒數。如果在指定的時間限制內無法取得鎖定，將會拋出 `Illuminate\Contracts\Cache\LockTimeoutException`：

    use Illuminate\Contracts\Cache\LockTimeoutException;

    $lock = Cache::lock('foo', 10);

    try {
        $lock->block(5);

        // 等待最多 5 秒後取得鎖定...
    } catch (LockTimeoutException $e) {
        // 無法取得鎖定...
    } finally {
        $lock->release();
    }

上述範例可以透過將閉包傳遞給 `block` 方法來簡化。當閉包傳遞給此方法時，Laravel 將嘗試在指定的秒數內取得鎖定，並在閉包執行後自動釋放鎖定：

    Cache::lock('foo', 10)->block(5, function () {
        // 等待最多 5 秒後取得鎖定...
    });

<a name="managing-locks-across-processes"></a>
### 跨程序管理鎖定

有時，您可能希望在一個程序中取得鎖定，並在另一個程序中釋放它。例如，您可能在網路請求期間取得鎖定，並希望在該請求觸發的佇列任務結束時釋放鎖定。在這種情況下，您應該將鎖定的範圍「擁有者權杖」傳遞給佇列任務，以便任務可以使用給定的權杖重新實例化鎖定。

在下面的範例中，如果成功取得鎖定，我們將分派一個佇列任務。此外，我們將透過鎖定的 `owner` 方法將鎖定的擁有者權杖傳遞給佇列任務：

    $podcast = Podcast::find($id);

    $lock = Cache::lock('processing', 120);

    if ($lock->get()) {
        ProcessPodcast::dispatch($podcast, $lock->owner());
    }

在我們應用程式的 `ProcessPodcast` 任務中，我們可以使用擁有者權杖恢復並釋放鎖定：

    Cache::restoreLock('processing', $this->owner)->release();

如果您想在不考慮其目前擁有者的情況下釋放鎖定，您可以使用 `forceRelease` 方法：

    Cache::lock('processing')->forceRelease();

<a name="adding-custom-cache-drivers"></a>
## 新增自訂快取驅動程式

<a name="writing-the-driver"></a>
### 撰寫驅動程式

要建立我們的自訂快取驅動程式，我們首先需要實作 `Illuminate\Contracts\Cache\Store` [契約](/docs/{{version}}/contracts)。因此，MongoDB 快取實作可能看起來像這樣：

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

我們只需要使用 MongoDB 連線來實作這些方法中的每一個。有關如何實作這些方法的範例，請查看 [Laravel 框架原始碼](https://github.com/laravel/framework)中的 `Illuminate\Cache\MemcachedStore`。一旦我們的實作完成，我們就可以透過呼叫 `Cache` Facade 的 `extend` 方法來完成我們的自訂驅動程式註冊：

    Cache::extend('mongo', function (Application $app) {
        return Cache::repository(new MongoStore);
    });

> [!NOTE]  
> 如果您想知道將自訂快取驅動程式程式碼放在哪裡，您可以在 `app` 目錄中建立一個 `Extensions` 命名空間。但是，請記住 Laravel 沒有嚴格的應用程式結構，您可以根據自己的偏好自由組織應用程式。

<a name="registering-the-driver"></a>
### 註冊驅動程式

為了向 Laravel 註冊自訂快取驅動程式，我們將使用 `Cache` Facade 上的 `extend` 方法。由於其他服務提供者可能會在其 `boot` 方法中嘗試讀取快取值，我們將在 `booting` 回呼中註冊我們的自訂驅動程式。透過使用 `booting` 回呼，我們可以確保在呼叫應用程式服務提供者的 `boot` 方法之前，但所有服務提供者的 `register` 方法呼叫之後，註冊自訂驅動程式。我們將在應用程式 `App\Providers\AppServiceProvider` 類別的 `register` 方法中註冊我們的 `booting` 回呼：

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

傳遞給 `extend` 方法的第一個參數是驅動程式的名稱。這將與您在 `config/cache.php` 設定檔中的 `driver` 選項相對應。第二個參數是一個閉包，它應該返回一個 `Illuminate\Cache\Repository` 實例。該閉包將傳遞一個 `$app` 實例，它是 [服務容器](/docs/{{version}}/container) 的一個實例。

一旦您的擴充功能註冊完成，請更新您應用程式 `config/cache.php` 設定檔中的 `CACHE_STORE` 環境變數或 `default` 選項為您的擴充功能名稱。

<a name="events"></a>
## 事件

要在每次快取操作時執行程式碼，您可以監聽快取分派的各種[事件](/docs/{{version}}/events)：

<div class="overflow-auto">

| 事件名稱 |
| --- |
| `Illuminate\Cache\Events\CacheHit` |
| `Illuminate\Cache\Events\CacheMissed` |
| `Illuminate\Cache\Events\KeyForgotten` |
| `Illuminate\Cache\Events\KeyWritten` |

</div>

為了提高效能，您可以透過在應用程式 `config/cache.php` 設定檔中將給定快取儲存區的 `events` 設定選項設定為 `false` 來禁用快取事件：

```php
'database' => [
    'driver' => 'database',
    // ...
    'events' => false,
],
```
