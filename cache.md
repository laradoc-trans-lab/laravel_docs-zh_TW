# 快取

- [簡介](#introduction)
- [設定](#configuration)
    - [驅動程式前置需求](#driver-prerequisites)
- [快取用法](#cache-usage)
    - [獲取快取實例](#obtaining-a-cache-instance)
    - [從快取檢索項目](#retrieving-items-from-the-cache)
    - [在快取中儲存項目](#storing-items-in-the-cache)
    - [從快取中刪除項目](#removing-items-from-the-cache)
    - [快取記憶化](#cache-memoization)
    - [快取輔助函式](#the-cache-helper)
- [快取標籤](#cache-tags)
- [原子鎖](#atomic-locks)
    - [管理鎖](#managing-locks)
    - [跨程序管理鎖](#managing-locks-across-processes)
    - [鎖與函式調用](#locks-and-function-invocations)
- [快取容錯移轉](#cache-failover)
- [新增自定義快取驅動程式](#adding-custom-cache-drivers)
    - [編寫驅動程式](#writing-the-driver)
    - [註冊驅動程式](#registering-the-driver)
- [事件](#events)

<a name="introduction"></a>
## 簡介

應用程式執行的一些資料檢索或處理任務可能非常耗費 CPU 資源，或者需要幾秒鐘才能完成。在這種情況下，通常會將檢索到的資料快取一段時間，以便在後續對相同資料的請求中可以快速地獲取。快取的資料通常儲存在非常快速的資料存儲中，例如 [Memcached](https://memcached.org) 或 [Redis](https://redis.io)。

值得慶幸的是，Laravel 為各種快取後端提供了富有表現力且統一的 API，讓您可以利用它們極快的資料檢索速度，進而加速您的 Web 應用程式。


<a name="configuration"></a>
## 設定

應用程式的快取設定檔位於 `config/cache.php`。在此檔案中，您可以指定整個應用程式預設要使用的快取儲存空間 (Store)。Laravel 開箱即用地支援流行的快取後端，如 [Memcached](https://memcached.org)、[Redis](https://redis.io)、[DynamoDB](https://aws.amazon.com/dynamodb) 以及關聯式資料庫。此外，還提供了一個基於檔案的快取驅動程式，而 `array` 與 `null` 快取驅動程式則為您的自動化測試提供了便利的快取後端。

快取設定檔還包含各種您可以查看的其他選項。預設情況下，Laravel 被設定為使用 `database` 快取驅動程式，它將序列化後的快取物件儲存在您的應用程式資料庫中。


<a name="driver-prerequisites"></a>
### 驅動程式前置需求


<a name="prerequisites-database"></a>
#### 資料庫

當使用 `database` 快取驅動程式時，您需要一個資料庫資料表來存放快取資料。通常，這已包含在 Laravel 預設的 `0001_01_01_000001_create_cache_table.php` [資料庫遷移 (Database Migration)](/docs/{{version}}/migrations) 中；然而，如果您的應用程式不包含此遷移，您可以使用 `make:cache-table` Artisan 指令來建立它：

```shell
php artisan make:cache-table

php artisan migrate
```


<a name="memcached"></a>
#### Memcached

使用 Memcached 驅動程式需要安裝 [Memcached PECL 套件](https://pecl.php.net/package/memcached)。您可以在 `config/cache.php` 設定檔中列出您所有的 Memcached 伺服器。此檔案已經包含一個 `memcached.servers` 項目讓您開始使用：

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

如果需要，您可以將 `host` 選項設定為 UNIX Socket 路徑。如果您這樣做，`port` 選項應設定為 `0`：

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

在 Laravel 使用 Redis 快取之前，您需要透過 PECL 安裝 PhpRedis PHP 擴充功能，或是透過 Composer 安裝 `predis/predis` 套件 (~2.0)。[Laravel Sail](/docs/{{version}}/sail) 已經包含了此擴充功能。此外，官方的 Laravel 應用程式平台，如 [Laravel Cloud](https://cloud.laravel.com) 與 [Laravel Forge](https://forge.laravel.com)，預設都已安裝 PhpRedis 擴充功能。

有關設定 Redis 的更多資訊，請參閱其 [Laravel 文件頁面](/docs/{{version}}/redis#configuration)。


<a name="dynamodb"></a>
#### DynamoDB

在使用 [DynamoDB](https://aws.amazon.com/dynamodb) 快取驅動程式之前，您必須建立一個 DynamoDB 資料表來儲存所有快取資料。通常，此資料表應命名為 `cache`。然而，您應該根據 `cache` 設定檔中 `stores.dynamodb.table` 設定值來命名該資料表。資料表名稱也可以透過 `DYNAMODB_CACHE_TABLE` 環境變數來設定。

此資料表還應該有一個字串型別的分區鍵 (Partition Key)，其名稱對應於應用程式 `cache` 設定檔中的 `stores.dynamodb.attributes.key` 設定項目。預設情況下，分區鍵應命名為 `key`。

通常，DynamoDB 不會主動從資料表中刪除過期的項目。因此，您應該在資料表上[啟用存活時間 (TTL)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)。在設定資料表的 TTL 設定時，您應該將 TTL 屬性名稱設定為 `expires_at`。

接下來，安裝 AWS SDK，以便您的 Laravel 應用程式可以與 DynamoDB 進行通訊：

```shell
composer require aws/aws-sdk-php
```

此外，您應該確保為 DynamoDB 快取儲存空間的設定選項提供了數值。通常這些選項（如 `AWS_ACCESS_KEY_ID` 和 `AWS_SECRET_ACCESS_KEY`）應定義在應用程式的 `.env` 設定檔中：

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

如果您正在使用 MongoDB，官方的 `mongodb/laravel-mongodb` 套件提供了一個 `mongodb` 快取驅動程式，並可以使用 `mongodb` 資料庫連線進行設定。MongoDB 支援 TTL 索引，可用於自動清除過期的快取項目。

有關設定 MongoDB 的更多資訊，請參閱 MongoDB 的[快取與鎖 (Cache and Locks) 文件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/cache/)。

<a name="cache-usage"></a>
## 快取用法


<a name="obtaining-a-cache-instance"></a>
### 獲取快取實例

要獲取快取存放區實例，您可以使用 `Cache` Facade，這也是我們在整份文件中所使用的。`Cache` Facade 為 Laravel 快取契約的底層實作提供了方便、簡潔的存取方式：

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
#### 存取多個快取存放區

使用 `Cache` Facade，您可以透過 `store` 方法存取各種快取存放區。傳遞給 `store` 方法的鍵 (Key) 應對應到您的 `cache` 設定檔中 `stores` 設定陣列裡的其中一個存放區：

```php
$value = Cache::store('file')->get('foo');

Cache::store('redis')->put('bar', 'baz', 600); // 10 Minutes
```


<a name="retrieving-items-from-the-cache"></a>
### 從快取檢索項目

`Cache` Facade 的 `get` 方法用於從快取中檢索項目。如果該項目在快取中不存在，則會回傳 `null`。如果您願意，可以向 `get` 方法傳遞第二個參數，指定在項目不存在時希望回傳的預設值：

```php
$value = Cache::get('key');

$value = Cache::get('key', 'default');
```

您甚至可以傳遞一個閉包 (Closure) 作為預設值。如果指定的項目在快取中不存在，則會回傳閉包的執行結果。傳遞閉包讓您可以延後從資料庫或其他外部服務獲取預設值的時間：

```php
$value = Cache::get('key', function () {
    return DB::table(/* ... */)->get();
});
```


<a name="determining-item-existence"></a>
#### 確定項目是否存在

`has` 方法可用於確定項目是否存在於快取中。如果項目存在但其值為 `null`，此方法也會回傳 `false`：

```php
if (Cache::has('key')) {
    // ...
}
```


<a name="incrementing-decrementing-values"></a>
#### 遞增 / 遞減數值

`increment` 和 `decrement` 方法可用於調整快取中整數項目的值。這兩個方法都接受選用的第二個參數，用來指示增加或減少該項目值的量：

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

有時您可能希望從快取中檢索一個項目，但若要求的項目不存在時也儲存一個預設值。例如，您可能希望從快取中檢索所有使用者，如果快取中沒有，則從資料庫中檢索並將其加入快取。您可以使用 `Cache::remember` 方法來達成此目的：

```php
$value = Cache::remember('users', $seconds, function () {
    return DB::table('users')->get();
});
```

如果該項目不存在於快取中，則會執行傳遞給 `remember` 方法的閉包，並將其結果放入快取中。

您可以使用 `rememberForever` 方法從快取中檢索項目，或者在項目不存在時將其永久儲存：

```php
$value = Cache::rememberForever('users', function () {
    return DB::table('users')->get();
});
```


<a name="swr"></a>
#### 尚可提供過期資料並背景更新

當使用 `Cache::remember` 方法時，如果快取值已過期，某些使用者可能會遇到回應時間緩慢的問題。對於某些類型的資料，允許在背景重新計算快取值的同時提供部分過期的資料會很有幫助，這可以防止部分使用者在計算快取值時遇到回應緩慢的問題。這通常被稱為「stale-while-revalidate」模式，而 `Cache::flexible` 方法提供了該模式的實作。

flexible 方法接受一個陣列，該陣列指定快取值被視為「新鮮 (Fresh)」的時間，以及它何時變得「過期 (Stale)」。陣列中的第一個值代表快取被視為新鮮的秒數，而第二個值定義了在必須重新計算之前，它可以作為過期資料提供的時間長度。

如果在新鮮期內（第一個值之前）提出請求，則會立即回傳快取而不進行重新計算。如果在過期期間內（兩個值之間）提出請求，則會將過期值提供給使用者，並註冊一個 [延遲函式](/docs/{{version}}/helpers#deferred-functions) 在將回應傳送給使用者後更新快取值。如果在第二個值之後提出請求，則快取被視為已失效，並立即重新計算該值，這可能會導致使用者的回應較慢：

```php
$value = Cache::flexible('users', [5, 10], function () {
    return DB::table('users')->get();
});
```


<a name="retrieve-delete"></a>
#### 檢索並刪除

如果您需要從快取中檢索一個項目後隨即刪除該項目，您可以使用 `pull` 方法。與 `get` 方法一樣，如果項目在快取中不存在，則會回傳 `null`：

```php
$value = Cache::pull('key');

$value = Cache::pull('key', 'default');
```


<a name="storing-items-in-the-cache"></a>
### 在快取中儲存項目

您可以使用 `Cache` Facade 上的 `put` 方法將項目儲存在快取中：

```php
Cache::put('key', 'value', $seconds = 10);
```

如果未將儲存時間傳遞給 `put` 方法，則該項目將永久儲存：

```php
Cache::put('key', 'value');
```

除了以整數形式傳遞秒數外，您也可以傳遞一個代表快取項目所需過期時間的 `DateTime` 實例：

```php
Cache::put('key', 'value', now()->plus(minutes: 10));
```


<a name="store-if-not-present"></a>
#### 若不存在則儲存

`add` 方法僅在項目尚不存在於快取存放區時，才會將項目加入快取。如果項目確實被加入快取，該方法將回傳 `true`。否則，該方法將回傳 `false`。`add` 方法是一個原子操作：

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
### 從快取中刪除項目

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
> 清除快取 (Flush) 不會考慮您設定的快取「前綴 (Prefix)」，並且會移除快取中的所有項目。在清除與其他應用程式共享的快取時，請務必謹慎考慮。

<a name="cache-memoization"></a>
### 快取記憶化

Laravel 的 `memo` 快取驅動程式允許你在單次請求或任務執行期間，將解析後的快取值暫時儲存在記憶體中。這可以防止在同一次執行中重複命中快取，從而顯著提高效能。

要使用記憶化快取，請調用 `memo` 方法：

```php
use Illuminate\Support\Facades\Cache;

$value = Cache::memo()->get('key');
```

`memo` 方法可以選擇性地接受快取儲存空間的名稱，這指定了記憶化驅動程式將要裝飾的底層快取儲存空間：

```php
// Using the default cache store...
$value = Cache::memo()->get('key');

// Using the Redis cache store...
$value = Cache::memo('redis')->get('key');
```

針對給定鍵的第一次 `get` 調用會從你的快取儲存空間中檢索值，但在同一個請求或任務中的後續調用則會從記憶體中檢索該值：

```php
// Hits the cache...
$value = Cache::memo()->get('key');

// Does not hit the cache, returns memoized value...
$value = Cache::memo()->get('key');
```

當調用修改快取值的方法（例如 `put`、`increment`、`remember` 等）時，記憶化快取會自動捨棄記憶化的值，並將變更方法調用委託給底層快取儲存空間：

```php
Cache::memo()->put('name', 'Taylor'); // Writes to underlying cache...
Cache::memo()->get('name');           // Hits underlying cache...
Cache::memo()->get('name');           // Memoized, does not hit cache...

Cache::memo()->put('name', 'Tim');    // Forgets memoized value, writes new value...
Cache::memo()->get('name');           // Hits underlying cache again...
```


<a name="the-cache-helper"></a>
### 快取輔助函式

除了使用 `Cache` Facade 以外，你也可以使用全域的 `cache` 函式來透過快取檢索與儲存資料。當調用 `cache` 函式並傳入單個字串參數時，它將回傳給定鍵的值：

```php
$value = cache('key');
```

如果你向該函式提供鍵值對陣列與過期時間，它將在快取中儲存這些值一段指定的持續時間：

```php
cache(['key' => 'value'], $seconds);

cache(['key' => 'value'], now()->plus(minutes: 10));
```

當調用 `cache` 函式且不帶任何參數時，它會回傳 `Illuminate\Contracts\Cache\Factory` 實作的實例，讓你可以調用其他快取方法：

```php
cache()->remember('users', $seconds, function () {
    return DB::table('users')->get();
});
```

> [!NOTE]
> 當測試全域 `cache` 函式的調用時，你可以使用 `Cache::shouldReceive` 方法，就像你在[測試 facade](/docs/{{version}}/mocking#mocking-facades) 時一樣。

<a name="cache-tags"></a>
## 快取標籤

> [!WARNING]
> 在使用 `file`、`dynamodb` 或 `database` 快取驅動程式時，不支援快取標籤。


<a name="storing-tagged-cache-items"></a>
### 儲存標籤快取項目

快取標籤允許您為快取中的相關項目加上標籤，然後清除所有已分配該標籤的快取值。您可以透過傳入標籤名稱的有序陣列來存取標籤快取。例如，讓我們存取一個標籤快取並將一個值 `put` 到快取中：

```php
use Illuminate\Support\Facades\Cache;

Cache::tags(['people', 'artists'])->put('John', $john, $seconds);
Cache::tags(['people', 'authors'])->put('Anne', $anne, $seconds);
```


<a name="accessing-tagged-cache-items"></a>
### 存取標籤快取項目

若未提供儲存該值時所使用的標籤，則無法存取透過標籤儲存的項目。要檢索標籤快取項目，請將相同的有序標籤列表傳遞給 `tags` 方法，然後使用您要檢索的鍵呼叫 `get` 方法：

```php
$john = Cache::tags(['people', 'artists'])->get('John');

$anne = Cache::tags(['people', 'authors'])->get('Anne');
```


<a name="removing-tagged-cache-items"></a>
### 移除標籤快取項目

您可以清除分配了標籤或標籤列表的所有項目。例如，以下程式碼將移除標記為 `people`、`authors` 或兩者兼有的所有快取。因此，`Anne` 和 `John` 都會從快取中被移除：

```php
Cache::tags(['people', 'authors'])->flush();
```

相比之下，下方的程式碼只會移除標記為 `authors` 的快取值，因此 `Anne` 會被移除，但 `John` 不會：

```php
Cache::tags('authors')->flush();
```


<a name="atomic-locks"></a>
## 原子鎖

> [!WARNING]
> 要利用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動程式作為應用程式的預設快取驅動程式。此外，所有伺服器必須與同一個中央快取伺服器進行通訊。


<a name="managing-locks"></a>
### 管理鎖

原子鎖允許在不擔心競態條件 (Race Conditions) 的情況下操作分散式鎖。例如，[Laravel Cloud](https://cloud.laravel.com) 使用原子鎖來確保伺服器上一次只執行一個遠端任務。您可以使用 `Cache::lock` 方法來建立與管理鎖：

```php
use Illuminate\Support\Facades\Cache;

$lock = Cache::lock('foo', 10);

if ($lock->get()) {
    // Lock acquired for 10 seconds...

    $lock->release();
}
```

`get` 方法也接受一個閉包 (Closure)。在閉包執行後，Laravel 會自動釋放鎖：

```php
Cache::lock('foo', 10)->get(function () {
    // Lock acquired for 10 seconds and automatically released...
});
```

如果您請求時鎖不可用，您可以指示 Laravel 等待指定的秒數。如果無法在指定的時間限制內取得鎖，則會拋出 `Illuminate\Contracts\Cache\LockTimeoutException`：

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

上面的範例可以透過向 `block` 方法傳遞一個閉包來簡化。當閉包被傳遞給此方法時，Laravel 會嘗試在指定的秒數內取得鎖，並在閉包執行完畢後自動釋放鎖：

```php
Cache::lock('foo', 10)->block(5, function () {
    // Lock acquired for 10 seconds after waiting a maximum of 5 seconds...
});
```


<a name="managing-locks-across-processes"></a>
### 跨程序管理鎖

有時，您可能希望在一個程序中取得鎖，並在另一個程序中釋放它。例如，您可能在 Web 請求期間取得鎖，並希望在該請求觸發的佇列任務 (Queued Job) 結束時釋放鎖。在這種情況下，您應該將鎖的範圍「所有者權杖 (Owner Token)」傳遞給佇列任務，以便該任務可以使用給定的權杖重新實例化鎖。

在下面的範例中，如果成功取得鎖，我們將發送一個佇列任務。此外，我們將透過鎖的 `owner` 方法將鎖的所有者權杖傳遞給佇列任務：

```php
$podcast = Podcast::find($id);

$lock = Cache::lock('processing', 120);

if ($lock->get()) {
    ProcessPodcast::dispatch($podcast, $lock->owner());
}
```

在我們應用程式的 `ProcessPodcast` 任務中，我們可以使用所有者權杖還原並釋放鎖：

```php
Cache::restoreLock('processing', $this->owner)->release();
```

如果您想在不考慮當前所有者的情況下釋放鎖，可以使用 `forceRelease` 方法：

```php
Cache::lock('processing')->forceRelease();
```


<a name="locks-and-function-invocations"></a>
### 鎖與函式調用

`withoutOverlapping` 方法提供了一種簡單的語法，用於在持有原子鎖的同時執行給定的閉包，從而確保在整個基礎架構中，一次只有一個閉包實例在運行：

```php
Cache::withoutOverlapping('foo', function () {
    // Lock acquired after waiting a maximum of 10 seconds...
});
```

預設情況下，直到閉包完成執行後鎖才會釋放，且該方法會等待最多 10 秒以取得鎖。您可以透過向該方法傳遞額外參數來自定義這些值：

```php
Cache::withoutOverlapping('foo', function () {
    // Lock acquired for 120 seconds after waiting a maximum of 5 seconds...
}, lockFor: 120, waitFor: 5);
```

如果無法在指定的等待時間內取得鎖，則會拋出 `Illuminate\Contracts\Cache\LockTimeoutException`。


<a name="cache-failover"></a>
## 快取容錯移轉

`failover` 快取驅動程式在與快取互動時提供自動容錯移轉功能。如果 `failover` 儲存庫的主要快取儲存因任何原因失敗，Laravel 將自動嘗試使用列表中下一個設定的儲存庫。這對於確保在快取可靠性至關重要的正式環境中的高可用性特別有用。

要設定容錯移轉快取儲存庫，請指定 `failover` 驅動程式並提供要按順序嘗試的儲存庫名稱陣列。預設情況下，Laravel 在您的應用程式 `config/cache.php` 設定檔中包含了一個範例容錯移轉設定：

```php
'failover' => [
    'driver' => 'failover',
    'stores' => [
        'database',
        'array',
    ],
],
```

設定好使用 `failover` 驅動程式的儲存庫後，您需要將您的應用程式 `.env` 檔案中的預設快取儲存庫設置為此容錯移轉儲存庫，以便使用容錯移轉功能：

```ini
CACHE_STORE=failover
```

當快取儲存操作失敗且啟動容錯移轉時，Laravel 會發送 `Illuminate\Cache\Events\CacheFailedOver` 事件，讓您可以報告或記錄某個快取儲存庫已失敗。

<a name="adding-custom-cache-drivers"></a>
## 新增自定義快取驅動程式

<a name="writing-the-driver"></a>
### 編寫驅動程式

要建立自定義的快取驅動程式，我們首先需要實作 `Illuminate\Contracts\Cache\Store` [契約](/docs/{{version}}/contracts)。因此，一個 MongoDB 的快取實作可能看起來像這樣：

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

我們只需要使用 MongoDB 連線來實作這些方法。關於如何實作這些方法的範例，請參閱 [Laravel 框架原始碼](https://github.com/laravel/framework) 中的 `Illuminate\Cache\MemcachedStore`。一旦我們的實作完成，我們可以藉由呼叫 `Cache` Facade 的 `extend` 方法來完成自定義驅動程式的註冊：

```php
Cache::extend('mongo', function (Application $app) {
    return Cache::repository(new MongoStore);
});
```

> [!NOTE]
> 如果您想知道將自定義快取驅動程式程式碼放在哪裡，您可以在 `app` 目錄中建立一個 `Extensions` 命名空間。但是，請記住 Laravel 並沒有嚴格的應用程式結構，您可以根據自己的喜好自由組織應用程式。

<a name="registering-the-driver"></a>
### 註冊驅動程式

要向 Laravel 註冊自定義快取驅動程式，我們將使用 `Cache` Facade 上的 `extend` 方法。由於其他服務提供者可能會嘗試在其 `boot` 方法中讀取快取值，我們將在 `booting` 回呼中註冊我們的自定義驅動程式。透過使用 `booting` 回呼，我們可以確保自定義驅動程式在應用程式服務提供者的 `boot` 方法被呼叫之前註冊，但在所有服務提供者的 `register` 方法被呼叫之後。我們將在應用程式的 `App\Providers\AppServiceProvider` 類別的 `register` 方法中註冊我們的 `booting` 回呼：

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

傳遞給 `extend` 方法的第一個參數是驅動程式的名稱。這將對應到 `config/cache.php` 設定檔中的 `driver` 選項。第二個參數是一個應回傳 `Illuminate\Cache\Repository` 實例的閉包。該閉包將被傳遞一個 `$app` 實例，它是 [服務容器](/docs/{{version}}/container) 的實例。

一旦您的擴充功能註冊完成，請將應用程式 .env 檔案中的 `CACHE_STORE` 環境變數或 `config/cache.php` 設定檔中的 `default` 選項更新為您的擴充功能名稱。

<a name="events"></a>
## 事件

要對每個快取操作執行程式碼，您可以監聽由快取發出的各種 [事件](/docs/{{version}}/events)：

<div class="overflow-auto">

| 事件名稱                                         |
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

為了提高效能，您可以在應用程式的 `config/cache.php` 設定檔中，將特定快取存放區的 `events` 設定選項設置為 `false` 來停用快取事件：

```php
'database' => [
    'driver' => 'database',
    // ...
    'events' => false,
],
```