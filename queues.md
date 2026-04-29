# 佇列 (Queues)

- [簡介](#introduction)
    - [連線 (Connections) vs. 佇列 (Queues)](#connections-vs-queues)
    - [驅動程式說明與先決條件](#driver-prerequisites)
- [建立任務 (Jobs)](#creating-jobs)
    - [產生任務類別](#generating-job-classes)
    - [類別結構](#class-structure)
    - [唯一任務 (Unique Jobs)](#unique-jobs)
    - [防抖動任務 (Debounced Jobs)](#debounced-jobs)
    - [加密任務](#encrypted-jobs)
- [任務中介層 (Middleware)](#job-middleware)
    - [速率限制 (Rate Limiting)](#rate-limiting)
    - [防止任務重疊](#preventing-job-overlaps)
    - [限制例外狀況頻率 (Throttling Exceptions)](#throttling-exceptions)
    - [跳過任務](#skipping-jobs)
- [分派任務](#dispatching-jobs)
    - [延遲分派](#delayed-dispatching)
    - [同步分派](#synchronous-dispatching)
    - [任務與資料庫交易](#jobs-and-database-transactions)
    - [任務鏈 (Job Chaining)](#job-chaining)
    - [自訂佇列與連線](#customizing-the-queue-and-connection)
    - [指定最大任務重試次數 / 超時時間](#max-job-attempts-and-timeout)
    - [SQS FIFO 與公平佇列 (Fair Queues)](#sqs-fifo-and-fair-queues)
    - [佇列容錯移轉 (Queue Failover)](#queue-failover)
    - [錯誤處理](#error-handling)
- [任務批次處理 (Job Batching)](#job-batching)
    - [定義可批次處理的任務](#defining-batchable-jobs)
    - [分派批次任務](#dispatching-batches)
    - [任務鏈與批次任務](#chains-and-batches)
    - [向批次中增加任務](#adding-jobs-to-batches)
    - [檢查批次](#inspecting-batches)
    - [取消批次](#cancelling-batches)
    - [批次失敗處理](#batch-failures)
    - [清理批次 (Pruning Batches)](#pruning-batches)
    - [將批次存儲在 DynamoDB](#storing-batches-in-dynamodb)
- [佇列化匿名函式 (Closures)](#queueing-closures)
- [執行佇列工作程式 (Queue Worker)](#running-the-queue-worker)
    - [`queue:work` 命令](#the-queue-work-command)
    - [佇列優先權](#queue-priorities)
    - [佇列工作程式與部署](#queue-workers-and-deployment)
    - [任務過期與超時](#job-expirations-and-timeouts)
    - [暫停與恢復佇列工作程式](#pausing-and-resuming-queue-workers)
- [Supervisor 設定](#supervisor-configuration)
- [處理失敗的任務](#dealing-with-failed-jobs)
    - [失敗任務後的清理](#cleaning-up-after-failed-jobs)
    - [重試失敗的任務](#retrying-failed-jobs)
    - [忽略遺失的模型](#ignoring-missing-models)
    - [清理失敗的任務](#pruning-failed-jobs)
    - [將失敗任務存儲在 DynamoDB](#storing-failed-jobs-in-dynamodb)
    - [停用失敗任務儲存](#disabling-failed-job-storage)
    - [失敗任務事件](#failed-job-events)
- [清除佇列中的任務](#clearing-jobs-from-queues)
- [監控您的佇列](#monitoring-your-queues)
- [測試](#testing)
    - [模擬 (Faking) 部分任務](#faking-a-subset-of-jobs)
    - [測試任務鏈](#testing-job-chains)
    - [測試任務批次](#testing-job-batches)
    - [測試任務與佇列的互動](#testing-job-queue-interactions)
- [任務事件](#job-events)

<a name="introduction"></a>
## 簡介

在開發網頁應用程式時，您可能會遇到一些任務（例如解析並儲存上傳的 CSV 檔案），這些任務在一般的網頁請求過程中需要花費太長時間執行。幸好， Laravel 讓您可以輕鬆建立佇列任務 (queued jobs)，以便在背景進行處理。透過將耗時的任務移至佇列，您的應用程式能以極快的速度回應網頁請求，並為您的客戶提供更好的使用者體驗。

Laravel 佇列為多種不同的佇列後端提供了一套統一的佇列 API，例如 [Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io)，甚至是關聯式資料庫。

Laravel 的佇列設定選項儲存在應用程式的 `config/queue.php` 設定檔中。在此檔案中，您會找到框架內含的各個佇列驅動程式的連線設定，包括資料庫、 [Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 與 [Beanstalkd](https://beanstalkd.github.io/) 驅動程式，以及一個會立即執行任務的同步驅動程式（用於開發或測試期間）。另外還包含一個 `null` 佇列驅動程式，它會直接捨棄已進入佇列的任務。

> [!NOTE]
> Laravel Horizon 是一個為您的 Redis 驅動佇列設計的精美儀表板與設定系統。請查看完整的 [Horizon 說明文件](/docs/{{version}}/horizon) 以取得更多資訊。

<a name="connections-vs-queues"></a>
### 連線 (Connections) vs. 佇列 (Queues)

在開始使用 Laravel 佇列之前，了解「連線 (connections)」與「佇列 (queues)」之間的區別非常重要。在您的 `config/queue.php` 設定檔中，有一個 `connections` 設定陣列。此選項定義了與後端佇列服務（如 Amazon SQS、Beanstalk 或 Redis）的連線。然而，任何指定的佇列連線都可能擁有多個「佇列」，您可以將其視為任務的不同堆疊或堆積。

請注意， `queue` 設定檔中的每個連線設定範例都包含一個 `queue` 屬性。這是任務在發送到特定連線時預設會被分派到的佇列。換句話說，如果您在分派任務時沒有明確定義該將其分派到哪個佇列，任務將會被放置在連線設定中 `queue` 屬性所定義的佇列上：

```php
use App\Jobs\ProcessPodcast;

// This job is sent to the default connection's default queue...
ProcessPodcast::dispatch();

// This job is sent to the default connection's "emails" queue...
ProcessPodcast::dispatch()->onQueue('emails');
```

某些應用程式可能永遠不需要將任務推送到多個佇列，而是偏好使用單一簡便的佇列。然而，將任務推送到多個佇列對於希望將任務處理方式進行優先排序或分段的應用程式來說特別有用，因為 Laravel 佇列工作程式允許您指定它應該按優先順序處理哪些佇列。例如，如果您將任務推送到 `high` 佇列，您可以執行一個給予其較高處理優先權的工作程式：

```shell
php artisan queue:work --queue=high,default
```

<a name="driver-prerequisites"></a>
### 驅動程式說明與先決條件

<a name="database"></a>
#### 資料庫

為了使用 `database` 佇列驅動程式，您需要一個資料庫資料表來存放任務。通常，這已包含在 Laravel 預設的 `0001_01_01_000002_create_jobs_table.php` [資料庫遷移 (database migration)](/docs/{{version}}/migrations) 中；然而，如果您的應用程式不包含此遷移，您可以使用 `make:queue-table` Artisan 命令來建立它：

```shell
php artisan make:queue-table

php artisan migrate
```

<a name="redis"></a>
#### Redis

為了使用 `redis` 佇列驅動程式，您應該在 `config/database.php` 設定檔中設定 Redis 資料庫連線。

> [!WARNING]
> `serializer` 與 `compression` Redis 選項不被 `redis` 佇列驅動程式支援。

<a name="redis-cluster"></a>
##### Redis Cluster

如果您的 Redis 佇列連線使用 [Redis Cluster](https://redis.io/docs/latest/operate/rs/databases/durability-ha/clustering)，則您的佇列名稱必須包含 [key hash tag](https://redis.io/docs/latest/develop/using-commands/keyspace/#hashtags)。這是為了確保特定佇列的所有 Redis 鍵 (Keys) 都被放置在相同的雜湊槽 (Hash slot) 中：

```php
'redis' => [
    'driver' => 'redis',
    'connection' => env('REDIS_QUEUE_CONNECTION', 'default'),
    'queue' => env('REDIS_QUEUE', '{default}'),
    'retry_after' => env('REDIS_QUEUE_RETRY_AFTER', 90),
    'block_for' => null,
    'after_commit' => false,
],
```

<a name="blocking"></a>
##### 阻塞 (Blocking)

使用 Redis 佇列時，您可以使用 `block_for` 設定選項來指定驅動程式在循環工作迴圈並重新輪詢 Redis 資料庫之前，應該等待任務可用的時間。

根據您的佇列負載調整此值，會比持續輪詢 Redis 資料庫以獲取新任務更有效率。例如，您可以將該值設定為 `5`，表示驅動程式在等待任務可用時應該阻塞 5 秒：

```php
'redis' => [
    'driver' => 'redis',
    'connection' => env('REDIS_QUEUE_CONNECTION', 'default'),
    'queue' => env('REDIS_QUEUE', 'default'),
    'retry_after' => env('REDIS_QUEUE_RETRY_AFTER', 90),
    'block_for' => 5,
    'after_commit' => false,
],
```

> [!WARNING]
> 將 `block_for` 設定為 `0` 會導致佇列工作程式無限期地阻塞，直到任務可用。這也會導致 `SIGTERM` 等訊號在下一個任務處理完畢之前無法被處理。

<a name="other-driver-prerequisites"></a>
#### 其他驅動程式先決條件

列出的佇列驅動程式需要以下依賴項目。這些依賴項目可以透過 Composer 套件管理員安裝：

<div class="content-list" markdown="1">

- Amazon SQS: `aws/aws-sdk-php ~3.0`
- Beanstalkd: `pda/pheanstalk ~5.0`
- Redis: `predis/predis ~2.0` 或 phpredis PHP 擴充功能
- [MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/queues/): `mongodb/laravel-mongodb`

</div>

<a name="creating-jobs"></a>
## 建立任務 (Jobs)

<a name="generating-job-classes"></a>
### 產生任務類別

預設情況下，應用程式中所有可被排入佇列的任務都儲存在 `app/Jobs` 目錄中。如果 `app/Jobs` 目錄不存在，當您執行 `make:job` Artisan 命令時，系統會自動建立該目錄：

```shell
php artisan make:job ProcessPodcast
```

產生的類別會實作 `Illuminate\Contracts\Queue\ShouldQueue` 介面，這是在告知 Laravel 該任務應被推送到佇列中非同步執行。

> [!NOTE]
> 任務 Stub 可使用 [Stub 發布](/docs/{{version}}/artisan#stub-customization) 功能進行自訂。

<a name="class-structure"></a>
### 類別結構

任務類別非常簡單，通常只包含一個在佇列處理任務時被呼叫的 `handle` 方法。首先，讓我們來看一個範例任務類別。在這個範例中，假設我們管理一個 Podcast 發布服務，需要在 Podcast 檔案發布前對其進行處理：

```php
<?php

namespace App\Jobs;

use App\Models\Podcast;
use App\Services\AudioProcessor;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

class ProcessPodcast implements ShouldQueue
{
    use Queueable;

    /**
     * Create a new job instance.
     */
    public function __construct(
        public Podcast $podcast,
    ) {}

    /**
     * Execute the job.
     */
    public function handle(AudioProcessor $processor): void
    {
        // Process uploaded podcast...
    }
}
```

在這個範例中，請注意我們可以將 [Eloquent 模型](/docs/{{version}}/eloquent) 直接傳入佇列任務的建構子。由於任務使用了 `Queueable` trait，Eloquent 模型及其已載入的關聯在任務執行時會被優雅地序列化與反序列化。

如果您的佇列任務在建構子中接收一個 Eloquent 模型，則只有該模型的識別碼會被序列化到佇列中。當任務實際被處理時，佇列系統會自動從資料庫重新取得完整的模型執行個體及其已載入的關聯。這種模型序列化的方法可以讓傳送到佇列驅動程式的任務酬載 (Payload) 小得多。

<a name="handle-method-dependency-injection"></a>
#### `handle` 方法的依賴注入

`handle` 方法是在任務被佇列處理時呼叫的。請注意，我們可以在任務的 `handle` 方法中對依賴進行型別提示 (Type-hint)。Laravel 的[服務容器 (Service Container)](/docs/{{version}}/container) 會自動注入這些依賴。

如果您想完全控制容器如何將依賴注入到 `handle` 方法中，可以使用容器的 `bindMethod` 方法。`bindMethod` 方法接收一個回呼函式，該函式會接收任務本身與容器。在回呼函式中，您可以隨意呼叫 `handle` 方法。通常，您應該在 `App\Providers\AppServiceProvider` [服務提供者(Service Providers)](/docs/{{version}}/providers) 的 `boot` 方法中呼叫此方法：

```php
use App\Jobs\ProcessPodcast;
use App\Services\AudioProcessor;
use Illuminate\Contracts\Foundation\Application;

$this->app->bindMethod([ProcessPodcast::class, 'handle'], function (ProcessPodcast $job, Application $app) {
    return $job->handle($app->make(AudioProcessor::class));
});
```

> [!WARNING]
> 二進位資料（例如原始圖片內容）在傳遞給佇列任務之前，應先通過 `base64_encode` 函式處理。否則，當任務被放入佇列時，可能無法正確序列化為 JSON。

<a name="handling-relationships"></a>
#### 佇列化關聯

由於所有已載入的 Eloquent 模型關聯在任務進入佇列時也會被序列化，序列化後的任務字串有時會變得很長。此外，當任務反序列化且從資料庫重新取得模型關聯時，它們會被完整地取得。在任務進入佇列過程的模型序列化之前所套用的任何關聯約束，在任務反序列化時都不會被套用。因此，如果您希望處理特定關聯的子集，則應在佇列任務中重新對該關聯加上約束。

或者，若要防止關聯被序列化，您可以在設定屬性值時對模型呼叫 `withoutRelations` 方法。此方法會回傳一個不含已載入關聯的模型執行個體：

```php
/**
 * Create a new job instance.
 */
public function __construct(
    Podcast $podcast,
) {
    $this->podcast = $podcast->withoutRelations();
}
```

如果您只需要移除特定關聯而保留其餘關聯，可以使用 `withoutRelation` 方法：

```php
$this->podcast = $podcast->withoutRelation('comments');
```

如果您正在使用 [PHP 建構子屬性升級 (Constructor Property Promotion)](https://www.php.net/manual/en/language.oop5.decon.php#language.oop5.decon.constructor.promotion)，且希望標明某個 Eloquent 模型不應將其關聯序列化，則可以使用 `WithoutRelations` 屬性 (Attribute)：

```php
use Illuminate\Queue\Attributes\WithoutRelations;

/**
 * Create a new job instance.
 */
public function __construct(
    #[WithoutRelations]
    public Podcast $podcast,
) {}
```

為了方便起見，如果您希望序列化所有模型且不包含任何關聯，可以將 `WithoutRelations` 屬性套用到整個類別，而不是套用到每個模型：

```php
<?php

namespace App\Jobs;

use App\Models\DistributionPlatform;
use App\Models\Podcast;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;
use Illuminate\Queue\Attributes\WithoutRelations;

#[WithoutRelations]
class ProcessPodcast implements ShouldQueue
{
    use Queueable;

    /**
     * Create a new job instance.
     */
    public function __construct(
        public Podcast $podcast,
        public DistributionPlatform $platform,
    ) {}
}
```

如果任務接收的是一個 Eloquent 模型的集合 (Collection) 或陣列，而不是單一模型，則該集合中的模型在任務反序列化並執行時，將不會恢復其關聯。這是為了防止在處理大量模型的任務時消耗過多資源。

<a name="unique-jobs"></a>
### 唯一任務 (Unique Jobs)

> [!WARNING]
> 唯一任務需要支援[鎖定 (Locks)](/docs/{{version}}/cache#atomic-locks)的快取驅動程式。目前 `memcached`、`redis`、`dynamodb`、`database`、`file` 與 `array` 快取驅動程式皆支援原子鎖 (Atomic locks)。

> [!WARNING]
> 唯一任務的限制不適用於批次 (Batches) 內的任務。

有時候，您可能希望確保在任何時間點，佇列中只有一個特定任務的執行體 (Instance)。您可以透過在任務類別中實作 `ShouldBeUnique` 介面來達成此目的。此介面不需要您在類別中定義任何額外的方法：

```php
<?php

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Queue\ShouldBeUnique;

class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    // ...
}
```

在上面的範例中，`UpdateSearchIndex` 任務是唯一的。因此，如果佇列中已經存在另一個該任務的執行體且尚未處理完成，則新的任務將不會被分派。

在某些情況下，您可能希望定義一個使任務唯一的特定「鍵 (Key)」，或者您可能希望指定一個超時時間，超過該時間後任務不再保持唯一。為此，您可以使用 `UniqueFor` 屬性並在任務類別中定義一個 `uniqueId` 方法：

```php
<?php

namespace App\Jobs;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Queue\ShouldBeUnique;
use Illuminate\Queue\Attributes\UniqueFor;

#[UniqueFor(3600)]
class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    /**
     * The product instance.
     *
     * @var \App\Models\Product
     */
    public $product;

    /**
     * Get the unique ID for the job.
     */
    public function uniqueId(): string
    {
        return $this->product->id;
    }
}
```
在上面的範例中，`UpdateSearchIndex` 任務會根據產品 ID 保持唯一。因此，任何具有相同產品 ID 的新分派任務都會被忽略，直到現有任務處理完成。此外，如果現有任務在一個小時內未被處理，唯一鎖定將會被釋放，另一個具有相同唯一鍵的任務就可以被分派到佇列中。

> [!WARNING]
> 如果您的應用程式從多個網頁伺服器或容器分派任務，您應確保所有伺服器都與同一個中央快取伺服器通訊，以便 Laravel 可以精確地判斷任務是否唯一。

<a name="keeping-jobs-unique-until-processing-begins"></a>
#### 在執行開始前保持任務唯一

預設情況下，唯一任務在處理完成或所有重試嘗試均失敗後才會「解鎖」。然而，在某些情況下，您可能希望任務在開始處理之前就立即解鎖。要實現這一點，您的任務應該實作 `ShouldBeUniqueUntilProcessing` 契約，而不是 `ShouldBeUnique` 契約：

```php
<?php

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Queue\ShouldBeUniqueUntilProcessing;

class UpdateSearchIndex implements ShouldQueue, ShouldBeUniqueUntilProcessing
{
    // ...
}
```

<a name="unique-job-locks"></a>
#### 唯一任務鎖定

在底層，當分派一個 `ShouldBeUnique` 任務時，Laravel 會嘗試獲取一個帶有 `uniqueId` 鍵的[鎖定 (Lock)](/docs/{{version}}/cache#atomic-locks)。如果鎖定已被佔用，則該任務不會被分派。當任務處理完成或所有重試嘗試均失敗時，此鎖定將會被釋放。預設情況下，Laravel 將使用預設的快取驅動程式來獲取此鎖定。但是，如果您希望使用另一個驅動程式來獲取鎖定，您可以定義一個 `uniqueVia` 方法，並回傳應使用的快取驅動程式：

```php
use Illuminate\Contracts\Cache\Repository;
use Illuminate\Support\Facades\Cache;

class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    // ...

    /**
     * Get the cache driver for the unique job lock.
     */
    public function uniqueVia(): Repository
    {
        return Cache::driver('redis');
    }
}
```

> [!NOTE]
> 如果您只需要限制任務的同時執行 (Concurrent Processing)，請改用 [WithoutOverlapping](/docs/{{version}}/queues#preventing-job-overlaps) 任務中介層。

<a name="debounced-jobs"></a>
### 防抖動任務 (Debounced Jobs)

有時候，您可能希望確保在短時間內多次分派同一個任務時，只有最後一次分派的任務會真正執行。您可以透過在任務中加入 `DebounceFor` 屬性來達成此目的：

```php
<?php

namespace App\Jobs;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;
use Illuminate\Queue\Attributes\DebounceFor;

#[DebounceFor(30)]
class UpdateSearchIndex implements ShouldQueue
{
    use Queueable;

    /**
     * Create a new job instance.
     */
    public function __construct(public int $productId)
    {
    }

    /**
     * Get the debounce ID for the job.
     */
    public function debounceId(): string
    {
        return (string) $this->productId;
    }
}
```

在上面的範例中，在 `30` 秒內重複為同一個產品分派 `UpdateSearchIndex` 將會防抖動該任務，使得只有最後一次分派會執行。

如果您想限制頻繁重新分派的任務可以被延遲多久，您可以為 `DebounceFor` 屬性提供 `maxWait` 引數：

```php
#[DebounceFor(30, maxWait: 120)]
class UpdateSearchIndex implements ShouldQueue
{
    use Queueable;

    // ...
}
```

您可以透過在任務中定義 `debounceVia` 方法來指定用於防抖動追蹤的快取儲存空間：

```php
use Illuminate\Contracts\Cache\Repository;
use Illuminate\Support\Facades\Cache;

public function debounceVia(): Repository
{
    return Cache::driver('redis');
}
```

如果一個防抖動任務被較新的分派所取代，Laravel 將會分派 `Illuminate\Queue\Events\JobDebounced` 事件，並從佇列中移除被取代的任務。

> [!WARNING]
> 防抖動任務與唯一任務是互斥的。使用 `DebounceFor` 屬性的任務不應實作 `ShouldBeUnique`。

> [!WARNING]
> 如果您的應用程式從多個網頁伺服器或容器分派防抖動任務，您應確保所有伺服器都與同一個中央快取伺服器通訊。

<a name="encrypted-jobs"></a>
### 加密任務

Laravel 允許您透過[加密](/docs/{{version}}/encryption)來確保任務資料的隱私性與完整性。要開始使用，只需將 `ShouldBeEncrypted` 介面加入到任務類別中。一旦將此介面加入類別，Laravel 就會在將您的任務推送到佇列之前自動對其進行加密：

```php
<?php

use Illuminate\Contracts\Queue\ShouldBeEncrypted;
use Illuminate\Contracts\Queue\ShouldQueue;

class UpdateSearchIndex implements ShouldQueue, ShouldBeEncrypted
{
    // ...
}
```

<a name="job-middleware"></a>
## 任務中介層 (Middleware)

任務中介層 (Middleware) 允許您在執行佇列任務時包裝自訂邏輯，減少任務本身的樣板程式碼。例如，請考慮以下 `handle` 方法，它利用 Laravel 的 Redis 速率限制功能，每五秒僅允許處理一個任務：

```php
use Illuminate\Support\Facades\Redis;

/**
 * Execute the job.
 */
public function handle(): void
{
    Redis::throttle('key')->block(0)->allow(1)->every(5)->then(function () {
        info('Lock obtained...');

        // Handle job...
    }, function () {
        // Could not obtain lock...

        return $this->release(5);
    });
}
```

雖然這段程式碼是有效的，但 `handle` 方法的實作會變得雜亂，因為它充斥著 Redis 速率限制邏輯。此外，這種速率限制邏輯必須針對我們想要限制速率的任何其他任務進行重複編寫。除了在 handle 方法中進行速率限制，我們也可以定義一個處理速率限制的任務中介層：

```php
<?php

namespace App\Jobs\Middleware;

use Closure;
use Illuminate\Support\Facades\Redis;

class RateLimited
{
    /**
     * Process the queued job.
     *
     * @param  \Closure(object): void  $next
     */
    public function handle(object $job, Closure $next): void
    {
        Redis::throttle('key')
            ->block(0)->allow(1)->every(5)
            ->then(function () use ($job, $next) {
                // Lock obtained...

                $next($job);
            }, function () use ($job) {
                // Could not obtain lock...

                $job->release(5);
            });
    }
}
```

如您所見，與 [路由中介層](/docs/{{version}}/middleware) 類似，任務中介層會接收正在處理的任務，以及一個應被叫用以繼續處理任務的回呼 (Callback)。

您可以使用 `make:job-middleware` Artisan 命令產生新的任務中介層類別。建立任務中介層後，可以透過任務的 `middleware` 方法回傳中介層來將其附加到任務中。`make:job` Artisan 命令產生的任務中並不包含此方法，因此您需要手動將其新增到您的任務類別中：

```php
use App\Jobs\Middleware\RateLimited;

/**
 * Get the middleware the job should pass through.
 *
 * @return array<int, object>
 */
public function middleware(): array
{
    return [new RateLimited];
}
```

> [!NOTE]
> 任務中介層也可以分配給 [佇列化事件監聽器](/docs/{{version}}/events#queued-event-listeners)、[Mailables](/docs/{{version}}/mail#queueing-mail) 與 [通知 (Notifications)](/docs/{{version}}/notifications#queueing-notifications)。

<a name="rate-limiting"></a>
### 速率限制 (Rate Limiting)

雖然我們剛才示範了如何撰寫自己的速率限制任務中介層，但 Laravel 實際上包含了一個您可以用來限制任務速率的速率限制中介層。與 [路由速率限制器](/docs/{{version}}/routing#defining-rate-limiters) 類似，任務速率限制器是使用 `RateLimiter` Facade 的 `for` 方法來定義的。

例如，您可能希望允許使用者每小時備份一次資料，而對尊榮客戶不設此限制。要實現這一點，您可以在 `AppServiceProvider` 的 `boot` 方法中定義一個 `RateLimiter`：

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    RateLimiter::for('backups', function (object $job) {
        return $job->user->vipCustomer()
            ? Limit::none()
            : Limit::perHour(1)->by($job->user->id);
    });
}
```

在上面的範例中，我們定義了每小時的速率限制；但是，您可以使用 `perMinute` 方法輕鬆定義基於分鐘的速率限制。此外，您可以將任何想要的值傳遞給速率限制的 `by` 方法；不過，這個值通常用於按客戶區分速率限制：

```php
return Limit::perMinute(50)->by($job->user->id);
```

定義速率限制後，您可以使用 `Illuminate\Queue\Middleware\RateLimited` 中介層將速率限制器附加到任務中。每當任務超過速率限制時，此中介層會根據速率限制持續時間將任務帶有適當延遲地釋放回佇列：

```php
use Illuminate\Queue\Middleware\RateLimited;

/**
 * Get the middleware the job should pass through.
 *
 * @return array<int, object>
 */
public function middleware(): array
{
    return [new RateLimited('backups')];
}
```

將受速率限制的任務釋放回佇列仍會增加任務的總 `attempts` (重試次數)。您可能需要相應地調整任務類別中的 `Tries` 和 `MaxExceptions` 屬性。或者，您可能希望使用 [retryUntil 方法](#time-based-attempts) 來定義不再嘗試任務之前的總時限。

使用 `releaseAfter` 方法，您也可以指定釋放後的任務在再次嘗試之前必須經過的秒數：

```php
/**
 * Get the middleware the job should pass through.
 *
 * @return array<int, object>
 */
public function middleware(): array
{
    return [(new RateLimited('backups'))->releaseAfter(60)];
}
```

如果您不希望在任務受速率限制時進行重試，可以使用 `dontRelease` 方法：

```php
/**
 * Get the middleware the job should pass through.
 *
 * @return array<int, object>
 */
public function middleware(): array
{
    return [(new RateLimited('backups'))->dontRelease()];
}
```

<a name="rate-limiting-with-redis"></a>
#### 使用 Redis 進行速率限制

如果您正在使用 Redis，則可以使用 `Illuminate\Queue\Middleware\RateLimitedWithRedis` 中介層，它專為 Redis 進行了最佳化，且比基礎的速率限制中介層更有效率：

```php
use Illuminate\Queue\Middleware\RateLimitedWithRedis;

public function middleware(): array
{
    return [new RateLimitedWithRedis('backups')];
}
```

`connection` 方法可用於指定中介層應使用的 Redis 連線：

```php
return [(new RateLimitedWithRedis('backups'))->connection('limiter')];
```

<a name="preventing-job-overlaps"></a>
### 防止任務重疊

Laravel 包含了一個 `Illuminate\Queue\Middleware\WithoutOverlapping` 中介層，讓您可以根據任意鍵 (Key) 來防止任務重疊。當佇列任務正在修改一個同一時間只能由一個任務修改的資源時，這非常有用。

舉例來說，假設您有一個更新使用者信用評分的佇列任務，且您想防止同一個使用者 ID 的信用評分更新任務發生重疊。為了達成這個目的，您可以從任務的 `middleware` 方法中回傳 `WithoutOverlapping` 中介層：

```php
use Illuminate\Queue\Middleware\WithoutOverlapping;

/**
 * Get the middleware the job should pass through.
 *
 * @return array<int, object>
 */
public function middleware(): array
{
    return [new WithoutOverlapping($this->user->id)];
}
```

將重疊的任務釋放回佇列仍會增加任務的總嘗試次數。您可能需要相應地調整任務類別中的 `Tries` 和 `MaxExceptions` 屬性。例如，若將 `Tries` 保留為預設值 1，將會阻止任何重疊的任務在稍後重試。

任何相同類型的重疊任務都會被釋放回佇列。您也可以指定釋放後的任務在再次嘗試前必須經過的秒數：

```php
/**
 * Get the middleware the job should pass through.
 *
 * @return array<int, object>
 */
public function middleware(): array
{
    return [(new WithoutOverlapping($this->order->id))->releaseAfter(60)];
}
```

如果您希望立即刪除任何重疊的任務而不進行重試，可以使用 `dontRelease` 方法：

```php
/**
 * Get the middleware the job should pass through.
 *
 * @return array<int, object>
 */
public function middleware(): array
{
    return [(new WithoutOverlapping($this->order->id))->dontRelease()];
}
```

`WithoutOverlapping` 中介層是由 Laravel 的原子鎖 (Atomic Lock) 功能所驅動。有時，您的任務可能會因非預期因素失敗或超時，導致鎖定未被釋放。因此，您可以使用 `expireAfter` 方法明確定義鎖定的過期時間。例如，下方的範例將指示 Laravel 在任務開始處理三分鐘後釋放 `WithoutOverlapping` 鎖定：

```php
/**
 * Get the middleware the job should pass through.
 *
 * @return array<int, object>
 */
public function middleware(): array
{
    return [(new WithoutOverlapping($this->order->id))->expireAfter(180)];
}
```

> [!WARNING]
> `WithoutOverlapping` 中介層需要支援[鎖定 (Locks)](/docs/{{version}}/cache#atomic-locks)的快取驅動程式。目前 `memcached`、`redis`、`dynamodb`、`database`、`file` 與 `array` 快取驅動程式皆支援原子鎖。

<a name="sharing-lock-keys"></a>
#### 跨任務類別共享鎖定鍵 (Lock Keys)

預設情況下，`WithoutOverlapping` 中介層僅會防止相同類別的任務重疊。因此，即使兩個不同的任務類別使用相同的鎖定鍵，它們也不會被阻止重疊。不過，您可以使用 `shared` 方法指示 Laravel 跨任務類別套用該鍵：

```php
use Illuminate\Queue\Middleware\WithoutOverlapping;

class ProviderIsDown
{
    // ...

    public function middleware(): array
    {
        return [
            (new WithoutOverlapping("status:{$this->provider}"))->shared(),
        ];
    }
}

class ProviderIsUp
{
    // ...

    public function middleware(): array
    {
        return [
            (new WithoutOverlapping("status:{$this->provider}"))->shared(),
        ];
    }
}
```

<a name="throttling-exceptions"></a>
### 限制例外狀況頻率 (Throttling Exceptions)

Laravel 包含了一個 `Illuminate\Queue\Middleware\ThrottlesExceptions` 中介層，讓您可以限制例外狀況的頻率。一旦任務拋出特定數量的例外狀況，所有後續執行該任務的嘗試都會延遲，直到指定的時間間隔過後為止。此中介層對於與不穩定的第三方服務進行互動的任務特別有用。

例如，假設有一個與第三方 API 互動的佇列任務開始拋出例外狀況。若要限制例外狀況的頻率，您可以從任務的 `middleware` 方法中回傳 `ThrottlesExceptions` 中介層。通常，此中介層應與實作了 [基於時間的重試次數](#time-based-attempts) 的任務搭配使用：

```php
use DateTime;
use Illuminate\Queue\Middleware\ThrottlesExceptions;

/**
 * Get the middleware the job should pass through.
 *
 * @return array<int, object>
 */
public function middleware(): array
{
    return [new ThrottlesExceptions(10, 5 * 60)];
}

/**
 * Determine the time at which the job should timeout.
 */
public function retryUntil(): DateTime
{
    return now()->plus(minutes: 30);
}
```

該中介層接受的第一個建構子引數是任務在受限之前可以拋出的例外狀況數量，而第二個建構子引數是任務受限後，再次嘗試該任務之前應經過的秒數。在上述程式碼範例中，如果任務連續拋出 10 次例外狀況，我們將等待 5 分鐘後再嘗試該任務，且受限於 30 分鐘的時間限制。

當任務拋出例外狀況但尚未達到例外狀況閾值時，任務通常會立即重試。但是，您可以在將中介層附加到任務時，透過呼叫 `backoff` 方法來指定此類任務應延遲的分鐘數：

```php
use Illuminate\Queue\Middleware\ThrottlesExceptions;

/**
 * Get the middleware the job should pass through.
 *
 * @return array<int, object>
 */
public function middleware(): array
{
    return [(new ThrottlesExceptions(10, 5 * 60))->backoff(5)];
}
```

在內部，此中介層使用 Laravel 的快取系統來實作速率限制，並將任務的類別名稱作為快取「鍵 (Key)」。您可以在將中介層附加到任務時，透過呼叫 `by` 方法來覆寫此鍵。如果您有多個任務與同一個第三方服務互動，並且希望它們共享同一個頻率限制「貯存槽 (Bucket)」，以確保它們遵守單一共享限制，這會非常有用：

```php
use Illuminate\Queue\Middleware\ThrottlesExceptions;

/**
 * Get the middleware the job should pass through.
 *
 * @return array<int, object>
 */
public function middleware(): array
{
    return [(new ThrottlesExceptions(10, 10 * 60))->by('key')];
}
```

預設情況下，此中介層會限制所有的例外狀況。您可以透過在將中介層附加到任務時叫用 `when` 方法來修改此行為。只有當提供給 `when` 方法的匿名函式回傳 `true` 時，該例外狀況才會被限制：

```php
use Illuminate\Http\Client\HttpClientException;
use Illuminate\Queue\Middleware\ThrottlesExceptions;

/**
 * Get the middleware the job should pass through.
 *
 * @return array<int, object>
 */
public function middleware(): array
{
    return [(new ThrottlesExceptions(10, 10 * 60))->when(
        fn (Throwable $throwable) => $throwable instanceof HttpClientException
    )];
}
```

與 `when` 方法不同（該方法會將任務釋放回佇列或拋出例外狀況），`deleteWhen` 方法允許您在發生特定例外狀況時完全刪除該任務：

```php
use App\Exceptions\CustomerDeletedException;
use Illuminate\Queue\Middleware\ThrottlesExceptions;

/**
 * Get the middleware the job should pass through.
 *
 * @return array<int, object>
 */
public function middleware(): array
{
    return [(new ThrottlesExceptions(2, 10 * 60))->deleteWhen(CustomerDeletedException::class)];
}
```

如果您希望將受限制的例外狀況回報給應用程式的例外狀況處理器，可以在將中介層附加到任務時叫用 `report` 方法。您可以選擇性地提供一個匿名函式給 `report` 方法，且只有在該匿名函式回傳 `true` 時，才會回報該例外狀況：

```php
use Illuminate\Http\Client\HttpClientException;
use Illuminate\Queue\Middleware\ThrottlesExceptions;

/**
 * Get the middleware the job should pass through.
 *
 * @return array<int, object>
 */
public function middleware(): array
{
    return [(new ThrottlesExceptions(10, 10 * 60))->report(
        fn (Throwable $throwable) => $throwable instanceof HttpClientException
    )];
}
```

<a name="throttling-exceptions-with-redis"></a>
#### 使用 Redis 限制例外狀況頻率

如果您正在使用 Redis，可以使用 `Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis` 中介層，它專為 Redis 進行了最佳化，比基礎的例外狀況限制中介層更有效率：

```php
use Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis;

public function middleware(): array
{
    return [new ThrottlesExceptionsWithRedis(10, 10 * 60)];
}
```

`connection` 方法可用於指定中介層應使用的 Redis 連線：

```php
return [(new ThrottlesExceptionsWithRedis(10, 10 * 60))->connection('limiter')];
```

<a name="skipping-jobs"></a>
### 跳過任務

`Skip` 中介層允許您指定應跳過或刪除任務，而無需修改任務本身的邏輯。`Skip::when` 方法會在給定條件評估為 `true` 時刪除任務，而 `Skip::unless` 方法會在條件評估為 `false` 時刪除任務：

```php
use Illuminate\Queue\Middleware\Skip;

/**
 * Get the middleware the job should pass through.
 */
public function middleware(): array
{
    return [
        Skip::when($condition),
    ];
}
```

您也可以將 `Closure` (匿名函式) 傳遞給 `when` 和 `unless` 方法，以進行更複雜的條件評估：

```php
use Illuminate\Queue\Middleware\Skip;

/**
 * Get the middleware the job should pass through.
 */
public function middleware(): array
{
    return [
        Skip::when(function (): bool {
            return $this->shouldSkip();
        }),
    ];
}
```

<a name="dispatching-jobs"></a>
## 分派任務

一旦您寫好了任務類別，就可以使用任務本身的 `dispatch` 方法來分派它。傳遞給 `dispatch` 方法的引數將會被提供給任務的建構子：

```php
<?php

namespace App\Http\Controllers;

use App\Jobs\ProcessPodcast;
use App\Models\Podcast;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class PodcastController extends Controller
{
    /**
     * Store a new podcast.
     */
    public function store(Request $request): RedirectResponse
    {
        $podcast = Podcast::create(/* ... */);

        // ...

        ProcessPodcast::dispatch($podcast);

        return redirect('/podcasts');
    }
}
```

如果您想根據條件分派任務，可以使用 `dispatchIf` 和 `dispatchUnless` 方法：

```php
ProcessPodcast::dispatchIf($accountActive, $podcast);

ProcessPodcast::dispatchUnless($accountSuspended, $podcast);
```

在新的 Laravel 應用程式中，`database` 連線被定義為預設佇列。您可以透過更改應用程式 `.env` 檔案中的 `QUEUE_CONNECTION` 環境變數來指定不同的預設佇列連線。

<a name="delayed-dispatching"></a>
### 延遲分派

如果您想指定任務不應立即讓佇列工作程式處理，可以在分派任務時使用 `delay` 方法。例如，讓我們指定一個任務在分派 10 分鐘後才能被處理：

```php
<?php

namespace App\Http\Controllers;

use App\Jobs\ProcessPodcast;
use App\Models\Podcast;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class PodcastController extends Controller
{
    /**
     * Store a new podcast.
     */
    public function store(Request $request): RedirectResponse
    {
        $podcast = Podcast::create(/* ... */);

        // ...

        ProcessPodcast::dispatch($podcast)
            ->delay(now()->plus(minutes: 10));

        return redirect('/podcasts');
    }
}
```

在某些情況下，任務可能已經設定了預設延遲。如果您需要跳過此延遲並立即處理任務，可以使用 `withoutDelay` 方法：

```php
ProcessPodcast::dispatch($podcast)->withoutDelay();
```

> [!WARNING]
> Amazon SQS 佇列服務的最大延遲時間為 15 分鐘。

<a name="synchronous-dispatching"></a>
### 同步分派

如果您想立即（同步）分派任務，可以使用 `dispatchSync` 方法。使用此方法時，任務將不會被排入佇列，而是立即在當前行程中執行：

```php
<?php

namespace App\Http\Controllers;

use App\Jobs\ProcessPodcast;
use App\Models\Podcast;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class PodcastController extends Controller
{
    /**
     * Store a new podcast.
     */
    public function store(Request $request): RedirectResponse
    {
        $podcast = Podcast::create(/* ... */);

        // Create podcast...

        ProcessPodcast::dispatchSync($podcast);

        return redirect('/podcasts');
    }
}
```

<a name="deferred-dispatching"></a>
#### 延期分派

藉由使用延期同步分派，您可以將任務分派為在當前行程中處理，但在 HTTP 回應發送給使用者之後執行。這讓您可以同步處理「佇列」任務，而不會減慢使用者的應用程式體驗。若要延後同步任務的執行，請將任務分派到 `deferred` 連線：

```php
RecordDelivery::dispatch($order)->onConnection('deferred');
```

`deferred` 連線也作為預設的 [容錯移轉佇列 (failover queue)](#queue-failover)。

同樣地，`background` 連線會在 HTTP 回應發送給使用者之後處理任務；不過，任務是在一個獨立啟動的 PHP 行程 (process) 中處理的，這讓 PHP-FPM / 應用程式工作程式可以騰出空間來處理另一個傳入的 HTTP 請求：

```php
RecordDelivery::dispatch($order)->onConnection('background');
```

<a name="jobs-and-database-transactions"></a>
### 任務與資料庫交易

雖然在資料庫交易中分派任務完全沒有問題，但您應該特別注意確保您的任務實際上能夠成功執行。在交易中分派任務時，任務可能會在父層交易提交之前就被工作程式處理。當這種情況發生時，您在資料庫交易期間對模型或資料庫紀錄所做的任何更新，可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫紀錄也可能不存在於資料庫中。

幸好，Laravel 提供了幾種解決此問題的方法。首先，您可以在佇列連線的設定陣列中設定 `after_commit` 連線選項：

```php
'redis' => [
    'driver' => 'redis',
    // ...
    'after_commit' => true,
],
```

當 `after_commit` 選項為 `true` 時，您可以在資料庫交易中分派任務；不過，Laravel 會等到開啟的父層資料庫交易提交後，才真正分派任務。當然，如果當前沒有開啟任何資料庫交易，任務將會立即分派。

如果交易因交易期間發生的例外狀況而回滾，則在該交易期間分派的任務將被捨棄。

> [!NOTE]
> 將 `after_commit` 設定選項設為 `true` 也會導致任何佇列化的事件監聽器、郵件 (mailables)、通知以及廣播事件，在所有開啟的資料庫交易提交後才被分派。

<a name="specifying-commit-dispatch-behavior-inline"></a>
#### 行內指定提交分派行為

如果您沒有將 `after_commit` 佇列連線設定選項設為 `true`，您仍然可以指示特定任務應在所有開啟的資料庫交易提交後才分派。若要達成此目的，您可以在分派操作上串接 `afterCommit` 方法：

```php
use App\Jobs\ProcessPodcast;

ProcessPodcast::dispatch($podcast)->afterCommit();
```

同樣地，如果 `after_commit` 設定選項設為 `true`，您可以指示特定任務應立即分派，而不必等待任何開啟的資料庫交易提交：

```php
ProcessPodcast::dispatch($podcast)->beforeCommit();
```

<a name="job-chaining"></a>
### 任務鏈 (Job Chaining)

任務鏈 (Job Chaining) 讓您可以指定一系列的佇列任務，在主任務成功執行後依序運作。若序列中的其中一個任務失敗，剩餘的任務將不會被執行。若要執行佇列任務鏈，您可以使用 `Bus` Facade 提供的 `chain` 方法。Laravel 的命令匯流排 (Command Bus) 是建構佇列任務分派功能的底層元件：

```php
use App\Jobs\OptimizePodcast;
use App\Jobs\ProcessPodcast;
use App\Jobs\ReleasePodcast;
use Illuminate\Support\Facades\Bus;

Bus::chain([
    new ProcessPodcast,
    new OptimizePodcast,
    new ReleasePodcast,
])->dispatch();
```

除了串接任務類別的實例，您也可以串接匿名函式 (Closures)：

```php
Bus::chain([
    new ProcessPodcast,
    new OptimizePodcast,
    function () {
        Podcast::update(/* ... */);
    },
])->dispatch();
```

> [!WARNING]
> 在任務中使用 `$this->delete()` 方法刪除任務並不會阻止後續串接的任務執行。只有當任務鏈中的某個任務失敗時，該任務鏈才會停止執行。


<a name="chain-connection-queue"></a>
#### 任務鏈的連線與佇列

如果您想為串接的任務指定應使用的連線與佇列，可以使用 `onConnection` 與 `onQueue` 方法。這些方法會指定任務鏈使用的佇列連線與佇列名稱，除非該佇列任務本身有顯式指定不同的連線或佇列：

```php
Bus::chain([
    new ProcessPodcast,
    new OptimizePodcast,
    new ReleasePodcast,
])->onConnection('redis')->onQueue('podcasts')->dispatch();
```


<a name="adding-jobs-to-the-chain"></a>
#### 向任務鏈中新增任務

有時候，您可能需要在任務鏈中的某個任務內部，向現有的任務鏈前端或後端新增任務。您可以透過 `prependToChain` 與 `appendToChain` 方法來達成：

```php
/**
 * Execute the job.
 */
public function handle(): void
{
    // ...

    // Prepend to the current chain, run job immediately after current job...
    $this->prependToChain(new TranscribePodcast);

    // Append to the current chain, run job at end of chain...
    $this->appendToChain(new TranscribePodcast);
}
```


<a name="chain-failures"></a>
#### 任務鏈失敗處理

在串接任務時，您可以使用 `catch` 方法來指定一個匿名函式，當任務鏈中的某個任務失敗時會被調用。該回呼函式會接收導致任務失敗的 `Throwable` 實例：

```php
use Illuminate\Support\Facades\Bus;
use Throwable;

Bus::chain([
    new ProcessPodcast,
    new OptimizePodcast,
    new ReleasePodcast,
])->catch(function (Throwable $e) {
    // A job within the chain has failed...
})->dispatch();
```

> [!WARNING]
> 由於任務鏈的回呼函式會被序列化並由 Laravel 佇列在稍後執行，因此您不應在任務鏈回呼函式中使用 `$this` 變數。


<a name="customizing-the-queue-and-connection"></a>
### 自訂佇列與連線


<a name="dispatching-to-a-particular-queue"></a>
#### 分派到特定佇列

透過將任務推送到不同的佇列，您可以為佇列任務進行「分類」，甚至可以根據不同佇列優先權來分配工作程式的數量。請記住，這並不是將任務推送到佇列設定檔中定義的不同「連線 (Connections)」，而是將其推送到單一連線中的特定佇列。若要指定佇列，請在分派任務時使用 `onQueue` 方法：

```php
<?php

namespace App\Http\Controllers;

use App\Jobs\ProcessPodcast;
use App\Models\Podcast;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class PodcastController extends Controller
{
    /**
     * Store a new podcast.
     */
    public function store(Request $request): RedirectResponse
    {
        $podcast = Podcast::create(/* ... */);

        // Create podcast...

        ProcessPodcast::dispatch($podcast)->onQueue('processing');

        return redirect('/podcasts');
    }
}
```

或者，您也可以在任務的建構子中調用 `onQueue` 方法來指定任務的佇列：

```php
<?php

namespace App\Jobs;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

class ProcessPodcast implements ShouldQueue
{
    use Queueable;

    /**
     * Create a new job instance.
     */
    public function __construct()
    {
        $this->onQueue('processing');
    }
}
```


<a name="dispatching-to-a-particular-connection"></a>
#### 分派到特定連線

如果您的應用程式會與多個佇列連線互動，您可以使用 `onConnection` 方法指定要將任務推送到哪個連線：

```php
<?php

namespace App\Http\Controllers;

use App\Jobs\ProcessPodcast;
use App\Models\Podcast;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class PodcastController extends Controller
{
    /**
     * Store a new podcast.
     */
    public function store(Request $request): RedirectResponse
    {
        $podcast = Podcast::create(/* ... */);

        // Create podcast...

        ProcessPodcast::dispatch($podcast)->onConnection('sqs');

        return redirect('/podcasts');
    }
}
```

您可以串聯使用 `onConnection` 與 `onQueue` 方法來為任務指定連線與佇列：

```php
ProcessPodcast::dispatch($podcast)
    ->onConnection('sqs')
    ->onQueue('processing');
```

或者，您也可以在任務的建構子中調用 `onConnection` 方法來指定任務的連線：

```php
<?php

namespace App\Jobs;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

class ProcessPodcast implements ShouldQueue
{
    use Queueable;

    /**
     * Create a new job instance.
     */
    public function __construct()
    {
        $this->onConnection('sqs');
    }
}
```


<a name="queue-routing"></a>
#### 佇列路由

您可以使用 `Queue` Facade 的 `route` 方法為特定的任務類別定義預設連線與佇列。當您想要確保某些任務總是使用特定的佇列，而不想在分派時逐一指定連線或佇列時，這非常有用。

除了路由特定的任務類別外，您還可以將介面 (Interface)、Trait 或父類別傳遞給 `route` 方法。當您這樣做時，任何實作該介面、使用該 Trait 或繼承該父類別的任務都會自動使用設定好的連線與佇列。

通常，您應該在服務提供者(Service Providers)的 `boot` 方法中調用 `route` 方法：

```php
use App\Concerns\RequiresVideo;
use App\Jobs\ProcessPodcast;
use App\Jobs\ProcessVideo;
use Illuminate\Support\Facades\Queue;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Queue::route(ProcessPodcast::class, connection: 'redis', queue: 'podcasts');
    Queue::route(RequiresVideo::class, queue: 'video');
}
```

如果在指定連線時沒有指定佇列，任務將會被傳送到預設佇列：

```php
Queue::route(ProcessPodcast::class, connection: 'redis');
```

您也可以透過向 `route` 方法傳遞陣列來一次路由多個任務類別：

```php
Queue::route([
    ProcessPodcast::class => ['podcasts', 'redis'], // Queue and connection
    ProcessVideo::class => 'videos', // Queue only (uses default connection)
]);
```

> [!NOTE]
> 佇列路由仍可在個別任務中被覆寫。

<a name="max-job-attempts-and-timeout"></a>
### 指定最大任務重試次數 / 超時時間

<a name="max-attempts"></a>
#### 最大嘗試次數 (Max Attempts)

任務嘗試次數 (Job attempts) 是 Laravel 佇列系統的核心概念，並驅動了許多進階功能。雖然這在一開始可能看起來很令人困惑，但在修改預設設定之前，瞭解它們的運作方式非常重要。

當任務被分派時，它會被推送到佇列中。接著一個工作程式會取出該任務並嘗試執行它。這就是一次任務嘗試。

然而，一次嘗試並不一定意味著任務的 `handle` 方法已被執行。嘗試也可以透過以下幾種方式被「消耗」：

<div class="content-list" markdown="1">

- 任務在執行過程中遇到未處理的例外狀況。
- 任務使用 `$this->release()` 手動釋放回佇列。
- 中介層（如 `WithoutOverlapping` 或 `RateLimited`）無法取得鎖定並釋放任務。
- 任務超時。
- 任務的 `handle` 方法執行完成且未拋出例外狀況。

</div>

您可能不希望無限期地持續嘗試某個任務。因此，Laravel 提供多種方式來指定任務可以嘗試的次數或時長。

> [!NOTE]
> 預設情況下，Laravel 只會嘗試任務一次。如果您的任務使用 `WithoutOverlapping` 或 `RateLimited` 等中介層，或者您正手動釋放任務，您可能需要透過 `tries` 選項增加允許的嘗試次數。

指定任務最大嘗試次數的一種方法是透過 Artisan 命令列上的 `--tries` 切換參數。這將適用於該工作程式處理的所有任務，除非正在處理的任務本身指定了可嘗試次數：

```shell
php artisan queue:work --tries=3
```

如果任務超過其最大嘗試次數，它將被視為「失敗」的任務。有關處理失敗任務的更多資訊，請參閱[處理失敗任務的說明文件](#dealing-with-failed-jobs)。如果將 `--tries=0` 提供給 `queue:work` 命令，任務將無限期重試。

您可以透過在任務類別本身使用 `Tries` 屬性定義任務的最大嘗試次數，採取更精細的方法。如果在任務上指定了最大嘗試次數，它的優先級將高於命令列上提供的 `--tries` 值：

```php
<?php

namespace App\Jobs;

use Illuminate\Queue\Attributes\Tries;

#[Tries(5)]
class ProcessPodcast implements ShouldQueue
{
    // ...
}
```

如果您需要動態控制特定任務的最大嘗試次數，可以在任務中定義一個 `tries` 方法：

```php
/**
 * Determine number of times the job may be attempted.
 */
public function tries(): int
{
    return 5;
}
```

<a name="time-based-attempts"></a>
#### 基於時間的嘗試 (Time Based Attempts)

除了定義任務在失敗前可以嘗試的次數外，您還可以定義任務不再被嘗試的時間點。這允許任務在給定的時間範圍內嘗試任意次數。要定義任務不再被嘗試的時間點，請在任務類別中新增一個 `retryUntil` 方法。該方法應回傳一個 `DateTime` 執行個體：

```php
use DateTime;

/**
 * Determine the time at which the job should timeout.
 */
public function retryUntil(): DateTime
{
    return now()->plus(minutes: 10);
}
```

如果同時定義了 `retryUntil` 和 `tries`，Laravel 會優先採用 `retryUntil` 方法。

> [!NOTE]
> 您也可以在[佇列化的事件監聽器](/docs/{{version}}/events#queued-event-listeners)和[佇列化的通知](/docs/{{version}}/notifications#queueing-notifications)中定義 `Tries` 屬性或 `retryUntil` 方法。

<a name="max-exceptions"></a>
#### 最大例外狀況次數 (Max Exceptions)

有時您可能希望指定任務可以嘗試多次，但如果重試是由給定數量的未處理例外狀況觸發的（而非直接透過 `release` 方法釋放），則應使其失敗。要實現這一點，您可以在任務類別上使用 `Tries` 和 `MaxExceptions` 屬性：

```php
<?php

namespace App\Jobs;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;
use Illuminate\Queue\Attributes\MaxExceptions;
use Illuminate\Queue\Attributes\Tries;
use Illuminate\Support\Facades\Redis;

#[Tries(25)]
#[MaxExceptions(3)]
class ProcessPodcast implements ShouldQueue
{
    use Queueable;

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        Redis::throttle('key')->allow(10)->every(60)->then(function () {
            // Lock obtained, process the podcast...
        }, function () {
            // Unable to obtain lock...
            return $this->release(10);
        });
    }
}
```

在此範例中，如果應用程式無法取得 Redis 鎖定，任務將被釋放十秒鐘，並將持續重試最多 25 次。然而，如果任務拋出三次未處理的例外狀況，任務將會失敗。

<a name="timeout"></a>
#### 超時 (Timeout)

通常，您大致知道預期佇列任務需要多長時間。因此，Laravel 允許您指定一個「超時 (timeout)」值。預設情況下，超時值為 60 秒。如果任務處理時間超過超時值指定的秒數，處理該任務的工作程式將帶著錯誤結束。通常，工作程式會由您[伺服器上設定的行程管理員](#supervisor-configuration)自動重新啟動。

任務可以執行的最大秒數可以使用 Artisan 命令列上的 `--timeout` 切換參數來指定：

```shell
php artisan queue:work --timeout=30
```

如果任務因持續超時而超過其最大嘗試次數，它將被標記為失敗。

您也可以使用任務類別上的 `Timeout` 屬性來定義任務允許執行的最大秒數。如果在任務上指定了超時時間，它的優先級將高於命令列上指定的任何超時時間：

```php
<?php

namespace App\Jobs;

use Illuminate\Queue\Attributes\Timeout;

#[Timeout(120)]
class ProcessPodcast implements ShouldQueue
{
    // ...
}
```

有時，通訊端 (Sockets) 或外送 HTTP 連線等 IO 阻塞程序可能不遵循您指定的超時時間。因此，在使用這些功能時，您也應該始終嘗試使用它們的 API 來指定超時時間。例如，使用 [Guzzle](https://docs.guzzlephp.org) 時，您應該始終指定連線和請求的超時值。

> [!WARNING]
> 必須安裝 [PCNTL](https://www.php.net/manual/en/book.pcntl.php) PHP 擴充功能才能指定任務超時。此外，任務的「超時」值應始終小於其[「重試時間 (retry after)」](#job-expiration)值。否則，任務可能會在實際執行完成或超時之前就重新嘗試執行。

<a name="failing-on-timeout"></a>
#### 超時即失敗 (Failing on Timeout)

如果您希望表示任務在超時時應被標記為[失敗](#dealing-with-failed-jobs)，您可以在任務類別上使用 `FailOnTimeout` 屬性：

```php
<?php

namespace App\Jobs;

use Illuminate\Queue\Attributes\FailOnTimeout;

#[FailOnTimeout]
class ProcessPodcast implements ShouldQueue
{
    // ...
}
```

> [!NOTE]
> 預設情況下，當任務超時時，它會消耗一次嘗試次數並被釋放回佇列（如果允許重試）。然而，如果您將任務配置為超時即失敗，則無論 tries 設定為何值，它都不會被重試。

<a name="sqs-fifo-and-fair-queues"></a>
### SQS FIFO 與公平佇列 (Fair Queues)

Laravel 支援 [Amazon SQS FIFO (先進先出)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html) 與 [公平 (fair)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fair-queues.html) 佇列。FIFO 佇列允許您依照發送的確切順序處理任務，同時透過訊息重複刪除 (Deduplication) 確保任務僅被處理一次。

FIFO 佇列需要訊息群組 ID (Message Group ID) 來判斷哪些任務可以平行處理。具有相同群組 ID 的任務會按順序處理，而具有不同群組 ID 的訊息則可以同時執行。

Laravel 提供了一個流暢的 `onGroup` 方法，用於在分派任務時指定訊息群組 ID：

```php
ProcessOrder::dispatch($order)
    ->onGroup("customer-{$order->customer_id}");
```

SQS FIFO 佇列支援訊息重複刪除，以確保僅被處理一次。請在您的任務類別中實作 `deduplicationId` 方法，以提供自訂的重複刪除 ID：

```php
<?php

namespace App\Jobs;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

class ProcessSubscriptionRenewal implements ShouldQueue
{
    use Queueable;

    // ...

    /**
     * Get the job's deduplication ID.
     */
    public function deduplicationId(): string
    {
        return "renewal-{$this->subscription->id}";
    }
}
```

<a name="fair-queues"></a>
#### 公平佇列 (Fair Queues)

如果您使用的是 SQS 標準佇列，設定訊息群組即可啟用公平佇列。換句話說，一旦您分配了群組，SQS 就會使用它們來維持多個租戶 (Tenants) 或工作負載 (Workloads) 之間的公平派送。不需要額外的 Laravel 設定。

除了在分派時呼叫 `onGroup` 外，您也可以直接在任務中定義 `messageGroup` 方法：

```php
<?php

namespace App\Jobs;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

class ProcessOrder implements ShouldQueue
{
    use Queueable;

    // ...

    /**
     * Get the job's message group.
     */
    public function messageGroup(): string
    {
        return "customer-{$this->order->customer_id}";
    }
}
```

<a name="fifo-listeners-mail-and-notifications"></a>
#### FIFO 監聽器、郵件與通知

使用 FIFO 佇列時，您還需要在監聽器、郵件和通知上定義訊息群組。或者，您可以將這些物件的佇列實例分派到非 FIFO 佇列。

若要為 [佇列事件監聽器](/docs/{{version}}/events#queued-event-listeners) 定義訊息群組，請在監聽器上定義 `messageGroup` 方法。您也可以選擇性地定義 `deduplicationId` 方法：

```php
<?php

namespace App\Listeners;

class SendShipmentNotification
{
    // ...

    /**
     * Get the job's message group.
     */
    public function messageGroup(): string
    {
        return 'shipments';
    }

    /**
     * Get the job's deduplication ID.
     */
    public function deduplicationId(): string
    {
        return "shipment-notification-{$this->shipment->id}";
    }
}
```

當發送將要在 FIFO 佇列上排隊的 [郵件訊息](/docs/{{version}}/mail) 時，您應該在發送通知時呼叫 `onGroup` 方法，並可以選擇性地使用 `withDeduplicator` 方法：

```php
use App\Mail\InvoicePaid;
use Illuminate\Support\Facades\Mail;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

Mail::to($request->user())->send($invoicePaid);
```

當發送將要在 FIFO 佇列上排隊的 [通知](/docs/{{version}}/notifications) 時，您應該在發送通知時呼叫 `onGroup` 方法，並可以選擇性地使用 `withDeduplicator` 方法：

```php
use App\Notifications\InvoicePaid;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

$user->notify($invoicePaid);
```

<a name="queue-failover"></a>
### 佇列容錯移轉 (Queue Failover)

`failover` 佇列驅動程式在向佇列推送任務時提供自動容錯移轉功能。如果 `failover` 設定中的主要佇列連線因任何原因失敗，Laravel 將自動嘗試將任務推送到清單中下一個設定的連線。這對於確保正式環境中的高可用性特別有用，因為在這些環境中，佇列的可靠性至關重要。

若要設定容錯移轉佇列連線，請指定 `failover` 驅動程式，並提供一個按順序嘗試的連線名稱陣列。預設情況下，Laravel 在應用程式的 `config/queue.php` 設定檔中包含了一個範例容錯移轉設定：

```php
'failover' => [
    'driver' => 'failover',
    'connections' => [
        'redis',
        'database',
        'sync',
    ],
],
```

一旦您設定了使用 `failover` 驅動程式的連線，您需要在應用程式的 `.env` 檔中將容錯移轉連線設為預設佇列連線，以便使用容錯移轉功能：

```ini
QUEUE_CONNECTION=failover
```

接下來，為容錯移轉連線清單中的每個連線啟動至少一個工作程式：

```bash
php artisan queue:work redis
php artisan queue:work database
```

> [!NOTE]
> 您不需要為使用 `sync`、`background` 或 `deferred` 佇列驅動程式的連線執行工作程式，因為這些驅動程式會在當前 PHP 行程中處理任務。

當佇列連線操作失敗且啟動容錯移轉時，Laravel 會分派 `Illuminate\Queue\Events\QueueFailedOver` 事件，讓您可以報告或記錄佇列連線已失敗。

> [!NOTE]
> 如果您使用 Laravel Horizon，請記住 Horizon 僅管理 Redis 佇列。如果您的容錯移轉清單包含 `database`，您應該在 Horizon 旁邊執行一般的 `php artisan queue:work database` 行程。

<a name="error-handling"></a>
### 錯誤處理

如果在處理任務時拋出例外狀況，該任務將自動被釋放回佇列，以便再次嘗試執行。任務會持續被釋放，直到達到應用程式允許的最大嘗試次數。最大嘗試次數是由 `queue:work` Artisan 命令使用的 `--tries` 選項來定義的。或者，也可以在任務類別本身定義最大嘗試次數。關於執行佇列工作程式的更多資訊[可以在下方找到](#running-the-queue-worker)。

<a name="manually-releasing-a-job"></a>
#### 手動釋放任務

有時您可能希望手動將任務釋放回佇列，以便稍後再次嘗試。您可以透過呼叫 `release` 方法來達成此目的：

```php
/**
 * Execute the job.
 */
public function handle(): void
{
    // ...

    $this->release();
}
```

預設情況下，`release` 方法會將任務釋放回佇列以立即處理。但是，您可以透過傳遞整數或日期實例給 `release` 方法，指示佇列在經過指定的秒數之前不要讓該任務進入可處理狀態：

```php
$this->release(10);

$this->release(now()->plus(seconds: 10));
```

<a name="manually-failing-a-job"></a>
#### 手動使任務失敗

有時您可能需要手動將任務標記為「失敗」。若要執行此操作，您可以呼叫 `fail` 方法：

```php
/**
 * Execute the job.
 */
public function handle(): void
{
    // ...

    $this->fail();
}
```

如果您想因為捕捉到的例外狀況而將任務標記為失敗，可以將該例外傳遞給 `fail` 方法。或者，為了方便起見，您可以傳遞一個字串錯誤訊息，它將會自動為您轉換為例外狀況：

```php
$this->fail($exception);

$this->fail('Something went wrong.');
```

> [!NOTE]
> 關於失敗任務的更多資訊，請參閱[處理任務失敗的說明文件](#dealing-with-failed-jobs)。

<a name="fail-jobs-on-exceptions"></a>
#### 針對特定例外狀況使任務失敗

`FailOnException` [任務中介層](#job-middleware) 允許您在拋出特定例外狀況時直接跳過重試。這讓您可以針對暫時性的例外狀況（例如外部 API 錯誤）進行重試，但對於持續性的例外狀況（例如使用者的權限被撤銷）則讓任務永久失敗：

```php
<?php

namespace App\Jobs;

use App\Models\User;
use Illuminate\Auth\Access\AuthorizationException;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;
use Illuminate\Queue\Attributes\Tries;
use Illuminate\Queue\Middleware\FailOnException;
use Illuminate\Support\Facades\Http;

#[Tries(3)]
class SyncChatHistory implements ShouldQueue
{
    use Queueable;

    /**
     * Create a new job instance.
     */
    public function __construct(
        public User $user,
    ) {}

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        $this->user->authorize('sync-chat-history');

        $response = Http::throw()->get(
            "https://chat.laravel.test/?user={$this->user->uuid}"
        );

        // ...
    }

    /**
     * Get the middleware the job should pass through.
     */
    public function middleware(): array
    {
        return [
            new FailOnException([AuthorizationException::class])
        ];
    }
}
```

<a name="job-batching"></a>
## 任務批次處理 (Job Batching)

Laravel 的任務批次處理功能讓您可以輕鬆地並行執行一組任務，並在該批次任務執行完成後執行某些動作。

在開始之前，您應該建立一個資料庫遷移來建立一張資料表，用於保存有關任務批次的元數據 (meta information)，例如完成百分比。可以使用 `make:queue-batches-table` Artisan 命令來產生此遷移：

```shell
php artisan make:queue-batches-table

php artisan migrate
```

<a name="defining-batchable-jobs"></a>
### 定義可批次處理的任務

要定義一個可批次處理的任務，您應該照常[建立一個可佇列化任務](#creating-jobs)；但是，您應該在任務類別中加入 `Illuminate\Bus\Batchable` trait。此 trait 提供了對 `batch` 方法的存取，可用於檢索任務正在其中執行的當前批次：

```php
<?php

namespace App\Jobs;

use Illuminate\Bus\Batchable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

class ImportCsv implements ShouldQueue
{
    use Batchable, Queueable;

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        if ($this->batch()->cancelled()) {
            // Determine if the batch has been cancelled...

            return;
        }

        // Import a portion of the CSV file...
    }
}
```

<a name="dispatching-batches"></a>
### 分派批次任務

要分派一批任務，您應該使用 `Bus` Facade 的 `batch` 方法。當然，批次處理在與完成回呼 (callback) 結合使用時最為有用。因此，您可以使用 `then`、`catch` 和 `finally` 方法來定義批次的完成回呼。這些回呼在被叫用時都會接收一個 `Illuminate\Bus\Batch` 執行個體。

當執行多個佇列工作程式時，批次中的任務將被並行處理。因此，任務完成的順序可能與它們被加入批次的順序不同。請參閱我們關於[任務鏈與批次任務](#chains-and-batches)的說明文件，瞭解如何依序執行一系列任務。

在此範例中，我們假設正在將一批任務放入佇列，每個任務分別處理 CSV 檔案中指定數量的資料列：

```php
use App\Jobs\ImportCsv;
use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;
use Throwable;

$batch = Bus::batch([
    new ImportCsv(1, 100),
    new ImportCsv(101, 200),
    new ImportCsv(201, 300),
    new ImportCsv(301, 400),
    new ImportCsv(401, 500),
])->before(function (Batch $batch) {
    // The batch has been created but no jobs have been added...
})->progress(function (Batch $batch) {
    // A single job has completed successfully...
})->then(function (Batch $batch) {
    // All jobs completed successfully...
})->catch(function (Batch $batch, Throwable $e) {
    // Batch job failure detected...
})->finally(function (Batch $batch) {
    // The batch has finished executing...
})->dispatch();

return $batch->id;
```

批次的 ID 可以透過 `$batch->id` 屬性存取，可用於在分派後[向 Laravel 命令總線 (Command Bus) 查詢](#inspecting-batches)有關該批次的資訊。

> [!WARNING]
> 由於批次回呼是由 Laravel 佇列序列化並在稍後執行的，因此您不應在回呼中使用 `$this` 變數。此外，由於批次任務被封裝在資料庫交易中，因此觸發隱式提交 (implicit commits) 的資料庫陳述式不應在任務中執行。

<a name="naming-batches"></a>
#### 命名批次

一些工具（如 [Laravel Horizon](/docs/{{version}}/horizon) 和 [Laravel Telescope](/docs/{{version}}/telescope)）若批次有命名，可以提供更易於閱讀的除錯資訊。要為批次指定任意名稱，您可以在定義批次時呼叫 `name` 方法：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import CSV')->dispatch();
```

<a name="batch-connection-queue"></a>
#### 批次的連線與佇列

如果您想指定用於批次任務的連線和佇列，可以使用 `onConnection` 和 `onQueue` 方法。所有批次任務必須在同一個連線和佇列中執行：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->onConnection('redis')->onQueue('imports')->dispatch();
```

<a name="chains-and-batches"></a>
### 任務鏈與批次任務

您可以在批次內定義一組[任務鏈](#job-chaining)，方法是將任務鏈放在陣列中。例如，我們可以並行執行兩個任務鏈，並在兩個任務鏈都處理完成時執行回呼：

```php
use App\Jobs\ReleasePodcast;
use App\Jobs\SendPodcastReleaseNotification;
use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;

Bus::batch([
    [
        new ReleasePodcast(1),
        new SendPodcastReleaseNotification(1),
    ],
    [
        new ReleasePodcast(2),
        new SendPodcastReleaseNotification(2),
    ],
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->dispatch();
```

相反地，您也可以在一個[任務鏈](#job-chaining)中執行多個批次任務。例如，您可以先執行一個批次任務來發布多個 Podcast，接著執行另一個批次任務來發送發布通知：

```php
use App\Jobs\FlushPodcastCache;
use App\Jobs\ReleasePodcast;
use App\Jobs\SendPodcastReleaseNotification;
use Illuminate\Support\Facades\Bus;

Bus::chain([
    new FlushPodcastCache,
    Bus::batch([
        new ReleasePodcast(1),
        new ReleasePodcast(2),
    ]),
    Bus::batch([
        new SendPodcastReleaseNotification(1),
        new SendPodcastReleaseNotification(2),
    ]),
])->dispatch();
```

<a name="adding-jobs-to-batches"></a>
### 向批次中增加任務

有時從批次任務內部向其所屬批次中新增額外任務是很有用的。當您需要批次處理數千個任務，而這些任務在一次網頁請求中分派可能耗時過長時，這種模式很有幫助。因此，您可以先分派一個初始的「載入 (loader)」任務批次，由它向批次中填充更多任務：

```php
$batch = Bus::batch([
    new LoadImportBatch,
    new LoadImportBatch,
    new LoadImportBatch,
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import Contacts')->dispatch();
```

在此範例中，我們將使用 `LoadImportBatch` 任務來向批次中填充額外任務。為此，我們可以使用批次執行個體上的 `add` 方法，該執行個體可以透過任務的 `batch` 方法存取：

```php
use App\Jobs\ImportContacts;
use Illuminate\Support\Collection;

/**
 * Execute the job.
 */
public function handle(): void
{
    if ($this->batch()->cancelled()) {
        return;
    }

    $this->batch()->add(Collection::times(1000, function () {
        return new ImportContacts;
    }));
}
```

> [!WARNING]
> 您只能從屬於同一個批次的任務中向該批次新增任務。

<a name="inspecting-batches"></a>
### 檢查批次

提供給批次完成後回呼函式的 `Illuminate\Bus\Batch` 實例具有多種屬性與方法，可協助您與指定的任務批次進行互動與檢查：

```php
// The UUID of the batch...
$batch->id;

// The name of the batch (if applicable)...
$batch->name;

// The number of jobs assigned to the batch...
$batch->totalJobs;

// The number of jobs that have not been processed by the queue...
$batch->pendingJobs;

// The number of jobs that have failed...
$batch->failedJobs;

// The number of jobs that have been processed thus far...
$batch->processedJobs();

// The completion percentage of the batch (0-100)...
$batch->progress();

// Indicates if the batch has finished executing...
$batch->finished();

// Cancel the execution of the batch...
$batch->cancel();

// Indicates if the batch has been cancelled...
$batch->cancelled();
```


<a name="returning-batches-from-routes"></a>
#### 從路由回傳批次

所有 `Illuminate\Bus\Batch` 實例都可以被 JSON 序列化，這代表您可以直接從應用程式的路由回傳它們，以取得包含批次資訊（包含其完成進度）的 JSON 負載。這讓您在應用程式的使用者介面中顯示批次完成進度變得非常方便。

若要透過 ID 取得批次，您可以使用 `Bus` Facade 的 `findBatch` 方法：

```php
use Illuminate\Support\Facades\Bus;
use Illuminate\Support\Facades\Route;

Route::get('/batch/{batchId}', function (string $batchId) {
    return Bus::findBatch($batchId);
});
```


<a name="cancelling-batches"></a>
### 取消批次

有時您可能需要取消特定批次的執行。這可以透過呼叫 `Illuminate\Bus\Batch` 實例上的 `cancel` 方法來達成：

```php
/**
 * Execute the job.
 */
public function handle(): void
{
    if ($this->user->exceedsImportLimit()) {
        $this->batch()->cancel();

        return;
    }

    if ($this->batch()->cancelled()) {
        return;
    }
}
```

如您在前面的範例中所注意到的，批次任務在繼續執行之前，通常應該先判斷其對應的批次是否已被取消。然而，為了方便起見，您可以改為將 `SkipIfBatchCancelled` [中介層](#job-middleware)分配給該任務。誠如其名，這個中介層將指示 Laravel，如果該任務對應的批次已被取消，則不要處理該任務：

```php
use Illuminate\Queue\Middleware\SkipIfBatchCancelled;

/**
 * Get the middleware the job should pass through.
 */
public function middleware(): array
{
    return [new SkipIfBatchCancelled];
}
```


<a name="batch-failures"></a>
### 批次失敗處理

當批次任務失敗時，將會呼叫 `catch` 回呼（如果有指定的話）。此回呼僅會針對批次中第一個失敗的任務被呼叫。


<a name="allowing-failures"></a>
#### 允許失敗

當批次中的某個任務失敗時，Laravel 會自動將該批次標記為「已取消」。如果您希望，可以停用此行為，讓任務失敗時不會自動將批次標記為已取消。這可以透過在分派批次時呼叫 `allowFailures` 方法來達成：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->allowFailures()->dispatch();
```

您也可以選擇提供一個匿名函式給 `allowFailures` 方法，該匿名函式將在每次任務失敗時執行：

```php
$batch = Bus::batch([
    // ...
])->allowFailures(function (Batch $batch, $exception) {
    // Handle individual job failures...
})->dispatch();
```


<a name="retrying-failed-batch-jobs"></a>
#### 重試失敗的批次任務

為了方便起見，Laravel 提供了一個 `queue:retry-batch` Artisan 命令，讓您可以輕鬆地重試指定批次的所有失敗任務。此命令接受要重試失敗任務的批次 UUID：

```shell
php artisan queue:retry-batch 32dbc76c-4f82-4749-b610-a639fe0099b5
```


<a name="pruning-batches"></a>
### 清理批次 (Pruning Batches)

如果不進行清理，`job_batches` 資料表會非常快速地累積紀錄。為了減輕這種情況，您應該[排程](/docs/{{version}}/scheduling) `queue:prune-batches` Artisan 命令每天執行：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches')->daily();
```

預設情況下，所有已完成超過 24 小時的批次都會被清理。您可以在呼叫命令時使用 `hours` 選項來決定保留批次資料的時間。例如，以下命令將刪除所有在 48 小時前完成的批次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48')->daily();
```

有時，您的 `job_batches` 資料表可能會累積一些從未成功完成的批次紀錄，例如某個任務失敗且該任務從未重試成功的批次。您可以透過 `unfinished` 選項指示 `queue:prune-batches` 命令清理這些未完成的批次紀錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --unfinished=72')->daily();
```

同樣地，您的 `job_batches` 資料表也可能累積已取消批次的紀錄。您可以透過 `cancelled` 選項指示 `queue:prune-batches` 命令清理這些已取消的批次紀錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --cancelled=72')->daily();
```

<a name="storing-batches-in-dynamodb"></a>
### 將批次存儲在 DynamoDB

Laravel 也支援將批次中繼資訊 (Meta information) 存儲在 [DynamoDB](https://aws.amazon.com/dynamodb) 中，而不是關聯式資料庫。但是，您需要手動建立一個 DynamoDB 資料表來存儲所有的批次紀錄。

通常，此資料表的名稱應為 `job_batches`，但您應該根據應用程式 `queue` 設定檔中 `queue.batching.table` 的設定值來為資料表命名。


<a name="dynamodb-batch-table-configuration"></a>
#### DynamoDB 批次資料表設定

`job_batches` 資料表應該有一個名為 `application` 的字串類型主要分割區索引鍵 (Primary partition key)，以及一個名為 `id` 的字串類型主要排序鍵 (Primary sort key)。索引鍵中的 `application` 部分將包含應用程式 `app` 設定檔中 `name` 設定值所定義的應用程式名稱。由於應用程式名稱是 DynamoDB 資料表索引鍵的一部分，因此您可以使用同一個資料表來存儲多個 Laravel 應用程式的任務批次。

此外，如果您想利用 [自動批次清理](#pruning-batches-in-dynamodb) 功能，可以為資料表定義 `ttl` 屬性。


<a name="dynamodb-configuration"></a>
#### DynamoDB 設定

接下來，安裝 AWS SDK 以便您的 Laravel 應用程式可以與 Amazon DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

然後，將 `queue.batching.driver` 設定選項的值設為 `dynamodb`。此外，您應該在 `batching` 設定陣列中定義 `key`、`secret` 和 `region` 設定選項。這些選項將用於 AWS 的身分驗證。使用 `dynamodb` 驅動程式時，不需要設定 `queue.batching.database` 選項：

```php
'batching' => [
    'driver' => env('QUEUE_BATCHING_DRIVER', 'dynamodb'),
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    'table' => 'job_batches',
],
```


<a name="pruning-batches-in-dynamodb"></a>
#### 在 DynamoDB 中清理批次

當利用 [DynamoDB](https://aws.amazon.com/dynamodb) 存儲任務批次資訊時，用於清理關聯式資料庫中批次的典型清理命令將無法運作。相反地，您可以利用 [DynamoDB 的原生 TTL 功能](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html) 來自動移除舊批次的紀錄。

如果您為 DynamoDB 資料表定義了 `ttl` 屬性，則可以定義設定參數來指示 Laravel 如何清理批次紀錄。`queue.batching.ttl_attribute` 設定值定義了持有 TTL 的屬性名稱，而 `queue.batching.ttl` 設定值則定義了在最後一次更新紀錄後，經過多少秒可以從 DynamoDB 資料表中移除該批次紀錄：

```php
'batching' => [
    'driver' => env('QUEUE_FAILED_DRIVER', 'dynamodb'),
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    'table' => 'job_batches',
    'ttl_attribute' => 'ttl',
    'ttl' => 60 * 60 * 24 * 7, // 7 days...
],
```

<a name="queueing-closures"></a>
## 佇列化匿名函式 (Closures)

除了將任務類別分派到佇列之外，您也可以分派一個匿名函式 (Closure)。這對於需要在當前請求週期之外執行的快速、簡單的任務來說非常方便。將匿名函式分派到佇列時，該函式的程式碼內容會經過加密簽名，因此在傳輸過程中無法被修改：

```php
use App\Models\Podcast;

$podcast = Podcast::find(1);

dispatch(function () use ($podcast) {
    $podcast->publish();
});
```

若要為佇列化的匿名函式指定名稱（可用於佇列報告儀表板，也會顯示在 `queue:work` 命令中），您可以使用 `name` 方法：

```php
dispatch(function () {
    // ...
})->name('Publish Podcast');
```

使用 `catch` 方法，您可以提供一個在佇列匿名函式耗盡所有[設定的重試次數](#max-job-attempts-and-timeout)後仍未成功完成時，所要執行的匿名函式：

```php
use Throwable;

dispatch(function () use ($podcast) {
    $podcast->publish();
})->catch(function (Throwable $e) {
    // This job has failed...
});
```

> [!WARNING]
> 由於 `catch` 回呼會被序列化並在稍後由 Laravel 佇列執行，因此您不應在 `catch` 回呼內使用 `$this` 變數。

<a name="running-the-queue-worker"></a>
## 執行佇列工作程式 (Queue Worker)

<a name="the-queue-work-command"></a>
### `queue:work` 命令

Laravel 包含一個 Artisan 命令，用於啟動佇列工作程式並處理推送到佇列的新任務。您可以使用 `queue:work` Artisan 命令來執行工作程式。請注意，一旦 `queue:work` 命令啟動後，它將持續運行，直到手動停止或關閉終端機為止：

```shell
php artisan queue:work
```

> [!NOTE]
> 為了讓 `queue:work` 行程(Processes) 永久在背景運行，您應該使用程序監控器，例如 [Supervisor](#supervisor-configuration)，以確保佇列工作程式不會停止運行。

如果您希望在命令的輸出中包含已處理的任務 ID、連線名稱和佇列名稱，可以在執行 `queue:work` 命令時加上 `-v` 旗標：

```shell
php artisan queue:work -v
```

請記住，佇列工作程式是長時間執行的行程(Processes)，並會將已啟動的應用程式狀態儲存在記憶體中。因此，在它們啟動後，它們將不會注意到程式碼庫的變更。所以，在您的部署過程中，請務必[重新啟動您的佇列工作程式](#queue-workers-and-deployment)。此外，請記住應用程式建立或修改的任何靜態狀態都不會在任務之間自動重置。

或者，您可以執行 `queue:listen` 命令。使用 `queue:listen` 命令時，當您想要重新載入更新後的程式碼或重置應用程式狀態時，不必手動重新啟動工作程式；然而，這個命令的效率明顯低於 `queue:work` 命令：

```shell
php artisan queue:listen
```

<a name="running-multiple-queue-workers"></a>
#### 執行多個佇列工作程式

若要為佇列分配多個工作程式並同時處理多個任務，您只需啟動多個 `queue:work` 行程(Processes) 即可。這可以透過在終端機開啟多個分頁在本地完成，或者在正式環境中使用程序管理器的設定來達成。[使用 Supervisor 時](#supervisor-configuration)，您可以使用 `numprocs` 設定值。

<a name="specifying-the-connection-queue"></a>
#### 指定連線與佇列

您也可以指定工作程式應使用的佇列連線。傳遞給 `work` 命令的連線名稱應對應於 `config/queue.php` 設定檔中定義的其中一個連線：

```shell
php artisan queue:work redis
```

預設情況下，`queue:work` 命令僅處理給定連線上預設佇列的任務。然而，您可以透過僅處理給定連線的特定佇列來進一步自訂您的佇列工作程式。例如，如果您的所有電子郵件都在 `redis` 佇列連線上的 `emails` 佇列中處理，您可以發出以下命令來啟動僅處理該佇列的工作程式：

```shell
php artisan queue:work redis --queue=emails
```

<a name="processing-a-specified-number-of-jobs"></a>
#### 處理指定數量的任務

`--once` 選項可用於指示工作程式僅從佇列中處理單個任務：

```shell
php artisan queue:work --once
```

`--max-jobs` 選項可用於指示工作程式在處理指定數量的任務後退出。此選項在與 [Supervisor](#supervisor-configuration) 配合使用時非常有用，以便您的工作程式在處理一定數量的任務後自動重新啟動，從而釋放它們可能累積的任何記憶體：

```shell
php artisan queue:work --max-jobs=1000
```

<a name="processing-all-queued-jobs-then-exiting"></a>
#### 處理所有佇列任務後結束

`--stop-when-empty` 選項可用於指示工作程式處理所有任務後優雅地退出。在 Docker 容器中處理 Laravel 佇列時，如果您希望在佇列清空後關閉容器，此選項非常有用：

```shell
php artisan queue:work --stop-when-empty
```

<a name="processing-jobs-for-a-given-number-of-seconds"></a>
#### 處理任務達指定秒數後結束

`--max-time` 選項可用於指示工作程式處理任務達指定的秒數後退出。此選項在與 [Supervisor](#supervisor-configuration) 配合使用時非常有用，以便您的工作程式在處理任務一段時間後自動重新啟動，從而釋放它們可能累積的任何記憶體：

```shell
# Process jobs for one hour and then exit...
php artisan queue:work --max-time=3600
```

<a name="worker-sleep-duration"></a>
#### 工作程式休眠時間

當佇列中有任務可用時，工作程式將持續處理任務，任務之間沒有延遲。然而，`sleep` 選項決定了如果沒有可用任務，工作程式將「休眠」多少秒。當然，在休眠期間，工作程式將不會處理任何新任務：

```shell
php artisan queue:work --sleep=3
```

<a name="maintenance-mode-queues"></a>
#### 維護模式與佇列

當您的應用程式處於 [維護模式](/docs/{{version}}/configuration#maintenance-mode) 時，不會處理任何佇列任務。一旦應用程式結束維護模式，任務將照常處理。

要強制佇列工作程式即使在啟用維護模式的情況下也處理任務，您可以使用 `--force` 選項：

```shell
php artisan queue:work --force
```

<a name="resource-considerations"></a>
#### 資源考量

守護行程(Daemon) 佇列工作程式在處理每個任務之前不會「重啟」框架。因此，您應該在每個任務完成後釋放任何沉重的資源。例如，如果您正在使用 [GD 函式庫](https://www.php.net/manual/en/book.image.php) 進行影像處理，則應在處理完影像後使用 `imagedestroy` 釋放記憶體。

<a name="queue-priorities"></a>
### 佇列優先權

有時您可能希望優先處理佇列的方式。例如，在您的 `config/queue.php` 設定檔中，您可以將 `redis` 連線的預設 `queue` 設定為 `low`。然而，有時您可能希望將任務推送到 `high` 優先權佇列，如下所示：

```php
dispatch((new Job)->onQueue('high'));
```

要啟動一個工作程式，並在繼續處理 `low` 佇列上的任何任務之前，先驗證所有 `high` 佇列任務都已處理完畢，請將以逗號分隔的佇列名稱列表傳遞給 `work` 命令：

```shell
php artisan queue:work --queue=high,low
```

<a name="queue-workers-and-deployment"></a>
### 佇列工作程式與部署

由於佇列工作程式是長時間執行的行程(Processes)，如果不重新啟動，它們將不會注意到程式碼的變更。因此，部署使用佇列工作程式的應用程式最簡單的方法是在部署過程中重新啟動工作程式。您可以透過發出 `queue:restart` 命令來優雅地重新啟動所有工作程式：

```shell
php artisan queue:restart
```

此命令將指示所有佇列工作程式在處理完目前任務後優雅地退出，以免遺失任何現有任務。由於執行 `queue:restart` 命令時佇列工作程式將會退出，因此您應該運行一個程序管理器（例如 [Supervisor](#supervisor-configuration)）來自動重新啟動佇列工作程式。

> [!NOTE]
> 佇列使用 [快取 (Cache)](/docs/{{version}}/cache) 來儲存重啟訊號，因此在使用此功能之前，您應驗證應用程式是否已正確設定快取驅動程式。

<a name="job-expirations-and-timeouts"></a>
### 任務過期與超時

<a name="job-expiration"></a>
#### 任務過期

在您的 `config/queue.php` 設定檔中，每個佇列連線都定義了一個 `retry_after` 選項。此選項指定佇列連線在重試正在處理的任務之前應等待多少秒。例如，如果 `retry_after` 的值設為 `90`，則該任務若處理了 90 秒仍未被釋放或刪除，就會被重新釋放回佇列。通常，您應該將 `retry_after` 的值設為任務合理完成處理所需的最大秒數。

> [!WARNING]
> 唯一不包含 `retry_after` 值的佇列連線是 Amazon SQS。SQS 將根據 AWS 控制台中管理的 [預設可見度超時 (Default Visibility Timeout)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html) 來重試任務。

<a name="worker-timeouts"></a>
#### 工作程式超時

`queue:work` Artisan 命令提供了一個 `--timeout` 選項。預設情況下，`--timeout` 值為 60 秒。如果任務處理時間長於超時值指定的秒數，處理該任務的工作程式將會出錯並結束執行。通常，工作程式會由您伺服器上設定的 [行程管理員(Processes)](#supervisor-configuration) 自動重啟：

```shell
php artisan queue:work --timeout=60
```

`retry_after` 設定選項與 `--timeout` CLI 選項不同，但它們會共同運作以確保任務不會遺失，且每個任務僅會被成功處理一次。

> [!WARNING]
> `--timeout` 的值應始終比 `retry_after` 設定值短至少幾秒鐘。這將確保處理僵死 (Frozen) 任務的工作程式始終在任務被重試之前被終止。如果您的 `--timeout` 選項比 `retry_after` 設定值長，您的任務可能會被處理兩次。

<a name="pausing-and-resuming-queue-workers"></a>
### 暫停與恢復佇列工作程式

有時您可能需要暫時防止佇列工作程式處理新任務，而不完全停止工作程式。例如，您可能希望在系統維護期間暫停任務處理。Laravel 提供了 `queue:pause` 與 `queue:continue` Artisan 命令來暫停與恢復佇列工作程式。

若要暫停特定佇列，請提供佇列連線名稱與佇列名稱：

```shell
php artisan queue:pause database:default
```

在此範例中，`database` 是佇列連線名稱，而 `default` 是佇列名稱。一旦佇列暫停，任何正在處理該佇列任務的工作程式都將繼續完成其目前的任務，但在佇列恢復之前不會接手任何新任務。

若要恢復處理已暫停佇列中的任務，請使用 `queue:continue` 命令：

```shell
php artisan queue:continue database:default
```

恢復佇列後，工作程式將立即開始處理該佇列中的新任務。請注意，暫停佇列並不會停止工作程式行程本身——它僅防止工作程式從指定的佇列中處理新任務。

<a name="worker-restart-and-pause-signals"></a>
#### 工作程式重啟與暫停訊號

預設情況下，佇列工作程式會在每次任務迭代中輪詢快取驅動程式以獲取重啟與暫停訊號。雖然這種輪詢對於回應 `queue:restart` 與 `queue:pause` 命令至關重要，但它確實會引入少量的效能開銷。

如果您需要優化效能且不需要這些中斷功能，您可以透過呼叫 `Queue` Facade 上的 `withoutInterruptionPolling` 方法來全域停用此輪詢。這通常應在您的 `AppServiceProvider` 的 `boot` 方法中完成：

```php
use Illuminate\Support\Facades\Queue;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Queue::withoutInterruptionPolling();
}
```

或者，您也可以透過在 `Illuminate\Queue\Worker` 類別上設定靜態屬性 `$restartable` 或 `$pausable` 來分別停用重啟或暫停輪詢：

```php
use Illuminate\Queue\Worker;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Worker::$restartable = false;
    Worker::$pausable = false;
}
```

> [!WARNING]
> 當停用中斷輪詢時，工作程式將不會回應 `queue:restart` 或 `queue:pause` 命令（取決於停用了哪些功能）。

<a name="supervisor-configuration"></a>
## Supervisor 設定

在正式環境中，您需要一種方法來維持 `queue:work` 行程(Processes)持續執行。`queue:work` 行程可能會因為多種原因停止運行，例如超出了工作程式的超時時間或執行了 `queue:restart` 命令。

因此，您需要設定一個行程監控器，它可以偵測您的 `queue:work` 行程何時結束並自動重啟它們。此外，行程監控器還能讓您指定要同時執行多少個 `queue:work` 行程。Supervisor 是 Linux 環境中常用的行程監控器，我們將在接下來的文件中討論如何設定它。


<a name="installing-supervisor"></a>
#### 安裝 Supervisor

Supervisor 是 Linux 作業系統的行程監控器，如果您的 `queue:work` 行程失敗，它將自動重啟它們。要在 Ubuntu 上安裝 Supervisor，您可以使用以下命令：

```shell
sudo apt-get install supervisor
```

> [!NOTE]
> 如果自行設定和管理 Supervisor 讓您感到困擾，請考慮使用 [Laravel Cloud](https://cloud.laravel.com)，它提供了一個全代管平台來執行 Laravel 佇列工作程式。


<a name="configuring-supervisor"></a>
#### 設定 Supervisor

Supervisor 的設定檔通常儲存在 `/etc/supervisor/conf.d` 目錄中。在此目錄中，您可以建立任意數量的設定檔，指示 Supervisor 如何監控您的行程。例如，讓我們建立一個 `laravel-worker.conf` 檔案來啟動並監控 `queue:work` 行程：

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /home/forge/app.com/artisan queue:work --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=forge
numprocs=8
redirect_stderr=true
stdout_logfile=/home/forge/app.com/worker.log
stopwaitsecs=3600
```

在此範例中，`numprocs` 指令將指示 Supervisor 執行 8 個 `queue:work` 行程並監控所有行程，如果它們失敗則自動重啟。您應該更改設定中的 `command` 指令，以反映您所需的佇列連線和工作程式選項。

> [!WARNING]
> 您應該確保 `stopwaitsecs` 的值大於您執行時間最長的任務所消耗的秒數。否則，Supervisor 可能會在任務完成處理之前就將其終止。


<a name="starting-supervisor"></a>
#### 啟動 Supervisor

設定檔建立完成後，您可以使用以下命令更新 Supervisor 設定並啟動行程：

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start "laravel-worker:*"
```

有關 Supervisor 的更多資訊，請參考 [Supervisor 說明文件](http://supervisord.org/index.html)。

<a name="dealing-with-failed-jobs"></a>
## 處理失敗的任務

有時候你的佇列任務會失敗。別擔心，事情並不總是按計畫進行！Laravel 提供了一種方便的方法來 [指定任務應嘗試的最大次數](#max-job-attempts-and-timeout)。在一個非同步任務超過此嘗試次數後，它將被插入到 `failed_jobs` 資料庫資料表中。[同步分派任務](/docs/{{version}}/queues#synchronous-dispatching) 若失敗則不會儲存在此資料表中，且其例外狀況會立即由應用程式處理。

在新安裝的 Laravel 應用程式中，通常已經包含了建立 `failed_jobs` 資料表的遷移。然而，如果你的應用程式中沒有這個資料表的遷移，你可以使用 `make:queue-failed-table` Artisan 命令來建立它：

```shell
php artisan make:queue-failed-table

php artisan migrate
```

執行 [佇列工作程式 (queue worker)](#running-the-queue-worker) 行程時，你可以使用 `queue:work` 命令上的 `--tries` 開關來指定任務應嘗試的最大次數。如果你沒有為 `--tries` 選項指定值，任務將僅嘗試一次，或者嘗試次數由任務類別的 `Tries` 屬性指定：

```shell
php artisan queue:work redis --tries=3
```

使用 `--backoff` 選項，你可以指定在重試遇到例外狀況的任務之前，Laravel 應該等待多少秒。預設情況下，任務會立即被釋放回佇列，以便可以再次嘗試：

```shell
php artisan queue:work redis --tries=3 --backoff=3
```

如果你想針對每個任務單獨設定遇到例外狀況後重試前應等待的秒數，你可以在任務類別上使用 `Backoff` 屬性：

```php
<?php

namespace App\Jobs;

use Illuminate\Queue\Attributes\Backoff;

#[Backoff(3)]
class ProcessPodcast implements ShouldQueue
{
    // ...
}
```

如果你需要更複雜的邏輯來決定任務的退避時間，可以在任務類別中定義一個 `backoff` 方法：

```php
/**
 * Calculate the number of seconds to wait before retrying the job.
 */
public function backoff(): int
{
    return 3;
}
```

你也可以透過定義一個退避值陣列來輕鬆設定「指數型」退避。在此範例中，第一次重試的延遲時間為 1 秒，第二次為 5 秒，第三次為 10 秒，如果還有剩餘嘗試次數，則之後的每次重試延遲時間均為 10 秒：

```php
<?php

namespace App\Jobs;

use Illuminate\Queue\Attributes\Backoff;

#[Backoff([1, 5, 10])]
class ProcessPodcast implements ShouldQueue
{
    // ...
}
```


<a name="cleaning-up-after-failed-jobs"></a>
### 失敗任務後的清理

當特定任務失敗時，你可能希望向使用者傳送警示或恢復該任務已部分完成的任何操作。為了實現這一點，你可以在任務類別中定義一個 `failed` 方法。導致任務失敗的 `Throwable` 執行個體將被傳遞給 `failed` 方法：

```php
<?php

namespace App\Jobs;

use App\Models\Podcast;
use App\Services\AudioProcessor;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;
use Throwable;

class ProcessPodcast implements ShouldQueue
{
    use Queueable;

    /**
     * Create a new job instance.
     */
    public function __construct(
        public Podcast $podcast,
    ) {}

    /**
     * Execute the job.
     */
    public function handle(AudioProcessor $processor): void
    {
        // Process uploaded podcast...
    }

    /**
     * Handle a job failure.
     */
    public function failed(?Throwable $exception): void
    {
        // Send user notification of failure, etc...
    }
}
```

> [!WARNING]
> 在呼叫 `failed` 方法之前，會先實例化該任務的新執行個體；因此，任何在 `handle` 方法中可能發生的類別屬性修改都將遺失。

失敗的任務並不一定是指遇到了未處理的例外狀況。當一個任務用盡了所有允許的嘗試次數時，也被視為失敗。嘗試次數可能會因為以下幾種方式被消耗：

<div class="content-list" markdown="1">

- 任務超時。
- 任務在執行過程中遇到了未處理的例外狀況。
- 任務被手動或透過中介層釋放回佇列。

</div>

如果最後一次嘗試是因為任務執行過程中拋出的例外狀況而失敗，該例外狀況將被傳遞給任務的 `failed` 方法。然而，如果任務是因為達到了最大允許重試次數而失敗，則 `$exception` 將是 `Illuminate\Queue\MaxAttemptsExceededException` 的執行個體。同樣地，如果任務是因為超過設定的超時時間而失敗，則 `$exception` 將是 `Illuminate\Queue\TimeoutExceededException` 的執行個體。


<a name="retrying-failed-jobs"></a>
### 重試失敗的任務

要查看所有已插入到 `failed_jobs` 資料庫資料表中的失敗任務，你可以使用 `queue:failed` Artisan 命令：

```shell
php artisan queue:failed
```

`queue:failed` 命令會列出任務 ID、連線、佇列、失敗時間以及關於該任務的其他資訊。任務 ID 可用於重試該失敗任務。例如，要重試 ID 為 `ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece` 的失敗任務，請執行以下命令：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece
```

如有需要，你可以向命令傳遞多個 ID：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece 91401d2c-0784-4f43-824c-34f94a33c24d
```

你也可以重試特定佇列的所有失敗任務：

```shell
php artisan queue:retry --queue=name
```

要重試所有的失敗任務，請執行 `queue:retry` 命令並將 `all` 作為 ID 傳入：

```shell
php artisan queue:retry all
```

如果你想刪除一個失敗的任務，可以使用 `queue:forget` 命令：

```shell
php artisan queue:forget 91401d2c-0784-4f43-824c-34f94a33c24d
```

> [!NOTE]
> 使用 [Horizon](/docs/{{version}}/horizon) 時，你應該使用 `horizon:forget` 命令來刪除失敗的任務，而不是 `queue:forget` 命令。

要從 `failed_jobs` 資料表中刪除所有失敗的任務，可以使用 `queue:flush` 命令：

```shell
php artisan queue:flush
```

`queue:flush` 命令會移除佇列中所有的失敗任務記錄，無論該失敗任務已存在多久。你可以使用 `--hours` 選項僅刪除特定小時數之前失敗的任務。例如，以下命令將刪除所有在 48 小時前插入的失敗任務記錄：

```shell
php artisan queue:flush --hours=48
```


<a name="ignoring-missing-models"></a>
### 忽略遺失的模型

當向任務注入 Eloquent 模型時，該模型會在放入佇列之前自動被序列化，並在任務處理時重新從資料庫中檢索。然而，如果模型在任務等待處理的過程中被刪除，你的任務可能會因為 `ModelNotFoundException` 而失敗。

為了方便起見，你可以透過在任務類別上使用 `DeleteWhenMissingModels` 屬性，來選擇自動刪除遺失模型的任務。當此屬性存在時，Laravel 會悄悄丟棄該任務而不引發例外狀況：

```php
<?php

namespace App\Jobs;

use Illuminate\Queue\Attributes\DeleteWhenMissingModels;

#[DeleteWhenMissingModels]
class ProcessPodcast implements ShouldQueue
{
    // ...
}
```


<a name="pruning-failed-jobs"></a>
### 清理失敗的任務

你可以透過執行 `queue:prune-failed` Artisan 命令來清理應用程式 `failed_jobs` 資料表中的記錄：

```shell
php artisan queue:prune-failed
```

預設情況下，所有超過 24 小時的失敗任務記錄都將被清理。如果你為該命令提供 `--hours` 選項，則僅會保留過去 N 小時內插入的失敗任務記錄。例如，以下命令將刪除所有在 48 小時前插入的失敗任務記錄：

```shell
php artisan queue:prune-failed --hours=48
```

<a name="storing-failed-jobs-in-dynamodb"></a>
### 將失敗任務存儲在 DynamoDB

Laravel 也支援將失敗的任務紀錄存儲在 [DynamoDB](https://aws.amazon.com/dynamodb) 中，而不是關聯式資料庫資料表中。但是，您必須手動建立一個 DynamoDB 資料表來儲存所有失敗任務的紀錄。通常，此資料表應命名為 `failed_jobs`，但您應該根據應用程式 `queue` 設定檔中 `queue.failed.table` 設定項的值來為資料表命名。

`failed_jobs` 資料表應包含一個名為 `application` 的字串型別主分割索引鍵 (Primary Partition Key)，以及一個名為 `uuid` 的字串型別主要排序鍵 (Primary Sort Key)。索引鍵中的 `application` 部分將包含應用程式 `app` 設定檔中 `name` 設定值所定義的應用程式名稱。由於應用程式名稱是 DynamoDB 資料表索引鍵的一部分，因此您可以使用同一個資料表來儲存多個 Laravel 應用程式的失敗任務。

此外，請確保安裝了 AWS SDK，以便您的 Laravel 應用程式可以與 Amazon DynamoDB 進行通訊：

```shell
composer require aws/aws-sdk-php
```

接下來，將 `queue.failed.driver` 設定選項的值設置為 `dynamodb`。此外，您應該在失敗任務設定陣列中定義 `key`、`secret` 與 `region` 設定選項。這些選項將用於 AWS 的驗證。當使用 `dynamodb` 驅動程式時，`queue.failed.database` 設定選項是不需要的：

```php
'failed' => [
    'driver' => env('QUEUE_FAILED_DRIVER', 'dynamodb'),
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    'table' => 'failed_jobs',
],
```


<a name="disabling-failed-job-storage"></a>
### 停用失敗任務儲存

您可以透過將 `queue.failed.driver` 設定選項的值設置為 `null`，來指示 Laravel 直接捨棄失敗的任務而不進行儲存。通常，這可以透過 `QUEUE_FAILED_DRIVER` 環境變數來完成：

```ini
QUEUE_FAILED_DRIVER=null
```


<a name="failed-job-events"></a>
### 失敗任務事件

如果您想要註冊一個在任務失敗時會被呼叫的事件監聽器，可以使用 `Queue` Facade 的 `failing` 方法。例如，我們可以在 Laravel 內建的 `AppServiceProvider` 的 `boot` 方法中，為此事件附加一個匿名函式：

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Queue;
use Illuminate\Support\ServiceProvider;
use Illuminate\Queue\Events\JobFailed;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register(): void
    {
        // ...
    }

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Queue::failing(function (JobFailed $event) {
            // $event->connectionName
            // $event->job
            // $event->exception
        });
    }
}
```

<a name="clearing-jobs-from-queues"></a>
## 清除佇列中的任務

> [!NOTE]
> 使用 [Horizon](/docs/{{version}}/horizon) 時，您應該使用 `horizon:clear` 命令來清除佇列中的任務，而不是使用 `queue:clear` 命令。

如果您想從預設連線的預設佇列中刪除所有任務，可以使用 `queue:clear` Artisan 命令：

```shell
php artisan queue:clear
```

您也可以提供 `connection` 引數與 `queue` 選項，從特定的連線與佇列中刪除任務：

```shell
php artisan queue:clear redis --queue=emails
```

> [!WARNING]
> 清除佇列任務僅適用於 SQS、Redis 與 database 佇列驅動程式。此外，SQS 的訊息刪除過程最多需要 60 秒，因此在清除佇列後的 60 秒內發送到 SQS 佇列的任務也可能被刪除。

<a name="monitoring-your-queues"></a>
## 監控您的佇列

如果您的佇列突然湧入大量任務，它可能會變得不堪負荷，導致任務完成的等待時間變長。如果您願意，Laravel 可以在您的佇列任務數量超過指定門檻時向您發出警報。

首先，您應該排定 `queue:monitor` 命令 [每分鐘執行一次](/docs/{{version}}/scheduling)。該命令接受您想要監控的佇列名稱，以及您預期的任務數量門檻：

```shell
php artisan queue:monitor redis:default,redis:deployments --max=100
```

僅排定此命令執行是不夠的，它不會自動觸發通知來提醒您佇列不堪負荷的狀態。當命令遇到一個任務數量超過門檻的佇列時，會分派一個 `Illuminate\Queue\Events\QueueBusy` 事件。您可以在應用程式的 `AppServiceProvider` 中監聽此事件，以便將通知發送給您或您的開發團隊：

```php
use App\Notifications\QueueHasLongWaitTime;
use Illuminate\Queue\Events\QueueBusy;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Notification;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Event::listen(function (QueueBusy $event) {
        Notification::route('mail', 'dev@example.com')
            ->notify(new QueueHasLongWaitTime(
                $event->connectionName,
                $event->queue,
                $event->size
            ));
    });
}
```

<a name="testing"></a>
## 測試

在測試分派任務的程式碼時，您可能希望指示 Laravel 不要實際執行任務本身，因為任務的程式碼可以獨立於分派任務的程式碼進行測試。當然，若要測試任務本身，您可以實例化該任務並在測試中直接呼叫 `handle` 方法。

您可以使用 `Queue` Facade 的 `fake` 方法來防止佇列任務實際被推送到佇列中。在呼叫 `Queue` Facade 的 `fake` 方法後，您就可以斷言應用程式是否有嘗試將任務推送到佇列：

```php tab=Pest
<?php

use App\Jobs\AnotherJob;
use App\Jobs\ShipOrder;
use Illuminate\Support\Facades\Queue;

test('orders can be shipped', function () {
    Queue::fake();

    // Perform order shipping...

    // Assert that no jobs were pushed...
    Queue::assertNothingPushed();

    // Assert a job was pushed to a given queue...
    Queue::assertPushedOn('queue-name', ShipOrder::class);

    // Assert a job was pushed
    Queue::assertPushed(ShipOrder::class);

    // Assert a job was pushed twice...
    Queue::assertPushedTimes(ShipOrder::class, 2);

    // Assert a job was not pushed...
    Queue::assertNotPushed(AnotherJob::class);

    // Assert that a closure was pushed to the queue...
    Queue::assertClosurePushed();

    // Assert that a closure was not pushed...
    Queue::assertClosureNotPushed();

    // Assert the total number of jobs that were pushed...
    Queue::assertCount(3);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Jobs\AnotherJob;
use App\Jobs\ShipOrder;
use Illuminate\Support\Facades\Queue;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_orders_can_be_shipped(): void
    {
        Queue::fake();

        // Perform order shipping...

        // Assert that no jobs were pushed...
        Queue::assertNothingPushed();

        // Assert a job was pushed to a given queue...
        Queue::assertPushedOn('queue-name', ShipOrder::class);

        // Assert a job was pushed
        Queue::assertPushed(ShipOrder::class);

        // Assert a job was pushed twice...
        Queue::assertPushedTimes(ShipOrder::class, 2);

        // Assert a job was not pushed...
        Queue::assertNotPushed(AnotherJob::class);

        // Assert that a closure was pushed to the queue...
        Queue::assertClosurePushed();

        // Assert that a closure was not pushed...
        Queue::assertClosureNotPushed();

        // Assert the total number of jobs that were pushed...
        Queue::assertCount(3);
    }
}
```

您可以傳遞一個匿名函式給 `assertPushed`、`assertNotPushed`、`assertClosurePushed` 或 `assertClosureNotPushed` 方法，以便斷言推送到佇列的任務是否通過指定的「真值測試 (Truth test)」。只要有至少一個任務通過給定的真值測試，斷言就會成功：

```php
use Illuminate\Queue\CallQueuedClosure;

Queue::assertPushed(function (ShipOrder $job) use ($order) {
    return $job->order->id === $order->id;
});

Queue::assertClosurePushed(function (CallQueuedClosure $job) {
    return $job->name === 'validate-order';
});
```


<a name="faking-a-subset-of-jobs"></a>
### 模擬 (Faking) 部分任務

如果您只需要模擬特定任務，同時允許其他任務正常執行，可以將要模擬的任務類別名稱傳遞給 `fake` 方法：

```php tab=Pest
test('orders can be shipped', function () {
    Queue::fake([
        ShipOrder::class,
    ]);

    // Perform order shipping...

    // Assert a job was pushed twice...
    Queue::assertPushedTimes(ShipOrder::class, 2);
});
```

```php tab=PHPUnit
public function test_orders_can_be_shipped(): void
{
    Queue::fake([
        ShipOrder::class,
    ]);

    // Perform order shipping...

    // Assert a job was pushed twice...
    Queue::assertPushedTimes(ShipOrder::class, 2);
}
```

您可以使用 `except` 方法模擬除了指定任務集以外的所有任務：

```php
Queue::fake()->except([
    ShipOrder::class,
]);
```


<a name="testing-job-chains"></a>
### 測試任務鏈

要測試任務鏈，您需要利用 `Bus` Facade 的模擬功能。`Bus` Facade 的 `assertChained` 方法可用於斷言是否分派了[任務鏈 (Job Chaining)](/docs/{{version}}/queues#job-chaining)。`assertChained` 方法的第一個引數接受一個包含鏈接任務的陣列：

```php
use App\Jobs\RecordShipment;
use App\Jobs\ShipOrder;
use App\Jobs\UpdateInventory;
use Illuminate\Support\Facades\Bus;

Bus::fake();

// ...

Bus::assertChained([
    ShipOrder::class,
    RecordShipment::class,
    UpdateInventory::class
]);
```

如上例所示，鏈接任務的陣列可以是任務類別名稱的陣列。然而，您也可以提供實際任務實例的陣列。當您這麼做時，Laravel 將確保任務實例屬於相同的類別，且具有與應用程式所分派鏈接任務相同的屬性值：

```php
Bus::assertChained([
    new ShipOrder,
    new RecordShipment,
    new UpdateInventory,
]);
```

您可以使用 `assertDispatchedWithoutChain` 方法來斷言任務是在沒有任務鏈的情況下被推送的：

```php
Bus::assertDispatchedWithoutChain(ShipOrder::class);
```


<a name="testing-chain-modifications"></a>
#### 測試任務鏈修改

如果鏈接中的任務[將任務前置或後置到現有任務鏈中](#adding-jobs-to-the-chain)，您可以使用任務的 `assertHasChain` 方法來斷言該任務是否擁有預期的剩餘任務鏈：

```php
$job = new ProcessPodcast;

$job->handle();

$job->assertHasChain([
    new TranscribePodcast,
    new OptimizePodcast,
    new ReleasePodcast,
]);
```

`assertDoesntHaveChain` 方法可用於斷言該任務的剩餘任務鏈為空：

```php
$job->assertDoesntHaveChain();
```


<a name="testing-chained-batches"></a>
#### 測試鏈接中的批次任務

如果您的任務鏈[包含批次任務](#chains-and-batches)，您可以透過在鏈結斷言中插入 `Bus::chainedBatch` 定義，來斷言鏈結中的批次任務是否符合您的預期：

```php
use App\Jobs\ShipOrder;
use App\Jobs\UpdateInventory;
use Illuminate\Bus\PendingBatch;
use Illuminate\Support\Facades\Bus;

Bus::assertChained([
    new ShipOrder,
    Bus::chainedBatch(function (PendingBatch $batch) {
        return $batch->jobs->count() === 3;
    }),
    new UpdateInventory,
]);
```

<a name="testing-job-batches"></a>
### 測試任務批次

`Bus` Facade 的 `assertBatched` 方法可用於斷言已分派了一個[批次任務 (Job Batching)](/docs/{{version}}/queues#job-batching)。提供給 `assertBatched` 方法的匿名函式會接收到一個 `Illuminate\Bus\PendingBatch` 的實例，可用來檢查該批次內的任務：

```php
use Illuminate\Bus\PendingBatch;
use Illuminate\Support\Facades\Bus;

Bus::fake();

// ...

Bus::assertBatched(function (PendingBatch $batch) {
    return $batch->name == 'Import CSV' &&
           $batch->jobs->count() === 10;
});
```

可以在待處理批次上使用 `hasJobs` 方法來驗證該批次是否包含預期的任務。該方法接受一個包含任務實例、類別名稱或匿名函式的陣列：

```php
Bus::assertBatched(function (PendingBatch $batch) {
    return $batch->hasJobs([
        new ProcessCsvRow(row: 1),
        new ProcessCsvRow(row: 2),
        new ProcessCsvRow(row: 3),
    ]);
});
```

當使用匿名函式時，匿名函式將接收任務實例。預期的任務型別將從匿名函式的型別提示 (Type hint) 中推斷出來：

```php
Bus::assertBatched(function (PendingBatch $batch) {
    return $batch->hasJobs([
        fn (ProcessCsvRow $job) => $job->row === 1,
        fn (ProcessCsvRow $job) => $job->row === 2,
        fn (ProcessCsvRow $job) => $job->row === 3,
    ]);
});
```

您可以使用 `assertBatchCount` 方法來斷言已分派了指定數量的批次：

```php
Bus::assertBatchCount(3);
```

您可以使用 `assertNothingBatched` 來斷言沒有分派任何批次：

```php
Bus::assertNothingBatched();
```

<a name="testing-job-batch-interaction"></a>
#### 測試任務與批次的互動

此外，您有時可能需要測試個別任務與其底層批次的互動。例如，您可能需要測試某個任務是否取消了其批次的後續處理。為了達成這個目的，您需要透過 `withFakeBatch` 方法為該任務分配一個模擬批次。`withFakeBatch` 方法會回傳一個包含任務實例與模擬批次的元組 (Tuple)：

```php
[$job, $batch] = (new ShipOrder)->withFakeBatch();

$job->handle();

$this->assertTrue($batch->cancelled());
$this->assertEmpty($batch->added);
```

<a name="testing-job-queue-interactions"></a>
### 測試任務與佇列的互動

有時，您可能需要測試一個佇列化任務是否[將自己釋放回佇列](#manually-releasing-a-job)。或者，您可能需要測試任務是否已刪除自己。您可以透過實例化任務並呼叫 `withFakeQueueInteractions` 方法來測試這些佇列互動。

一旦模擬了任務的佇列互動，您就可以在任務上呼叫 `handle` 方法。在呼叫任務之後，有多種斷言方法可用來驗證任務的佇列互動：

```php
use App\Exceptions\CorruptedAudioException;
use App\Jobs\ProcessPodcast;

$job = (new ProcessPodcast)->withFakeQueueInteractions();

$job->handle();

$job->assertReleased(delay: 30);
$job->assertDeleted();
$job->assertNotDeleted();
$job->assertFailed();
$job->assertFailedWith(CorruptedAudioException::class);
$job->assertNotFailed();
```

<a name="job-events"></a>
## 任務事件

透過使用 `Queue` [Facade](/docs/{{version}}/facades) 的 `before` 與 `after` 方法，您可以指定在處理佇列任務之前或之後執行的回呼 (Callback)。這些回呼是執行額外記錄或增加儀表板統計數據的絕佳機會。通常，您應該在 [服務提供者(Service Providers)](/docs/{{version}}/providers) 的 `boot` 方法中呼叫這些方法。例如，我們可以使用 Laravel 內建的 `AppServiceProvider`：

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Queue;
use Illuminate\Support\ServiceProvider;
use Illuminate\Queue\Events\JobProcessed;
use Illuminate\Queue\Events\JobProcessing;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register(): void
    {
        // ...
    }

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Queue::before(function (JobProcessing $event) {
            // $event->connectionName
            // $event->job
            // $event->job->payload()
        });

        Queue::after(function (JobProcessed $event) {
            // $event->connectionName
            // $event->job
            // $event->job->payload()
        });
    }
}
```

透過使用 `Queue` [Facade](/docs/{{version}}/facades) 的 `looping` 方法，您可以指定在工作程式嘗試從佇列中抓取任務之前執行的回呼。例如，您可以註冊一個匿名函式來回滾 (Rollback) 任何由先前失敗任務所遺留且未關閉的交易：

```php
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Queue;

Queue::looping(function () {
    while (DB::transactionLevel() > 0) {
        DB::rollBack();
    }
});
```