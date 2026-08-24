# 快取

- [簡介](#introduction)
- [設定](#configuration)
    - [驅動器前置準備](#driver-prerequisites)
- [快取的使用](#cache-usage)
    - [取得快取實例](#obtaining-a-cache-instance)
    - [從快取中取得項目](#retrieving-items-from-the-cache)
    - [將項目存入快取](#storing-items-in-the-cache)
    - [延長項目存活時間](#extending-item-lifetime)
    - [從快取中刪除項目](#removing-items-from-the-cache)
    - [快取記憶化](#cache-memoization)
    - [Cache 輔助函式](#the-cache-helper)
- [快取標籤](#cache-tags)
- [原子鎖](#atomic-locks)
    - [管理鎖](#managing-locks)
    - [跨行程管理鎖](#managing-locks-across-processes)
    - [重新整理鎖](#refreshing-locks)
    - [並行限制](#concurrency-limiting)
- [快取故障轉移](#cache-failover)
- [新增自訂快取驅動器](#adding-custom-cache-drivers)
    - [撰寫驅動器](#writing-the-driver)
    - [註冊驅動器](#registering-the-driver)
- [事件](#events)

<a name="introduction"></a>
## 簡介

應用程式執行的某些資料取得或處理任務可能會非常消耗 CPU 資源，或是需要花費數秒鐘才能完成。當遇到這種情況時，通常會將取得的資料快取一段時間，以便在後續請求相同的資料時能快速讀取。快取的資料通常會儲存在速度非常快的資料存放區中，例如 [Memcached](https://memcached.org) 或 [Redis](https://redis.io)。

幸運的是，Laravel 為各種快取後端提供了直覺且統一的 API，讓您可以利用它們極速的資料讀取能力來提升 Web 應用程式的效能。


<a name="configuration"></a>
## 設定

您的應用程式快取設定檔位於 `config/cache.php`。在此檔案中，您可以指定整個應用程式預設要使用哪一個快取存放區。Laravel 開箱即支援熱門的快取後端，例如 [Memcached](https://memcached.org)、[Redis](https://redis.io)、[DynamoDB](https://aws.amazon.com/dynamodb)、關聯式資料庫以及檔案系統磁碟。此外，還提供了一個基於檔案的快取驅動器，而 `array` 與 `null` 快取驅動器則為您的自動化測試提供了便利的快取後端。

快取設定檔中還包含許多您可以檢視的其他選項。預設情況下，Laravel 設定為使用 `database` 快取驅動器，它會將序列化後的快取物件儲存在您的應用程式資料庫中。


<a name="driver-prerequisites"></a>
### 驅動器前置準備


<a name="prerequisites-database"></a>
#### Database

當使用 `database` 快取驅動器時，您需要一個資料庫資料表來存放快取資料。通常，這已經包含在 Laravel 預設的 `0001_01_01_000001_create_cache_table.php` [資料庫遷移](/docs/{{version}}/migrations)中；然而，如果您的應用程式不包含此遷移檔，您可以使用 `make:cache-table` Artisan 命令來建立它：

```shell
php artisan make:cache-table

php artisan migrate
```


<a name="memcached"></a>
#### Memcached

使用 Memcached 驅動器需要安裝 [Memcached PECL 套件](https://pecl.php.net/package/memcached)。您可以在 `config/cache.php` 設定檔中列出您所有的 Memcached 伺服器。該檔案已經包含一個 `memcached.servers` 項目供您快速開始：

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

如果有需要，您可以將 `host` 選項設定為 UNIX socket 路徑。如果您這麼做，`port` 選項應設定為 `0`：

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

在 Laravel 中使用 Redis 快取之前，您需要透過 PECL 安裝 PhpRedis PHP 擴充套件，或者透過 Composer 安裝 `predis/predis` 套件。[Laravel Sail](/docs/{{version}}/sail) 已經內建了此擴充套件。此外，官方的 Laravel 應用程式平台，例如 [Laravel Cloud](https://cloud.laravel.com) 與 [Laravel Forge](https://forge.laravel.com)，預設皆已安裝 PhpRedis 擴充套件。

如需更多關於設定 Redis 的資訊，請參考其 [Laravel 文件頁面](/docs/{{version}}/redis#configuration)。


<a name="storage"></a>
#### Storage

`storage` 快取驅動器允許您將快取值儲存在任何已設定的[檔案系統磁碟](/docs/{{version}}/filesystem)上。當您想要將現有的磁碟（例如 S3 磁碟）用作鍵 / 值（key / value）快取存放區時，這會非常實用：

```php
'storage' => [
    'driver' => 'storage',
    'disk' => env('CACHE_STORAGE_DISK'),
    'path' => env('CACHE_STORAGE_PATH', 'framework/cache/data'),
],
```


<a name="dynamodb"></a>
#### DynamoDB

在使用 [DynamoDB](https://aws.amazon.com/dynamodb) 快取驅動器之前，您必須建立一個 DynamoDB 資料表來儲存所有的快取資料。通常，這個資料表應該命名為 `cache`。不過，您應該根據 `cache` 設定檔中的 `stores.dynamodb.table` 設定值來命名資料表。資料表名稱也可以透過 `DYNAMODB_CACHE_TABLE` 環境變數來設定。

此資料表還必須有一個字串分割鍵（Partition Key），其名稱需對應至應用程式 `cache` 設定檔中的 `stores.dynamodb.attributes.key` 設定項目。預設情況下，分割鍵的名稱應為 `key`。

通常情況下，DynamoDB 不會主動從資料表中刪除過期項目。因此，您應該在該資料表上[啟用存活時間 (TTL)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)。在設定資料表的 TTL 設定時，您應該將 TTL 屬性名稱設定為 `expires_at`。

接下來，安裝 AWS SDK 以便您的 Laravel 應用程式可以與 DynamoDB 進行通訊：

```shell
composer require aws/aws-sdk-php
```

此外，您應該確保為 DynamoDB 快取存放區設定選項提供了相應的值。通常這些選項（例如 `AWS_ACCESS_KEY_ID` 和 `AWS_SECRET_ACCESS_KEY`）應該在您應用程式的 `.env` 設定檔中定義：

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

如果您使用的是 MongoDB，官方的 `mongodb/laravel-mongodb` 套件提供了 `mongodb` 快取驅動器，並且可以使用 `mongodb` 資料庫連線進行設定。MongoDB 支援 TTL 索引，可用於自動清除過期的快取項目。

如需更多關於設定 MongoDB 的資訊，請參考 MongoDB 的[快取與鎖定文件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/cache/)。

<a name="cache-usage"></a>
## 快取的使用

<a name="obtaining-a-cache-instance"></a>
### 取得快取實例

要取得快取儲存庫實例，您可以使用 `Cache` Facade，這也是我們在整個說明文件中將使用的工具。`Cache` Facade 提供了簡潔、方便的方式來存取 Laravel 快取契約(Contracts)的底層實作：

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
#### 存取多個快取儲存庫

使用 `Cache` Facade，您可以透過 `store` 方法存取各種快取儲存庫。傳遞給 `store` 方法的鍵名應對應至 `cache` 設定檔中 `stores` 設定陣列裡所列出的其中一個儲存庫：

```php
$value = Cache::store('file')->get('foo');

Cache::store('redis')->put('bar', 'baz', 600); // 10 Minutes
```

<a name="retrieving-items-from-the-cache"></a>
### 從快取中取得項目

`Cache` Facade 的 `get` 方法用於從快取中取得項目。若該項目不存在於快取中，將回傳 `null`。若您需要，可以傳遞第二個引數給 `get` 方法，指定項目不存在時所要回傳的預設值：

```php
$value = Cache::get('key');

$value = Cache::get('key', 'default');
```

您甚至可以傳遞閉包 (Closure) 作為預設值。如果指定的項目不存在於快取中，將會回傳該閉包的執行結果。傳遞閉包可以讓您延後從資料庫或其他外部服務中取得預設值：

```php
$value = Cache::get('key', function () {
    return DB::table(/* ... */)->get();
});
```

<a name="determining-item-existence"></a>
#### 確認項目是否存在

`has` 方法可以用來確認項目是否存在於快取中。如果項目存在但其值為 `null`，該方法也會回傳 `false`：

```php
if (Cache::has('key')) {
    // ...
}
```

<a name="incrementing-decrementing-values"></a>
#### 遞增 / 遞減數值

`increment` 與 `decrement` 方法可以用來調整快取中整數項目的數值。這兩個方法都可以接受選擇性的第二個引數，用以指定要遞增或遞減項目數值的份量：

```php
// Initialize the value if it does not exist...
Cache::add('key', 0, now()->plus(hours: 4));

// Increment or decrement the value...
Cache::increment('key');
Cache::increment('key', $amount);
Cache::decrement('key');
Cache::decrement('key', $amount);
```

<a name="retrieve-store"></a>
#### 取得並存儲

有時您可能希望從快取中取得某個項目，但如果請求的項目不存在，也同時存入預設值。例如，您可能想從快取中取得所有使用者，但如果快取中不存在，則從資料庫中取得並將其新增至快取中。您可以透過 `Cache::remember` 方法來做到這一點：

```php
$value = Cache::remember('users', $seconds, function () {
    return DB::table('users')->get();
});
```

如果項目不存在於快取中，傳遞給 `remember` 方法的閉包將會被執行，且其結果會被放進快取中。

若您需要知道項目是從快取中取得的，還是透過執行給定的閉包取得的，您可以使用 `rememberWithWarmth` 方法。此方法會回傳一個包含快取值以及一個布林值的陣列，該布林值指示該項目是否為「溫熱 (warm)」，意即該項目是從快取中取得，而非從閉包解析出來的：

```php
[$value, $warm] = Cache::rememberWithWarmth('users', $seconds, function () {
    return DB::table('users')->get();
});
```

您可以使用 `rememberForever` 方法從快取中取得項目，若項目不存在則將其永久存儲：

```php
$value = Cache::rememberForever('users', function () {
    return DB::table('users')->get();
});
```

<a name="swr"></a>
#### 過期重新驗證

在使用 `Cache::remember` 方法時，如果快取值已過期，某些使用者可能會遇到回應時間過慢的問題。對於特定類型的資料，允許在背景重新計算快取值的同時提供部分過期的資料會非常有用，從而避免使用者在計算快取值時經歷慢速的回應。這通常被稱為「過期重新驗證 (stale-while-revalidate)」模式，而 `Cache::flexible` 方法提供了此模式的實作。

flexible 方法接受一個陣列，用來指定快取值被視為「有效 (fresh)」的時間以及何時變為「過期 (stale)」。陣列中的第一個值代表快取被視為有效的秒數，而第二個值則定義在必須重新計算之前，該資料可以作為過期資料被提供多久。

如果在有效期間內（第一個值之前）發出請求，快取會立即回傳而無需重新計算。如果在過期期間內（兩個值之間）發出請求，過期值會提供給使用者，並且會在將回應傳送給使用者之後註冊一個[延後函式 (deferred function)](/docs/{{version}}/helpers#deferred-functions) 來更新快取值。如果在第二個值之後發出請求，快取會被視為已失效，並且會立即重新計算數值，這可能會導致使用者的回應速度較慢：

```php
$value = Cache::flexible('users', [5, 10], function () {
    return DB::table('users')->get();
});
```

<a name="retrieve-delete"></a>
#### 取得並刪除

如果您需要從快取中取得一個項目然後刪除該項目，您可以使用 `pull` 方法。與 `get` 方法相同，若該項目不存在於快取中，將會回傳 `null`：

```php
$value = Cache::pull('key');

$value = Cache::pull('key', 'default');
```

<a name="storing-items-in-the-cache"></a>
### 將項目存入快取

您可以使用 `Cache` Facade 上的 `put` 方法將項目存入快取：

```php
Cache::put('key', 'value', $seconds = 10);
```

如果沒有將儲存時間傳遞給 `put` 方法，項目將會被無限期存儲：

```php
Cache::put('key', 'value');
```

除了傳遞整數的秒數外，您也可以傳遞一個代表快取項目預期過期時間的 `DateTime` 實例：

```php
Cache::put('key', 'value', now()->plus(minutes: 10));
```

<a name="store-if-not-present"></a>
#### 當項目不存在時才存儲

`add` 方法只有在項目尚不存在於快取儲存庫時，才會將項目新增至快取中。如果項目確實被新增到快取，該方法會回傳 `true`。否則，該方法將回傳 `false`。`add` 方法是一個原子操作：

```php
Cache::add('key', 'value', $seconds);
```

<a name="extending-item-lifetime"></a>
### 延長項目存活時間

`touch` 方法允許您延長現有快取項目的存活時間 (TTL)。如果快取項目存在且其過期時間成功被延長，`touch` 方法將回傳 `true`。如果項目不存在於快取中，該方法將回傳 `false`：

```php
Cache::touch('key', 3600);
```

您可以提供 `DateTimeInterface`、`DateInterval` 或 `Carbon` 實例來指定確切的過期時間：

```php
Cache::touch('key', now()->addHours(2));
```

<a name="storing-items-forever"></a>
#### 永久存儲項目

`forever` 方法可以用來將項目永久存儲於快取中。由於這些項目不會過期，因此必須使用 `forget` 方法手動將其從快取中刪除：

```php
Cache::forever('key', 'value');
```

> [!NOTE]
> 如果您使用的是 Memcached 驅動器，當快取達到其大小限制時，永久存儲的項目可能會被刪除。

<a name="removing-items-from-the-cache"></a>
### 從快取中刪除項目

您可以使用 `forget` 方法從快取中刪除項目：

```php
Cache::forget('key');
```

您也可以透過提供 0 或負數的過期秒數來刪除項目：

```php
Cache::put('key', 'value', 0);

Cache::put('key', 'value', -5);
```

您可以使用 `flush` 方法清空整個快取：

```php
Cache::flush();
```

您可以使用 `flushLocks` 方法清空快取中的所有原子鎖：

```php
Cache::flushLocks();
```

> [!WARNING]
> 清空快取不會考量您設定的快取 "prefix"（前綴），並且會移除快取中的所有項目。在清空與其他應用程式共用的快取時，請務必審慎評估。


<a name="cache-memoization"></a>
### 快取記憶化

Laravel 的 `memo` 快取驅動器允許您在單一請求或任務 (job) 執行的過程中，將解析後的快取值暫存於記憶體中。這能防止在同一次執行中重複存取快取，從而顯著提升效能。

要使用記憶化快取，請呼叫 `memo` 方法：

```php
use Illuminate\Support\Facades\Cache;

$value = Cache::memo()->get('key');
```

`memo` 方法可以選擇性地接受快取儲存庫的名稱，這指定了記憶化驅動器所要裝飾的底層快取儲存庫：

```php
// Using the default cache store...
$value = Cache::memo()->get('key');

// Using the Redis cache store...
$value = Cache::memo('redis')->get('key');
```

針對特定鍵 (key) 的第一次 `get` 呼叫會從您的快取儲存庫取得數值，但在同一個請求或任務中的後續呼叫則會直接從記憶體中讀取：

```php
// Hits the cache...
$value = Cache::memo()->get('key');

// Does not hit the cache, returns memoized value...
$value = Cache::memo()->get('key');
```

當呼叫會修改快取值的方法時（例如 `put`、`increment`、`remember` 等），記憶化快取會自動清除記憶中的數值，並將此變更狀態的方法呼叫委派給底層的快取儲存庫：

```php
Cache::memo()->put('name', 'Taylor'); // Writes to underlying cache...
Cache::memo()->get('name');           // Hits underlying cache...
Cache::memo()->get('name');           // Memoized, does not hit cache...

Cache::memo()->put('name', 'Tim');    // Forgets memoized value, writes new value...
Cache::memo()->get('name');           // Hits underlying cache again...
```


<a name="the-cache-helper"></a>
### Cache 輔助函式

除了使用 `Cache` Facade 之外，您也可以使用全域的 `cache` 函式透過快取來取得與儲存資料。當呼叫 `cache` 函式且只傳入單一字串引數時，它將會回傳該鍵的值：

```php
$value = cache('key');
```

如果您向該函式傳入一個鍵／值對陣列以及過期時間，它將會在指定的時長內將這些值存入快取中：

```php
cache(['key' => 'value'], $seconds);

cache(['key' => 'value'], now()->plus(minutes: 10));
```

當呼叫 `cache` 函式且不帶任何引數時，它會回傳 `Illuminate\Contracts\Cache\Factory` 實作的實例，讓您可以呼叫其他快取方法：

```php
cache()->remember('users', $seconds, function () {
    return DB::table('users')->get();
});
```

> [!NOTE]
> 當測試對全域 `cache` 函式的呼叫時，您可以像[測試 Facade](/docs/{{version}}/mocking#mocking-facades) 一樣使用 `Cache::shouldReceive` 方法。

<a name="cache-tags"></a>
## 快取標籤

> [!WARNING]
> 當使用 `file`、`dynamodb`、`database` 或 `storage` 快取驅動器時，不支援快取標籤。

<a name="storing-tagged-cache-items"></a>
### 儲存帶有標籤的快取項目

快取標籤允許您為快取中的相關項目打上標籤，接著就能清除所有被指派該標籤的快取值。您可以透過傳入標籤名稱的有序陣列來存取帶有標籤的快取。例如，讓我們存取帶有標籤的快取並將一個值 `put` 到快取中：

```php
use Illuminate\Support\Facades\Cache;

Cache::tags(['people', 'artists'])->put('John', $john, $seconds);
Cache::tags(['people', 'authors'])->put('Anne', $anne, $seconds);
```

<a name="accessing-tagged-cache-items"></a>
### 存取帶有標籤的快取項目

透過標籤儲存的項目，若未同時提供用於儲存該值的標籤，將無法被存取。若要取得帶有標籤的快取項目，請將相同順序的標籤列表傳入 `tags` 方法，然後呼叫 `get` 方法並傳入您想要取得的鍵名：

```php
$john = Cache::tags(['people', 'artists'])->get('John');

$anne = Cache::tags(['people', 'authors'])->get('Anne');
```

<a name="removing-tagged-cache-items"></a>
### 刪除帶有標籤的快取項目

您可以清除指派給某個標籤或標籤列表的所有項目。例如，以下程式碼將刪除標有 `people`、`authors` 或兩者皆有的所有快取。因此，`Anne` 和 `John` 都會從快取中被刪除：

```php
Cache::tags(['people', 'authors'])->flush();
```

相對地，以下程式碼只會刪除標有 `authors` 的快取值，因此 `Anne` 會被刪除，但 `John` 不會：

```php
Cache::tags('authors')->flush();
```

<a name="atomic-locks"></a>
## 原子鎖

> [!WARNING]
> 若要使用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動器作為應用程式的預設快取驅動器。此外，所有伺服器都必須與相同的中央快取伺服器進行通訊。

<a name="managing-locks"></a>
### 管理鎖

原子鎖允許操作分散式鎖，而無需擔心競爭條件 (Race conditions)。例如，[Laravel Cloud](https://cloud.laravel.com) 使用原子鎖來確保伺服器上一次只會執行一個遠端任務。您可以使用 `Cache::lock` 方法來建立與管理鎖：

```php
use Illuminate\Support\Facades\Cache;

$lock = Cache::lock('foo', 10);

if ($lock->get()) {
    // Lock acquired for 10 seconds...

    $lock->release();
}
```

`get` 方法也接受閉包 (Closure)。在閉包執行完成後，Laravel 會自動釋放該鎖：

```php
Cache::lock('foo', 10)->get(function () {
    // Lock acquired for 10 seconds and automatically released...
});
```

若在請求鎖的當下無法取得該鎖，您可以指示 Laravel 等待指定的秒數。若無法在指定的時間限制內取得鎖，將會拋出 `Illuminate\Contracts\Cache\LockTimeoutException` 例外：

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

上述範例可以透過將閉包傳遞給 `block` 方法來簡化。當傳遞閉包給此方法時，Laravel 會嘗試在指定的秒數內取得鎖，並在閉包執行完畢後自動釋放該鎖：

```php
Cache::lock('foo', 10)->block(5, function () {
    // Lock acquired for 10 seconds after waiting a maximum of 5 seconds...
});
```

<a name="managing-locks-across-processes"></a>
### 跨行程管理鎖

有時候，您可能希望在某個行程 (Process) 中取得鎖，並在另一個行程中釋放它。例如，您可能會在網頁請求期間取得鎖，並希望在該請求所觸發的佇列任務 (Queued job) 結束時釋放鎖。在這種情境下，您應該將該鎖作用域內的「所有者令牌 (Owner token)」傳遞給佇列任務，以便該任務可以使用給定的令牌重新實例化該鎖。

在下面的範例中，若成功取得鎖，我們將分派一個佇列任務。此外，我們將透過鎖的 `owner` 方法把鎖的所有者令牌傳遞給佇列任務：

```php
$podcast = Podcast::find($id);

$lock = Cache::lock('processing', 120);

if ($lock->get()) {
    ProcessPodcast::dispatch($podcast, $lock->owner());
}
```

在我們的應用程式 `ProcessPodcast` 任務中，我們可以使用所有者令牌來復原並釋放該鎖：

```php
Cache::restoreLock('processing', $this->owner)->release();
```

若您想無視目前的鎖所有者並直接釋放鎖，可以使用 `forceRelease` 方法：

```php
Cache::lock('processing')->forceRelease();
```

<a name="refreshing-locks"></a>
### 重新整理鎖

若您需要延長目前擁有的鎖的過期時間，可以使用 `refresh` 方法。若未提供秒數，則會使用該鎖原本的持續時間。這對於長時間執行的操作非常有用，您可以選擇取得短時間的鎖並定期延長它，而不是取得過期時間非常長的鎖：

```php
$lock = Cache::lock('generate-reports', 60);

if ($lock->get()) {
    foreach ($reports as $report) {
        $report->generate();

        // Extend the lock for another 60 seconds...
        $lock->refresh();
    }

    $lock->release();
}
```

<a name="concurrency-limiting"></a>
### 並行限制

Laravel 的原子鎖功能也提供了幾種限制閉包並行執行的做法。當您希望整個基礎架構中只允許一個正在執行的實例時，可以使用 `withoutOverlapping`：

```php
Cache::withoutOverlapping('foo', function () {
    // Lock acquired after waiting a maximum of 10 seconds...
});
```

預設情況下，鎖會保持被持有的狀態直到閉包執行完畢，且該方法最多等待 10 秒來取得鎖。您可以透過額外的引數來自訂這些數值：

```php
Cache::withoutOverlapping('foo', function () {
    // Lock acquired for 120 seconds after waiting a maximum of 5 seconds...
}, lockFor: 120, waitFor: 5);
```

若無法在指定的等待時間內取得鎖，將會拋出 `Illuminate\Contracts\Cache\LockTimeoutException` 例外。

若您想要有控制地進行並列處理，可以使用 `funnel` 方法來設定最大的並行執行數量。`funnel` 方法適用於任何支援鎖的快取驅動器：

```php
Cache::funnel('foo')
    ->limit(3)
    ->releaseAfter(60)
    ->block(10)
    ->then(function () {
        // Concurrency lock acquired...
    }, function () {
        // Could not acquire concurrency lock...
    });
```

`funnel` 的金鑰用來識別被限制的資源。`limit` 方法定義了最大的並行執行數。`releaseAfter` 方法設定了一個以秒為單位的安全逾時時間，在該時間過後，已取得的名額會自動釋放。`block` 方法設定了要等待可用名額的秒數。

若您偏好透過例外處理逾時情況，而不是提供失敗時的閉包，則可以省略第二個閉包。若無法在指定的等待時間內取得鎖，將會拋出 `Illuminate\Cache\Limiters\LimiterTimeoutException` 例外：

```php
use Illuminate\Cache\Limiters\LimiterTimeoutException;

try {
    Cache::funnel('foo')
        ->limit(3)
        ->releaseAfter(60)
        ->block(10)
        ->then(function () {
            // Concurrency lock acquired...
        });
} catch (LimiterTimeoutException $e) {
    // Unable to acquire concurrency lock...
}
```

若您想為並行限制器使用特定的快取 store，可以在所需的 store 上呼叫 `funnel` 方法：

```php
Cache::store('redis')->funnel('foo')
    ->limit(3)
    ->block(10)
    ->then(function () {
        // Concurrency lock acquired using the "redis" store...
    });
```

> [!NOTE]
> `funnel` 方法需要快取 store 實作 `Illuminate\Contracts\Cache\LockProvider` 介面。若您嘗試在不支援鎖的快取 store 上使用 `funnel`，將會拋出 `BadMethodCallException`。

<a name="cache-failover"></a>
## 快取故障轉移

`failover` 快取驅動器在與快取互動時提供了自動故障轉移 (Failover) 的功能。若 `failover` store 的主要快取 store 因任何原因失敗，Laravel 將會自動嘗試使用設定清單中的下一個 store。這對於在快取可靠性至關重要的正式環境中確保高可用性特別有用。

若要設定故障轉移快取 store，請指定 `failover` 驅動器並提供依序嘗試的 store 名稱陣列。預設情況下，Laravel 在您應用程式的 `config/cache.php` 設定檔中包含了一個故障轉移設定的範例：

```php
'failover' => [
    'driver' => 'failover',
    'stores' => [
        'database',
        'array',
    ],
],
```

當您設定好使用 `failover` 驅動器的 store 後，您需要將應用程式 `.env` 檔案中的預設快取 store 設定為該故障轉移 store，以發揮故障轉移功能：

```ini
CACHE_STORE=failover
```

當快取 store 操作失敗且觸發故障轉移時，Laravel 會分派 `Illuminate\Cache\Events\CacheFailedOver` 事件，讓您可以回報或記錄快取 store 已經發生故障。

<a name="adding-custom-cache-drivers"></a>
## 新增自訂快取驅動器


<a name="writing-the-driver"></a>
### 撰寫驅動器

要建立自訂的快取驅動器，我們首先需要實作 `Illuminate\Contracts\Cache\Store` [契約(Contracts)](/docs/{{version}}/contracts)。因此，一個 MongoDB 的快取實作可能會像這樣：

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

我們只需要使用 MongoDB 連線來實作這些方法即可。關於如何實作這些方法的範例，可以參考 [Laravel 框架原始碼](https://github.com/laravel/framework) 中的 `Illuminate\Cache\MemcachedStore`。完成實作後，我們可以透過呼叫 `Cache` Facade 的 `extend` 方法來完成自訂驅動器的註冊：

```php
Cache::extend('mongo', function (Application $app) {
    return Cache::repository(new MongoStore);
});
```

> [!NOTE]
> 如果您不知道該將自訂快取驅動器的程式碼放置何處，您可以在 `app` 目錄下建立一個 `Extensions` 命名空間。不過請記住，Laravel 並沒有嚴格限制應用程式結構，您可以根據個人喜好自由規劃應用程式組織方式。


<a name="registering-the-driver"></a>
### 註冊驅動器

要在 Laravel 中註冊自訂快取驅動器，我們將使用 `Cache` Facade 的 `extend` 方法。由於其他的服務提供者(Service Providers)可能會在其 `boot` 方法中嘗試讀取快取值，因此我們將在 `booting` 回呼函式中註冊自訂驅動器。透過使用 `booting` 回呼，我們可以確保自訂驅動器會在應用程式的服務提供者呼叫 `boot` 方法之前、且在所有服務提供者都呼叫完 `register` 方法之後被註冊。我們將在應用程式的 `App\Providers\AppServiceProvider` 類別中的 `register` 方法裡註冊 `booting` 回呼：

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

傳遞給 `extend` 方法的第一個引數是驅動器的名稱，這會對應到 `config/cache.php` 設定檔中的 `driver` 選項。第二個引數是一個閉包，該閉包應回傳一個 `Illuminate\Cache\Repository` 實例。這個閉包會接收一個 `$app` 實例，該實例為 [服務容器](/docs/{{version}}/container) 的實例。

註冊擴充功能後，請將應用程式的 `CACHE_STORE` 環境變數或 `config/cache.php` 設定檔中的 `default` 選項更新為您擴充功能的名稱。


<a name="events"></a>
## 事件

若要在每次快取操作時執行程式碼，您可以監聽由快取所發出的各種 [事件](/docs/{{version}}/events)：

<div class="overflow-auto">

| 事件名稱                                        |
|-------------------------------------------------|
| `Illuminate\Cache\Events\CacheFlushed`          |
| `Illuminate\Cache\Events\CacheFlushing`         |
| `Illuminate\Cache\Events\CacheFlushFailed`      |
| `Illuminate\Cache\Events\CacheLocksFlushed`     |
| `Illuminate\Cache\Events\CacheLocksFlushing`    |
| `Illuminate\Cache\Events\CacheLocksFlushFailed` |
| `Illuminate\Cache\Events\CacheHit`              |
| `Illuminate\Cache\Events\CacheMissed`           |
| `Illuminate\Cache\Events\ForgettingKey`         |
| `Illuminate\Cache\Events\KeyForgetFailed`       |
| `Illuminate\Cache\Events\KeyForgotten`          |
| `Illuminate\Cache\Events\KeyWriteFailed`        |
| `Illuminate\Cache\Events\KeyWritten`            |
| `Illuminate\Cache\Events\RetrievingKey`         |
| `Illuminate\Cache\Events\RetrievingManyKeys`    |
| `Illuminate\Cache\Events\WritingKey`            |
| `Illuminate\Cache\Events\WritingManyKeys`       |

</div>

若要提升效能，您可以透過在應用程式的 `config/cache.php` 設定檔中將特定快取 Store 的 `events` 設定選項設為 `false`，藉此停用快取事件：

```php
'database' => [
    'driver' => 'database',
    // ...
    'events' => false,
],
```