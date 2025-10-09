# 快取

- [介紹](#introduction)
- [設定](#configuration)
    - [驅動程式先決條件](#driver-prerequisites)
- [快取使用](#cache-usage)
    - [取得快取實例](#obtaining-a-cache-instance)
    - [從快取中取回項目](#retrieving-items-from-the-cache)
    - [將項目儲存到快取](#storing-items-in-the-cache)
    - [從快取中移除項目](#removing-items-from-the-cache)
    - [快取記憶化](#cache-memoization)
    - [快取輔助函式](#the-cache-helper)
- [快取標籤](#cache-tags)
- [原子鎖](#atomic-locks)
    - [管理鎖定](#managing-locks)
    - [跨程序管理鎖定](#managing-locks-across-processes)
- [新增自訂快取驅動程式](#adding-custom-cache-drivers)
    - [編寫驅動程式](#writing-the-driver)
    - [註冊驅動程式](#registering-the-driver)
- [事件](#events)

<a name="introduction"></a>
## 介紹

您的應用程式執行的一些資料擷取或處理任務可能會是 CPU 密集型，或需要數秒才能完成。在這種情況下，通常會將擷取到的資料快取一段時間，以便後續對相同資料的請求能夠快速取回。快取資料通常儲存在速度非常快的資料儲存庫中，例如 [Memcached](https://memcached.org) 或 [Redis](https://redis.io)。

值得慶幸的是，Laravel 為各種快取後端提供了富有表達力且統一的 API，讓您可以利用它們極快的資料擷取速度，並加速您的網路應用程式。

<a name="configuration"></a>
## 設定

您的應用程式快取設定檔位於 `config/cache.php`。在此檔案中，您可以指定在應用程式中預設要使用的快取儲存庫。Laravel 開箱即用支援熱門的快取後端，例如 [Memcached](https://memcached.org)、[Redis](https://redis.io)、[DynamoDB](https://aws.amazon.com/dynamodb) 和關聯式資料庫。此外，還提供了檔案型快取驅動程式，而 `array` 和 `null` 快取驅動程式則為您的自動化測試提供了方便的快取後端。

快取設定檔還包含您可以查閱的各種其他選項。預設情況下，Laravel 配置為使用 `database` 快取驅動程式，該驅動程式將序列化後的快取物件儲存在您的應用程式資料庫中。

<a name="driver-prerequisites"></a>
### 驅動程式先決條件

<a name="prerequisites-database"></a>
#### 資料庫

使用 `database` 快取驅動程式時，您將需要一個資料庫表格來存放快取資料。通常，這包含在 Laravel 預設的 `0001_01_01_000001_create_cache_table.php` [資料庫遷移](/docs/{{version}}/migrations)中；但是，如果您的應用程式不包含此遷移，您可以使用 `make:cache-table` Artisan 指令來建立它：

```shell
php artisan make:cache-table

php artisan migrate
```

<a name="memcached"></a>
#### Memcached

使用 Memcached 驅動程式需要安裝 [Memcached PECL 套件](https://pecl.php.net/package/memcached)。您可以在 `config/cache.php` 設定檔中列出所有 Memcached 伺服器。此檔案已包含一個 `memcached.servers` 項目以供您開始使用：

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

在將 Redis 快取與 Laravel 搭配使用之前，您需要透過 PECL 安裝 PhpRedis PHP 擴充功能，或透過 Composer 安裝 `predis/predis` 套件 (~2.0)。[Laravel Sail](/docs/{{version}}/sail) 已包含此擴充功能。此外，官方 Laravel 應用程式平台，例如 [Laravel Cloud](https://cloud.laravel.com) 和 [Laravel Forge](https://forge.laravel.com)，預設都安裝了 PhpRedis 擴充功能。

有關設定 Redis 的更多資訊，請查閱其 [Laravel 文件頁面](/docs/{{version}}/redis#configuration)。

<a name="dynamodb"></a>
#### DynamoDB

在使用 [DynamoDB](https://aws.amazon.com/dynamodb) 快取驅動程式之前，您必須建立一個 DynamoDB 表格來儲存所有快取資料。通常，此表格應命名為 `cache`。不過，您應根據 `cache` 設定檔中 `stores.dynamodb.table` 設定值來命名表格。表格名稱也可以透過 `DYNAMODB_CACHE_TABLE` 環境變數來設定。

此表格還應有一個字串分割鍵，其名稱與您的應用程式 `cache` 設定檔中 `stores.dynamodb.attributes.key` 設定項目相對應。預設情況下，分割鍵應命名為 `key`。

通常，DynamoDB 不會主動從表格中移除過期項目。因此，您應該在表格上 [啟用 Time to Live (TTL)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)。設定表格的 TTL 時，您應該將 TTL 屬性名稱設定為 `expires_at`。

接下來，安裝 AWS SDK，以便您的 Laravel 應用程式可以與 DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

此外，您應確保為 DynamoDB 快取儲存庫設定選項提供了值。通常，這些選項，例如 `AWS_ACCESS_KEY_ID` 和 `AWS_SECRET_ACCESS_KEY`，應在您的應用程式 `.env` 設定檔中定義：

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

如果您使用 MongoDB，官方的 `mongodb/laravel-mongodb` 套件提供了 `mongodb` 快取驅動程式，並可以使用 `mongodb` 資料庫連線進行配置。MongoDB 支援 TTL 索引，可用於自動清除過期的快取項目。

有關配置 MongoDB 的更多資訊，請參閱 MongoDB 的 [快取與鎖定文件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/cache/)。

<a name="cache-usage"></a>
## 快取使用

<a name="obtaining-a-cache-instance"></a>
### 取得快取實例

要取得快取儲存實例，您可以使用 `Cache` facade，這也是我們在整個文件中將會使用的。`Cache` facade 提供了方便、簡潔的存取方式，用於底層的 Laravel 快取合約實作：

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
#### 存取多個快取儲存區

使用 `Cache` facade，您可以透過 `store` 方法存取各種快取儲存區。傳遞給 `store` 方法的鍵 (key) 應該與您 `cache` 設定檔中 `stores` 設定陣列裡列出的其中一個儲存區相對應：

```php
$value = Cache::store('file')->get('foo');

Cache::store('redis')->put('bar', 'baz', 600); // 10 Minutes
```

<a name="retrieving-items-from-the-cache"></a>
### 從快取中取回項目

`Cache` facade 的 `get` 方法用於從快取中取回項目。如果快取中不存在該項目，將會回傳 `null`。如果您願意，可以向 `get` 方法傳遞第二個參數，指定當項目不存在時您希望回傳的預設值：

```php
$value = Cache::get('key');

$value = Cache::get('key', 'default');
```

您甚至可以傳遞一個閉包 (closure) 作為預設值。如果指定的項目在快取中不存在，該閉包的結果將會被回傳。傳遞閉包允許您延遲從資料庫或其他外部服務中取回預設值：

```php
$value = Cache::get('key', function () {
    return DB::table(/* ... */)->get();
});
```

<a name="determining-item-existence"></a>
#### 判斷項目是否存在

`has` 方法可用於判斷項目是否存在於快取中。如果項目存在但其值為 `null`，此方法也將回傳 `false`：

```php
if (Cache::has('key')) {
    // ...
}
```

<a name="incrementing-decrementing-values"></a>
#### 增加 / 減少數值

`increment` 和 `decrement` 方法可用於調整快取中整數項目的值。這兩個方法都接受一個可選的第二個參數，指示要增加或減少項目值的數量：

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
#### 取回並儲存

有時您可能希望從快取中取回項目，但如果請求的項目不存在，也希望儲存一個預設值。例如，您可能希望從快取中取回所有使用者，或者如果他們不存在，則從資料庫中取回並將他們加入快取。您可以使用 `Cache::remember` 方法來完成此操作：

```php
$value = Cache::remember('users', $seconds, function () {
    return DB::table('users')->get();
});
```

如果快取中不存在該項目，傳遞給 `remember` 方法的閉包將會執行，其結果將被放入快取中。

您可以使用 `rememberForever` 方法從快取中取回項目，或者如果它不存在，則永久儲存它：

```php
$value = Cache::rememberForever('users', function () {
    return DB::table('users')->get();
});
```

<a name="swr"></a>
#### 過期時重新驗證 (Stale While Revalidate)

使用 `Cache::remember` 方法時，如果快取值已過期，某些使用者可能會遇到緩慢的回應時間。對於某些類型的資料，在背景重新計算快取值的同時，允許提供部分過時的資料會很有用，這可以防止某些使用者在計算快取值時遇到緩慢的回應時間。這通常被稱為「過期時重新驗證 (stale-while-revalidate)」模式，而 `Cache::flexible` 方法提供了此模式的實作。

此 flexible 方法接受一個陣列，用於指定快取值被視為「新鮮 (fresh)」的時間長度，以及何時會變為「過時 (stale)」。陣列中的第一個值代表快取被視為新鮮的秒數，而第二個值則定義了在需要重新計算之前，它可以作為過時資料提供的時間長度。

如果在新鮮期內（在第一個值之前）提出請求，快取將立即回傳，無需重新計算。如果在過時期間（在兩個值之間）提出請求，過時值將提供給使用者，並且會註冊一個 [延遲函式](/docs/{{version}}/helpers#deferred-functions) 在回應傳送給使用者後重新整理快取值。如果在第二個值之後提出請求，快取將被視為已過期，並且會立即重新計算值，這可能會導致使用者回應速度變慢：

```php
$value = Cache::flexible('users', [5, 10], function () {
    return DB::table('users')->get();
});
```

<a name="retrieve-delete"></a>
#### 取回並刪除

如果您需要從快取中取回一個項目，然後刪除該項目，您可以使用 `pull` 方法。與 `get` 方法一樣，如果快取中不存在該項目，將會回傳 `null`：

```php
$value = Cache::pull('key');

$value = Cache::pull('key', 'default');
```

<a name="storing-items-in-the-cache"></a>
### 將項目儲存到快取

您可以使用 `Cache` facade 上的 `put` 方法將項目儲存到快取中：

```php
Cache::put('key', 'value', $seconds = 10);
```

如果沒有將儲存時間傳遞給 `put` 方法，該項目將會被永久儲存：

```php
Cache::put('key', 'value');
```

除了傳遞秒數作為整數外，您還可以傳遞一個 `DateTime` 實例，代表快取項目期望的過期時間：

```php
Cache::put('key', 'value', now()->addMinutes(10));
```

<a name="store-if-not-present"></a>
#### 若不存在則儲存

`add` 方法只會在快取儲存區中不存在該項目時，才將其加入快取。如果項目確實已加入快取，該方法將會回傳 `true`。否則，該方法將會回傳 `false`。`add` 方法是一個原子操作 (atomic operation)：

```php
Cache::add('key', 'value', $seconds);
```

<a name="storing-items-forever"></a>
#### 永久儲存項目

`forever` 方法可用於永久儲存快取中的項目。由於這些項目不會過期，必須使用 `forget` 方法手動從快取中移除：

```php
Cache::forever('key', 'value');
```

> [!NOTE]
> 如果您使用 Memcached 驅動程式，當快取達到其大小限制時，永久儲存的項目可能會被移除。

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
> 清除快取不會遵守您設定的快取「前綴 (prefix)」，並且將會從快取中移除所有條目。在清除由其他應用程式共用的快取時，請仔細考慮這一點。

<a name="cache-memoization"></a>
### 快取記憶化

Laravel 的 `memo` 快取驅動程式允許您在單次請求或任務執行期間，將解析後的快取值暫時儲存在記憶體中。這可以防止在同一次執行中重複命中快取，顯著提升效能。

若要使用記憶化快取，請呼叫 `memo` 方法：

```php
use Illuminate\Support\Facades\Cache;

$value = Cache::memo()->get('key');
```

`memo` 方法可選擇地接受快取儲存區的名稱，用於指定記憶化驅動程式將會裝飾的底層快取儲存區：

```php
// Using the default cache store...
$value = Cache::memo()->get('key');

// Using the Redis cache store...
$value = Cache::memo('redis')->get('key');
```

給定鍵的第一次 `get` 呼叫會從您的快取儲存區中取回值，但後續在相同請求或任務中的呼叫將會從記憶體中取回值：

```php
// Hits the cache...
$value = Cache::memo()->get('key');

// Does not hit the cache, returns memoized value...
$value = Cache::memo()->get('key');
```

當呼叫修改快取值的方法（例如 `put`、`increment`、`remember` 等）時，記憶化快取會自動忘記記憶化值，並將變異方法呼叫委派給底層快取儲存區：

```php
Cache::memo()->put('name', 'Taylor'); // Writes to underlying cache...
Cache::memo()->get('name');           // Hits underlying cache...
Cache::memo()->get('name');           // Memoized, does not hit cache...

Cache::memo()->put('name', 'Tim');    // Forgets memoized value, writes new value...
Cache::memo()->get('name');           // Hits underlying cache again...
```


<a name="the-cache-helper"></a>
### 快取輔助函式

除了使用 `Cache` Facade 之外，您也可以使用全域 `cache` 函式來透過快取取回和儲存資料。當 `cache` 函式以單一字串引數呼叫時，它將會回傳給定鍵的值：

```php
$value = cache('key');
```

如果您提供鍵/值對的陣列和過期時間給此函式，它將會將值以指定時間儲存在快取中：

```php
cache(['key' => 'value'], $seconds);

cache(['key' => 'value'], now()->addMinutes(10));
```

當 `cache` 函式在沒有任何引數的情況下被呼叫時，它會回傳 `Illuminate\Contracts\Cache\Factory` 實作的一個實例，允許您呼叫其他快取方法：

```php
cache()->remember('users', $seconds, function () {
    return DB::table('users')->get();
});
```

> [!NOTE]
> 當測試對全域 `cache` 函式的呼叫時，您可以使用 `Cache::shouldReceive` 方法，就像您在[測試 Facade](/docs/{{version}}/mocking#mocking-facades) 時一樣。

<a name="cache-tags"></a>
## 快取標籤

> [!WARNING]
> 快取標籤在使用 `file`、`dynamodb` 或 `database` 快取驅動程式時不受支援。


<a name="storing-tagged-cache-items"></a>
### 儲存帶標籤的快取項目

快取標籤允許您為快取中的相關項目加上標籤，然後清除所有已指定標籤的快取值。您可以透過傳入一個有序的標籤名稱陣列來存取帶標籤的快取。例如，讓我們存取一個帶標籤的快取並將一個值 `put` 到快取中：

```php
use Illuminate\Support\Facades\Cache;

Cache::tags(['people', 'artists'])->put('John', $john, $seconds);
Cache::tags(['people', 'authors'])->put('Anne', $anne, $seconds);
```


<a name="accessing-tagged-cache-items"></a>
### 存取帶標籤的快取項目

透過標籤儲存的項目，若沒有同時提供用於儲存值的標籤，則無法存取。要取回一個帶標籤的快取項目，請將相同的有序標籤列表傳遞給 `tags` 方法，然後使用您希望取回的鍵呼叫 `get` 方法：

```php
$john = Cache::tags(['people', 'artists'])->get('John');

$anne = Cache::tags(['people', 'authors'])->get('Anne');
```


<a name="removing-tagged-cache-items"></a>
### 移除帶標籤的快取項目

您可以清除所有已指定一個或多個標籤的項目。例如，以下程式碼將移除所有標籤為 `people`、`authors` 或兩者皆有的快取。因此，`Anne` 和 `John` 都會從快取中移除：

```php
Cache::tags(['people', 'authors'])->flush();
```

相較之下，下方的程式碼將只移除標籤為 `authors` 的快取值，因此 `Anne` 將被移除，但 `John` 不會：

```php
Cache::tags('authors')->flush();
```


<a name="atomic-locks"></a>
## 原子鎖

> [!WARNING]
> 若要使用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動程式作為應用程式的預設快取驅動程式。此外，所有伺服器必須與相同的中央快取伺服器進行通訊。


<a name="managing-locks"></a>
### 管理鎖定

原子鎖允許操作分散式鎖定，無需擔心競爭條件。例如，[Laravel Cloud](https://cloud.laravel.com) 使用原子鎖來確保同一時間只有一個遠端任務在伺服器上執行。您可以使用 `Cache::lock` 方法來建立和管理鎖定：

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

如果在您請求時鎖定不可用，您可以指示 Laravel 等待指定的秒數。如果在指定時間限制內無法取得鎖定，將會拋出 `Illuminate\Contracts\Cache\LockTimeoutException` 異常：

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

上述範例可以透過將閉包傳遞給 `block` 方法來簡化。當閉包傳遞給此方法時，Laravel 將嘗試在指定秒數內取得鎖定，並在閉包執行後自動釋放鎖定：

```php
Cache::lock('foo', 10)->block(5, function () {
    // Lock acquired for 10 seconds after waiting a maximum of 5 seconds...
});
```


<a name="managing-locks-across-processes"></a>
### 跨程序管理鎖定

有時，您可能希望在一個程序中取得鎖定，並在另一個程序中釋放它。例如，您可以在一個網頁請求期間取得鎖定，並希望在該請求觸發的佇列任務結束時釋放該鎖定。在這種情況下，您應該將鎖定的限定範圍「擁有者 token」傳遞給佇列任務，以便該任務可以使用給定的 token 重新實例化鎖定。

在以下範例中，如果成功取得鎖定，我們將分派一個佇列任務。此外，我們將透過鎖定的 `owner` 方法將鎖定的擁有者 token 傳遞給佇列任務：

```php
$podcast = Podcast::find($id);

$lock = Cache::lock('processing', 120);

if ($lock->get()) {
    ProcessPodcast::dispatch($podcast, $lock->owner());
}
```

在應用程式的 `ProcessPodcast` 任務中，我們可以使用擁有者 token 來還原和釋放鎖定：

```php
Cache::restoreLock('processing', $this->owner)->release();
```

如果您希望在不考慮當前擁有者的情況下釋放鎖定，您可以使用 `forceRelease` 方法：

```php
Cache::lock('processing')->forceRelease();
```

<a name="adding-custom-cache-drivers"></a>
## 新增自訂快取驅動程式


<a name="writing-the-driver"></a>
### 編寫驅動程式

為了建立我們的自訂快取驅動程式，我們首先需要實作 `Illuminate\Contracts\Cache\Store` [契約](/docs/{{version}}/contracts)。因此，MongoDB 快取實作可能看起來像這樣：

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

我們只需要使用 MongoDB 連線來實作這些方法中的每一個。有關如何實作這些方法的範例，請參閱 [Laravel 框架原始碼](https://github.com/laravel/framework)中的 `Illuminate\Cache\MemcachedStore`。一旦我們的實作完成，我們就可以透過呼叫 `Cache` facade 的 `extend` 方法來完成自訂驅動程式的註冊：

```php
Cache::extend('mongo', function (Application $app) {
    return Cache::repository(new MongoStore);
});
```

> [!NOTE]
> 如果您想知道將自訂快取驅動程式程式碼放在哪裡，您可以在 `app` 目錄中建立一個 `Extensions` 命名空間。然而，請記住 Laravel 沒有嚴格的應用程式結構，您可以根據自己的偏好自由組織應用程式。


<a name="registering-the-driver"></a>
### 註冊驅動程式

為了在 Laravel 中註冊自訂快取驅動程式，我們將使用 `Cache` facade 上的 `extend` 方法。由於其他服務提供者可能會在其 `boot` 方法中嘗試讀取快取值，我們將在 `booting` 回呼中註冊我們的自訂驅動程式。透過使用 `booting` 回呼，我們可以確保在應用程式的服務提供者呼叫 `boot` 方法之前，以及在所有服務提供者呼叫 `register` 方法之後，註冊自訂驅動程式。我們將在應用程式的 `App\Providers\AppServiceProvider` 類別的 `register` 方法中註冊我們的 `booting` 回呼：

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

傳遞給 `extend` 方法的第一個參數是驅動程式的名稱。這將對應於您在 `config/cache.php` 設定檔中的 `driver` 選項。第二個參數是一個閉包 (closure)，它應該回傳一個 `Illuminate\Cache\Repository` 實例。該閉包將被傳遞一個 `$app` 實例，它是 [服務容器](/docs/{{version}}/container) 的一個實例。

一旦您的擴充功能註冊完成，請將應用程式 `config/cache.php` 設定檔中的 `CACHE_STORE` 環境變數或 `default` 選項更新為您擴充功能的名稱。


<a name="events"></a>
## 事件

若要執行每個快取操作上的程式碼，您可以監聽由快取分派的各種 [事件](/docs/{{version}}/events)：

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

為了提高效能，您可以透過將應用程式 `config/cache.php` 設定檔中特定快取儲存的 `events` 設定選項設定為 `false` 來禁用快取事件：

```php
'database' => [
    'driver' => 'database',
    // ...
    'events' => false,
],
```