# Cache

- [簡介](#introduction)
- [設定](#configuration)
    - [驅動程式準備工作](#driver-prerequisites)
- [快取用法](#cache-usage)
    - [取得快取實例](#obtaining-a-cache-instance)
    - [從快取取得項目](#retrieving-items-from-the-cache)
    - [將項目存入快取](#storing-items-in-the-cache)
    - [延長項目存活時間](#extending-item-lifetime)
    - [從快取移除項目](#removing-items-from-the-cache)
    - [快取記憶化](#cache-memoization)
    - [Cache 輔助函式](#the-cache-helper)
- [快取標籤](#cache-tags)
- [原子鎖](#atomic-locks)
    - [管理鎖](#managing-locks)
    - [跨行程管理鎖](#managing-locks-across-processes)
    - [併發限制](#concurrency-limiting)
- [快取容錯移轉](#cache-failover)
- [新增自訂快取驅動程式](#adding-custom-cache-drivers)
    - [撰寫驅動程式](#writing-the-driver)
    - [註冊驅動程式](#registering-the-driver)
- [事件](#events)

<a name="introduction"></a>
## 簡介

您應用程式執行的某些資料檢索或處理任務可能會非常耗費 CPU，或者需要花費數秒才能完成。在這種情況下，通常會將擷取到的資料快取一段時間，以便在後續請求相同資料時能夠快速擷取。快取的資料通常會儲存在極快的資料儲存庫中，例如 [Memcached](https://memcached.org) 或 [Redis](https://redis.io)。

幸好，Laravel 為各種快取後端提供了表達力豐富、統一的 API，讓您可以利用它們極快的資料擷取速度，來加速您的網頁應用程式。


<a name="configuration"></a>
## 設定

您的應用程式快取設定檔位於 `config/cache.php`。在此檔案中，您可以指定整個應用程式預設要使用哪個快取儲存庫。Laravel 開箱即支援常見的快取後端，如 [Memcached](https://memcached.org)、[Redis](https://redis.io)、[DynamoDB](https://aws.amazon.com/dynamodb)、關聯式資料庫以及檔案系統磁碟。此外，還提供了一個基於檔案的快取驅動程式，而 `array` 和 `null` 快取驅動程式則為您的自動化測試提供了便利的快取後端。

快取設定檔中還包含許多您可以檢視的其他選項。預設情況下，Laravel 被設定為使用 `database` 快取驅動程式，它會將序列化後的快取物件儲存在您應用程式的資料庫中。


<a name="driver-prerequisites"></a>
### 驅動程式準備工作


<a name="prerequisites-database"></a>
#### 資料庫

當使用 `database` 快取驅動程式時，您需要一個資料庫資料表來存放快取資料。通常，這已經包含在 Laravel 預設的 `0001_01_01_000001_create_cache_table.php` [資料庫遷移](/docs/{{version}}/migrations)中；然而，如果您的應用程式不包含此遷移，您可以使用 `make:cache-table` Artisan 指令來建立它：

```shell
php artisan make:cache-table

php artisan migrate
```


<a name="memcached"></a>
#### Memcached

使用 Memcached 驅動程式需要安裝 [Memcached PECL 套件](https://pecl.php.net/package/memcached)。您可以在 `config/cache.php` 設定檔中列出所有的 Memcached 伺服器。此檔案已經包含一個 `memcached.servers` 項目來協助您開始使用：

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

如果需要，您可以將 `host` 選項設定為 UNIX socket 路徑。如果您這樣做，`port` 選項應該設定為 `0`：

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

在將 Redis 快取與 Laravel 搭配使用之前，您需要透過 PECL 安裝 PhpRedis PHP 擴充功能，或者透過 Composer 安裝 `predis/predis` 套件（~2.0）。[Laravel Sail](/docs/{{version}}/sail) 已經包含了此擴充功能。此外，官方的 Laravel 應用程式平台（如 [Laravel Cloud](https://cloud.laravel.com) 和 [Laravel Forge](https://forge.laravel.com)）預設都已安裝 PhpRedis 擴充功能。

有關設定 Redis 的更多資訊，請參閱其 [Laravel 說明文件頁面](/docs/{{version}}/redis#configuration)。


<a name="storage"></a>
#### Storage

`storage` 快取驅動程式允許您將快取值儲存在任何您應用程式已設定的[檔案系統磁碟](/docs/{{version}}/filesystem)上。當您想要使用現有的磁碟（例如 S3 磁碟）作為鍵值對（key / value）快取儲存庫時，這非常有用：

```php
'storage' => [
    'driver' => 'storage',
    'disk' => env('CACHE_STORAGE_DISK'),
    'path' => env('CACHE_STORAGE_PATH', 'framework/cache/data'),
],
```


<a name="dynamodb"></a>
#### DynamoDB

在使用 [DynamoDB](https://aws.amazon.com/dynamodb) 快取驅動程式之前，您必須建立一個 DynamoDB 資料表來儲存所有的快取資料。通常，此資料表應命名為 `cache`。然而，您應該根據 `cache` 設定檔中的 `stores.dynamodb.table` 設定值來為該資料表命名。資料表名稱也可以透過 `DYNAMODB_CACHE_TABLE` 環境變數來設定。

此資料表還應該有一個字串分割鍵（Partition key），其名稱對應到您應用程式的 `cache` 設定檔中的 `stores.dynamodb.attributes.key` 設定項目。預設情況下，分割鍵應命名為 `key`。

通常，DynamoDB 不會主動從資料表中刪除過期的項目。因此，您應該在資料表上[啟用存活時間 (TTL)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)。在設定資料表的 TTL 設定時，您應該將 TTL 屬性名稱設定為 `expires_at`。

接下來，安裝 AWS SDK，以便您的 Laravel 應用程式可以與 DynamoDB 進行通訊：

```shell
composer require aws/aws-sdk-php
```

此外，您應該確保為 DynamoDB 快取儲存庫設定選項提供對應的值。通常，這些選項（例如 `AWS_ACCESS_KEY_ID` 和 `AWS_SECRET_ACCESS_KEY`）應該在您應用程式的 `.env` 設定檔中定義：

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

如果您正在使用 MongoDB，官方的 `mongodb/laravel-mongodb` 套件提供了 `mongodb` 快取驅動程式，並且可以使用 `mongodb` 資料庫連線進行設定。MongoDB 支援 TTL 索引，可用於自動清除過期的快取項目。

有關設定 MongoDB 的更多資訊，請參閱 MongoDB [快取與鎖定說明文件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/cache/)。

<a name="cache-usage"></a>
## 快取用法


<a name="obtaining-a-cache-instance"></a>
### 取得快取實例

若要取得快取儲存庫實例，您可以使用 `Cache` Facade，這也是我們在整篇文件中將會使用的方法。`Cache` Facade 提供了便利、簡潔的方法來存取 Laravel 快取契約(Contracts)的底層實作：

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

使用 `Cache` Facade，您可以透過 `store` 方法存取各種不同的快取儲存庫。傳遞給 `store` 方法的鍵（Key）應該對應到您 `cache` 設定檔中 `stores` 設定陣列裡所列出的其中一個儲存庫：

```php
$value = Cache::store('file')->get('foo');

Cache::store('redis')->put('bar', 'baz', 600); // 10 Minutes
```


<a name="retrieving-items-from-the-cache"></a>
### 從快取取得項目

`Cache` Facade 的 `get` 方法用於從快取中取得項目。如果該項目不存在於快取中，將會回傳 `null`。如果您需要，可以向 `get` 方法傳遞第二個引數，以指定當該項目不存在時您希望回傳的預設值：

```php
$value = Cache::get('key');

$value = Cache::get('key', 'default');
```

您甚至可以傳遞一個閉包 (Closure) 作為預設值。如果指定的項目在快取中不存在，將會回傳該閉包的執行結果。傳遞閉包能讓您延遲從資料庫或其他外部服務中取得預設值：

```php
$value = Cache::get('key', function () {
    return DB::table(/* ... */)->get();
});
```


<a name="determining-item-existence"></a>
#### 判斷項目是否存在

`has` 方法可用於判斷某個項目是否存在於快取中。如果該項目存在但其值為 `null`，此方法也會回傳 `false`：

```php
if (Cache::has('key')) {
    // ...
}
```


<a name="incrementing-decrementing-values"></a>
#### 遞增與遞減數值

`increment` 與 `decrement` 方法可用於調整快取中整數項目的值。這兩個方法都接受一個選填的第二個引數，用以指定該項目值要遞增或遞減的幅度：

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
#### 取得並儲存

有時您可能會想從快取中取得一個項目，但同時在該請求項目不存在時儲存一個預設值。例如，您可能想從快取中取得所有使用者，如果快取中不存在，則從資料庫中取得並將其加入快取。您可以使用 `Cache::remember` 方法來達到這個目的：

```php
$value = Cache::remember('users', $seconds, function () {
    return DB::table('users')->get();
});
```

如果該項目不存在於快取中，傳遞給 `remember` 方法的閉包將會被執行，且其執行結果會被寫入快取中。

您可以使用 `rememberForever` 方法來從快取中取得項目，或者在項目不存在時將其永久儲存：

```php
$value = Cache::rememberForever('users', function () {
    return DB::table('users')->get();
});
```


<a name="swr"></a>
#### 過期重新驗證

當使用 `Cache::remember` 方法時，如果快取值已經過期，某些使用者可能會遇到回應緩慢的問題。對於某些類型的資料，允許在背景重新計算快取值時先提供部分過期的資料是很有幫助的，這能防止部分使用者在計算快取值時遇到回應緩慢的情形。這通常被稱為 "stale-while-revalidate"（過期重新驗證）模式，而 `Cache::flexible` 方法提供了此模式的實作。

這個靈活的方法（flexible method）接受一個陣列，用以指定快取值在多長時間內被視為「有效（fresh）」以及何時變為「過期（stale）」。陣列中的第一個值代表快取被視為有效的秒數，而第二個值則定義了在必須重新計算之前，該過期資料還可以被提供多久。

如果請求是在有效期間內（在第一個值之前）發出，快取會立即回傳而無需重新計算。如果請求是在過期期間內（介於這兩個值之間）發出，過期的值會先提供給使用者，並註冊一個[延遲函式(Deferred functions)](/docs/{{version}}/helpers#deferred-functions)在回應傳送給使用者後更新快取值。如果請求是在第二個值之後發出，快取將被視為已過期，且會立即重新計算該值，這可能會導致該使用者收到較慢的回應：

```php
$value = Cache::flexible('users', [5, 10], function () {
    return DB::table('users')->get();
});
```


<a name="retrieve-delete"></a>
#### 取得並刪除

如果您需要從快取中取得一個項目然後將其刪除，可以使用 `pull` 方法。與 `get` 方法類似，如果該項目不存在於快取中，將會回傳 `null`：

```php
$value = Cache::pull('key');

$value = Cache::pull('key', 'default');
```


<a name="storing-items-in-the-cache"></a>
### 將項目存入快取

您可以使用 `Cache` Facade 的 `put` 方法將項目存入快取中：

```php
Cache::put('key', 'value', $seconds = 10);
```

如果沒有將儲存時間傳遞給 `put` 方法，該項目將被永久儲存：

```php
Cache::put('key', 'value');
```

除了傳遞整數的秒數之外，您也可以傳遞一個代表快取項目預期過期時間的 `DateTime` 實例：

```php
Cache::put('key', 'value', now()->plus(minutes: 10));
```


<a name="store-if-not-present"></a>
#### 若不存在則儲存

`add` 方法只有在該項目尚不存在於快取儲存庫時，才會將其加入快取。如果項目成功加入快取，此方法將回傳 `true`。否則，此方法將回傳 `false`。`add` 方法是一個原子操作（Atomic Operation）：

```php
Cache::add('key', 'value', $seconds);
```


<a name="extending-item-lifetime"></a>
### 延長項目存活時間

`touch` 方法允許您延長現有快取項目的存活時間 (TTL)。如果快取項目存在且成功延長其過期時間，`touch` 方法將回傳 `true`。如果該項目不存在於快取中，則該方法會回傳 `false`：

```php
Cache::touch('key', 3600);
```

您可以提供 `DateTimeInterface`、`DateInterval` 或 `Carbon` 實例來指定確切的過期時間：

```php
Cache::touch('key', now()->addHours(2));
```


<a name="storing-items-forever"></a>
#### 永久儲存項目

`forever` 方法可用於將項目永久儲存於快取中。由於這些項目不會過期，因此必須使用 `forget` 方法手動將它們從快取中移除：

```php
Cache::forever('key', 'value');
```

> [!NOTE]
> 如果您使用的是 Memcached 驅動程式，當快取達到其大小限制時，被儲存為 "forever"（永久）的項目仍可能會被移除。

<a name="removing-items-from-the-cache"></a>
### 從快取移除項目

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

您可以使用 `flushLocks` 方法清除快取中的所有原子鎖：

```php
Cache::flushLocks();
```

> [!WARNING]
> 清除整個快取（Flushing）並不會遵守您設定的快取 "prefix"，並且會移除快取中的所有項目。在清除與其他應用程式共享的快取時，請務必仔細考慮。

<a name="cache-memoization"></a>
### 快取記憶化

Laravel 的 `memo` 快取驅動程式允許您在單次請求或任務（Job）執行期間，將解析後的快取值暫時儲存在記憶體中。這可以防止在同一次執行中重複讀取快取，從而顯著提高效能。

若要使用記憶化快取，請呼叫 `memo` 方法：

```php
use Illuminate\Support\Facades\Cache;

$value = Cache::memo()->get('key');
```

`memo` 方法可以選擇性地接受快取存放區（Store）的名稱，用以指定記憶化驅動程式要裝飾的底層快取存放區：

```php
// Using the default cache store...
$value = Cache::memo()->get('key');

// Using the Redis cache store...
$value = Cache::memo('redis')->get('key');
```

針對特定鍵（Key）的第一次 `get` 呼叫會從您的快取存放區中取得值，但在同一個請求或任務中，後續的呼叫將會直接從記憶體中取得該值：

```php
// Hits the cache...
$value = Cache::memo()->get('key');

// Does not hit the cache, returns memoized value...
$value = Cache::memo()->get('key');
```

當呼叫修改快取值的方法時（例如 `put`、`increment`、`remember` 等），記憶化快取會自動清除記憶體中的快取值，並將變更方法呼叫委託給底層的快取存放區：

```php
Cache::memo()->put('name', 'Taylor'); // Writes to underlying cache...
Cache::memo()->get('name');           // Hits underlying cache...
Cache::memo()->get('name');           // Memoized, does not hit cache...

Cache::memo()->put('name', 'Tim');    // Forgets memoized value, writes new value...
Cache::memo()->get('name');           // Hits underlying cache again...
```

<a name="the-cache-helper"></a>
### Cache 輔助函式

除了使用 `Cache` Facade 外，您還可以使用全域的 `cache` 函式來透過快取檢索與儲存資料。當呼叫 `cache` 函式並傳入單一字串引數時，它會返回該指定鍵的值：

```php
$value = cache('key');
```

如果您向該函式提供鍵值對陣列和過期時間，它會將這些值儲存在快取中並維持指定的持續時間：

```php
cache(['key' => 'value'], $seconds);

cache(['key' => 'value'], now()->plus(minutes: 10));
```

當呼叫 `cache` 函式且不傳入任何引數時，它會返回一個 `Illuminate\Contracts\Cache\Factory` 實作的實例，讓您可以呼叫其他的快取方法：

```php
cache()->remember('users', $seconds, function () {
    return DB::table('users')->get();
});
```

> [!NOTE]
> 當測試對全域 `cache` 函式的呼叫時，您可以像 [測試 Facade](/docs/{{version}}/mocking#mocking-facades) 一樣使用 `Cache::shouldReceive` 方法。

<a name="cache-tags"></a>
## 快取標籤

> [!WARNING]
> 使用 `file`、`dynamodb`、`database` 或 `storage` 快取驅動程式時，不支援快取標籤。


<a name="storing-tagged-cache-items"></a>
### 儲存有標籤的快取項目

快取標籤允許您為快取中相關的項目建立標記，接著就能清除所有被分配該標籤的快取值。您可以透過傳入一個有序的標籤名稱陣列來存取被標記的快取。例如，讓我們存取一個被標記的快取，並將一個值 `put` 寫入快取中：

```php
use Illuminate\Support\Facades\Cache;

Cache::tags(['people', 'artists'])->put('John', $john, $seconds);
Cache::tags(['people', 'authors'])->put('Anne', $anne, $seconds);
```


<a name="accessing-tagged-cache-items"></a>
### 存取有標籤的快取項目

若未提供儲存時所使用的標籤，則無法存取透過標籤儲存的項目。要取得一個有標籤的快取項目，請將相同順序的標籤列表傳遞給 `tags` 方法，然後使用您想取得的鍵值呼叫 `get` 方法：

```php
$john = Cache::tags(['people', 'artists'])->get('John');

$anne = Cache::tags(['people', 'authors'])->get('Anne');
```


<a name="removing-tagged-cache-items"></a>
### 移除有標籤的快取項目

您可以清除被分配了某個標籤或標籤列表的所有項目。例如，以下程式碼將會移除所有標記為 `people`、`authors` 或兩者皆有的快取。因此，`Anne` 與 `John` 都會從快取中被移除：

```php
Cache::tags(['people', 'authors'])->flush();
```

相反地，下方的程式碼只會移除標記為 `authors` 的快取值，因此 `Anne` 會被移除，但 `John` 則不會：

```php
Cache::tags('authors')->flush();
```


<a name="atomic-locks"></a>
## 原子鎖

> [!WARNING]
> 要使用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動程式作為應用程式的預設快取驅動程式。此外，所有的伺服器都必須與同一個中央快取伺服器進行通訊。


<a name="managing-locks"></a>
### 管理鎖

原子鎖允許操作分散式鎖，而無需擔心競態條件（Race Conditions）。例如，[Laravel Cloud](https://cloud.laravel.com) 使用原子鎖來確保伺服器上一次只執行一個遠端任務。您可以使用 `Cache::lock` 方法來建立與管理鎖：

```php
use Illuminate\Support\Facades\Cache;

$lock = Cache::lock('foo', 10);

if ($lock->get()) {
    // Lock acquired for 10 seconds...

    $lock->release();
}
```

`get` 方法也接受一個閉包（Closure）。在閉包執行完畢後，Laravel 會自動釋放該鎖：

```php
Cache::lock('foo', 10)->get(function () {
    // Lock acquired for 10 seconds and automatically released...
});
```

如果您在請求鎖的當下無法取得它，您可以指示 Laravel 等待指定的秒數。如果無法在指定的時間限制內取得鎖，將會拋出 `Illuminate\Contracts\Cache\LockTimeoutException` 異常：

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

上述範例可以透過向 `block` 方法傳遞一個閉包（Closure）來簡化。當閉包傳遞給此方法時，Laravel 將會嘗試在指定的秒數內取得鎖，並在閉包執行完畢後自動釋放鎖：

```php
Cache::lock('foo', 10)->block(5, function () {
    // Lock acquired for 10 seconds after waiting a maximum of 5 seconds...
});
```


<a name="managing-locks-across-processes"></a>
### 跨行程管理鎖

有時候，您可能希望在某個行程中取得鎖，並在另一個行程中釋放它。例如，您可能在網頁請求期間取得了鎖，並希望在該請求所觸發的佇列任務（Queued Job）結束時釋放該鎖。在這種情境下，您應該將鎖的作用域 "owner token" 傳遞給佇列任務，以便該任務可以使用給定的令牌重新實例化該鎖。

在下方的範例中，如果成功取得鎖，我們將會發送一個佇列任務。此外，我們會透過鎖的 `owner` 方法將鎖的擁有者令牌傳遞給該佇列任務：

```php
$podcast = Podcast::find($id);

$lock = Cache::lock('processing', 120);

if ($lock->get()) {
    ProcessPodcast::dispatch($podcast, $lock->owner());
}
```

在我們應用程式的 `ProcessPodcast` 任務中，我們可以使用擁有者令牌來還原並釋放該鎖：

```php
Cache::restoreLock('processing', $this->owner)->release();
```

如果您想直接釋放鎖而不考慮其目前的擁有者，可以使用 `forceRelease` 方法：

```php
Cache::lock('processing')->forceRelease();
```


<a name="concurrency-limiting"></a>
### 併發限制

Laravel 的原子鎖功能還提供了一些限制閉包（Closures）併發執行的方法。當您希望在整個基礎架構中只允許一個執行中的實例時，請使用 `withoutOverlapping`：

```php
Cache::withoutOverlapping('foo', function () {
    // Lock acquired after waiting a maximum of 10 seconds...
});
```

預設情況下，該鎖會一直持有直到閉包執行完畢為止，且該方法最多會等待 10 秒來取得鎖。您可以透過額外的引數來自訂這些值：

```php
Cache::withoutOverlapping('foo', function () {
    // Lock acquired for 120 seconds after waiting a maximum of 5 seconds...
}, lockFor: 120, waitFor: 5);
```

如果無法在指定的等待時間內取得鎖，將會拋出 `Illuminate\Contracts\Cache\LockTimeoutException` 異常。

如果您想要控制平行處理（Parallelism），可以使用 `funnel` 方法來設定最大併發執行數。`funnel` 方法適用於任何支援鎖的快取驅動程式：

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

`funnel` 鍵值用於識別被限制的資源。`limit` 方法定義了最大併發執行數。`releaseAfter` 方法設定了一個安全逾時時間（以秒為單位），在取得的插槽（Slot）自動釋放之前。`block` 方法設定了要等待可用插槽的秒數。

如果您偏好透過異常處理逾時，而不是提供失敗時的閉包，您可以省略第二個閉包。如果無法在指定的等待時間內取得鎖，將會拋出 `Illuminate\Cache\Limiters\LimiterTimeoutException` 異常：

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
> `funnel` 方法要求快取儲存必須實作 `Illuminate\Contracts\Cache\LockProvider` 介面。如果您嘗試在不支援鎖的快取儲存上使用 `funnel`，將會拋出 `BadMethodCallException`。

<a name="cache-failover"></a>
## 快取容錯移轉

`failover` 快取驅動程式在與快取進行互動時提供了自動容錯移轉功能。如果 `failover` 儲存空間的主快取儲存空間因任何原因失效，Laravel 將會自動嘗試使用列表中下一個設定的儲存空間。這在快取可靠性至關重要的正式環境中，對於確保高可用性特別有用。

要設定容錯移轉快取儲存空間，請指定 `failover` 驅動程式並提供一個依序嘗試的儲存空間名稱陣列。預設情況下，Laravel 在應用程式的 `config/cache.php` 設定檔中包含了一個容錯移轉設定範例：

```php
'failover' => [
    'driver' => 'failover',
    'stores' => [
        'database',
        'array',
    ],
],
```

一旦您設定了使用 `failover` 驅動程式的儲存空間，您將需要於應用程式的 `.env` 檔案中將該容錯移轉儲存空間設為預設快取儲存空間，才能使用容錯移轉功能：

```ini
CACHE_STORE=failover
```

當快取儲存空間操作失敗且啟用了容錯移轉時，Laravel 將會發送 `Illuminate\Cache\Events\CacheFailedOver` 事件，讓您可以回報或記錄快取儲存空間失效的情況。


<a name="adding-custom-cache-drivers"></a>
## 新增自訂快取驅動程式


<a name="writing-the-driver"></a>
### 撰寫驅動程式

要建立自訂的快取驅動程式，我們首先需要實作 `Illuminate\Contracts\Cache\Store` [契約(Contracts)](/docs/{{version}}/contracts)。因此，MongoDB 快取的實作可能會像這樣：

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

我們只需要使用 MongoDB 連線來實作這些方法。關於如何實作這些方法的範例，可以參考 [Laravel 框架原始碼](https://github.com/laravel/framework) 中的 `Illuminate\Cache\MemcachedStore`。一旦實作完成，我們就可以透過呼叫 `Cache` Facade 的 `extend` 方法來完成自訂驅動程式的註冊：

```php
Cache::extend('mongo', function (Application $app) {
    return Cache::repository(new MongoStore);
});
```

> [!NOTE]
> 如果您在想該把自訂快取驅動程式的程式碼放在哪裡，您可以在 `app` 目錄下建立一個 `Extensions` 命名空間。不過，請記住 Laravel 並沒有硬性規定應用程式的架構，您可以根據自己的喜好自由組織應用程式。


<a name="registering-the-driver"></a>
### 註冊驅動程式

為了向 Laravel 註冊自訂快取驅動程式，我們將使用 `Cache` Facade 的 `extend` 方法。由於其他服務提供者可能會嘗試在其 `boot` 方法中讀取快取值，因此我們將在 `booting` 回呼中註冊我們的自訂驅動程式。藉由使用 `booting` 回呼，我們可以確保自訂驅動程式在應用程式的服務提供者的 `boot` 方法被呼叫前註冊，但在所有服務提供者的 `register` 方法被呼叫之後註冊。我們將在應用程式的 `App\Providers\AppServiceProvider` 類別的 `register` 方法中註冊此 `booting` 回呼：

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

傳入 `extend` 方法的第一個引數是驅動程式的名稱。這將對應到您在 `config/cache.php` 設定檔中的 `driver` 選項。第二個引數是應回傳 `Illuminate\Cache\Repository` 實例的閉包。該閉包將會接收一個 `$app` 實例，該實例是 [服務容器](/docs/{{version}}/container) 的實例。

一旦註冊了您的擴充功能，請將應用程式 `config/cache.php` 設定檔中的 `CACHE_STORE` 環境變數或 `default` 選項更新為您擴充功能的名稱。


<a name="events"></a>
## 事件

要在每次快取操作時執行程式碼，您可以監聽由快取發送的各種 [事件](/docs/{{version}}/events)：

<div class="overflow-auto">

| Event Name                                      |
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

為了提高效能，您可以透過在應用程式的 `config/cache.php` 設定檔中將特定快取儲存空間的 `events` 設定選項設為 `false`，以停用快取事件：

```php
'database' => [
    'driver' => 'database',
    // ...
    'events' => false,
],
```