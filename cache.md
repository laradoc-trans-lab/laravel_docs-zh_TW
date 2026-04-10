# 快取

- [簡介](#introduction)
- [設定](#configuration)
    - [驅動程式前提條件](#driver-prerequisites)
- [快取使用方式](#cache-usage)
    - [獲取快取實例](#obtaining-a-cache-instance)
    - [從快取中檢索項目](#retrieving-items-from-the-cache)
    - [將項目儲存在快取中](#storing-items-in-the-cache)
    - [延長項目生命週期](#extending-item-lifetime)
    - [從快取中移除項目](#removing-items-from-the-cache)
    - [快取記憶化](#cache-memoization)
    - [快取輔助函式](#the-cache-helper)
- [快取標記](#cache-tags)
- [原子鎖](#atomic-locks)
    - [管理鎖](#managing-locks)
    - [跨行程管理鎖](#managing-locks-across-processes)
    - [限制併發量](#concurrency-limiting)
- [快取故障轉移](#cache-failover)
- [新增自定義快取驅動程式](#adding-custom-cache-drivers)
    - [撰寫驅動程式](#writing-the-driver)
    - [註冊驅動程式](#registering-the-driver)
- [事件](#events)

<a name="introduction"></a>
## 簡介

您的應用程式所執行的一些資料檢索或處理任務可能會耗費 CPU 資源，或需要數秒鐘才能完成。在這種情況下，通常會將檢索到的資料快取一段時間，以便在隨後對相同資料的請求中能快速檢索。快取資料通常儲存在極速的資料儲存庫中，例如 [Memcached](https://memcached.org) 或 [Redis](https://redis.io)。

幸運的是，Laravel 為各種快取後端提供了一個具表達力且統一的 API，讓您能夠利用其極速的資料檢索能力，並加快您 Web 應用程式的運行速度。


<a name="configuration"></a>
## 設定

您的應用程式快取設定檔位於 `config/cache.php`。在此檔案中，您可以指定應用程式在預設情況下要使用的快取儲存庫。Laravel 開箱即用就支持流行的快取後端，如 [Memcached](https://memcached.org)、[Redis](https://redis.io)、[DynamoDB](https://aws.amazon.com/dynamodb) 以及關聯式資料庫。此外，還提供了一個基於檔案的快取驅動程式，而 `array` 和 `null` 快取驅動程式則為您的自動化測試提供了方便的快取後端。

快取設定檔還包含許多其他選項供您審閱。預設情況下，Laravel 被設定為使用 `database` 快取驅動程式，它將序列化後的快取物件儲存在您應用程式的資料庫中。


<a name="driver-prerequisites"></a>
### 驅動程式前提條件


<a name="prerequisites-database"></a>
#### Database

當使用 `database` 快取驅動程式時，您會需要一個資料庫資料表來存放快取資料。通常，這已包含在 Laravel 預設的 `0001_01_01_000001_create_cache_table.php` [資料庫遷移](/docs/{{version}}/migrations) 中；然而，如果您的應用程式不包含此遷移，您可以使用 `make:cache-table` Artisan 命令來建立它：

```shell
php artisan make:cache-table

php artisan migrate
```


<a name="memcached"></a>
#### Memcached

使用 Memcached 驅動程式需要安裝 [Memcached PECL 套件](https://pecl.php.net/package/memcached)。您可以在 `config/cache.php` 設定檔中列出所有的 Memcached 伺服器。此檔案中已經包含了一個 `memcached.servers` 項目供您開始使用：

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

在 Laravel 中使用 Redis 快取之前，您需要透過 PECL 安裝 PhpRedis PHP 擴充功能，或者透過 Composer 安裝 `predis/predis` 套件 (~2.0)。[Laravel Sail](/docs/{{version}}/sail) 已經包含了此擴充功能。此外，官方的 Laravel 應用程式平台如 [Laravel Cloud](https://cloud.laravel.com) 和 [Laravel Forge](https://forge.laravel.com) 預設已安裝 PhpRedis 擴充功能。

有關配置 Redis 的更多資訊，請參閱其 [Laravel 說明文件頁面](/docs/{{version}}/redis#configuration)。


<a name="dynamodb"></a>
#### DynamoDB

在使用 [DynamoDB](https://aws.amazon.com/dynamodb) 快取驅動程式之前，您必須建立一個 DynamoDB 資料表來儲存所有快取資料。通常，此資料表應命名為 `cache`。然而，您應該根據 `cache` 設定檔中 `stores.dynamodb.table` 設定值來命名資料表。資料表名稱也可以透過 `DYNAMODB_CACHE_TABLE` 環境變數來設定。

此資料表還應該有一個字串分區鍵 (partition key)，其名稱需對應於應用程式 `cache` 設定檔中 `stores.dynamodb.attributes.key` 設定項的值。預設情況下，分區鍵應命名為 `key`。

通常，DynamoDB 不會主動從資料表中移除過期的項目。因此，您應該在資料表上 [啟用生存時間 (TTL)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)。在配置資料表的 TTL 設定時，您應該將 TTL 屬性名稱設定為 `expires_at`。

接下來，安裝 AWS SDK 以便您的 Laravel 應用程式能與 DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

此外，您應確保為 DynamoDB 快取儲存配置選項提供值。通常這些選項，例如 `AWS_ACCESS_KEY_ID` 和 `AWS_SECRET_ACCESS_KEY`，應定義在您應用程式的 `.env` 設定檔中：

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

如果您使用 MongoDB，官方的 `mongodb/laravel-mongodb` 套件提供了 `mongodb` 快取驅動程式，並可以使用 `mongodb` 資料庫連接進行配置。MongoDB 支持 TTL 索引，可用於自動清除過期的快取項目。

有關配置 MongoDB 的更多資訊，請參閱 MongoDB 的 [快取與鎖說明文件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/cache/)。

<a name="cache-usage"></a>
## 快取使用方式


<a name="obtaining-a-cache-instance"></a>
### 獲取快取實例

要獲取快取儲存實例，您可以使用 `Cache` facade，這也是我們在整個文件中將會使用的方式。`Cache` facade 為 Laravel 快取契約（contracts）的底層實作提供了方便且簡潔的存取方式：

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

使用 `Cache` facade，您可以透過 `store` 方法來存取不同的快取儲存。傳遞給 `store` 方法的鍵值應對應於 `cache` 設定檔中 `stores` 設定陣列所列出的其中一個儲存：

```php
$value = Cache::store('file')->get('foo');

Cache::store('redis')->put('bar', 'baz', 600); // 10 Minutes
```


<a name="retrieving-items-from-the-cache"></a>
### 從快取中檢索項目

`Cache` facade 的 `get` 方法用於從快取中檢索項目。如果項目不存在於快取中，將會回傳 `null`。如果您願意，可以傳遞第二個引數給 `get` 方法，指定在項目不存在時您希望回傳的預設值：

```php
$value = Cache::get('key');

$value = Cache::get('key', 'default');
```

您甚至可以傳遞一個閉包作為預設值。如果指定的項目不存在於快取中，將會回傳該閉包的執行結果。傳遞閉包允許您延遲從資料庫或其他外部服務檢索預設值的過程：

```php
$value = Cache::get('key', function () {
    return DB::table(/* ... */)->get();
});
```


<a name="determining-item-existence"></a>
#### 判斷項目是否存在

`has` 方法可用於判斷項目是否存在於快取中。如果項目存在但其值為 `null`，此方法同樣會回傳 `false`：

```php
if (Cache::has('key')) {
    // ...
}
```


<a name="incrementing-decrementing-values"></a>
#### 增加 / 減少值

`increment` 與 `decrement` 方法可用於調整快取中整數項目的值。這兩個方法都接受一個可選的第二個引數，用以指定增加或減少項目的數值量：

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
#### 檢索並儲存

有時您可能希望從快取中檢索一個項目，但如果請求的項目不存在，則儲存一個預設值。例如，您可能希望從快取中檢索所有使用者，如果他們不存在，則從資料庫檢索並將其添加到快取中。您可以使用 `Cache::remember` 方法來實現此功能：

```php
$value = Cache::remember('users', $seconds, function () {
    return DB::table('users')->get();
});
```

如果項目不存在於快取中，傳遞給 `remember` 方法的閉包將被執行，其結果將被放入快取中。

您可以使用 `rememberForever` 方法來從快取中檢索項目，或者在項目不存在時將其永久儲存：

```php
$value = Cache::rememberForever('users', function () {
    return DB::table('users')->get();
});
```


<a name="swr"></a>
#### 過期重新驗證 (Stale While Revalidate)

在使用 `Cache::remember` 方法時，如果快取值已過期，某些使用者可能會遇到回應時間較慢的情況。對於某些類型的數據，允許在背景重新計算快取值的同時提供部分過期的數據會很有用，這樣可以防止某些使用者在計算快取值時遇到回應緩慢的問題。這通常被稱為 "stale-while-revalidate" 模式，而 `Cache::flexible` 方法提供了此模式的實作。

flexible 方法接受一個陣列，用以指定快取值被視為「有效」多久以及何時變為「過期」。陣列中的第一個值代表快取被視為有效的秒數，而第二個值定義了在必要重新計算之前，它可以作為過期數據提供服務的時間長度。

如果在有效期間內（第一個值之前）發出請求，快取將立即回傳而無需重新計算。如果在過期期間內（兩個值之間）發出請求，過期的值將提供給使用者，並且會註冊一個 [延遲函式](/docs/{{version}}/helpers#deferred-functions)，在回應發送給使用者後重新整理快取值。如果在第二個值之後發出請求，快取被視為已失效，值將立即重新計算，這可能會導致使用者感受到較慢的回應：

```php
$value = Cache::flexible('users', [5, 10], function () {
    return DB::table('users')->get();
});
```


<a name="retrieve-delete"></a>
#### 檢索並刪除

如果您需要從快取中檢索一個項目然後刪除該項目，可以使用 `pull` 方法。與 `get` 方法一樣，如果項目不存在於快取中，將會回傳 `null`：

```php
$value = Cache::pull('key');

$value = Cache::pull('key', 'default');
```


<a name="storing-items-in-the-cache"></a>
### 將項目儲存在快取中

您可以使用 `Cache` facade 的 `put` 方法將項目儲存在快取中：

```php
Cache::put('key', 'value', $seconds = 10);
```

如果沒有傳遞儲存時間給 `put` 方法，該項目將被無限期儲存：

```php
Cache::put('key', 'value');
```

除了傳遞整數秒數外，您也可以傳遞一個 `DateTime` 實例來表示快取項目所需的過期時間：

```php
Cache::put('key', 'value', now()->plus(minutes: 10));
```


<a name="store-if-not-present"></a>
#### 若不存在則儲存

`add` 方法僅在快取儲存中尚不存在該項目時，才會將其添加到快取中。如果項目確實被添加到快取中，該方法將回傳 `true`；否則，該方法將回傳 `false`。`add` 方法是一個原子操作：

```php
Cache::add('key', 'value', $seconds);
```


<a name="extending-item-lifetime"></a>
### 延長項目生命週期

`touch` 方法允許您延長現有快取項目的生命週期 (TTL)。如果快取項目存在且其過期時間成功延長，`touch` 方法將回傳 `true`。如果項目不存在於快取中，該方法將回傳 `false`：

```php
Cache::touch('key', 3600);
```

您可以提供 `DateTimeInterface`、`DateInterval` 或 `Carbon` 實例來指定確切的過期時間：

```php
Cache::touch('key', now()->addHours(2));
```


<a name="storing-items-forever"></a>
#### 永久儲存項目

`forever` 方法可用於將項目永久儲存在快取中。由於這些項目不會過期，必須使用 `forget` 方法手動從快取中移除：

```php
Cache::forever('key', 'value');
```

> [!NOTE]
> 如果您使用 Memcached 驅動程式，儲存為 "forever" 的項目在快取達到大小限制時可能會被移除。

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

您可以使用 `flushLocks` 方法清除快取中所有的原子鎖：

```php
Cache::flushLocks();
```

> [!WARNING]
> 清除快取不會理會您設定的快取 "prefix"，且會移除快取中的所有項目。當您清除由其他應用程式共用的快取時，請務必謹慎考慮。


<a name="cache-memoization"></a>
### 快取記憶化

Laravel 的 `memo` 快取驅動程式允許您在單次請求或工作執行期間，將解析後的快取值暫時儲存在記憶體中。這可以防止在同一次執行中重複命中快取，進而顯著提升效能。

若要使用記憶化快取，請呼叫 `memo` 方法：

```php
use Illuminate\Support\Facades\Cache;

$value = Cache::memo()->get('key');
```

`memo` 方法可以選擇性地接收快取儲存名稱，用以指定記憶化驅動程式要裝飾的底層快取儲存：

```php
// Using the default cache store...
$value = Cache::memo()->get('key');

// Using the Redis cache store...
$value = Cache::memo('redis')->get('key');
```

針對給定鍵值的第一次 `get` 呼叫會從您的快取儲存中檢索值，但同一次請求或工作中的後續呼叫將直接從記憶體中檢索值：

```php
// Hits the cache...
$value = Cache::memo()->get('key');

// Does not hit the cache, returns memoized value...
$value = Cache::memo()->get('key');
```

當呼叫修改快取值的方法（例如 `put`、`increment`、`remember` 等）時，記憶化快取會自動忘記該記憶化值，並將修改方法呼叫委派給底層快取儲存：

```php
Cache::memo()->put('name', 'Taylor'); // Writes to underlying cache...
Cache::memo()->get('name');           // Hits underlying cache...
Cache::memo()->get('name');           // Memoized, does not hit cache...

Cache::memo()->put('name', 'Tim');    // Forgets memoized value, writes new value...
Cache::memo()->get('name');           // Hits underlying cache again...
```


<a name="the-cache-helper"></a>
### 快取輔助函式

除了使用 `Cache` Facade 外，您也可以使用全域的 `cache` 函式透過快取來檢索與儲存資料。當 `cache` 函式被傳入單個字串引數時，它會回傳該鍵值對應的值：

```php
$value = cache('key');
```

如果您向函式提供鍵值對陣列以及過期時間，它會將值儲存在快取中指定的時間長度：

```php
cache(['key' => 'value'], $seconds);

cache(['key' => 'value'], now()->plus(minutes: 10));
```

當 `cache` 函式在沒有任何引數的情況下被呼叫時，它會回傳一個 `Illuminate\Contracts\Cache\Factory` 實作的實例，讓您可以呼叫其他快取方法：

```php
cache()->remember('users', $seconds, function () {
    return DB::table('users')->get();
});
```

> [!NOTE]
> 在測試對全域 `cache` 函式的呼叫時，您可以使用 `Cache::shouldReceive` 方法，就像您在[測試 Facade](/docs/{{version}}/mocking#mocking-facades)時一樣。

<a name="cache-tags"></a>
## 快取標記

> [!WARNING]
> 使用 `file`、`dynamodb` 或 `database` 快取驅動程式時，不支援快取標記。


<a name="storing-tagged-cache-items"></a>
### 儲存標記快取項目

快取標記允許您為快取中相關的項目標記標籤，然後清除所有被分配給特定標記的快取值。您可以透過傳入一個標記名稱的有序陣列來存取標記快取。例如，讓我們存取一個標記快取並將值 `put` 到快取中：

```php
use Illuminate\Support\Facades\Cache;

Cache::tags(['people', 'artists'])->put('John', $john, $seconds);
Cache::tags(['people', 'authors'])->put('Anne', $anne, $seconds);
```


<a name="accessing-tagged-cache-items"></a>
### 存取標記快取項目

透過標記儲存的項目，在存取時必須提供儲存該值時所使用的標記。要檢索標記快取項目，請將相同的有序標記列表傳遞給 `tags` 方法，然後使用您要檢索的鍵呼叫 `get` 方法：

```php
$john = Cache::tags(['people', 'artists'])->get('John');

$anne = Cache::tags(['people', 'authors'])->get('Anne');
```


<a name="removing-tagged-cache-items"></a>
### 移除標記快取項目

您可以清除所有被分配給某個標記或標記列表的項目。例如，以下程式碼將移除所有標記為 `people`、`authors` 或兩者皆有的快取。因此，`Anne` 和 `John` 都會從快取中被移除：

```php
Cache::tags(['people', 'authors'])->flush();
```

相反地，以下程式碼僅會移除標記為 `authors` 的快取值，因此 `Anne` 會被移除，但 `John` 不會：

```php
Cache::tags('authors')->flush();
```


<a name="atomic-locks"></a>
## 原子鎖

> [!WARNING]
> 要使用此功能，您的應用程式必須將 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動程式設定為預設的快取驅動程式。此外，所有伺服器必須與同一個中央快取伺服器進行通訊。


<a name="managing-locks"></a>
### 管理鎖

原子鎖允許操作分散式鎖而無需擔心競態條件 (race conditions)。例如，[Laravel Cloud](https://cloud.laravel.com) 使用原子鎖來確保伺服器上一次僅執行一個遠端任務。您可以使用 `Cache::lock` 方法來建立並管理鎖：

```php
use Illuminate\Support\Facades\Cache;

$lock = Cache::lock('foo', 10);

if ($lock->get()) {
    // Lock acquired for 10 seconds...

    $lock->release();
}
```

`get` 方法也接受一個閉包 (closure)。在閉包執行完畢後，Laravel 會自動釋放鎖：

```php
Cache::lock('foo', 10)->get(function () {
    // Lock acquired for 10 seconds and automatically released...
});
```

如果您請求鎖時該鎖不可用，您可以指示 Laravel 等待指定秒數。如果在指定的時間限制內無法獲取鎖，將會拋出 `Illuminate\Contracts\Cache\LockTimeoutException` 異常：

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

上述範例可以透過將閉包傳遞給 `block` 方法來簡化。當將閉包傳遞給此方法時，Laravel 將嘗試在指定秒數內獲取鎖，並在閉包執行完畢後自動釋放鎖：

```php
Cache::lock('foo', 10)->block(5, function () {
    // Lock acquired for 10 seconds after waiting a maximum of 5 seconds...
});
```


<a name="managing-locks-across-processes"></a>
### 跨行程管理鎖

有時候，您可能希望在一個行程中獲取鎖，並在另一個行程中釋放它。例如，您可能在 Web 請求期間獲取鎖，並希望在由該請求觸發的佇列作業 (queued job) 結束時釋放該鎖。在這種情況下，您應該將鎖的範圍限定「擁有者令牌 (owner token)」傳遞給佇列作業，以便該作業可以使用給定的令牌重新實例化該鎖。

在下面的範例中，如果成功獲取鎖，我們將分派一個佇列作業。此外，我們將透過鎖的 `owner` 方法將擁有者令牌傳遞給該佇列作業：

```php
$podcast = Podcast::find($id);

$lock = Cache::lock('processing', 120);

if ($lock->get()) {
    ProcessPodcast::dispatch($podcast, $lock->owner());
}
```

在應用程式的 `ProcessPodcast` 作業中，我們可以使用擁有者令牌來還原並釋放鎖：

```php
Cache::restoreLock('processing', $this->owner)->release();
```

如果您想在不考慮當前擁有者的情況下釋放鎖，可以使用 `forceRelease` 方法：

```php
Cache::lock('processing')->forceRelease();
```


<a name="concurrency-limiting"></a>
### 限制併發量

Laravel 的原子鎖功能還提供了幾種限制閉包併發執行的方法。當您希望在整個基礎設施中僅允許一個執行實例時，請使用 `withoutOverlapping`：

```php
Cache::withoutOverlapping('foo', function () {
    // Lock acquired after waiting a maximum of 10 seconds...
});
```

預設情況下，鎖會一直持有直到閉包執行完畢，且該方法最多等待 10 秒以獲取鎖。您可以使用額外參數來自定義這些值：

```php
Cache::withoutOverlapping('foo', function () {
    // Lock acquired for 120 seconds after waiting a maximum of 5 seconds...
}, lockFor: 120, waitFor: 5);
```

如果在指定的等待時間內無法獲取鎖，將會拋出 `Illuminate\Contracts\Cache\LockTimeoutException` 異常。

如果您想要控制並行度，請使用 `funnel` 方法來設定最大併發執行數量。`funnel` 方法適用於任何支援鎖的快取驅動程式：

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

`funnel` 鍵用於識別被限制的資源。`limit` 方法定義最大併發執行數量。`releaseAfter` 方法設定一個安全逾時秒數，在此時間後獲取的槽位 (slot) 會被自動釋放。`block` 方法設定等待可用槽位的秒數。

如果您偏好透過異常而非提供失敗閉包來處理逾時，可以省略第二個閉包。如果在指定的等待時間內無法獲取鎖，將會拋出 `Illuminate\Cache\Limiters\LimiterTimeoutException` 異常：

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

如果您想為併發限制器使用特定的快取儲存，可以在所需的儲存上呼叫 `funnel` 方法：

```php
Cache::store('redis')->funnel('foo')
    ->limit(3)
    ->block(10)
    ->then(function () {
        // Concurrency lock acquired using the "redis" store...
    });
```

> [!NOTE]
> `funnel` 方法要求快取儲存必須實作 `Illuminate\Contracts\Cache\LockProvider` 介面。如果您嘗試在不支援鎖的快取儲存上使用 `funnel`，將會拋出 `BadMethodCallException` 異常。

<a name="cache-failover"></a>
## 快取故障轉移

`failover` 快取驅動程式在與快取互動時提供了自動故障轉移功能。如果 `failover` 儲存庫的主要快取儲存庫因任何原因失效，Laravel 將自動嘗試使用清單中接下來設定的儲存庫。這對於確保快取可靠性至關重要的正式環境中的高可用性特別有用。

要設定故障轉移快取儲存庫，請指定 `failover` 驅動程式並提供一個要按順序嘗試的儲存庫名稱陣列。預設情況下，Laravel 在您的應用程式 `config/cache.php` 設定檔中包含了一個故障轉移設定範例：

```php
'failover' => [
    'driver' => 'failover',
    'stores' => [
        'database',
        'array',
    ],
],
```

一旦您設定了使用 `failover` 驅動程式的儲存庫，您將需要在應用程式的 `.env` 檔案中將該故障轉移儲存庫設定為預設快取儲存庫，以使用故障轉移功能：

```ini
CACHE_STORE=failover
```

當快取儲存庫操作失敗且啟動故障轉移時，Laravel 將會發送 `Illuminate\Cache\Events\CacheFailedOver` 事件，讓您可以回報或記錄快取儲存庫已失效。


<a name="adding-custom-cache-drivers"></a>
## 新增自定義快取驅動程式


<a name="writing-the-driver"></a>
### 撰寫驅動程式

要建立自定義快取驅動程式，我們首先需要實作 `Illuminate\Contracts\Cache\Store` [契約(Contracts)](/docs/{{version}}/contracts)。因此，一個 MongoDB 快取實作看起來可能像這樣：

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

我們只需要使用 MongoDB 連線來實作這些方法。關於如何實作這些方法的範例，請參閱 [Laravel 框架原始碼](https://github.com/laravel/framework) 中的 `Illuminate\Cache\MemcachedStore`。一旦我們的實作完成，我們就可以透過呼叫 `Cache` Facade 的 `extend` 方法來完成自定義驅動程式的註冊：

```php
Cache::extend('mongo', function (Application $app) {
    return Cache::repository(new MongoStore);
});
```

> [!NOTE]
> 如果您在考慮將自定義快取驅動程式的程式碼放在哪裡，您可以在 `app` 目錄中建立一個 `Extensions` 命名空間。不過，請記住 Laravel 沒有僵化的應用程式結構，您可以根據自己的喜好來組織應用程式。


<a name="registering-the-driver"></a>
### 註冊驅動程式

要將自定義快取驅動程式註冊到 Laravel，我們將使用 `Cache` Facade 上的 `extend` 方法。由於其他服務提供者(Service Providers)可能會在它們的 `boot` 方法中嘗試讀取快取值，因此我們將在 `booting` 回呼函式中註冊我們的自定義驅動程式。透過使用 `booting` 回呼函式，我們可以確保自定義驅動程式在應用程式的服務提供者呼叫 `boot` 方法之前，且在所有服務提供者呼叫 `register` 方法之後完成註冊。我們將在應用程式 `App\Providers\AppServiceProvider` 類別的 `register` 方法中註冊我們的 `booting` 回呼函式：

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

傳遞給 `extend` 方法的第一個引數是驅動程式的名稱。這將對應於 `config/cache.php` 設定檔中的 `driver` 選項。第二個引數是一個應該回傳 `Illuminate\Cache\Repository` 實例的閉包。該閉包會收到一個 `$app` 實例，這是一個 [服務容器](/docs/{{version}}/container) 的實例。

一旦您的擴充功能註冊完成，請將應用程式 `config/cache.php` 設定檔中的 `CACHE_STORE` 環境變數或 `default` 選項更新為您的擴充功能名稱。


<a name="events"></a>
## 事件

若要在每次快取操作時執行程式碼，您可以監聽快取發送的各種 [事件](/docs/{{version}}/events)：

<div class="overflow-auto">

| 事件名稱                                      |
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

為了提高效能，您可以將應用程式 `config/cache.php` 設定檔中特定快取儲存庫的 `events` 設定選項設為 `false` 來停用快取事件：

```php
'database' => [
    'driver' => 'database',
    // ...
    'events' => false,
],
```