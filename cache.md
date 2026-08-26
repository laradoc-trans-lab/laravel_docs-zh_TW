# 快取 (Cache)

- [簡介](#introduction)
- [設定](#configuration)
    - [驅動程式前置準備](#driver-prerequisites)
- [快取用法](#cache-usage)
    - [取得快取實例](#obtaining-a-cache-instance)
    - [從快取讀取項目](#retrieving-items-from-the-cache)
    - [將項目寫入快取](#storing-items-in-the-cache)
    - [延長項目生命週期](#extending-item-lifetime)
    - [從快取移除項目](#removing-items-from-the-cache)
    - [快取 Memoization](#cache-memoization)
    - [Cache Helper](#the-cache-helper)
- [快取標籤](#cache-tags)
    - [儲存標籤化的快取項目](#storing-tagged-cache-items)
    - [存取標籤化的快取項目](#accessing-tagged-cache-items)
    - [移除標籤化的快取項目](#removing-tagged-cache-items)
- [原子鎖](#atomic-locks)
    - [管理鎖](#managing-locks)
    - [跨行程管理鎖](#managing-locks-across-processes)
    - [重新整理鎖](#refreshing-locks)
    - [並行限制](#concurrency-limiting)
- [快取故障轉移](#cache-failover)
- [新增自訂快取驅動程式](#adding-custom-cache-drivers)
    - [撰寫驅動程式](#writing-the-driver)
    - [註冊驅動程式](#registering-the-driver)
- [事件](#events)

<a name="introduction"></a>
## 簡介

應用程式執行的某些資料讀取或處理任務可能相當消耗 CPU 資源，或是需要幾秒鐘才能完成。在這種情況下，通常會將讀取到的資料快取一段時間，以便後續對相同資料的請求能快速取得。快取資料通常儲存在極快的資料儲存庫中，例如 [Memcached](https://memcached.org) 或 [Redis](https://redis.io)。

幸運的是，Laravel 為各種快取後端提供了直觀且統一的 API，讓你能夠利用其極速的資料讀取能力，藉此加快 Web 應用程式的執行速度。


<a name="configuration"></a>
## 設定

應用程式的快取設定檔位於 `config/cache.php`。在這個檔案中，你可以指定整個應用程式預設要使用哪一個快取儲存區 (Cache Store)。Laravel 開箱即支援許多熱門的快取後端，例如 [Memcached](https://memcached.org)、[Redis](https://redis.io)、[DynamoDB](https://aws.amazon.com/dynamodb)、關聯式資料庫以及檔案系統磁碟 (Filesystem Disks)。此外，還提供了基於檔案的快取驅動程式，而 `array` 與 `null` 快取驅動程式則為自動化測試提供了方便的快取後端。

快取設定檔還包含許多其他可供檢視的選項。預設情況下，Laravel 設定使用 `database` 快取驅動程式，它會將序列化後的快取物件儲存在應用程式的資料庫中。


<a name="driver-prerequisites"></a>
### 驅動程式前置準備


<a name="prerequisites-database"></a>
#### 資料庫

使用 `database` 快取驅動程式時，你需要一個資料庫表單來存放快取資料。通常，這已經包含在 Laravel 預設的 `0001_01_01_000001_create_cache_table.php` [資料庫遷移](/docs/{{version}}/migrations) 中；但是，如果你的應用程式不包含此遷移，可以使用 `make:cache-table` Artisan 指令來建立它：

```shell
php artisan make:cache-table

php artisan migrate
```


<a name="memcached"></a>
#### Memcached

使用 Memcached 驅動程式需要先安裝 [Memcached PECL 套件](https://pecl.php.net/package/memcached)。你可以在 `config/cache.php` 設定檔中列出所有的 Memcached 伺服器。這個檔案已經包含了一個 `memcached.servers` 項目供你快速上手：

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

如有需要，你可以將 `host` 選項設定為 UNIX socket 路徑。若這樣做，`port` 選項應設定為 `0`：

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

在 Laravel 使用 Redis 快取之前，你需要透過 PECL 安裝 PhpRedis PHP 擴充功能，或是透過 Composer 安裝 `predis/predis` 套件。[Laravel Sail](/docs/{{version}}/sail) 已經內建此擴充功能。此外，官方的 Laravel 應用程式平台，如 [Laravel Cloud](https://cloud.laravel.com) 與 [Laravel Forge](https://forge.laravel.com)，預設皆已安裝 PhpRedis 擴充功能。

關於設定 Redis 的更多資訊，請參閱其 [Laravel 文件頁面](/docs/{{version}}/redis#configuration)。


<a name="storage"></a>
#### Storage

`storage` 快取驅動程式允許你將快取值儲存在應用程式已設定的任何 [檔案系統磁碟](/docs/{{version}}/filesystem) 上。當你想使用現有的磁碟（例如 S3 磁碟）作為鍵 / 值 (Key / Value) 快取儲存區時，這會非常實用：

```php
'storage' => [
    'driver' => 'storage',
    'disk' => env('CACHE_STORAGE_DISK'),
    'path' => env('CACHE_STORAGE_PATH', 'framework/cache/data'),
],
```


<a name="dynamodb"></a>
#### DynamoDB

在使用 [DynamoDB](https://aws.amazon.com/dynamodb) 快取驅動程式之前，你必須建立一個 DynamoDB 資料表來儲存所有快取資料。通常，此資料表應命名為 `cache`。不過，你應該根據 `cache` 設定檔中 `stores.dynamodb.table` 設定值來命名該資料表。資料表名稱也可以透過 `DYNAMODB_CACHE_TABLE` 環境變數來設定。

此資料表還應具有一個字串型別的分區鍵 (Partition Key)，其名稱需對應至應用程式 `cache` 設定檔中的 `stores.dynamodb.attributes.key` 設定項目。預設情況下，分區鍵的名稱應為 `key`。

通常，DynamoDB 不會主動從資料表中刪除過期項目。因此，你應該在資料表上 [啟用生存時間 (TTL)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)。設定資料表的 TTL 時，應將 TTL 屬性名稱指定為 `expires_at`。

接下來，安裝 AWS SDK 以便你的 Laravel 應用程式能與 DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

此外，你還應確保已為 DynamoDB 快取儲存區選項提供設定值。通常這些選項（例如 `AWS_ACCESS_KEY_ID` 和 `AWS_SECRET_ACCESS_KEY`）應定義在應用程式的 `.env` 設定檔中：

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

如果你使用的是 MongoDB，官方的 `mongodb/laravel-mongodb` 套件提供了 `mongodb` 快取驅動程式，且可以使用 `mongodb` 資料庫連線進行設定。MongoDB 支援 TTL 索引，可用於自動清理過期的快取項目。

關於設定 MongoDB 的更多資訊，請參閱 MongoDB 的 [Cache and Locks 文件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/cache/)。

<a name="cache-usage"></a>
## 快取用法


<a name="obtaining-a-cache-instance"></a>
### 取得快取實例

若要取得快取商店實例，您可以使用 `Cache` Facade，這也是我們在整個文件中所使用的方式。`Cache` Facade 提供了方便且精簡的存取介面，用來存取 Laravel 快取契約(Contracts)的底層實作：

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
#### 存取多個快取商店

使用 `Cache` Facade，您可透過 `store` 方法存取各種快取商店。傳遞給 `store` 方法的鍵名應該對應到 `cache` 設定檔中 `stores` 設定陣列裡列出的其中一個商店：

```php
$value = Cache::store('file')->get('foo');

Cache::store('redis')->put('bar', 'baz', 600); // 10 Minutes
```


<a name="retrieving-items-from-the-cache"></a>
### 從快取讀取項目

`Cache` Facade 的 `get` 方法用於從快取中擷取項目。若該項目不存在於快取中，將會傳回 `null`。若有需要，您可以傳遞第二個引數給 `get` 方法，指定項目不存在時所要傳回的預設值：

```php
$value = Cache::get('key');

$value = Cache::get('key', 'default');
```

您甚至可以傳遞閉包作為預設值。若指定的項目不存在於快取中，將會傳回閉包的執行結果。傳遞閉包可讓您延後從資料庫或其他外部服務讀取預設值的時機：

```php
$value = Cache::get('key', function () {
    return DB::table(/* ... */)->get();
});
```


<a name="determining-item-existence"></a>
#### 確認項目是否存在

`has` 方法可以用來確認項目是否存在於快取中。若項目存在但其值為 `null`，該方法也會傳回 `false`：

```php
if (Cache::has('key')) {
    // ...
}
```


<a name="incrementing-decrementing-values"></a>
#### 數值遞增與遞減

`increment` 與 `decrement` 方法可以用來調整快取中整數項目的數值。這兩個方法皆可接受選擇性的第二個引數，用以指定該項目的數值要遞增或遞減多少：

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
#### 讀取並儲存

有時您可能希望從快取中讀取一個項目，但若請求的項目不存在時，同時也儲存一個預設值。例如，您可能想要從快取中讀取所有使用者，若他們不存在，則從資料庫擷取並將其新增至快取中。您可以使用 `Cache::remember` 方法來達成此目的：

```php
$value = Cache::remember('users', $seconds, function () {
    return DB::table('users')->get();
});
```

若項目不存在於快取中，傳遞給 `remember` 方法的閉包將會被執行，且其結果會被放置於快取中。

如果您需要知道該項目是從快取讀取出來的，還是透過執行給定的閉包解析出來的，您可以使用 `rememberWithWarmth` 方法。該方法會傳回包含快取值與布林值的陣列，該布林值指示該項目是否為「溫熱（warm）」狀態，意即它是從快取讀取而非由閉包解析：

```php
[$value, $warm] = Cache::rememberWithWarmth('users', $seconds, function () {
    return DB::table('users')->get();
});
```

您可以使用 `rememberForever` 方法從快取中讀取項目，或者若項目不存在時將其永久儲存：

```php
$value = Cache::rememberForever('users', function () {
    return DB::table('users')->get();
});
```


<a name="swr"></a>
#### 過期重新驗證 (Stale While Revalidate)

當使用 `Cache::remember` 方法時，如果快取值已經過期，某些使用者可能會遇到回應速度較慢的情況。對於特定類型的資料，在背景重新計算快取值的同時，允許提供部分過期的資料會非常有用，可避免使用者在計算快取值時遭受較慢的回應。這通常被稱為「stale-while-revalidate」（過期重新驗證）模式，而 `Cache::flexible` 方法便提供了該模式的實作。

flexible 方法接受一個陣列，用來指定快取值被視為「有效（fresh）」的時間長度，以及何時變為「過期（stale）」。陣列中的第一個值代表快取被視為有效的秒數，而第二個值定義在需要重新計算前，過期資料可以提供服務的時間長度。

如果在有效期限內（第一個值之前）提出請求，快取會立即傳回而不需重新計算。如果在過期期間（兩個值之間）提出請求，過期的值會提供給使用者，並且會註冊一個[延遲函式(deferred function)](/docs/{{version}}/helpers#deferred-functions)，在回應發送給使用者後刷新快取值。如果在第二個值之後提出請求，快取會被視為過期，且會立即重新計算數值，這可能會導致使用者接收回應的時間較慢：

```php
$value = Cache::flexible('users', [5, 10], function () {
    return DB::table('users')->get();
});
```


<a name="retrieve-delete"></a>
#### 讀取並刪除

若您需要從快取中讀取項目並隨後將該項目刪除，可以使用 `pull` 方法。就像 `get` 方法一樣，若項目不存在於快取中，將會傳回 `null`：

```php
$value = Cache::pull('key');

$value = Cache::pull('key', 'default');
```


<a name="storing-items-in-the-cache"></a>
### 將項目寫入快取

您可以使用 `Cache` Facade 上的 `put` 方法將項目寫入快取：

```php
Cache::put('key', 'value', $seconds = 10);
```

若未傳遞儲存時間給 `put` 方法，該項目將被永久儲存：

```php
Cache::put('key', 'value');
```

除了傳遞整數的秒數之外，您也可以傳遞代表快取項目預期過期時間的 `DateTime` 實例：

```php
Cache::put('key', 'value', now()->plus(minutes: 10));
```


<a name="store-if-not-present"></a>
#### 當不存在時才儲存

`add` 方法僅會在項目尚未存在於快取商店時，才將項目新增至快取。如果項目成功新增至快取，該方法將傳回 `true`。否則，該方法將傳回 `false`。`add` 方法是一個原子操作：

```php
Cache::add('key', 'value', $seconds);
```


<a name="extending-item-lifetime"></a>
### 延長項目生命週期

`touch` 方法允許您延長既有快取項目的生命週期（TTL）。若快取項目存在且其過期時間成功延長，`touch` 方法將傳回 `true`。若該項目不存在於快取中，該方法將傳回 `false`：

```php
Cache::touch('key', 3600);
```

您可以提供 `DateTimeInterface`、`DateInterval` 或 `Carbon` 實例來指定精確的過期時間：

```php
Cache::touch('key', now()->addHours(2));
```


<a name="storing-items-forever"></a>
#### 永久儲存項目

`forever` 方法可用於將項目永久儲存在快取中。由於這些項目不會過期，因此必須使用 `forget` 方法手動從快取中移除：

```php
Cache::forever('key', 'value');
```

> [!NOTE]
> 若您使用的是 Memcached 驅動程式，當快取達到其容量限制時，即使是「永久」儲存的項目也可能會被移除。

<a name="removing-items-from-the-cache"></a>
### 從快取移除項目

您可以使用 `forget` 方法從快取中移除項目：

```php
Cache::forget('key');
```

您也可以透過提供 0 或負數的過期秒數來移除項目：

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
> 清除快取時並不會遵守您所設定的快取 "prefix"，並且會移除快取中的所有項目。當您要清除與其他應用程式共用的快取時，請務必謹慎考慮。


<a name="cache-memoization"></a>
### 快取 Memoization

Laravel 的 `memo` 快取驅動程式允許您在單一請求或 Job 執行期間，將解析後的快取值暫存於記憶體中。這能避免在同一次執行中重複讀取快取，從而大幅提升效能。

若要使用 Memoized 快取，請呼叫 `memo` 方法：

```php
use Illuminate\Support\Facades\Cache;

$value = Cache::memo()->get('key');
```

`memo` 方法可以選擇性地傳入快取儲存庫的名稱，用於指定 Memoized 驅動程式所要包裝的底層快取儲存庫：

```php
// Using the default cache store...
$value = Cache::memo()->get('key');

// Using the Redis cache store...
$value = Cache::memo('redis')->get('key');
```

針對特定鍵名的第一個 `get` 呼叫會從您的快取儲存庫中取得數值，但在同一個請求或 Job 中的後續呼叫則會直接從記憶體中取得數值：

```php
// Hits the cache...
$value = Cache::memo()->get('key');

// Does not hit the cache, returns memoized value...
$value = Cache::memo()->get('key');
```

當呼叫會修改快取值的方法（例如 `put`、`increment`、`remember` 等）時，Memoized 快取會自動忘記已暫存的值，並將修改狀態的方法呼叫轉交給底層的快取儲存庫處理：

```php
Cache::memo()->put('name', 'Taylor'); // Writes to underlying cache...
Cache::memo()->get('name');           // Hits underlying cache...
Cache::memo()->get('name');           // Memoized, does not hit cache...

Cache::memo()->put('name', 'Tim');    // Forgets memoized value, writes new value...
Cache::memo()->get('name');           // Hits underlying cache again...
```


<a name="the-cache-helper"></a>
### Cache Helper

除了使用 `Cache` Facade 之外，您也可以使用全域的 `cache` 函式來透過快取讀取與儲存資料。當傳入單一字串引數呼叫 `cache` 函式時，它將回傳該給定鍵名的值：

```php
$value = cache('key');
```

如果您向該函式提供鍵／值對陣列以及過期時間，它將在指定的時長內將數值儲存於快取中：

```php
cache(['key' => 'value'], $seconds);

cache(['key' => 'value'], now()->plus(minutes: 10));
```

當呼叫 `cache` 函式而不傳入任何引數時，它會回傳 `Illuminate\Contracts\Cache\Factory` 實作的實例，讓您可以呼叫其他快取方法：

```php
cache()->remember('users', $seconds, function () {
    return DB::table('users')->get();
});
```

> [!NOTE]
> 當測試對全域 `cache` 函式的呼叫時，您可以如同[測試 Facade](/docs/{{version}}/mocking#mocking-facades) 一樣使用 `Cache::shouldReceive` 方法。

<a name="cache-tags"></a>
## 快取標籤

> [!WARNING]
> 使用 `file`、`dynamodb`、`database` 或 `storage` 快取驅動程式時，不支援快取標籤。


<a name="storing-tagged-cache-items"></a>
### 儲存標籤化的快取項目

快取標籤允許您對快取中的相關項目打上標籤，然後清除所有被指派該標籤的快取值。您可以透過傳入標籤名稱的有序陣列來存取標籤化的快取。例如，讓我們存取標籤化的快取並將一個值 `put` 到快取中：

```php
use Illuminate\Support\Facades\Cache;

Cache::tags(['people', 'artists'])->put('John', $john, $seconds);
Cache::tags(['people', 'authors'])->put('Anne', $anne, $seconds);
```


<a name="accessing-tagged-cache-items"></a>
### 存取標籤化的快取項目

透過標籤儲存的項目，若沒有提供當時儲存該值所使用的標籤，就無法進行存取。若要讀取標籤化的快取項目，請將相同順序的標籤列表傳遞給 `tags` 方法，然後使用您想讀取的鍵名來呼叫 `get` 方法：

```php
$john = Cache::tags(['people', 'artists'])->get('John');

$anne = Cache::tags(['people', 'authors'])->get('Anne');
```


<a name="removing-tagged-cache-items"></a>
### 移除標籤化的快取項目

您可以清除所有被指派了某個標籤或標籤列表的項目。例如，以下程式碼會移除所有標有 `people`、`authors` 或兩者皆有的快取。因此，`Anne` 和 `John` 都會從快取中被移除：

```php
Cache::tags(['people', 'authors'])->flush();
```

相對地，下方程式碼將僅移除標有 `authors` 的快取值，因此 `Anne` 會被移除，但 `John` 不會：

```php
Cache::tags('authors')->flush();
```

<a name="atomic-locks"></a>
## 原子鎖

> [!WARNING]
> 若要使用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動程式作為應用程式的預設快取驅動程式。此外，所有伺服器都必須與同一個中央快取伺服器進行通訊。


<a name="managing-locks"></a>
### 管理鎖

原子鎖允許操作分散式鎖，而無需擔心競態條件 (Race conditions)。例如，[Laravel Cloud](https://cloud.laravel.com) 使用原子鎖來確保同一時間伺服器上只會執行一個遠端任務。您可以使用 `Cache::lock` 方法建立與管理鎖：

```php
use Illuminate\Support\Facades\Cache;

$lock = Cache::lock('foo', 10);

if ($lock->get()) {
    // Lock acquired for 10 seconds...

    $lock->release();
}
```

`get` 方法也接受閉包。在閉包執行完畢後，Laravel 會自動釋放鎖：

```php
Cache::lock('foo', 10)->get(function () {
    // Lock acquired for 10 seconds and automatically released...
});
```

若在您請求鎖時無法立即取得鎖，您可以指示 Laravel 等待指定的秒數。若無法在指定的時間限制內取得鎖，將會拋出 `Illuminate\Contracts\Cache\LockTimeoutException`：

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

上述範例可以透過將閉包傳遞給 `block` 方法來進行簡化。當將閉包傳遞給此方法時，Laravel 會嘗試在指定的秒數內取得鎖，並會在閉包執行完成後自動釋放鎖：

```php
Cache::lock('foo', 10)->block(5, function () {
    // Lock acquired for 10 seconds after waiting a maximum of 5 seconds...
});
```


<a name="managing-locks-across-processes"></a>
### 跨行程管理鎖

有時，您可能希望在一個行程 (Process) 中取得鎖，並在另一個行程中釋放該鎖。例如，您可能在 Web 請求期間取得鎖，並希望在該請求所觸發的佇列任務結束時釋放鎖。在此情境下，您應該將該鎖的作用域「擁有者令牌 (owner token)」傳遞給佇列任務，以便該任務可以使用給定的令牌重新實例化該鎖。

在下面的範例中，如果成功取得鎖，我們將分派一個佇列任務。此外，我們將透過鎖的 `owner` 方法將鎖的擁有者令牌傳遞給佇列任務：

```php
$podcast = Podcast::find($id);

$lock = Cache::lock('processing', 120);

if ($lock->get()) {
    ProcessPodcast::dispatch($podcast, $lock->owner());
}
```

在我們的應用程式 `ProcessPodcast` 任務中，我們可以使用擁有者令牌來還原並釋放鎖：

```php
Cache::restoreLock('processing', $this->owner)->release();
```

如果您想在無視目前擁有者的情況下釋放鎖，可以使用 `forceRelease` 方法：

```php
Cache::lock('processing')->forceRelease();
```


<a name="refreshing-locks"></a>
### 重新整理鎖

若您需要延長您目前擁有的鎖的過期時間，可以使用 `refresh` 方法。若未提供秒數，將會使用鎖的原始時長。這對於長時間執行的操作非常有用，您可以取得短時間的鎖並定期延長它，而不是取得一個過期時間非常長的鎖：

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

Laravel 的原子鎖功能還提供了一些限制閉包同時執行的簡單方法。當您希望在整個基礎架構中僅允許一個正在運行的實例時，請使用 `withoutOverlapping`：

```php
Cache::withoutOverlapping('foo', function () {
    // Lock acquired after waiting a maximum of 10 seconds...
});
```

預設情況下，鎖會保持持有直到閉包執行完畢，且該方法最多等待 10 秒來取得鎖。您可以透過額外的引數自訂這些值：

```php
Cache::withoutOverlapping('foo', function () {
    // Lock acquired for 120 seconds after waiting a maximum of 5 seconds...
}, lockFor: 120, waitFor: 5);
```

若無法在指定的等待時間內取得鎖，將會拋出 `Illuminate\Contracts\Cache\LockTimeoutException`。

若您需要受控的平行處理，可以使用 `funnel` 方法來設定最大同時執行數。`funnel` 方法支援任何支援鎖的快取驅動程式：

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

`funnel` 鍵名標示了正在限制的資源。`limit` 方法定義了最大同時執行數。`releaseAfter` 方法設定安全超時秒數，在過期後取得的槽位將會自動釋放。`block` 方法則設定要等待可用槽位的秒數。

若您希望透過例外處理超時情況，而不是提供失敗閉包，您可以省略第二個閉包。若無法在指定的等待時間內取得鎖，將會拋出 `Illuminate\Cache\Limiters\LimiterTimeoutException`：

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

若您想對並行限制器使用特定的快取儲存庫，可以在所選的儲存庫上呼叫 `funnel` 方法：

```php
Cache::store('redis')->funnel('foo')
    ->limit(3)
    ->block(10)
    ->then(function () {
        // Concurrency lock acquired using the "redis" store...
    });
```

> [!NOTE]
> `funnel` 方法要求快取儲存庫實作 `Illuminate\Contracts\Cache\LockProvider` 介面。若您嘗試在不支援鎖的快取儲存庫上使用 `funnel`，將會拋出 `BadMethodCallException`。


<a name="cache-failover"></a>
## 快取故障轉移

`failover` 快取驅動程式在與快取進行互動時提供了自動故障轉移 (Failover) 功能。如果 `failover` 儲存庫的主快取儲存庫因任何原因發生故障，Laravel 將自動嘗試使用清單中設定的下一個儲存庫。這對於確保生產環境中的高可用性特別有用，因為在生產環境中快取的可靠性至關重要。

要設定故障轉移快取儲存庫，請指定 `failover` 驅動程式並按順序提供要嘗試的儲存庫名稱陣列。預設情況下，Laravel 在您應用程式的 `config/cache.php` 設定檔中包含了一個範例故障轉移設定：

```php
'failover' => [
    'driver' => 'failover',
    'stores' => [
        'database',
        'array',
    ],
],
```

一旦設定了使用 `failover` 驅動程式的儲存庫，您將需要在應用程式的 `.env` 檔案中將故障轉移儲存庫設為預設快取儲存庫，以使用故障轉移功能：

```ini
CACHE_STORE=failover
```

當快取儲存庫操作失敗且故障轉移啟動時，Laravel 將發送 `Illuminate\Cache\Events\CacheFailedOver` 事件，允許您回報或記錄快取儲存庫發生的故障。

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

我們只需要使用 MongoDB 連線來實作這些方法中的每一個。關於如何實作這些方法的範例，可以參考 [Laravel 框架原始碼](https://github.com/laravel/framework) 中的 `Illuminate\Cache\MemcachedStore`。實作完成後，我們可以透過呼叫 `Cache` Facade 的 `extend` 方法來完成自訂驅動程式的註冊：

```php
Cache::extend('mongo', function (Application $app) {
    return Cache::repository(new MongoStore);
});
```

> [!NOTE]
> 如果您想知道該把自訂快取驅動程式的程式碼放在哪裡，可以在 `app` 目錄下建立一個 `Extensions` 命名空間。不過請記住，Laravel 並沒有嚴格限制應用程式架構，您可以自由地根據個人喜好來組織您的應用程式。


<a name="registering-the-driver"></a>
### 註冊驅動程式

要向 Laravel 註冊自訂快取驅動程式，我們將使用 `Cache` Facade 上的 `extend` 方法。由於其他服務提供者(Service Providers)可能會嘗試在他們的 `boot` 方法中讀取快取值，因此我們將在 `booting` 回呼函式內註冊自訂驅動程式。透過使用 `booting` 回呼函式，我們可以確保自訂驅動程式是在應用程式服務提供者的 `boot` 方法被呼叫前、且在所有服務提供者的 `register` 方法被呼叫後完成註冊。我們將在應用程式的 `App\Providers\AppServiceProvider` 類別的 `register` 方法內註冊 `booting` 回呼函式：

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

傳給 `extend` 方法的第一個引數是驅動程式的名稱。這將對應到 `config/cache.php` 設定檔中的 `driver` 選項。第二個引數是一個閉包 (Closure)，該閉包應回傳一個 `Illuminate\Cache\Repository` 實例。閉包會接收到一個 `$app` 實例，該實例即為[服務容器](/docs/{{version}}/container)的實例。

擴充功能註冊完成後，請將應用程式 `config/cache.php` 設定檔內的 `CACHE_STORE` 環境變數或 `default` 選項更新為您擴充功能的名稱。


<a name="events"></a>
## 事件

若要在每次進行快取操作時執行程式碼，您可以監聽由快取所發出的各種[事件](/docs/{{version}}/events)：

<div class="overflow-auto">

| 事件名稱 |
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

為了提高效能，您可以在應用程式的 `config/cache.php` 設定檔中，將特定快取 Store 的 `events` 設定選項設為 `false` 來停用快取事件：

```php
'database' => [
    'driver' => 'database',
    // ...
    'events' => false,
],
```