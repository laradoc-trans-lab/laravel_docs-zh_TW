# 快取 (Cache)

- [簡介](#introduction)
- [設定](#configuration)
    - [驅動程式前置需求](#driver-prerequisites)
- [快取用法](#cache-usage)
    - [取得快取實例](#obtaining-a-cache-instance)
    - [從快取檢索項目](#retrieving-items-from-the-cache)
    - [在快取中儲存項目](#storing-items-in-the-cache)
    - [從快取中移除項目](#removing-items-from-the-cache)
    - [快取備忘 (Cache Memoization)](#cache-memoization)
    - [Cache 輔助函式](#the-cache-helper)
- [快取標籤](#cache-tags)
- [原子鎖 (Atomic Locks)](#atomic-locks)
    - [管理鎖](#managing-locks)
    - [跨行程管理鎖](#managing-locks-across-processes)
    - [併發限制](#concurrency-limiting)
- [快取故障轉移 (Cache Failover)](#cache-failover)
- [新增自定義快取驅動程式](#adding-custom-cache-drivers)
    - [撰寫驅動程式](#writing-the-driver)
    - [註冊驅動程式](#registering-the-driver)
- [事件](#events)

<a name="introduction"></a>
## 簡介

應用程式執行的某些資料檢索或處理任務可能是 CPU 密集型的，或者需要幾秒鐘才能完成。在這種情況下，通常會將檢索到的資料快取一段時間，以便在後續對相同資料的請求中能快速檢索。快取的資料通常儲存在非常快速的資料儲存區中，例如 [Memcached](https://memcached.org) 或 [Redis](https://redis.io)。

慶幸的是，Laravel 為各種快取後端提供了表達式豐富且統一的 API，讓您可以利用它們極快的資料檢索速度來加快您的 Web 應用程式。


<a name="configuration"></a>
## 設定

您應用程式的快取設定檔位於 `config/cache.php`。在此檔案中，您可以指定整個應用程式預設要使用的快取儲存區 (Cache Store)。Laravel 開箱即用地支援流行的快取後端，如 [Memcached](https://memcached.org)、[Redis](https://redis.io)、[DynamoDB](https://aws.amazon.com/dynamodb) 和關聯式資料庫。此外，還提供了一個基於檔案的快取驅動程式，而 `array` 和 `null` 快取驅動程式則為您的自動化測試提供了便利的快取後端。

快取設定檔還包含各種其他您可以查看的選項。預設情況下，Laravel 設定為使用 `database` 快取驅動程式，它將序列化的快取物件儲存在應用程式的資料庫中。


<a name="driver-prerequisites"></a>
### 驅動程式前置需求


<a name="prerequisites-database"></a>
#### 資料庫

當使用 `database` 快取驅動程式時，您需要一個資料庫表來包含快取資料。通常，這包含在 Laravel 預設的 `0001_01_01_000001_create_cache_table.php` [資料庫遷移 (Migration)](/docs/{{version}}/migrations) 中；但是，如果您的應用程式不包含此遷移，您可以使用 `make:cache-table` Artisan 指令來建立它：

```shell
php artisan make:cache-table

php artisan migrate
```


<a name="memcached"></a>
#### Memcached

使用 Memcached 驅動程式需要安裝 [Memcached PECL 套件](https://pecl.php.net/package/memcached)。您可以在 `config/cache.php` 設定檔中列出所有的 Memcached 伺服器。此檔案已經包含了一個 `memcached.servers` 項目來供您開始使用：

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

在將 Redis 快取與 Laravel 一起使用之前，您需要透過 PECL 安裝 PhpRedis PHP 擴充功能，或者透過 Composer 安裝 `predis/predis` 套件 (~2.0)。[Laravel Sail](/docs/{{version}}/sail) 已經包含了這個擴充功能。此外，官方的 Laravel 應用程式平台，如 [Laravel Cloud](https://cloud.laravel.com) 和 [Laravel Forge](https://forge.laravel.com)，預設都安裝了 PhpRedis 擴充功能。

有關設定 Redis 的更多資訊，請參閱其 [Laravel 文件頁面](/docs/{{version}}/redis#configuration)。


<a name="dynamodb"></a>
#### DynamoDB

在使用 [DynamoDB](https://aws.amazon.com/dynamodb) 快取驅動程式之前，您必須建立一個 DynamoDB 資料表來儲存所有快取資料。通常，此表應命名為 `cache`。但是，您應該根據 `cache` 設定檔中 `stores.dynamodb.table` 設定項的值來命名該表。表名也可以透過 `DYNAMODB_CACHE_TABLE` 環境變數來設定。

此表還應具有一個字串分割鍵 (Partition Key)，其名稱對應於應用程式 `cache` 設定檔中 `stores.dynamodb.attributes.key` 設定項的值。預設情況下，分割鍵應命名為 `key`。

通常，DynamoDB 不會主動從表中移除過期的項目。因此，您應該在該表上 [啟用存活時間 (TTL)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)。設定表的 TTL 設定時，應將 TTL 屬性名稱設定為 `expires_at`。

接下來，安裝 AWS SDK，以便您的 Laravel 應用程式可以與 DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

此外，您應確保為 DynamoDB 快取儲存區設定選項提供值。通常這些選項（如 `AWS_ACCESS_KEY_ID` 和 `AWS_SECRET_ACCESS_KEY`）應在應用程式的 `.env` 設定檔中定義：

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

如果您正在使用 MongoDB，官方的 `mongodb/laravel-mongodb` 套件提供了一個 `mongodb` 快取驅動程式，並且可以使用 `mongodb` 資料庫連線進行設定。MongoDB 支援 TTL 索引，可用於自動清除過期的快取項目。

有關設定 MongoDB 的更多資訊，請參閱 MongoDB [快取與鎖 (Cache and Locks) 文件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/cache/)。

<a name="cache-usage"></a>
## 快取用法

<a name="obtaining-a-cache-instance"></a>
### 取得快取實例

若要取得快取儲存空間的實例，您可以使用 `Cache` facade，這也是我們在整份文件中所使用的。`Cache` facade 提供了簡潔且方便的方式來存取 Laravel 快取 contracts 的底層實作：

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
#### 存取多個快取儲存空間

使用 `Cache` facade，您可以透過 `store` 方法存取各種快取儲存空間。傳遞給 `store` 方法的鍵 (Key) 應對應於 `cache` 設定檔中 `stores` 設定陣列裡列出的其中一個儲存空間：

```php
$value = Cache::store('file')->get('foo');

Cache::store('redis')->put('bar', 'baz', 600); // 10 Minutes
```

<a name="retrieving-items-from-the-cache"></a>
### 從快取檢索項目

`Cache` facade 的 `get` 方法用於從快取中檢索項目。如果該項目在快取中不存在，則會回傳 `null`。如果您願意，可以將第二個參數傳遞給 `get` 方法，指定如果項目不存在時您希望回傳的預設值：

```php
$value = Cache::get('key');

$value = Cache::get('key', 'default');
```

您甚至可以傳遞一個閉包 (Closure) 作為預設值。如果指定的項目在快取中不存在，則會回傳該閉包的結果。傳遞閉包允許您延遲從資料庫或其他外部服務中檢索預設值：

```php
$value = Cache::get('key', function () {
    return DB::table(/* ... */)->get();
});
```

<a name="determining-item-existence"></a>
#### 判斷項目是否存在

`has` 方法可以用於判斷項目是否存在於快取中。如果項目存在但其值為 `null`，此方法也會回傳 `false`：

```php
if (Cache::has('key')) {
    // ...
}
```

<a name="incrementing-decrementing-values"></a>
#### 遞增 / 遞減數值

`increment` 和 `decrement` 方法可用於調整快取中整數項目的值。這兩個方法都接受一個選用的第二個參數，指示要遞增或遞減該項目值的數量：

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
#### 檢索與儲存

有時您可能希望從快取中檢索一個項目，但若要求的項目不存在時也存入一個預設值。例如，您可能希望從快取中檢索所有使用者，如果不存在，則從資料庫檢索並將其新增到快取中。您可以透過 `Cache::remember` 方法來達成：

```php
$value = Cache::remember('users', $seconds, function () {
    return DB::table('users')->get();
});
```

如果項目不存在於快取中，傳遞給 `remember` 方法的閉包將被執行，其結果將被放入快取中。

您可以使用 `rememberForever` 方法從快取中檢索項目，或者在項目不存在時將其永久儲存：

```php
$value = Cache::rememberForever('users', function () {
    return DB::table('users')->get();
});
```

<a name="swr"></a>
#### 陳舊時重新驗證 (Stale While Revalidate)

當使用 `Cache::remember` 方法時，如果快取值已過期，某些使用者可能會遇到回應時間變慢的情況。對於某些類型的資料，在背景重新計算快取值的同時允許提供部分陳舊 (Stale) 的資料會很有用，這可以防止某些使用者在計算快取值時遇到回應時間變慢的情況。這通常被稱為 "stale-while-revalidate" 模式，而 `Cache::flexible` 方法提供了該模式的一種實作。

此 flexible 方法接受一個陣列，該陣列指定快取值被視為「新鮮 (Fresh)」的時間以及何時變為「陳舊 (Stale)」。陣列中的第一個值代表快取被視為新鮮的秒數，而第二個值定義了在需要重新計算之前，它可以作為陳舊資料提供的時間。

如果在新鮮期內（第一個值之前）發出請求，則立即回傳快取而無需重新計算。如果在陳舊期內（兩個值之間）發出請求，則向使用者提供陳舊值，並註冊一個 [延遲函式 (deferred function)](/docs/{{version}}/helpers#deferred-functions) 以在將回應發送給使用者後更新快取值。如果在第二個值之後發出請求，則快取被視為已過期，並且會立即重新計算數值，這可能會導致使用者的回應速度較慢：

```php
$value = Cache::flexible('users', [5, 10], function () {
    return DB::table('users')->get();
});
```

<a name="retrieve-delete"></a>
#### 檢索與刪除

如果您需要從快取中檢索一個項目後立即刪除該項目，您可以使用 `pull` 方法。與 `get` 方法一樣，如果該項目不存在於快取中，將回傳 `null`：

```php
$value = Cache::pull('key');

$value = Cache::pull('key', 'default');
```

<a name="storing-items-in-the-cache"></a>
### 在快取中儲存項目

您可以使用 `Cache` facade 上的 `put` 方法將項目儲存在快取中：

```php
Cache::put('key', 'value', $seconds = 10);
```

如果沒有將儲存時間傳遞給 `put` 方法，該項目將被無限期儲存：

```php
Cache::put('key', 'value');
```

除了傳遞整數秒數外，您還可以傳遞一個 `DateTime` 實例，代表快取項目的期望過期時間：

```php
Cache::put('key', 'value', now()->plus(minutes: 10));
```

<a name="store-if-not-present"></a>
#### 若不存在則儲存

`add` 方法僅在項目尚不存在於快取儲存空間時，才會將其新增至快取。如果項目確實被新增到快取中，該方法將回傳 `true`。否則，該方法將回傳 `false`。`add` 方法是一個原子性操作：

```php
Cache::add('key', 'value', $seconds);
```

<a name="storing-items-forever"></a>
#### 永久儲存項目

`forever` 方法可用於將項目永久儲存在快取中。由於這些項目不會過期，因此必須使用 `forget` 方法手動將其從快取中移除：

```php
Cache::forever('key', 'value');
```

> [!NOTE]
> 如果您使用的是 Memcached 驅動程式，當快取達到其大小限制時，儲存為「永久」的項目可能會被移除。

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
> 清除 (Flush) 快取時不會考慮您設定的快取 "prefix"，並會移除快取中的所有條目。在清除由其他應用程式共享的快取時，請務必謹慎考慮。

<a name="cache-memoization"></a>
### 快取備忘 (Cache Memoization)

Laravel 的 `memo` 快取驅動程式允許你在單次請求或任務執行期間，將已解析的快取值暫時儲存在記憶體中。這可以防止在同一次執行中重複觸及快取，從而顯著提高效能。

要使用備忘快取，請呼叫 `memo` 方法：

```php
use Illuminate\Support\Facades\Cache;

$value = Cache::memo()->get('key');
```

`memo` 方法可以選擇性地接受快取儲存空間的名稱，這指定了備忘驅動程式將裝飾 (decorate) 的底層快取儲存空間：

```php
// Using the default cache store...
$value = Cache::memo()->get('key');

// Using the Redis cache store...
$value = Cache::memo('redis')->get('key');
```

針對給定鍵的第一次 `get` 呼叫會從你的快取儲存空間中檢索值，但在同一個請求或任務中的後續呼叫將直接從記憶體中檢索該值：

```php
// Hits the cache...
$value = Cache::memo()->get('key');

// Does not hit the cache, returns memoized value...
$value = Cache::memo()->get('key');
```

當呼叫修改快取值的方法（例如 `put`、`increment`、`remember` 等）時，備忘快取會自動清除備忘值，並將變動方法呼叫委派給底層快取儲存空間：

```php
Cache::memo()->put('name', 'Taylor'); // Writes to underlying cache...
Cache::memo()->get('name');           // Hits underlying cache...
Cache::memo()->get('name');           // Memoized, does not hit cache...

Cache::memo()->put('name', 'Tim');    // Forgets memoized value, writes new value...
Cache::memo()->get('name');           // Hits underlying cache again...
```


<a name="the-cache-helper"></a>
### Cache 輔助函式

除了使用 `Cache` Facade 之外，你也可以使用全域的 `cache` 函式來透過快取檢索和儲存資料。當使用單一字串參數呼叫 `cache` 函式時，它將回傳給定鍵的值：

```php
$value = cache('key');
```

如果你向該函式提供一個鍵值對陣列和過期時間，它將在快取中儲存指定持續時間的值：

```php
cache(['key' => 'value'], $seconds);

cache(['key' => 'value'], now()->plus(minutes: 10));
```

當不帶任何參數呼叫 `cache` 函式時，它會回傳一個 `Illuminate\Contracts\Cache\Factory` 實作的實例，讓你可以呼叫其他快取方法：

```php
cache()->remember('users', $seconds, function () {
    return DB::table('users')->get();
});
```

> [!NOTE]
> 測試對全域 `cache` 函式的呼叫時，你可以像[測試 Facade](/docs/{{version}}/mocking#mocking-facades)一樣使用 `Cache::shouldReceive` 方法。

<a name="cache-tags"></a>
## 快取標籤

> [!WARNING]
> 使用 `file`、`dynamodb` 或 `database` 快取驅動程式時，不支援快取標籤。

<a name="storing-tagged-cache-items"></a>
### 儲存快取標籤項目

快取標籤允許您為快取中的相關項目加上標籤，然後清除所有被分配了特定標籤的快取值。您可以透過傳入標籤名稱的有順序陣列來存取已標記標籤的快取。例如，讓我們存取一個已標記標籤的快取，並將一個值 `put` 到快取中：

```php
use Illuminate\Support\Facades\Cache;

Cache::tags(['people', 'artists'])->put('John', $john, $seconds);
Cache::tags(['people', 'authors'])->put('Anne', $anne, $seconds);
```

<a name="accessing-tagged-cache-items"></a>
### 存取快取標籤項目

若未提供儲存該值時所使用的標籤，則無法存取透過標籤儲存的項目。要檢索快取標籤項目，請將相同的有順序標籤列表傳遞給 `tags` 方法，然後使用您要檢索的鍵呼叫 `get` 方法：

```php
$john = Cache::tags(['people', 'artists'])->get('John');

$anne = Cache::tags(['people', 'authors'])->get('Anne');
```

<a name="removing-tagged-cache-items"></a>
### 移除快取標籤項目

您可以清除分配了某個標籤或標籤列表的所有項目。例如，以下程式碼將移除所有標記為 `people`、`authors` 或同時標記為兩者的快取。因此，`Anne` 和 `John` 都會從快取中移除：

```php
Cache::tags(['people', 'authors'])->flush();
```

相比之下，下方的程式碼只會移除標記為 `authors` 的快取值，因此 `Anne` 會被移除，但 `John` 不會：

```php
Cache::tags('authors')->flush();
```

<a name="atomic-locks"></a>
## 原子鎖 (Atomic Locks)

> [!WARNING]
> 要使用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動程式作為應用程式的預設快取驅動程式。此外，所有伺服器必須與同一個中央快取伺服器通訊。

<a name="managing-locks"></a>
### 管理鎖

原子鎖允許在不擔心競態條件 (Race Conditions) 的情況下操作分散式鎖。例如，[Laravel Cloud](https://cloud.laravel.com) 使用原子鎖來確保伺服器上一次只執行一個遠端任務。您可以使用 `Cache::lock` 方法建立和管理鎖：

```php
use Illuminate\Support\Facades\Cache;

$lock = Cache::lock('foo', 10);

if ($lock->get()) {
    // Lock acquired for 10 seconds...

    $lock->release();
}
```

`get` 方法也接受一個閉包 (Closure)。執行閉包後，Laravel 會自動釋放鎖：

```php
Cache::lock('foo', 10)->get(function () {
    // Lock acquired for 10 seconds and automatically released...
});
```

如果您請求鎖時該鎖不可用，您可以指示 Laravel 等待指定的秒數。如果無法在指定的時間限制內取得鎖，則會拋出 `Illuminate\Contracts\Cache\LockTimeoutException`：

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

可以透過將閉包傳遞給 `block` 方法來簡化上述範例。當閉包傳遞給此方法時，Laravel 將嘗試在指定的秒數內取得鎖，並在閉包執行完畢後自動釋放鎖：

```php
Cache::lock('foo', 10)->block(5, function () {
    // Lock acquired for 10 seconds after waiting a maximum of 5 seconds...
});
```

<a name="managing-locks-across-processes"></a>
### 跨行程管理鎖

有時，您可能希望在一個行程中取得鎖，並在另一個行程中釋放它。例如，您可能在一個網頁請求期間取得鎖，並希望在該請求觸發的佇列任務 (Queued Job) 結束時釋放鎖。在這種情況下，您應該將鎖的範圍「擁有者權杖 (Owner Token)」傳遞給佇列任務，以便該任務可以使用給定的權杖重新實例化該鎖。

在下面的範例中，如果成功取得鎖，我們將派遣一個佇列任務。此外，我們將透過鎖的 `owner` 方法將鎖的擁有者權杖傳遞給佇列任務：

```php
$podcast = Podcast::find($id);

$lock = Cache::lock('processing', 120);

if ($lock->get()) {
    ProcessPodcast::dispatch($podcast, $lock->owner());
}
```

在我們應用程式的 `ProcessPodcast` 任務中，我們可以使用擁有者權杖還原並釋放鎖：

```php
Cache::restoreLock('processing', $this->owner)->release();
```

如果您想在不考慮當前擁有者的情況下釋放鎖，可以使用 `forceRelease` 方法：

```php
Cache::lock('processing')->forceRelease();
```

<a name="concurrency-limiting"></a>
### 併發限制

Laravel 的原子鎖功能還提供了一些限制閉包併發執行的方法。當您希望在整個基礎設施中僅允許一個執行實例時，請使用 `withoutOverlapping`：

```php
Cache::withoutOverlapping('foo', function () {
    // Lock acquired after waiting a maximum of 10 seconds...
});
```

預設情況下，鎖會一直持有直到閉包執行完畢，且該方法最多等待 10 秒以取得鎖。您可以使用額外的參數來自定義這些值：

```php
Cache::withoutOverlapping('foo', function () {
    // Lock acquired for 120 seconds after waiting a maximum of 5 seconds...
}, lockFor: 120, waitFor: 5);
```

如果無法在指定的等待時間內取得鎖，則會拋出 `Illuminate\Contracts\Cache\LockTimeoutException`。

如果您想要受控的平行執行，請使用 `funnel` 方法來設定最大併發執行數。`funnel` 方法適用於任何支援鎖的快取驅動程式：

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

`funnel` 的鍵 (Key) 用於識別被限制的資源。`limit` 方法定義了最大併發執行數。`releaseAfter` 方法設定了一個安全逾時（以秒為單位），在自動釋放已取得的插槽之前。`block` 方法設定了等待可用插槽的秒數。

如果您偏好透過例外處理逾時，而不是提供失敗閉包，則可以省略第二個閉包。如果無法在指定的等待時間內取得鎖，將會拋出 `Illuminate\Cache\Limiters\LimiterTimeoutException`：

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

如果您想為併發限制器使用特定的快取儲存空間，可以在所需的儲存空間上呼叫 `funnel` 方法：

```php
Cache::store('redis')->funnel('foo')
    ->limit(3)
    ->block(10)
    ->then(function () {
        // Concurrency lock acquired using the "redis" store...
    });
```

> [!NOTE]
> `funnel` 方法要求快取儲存空間實作 `Illuminate\Contracts\Cache\LockProvider` 介面。如果您嘗試在不支援鎖的快取儲存空間上使用 `funnel`，則會拋出 `BadMethodCallException`。

<a name="cache-failover"></a>
## 快取故障轉移 (Cache Failover)

`failover` 快取驅動程式在與快取互動時提供自動故障轉移功能。如果 `failover` 商店的主要快取商店因任何原因失敗，Laravel 將自動嘗試使用清單中設定的下一個商店。這對於確保生產環境中的高可用性特別有用，因為在這些環境中快取的可靠性至關重要。

要設定故障轉移快取商店，請指定 `failover` 驅動程式並提供一個要按順序嘗試的商店名稱陣列。預設情況下，Laravel 在應用程式的 `config/cache.php` 設定檔中包含了一個範例故障轉移設定：

```php
'failover' => [
    'driver' => 'failover',
    'stores' => [
        'database',
        'array',
    ],
],
```

一旦您設定了使用 `failover` 驅動程式的商店，您就需要在應用程式的 `.env` 檔案中將該故障轉移商店設為預設快取商店，以使用故障轉移功能：

```ini
CACHE_STORE=failover
```

當快取商店操作失敗並啟動故障轉移時，Laravel 會派發 `Illuminate\Cache\Events\CacheFailedOver` 事件，讓您可以報告或記錄某個快取商店已失敗。


<a name="adding-custom-cache-drivers"></a>
## 新增自定義快取驅動程式


<a name="writing-the-driver"></a>
### 撰寫驅動程式

要建立我們的自定義快取驅動程式，我們首先需要實作 `Illuminate\Contracts\Cache\Store` [契約 (Contract)](/docs/{{version}}/contracts)。因此，MongoDB 的快取實作可能看起來像這樣：

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

我們只需要使用 MongoDB 連線來實作這些方法中的每一個。有關如何實作這些方法的範例，請查看 [Laravel 框架原始碼](https://github.com/laravel/framework) 中的 `Illuminate\Cache\MemcachedStore`。一旦我們的實作完成，我們就可以透過呼叫 `Cache` Facade 的 `extend` 方法來完成自定義驅動程式的註冊：

```php
Cache::extend('mongo', function (Application $app) {
    return Cache::repository(new MongoStore);
});
```

> [!NOTE]
> 如果您想知道將自定義快取驅動程式程式碼放在哪裡，您可以在 `app` 目錄中建立一個 `Extensions` 命名空間。但是，請記住 Laravel 沒有嚴格的應用程式結構，您可以根據自己的偏好自由組織您的應用程式。


<a name="registering-the-driver"></a>
### 註冊驅動程式

要在 Laravel 中註冊自定義快取驅動程式，我們將使用 `Cache` Facade 上的 `extend` 方法。由於其他服務提供者可能會嘗試在其 `boot` 方法中讀取快取值，我們將在 `booting` 回呼中註冊自定義驅動程式。透過使用 `booting` 回呼，我們可以確保自定義驅動程式在應用程式服務提供者呼叫 `boot` 方法之前註冊，但在所有服務提供者呼叫 `register` 方法之後。我們將在應用程式 `App\Providers\AppServiceProvider` 類別的 `register` 方法中註冊 `booting` 回呼：

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

傳遞給 `extend` 方法的第一個參數是驅動程式的名稱。這將對應到您的 `config/cache.php` 設定檔中的 `driver` 選項。第二個參數是一個閉包 (Closure)，它應該回傳一個 `Illuminate\Cache\Repository` 實例。該閉包將被傳遞一個 `$app` 實例，它是 [服務容器 (Service Container)](/docs/{{version}}/container) 的實例。

一旦您的擴充功能註冊完成，請將環境變數 `CACHE_STORE` 或應用程式 `config/cache.php` 設定檔中的 `default` 選項更新為您的擴充功能名稱。


<a name="events"></a>
## 事件

要在每次快取操作時執行程式碼，您可以監聽由快取派發的各種 [事件 (Events)](/docs/{{version}}/events)：

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

為了提高效能，您可以在應用程式的 `config/cache.php` 設定檔中，將特定快取商店的 `events` 設定選項設為 `false`，藉此停用快取事件：

```php
'database' => [
    'driver' => 'database',
    // ...
    'events' => false,
],
```