# 快取

- [簡介](#introduction)
- [設定](#configuration)
    - [驅動程式先決條件](#driver-prerequisites)
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
    - [跨行程管理鎖定](#managing-locks-across-processes)
- [快取容錯移轉](#cache-failover)
- [新增自訂快取驅動程式](#adding-custom-cache-drivers)
    - [撰寫驅動程式](#writing-the-driver)
    - [註冊驅動程式](#registering-the-driver)
- [事件](#events)

<a name="introduction"></a>
## 簡介

您的應用程式執行的一些資料擷取或處理任務可能耗費大量 CPU 資源或需要數秒才能完成。在這種情況下，通常會將擷取的資料快取一段時間，以便在後續的相同資料請求中能快速擷取。快取資料通常儲存在速度非常快的資料儲存區，例如 [Memcached](https://memcached.org) 或 [Redis](https://redis.io)。

幸運的是，Laravel 為各種快取後端提供了富有表達力且統一的 API，讓您可以利用其極快的資料擷取速度來加速您的 Web 應用程式。

<a name="configuration"></a>
## 設定

您的應用程式快取設定檔位於 `config/cache.php`。在此檔案中，您可以指定應用程式預設要使用的快取儲存區。Laravel 支援常見的快取後端，例如 [Memcached](https://memcached.org)、[Redis](https://redis.io)、[DynamoDB](https://aws.amazon.com/dynamodb) 和關聯式資料庫。此外，還提供了基於檔案的快取驅動程式，而 `array` 和 `null` 快取驅動程式則為您的自動化測試提供了便捷的快取後端。

快取設定檔還包含其他多種可供您檢閱的選項。預設情況下，Laravel 配置為使用 `database` 快取驅動程式，它將序列化的快取物件儲存在您的應用程式資料庫中。

<a name="driver-prerequisites"></a>
### 驅動程式先決條件

<a name="prerequisites-database"></a>
#### 資料庫

使用 `database` 快取驅動程式時，您需要一個資料庫表格來存放快取資料。通常，這已包含在 Laravel 預設的 `0001_01_01_000001_create_cache_table.php` [資料庫遷移](/docs/{{version}}/migrations)中；但是，如果您的應用程式不包含此遷移，您可以使用 `make:cache-table` Artisan 指令來建立它：

```shell
php artisan make:cache-table

php artisan migrate
```

<a name="memcached"></a>
#### Memcached

使用 Memcached 驅動程式需要安裝 [Memcached PECL 套件](https://pecl.php.net/package/memcached)。您可以在 `config/cache.php` 設定檔中列出所有 Memcached 伺服器。此檔案已包含 `memcached.servers` 項目以供您開始使用：

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

如有需要，您可以將 `host` 選項設定為 UNIX socket 路徑。如果您這樣做，`port` 選項應設定為 `0`：

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

在 Laravel 中使用 Redis 快取之前，您需要透過 PECL 安裝 PhpRedis PHP 擴充功能，或透過 Composer 安裝 `predis/predis` 套件 (~2.0)。[Laravel Sail](/docs/{{version}}/sail) 已包含此擴充功能。此外，官方 Laravel 應用程式平台，例如 [Laravel Cloud](https://cloud.laravel.com) 和 [Laravel Forge](https://forge.laravel.com)，預設都安裝了 PhpRedis 擴充功能。

有關配置 Redis 的更多資訊，請查閱其 [Laravel 文件頁面](/docs/{{version}}/redis#configuration)。

<a name="dynamodb"></a>
#### DynamoDB

在使用 [DynamoDB](https://aws.amazon.com/dynamodb) 快取驅動程式之前，您必須建立一個 DynamoDB 表格來儲存所有快取資料。通常，此表格應命名為 `cache`。但是，您應該根據 `cache` 設定檔中 `stores.dynamodb.table` 設定值來命名表格。表格名稱也可以透過 `DYNAMODB_CACHE_TABLE` 環境變數來設定。

此表格還應具有一個字串分區鍵，其名稱應與應用程式 `cache` 設定檔中 `stores.dynamodb.attributes.key` 設定項目相對應。預設情況下，分區鍵應命名為 `key`。

通常，DynamoDB 不會主動從表格中移除過期的項目。因此，您應該在表格上 [啟用存留時間 (TTL)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)。配置表格的 TTL 設定時，您應該將 TTL 屬性名稱設定為 `expires_at`。

接下來，安裝 AWS SDK 以便您的 Laravel 應用程式可以與 DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

此外，您應該確保為 DynamoDB 快取儲存區設定選項提供值。通常，這些選項，例如 `AWS_ACCESS_KEY_ID` 和 `AWS_SECRET_ACCESS_KEY`，應在您應用程式的 `.env` 設定檔中定義：

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

如果您使用 MongoDB，官方的 `mongodb/laravel-mongodb` 套件提供了 `mongodb` 快取驅動程式，並且可以使用 `mongodb` 資料庫連線進行配置。MongoDB 支援 TTL 索引，可用於自動清除過期的快取項目。

有關配置 MongoDB 的更多資訊，請參閱 MongoDB [快取和鎖定文件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/cache/)。

<a name="cache-usage"></a>
## 快取使用

<a name="obtaining-a-cache-instance"></a>
### 取得快取實例

若要取得快取儲存實例，您可以使用 `Cache` facade，這也是我們將在本文件中使用的。`Cache` facade 提供方便且簡潔的方式來存取 Laravel 快取契約的底層實作：

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

您可以使用 `Cache` facade 透過 `store` 方法存取各種快取儲存。傳遞給 `store` 方法的鍵應對應您 `cache` 設定檔中 `stores` 設定陣列中列出的其中一個儲存：

```php
$value = Cache::store('file')->get('foo');

Cache::store('redis')->put('bar', 'baz', 600); // 10 Minutes
```

<a name="retrieving-items-from-the-cache"></a>
### 從快取中擷取項目

`Cache` facade 的 `get` 方法用於從快取中擷取項目。如果該項目不存在於快取中，則會傳回 `null`。如果您願意，可以將第二個參數傳遞給 `get` 方法，指定當項目不存在時您希望傳回的預設值：

```php
$value = Cache::get('key');

$value = Cache::get('key', 'default');
```

您甚至可以傳遞閉包作為預設值。如果指定的項目不存在於快取中，則會傳回閉包的結果。傳遞閉包讓您可以延遲從資料庫或其他外部服務中擷取預設值：

```php
$value = Cache::get('key', function () {
    return DB::table(/* ... */)->get();
});
```

<a name="determining-item-existence"></a>
#### 判斷項目是否存在

`has` 方法可用於判斷項目是否存在於快取中。如果項目存在但其值為 `null`，此方法也會傳回 `false`：

```php
if (Cache::has('key')) {
    // ...
}
```

<a name="incrementing-decrementing-values"></a>
#### 遞增 / 遞減值

`increment` 和 `decrement` 方法可用於調整快取中整數項目的值。這兩個方法都接受一個可選的第二個參數，指示要遞增或遞減項目值的數量：

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
#### 擷取並儲存

有時您可能希望從快取中擷取項目，但如果請求的項目不存在，也儲存一個預設值。例如，您可能希望從快取中擷取所有使用者，或者如果他們不存在，則從資料庫中擷取他們並將他們新增到快取中。您可以使用 `Cache::remember` 方法來完成此操作：

```php
$value = Cache::remember('users', $seconds, function () {
    return DB::table('users')->get();
});
```

如果項目不存在於快取中，則傳遞給 `remember` 方法的閉包將會執行，其結果將被放置在快取中。

您可以使用 `rememberForever` 方法從快取中擷取項目，或者如果項目不存在則永久儲存它：

```php
$value = Cache::rememberForever('users', function () {
    return DB::table('users')->get();
});
```

<a name="swr"></a>
#### 舊資料重新驗證

使用 `Cache::remember` 方法時，如果快取值已過期，部分使用者可能會遇到較慢的回應時間。對於某些類型的資料，在背景重新計算快取值的同時允許提供部分過時的資料會很有用，這可以防止部分使用者在計算快取值時遇到較慢的回應時間。這通常被稱為「舊資料重新驗證」模式，而 `Cache::flexible` 方法提供了此模式的實作。

該彈性方法接受一個陣列，該陣列指定快取值被視為「新鮮」的時間，以及何時會變為「過時」。陣列中的第一個值代表快取被視為新鮮的秒數，而第二個值定義了在重新計算之前可以作為過時資料提供的時間長度。

如果在新鮮期 (在第一個值之前) 發出請求，快取將立即傳回，無需重新計算。如果在過時期 (在兩個值之間) 發出請求，則將過時值提供給使用者，並註冊一個 [延遲函式](/docs/{{version}}/helpers#deferred-functions) 以在回應傳送給使用者後重新整理快取值。如果請求在第二個值之後發出，快取將被視為已過期，並且會立即重新計算值，這可能會導致使用者回應較慢：

```php
$value = Cache::flexible('users', [5, 10], function () {
    return DB::table('users')->get();
});
```

<a name="retrieve-delete"></a>
#### 擷取並刪除

如果您需要從快取中擷取項目然後刪除該項目，您可以使用 `pull` 方法。與 `get` 方法一樣，如果項目不存在於快取中，將傳回 `null`：

```php
$value = Cache::pull('key');

$value = Cache::pull('key', 'default');
```

<a name="storing-items-in-the-cache"></a>
### 在快取中儲存項目

您可以使用 `Cache` facade 上的 `put` 方法將項目儲存到快取中：

```php
Cache::put('key', 'value', $seconds = 10);
```

如果沒有將儲存時間傳遞給 `put` 方法，該項目將會無限期儲存：

```php
Cache::put('key', 'value');
```

除了將秒數作為整數傳遞外，您還可以傳遞一個 `DateTime` 實例，代表快取項目所需的過期時間：

```php
Cache::put('key', 'value', now()->plus(minutes: 10));
```

<a name="store-if-not-present"></a>
#### 若不存在則儲存

`add` 方法只會在快取儲存中不存在該項目時才將其新增到快取中。如果項目確實已新增到快取中，該方法將傳回 `true`。否則，該方法將傳回 `false`。`add` 方法是一個原子操作：

```php
Cache::add('key', 'value', $seconds);
```

<a name="storing-items-forever"></a>
#### 永久儲存項目

`forever` 方法可用於將項目永久儲存到快取中。由於這些項目不會過期，因此必須使用 `forget` 方法手動從快取中移除：

```php
Cache::forever('key', 'value');
```

> [!NOTE]
> 如果您正在使用 Memcached 驅動程式，則「永久」儲存的項目可能會在快取達到其大小限制時被移除。

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
> 清除快取不尊重您設定的快取「前綴」，並將從快取中移除所有條目。在清除由其他應用程式共用的快取時，請仔細考慮這一點。

<a name="cache-memoization"></a>
### 快取記憶化

Laravel 的 `memo` 快取驅動程式讓您可以在單一請求或 Job 執行期間，在記憶體中暫時儲存已解析的快取值。這可以避免在相同的執行中重複命中快取，大幅提升效能。

若要使用記憶化的快取，請呼叫 `memo` 方法：

```php
use Illuminate\Support\Facades\Cache;

$value = Cache::memo()->get('key');
```

`memo` 方法可以選擇性地接受快取儲存的名稱，這會指定記憶化驅動程式將會裝飾的底層快取儲存：

```php
// Using the default cache store...
$value = Cache::memo()->get('key');

// Using the Redis cache store...
$value = Cache::memo('redis')->get('key');
```

對於給定 Key 的第一次 `get` 呼叫會從您的快取儲存中擷取值，但在相同請求或 Job 中的後續呼叫將會從記憶體中擷取值：

```php
// Hits the cache...
$value = Cache::memo()->get('key');

// Does not hit the cache, returns memoized value...
$value = Cache::memo()->get('key');
```

當呼叫修改快取值的方法時 (例如 `put`、`increment`、`remember` 等)，記憶化的快取會自動忘記記憶化的值，並將變動性方法呼叫委派給底層快取儲存：

```php
Cache::memo()->put('name', 'Taylor'); // Writes to underlying cache...
Cache::memo()->get('name');           // Hits underlying cache...
Cache::memo()->get('name');           // Memoized, does not hit cache...

Cache::memo()->put('name', 'Tim');    // Forgets memoized value, writes new value...
Cache::memo()->get('name');           // Hits underlying cache again...
```

<a name="the-cache-helper"></a>
### 快取輔助函式

除了使用 `Cache` Facade 之外，您也可以使用全域的 `cache` 函式來透過快取擷取及儲存資料。當 `cache` 函式被呼叫並帶有一個字串參數時，它會回傳給定 Key 的值：

```php
$value = cache('key');
```

若您提供 Key / Value 陣列與到期時間給函式，它會將值儲存在快取中，並持續指定的期間：

```php
cache(['key' => 'value'], $seconds);

cache(['key' => 'value'], now()->plus(minutes: 10));
```

當 `cache` 函式在沒有任何引數的情況下被呼叫時，它會回傳 `Illuminate\Contracts\Cache\Factory` 實作的實例，讓您可以呼叫其他快取方法：

```php
cache()->remember('users', $seconds, function () {
    return DB::table('users')->get();
});
```

> [!NOTE]
> 當測試對全域 `cache` 函式的呼叫時，您可以使用 `Cache::shouldReceive` 方法，就像您在[測試 Facade](/docs/{{version}}/mocking#mocking-facades) 一樣。

<a name="cache-tags"></a>
## 快取標籤

> [!WARNING]
> 當使用 `file`、`dynamodb` 或 `database` 快取驅動程式時，不支援快取標籤。


<a name="storing-tagged-cache-items"></a>
### 儲存帶有標籤的快取項目

快取標籤允許您為快取中的相關項目貼上標籤，然後清除所有分配了給定標籤的快取值。您可以透過傳入一個有序的標籤名稱陣列來存取帶有標籤的快取。例如，讓我們存取一個帶有標籤的快取並將一個值 `put` 到快取中：

```php
use Illuminate\Support\Facades\Cache;

Cache::tags(['people', 'artists'])->put('John', $john, $seconds);
Cache::tags(['people', 'authors'])->put('Anne', $anne, $seconds);
```


<a name="accessing-tagged-cache-items"></a>
### 存取帶有標籤的快取項目

透過標籤儲存的項目，在未提供用於儲存該值的標籤時，可能無法存取。要擷取帶有標籤的快取項目，請將相同的有序標籤列表傳遞給 `tags` 方法，然後呼叫 `get` 方法並帶上您希望擷取的鍵：

```php
$john = Cache::tags(['people', 'artists'])->get('John');

$anne = Cache::tags(['people', 'authors'])->get('Anne');
```


<a name="removing-tagged-cache-items"></a>
### 移除帶有標籤的快取項目

您可以清除所有分配了某個標籤或多個標籤的項目。例如，以下程式碼將移除所有標記為 `people`、`authors` 或兩者皆是的快取。因此，`Anne` 和 `John` 都將從快取中移除：

```php
Cache::tags(['people', 'authors'])->flush();
```

相比之下，下面的程式碼將僅移除標記為 `authors` 的快取值，因此 `Anne` 將被移除，但 `John` 不會：

```php
Cache::tags('authors')->flush();
```


<a name="atomic-locks"></a>
## 原子鎖

> [!WARNING]
> 要使用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動程式作為應用程式的預設快取驅動程式。此外，所有伺服器都必須與同一個中央快取伺服器通訊。


<a name="managing-locks"></a>
### 管理鎖定

原子鎖允許操作分散式鎖定而無需擔心競爭條件。例如，[Laravel Cloud](https://cloud.laravel.com) 使用原子鎖來確保在任何時候只有一個遠端任務在伺服器上執行。您可以透過 `Cache::lock` 方法來建立和管理鎖定：

```php
use Illuminate\Support\Facades\Cache;

$lock = Cache::lock('foo', 10);

if ($lock->get()) {
    // Lock acquired for 10 seconds...

    $lock->release();
}
```

`get` 方法也接受一個閉包。閉包執行後，Laravel 將自動釋放鎖定：

```php
Cache::lock('foo', 10)->get(function () {
    // Lock acquired for 10 seconds and automatically released...
});
```

如果您請求時鎖定不可用，您可以指示 Laravel 等待指定的秒數。如果在指定時間限制內無法取得鎖定，將會拋出 `Illuminate\Contracts\Cache\LockTimeoutException`：

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

透過將閉包傳遞給 `block` 方法，上述範例可以簡化。當閉包傳遞給此方法時，Laravel 將嘗試在指定秒數內取得鎖定，並在閉包執行後自動釋放鎖定：

```php
Cache::lock('foo', 10)->block(5, function () {
    // Lock acquired for 10 seconds after waiting a maximum of 5 seconds...
});
```


<a name="managing-locks-across-processes"></a>
### 跨行程管理鎖定

有時，您可能希望在一個行程中取得鎖定，並在另一個行程中釋放它。例如，您可能在網頁請求期間取得一個鎖定，並希望在該請求觸發的佇列任務結束時釋放該鎖定。在這種情況下，您應該將鎖定的範圍「擁有者代幣」傳遞給佇列任務，以便該任務可以使用給定的代幣重新實例化鎖定。

在下面的範例中，如果成功取得鎖定，我們將分派一個佇列任務。此外，我們將透過鎖定的 `owner` 方法將鎖定的擁有者代幣傳遞給佇列任務：

```php
$podcast = Podcast::find($id);

$lock = Cache::lock('processing', 120);

if ($lock->get()) {
    ProcessPodcast::dispatch($podcast, $lock->owner());
}
```

在我們應用程式的 `ProcessPodcast` 任務中，我們可以使用擁有者代幣來還原並釋放鎖定：

```php
Cache::restoreLock('processing', $this->owner)->release();
```

如果您希望在不考慮其當前擁有者的情況下釋放鎖定，您可以使用 `forceRelease` 方法：

```php
Cache::lock('processing')->forceRelease();
```


<a name="cache-failover"></a>
## 快取容錯移轉

`failover` 快取驅動程式在與快取互動時提供了自動容錯移轉功能。如果 `failover` 儲存的主要快取儲存因任何原因失敗，Laravel 將自動嘗試使用列表中的下一個配置儲存。這對於確保快取可靠性至關重要的生產環境中的高可用性特別有用。

要配置一個容錯移轉快取儲存，請指定 `failover` 驅動程式並提供一個儲存名稱陣列，以依序嘗試。預設情況下，Laravel 在您應用程式的 `config/cache.php` 設定檔中包含一個容錯移轉範例配置：

```php
'failover' => [
    'driver' => 'failover',
    'stores' => [
        'database',
        'array',
    ],
],
```

一旦您配置了一個使用 `failover` 驅動程式的儲存，您將需要將容錯移轉儲存設定為您應用程式 `.env` 檔案中的預設快取儲存，以便利用容錯移轉功能：

```ini
CACHE_STORE=failover
```

當快取儲存操作失敗並啟動容錯移轉時，Laravel 將會分派 `Illuminate\Cache\Events\CacheFailedOver` 事件，允許您報告或記錄快取儲存已失敗。

<a name="adding-custom-cache-drivers"></a>
## 新增自訂快取驅動程式

<a name="writing-the-driver"></a>
### 撰寫驅動程式

為了建立自訂快取驅動程式，我們首先需要實作 `Illuminate\Contracts\Cache\Store` [Contract](/docs/{{version}}/contracts)。因此，MongoDB 快取實作可能看起來像這樣：

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

我們只需要使用 MongoDB 連線來實作這些方法。關於如何實作這些方法的範例，請參考 [Laravel 框架原始碼](https://github.com/laravel/framework) 中的 `Illuminate\Cache\MemcachedStore`。一旦我們的實作完成，我們就可以透過呼叫 `Cache` Facade 的 `extend` 方法來完成自訂驅動程式的註冊：

```php
Cache::extend('mongo', function (Application $app) {
    return Cache::repository(new MongoStore);
});
```

> [!NOTE]
> 如果您想知道自訂快取驅動程式程式碼應該放在哪裡，您可以在 `app` 目錄中建立一個 `Extensions` 命名空間。不過，請記住 Laravel 並沒有嚴格的應用程式結構，您可以自由地根據自己的喜好來組織應用程式。

<a name="registering-the-driver"></a>
### 註冊驅動程式

為了向 Laravel 註冊自訂快取驅動程式，我們將在 `Cache` Facade 上使用 `extend` 方法。由於其他服務提供者可能會在其 `boot` 方法中嘗試讀取快取值，我們將在 `booting` 回呼函式中註冊自訂驅動程式。透過使用 `booting` 回呼函式，我們可以確保自訂驅動程式在應用程式服務提供者的 `boot` 方法被呼叫之前、但所有服務提供者的 `register` 方法被呼叫之後註冊。我們將在應用程式 `App\Providers\AppServiceProvider` 類別的 `register` 方法中註冊 `booting` 回呼函式：

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

傳遞給 `extend` 方法的第一個參數是驅動程式的名稱。這將對應於您在 `config/cache.php` 設定檔中的 `driver` 選項。第二個參數是一個閉包，它應該回傳一個 `Illuminate\Cache\Repository` 實例。此閉包將會傳遞一個 `$app` 實例，該實例是 [服務容器](/docs/{{version}}/container) 的實例。

一旦您的擴充功能註冊完成，請將應用程式 `config/cache.php` 設定檔中的 `CACHE_STORE` 環境變數或 `default` 選項更新為您的擴充功能名稱。

<a name="events"></a>
## 事件

若要在每個快取操作上執行程式碼，您可以監聽由快取分派的各種 [事件](/docs/{{version}}/events)：

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

為了提高效能，您可以透過在應用程式的 `config/cache.php` 設定檔中，將特定快取儲存的 `events` 設定選項設為 `false`，以停用快取事件：

```php
'database' => [
    'driver' => 'database',
    // ...
    'events' => false,
],
```