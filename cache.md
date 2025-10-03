# 快取

- [簡介](#introduction)
- [設定](#configuration)
    - [驅動器先決條件](#driver-prerequisites)
- [快取使用](#cache-usage)
    - [取得快取實例](#obtaining-a-cache-instance)
    - [從快取中擷取項目](#retrieving-items-from-the-cache)
    - [在快取中儲存項目](#storing-items-in-the-cache)
    - [從快取中移除項目](#removing-items-from-the-cache)
    - [快取記憶化](#cache-memoization)
    - [快取輔助函式](#the-cache-helper)
- [快取標籤](#cache-tags)
- [原子鎖](#atomic-locks)
    - [管理鎖定](#managing-locks)
    - [跨程序管理鎖定](#managing-locks-across-processes)
- [新增自訂快取驅動器](#adding-custom-cache-drivers)
    - [撰寫驅動器](#writing-the-driver)
    - [註冊驅動器](#registering-the-driver)
- [事件](#events)

<a name="introduction"></a>
## 簡介

您的應用程式執行的某些資料擷取或處理任務可能是 CPU 密集型，或需要數秒才能完成。在這種情況下，通常會將擷取到的資料快取一段時間，以便在後續對相同資料的請求中快速取得。快取資料通常儲存在非常快速的資料儲存，例如 [Memcached](https://memcached.org) 或 [Redis](https://redis.io)。

幸運的是，Laravel 為各種快取後端提供了一個表達性強且統一的 API，讓您可以利用它們極快的資料擷取速度，並加速您的網路應用程式。


<a name="configuration"></a>
## 設定

您的應用程式的快取設定檔位於 `config/cache.php`。在此檔案中，您可以指定應用程式中預設要使用的快取儲存。Laravel 開箱即用，支援 [Memcached](https://memcached.org)、[Redis](https://redis.io)、[DynamoDB](https://aws.amazon.com/dynamodb) 和關聯式資料庫等熱門快取後端。此外，還提供了檔案型快取驅動器，而 `array` 和 `null` 快取驅動器則為您的自動化測試提供了便捷的快取後端。

快取設定檔還包含您可以查看的各種其他選項。預設情況下，Laravel 配置為使用 `database` 快取驅動器，它將序列化的快取物件儲存在您的應用程式資料庫中。


<a name="driver-prerequisites"></a>
### 驅動器先決條件


<a name="prerequisites-database"></a>
#### 資料庫

當使用 `database` 快取驅動器時，您需要一個資料庫表格來儲存快取資料。通常，這包含在 Laravel 預設的 `0001_01_01_000001_create_cache_table.php` [資料庫遷移](/docs/{{version}}/migrations)中；但是，如果您的應用程式不包含此遷移，您可以使用 `make:cache-table` Artisan 指令來建立它：

```shell
php artisan make:cache-table

php artisan migrate
```


<a name="memcached"></a>
#### Memcached

使用 Memcached 驅動器需要安裝 [Memcached PECL 套件](https://pecl.php.net/package/memcached)。您可以在 `config/cache.php` 設定檔中列出所有 Memcached 伺服器。此檔案已包含一個 `memcached.servers` 條目，供您開始使用：

```php
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
```

如果需要，您可以將 `host` 選項設定為 UNIX socket 路徑。如果您這樣做，`port` 選項應設定為 `0`：

```php
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
```


<a name="redis"></a>
#### Redis

在 Laravel 中使用 Redis 快取之前，您需要透過 PECL 安裝 PhpRedis PHP 擴充功能，或透過 Composer 安裝 `predis/predis` 套件 (~2.0)。[Laravel Sail](/docs/{{version}}/sail) 已包含此擴充功能。此外，Laravel 官方應用程式平台，例如 [Laravel Cloud](https://cloud.laravel.com) 和 [Laravel Forge](https://forge.laravel.com)，預設都已安裝 PhpRedis 擴充功能。

有關配置 Redis 的更多資訊，請查閱其 [Laravel 文件頁面](/docs/{{version}}/redis#configuration)。


<a name="dynamodb"></a>
#### DynamoDB

在使用 [DynamoDB](https://aws.amazon.com/dynamodb) 快取驅動器之前，您必須建立一個 DynamoDB 表格來儲存所有快取資料。通常，此表格應命名為 `cache`。但是，您應根據 `cache` 設定檔中 `stores.dynamodb.table` 設定值的命名規則來命名表格。表格名稱也可以透過 `DYNAMODB_CACHE_TABLE` 環境變數設定。

此表格還應具有一個字串分割區鍵，其名稱應與您的應用程式 `cache` 設定檔中 `stores.dynamodb.attributes.key` 設定項目的值相對應。預設情況下，分割區鍵應命名為 `key`。

通常，DynamoDB 不會主動從表格中移除過期的項目。因此，您應該在表格上 [啟用存留時間 (TTL)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)。配置表格的 TTL 設定時，您應將 TTL 屬性名稱設定為 `expires_at`。

接下來，安裝 AWS SDK，以便您的 Laravel 應用程式可以與 DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

此外，您應確保為 DynamoDB 快取儲存設定選項提供值。通常，這些選項，例如 `AWS_ACCESS_KEY_ID` 和 `AWS_SECRET_ACCESS_KEY`，應在您的應用程式 `.env` 設定檔中定義：

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

如果您使用 MongoDB，官方的 `mongodb/laravel-mongodb` 套件提供了 `mongodb` 快取驅動器，並且可以使用 `mongodb` 資料庫連線進行配置。MongoDB 支援 TTL 索引，可用於自動清除過期的快取項目。

有關配置 MongoDB 的更多資訊，請參閱 MongoDB [快取與鎖定文件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/cache/)。

<a name="cache-usage"></a>
## 快取使用


<a name="obtaining-a-cache-instance"></a>
### 取得快取實例

要取得快取儲存實例，您可以使用 `Cache` facade，這也是我們將在本文檔中使用的。`Cache` facade 提供了便捷簡潔的途徑來存取 Laravel 快取合約的底層實作：

```php
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
```


<a name="accessing-multiple-cache-stores"></a>
#### 存取多個快取儲存

使用 `Cache` facade，您可以透過 `store` 方法存取各種快取儲存。傳遞給 `store` 方法的鍵應對應您 `cache` 設定檔中 `stores` 設定陣列中列出的其中一個儲存：

```php
$value = Cache::store('file')->get('foo');

Cache::store('redis')->put('bar', 'baz', 600); // 10 Minutes
```


<a name="retrieving-items-from-the-cache"></a>
### 從快取中擷取項目

`Cache` facade 的 `get` 方法用於從快取中擷取項目。如果快取中不存在該項目，將會回傳 `null`。如果需要，您可以向 `get` 方法傳遞第二個參數，指定當項目不存在時希望回傳的預設值：

```php
$value = Cache::get('key');

$value = Cache::get('key', 'default');
```

您甚至可以傳遞一個閉包 (closure) 作為預設值。如果指定的項目在快取中不存在，將會回傳該閉包的結果。傳遞閉包允許您延遲從資料庫或其他外部服務擷取預設值：

```php
$value = Cache::get('key', function () {
    return DB::table(/* ... */)->get();
});
```


<a name="determining-item-existence"></a>
#### 判斷項目是否存在

`has` 方法可用來判斷項目是否存在於快取中。如果項目存在但其值為 `null`，此方法也將回傳 `false`：

```php
if (Cache::has('key')) {
    // ...
}
```


<a name="incrementing-decrementing-values"></a>
#### 遞增 / 遞減值

`increment` 和 `decrement` 方法可用來調整快取中整數項目的值。這兩個方法都接受一個可選的第二個參數，指示要遞增或遞減項目值的數量：

```php
// Initialize the value if it does not exist...
Cache::add('key', 0, now()->addHours(4));

// Increment or decrement the value...
Cache::increment('key');
Cache::increment('key', $amount);
Cache::decrement('key');
Cache::decrement('key', $amount);
```


<a name="retrieve-store"></a>
#### 擷取並儲存

有時您可能希望從快取中擷取項目，但如果請求的項目不存在，也同時儲存一個預設值。例如，您可能希望從快取中擷取所有使用者，或者如果他們不存在，則從資料庫中擷取並將其加入快取。您可以透過 `Cache::remember` 方法來做到這一點：

```php
$value = Cache::remember('users', $seconds, function () {
    return DB::table('users')->get();
});
```

如果快取中不存在該項目，傳遞給 `remember` 方法的閉包將會執行，其結果將被放置在快取中。

您可以使用 `rememberForever` 方法來從快取中擷取項目，或者如果它不存在，則永久儲存它：

```php
$value = Cache::rememberForever('users', function () {
    return DB::table('users')->get();
});
```


<a name="swr"></a>
#### 陳舊時重新驗證

當使用 `Cache::remember` 方法時，如果快取值已過期，某些使用者可能會遇到緩慢的回應時間。對於某些類型的資料，在背景重新計算快取值的同時，允許提供部分陳舊的資料會很有用，這可以防止某些使用者在快取值計算期間遇到緩慢的回應時間。這通常被稱為「陳舊時重新驗證」(stale-while-revalidate) 模式，`Cache::flexible` 方法提供了此模式的實作。

flexible 方法接受一個陣列，指定快取值被視為「新鮮」的時間長度以及何時變為「陳舊」。陣列中的第一個值代表快取被視為新鮮的秒數，而第二個值定義了在需要重新計算之前，它可以作為陳舊資料提供的時間長度。

如果請求在新鮮期內（在第一個值之前）提出，快取將立即回傳，而無需重新計算。如果請求在陳舊期內（在兩個值之間）提出，陳舊值將提供給使用者，並且會註冊一個 [延遲函式](/docs/{{version}}/helpers#deferred-functions) 在回應傳送給使用者後重新整理快取值。如果請求在第二個值之後提出，快取將被視為過期，並立即重新計算值，這可能會導致使用者回應速度變慢：

```php
$value = Cache::flexible('users', [5, 10], function () {
    return DB::table('users')->get();
});
```


<a name="retrieve-delete"></a>
#### 擷取並刪除

如果您需要從快取中擷取項目然後刪除它，可以使用 `pull` 方法。與 `get` 方法一樣，如果快取中不存在該項目，將會回傳 `null`：

```php
$value = Cache::pull('key');

$value = Cache::pull('key', 'default');
```


<a name="storing-items-in-the-cache"></a>
### 在快取中儲存項目

您可以使用 `Cache` facade 的 `put` 方法將項目儲存到快取中：

```php
Cache::put('key', 'value', $seconds = 10);
```

如果沒有將儲存時間傳遞給 `put` 方法，項目將無限期儲存：

```php
Cache::put('key', 'value');
```

除了傳遞整數秒數，您也可以傳遞一個 `DateTime` 實例，代表快取項目所需的過期時間：

```php
Cache::put('key', 'value', now()->addMinutes(10));
```


<a name="store-if-not-present"></a>
#### 如果不存在則儲存

`add` 方法只會在快取儲存中不存在該項目時才將其加入。如果項目確實已加入快取，此方法將回傳 `true`。否則，此方法將回傳 `false`。`add` 方法是一個原子操作：

```php
Cache::add('key', 'value', $seconds);
```


<a name="storing-items-forever"></a>
#### 永久儲存項目

`forever` 方法可用來永久儲存快取中的項目。由於這些項目不會過期，必須使用 `forget` 方法手動從快取中移除：

```php
Cache::forever('key', 'value');
```

> [!NOTE]
> If you are using the Memcached driver, items that are stored "forever" may be removed when the cache reaches its size limit.


<a name="removing-items-from-the-cache"></a>
### 從快取中移除項目

您可以使用 `forget` 方法從快取中移除項目：

```php
Cache::forget('key');
```

您也可以透過提供零或負數的過期秒數來移除項目：

```php
Cache::put('key', 'value', 0);

Cache::put('key', 'value', -5);
```

您可以使用 `flush` 方法清除整個快取：

```php
Cache::flush();
```

> [!WARNING]
> Flushing the cache does not respect your configured cache "prefix" and will remove all entries from the cache. Consider this carefully when clearing a cache which is shared by other applications.

<a name="cache-memoization"></a>
### 快取記憶化

Laravel 的 `memo` 快取驅動器允許您在單次請求或任務執行期間，將已解析的快取值暫時儲存於記憶體中。這可以避免在相同的執行中重複快取命中，大幅提升效能。

若要使用記憶化快取，請呼叫 `memo` 方法：

```php
use Illuminate\Support\Facades\Cache;

$value = Cache::memo()->get('key');
```

`memo` 方法可選地接受一個快取儲存區名稱，該名稱指定了記憶化驅動器將裝飾的底層快取儲存區：

```php
// Using the default cache store...
$value = Cache::memo()->get('key');

// Using the Redis cache store...
$value = Cache::memo('redis')->get('key');
```

對於給定鍵的首次 `get` 呼叫會從您的快取儲存區擷取值，但在相同的請求或任務中，後續呼叫將從記憶體中擷取值：

```php
// Hits the cache...
$value = Cache::memo()->get('key');

// Does not hit the cache, returns memoized value...
$value = Cache::memo()->get('key');
```

當呼叫修改快取值的方法（例如 `put`、`increment`、`remember` 等）時，記憶化快取會自動忘記記憶化值，並將變更方法呼叫委託給底層快取儲存區：

```php
Cache::memo()->put('name', 'Taylor'); // Writes to underlying cache...
Cache::memo()->get('name');           // Hits underlying cache...
Cache::memo()->get('name');           // Memoized, does not hit cache...

Cache::memo()->put('name', 'Tim');    // Forgets memoized value, writes new value...
Cache::memo()->get('name');           // Hits underlying cache again...
```

<a name="the-cache-helper"></a>
### 快取輔助函式

除了使用 `Cache` Facade 之外，您也可以使用全域 `cache` 函式來透過快取擷取並儲存資料。當 `cache` 函式以單一字串參數呼叫時，它將傳回給定鍵的值：

```php
$value = cache('key');
```

如果您向函式提供一個鍵/值對陣列和一個過期時間，它將在指定期間內將值儲存於快取中：

```php
cache(['key' => 'value'], $seconds);

cache(['key' => 'value'], now()->addMinutes(10));
```

當 `cache` 函式不帶任何參數呼叫時，它會傳回 `Illuminate\Contracts\Cache\Factory` 實作的一個實例，讓您能夠呼叫其他快取方法：

```php
cache()->remember('users', $seconds, function () {
    return DB::table('users')->get();
});
```

> [!NOTE]
> 測試全域 `cache` 函式的呼叫時，您可以像 [測試 Facade](/docs/{{version}}/mocking#mocking-facades) 一樣使用 `Cache::shouldReceive` 方法。

<a name="cache-tags"></a>
## 快取標籤

> [!WARNING]
> 當使用 `file`、`dynamodb` 或 `database` 快取驅動器時，不支援快取標籤。

<a name="storing-tagged-cache-items"></a>
### 儲存帶有標籤的快取項目

快取標籤允許您標記快取中相關的項目，然後清除所有已分配給指定標籤的快取值。您可以透過傳遞一個有序的標籤名稱陣列來存取帶有標籤的快取。例如，讓我們存取一個帶有標籤的快取並將一個值 `put` 到快取中：

    use Illuminate\Support\Facades\Cache;

    Cache::tags(['people', 'artists'])->put('John', $john, $seconds);
    Cache::tags(['people', 'authors'])->put('Anne', $anne, $seconds);

<a name="accessing-tagged-cache-items"></a>
### 存取帶有標籤的快取項目

透過標籤儲存的項目，在未提供用於儲存該值的標籤的情況下，可能無法被存取。要擷取帶有標籤的快取項目，請將相同的有序標籤列表傳遞給 `tags` 方法，然後呼叫 `get` 方法並提供您欲擷取之鍵：

    $john = Cache::tags(['people', 'artists'])->get('John');

    $anne = Cache::tags(['people', 'authors'])->get('Anne');

<a name="removing-tagged-cache-items"></a>
### 移除帶有標籤的快取項目

您可以清除所有分配給單一標籤或標籤列表的項目。例如，以下程式碼將移除所有帶有 `people`、`authors` 或兩者皆有的標籤的快取。因此，`Anne` 和 `John` 都將從快取中移除：

    Cache::tags(['people', 'authors'])->flush();

相對地，下面的程式碼將只移除帶有 `authors` 標籤的快取值，因此 `Anne` 將被移除，但 `John` 不會：

    Cache::tags('authors')->flush();

<a name="atomic-locks"></a>
## 原子鎖

> [!WARNING]
> 要使用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動器作為應用程式的預設快取驅動器。此外，所有伺服器必須與相同的中央快取伺服器進行通訊。

<a name="managing-locks"></a>
### 管理鎖定

原子鎖允許操作分散式鎖定，而無需擔心競爭條件。例如，Laravel Cloud 使用原子鎖來確保在伺服器上一次只執行一個遠端任務。您可以透過 `Cache::lock` 方法建立與管理鎖定：

```php
use Illuminate\Support\Facades\Cache;

$lock = Cache::lock('foo', 10);

if ($lock->get()) {
    // Lock acquired for 10 seconds...

    $lock->release();
}
```

`get` 方法也接受一個閉包。閉包執行後，Laravel 會自動釋放鎖定：

```php
Cache::lock('foo', 10)->get(function () {
    // Lock acquired for 10 seconds and automatically released...
});
```

如果鎖定在您請求時不可用，您可以指示 Laravel 等待指定的秒數。如果在指定時間限制內無法取得鎖定，一個 `Illuminate\Contracts\Cache\LockTimeoutException` 將會被拋出：

```php
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
```

上述範例可以透過傳遞閉包給 `block` 方法來簡化。當閉包傳遞給此方法時，Laravel 將嘗試取得指定秒數的鎖定，並在閉包執行後自動釋放鎖定：

```php
Cache::lock('foo', 10)->block(5, function () {
    // Lock acquired for 10 seconds after waiting a maximum of 5 seconds...
});
```

<a name="managing-locks-across-processes"></a>
### 跨程序管理鎖定

有時，您可能希望在一個程序中取得鎖定，並在另一個程序中釋放它。例如，您可以在網頁請求期間取得鎖定，並希望在該請求觸發的排隊任務結束時釋放鎖定。在這種情況下，您應該將鎖定的作用域「擁有者令牌 (owner token)」傳遞給排隊任務，以便該任務可以使用給定的令牌重新實例化鎖定。

在下面的範例中，如果鎖定成功取得，我們將分派一個排隊任務。此外，我們將透過鎖定的 `owner` 方法將鎖定的擁有者令牌傳遞給排隊任務：

```php
$podcast = Podcast::find($id);

$lock = Cache::lock('processing', 120);

if ($lock->get()) {
    ProcessPodcast::dispatch($podcast, $lock->owner());
}
```

在我們應用程式的 `ProcessPodcast` 任務中，我們可以恢復並釋放鎖定，使用擁有者令牌：

```php
Cache::restoreLock('processing', $this->owner)->release();
```

如果您想在不考慮其目前擁有者的情況下釋放鎖定，您可以使用 `forceRelease` 方法：

```php
Cache::lock('processing')->forceRelease();
```

<a name="adding-custom-cache-drivers"></a>
## 新增自訂快取驅動器


<a name="writing-the-driver"></a>
### 撰寫驅動器

為了建立我們自訂的快取驅動器，我們首先需要實作 `Illuminate\Contracts\Cache\Store` [契約](/docs/{{version}}/contracts)。因此，MongoDB 快取實作可能看起來像這樣：

```php
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
```

我們只需要使用 MongoDB 連線來實作這些方法。關於如何實作這些方法的範例，請參考 [Laravel 框架原始碼](https://github.com/laravel/framework) 中的 `Illuminate\Cache\MemcachedStore`。一旦我們的實作完成，我們就可以透過呼叫 `Cache` Facade 的 `extend` 方法來完成自訂驅動器的註冊：

```php
Cache::extend('mongo', function (Application $app) {
    return Cache::repository(new MongoStore);
});
```

> [!NOTE]
> 如果您想知道將自訂快取驅動器程式碼放在哪裡，您可以在 `app` 目錄中建立一個 `Extensions` 命名空間。然而，請記住 Laravel 沒有嚴格的應用程式結構，您可以自由地根據自己的偏好來組織應用程式。


<a name="registering-the-driver"></a>
### 註冊驅動器

為了向 Laravel 註冊自訂快取驅動器，我們將使用 `Cache` Facade 上的 `extend` 方法。由於其他服務提供者可能會在它們的 `boot` 方法中嘗試讀取快取值，我們將在 `booting` 回呼中註冊我們的自訂驅動器。透過使用 `booting` 回呼，我們可以確保自訂驅動器在應用程式的服務提供者呼叫 `boot` 方法之前註冊，但在所有服務提供者呼叫 `register` 方法之後。我們將在應用程式的 `App\Providers\AppServiceProvider` 類別的 `register` 方法中註冊我們的 `booting` 回呼：

```php
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
```

傳遞給 `extend` 方法的第一個引數是驅動器的名稱。這將對應您在 `config/cache.php` 設定檔中的 `driver` 選項。第二個引數是一個閉包，它應該返回一個 `Illuminate\Cache\Repository` 實例。該閉包將傳遞一個 `$app` 實例，該實例是 [服務容器](/docs/{{version}}/container) 的一個實例。

一旦您的擴充功能註冊完成，請將應用程式的 `config/cache.php` 設定檔中的 `CACHE_STORE` 環境變數或 `default` 選項更新為您擴充功能的名稱。


<a name="events"></a>
## 事件

若要在每個快取操作上執行程式碼，您可以監聽由快取分派的各種[事件](/docs/{{version}}/events)：

<div class="overflow-auto">

| 事件名稱                                     |
|----------------------------------------------|
| `Illuminate\Cache\Events\CacheFlushed`       |
| `Illuminate\Cache\Events\CacheFlushing`      |
| `Illuminate\Cache\Events\CacheHit`           |
| `Illuminate\Cache\Events\CacheMissed`        |
| `Illuminate\Cache\Events\ForgettingKey`      |
| `Illuminate\Cache\Events\KeyForgetFailed`    |
| `Illuminate\Cache\Events\KeyForgotten`       |
| `Illuminate\Cache\Events\KeyWriteFailed`     |
| `Illuminate\Cache\Events\KeyWritten`         |
| `Illuminate\Cache\Events\RetrievingKey`      |
| `Illuminate\Cache\Events\RetrievingManyKeys` |
| `Illuminate\Cache\Events\WritingKey`         |
| `Illuminate\Cache\Events\WritingManyKeys`    |

</div>

為了提高效能，您可以透過在應用程式的 `config/cache.php` 設定檔中，將特定快取儲存區的 `events` 設定選項設為 `false` 來停用快取事件：

```php
'database' => [
    'driver' => 'database',
    // ...
    'events' => false,
],
```