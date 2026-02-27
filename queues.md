# Queues

- [簡介](#introduction)
    - [連線 vs. 佇列](#connections-vs-queues)
    - [驅動程式備註與前置需求](#driver-prerequisites)
- [建立 Job](#creating-jobs)
    - [產生 Job 類別](#generating-job-classes)
    - [類別結構](#class-structure)
    - [唯一 Job](#unique-jobs)
    - [加密 Job](#encrypted-jobs)
- [Job 中介層](#job-middleware)
    - [速率限制](#rate-limiting)
    - [防止 Job 重疊](#preventing-job-overlaps)
    - [限制例外次數](#throttling-exceptions)
    - [跳過 Job](#skipping-jobs)
- [分發 Job](#dispatching-jobs)
    - [延遲分發](#delayed-dispatching)
    - [同步分發](#synchronous-dispatching)
    - [Job 與資料庫交易](#jobs-and-database-transactions)
    - [Job 鏈結](#job-chaining)
    - [自訂佇列與連線](#customizing-the-queue-and-connection)
    - [指定最大 Job 重試次數 / 逾時值](#max-job-attempts-and-timeout)
    - [SQS FIFO 與公平佇列](#sqs-fifo-and-fair-queues)
    - [佇列故障轉移](#queue-failover)
    - [錯誤處理](#error-handling)
- [Job 批次處理](#job-batching)
    - [定義可批次處理的 Job](#defining-batchable-jobs)
    - [分發批次](#dispatching-batches)
    - [鏈結與批次](#chains-and-batches)
    - [新增 Job 至批次](#adding-jobs-to-batches)
    - [檢查批次](#inspecting-batches)
    - [取消批次](#cancelling-batches)
    - [批次失敗](#batch-failures)
    - [清理批次](#pruning-batches)
    - [將批次儲存在 DynamoDB](#storing-batches-in-dynamodb)
- [將 Closure 排入佇列](#queueing-closures)
- [執行佇列工作者](#running-the-queue-worker)
    - [`queue:work` 命令](#the-queue-work-command)
    - [佇列優先順序](#queue-priorities)
    - [佇列工作者與部署](#queue-workers-and-deployment)
    - [Job 到期與逾時](#job-expirations-and-timeouts)
    - [暫停與恢復佇列工作者](#pausing-and-resuming-queue-workers)
- [Supervisor 設定](#supervisor-configuration)
- [處理失敗的 Job](#dealing-with-failed-jobs)
    - [失敗 Job 的清理工作](#cleaning-up-after-failed-jobs)
    - [重試失敗的 Job](#retrying-failed-jobs)
    - [忽略遺失的模型](#ignoring-missing-models)
    - [清理失敗的 Job](#pruning-failed-jobs)
    - [將失敗的 Job 儲存在 DynamoDB](#storing-failed-jobs-in-dynamodb)
    - [停用失敗 Job 儲存](#disabling-failed-job-storage)
    - [失敗 Job 事件](#failed-job-events)
- [清除佇列中的 Job](#clearing-jobs-from-queues)
- [監控您的佇列](#monitoring-your-queues)
- [測試](#testing)
    - [模擬一部分的 Job](#faking-a-subset-of-jobs)
    - [測試 Job 鏈結](#testing-job-chains)
    - [測試 Job 批次](#testing-job-batches)
    - [測試 Job / 佇列互動](#testing-job-queue-interactions)
- [Job 事件](#job-events)

<a name="introduction"></a>
## 簡介

在開發網頁應用程式時，您可能會有一些任務，例如解析並儲存上傳的 CSV 檔案，這些任務在一般的網頁請求中執行需要耗費太長時間。幸運的是， Laravel 讓您能輕鬆建立可在背景處理的排隊任務 (queued jobs)。藉由將耗時任務移至佇列 (queue)，您的應用程式能以極快的速度回應網頁請求，並為客戶提供更好的使用者體驗。

Laravel 佇列為多種不同的佇列後端（如 [Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 或甚至是關聯式資料庫）提供了一套統一的佇列 API。

Laravel 的佇列設定選項儲存在應用程式的 `config/queue.php` 設定檔中。在此檔案中，您將找到框架內建的每個佇列驅動程式的連線設定，包含資料庫、[Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 和 [Beanstalkd](https://beanstalkd.github.io/) 驅動程式，以及一個會立即執行任務的同步驅動程式（用於開發或測試）。此外也包含一個會丟棄排隊任務的 `null` 佇列驅動程式。

> [!NOTE]
> Laravel Horizon 是一個為您的 Redis 佇列打造的精美儀表板與設定系統。請參考完整的 [Horizon 文件](/docs/{{version}}/horizon) 以取得更多資訊。


<a name="connections-vs-queues"></a>
### 連線 vs. 佇列

在開始使用 Laravel 佇列之前，瞭解「連線 (connections)」與「佇列 (queues)」之間的區別非常重要。在您的 `config/queue.php` 設定檔中，有一個 `connections` 設定陣列。此選項定義了與後端佇列服務（如 Amazon SQS、Beanstalk 或 Redis）的連線。然而，任何給定的佇列連線都可以擁有多個「佇列」，這些佇列可以被視為排隊任務的不同堆疊或堆湊。

請注意，`queue` 設定檔中的每個連線設定範例都包含一個 `queue` 屬性。這是當任務被發送到給定連線時，預設會被分發到的佇列。換句話說，如果您在分發任務時沒有明確定義它應該被分發到哪個佇列，該任務將會被放置在連線設定中 `queue` 屬性所定義的佇列上：

```php
use App\Jobs\ProcessPodcast;

// This job is sent to the default connection's default queue...
ProcessPodcast::dispatch();

// This job is sent to the default connection's "emails" queue...
ProcessPodcast::dispatch()->onQueue('emails');
```

某些應用程式可能永遠不需要將任務推送到多個佇列，而是傾向於使用一個簡單的佇列。然而，將任務推送到多個佇列對於希望優先處理或分割任務處理方式的應用程式來說特別有用，因為 Laravel 佇列工作者允許您指定應按優先順序處理哪些佇列。例如，如果您將任務推送到 `high` 佇列，您可以執行一個給予它們更高處理優先權的工作者：

```shell
php artisan queue:work --queue=high,default
```


<a name="driver-prerequisites"></a>
### 驅動程式備註與前置需求


<a name="database"></a>
#### 資料庫

為了使用 `database` 佇列驅動程式，您需要一個資料庫表來存放任務。通常，這已包含在 Laravel 預設的 `0001_01_01_000002_create_jobs_table.php` [資料庫遷移](/docs/{{version}}/migrations) 中；然而，如果您的應用程式不包含此遷移，您可以使用 `make:queue-table` Artisan 命令來建立它：

```shell
php artisan make:queue-table

php artisan migrate
```


<a name="redis"></a>
#### Redis

為了使用 `redis` 佇列驅動程式，您應該在 `config/database.php` 設定檔中設定 Redis 資料庫連線。

> [!WARNING]
> `redis` 佇列驅動程式不支援 `serializer` 和 `compression` 的 Redis 選項。


<a name="redis-cluster"></a>
##### Redis 叢集

如果您的 Redis 佇列連線使用的是 [Redis 叢集 (Redis Cluster)](https://redis.io/docs/latest/operate/rs/databases/durability-ha/clustering)，則您的佇列名稱必須包含 [Key Hash Tag](https://redis.io/docs/latest/develop/using-commands/keyspace/#hashtags)。這是為了確保給定佇列的所有 Redis Key 都被放置在同一個 Hash Slot 中：

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

使用 Redis 佇列時，您可以使用 `block_for` 設定選項來指定驅動程式在循環工作者迴圈並重新輪詢 Redis 資料庫之前，應該等待任務變為可用的時間。

根據您的佇列負載調整此值，會比持續輪詢 Redis 資料庫以獲取新任務更有效率。例如，您可以將該值設定為 `5`，表示驅動程式在等待任務變為可用時應阻塞五秒：

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
> 將 `block_for` 設定為 `0` 會導致佇列工作者無限期阻塞，直到有任務可用。這也會防止訊號（例如 `SIGTERM`）在下一個任務被處理之前被處理。


<a name="other-driver-prerequisites"></a>
#### 其他驅動程式的前置需求

下列佇列驅動程式需要對應的依賴套件。這些依賴套件可以透過 Composer 套件管理員安裝：

<div class="content-list" markdown="1">

- Amazon SQS: `aws/aws-sdk-php ~3.0`
- Beanstalkd: `pda/pheanstalk ~5.0`
- Redis: `predis/predis ~2.0` 或 phpredis PHP 擴充功能
- [MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/queues/): `mongodb/laravel-mongodb`

</div>

<a name="creating-jobs"></a>
## 建立 Job


<a name="generating-job-classes"></a>
### 產生 Job 類別

預設情況下，應用程式中所有可排入佇列的 Job 都儲存在 `app/Jobs` 目錄中。如果 `app/Jobs` 目錄不存在，當您執行 `make:job` Artisan 命令時，系統將會自動建立該目錄：

```shell
php artisan make:job ProcessPodcast
```

產生的類別將實作 `Illuminate\Contracts\Queue\ShouldQueue` 介面，這向 Laravel 表示該 Job 應該被推送到佇列中以非同步執行。

> [!NOTE]
> Job stub 可以使用 [stub 發佈](/docs/{{version}}/artisan#stub-customization)進行自訂。


<a name="class-structure"></a>
### 類別結構

Job 類別非常簡單，通常只包含一個 `handle` 方法，該方法在佇列處理 Job 時被呼叫。首先，讓我們看一個範例 Job 類別。在這個範例中，我們假設我們管理一個播客 (Podcast) 發佈服務，並且需要在上傳的播客檔案發佈之前對其進行處理：

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

在此範例中，請注意我們能夠將 [Eloquent 模型](/docs/{{version}}/eloquent)直接傳遞給已排入佇列的 Job 的建構子。由於 Job 使用了 `Queueable` trait，Eloquent 模型及其已載入的關聯在 Job 處理時將會被優雅地序列化 (Serialize) 與反序列化 (Unserialize)。

如果您的佇列 Job 在其建構子中接收 Eloquent 模型，則只有該模型的識別碼會被序列化到佇列中。當 Job 實際被處理時，佇列系統將自動從資料庫中重新取得完整的模型實例及其載入的關聯。這種模型序列化的方法可以讓發送到佇列驅動程式的 Job 負載 (Payload) 小得多。


<a name="handle-method-dependency-injection"></a>
#### `handle` 方法的依賴注入

當 Job 由佇列處理時，會呼叫 `handle` 方法。請注意，我們可以在 Job 的 `handle` 方法上對依賴項進行型別提示 (Type-hint)。Laravel [服務容器](/docs/{{version}}/container)會自動注入這些依賴項。

如果您想完全控制容器如何將依賴項注入 `handle` 方法，可以使用容器的 `bindMethod` 方法。`bindMethod` 方法接受一個回呼 (Callback)，該回呼接收 Job 與容器。在回呼中，您可以隨意呼叫 `handle` 方法。通常，您應該在 `App\Providers\AppServiceProvider` [服務提供者](/docs/{{version}}/providers)的 `boot` 方法中呼叫此方法：

```php
use App\Jobs\ProcessPodcast;
use App\Services\AudioProcessor;
use Illuminate\Contracts\Foundation\Application;

$this->app->bindMethod([ProcessPodcast::class, 'handle'], function (ProcessPodcast $job, Application $app) {
    return $job->handle($app->make(AudioProcessor::class));
});
```

> [!WARNING]
> 二進位資料（如原始圖片內容）在傳遞給佇列 Job 之前，應先通過 `base64_encode` 函式處理。否則，Job 在放入佇列時可能無法正確地序列化為 JSON。


<a name="handling-relationships"></a>
#### 佇列關聯

因為所有已載入的 Eloquent 模型關聯在 Job 排入佇列時也會被序列化，所以序列化後的 Job 字串有時會變得相當龐大。此外，當 Job 被反序列化並從資料庫重新取得模型關聯時，它們將會被完整地取得。在 Job 排入佇列過程中，模型序列化前所套用的任何先前關聯約束，在 Job 反序列化時都不會被套用。因此，如果您想處理給定關聯的子集，則應該在佇列 Job 中重新約束該關聯。

或者，為了防止關聯被序列化，您可以在設定屬性值時對模型呼叫 `withoutRelations` 方法。此方法將回傳一個不包含已載入關聯的模型實例：

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

如果您正在使用 [PHP 建構子屬性提升 (Constructor Property Promotion)](https://www.php.net/manual/en/language.oop5.decon.php#language.oop5.decon.constructor.promotion)，並且想要表示某個 Eloquent 模型不應序列化其關聯，可以使用 `WithoutRelations` 屬性：

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

為了方便起見，如果您希望序列化所有不具備關聯的模型，可以將 `WithoutRelations` 屬性套用於整個類別，而不是套用於每個模型：

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

如果 Job 接收的是 Eloquent 模型的集合 (Collection) 或陣列，而不是單個模型，則在 Job 反序列化並執行時，該集合內的模型將不會恢復其關聯。這是為了防止在處理大量模型的 Job 上消耗過多資源。

<a name="unique-jobs"></a>
### 唯一 Job

> [!WARNING]
> 唯一 Job 需要支援 [鎖定 (locks)](/docs/{{version}}/cache#atomic-locks) 的快取驅動程式。目前 `memcached`, `redis`, `dynamodb`, `database`, `file`, 和 `array` 快取驅動程式都支援原子鎖。

> [!WARNING]
> 唯一 Job 的限制不適用於批次 (batches) 中的 Job。

有時候，您可能希望確保在任何時間點，佇列中只有一個特定 Job 的實例。您可以透過在 Job 類別實作 `ShouldBeUnique` 介面來達成此目的。此介面不需要您在類別中定義任何額外的方法：

```php
<?php

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Queue\ShouldBeUnique;

class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    // ...
}
```

在上面的範例中，`UpdateSearchIndex` Job 是唯一的。因此，如果該 Job 的另一個實例已經在佇列中且尚未完成處理，則該 Job 將不會被分發。

在某些情況下，您可能想要定義一個特定的「key」來使 Job 變得唯一，或者您可能想要指定一個逾時時間，超過該時間後該 Job 就不再保持唯一。為了達成此目的，您可以在 Job 類別定義 `uniqueId` 和 `uniqueFor` 屬性或方法：

```php
<?php

namespace App\Jobs;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Queue\ShouldBeUnique;

class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    /**
     * The product instance.
     *
     * @var \App\Models\Product
     */
    public $product;

    /**
     * The number of seconds after which the job's unique lock will be released.
     *
     * @var int
     */
    public $uniqueFor = 3600;

    /**
     * Get the unique ID for the job.
     */
    public function uniqueId(): string
    {
        return $this->product->id;
    }
}
```

在上面的範例中，`UpdateSearchIndex` Job 根據產品 ID 保持唯一。因此，任何具有相同產品 ID 的新 Job 分發都將被忽略，直到現有 Job 完成處理。此外，如果現有 Job 在一小時內未處理完成，唯一鎖將被釋放，另一個具有相同唯一 key 的 Job 即可被分發到佇列中。

> [!WARNING]
> 如果您的應用程式從多個網頁伺服器或容器分發 Job，您應該確保所有伺服器都在與同一個中央快取伺服器通訊，以便 Laravel 可以準確判斷 Job 是否唯一。

<a name="keeping-jobs-unique-until-processing-begins"></a>
#### 保持 Job 唯一直到開始處理

預設情況下，唯一 Job 會在完成處理或所有重試嘗試均失敗後「解鎖」。然而，在某些情況下，您可能希望 Job 在處理之前立即解鎖。要達成此目的，您的 Job 應該實作 `ShouldBeUniqueUntilProcessing` 合約，而不是 `ShouldBeUnique` 合約：

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
#### 唯一 Job 鎖定

在幕後，當分發 `ShouldBeUnique` Job 時，Laravel 會嘗試獲取一個以 `uniqueId` 為 key 的 [鎖定 (lock)](/docs/{{version}}/cache#atomic-locks)。如果鎖定已被持有，則不會分發 Job。當 Job 完成處理或所有重試嘗試均失敗時，此鎖定會被釋放。預設情況下，Laravel 會使用預設的快取驅動程式來取得此鎖定。但是，如果您希望使用另一個驅動程式來獲取鎖定，您可以定義一個 `uniqueVia` 方法來回傳應使用的快取驅動程式：

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
> 如果您只需要限制 Job 的並行處理，請改用 [WithoutOverlapping](/docs/{{version}}/queues#preventing-job-overlaps) Job 中介層。

<a name="encrypted-jobs"></a>
### 加密 Job

Laravel 允許您透過 [加密](/docs/{{version}}/encryption) 來確保 Job 資料的隱私與完整性。要開始使用，只需將 `ShouldBeEncrypted` 介面加入到 Job 類別即可。一旦類別加入了此介面，Laravel 就會在將 Job 推送到佇列之前自動對其進行加密：

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
## Job 中介層

Job 中介層允許您在執行佇列中的 Job 時包裝自訂邏輯，從而減少 Job 本身的樣板程式碼。例如，考慮以下 `handle` 方法，它利用 Laravel 的 Redis 速率限制功能，每五秒僅允許處理一個 Job：

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

雖然這段程式碼是有效的，但 `handle` 方法的實作會變得雜亂，因為它充斥著 Redis 的速率限制邏輯。此外，對於任何我們想要限制速率的其他 Job，都必須重複這段速率限制邏輯。與其在 handle 方法中限制速率，我們可以定義一個處理速率限制的 Job 中介層：

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

如您所見，就像 [路由中介層](/docs/{{version}}/middleware)，Job 中介層會接收正在處理的 Job 以及一個應該被呼叫以繼續處理 Job 的回呼 (Callback)。

您可以使用 `make:job-middleware` Artisan 命令產生新的 Job 中介層類別。建立 Job 中介層後，可以透過 Job 的 `middleware` 方法回傳它們，進而將其附加到 Job 上。由 `make:job` Artisan 命令生成的 Job 預設不包含此方法，因此您需要手動將其新增到您的 Job 類別中：

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
> Job 中介層也可以分配給 [可排入佇列的事件監聽器](/docs/{{version}}/events#queued-event-listeners)、[Mailables](/docs/{{version}}/mail#queueing-mail) 和 [通知](/docs/{{version}}/notifications#queueing-notifications)。


<a name="rate-limiting"></a>
### 速率限制

雖然我們剛剛展示了如何編寫自己的速率限制 Job 中介層，但 Laravel 實際上已經包含了一個可以用來限制 Job 速率的速率限制中介層。就像 [路由速率限制器](/docs/{{version}}/routing#defining-rate-limiters)，Job 速率限制器是使用 `RateLimiter` Facade 的 `for` 方法定義的。

例如，您可能希望允許使用者每小時備份一次數據，但對高級客戶不加限制。要達成此目的，您可以在 `AppServiceProvider` 的 `boot` 方法中定義一個 `RateLimiter`：

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

在上面的範例中，我們定義了每小時的速率限制；但是，您可以使用 `perMinute` 方法輕鬆地定義基於分鐘的速率限制。此外，您可以將任何您想要的值傳遞給速率限制的 `by` 方法；然而，這個值最常用於按客戶區分速率限制：

```php
return Limit::perMinute(50)->by($job->user->id);
```

一旦定義了速率限制，您就可以使用 `Illuminate\Queue\Middleware\RateLimited` 中介層將該速率限制器附加到 Job 上。每當 Job 超過速率限制時，此中介層會根據速率限制持續時間，以適當的延遲將 Job 釋放回佇列中：

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

將受速率限制的 Job 釋放回佇列仍會增加該 Job 的 `attempts` 總數。您可能需要相應地調整 Job 類別上的 `tries` 和 `maxExceptions` 屬性。或者，您可能希望使用 [retryUntil 方法](#time-based-attempts) 來定義 Job 不再嘗試之前的時間長度。

使用 `releaseAfter` 方法，您還可以指定釋放後的 Job 在再次嘗試之前必須經過的秒數：

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

如果您不希望在 Job 受到速率限制時進行重試，您可以使用 `dontRelease` 方法：

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

如果您正在使用 Redis，可以使用 `Illuminate\Queue\Middleware\RateLimitedWithRedis` 中介層，該中介層針對 Redis 進行了微調，比基本的速率限制中介層更有效率：

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
### 防止 Job 重疊

Laravel 包含了一個 `Illuminate\Queue\Middleware\WithoutOverlapping` 中介層，讓您可以根據自訂的鍵值來防止 Job 重疊。當一個排入佇列的 Job 正在修改某個同時只能被一個 Job 修改的資源時，這會非常有用。

例如，想像您有一個排入佇列的 Job 用於更新使用者的信用分數，而您想要防止同一個使用者 ID 的信用分數更新 Job 重疊。為了達成這個目的，您可以從 Job 的 `middleware` 方法中回傳 `WithoutOverlapping` 中介層：

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

將重疊的 Job 釋放回佇列仍會增加該 Job 的總嘗試次數。您可能需要相應地調整 Job 類別中的 `tries` 與 `maxExceptions` 屬性。例如，將 `tries` 屬性保持為預設值 1，將會防止任何重疊的 Job 在稍後重試。

任何同類型的重疊 Job 都會被釋放回佇列。您也可以指定釋放後的 Job 在重新嘗試前必須經過的秒數：

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

如果您希望立即刪除任何重疊的 Job 以使其不再重試，可以使用 `dontRelease` 方法：

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

`WithoutOverlapping` 中介層是由 Laravel 的原子鎖 (Atomic Lock) 功能所驅動。有時候，您的 Job 可能會意外失敗或逾時，導致鎖定未被釋放。因此，您可以使用 `expireAfter` 方法明確定義鎖定過期時間。例如，下方的範例將指示 Laravel 在 Job 開始處理三分鐘後釋放 `WithoutOverlapping` 鎖定：

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
#### 跨 Job 類別共享鎖定鍵值

預設情況下，`WithoutOverlapping` 中介層只會防止相同類別的 Job 重疊。因此，即使兩個不同的 Job 類別使用相同的鎖定鍵值，它們也不會被阻止重疊。然而，您可以使用 `shared` 方法指示 Laravel 跨 Job 類別套用鍵值：

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
### 限制例外次數

Laravel 包含了一個 `Illuminate\Queue\Middleware\ThrottlesExceptions` 中介層，允許您限制例外次數。一旦 Job 拋出特定次數的例外，所有後續執行該 Job 的嘗試都將延遲，直到指定的間隔時間過後。此中介層對於與不穩定的第三方服務互動的 Job 特別有用。

例如，假設有一個與第三方 API 互動的佇列 Job 開始拋出例外。要限制例外，您可以從 Job 的 `middleware` 方法回傳 `ThrottlesExceptions` 中介層。通常，此中介層應與實作了 [基於時間重試](#time-based-attempts) 的 Job 配合使用：

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

中介層接受的第一個建構函式參數是 Job 在被限制之前可以拋出的例外次數，而第二個建構函式參數是限制後再次嘗試 Job 之前應經過的秒數。在上面的程式碼範例中，如果 Job 連續拋出 10 次例外，我們將等待 5 分鐘後再嘗試該 Job，並受 30 分鐘時間限制的約束。

當 Job 拋出例外但尚未達到例外閾值時，Job 通常會立即重試。但是，您可以透過在將中介層附加到 Job 時呼叫 `backoff` 方法來指定此類 Job 應延遲的分鐘數：

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

在內部，此中介層使用 Laravel 的快取系統來實作速率限制，並將 Job 的類別名稱用作快取「key」。您可以在將中介層附加到 Job 時呼叫 `by` 方法來覆寫此 key。如果您有多個與同一個第三方服務互動的 Job，並且希望它們共享一個通用的限制「值（bucket）」，以確保它們遵守單一的共享限制，這將非常有用：

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

預設情況下，此中介層會限制每個例外。您可以透過在將中介層附加到 Job 時呼叫 `when` 方法來修改此行為。只有當提供給 `when` 方法的 Closure 回傳 `true` 時，該例外才會被限制：

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

與 `when` 方法不同（後者會將 Job 重新放回佇列或拋出例外），`deleteWhen` 方法允許您在發生特定例外時完全刪除 Job：

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

如果您希望將受限制的例外報告給應用程式的例外處理常式，可以在將中介層附加到 Job 時呼叫 `report` 方法。或者，您可以為 `report` 方法提供一個 Closure，只有當該 Closure 回傳 `true` 時，才會報告該例外：

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
#### 使用 Redis 限制例外次數

如果您正在使用 Redis，可以使用 `Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis` 中介層，該中介層針對 Redis 進行了最佳化，比基礎的例外限制中介層更有效率：

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
### 跳過 Job

`Skip` 中介層允許您指定應跳過 / 刪除 Job，而無需修改 Job 的邏輯。當給定條件評估為 `true` 時，`Skip::when` 方法將刪除 Job；而當條件評估為 `false` 時，`Skip::unless` 方法將刪除 Job：

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

您也可以將 `Closure` 傳遞給 `when` 和 `unless` 方法，以進行更複雜的條件評估：

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
## 分發 Job

當您編寫完 Job 類別後，您可以使用 Job 本身的 `dispatch` 方法來分發它。傳遞給 `dispatch` 方法的參數將會被提供給 Job 的建構函式：

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

如果您想要有條件地分發 Job，您可以使用 `dispatchIf` 和 `dispatchUnless` 方法：

```php
ProcessPodcast::dispatchIf($accountActive, $podcast);

ProcessPodcast::dispatchUnless($accountSuspended, $podcast);
```

在新的 Laravel 應用程式中，`database` 連線被定義為預設佇列。您可以透過更改應用程式 `.env` 檔案中的 `QUEUE_CONNECTION` 環境變數來指定不同的預設佇列連線。


<a name="delayed-dispatching"></a>
### 延遲分發

如果您想指定 Job 不應立即供佇列工作者處理，您可以在分發 Job 時使用 `delay` 方法。例如，讓我們指定一個 Job 在分發後 10 分鐘內不開放處理：

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

在某些情況下，Job 可能配置了預設延遲。如果您需要繞過此延遲並分發 Job 以進行立即處理，可以使用 `withoutDelay` 方法：

```php
ProcessPodcast::dispatch($podcast)->withoutDelay();
```

> [!WARNING]
> Amazon SQS 佇列服務的最大延遲時間為 15 分鐘。


<a name="synchronous-dispatching"></a>
### 同步分發

如果您想立即（同步）分發 Job，可以使用 `dispatchSync` 方法。使用此方法時，Job 將不會排入佇列，而是立即在當前程序中執行：

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
#### 延期分發

使用延期同步分發，您可以分發一個 Job 在當前程序中處理，但在 HTTP 回應發送給使用者之後才執行。這讓您能同步處理「佇列化」的 Job，而不會降低使用者的應用程式體驗。要延期執行同步 Job，請將 Job 分發到 `deferred` 連線：

```php
RecordDelivery::dispatch($order)->onConnection('deferred');
```

`deferred` 連線也作為預設的 [佇列故障轉移](#queue-failover)。

同樣地，`background` 連線會在 HTTP 回應發送給使用者後處理 Job；然而，該 Job 會在單獨啟動的 PHP 程序中處理，這讓 PHP-FPM / 應用程式工作者可以空出來處理另一個傳入的 HTTP 請求：

```php
RecordDelivery::dispatch($order)->onConnection('background');
```


<a name="jobs-and-database-transactions"></a>
### Job 與資料庫交易

雖然在資料庫交易中分發 Job 是完全沒問題的，但您應該特別注意確保 Job 實際上能夠成功執行。在交易中分發 Job 時，Job 可能在父交易提交之前就已經被工作者處理了。當這種情況發生時，您在資料庫交易期間對模型或資料庫紀錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫紀錄可能還不存在於資料庫中。

幸好，Laravel 提供了幾種方法來解決這個問題。首先，您可以在佇列連線的設定陣列中設定 `after_commit` 連線選項：

```php
'redis' => [
    'driver' => 'redis',
    // ...
    'after_commit' => true,
],
```

當 `after_commit` 選項為 `true` 時，您可以在資料庫交易中分發 Job；但是，Laravel 會等到開啟的父資料庫交易提交後，才會真正分發 Job。當然，如果當前沒有開啟任何資料庫交易，Job 將會立即分發。

如果交易因為交易期間發生的例外而還原，則在該交易期間分發的 Job 將會被捨棄。

> [!NOTE]
> 將 `after_commit` 設定選項設為 `true` 也會導致任何佇列事件監聽器、Mailable、通知和廣播事件在所有開啟的資料庫交易提交後才被分發。


<a name="specifying-commit-dispatch-behavior-inline"></a>
#### 以行內方式指定提交分發行為

如果您沒有將 `after_commit` 佇列連線設定選項設為 `true`，您仍然可以指示特定的 Job 應在所有開啟的資料庫交易提交後才分發。要實現這一點，您可以在分發操作上鏈結 `afterCommit` 方法：

```php
use App\Jobs\ProcessPodcast;

ProcessPodcast::dispatch($podcast)->afterCommit();
```

同樣地，如果 `after_commit` 設定選項設為 `true`，您可以指示特定的 Job 應立即分發，而不必等待任何開啟的資料庫交易提交：

```php
ProcessPodcast::dispatch($podcast)->beforeCommit();
```

<a name="job-chaining"></a>
### Job 鏈結

Job 鏈結允許您指定一系列在主要 Job 成功執行後應按順序運行的佇列 Job。如果序列中的其中一個 Job 失敗，其餘的 Job 將不會被執行。若要執行佇列 Job 鏈結，您可以使用 `Bus` Facade 提供的 `chain` 方法。Laravel 的命令匯流排 (Command Bus) 是建構佇列 Job 分發功能的底層元件：

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

除了鏈結 Job 類別實例，您也可以鏈結 Closure：

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
> 在 Job 內使用 `$this->delete()` 方法刪除 Job 並不會阻止鏈結中的 Job 被處理。只有當鏈結中的某個 Job 失敗時，鏈結才會停止執行。


<a name="chain-connection-queue"></a>
#### 鏈結的連線與佇列

如果您想指定鏈結 Job 應使用的連線與佇列，可以使用 `onConnection` 與 `onQueue` 方法。這些方法指定了應使用的佇列連線與佇列名稱，除非該佇列 Job 已被明確指派不同的連線 / 佇列：

```php
Bus::chain([
    new ProcessPodcast,
    new OptimizePodcast,
    new ReleasePodcast,
])->onConnection('redis')->onQueue('podcasts')->dispatch();
```


<a name="adding-jobs-to-the-chain"></a>
#### 新增 Job 至鏈結

有時，您可能需要在鏈結中的某個 Job 內，將一個 Job 加到現有 Job 鏈結的最前面或最後面。您可以透過 `prependToChain` 與 `appendToChain` 方法來達成：

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
#### 鏈結失敗

鏈結 Job 時，您可以使用 `catch` 方法來指定當鏈結中的 Job 失敗時應呼叫的 Closure。該回呼將接收導致 Job 失敗的 `Throwable` 實例：

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
> 由於鏈結回呼會被序列化並在稍後由 Laravel 佇列執行，因此您不應在鏈結回呼中使用 `$this` 變數。


<a name="customizing-the-queue-and-connection"></a>
### 自訂佇列與連線


<a name="dispatching-to-a-particular-queue"></a>
#### 分發至特定佇列

透過將 Job 推送到不同的佇列，您可以「分類」您的佇列 Job，甚至可以優先安排分配給各個佇列的工作者 (Worker) 數量。請記住，這並不會將 Job 推送到佇列設定檔中定義的不同佇列「連線」，而僅是推送到單一連線中的特定佇列。若要指定佇列，請在分發 Job 時使用 `onQueue` 方法：

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

或者，您也可以在 Job 的建構子內呼叫 `onQueue` 方法來指定 Job 的佇列：

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
#### 分發至特定連線

如果您的應用程式與多個佇列連線互動，您可以使用 `onConnection` 方法指定要將 Job 推送到哪個連線：

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

您可以將 `onConnection` 與 `onQueue` 方法鏈結在一起，為 Job 指定連線與佇列：

```php
ProcessPodcast::dispatch($podcast)
    ->onConnection('sqs')
    ->onQueue('processing');
```

或者，您也可以在 Job 的建構子內呼叫 `onConnection` 方法來指定 Job 的連線：

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

<a name="max-job-attempts-and-timeout"></a>
### 指定最大 Job 重試次數 / 逾時值

<a name="max-attempts"></a>
#### 最大重試次數

Job 重試次數是 Laravel 佇列系統的核心概念，並驅動了許多進階功能。雖然一開始看起來可能很令人困惑，但在修改預設設定之前，了解其運作方式非常重要。

當一個 Job 被分發時，它會被推送到佇列中。接著工作者會取得它並嘗試執行。這就是一次 Job 重試。

然而，一次重試並不一定代表 Job 的 `handle` 方法已經執行。重試次數也可能以多種方式被「消耗」：

<div class="content-list" markdown="1">

- Job 在執行期間遇到未處理的例外。
- Job 使用 `$this->release()` 手動釋放回佇列。
- 中介層（如 `WithoutOverlapping` 或 `RateLimited`）取得鎖定失敗並釋放 Job。
- Job 逾時。
- Job 的 `handle` 方法執行並完成，且未拋出例外。

</div>

您可能不想無限期地嘗試執行某個 Job。因此，Laravel 提供了多種方式來指定 Job 可以重試的次數或時長。

> [!NOTE]
> 預設情況下，Laravel 只會嘗試執行 Job 一次。如果您的 Job 使用了像 `WithoutOverlapping` 或 `RateLimited` 這樣的中介層，或者您正在手動釋放 Job，您可能需要透過 `tries` 選項增加允許的重試次數。

指定 Job 最大重試次數的一種方法是透過 Artisan 命令列上的 `--tries` 切換參數。除非正在處理的 Job 指定了可重試的次數，否則這將套用於工作者處理的所有 Job：

```shell
php artisan queue:work --tries=3
```

如果 Job 超過其最大重試次數，它將被視為「失敗」的 Job。有關處理失敗 Job 的更多資訊，請參閱[處理失敗的 Job](#dealing-with-failed-jobs)。如果將 `--tries=0` 提供給 `queue:work` 命令，該 Job 將無限次地重試。

您可以透過在 Job 類別本身定義 Job 最大重試次數，來採取更細粒度的方法。如果在 Job 上指定了最大重試次數，它的優先權將高於命令列上提供的 `--tries` 值：

```php
<?php

namespace App\Jobs;

class ProcessPodcast implements ShouldQueue
{
    /**
     * The number of times the job may be attempted.
     *
     * @var int
     */
    public $tries = 5;
}
```

如果您需要對特定 Job 的最大重試次數進行動態控制，您可以在 Job 上定義一個 `tries` 方法：

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
#### 基於時間的重試

除了定義 Job 在失敗前可以重試的次數外，您也可以定義一個不再嘗試 Job 的時間。這允許 Job 在給定的時間範圍內進行任意次數的重試。要定義不再嘗試 Job 的時間，請在您的 Job 類別中加入 `retryUntil` 方法。此方法應回傳一個 `DateTime` 執行個體：

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

如果同時定義了 `retryUntil` 與 `tries`，Laravel 會優先採用 `retryUntil` 方法。

> [!NOTE]
> 您也可以在[排入佇列的事件監聽器](/docs/{{version}}/events#queued-event-listeners)和[排入佇列的通知](/docs/{{version}}/notifications#queueing-notifications)中定義 `tries` 屬性或 `retryUntil` 方法。

<a name="max-exceptions"></a>
#### 最大例外次數

有時您可能希望指定一個 Job 可以重試多次，但如果重試是由給定次數的未處理例外所觸發的（而不是直接由 `release` 方法釋放），則該 Job 應該失敗。要實現這一點，您可以在 Job 類別上定義 `maxExceptions` 屬性：

```php
<?php

namespace App\Jobs;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;
use Illuminate\Support\Facades\Redis;

class ProcessPodcast implements ShouldQueue
{
    use Queueable;

    /**
     * The number of times the job may be attempted.
     *
     * @var int
     */
    public $tries = 25;

    /**
     * The maximum number of unhandled exceptions to allow before failing.
     *
     * @var int
     */
    public $maxExceptions = 3;

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

在這個範例中，如果應用程式無法取得 Redis 鎖定，Job 會被釋放 10 秒，並將繼續重試最多 25 次。然而，如果 Job 拋出 3 個未處理的例外，則該 Job 將會失敗。

<a name="timeout"></a>
#### 逾時

通常，您大致知道您預期排隊的 Job 需要花費多長時間。因此，Laravel 允許您指定一個「逾時 (timeout)」值。預設情況下，逾時值為 60 秒。如果 Job 處理的時間超過逾時值指定的秒數，處理該 Job 的工作者將以錯誤結束。通常，工作者會由[在您的伺服器上設定的程序管理員](#supervisor-configuration)自動重啟。

可以使用 Artisan 命令列上的 `--timeout` 切換參數來指定 Job 可以執行的最大秒數：

```shell
php artisan queue:work --timeout=30
```

如果 Job 由於不斷逾時而超過其最大嘗試次數，它將被標記為失敗。

您也可以在 Job 類別本身定義 Job 允許執行的最大秒數。如果在 Job 上指定了逾時，它的優先權將高於命令列上指定的任何逾時值：

```php
<?php

namespace App\Jobs;

class ProcessPodcast implements ShouldQueue
{
    /**
     * The number of seconds the job can run before timing out.
     *
     * @var int
     */
    public $timeout = 120;
}
```

有時，通訊端 (sockets) 或外連 HTTP 連線等 IO 阻塞程序可能不會遵守您指定的逾時值。因此，在使用這些功能時，您應該始終嘗試使用其 API 來指定逾時值。例如，使用 [Guzzle](https://docs.guzzlephp.org) 時，您應該始終指定連線和請求的逾時值。

> [!WARNING]
> 必須安裝 [PCNTL](https://www.php.net/manual/en/book.pcntl.php) PHP 擴充功能才能指定 Job 逾時。此外，Job 的「逾時」值應始終小於其[「retry after」](#job-expiration)值。否則，Job 可能會在實際完成執行或逾時之前就被重新嘗試。

<a name="failing-on-timeout"></a>
#### 逾時時失敗

如果您希望指示 Job 在逾時時應標記為[失敗](#dealing-with-failed-jobs)，您可以在 Job 類別上定義 `$failOnTimeout` 屬性：

```php
/**
 * Indicate if the job should be marked as failed on timeout.
 *
 * @var bool
 */
public $failOnTimeout = true;
```

> [!NOTE]
> 預設情況下，當 Job 逾時時，它會消耗一次重試次數並被釋放回佇列（如果允許重試）。但是，如果您將 Job 設定為在逾時時失敗，則無論 tries 設定的值為何，都不會再重試該 Job。

<a name="sqs-fifo-and-fair-queues"></a>
### SQS FIFO 與公平佇列

Laravel 支援 [Amazon SQS FIFO (先進先出)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html) 佇列，允許您按照發送的確切順序處理 Job，同時透過訊息去重來確保恰好一次 (exactly-once) 的處理。

FIFO 佇列需要一個訊息群組 ID 來決定哪些 Job 可以平行處理。具有相同群組 ID 的 Job 會依序處理，而具有不同群組 ID 的訊息則可以並行處理。

Laravel 提供了一個流暢的 `onGroup` 方法，可在分發 Job 時指定訊息群組 ID：

```php
ProcessOrder::dispatch($order)
    ->onGroup("customer-{$order->customer_id}");
```

SQS FIFO 佇列支援訊息去重，以確保恰好一次的處理。請在您的 Job 類別中實作 `deduplicationId` 方法，以提供自訂的去重 ID：

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


<a name="fifo-listeners-mail-and-notifications"></a>
#### FIFO 接聽器、郵件與通知

使用 FIFO 佇列時，您還需要在接聽器、郵件 (Mail) 和通知 (Notification) 上定義訊息群組。或者，您可以將這些物件的佇列實例分發到非 FIFO 佇列。

要為 [佇列事件接聽器](/docs/{{version}}/events#queued-event-listeners) 定義訊息群組，請在接聽器上定義 `messageGroup` 方法。您也可以選擇性地定義 `deduplicationId` 方法：

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

當發送將要在 FIFO 佇列上排隊的 [郵件訊息](/docs/{{version}}/mail) 時，您應該在發送通知時呼叫 `onGroup` 方法，並可以選擇性地呼叫 `withDeduplicator` 方法：

```php
use App\Mail\InvoicePaid;
use Illuminate\Support\Facades\Mail;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

Mail::to($request->user())->send($invoicePaid);
```

當發送將要在 FIFO 佇列上排隊的 [通知](/docs/{{version}}/notifications) 時，您應該在發送通知時呼叫 `onGroup` 方法，並可以選擇性地呼叫 `withDeduplicator` 方法：

```php
use App\Notifications\InvoicePaid;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

$user->notify($invoicePaid);
```


<a name="queue-failover"></a>
### 佇列故障轉移

`failover` 佇列驅動程式在將 Job 推送到佇列時提供了自動故障轉移 (Failover) 功能。如果 `failover` 設定中的主要佇列連線因任何原因失敗，Laravel 將自動嘗試將 Job 推送到清單中下一個設定的連線。這對於確保佇列可靠性至關重要的正式環境中實現高可用性特別有用。

要設定故障轉移佇列連線，請指定 `failover` 驅動程式，並提供要依序嘗試的連線名稱陣列。預設情況下，Laravel 在應用程式的 `config/queue.php` 設定檔中包含了一個故障轉移設定範例：

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

設定完使用 `failover` 驅動程式的連線後，您需要將應用程式 `.env` 檔中的預設佇列連線設置為故障轉移連線，以便利用故障轉移功能：

```ini
QUEUE_CONNECTION=failover
```

接著，為故障轉移連線清單中的每個連線啟動至少一個工作者：

```bash
php artisan queue:work redis
php artisan queue:work database
```

> [!NOTE]
> 您不需要為使用 `sync`、`background` 或 `deferred` 佇列驅動程式的連線執行工作者，因為這些驅動程式會在目前的 PHP 程序中處理 Job。

當佇列連線操作失敗且故障轉移被啟動時，Laravel 將發送 `Illuminate\Queue\Events\QueueFailedOver` 事件，讓您可以回報或記錄該佇列連線已失敗。

> [!NOTE]
> 如果您使用 Laravel Horizon，請記住 Horizon 僅管理 Redis 佇列。如果您的故障轉移清單包含 `database`，您應該在 Horizon 之外執行一般的 `php artisan queue:work database` 程序。

<a name="error-handling"></a>
### 錯誤處理

若在處理 job 時拋出例外，該 job 將會自動被釋放回佇列中，以便再次嘗試。job 將持續被釋放，直到達到您應用程式所允許的最大嘗試次數。最大嘗試次數是由 `queue:work` Artisan 命令所使用的 `--tries` 切換參數定義的。或者，也可以在 job 類別本身定義最大嘗試次數。有關執行佇列工作者的更多資訊[可以在下方找到](#running-the-queue-worker)。


<a name="manually-releasing-a-job"></a>
#### 手動釋放 Job

有時您可能希望手動將 job 釋放回佇列，以便稍後再次嘗試。您可以透過呼叫 `release` 方法來完成此操作：

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

預設情況下，`release` 方法會將 job 釋放回佇列以立即處理。然而，您可以透過向 `release` 方法傳遞一個整數或日期實例，來指示佇列在經過指定秒數之前不要讓該 job 可供處理：

```php
$this->release(10);

$this->release(now()->plus(seconds: 10));
```


<a name="manually-failing-a-job"></a>
#### 手動使 Job 失敗

有時您可能需要手動將 job 標記為「失敗」。若要執行此操作，您可以呼叫 `fail` 方法：

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

如果您想因為擷取到的例外而將 job 標記為失敗，您可以將例外傳遞給 `fail` 方法。或者，為了方便起見，您可以傳遞一個字串錯誤訊息，它將為您轉換為例外：

```php
$this->fail($exception);

$this->fail('Something went wrong.');
```

> [!NOTE]
> 有關失敗 job 的更多資訊，請查看[處理失敗 job 的文件](#dealing-with-failed-jobs)。


<a name="fail-jobs-on-exceptions"></a>
#### 因特定例外而使 Job 失敗

`FailOnException` [job 中介層](#job-middleware) 允許您在拋出特定例外時直接中斷重試。這允許在發生暫時性例外（如外部 API 錯誤）時進行重試，但在發生持久性例外（如使用者的權限被撤銷）時則永久地使 job 失敗：

```php
<?php

namespace App\Jobs;

use App\Models\User;
use Illuminate\Auth\Access\AuthorizationException;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;
use Illuminate\Queue\Middleware\FailOnException;
use Illuminate\Support\Facades\Http;

class SyncChatHistory implements ShouldQueue
{
    use Queueable;

    public $tries = 3;

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
## Job 批次處理

Laravel 的 Job 批次處理 (Batching) 功能讓您可以輕鬆執行一批 Job，並在該批 Job 執行完成後執行某些動作。在開始之前，您應該建立一個資料庫遷移 (Migration) 來建立一張資料表，該表將包含有關 Job 批次的元資料 (Meta Information)，例如完成百分比。可以使用 `make:queue-batches-table` Artisan 命令來產生此遷移：

```shell
php artisan make:queue-batches-table

php artisan migrate
```


<a name="defining-batchable-jobs"></a>
### 定義可批次處理的 Job

要定義一個可批次處理的 Job，您應該像平常一樣 [建立一個可排入佇列的 Job](#creating-jobs)；但是，您應該在 Job 類別中加入 `Illuminate\Bus\Batchable` trait。這個 trait 提供了 `batch` 方法的存取權限，該方法可用於檢索目前 Job 正在其中執行的批次：

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
### 分發批次

要分發一整批 Job，您應該使用 `Bus` Facade 的 `batch` 方法。當然，批次處理在與完成回呼 (Completion Callbacks) 結合使用時最為有用。因此，您可以使用 `then`、`catch` 與 `finally` 方法來為批次定義完成回呼。這些回呼在被叫用時都會接收到一個 `Illuminate\Bus\Batch` 執行個體。

當執行多個佇列工作者時，批次中的 Job 將會平行處理。因此，Job 完成的順序可能與它們被加入到批次的順序不同。請參閱我們關於 [Job 鏈結與批次](#chains-and-batches) 的文件，以取得如何依序執行一系列 Job 的資訊。

在這個範例中，我們假設我們正在排入一整批 Job，每個 Job 都處理 CSV 檔案中指定數量的資料列：

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

批次的 ID 可以透過 `$batch->id` 屬性存取，可用於在分發批次後向 Laravel 命令匯流排 (Command Bus) [查詢批次的相關資訊](#inspecting-batches) 的資訊。

> [!WARNING]
> 由於批次回呼是由 Laravel 佇列序列化並在稍後執行的，因此您不應在回呼中使用 `$this` 變數。此外，由於批次處理的 Job 被包裝在資料庫交易中，因此觸發隱式提交 (Implicit Commits) 的資料庫語句不應在 Job 內執行。


<a name="naming-batches"></a>
#### 命名批次

如果批次有命名，[Horizon](/docs/{{version}}/horizon) 與 [Telescope](/docs/{{version}}/telescope) 等工具可能會為批次提供更友善的除錯資訊。要為批次分配一個任意名稱，您可以在定義批次時呼叫 `name` 方法：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import CSV')->dispatch();
```


<a name="batch-connection-queue"></a>
#### 批次連線與佇列

如果您想指定用於批次 Job 的連線與佇列，可以使用 `onConnection` 與 `onQueue` 方法。所有批次處理的 Job 都必須在同一個連線與佇列中執行：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->onConnection('redis')->onQueue('imports')->dispatch();
```


<a name="chains-and-batches"></a>
### 鏈結與批次

您可以透過將 [鏈結 Job](#job-chaining) 放入陣列中，在批次內定義一組鏈結 Job。例如，我們可以平行執行兩個 Job 鏈結，並在兩個 Job 鏈結都處理完成時執行一個回呼：

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

相反地，您也可以透過在 [鏈結](#job-chaining) 中定義批次，在鏈結中執行 Job 批次。例如，您可以先執行一整批 Job 來發佈多個 Podcast，然後執行一整批 Job 來發送發佈通知：

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
### 新增 Job 至批次

有時，從批次處理的 Job 內部向批次中新增額外的 Job 可能會很有用。當您需要批次處理數千個 Job，且這些 Job 在網頁請求期間分發可能耗時過長時，這種模式會非常有用。因此，您可以改為分發初始的一批「載入器 (Loader)」Job，由它們為批次填充 (Hydrate) 更多的 Job：

```php
$batch = Bus::batch([
    new LoadImportBatch,
    new LoadImportBatch,
    new LoadImportBatch,
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import Contacts')->dispatch();
```

在這個範例中，我們將使用 `LoadImportBatch` Job 來為批次填充額外的 Job。為此，我們可以使用批次執行個體上的 `add` 方法，該執行個體可透過 Job 的 `batch` 方法存取：

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
> 您只能從屬於同一個批次的 Job 內部向該批次新增 Job。

<a name="inspecting-batches"></a>
### 檢查批次

提供給批次完成回呼的 `Illuminate\Bus\Batch` 實例擁有多種屬性與方法，可協助您與指定的 Job 批次進行互動與檢查：

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

所有的 `Illuminate\Bus\Batch` 實例都是可 JSON 序列化的，這意味著您可以直接從應用程式的路由中回傳它們，以取得包含批次資訊（包含其完成進度）的 JSON 酬載。這使得在應用程式的 UI 中顯示批次完成進度的資訊變得非常方便。

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

有時您可能需要取消特定批次的執行。這可以透過呼叫 `Illuminate\Bus\Batch` 實例上的 `cancel` 方法來完成：

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

正如您在先前的範例中所注意到的，批次處理的 Job 通常應在繼續執行之前判斷其對應的批次是否已取消。然而，為了方便起見，您可以改為將 `SkipIfBatchCancelled` [中介層](#job-middleware)指派給 Job。顧名思義，此中介層將指示 Laravel 在其對應的批次被取消時不要處理該 Job：

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
### 批次失敗

當批次中的 Job 失敗時，`catch` 回呼（若有指派）將會被呼叫。此回呼僅會針對批次中第一個失敗的 Job 呼叫。


<a name="allowing-failures"></a>
#### 允許失敗

當批次中的 Job 失敗時，Laravel 會自動將該批次標記為「已取消」。如果您願意，可以停用此行為，使 Job 失敗時不會自動將批次標記為已取消。這可以透過在分發批次時呼叫 `allowFailures` 方法來實現：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->allowFailures()->dispatch();
```

您也可以選擇提供一個 Closure 給 `allowFailures` 方法，該 Closure 將在每次 Job 失敗時執行：

```php
$batch = Bus::batch([
    // ...
])->allowFailures(function (Batch $batch, $exception) {
    // Handle individual job failures...
})->dispatch();
```


<a name="retrying-failed-batch-jobs"></a>
#### 重試失敗的批次 Job

為了方便起見，Laravel 提供了一個 `queue:retry-batch` Artisan 命令，讓您可以輕鬆地重試特定批次中所有失敗的 Job。此命令接受應重試其失敗 Job 的批次 UUID：

```shell
php artisan queue:retry-batch 32dbc76c-4f82-4749-b610-a639fe0099b5
```


<a name="pruning-batches"></a>
### 清理批次

如果不進行清理，`job_batches` 資料表會非常快地累積紀錄。為了減輕這種情況，您應該[排程](/docs/{{version}}/scheduling) `queue:prune-batches` Artisan 命令每天執行：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches')->daily();
```

預設情況下，所有已完成且超過 24 小時的批次都將被清理。您可以在呼叫命令時使用 `hours` 選項來決定批次資料的保留時間。例如，以下命令將刪除所有在 48 小時前完成的批次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48')->daily();
```

有時，您的 `jobs_batches` 資料表可能會累積一些從未成功完成的批次紀錄，例如某個 Job 失敗且該 Job 從未重試成功的批次。您可以使用 `unfinished` 選項指示 `queue:prune-batches` 命令清理這些未完成的批次紀錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --unfinished=72')->daily();
```

同樣地，您的 `jobs_batches` 資料表也可能累積已取消批次的紀錄。您可以使用 `cancelled` 選項指示 `queue:prune-batches` 命令清理這些已取消的批次紀錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --cancelled=72')->daily();
```

<a name="storing-batches-in-dynamodb"></a>
### 將批次儲存在 DynamoDB

Laravel 也支援將批次的中繼資訊 (Meta Information) 儲存在 [DynamoDB](https://aws.amazon.com/dynamodb) 中，而非關聯式資料庫。然而，您需要手動建立一個 DynamoDB 資料表來儲存所有的批次紀錄。

通常這個資料表的名稱應為 `job_batches`，但您應該根據應用程式 `queue` 設定檔中的 `queue.batching.table` 設定值來命名該資料表。


<a name="dynamodb-batch-table-configuration"></a>
#### DynamoDB 批次資料表設定

`job_batches` 資料表應具有一個名為 `application` 的字串類型主要分割鍵 (Primary Partition Key)，以及一個名為 `id` 的字串類型主要排序鍵 (Primary Sort Key)。鍵的 `application` 部分將包含您的應用程式名稱，這定義於應用程式 `app` 設定檔中的 `name` 設定值。由於應用程式名稱是 DynamoDB 資料表鍵的一部分，因此您可以使用同一個資料表來儲存多個 Laravel 應用程式的 Job 批次。

此外，如果您想利用 [自動批次清理](#pruning-batches-in-dynamodb) 的功能，可以為您的資料表定義 `ttl` 屬性。


<a name="dynamodb-configuration"></a>
#### DynamoDB 設定

接著，安裝 AWS SDK，以便您的 Laravel 應用程式可以與 Amazon DynamoDB 進行通訊：

```shell
composer require aws/aws-sdk-php
```

然後，將 `queue.batching.driver` 設定選項的值設為 `dynamodb`。此外，您應該在 `batching` 設定陣列中定義 `key`、`secret` 與 `region` 設定選項。這些選項將用於 AWS 的身分驗證。使用 `dynamodb` 驅動程式時，不需要 `queue.batching.database` 設定選項：

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

當利用 [DynamoDB](https://aws.amazon.com/dynamodb) 來儲存 Job 批次資訊時，用於清理儲存在關聯式資料庫中批次的典型清理命令將無法運作。相反地，您可以利用 [DynamoDB 原生的 TTL 功能](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html) 來自動移除舊批次的紀錄。

如果您為 DynamoDB 資料表定義了 `ttl` 屬性，則可以定義設定參數來指示 Laravel 如何清理批次紀錄。`queue.batching.ttl_attribute` 設定值定義了存放 TTL 的屬性名稱，而 `queue.batching.ttl` 設定值定義了批次紀錄從 DynamoDB 資料表中移除前所需的秒數（相對於紀錄最後一次更新的時間）：

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
## 將 Closure 排入佇列

除了分發一個 Job 類別到佇列外，您也可以分發一個 Closure。這對於需要在目前請求週期之外執行的快速、簡單任務非常有用。將 Closure 分發到佇列時，Closure 的程式碼內容會經過加密簽署，因此在傳輸過程中無法被修改：

```php
use App\Models\Podcast;

$podcast = Podcast::find(1);

dispatch(function () use ($podcast) {
    $podcast->publish();
});
```

若要為排入佇列的 Closure 指定一個名稱，以便在佇列報告儀表板中使用，或是顯示在 `queue:work` 命令中，您可以使用 `name` 方法：

```php
dispatch(function () {
    // ...
})->name('Publish Podcast');
```

使用 `catch` 方法，您可以提供一個在排入佇列的 Closure 耗盡所有佇列的[設定重試次數](#max-job-attempts-and-timeout)後仍無法成功完成時，所應執行的 Closure：

```php
use Throwable;

dispatch(function () use ($podcast) {
    $podcast->publish();
})->catch(function (Throwable $e) {
    // This job has failed...
});
```

> [!WARNING]
> 由於 `catch` 回呼會被序列化，並在稍後的時間點由 Laravel 佇列執行，因此您不應在 `catch` 回呼中使用 `$this` 變數。

<a name="running-the-queue-worker"></a>
## 執行佇列工作者


<a name="the-queue-work-command"></a>
### `queue:work` 命令

Laravel 包含一個 Artisan 命令，它將啟動一個佇列工作者並處理推送到佇列中的新 Job。您可以使用 `queue:work` Artisan 命令執行工作者。請注意，一旦 `queue:work` 命令啟動，它將持續運行，直到被手動停止或您關閉終端機：

```shell
php artisan queue:work
```

> [!NOTE]
> 若要讓 `queue:work` 程序在背景永久運行，您應該使用程序監視器，例如 [Supervisor](#supervisor-configuration)，以確保佇列工作者不會停止運行。

如果您希望在命令輸出中包含已處理的 Job ID、連線名稱和佇列名稱，您可以在調用 `queue:work` 命令時加入 `-v` 旗標：

```shell
php artisan queue:work -v
```

請記住，佇列工作者是長期運行的程序，並將引導後的應用程式狀態儲存在記憶體中。因此，它們在啟動後不會察覺到程式碼庫中的變更。所以，在您的部署過程中，請務必[重啟您的佇列工作者](#queue-workers-and-deployment)。此外，請記住，應用程式建立或修改的任何全域靜態狀態都不會在 Job 之間自動重設。

或者，您可以執行 `queue:listen` 命令。使用 `queue:listen` 命令時，當您想要重新載入更新後的程式碼或重設應用程式狀態時，不必手動重啟工作者；然而，此命令的效率明顯低於 `queue:work` 命令：

```shell
php artisan queue:listen
```


<a name="running-multiple-queue-workers"></a>
#### 執行多個佇列工作者

要為佇列分配多個工作者並同時處理 Job，您只需啟動多個 `queue:work` 程序即可。這可以在本機透過終端機中的多個分頁完成，或在正式環境中使用程序管理員的設定。[當使用 Supervisor 時](#supervisor-configuration)，您可以使用 `numprocs` 設定值。


<a name="specifying-the-connection-queue"></a>
#### 指定連線與佇列

您也可以指定工作者應該利用哪個佇列連線。傳遞給 `work` 命令的連線名稱應對應於 `config/queue.php` 設定檔中定義的其中一個連線：

```shell
php artisan queue:work redis
```

預設情況下，`queue:work` 命令僅處理給定連線上預設佇列的 Job。然而，您可以透過僅處理特定連線的特定佇列來進一步自訂您的佇列工作者。例如，如果您的所有電子郵件都在 `redis` 佇列連線的 `emails` 佇列中處理，您可以發布以下命令來啟動僅處理該佇列的工作者：

```shell
php artisan queue:work redis --queue=emails
```


<a name="processing-a-specified-number-of-jobs"></a>
#### 處理指定數量的 Job

`--once` 選項可用於指示工作者僅從佇列中處理單個 Job：

```shell
php artisan queue:work --once
```

`--max-jobs` 選項可用於指示工作者處理給定數量的 Job 然後退出。此選項在與 [Supervisor](#supervisor-configuration) 結合使用時可能很有用，以便在處理給定數量的 Job 後自動重啟工作者，從而釋放它們可能累積的任何記憶體：

```shell
php artisan queue:work --max-jobs=1000
```


<a name="processing-all-queued-jobs-then-exiting"></a>
#### 處理所有佇列中的 Job 然後退出

`--stop-when-empty` 選項可用於指示工作者處理所有 Job 然後正常退出。如果您希望在佇列清空後關閉容器，則在 Docker 容器中處理 Laravel 佇列時，此選項會很有用：

```shell
php artisan queue:work --stop-when-empty
```


<a name="processing-jobs-for-a-given-number-of-seconds"></a>
#### 處理 Job 指定秒數

`--max-time` 選項可用於指示工作者處理 Job 給定的秒數然後退出。此選項在與 [Supervisor](#supervisor-configuration) 結合使用時可能很有用，以便在處理 Job 給定時間後自動重啟工作者，從而釋放它們可能累積的任何記憶體：

```shell
# Process jobs for one hour and then exit...
php artisan queue:work --max-time=3600
```


<a name="worker-sleep-duration"></a>
#### 工作者睡眠時間

當佇列中有 Job 時，工作者將持續處理 Job，Job 之間沒有延遲。然而，`sleep` 選項決定了如果沒有可用的 Job，工作者將「睡眠」多少秒。當然，在睡眠期間，工作者將不會處理任何新 Job：

```shell
php artisan queue:work --sleep=3
```


<a name="maintenance-mode-queues"></a>
#### 維護模式與佇列

當您的應用程式處於[維護模式](/docs/{{version}}/configuration#maintenance-mode)時，不會處理任何排隊的 Job。一旦應用程式脫離維護模式，Job 將照常繼續處理。

要強制您的佇列工作者處理 Job，即使啟用了維護模式，您也可以使用 `--force` 選項：

```shell
php artisan queue:work --force
```


<a name="resource-considerations"></a>
#### 資源考量

守護進程 (Daemon) 佇列工作者在處理每個 Job 之前不會「重啟」框架。因此，您應該在每個 Job 完成後釋放任何繁重的資源。例如，如果您使用 [GD 函式庫](https://www.php.net/manual/en/book.image.php)進行影像處理，則在處理完影像後，您應該使用 `imagedestroy` 釋放記憶體。


<a name="queue-priorities"></a>
### 佇列優先順序

有時您可能希望優先處理佇列。例如，在您的 `config/queue.php` 設定檔中，您可以將 `redis` 連線的預設 `queue` 設定為 `low`。然而，偶爾您可能希望將一個 Job 推送到 `high` 優先順序佇列，如下所示：

```php
dispatch((new Job)->onQueue('high'));
```

要啟動一個工作者，以驗證在繼續處理 `low` 佇列上的任何 Job 之前處理所有 `high` 佇列 Job，請將逗號分隔的佇列名稱列表傳遞給 `work` 命令：

```shell
php artisan queue:work --queue=high,low
```


<a name="queue-workers-and-deployment"></a>
### 佇列工作者與部署

由於佇列工作者是長期運行的程序，如果不重啟，它們將不會察覺到程式碼的變更。因此，部署使用佇列工作者的應用程式最簡單方法是在部署過程中重啟工作者。您可以透過發布 `queue:restart` 命令來正常重啟所有工作者：

```shell
php artisan queue:restart
```

此命令將指示所有佇列工作者在完成處理當前 Job 後正常退出，以免遺失任何現有的 Job。由於佇列工作者在執行 `queue:restart` 命令時會退出，因此您應該運行一個程序管理員，例如 [Supervisor](#supervisor-configuration)，以自動重啟佇列工作者。

> [!NOTE]
> 佇列使用[快取](/docs/{{version}}/cache)來儲存重啟訊號，因此在使用此功能之前，您應該驗證是否為您的應用程式正確設定了快取驅動程式。

<a name="job-expirations-and-timeouts"></a>
### Job 到期與逾時


<a name="job-expiration"></a>
#### Job 到期

在您的 `config/queue.php` 設定檔中，每個佇列連線都定義了一個 `retry_after` 選項。此選項指定了佇列連線在重試正在處理中的 Job 之前應等待的秒數。例如，如果 `retry_after` 的值設定為 `90`，則若該 Job 已處理 90 秒且未被釋放或刪除，該 Job 將會被放回佇列中。通常，您應該將 `retry_after` 的值設定為您的 Job 完成處理所合理花費的最大秒數。

> [!WARNING]
> 唯一不包含 `retry_after` 值的佇列連線是 Amazon SQS。SQS 將根據在 AWS 控制台中管理的 [Default Visibility Timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html) 來重試 Job。


<a name="worker-timeouts"></a>
#### 工作者逾時

`queue:work` Artisan 命令提供了一個 `--timeout` 選項。預設情況下，`--timeout` 的值為 60 秒。如果 Job 的處理時間超過逾時值指定的秒數，處理該 Job 的工作者將會因錯誤而結束。通常，工作者會由您伺服器上設定的 [程序管理員](#supervisor-configuration) 自動重啟：

```shell
php artisan queue:work --timeout=60
```

`retry_after` 設定選項與 `--timeout` CLI 選項不同，但兩者相輔相成，共同確保 Job 不會遺失，且 Job 只會被成功處理一次。

> [!WARNING]
> `--timeout` 的值應始終比您的 `retry_after` 設定值短至少幾秒鐘。這將確保處理卡住 Job 的工作者在 Job 重試之前已被終止。如果您的 `--timeout` 選項長於 `retry_after` 設定值，您的 Job 可能會被處理兩次。


<a name="pausing-and-resuming-queue-workers"></a>
### 暫停與恢復佇列工作者

有時您可能需要暫時防止佇列工作者處理新的 Job，而不需要完全停止工作者。例如，您可能想在系統維護期間暫停 Job 處理。Laravel 提供了 `queue:pause` 與 `queue:continue` Artisan 命令來暫停與恢復佇列工作者。

要暫停特定的佇列，請提供佇列連線名稱與佇列名稱：

```shell
php artisan queue:pause database:default
```

在此範例中，`database` 是佇列連線名稱，而 `default` 是佇列名稱。一旦佇列被暫停，任何從該佇列處理 Job 的工作者將會繼續完成其當前的 Job，但在該佇列恢復之前不會取得任何新的 Job。

要恢復處理已暫停佇列上的 Job，請使用 `queue:continue` 命令：

```shell
php artisan queue:continue database:default
```

恢復佇列後，工作者將立即開始從該佇列處理新的 Job。請注意，暫停佇列並不會停止工作者程序本身——它僅防止工作者從指定的佇列處理新的 Job。


<a name="worker-restart-and-pause-signals"></a>
#### 工作者重啟與暫停訊號

預設情況下，佇列工作者會在每次 Job 疊代時輪詢快取驅動程式以獲取重啟與暫停訊號。雖然這種輪詢對於回應 `queue:restart` 與 `queue:pause` 命令至關重要，但它確實會帶來少量的效能開銷。

如果您需要優化效能且不需要這些中斷功能，您可以透過呼叫 `Queue` Facade 上的 `withoutInterruptionPolling` 方法來全域停用此輪詢。這通常應該在您 `AppServiceProvider` 的 `boot` 方法中完成：

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

或者，您可以透過設定 `Illuminate\Queue\Worker` 類別上的靜態屬性 `$restartable` 或 `$pausable` 來個別停用重啟或暫停輪詢：

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
> 當停用中斷輪詢時，工作者將不會對 `queue:restart` 或 `queue:pause` 命令做出回應（取決於停用了哪些功能）。

<a name="supervisor-configuration"></a>
## Supervisor 設定

在正式環境中，您需要一種方法來保持 `queue:work` 程序持續執行。`queue:work` 程序可能會因為各種原因停止執行，例如超過工作者 (Worker) 逾時時間或執行了 `queue:restart` 命令。

因此，您需要設定一個程序監控器 (Process Monitor)，它可以偵測您的 `queue:work` 程序何時結束並自動重新啟動它們。此外，程序監控器還可以讓您指定想要同時執行多少個 `queue:work` 程序。Supervisor 是一個常用於 Linux 環境的程序監控器，我們將在以下文件中討論如何設定它。


<a name="installing-supervisor"></a>
#### 安裝 Supervisor

Supervisor 是 Linux 作業系統的一個程序監控器，如果您的 `queue:work` 程序失敗，它會自動重啟。要在 Ubuntu 上安裝 Supervisor，您可以使用以下命令：

```shell
sudo apt-get install supervisor
```

> [!NOTE]
> 如果您覺得自行設定和管理 Supervisor 太過繁瑣，可以考慮使用 [Laravel Cloud](https://cloud.laravel.com)，它提供了一個全代管平台來執行 Laravel 佇列工作者。


<a name="configuring-supervisor"></a>
#### 設定 Supervisor

Supervisor 的設定檔通常儲存在 `/etc/supervisor/conf.d` 目錄中。在此目錄中，您可以建立任意數量的設定檔，指示 Supervisor 應如何監控您的程序。例如，讓我們建立一個 `laravel-worker.conf` 檔案，用來啟動並監控 `queue:work` 程序：

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

在此範例中，`numprocs` 指令將指示 Supervisor 執行八個 `queue:work` 程序並監控所有程序，如果它們失敗則會自動重啟。您應該修改設定中的 `command` 指令，以反映您所需的佇列連線和工作者選項。

> [!WARNING]
> 您應確保 `stopwaitsecs` 的值大於您執行時間最長的 Job 所耗費的秒數。否則，Supervisor 可能會在 Job 處理完成前就將其終止。


<a name="starting-supervisor"></a>
#### 啟動 Supervisor

建立設定檔後，您可以使用以下命令更新 Supervisor 設定並啟動程序：

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start "laravel-worker:*"
```

有關 Supervisor 的更多資訊，請參閱 [Supervisor 說明文件](http://supervisord.org/index.html)。

<a name="dealing-with-failed-jobs"></a>
## 處理失敗的 Job

有時候您排入佇列的 Job 會失敗。別擔心，事情並不總是按計劃進行！Laravel 提供了一種便利的方法來 [指定 Job 應該嘗試的最大次數](#max-job-attempts-and-timeout)。在一個非同步 Job 超過此嘗試次數後，它將被插入到 `failed_jobs` 資料庫表中。失敗的 [同步分發 Job](/docs/{{version}}/queues#synchronous-dispatching) 不會儲存在此表中，其例外狀況會立即由應用程式處理。

在新的 Laravel 應用程式中，通常已經存在建立 `failed_jobs` 表的遷移。但是，如果您的應用程式不包含此表的遷移，您可以使用 `make:queue-failed-table` 命令來建立遷移：

```shell
php artisan make:queue-failed-table

php artisan migrate
```

執行 [佇列工作者](#running-the-queue-worker) 處理程序時，您可以使用 `queue:work` 命令上的 `--tries` 切換參數來指定 Job 應嘗試的最大次數。如果您沒有為 `--tries` 選項指定值，則 Job 僅會嘗試一次，或根據 Job 類別的 `$tries` 屬性所指定的次數進行嘗試：

```shell
php artisan queue:work redis --tries=3
```

使用 `--backoff` 選項，您可以指定 Laravel 在重試遇到例外的 Job 之前應等待多少秒。預設情況下，Job 會立即釋放回佇列中，以便再次嘗試：

```shell
php artisan queue:work redis --tries=3 --backoff=3
```

如果您想在個別 Job 的基礎上設定 Laravel 在重試遇到例外的 Job 之前應等待多少秒，您可以透過在 Job 類別上定義 `backoff` 屬性來達成：

```php
/**
 * The number of seconds to wait before retrying the job.
 *
 * @var int
 */
public $backoff = 3;
```

如果您需要更複雜的邏輯來決定 Job 的退避時間，您可以在 Job 類別中定義一個 `backoff` 方法：

```php
/**
 * Calculate the number of seconds to wait before retrying the job.
 */
public function backoff(): int
{
    return 3;
}
```

您可以透過從 `backoff` 方法回傳一個退避值的陣列，來輕鬆設定「指數級」退避。在此範例中，若仍有剩餘的嘗試次數，第一次重試的延遲將為 1 秒，第二次重試為 5 秒，第三次重試為 10 秒，之後的每次重試皆為 10 秒：

```php
/**
 * Calculate the number of seconds to wait before retrying the job.
 *
 * @return array<int, int>
 */
public function backoff(): array
{
    return [1, 5, 10];
}
```


<a name="cleaning-up-after-failed-jobs"></a>
### 失敗 Job 的清理工作

當特定 Job 失敗時，您可能希望向使用者傳送警示，或還原該 Job 已部分完成的任何操作。為此，您可以在 Job 類別中定義一個 `failed` 方法。導致 Job 失敗的 `Throwable` 執行個體將被傳遞給 `failed` 方法：

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
> 在呼叫 `failed` 方法之前會先實例化一個新的 Job 執行個體；因此，在 `handle` 方法中可能發生的任何類別屬性修改都將遺失。

失敗的 Job 不一定是指遇到未處理例外的 Job。當 Job 耗盡了所有允許的嘗試次數時，也被視為失敗。這些嘗試次數可以透過多種方式消耗：

<div class="content-list" markdown="1">

- Job 逾時。
- Job 在執行期間遇到未處理的例外。
- Job 手動或透過中介層釋放回佇列。

</div>

如果最後一次嘗試是因為 Job 執行期間拋出的例外而失敗，該例外將被傳遞給 Job 的 `failed` 方法。但是，如果 Job 是因為達到最大允許嘗試次數而失敗，則 `$exception` 將會是 `Illuminate\Queue\MaxAttemptsExceededException` 的執行個體。同樣地，如果 Job 是因為超過設定的逾時時間而失敗，則 `$exception` 將會是 `Illuminate\Queue\TimeoutExceededException` 的執行個體。


<a name="retrying-failed-jobs"></a>
### 重試失敗的 Job

若要查看已插入 `failed_jobs` 資料庫表中的所有失敗 Job，您可以使用 `queue:failed` Artisan 命令：

```shell
php artisan queue:failed
```

`queue:failed` 命令會列出 Job ID、連線、佇列、失敗時間以及關於該 Job 的其他資訊。Job ID 可用於重試失敗的 Job。例如，要重試 ID 為 `ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece` 的失敗 Job，請執行以下命令：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece
```

如有必要，您可以向命令傳遞多個 ID：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece 91401d2c-0784-4f43-824c-34f94a33c24d
```

您也可以重試特定佇列的所有失敗 Job：

```shell
php artisan queue:retry --queue=name
```

要重試所有的失敗 Job，請執行 `queue:retry` 命令並將 `all` 作為 ID 傳遞：

```shell
php artisan queue:retry all
```

如果您想刪除一個失敗的 Job，您可以使用 `queue:forget` 命令：

```shell
php artisan queue:forget 91401d2c-0784-4f43-824c-34f94a33c24d
```

> [!NOTE]
> 使用 [Horizon](/docs/{{version}}/horizon) 時，您應該使用 `horizon:forget` 命令來刪除失敗的 Job，而不是 `queue:forget` 命令。

要從 `failed_jobs` 表中刪除所有失敗的 Job，您可以使用 `queue:flush` 命令：

```shell
php artisan queue:flush
```

`queue:flush` 命令會從您的佇列中移除所有失敗 Job 的紀錄，無論該失敗 Job 有多舊。您可以使用 `--hours` 選項來僅刪除特定小時數之前失敗的 Job：

```shell
php artisan queue:flush --hours=48
```


<a name="ignoring-missing-models"></a>
### 忽略遺失的模型

將 Eloquent 模型注入到 Job 時，模型在被放入佇列之前會自動序列化，並在 Job 處理時從資料庫中重新取得。但是，如果模型在 Job 等待工作者處理期間已被刪除，您的 Job 可能會失敗並拋出 `ModelNotFoundException`。

為了方便起見，您可以透過將 Job 的 `deleteWhenMissingModels` 屬性設定為 `true`，來選擇自動刪除遺失模型的 Job。當此屬性設定為 `true` 時，Laravel 將悄悄地捨棄該 Job 而不引發例外：

```php
/**
 * Delete the job if its models no longer exist.
 *
 * @var bool
 */
public $deleteWhenMissingModels = true;
```


<a name="pruning-failed-jobs"></a>
### 清理失敗的 Job

您可以透過呼叫 `queue:prune-failed` Artisan 命令來清理應用程式 `failed_jobs` 表中的紀錄：

```shell
php artisan queue:prune-failed
```

預設情況下，所有超過 24 小時的失敗 Job 紀錄都將被清理。如果您為命令提供 `--hours` 選項，則僅會保留在過去 N 小時內插入的失敗 Job 紀錄。例如，以下命令將刪除所有在 48 小時前插入的失敗 Job 紀錄：

```shell
php artisan queue:prune-failed --hours=48
```

<a name="storing-failed-jobs-in-dynamodb"></a>
### 將失敗的 Job 儲存在 DynamoDB

Laravel 也支援將您的失敗 Job 紀錄儲存在 [DynamoDB](https://aws.amazon.com/dynamodb) 中，而非關聯式資料庫資料表。然而，您必須手動建立一個 DynamoDB 資料表來儲存所有失敗的 Job 紀錄。通常，此資料表的名稱應為 `failed_jobs`，但您應根據應用程式 `queue` 設定檔中的 `queue.failed.table` 設定值來命名該資料表。

`failed_jobs` 資料表應具備一個名為 `application` 的字串型別主要分割鍵 (Primary Partition Key)，以及一個名為 `uuid` 的字串型別主要排序鍵 (Primary Sort Key)。鍵值的 `application` 部分將包含您的應用程式名稱，此名稱定義於應用程式 `app` 設定檔中的 `name` 設定值。由於應用程式名稱是 DynamoDB 資料表鍵值的一部分，因此您可以使用同一個資料表來儲存多個 Laravel 應用程式的失敗 Job。

此外，請確保您已安裝 AWS SDK，以便您的 Laravel 應用程式可以與 Amazon DynamoDB 進行通訊：

```shell
composer require aws/aws-sdk-php
```

接著，將 `queue.failed.driver` 設定選項的值設為 `dynamodb`。此外，您應該在失敗 Job 的設定陣列中定義 `key`、`secret` 及 `region` 設定選項。這些選項將用於向 AWS 進行身份驗證。使用 `dynamodb` 驅動程式時，`queue.failed.database` 設定選項是不需要的：

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
### 停用失敗 Job 儲存

您可以透過將 `queue.failed.driver` 設定選項的值設為 `null`，來指示 Laravel 直接捨棄失敗的 Job 而不進行儲存。通常，這可以透過 `QUEUE_FAILED_DRIVER` 環境變數來完成：

```ini
QUEUE_FAILED_DRIVER=null
```

<a name="failed-job-events"></a>
### 失敗 Job 事件

如果您想要註冊一個在 Job 失敗時被呼叫的事件監聽器，可以使用 `Queue` Facade 的 `failing` 方法。例如，我們可以在 Laravel 內建的 `AppServiceProvider` 的 `boot` 方法中，為此事件附加一個 Closure：

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
## 清除佇列中的 Job

> [!NOTE]
> 使用 [Horizon](/docs/{{version}}/horizon) 時，您應該使用 `horizon:clear` 命令從佇列中清除 Job，而不是使用 `queue:clear` 命令。

如果您想刪除預設連線的預設佇列中的所有 Job，可以使用 `queue:clear` Artisan 命令：

```shell
php artisan queue:clear
```

您也可以提供 `connection` 引數和 `queue` 選項，以從特定的連線與佇列中刪除 Job：

```shell
php artisan queue:clear redis --queue=emails
```

> [!WARNING]
> 從佇列清除 Job 的功能僅適用於 SQS、Redis 和 database 佇列驅動程式。此外，SQS 的訊息刪除程序最多需要 60 秒，因此在您清除佇列後 60 秒內發送到 SQS 佇列的 Job 也可能會被刪除。

<a name="monitoring-your-queues"></a>
## 監控您的佇列

如果您的佇列突然湧入大量 Job，它可能會變得過於擁擠，導致 Job 完成的等待時間過長。如果您願意，Laravel 可以在您的佇列 Job 數量超過指定門檻時發出警報。

首先，您應該將 `queue:monitor` 命令排程為 [每分鐘執行一次](/docs/{{version}}/scheduling)。該命令接受您想要監控的佇列名稱以及您指定的 Job 數量門檻：

```shell
php artisan queue:monitor redis:default,redis:deployments --max=100
```

僅排程此命令不足以觸發通知來提醒您佇列負荷過重的狀態。當該命令遇到 Job 數量超過門檻的佇列時，會分發一個 `Illuminate\Queue\Events\QueueBusy` 事件。您可以在應用程式的 `AppServiceProvider` 中監聽此事件，以便向您或您的開發團隊發送通知：

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
                $event->connection,
                $event->queue,
                $event->size
            ));
    });
}
```

<a name="testing"></a>
## 測試

當測試會分發 Job 的程式碼時，您可能希望指示 Laravel 不要實際執行該 Job 本身，因為 Job 的程式碼可以獨立於分發它的程式碼之外，直接且單獨進行測試。當然，若要測試 Job 本身，您可以實例化 Job 實例，並在測試中直接呼叫 `handle` 方法。

您可以使用 `Queue` Facade 的 `fake` 方法來防止佇列中的 Job 實際被推送到佇列中。呼叫 `Queue` Facade 的 `fake` 方法後，您可以斷言 (Assert) 應用程式曾嘗試將 Job 推送到佇列：

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

您可以將 Closure 傳遞給 `assertPushed`、`assertNotPushed`、`assertClosurePushed` 或 `assertClosureNotPushed` 方法，以斷言推送到佇列的 Job 是否通過指定的「真值測試 (Truth Test)」。如果至少有一個 Job 通過指定的真值測試，則斷言將會成功：

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
### 模擬一部分的 Job

如果您只需要模擬特定的 Job，同時允許其他 Job 正常執行，可以將要模擬的 Job 類別名稱傳遞給 `fake` 方法：

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

您可以使用 `except` 方法來模擬除了指定的一組 Job 以外的所有 Job：

```php
Queue::fake()->except([
    ShipOrder::class,
]);
```

<a name="testing-job-chains"></a>
### 測試 Job 鏈結

為了測試 Job 鏈結，您需要利用 `Bus` Facade 的模擬功能。`Bus` Facade 的 `assertChained` 方法可用於斷言 [Job 鏈結](/docs/{{version}}/queues#job-chaining)已被分發。`assertChained` 方法接受一個鏈結 Job 陣列作為其第一個參數：

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

如上例所示，鏈結 Job 陣列可以是 Job 的類別名稱陣列。不過，您也可以提供實際的 Job 實例陣列。這樣做時，Laravel 將確保 Job 實例與應用程式分發的鏈結 Job 具有相同的類別和屬性值：

```php
Bus::assertChained([
    new ShipOrder,
    new RecordShipment,
    new UpdateInventory,
]);
```

您可以使用 `assertDispatchedWithoutChain` 方法來斷言 Job 已在沒有 Job 鏈結的情況下被推送：

```php
Bus::assertDispatchedWithoutChain(ShipOrder::class);
```

<a name="testing-chain-modifications"></a>
#### 測試鏈結修改

如果一個鏈結中的 Job 在[現有鏈結的前後新增 Job](#adding-jobs-to-the-chain)，您可以使用該 Job 的 `assertHasChain` 方法來斷言該 Job 具有預期的剩餘鏈結 Job：

```php
$job = new ProcessPodcast;

$job->handle();

$job->assertHasChain([
    new TranscribePodcast,
    new OptimizePodcast,
    new ReleasePodcast,
]);
```

`assertDoesntHaveChain` 方法可用於斷言該 Job 的剩餘鏈結為空：

```php
$job->assertDoesntHaveChain();
```

<a name="testing-chained-batches"></a>
#### 測試鏈結批次

如果您的 Job 鏈結[包含一個 Job 批次](#chains-and-batches)，您可以透過在鏈結斷言中插入 `Bus::chainedBatch` 定義，來斷言該鏈結批次符合您的預期：

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
### 測試 Job 批次

`Bus` Facade 的 `assertBatched` 方法可用於斷言某個 [Job 批次](/docs/{{version}}/queues#job-batching) 已被分發。傳遞給 `assertBatched` 方法的 Closure 會接收一個 `Illuminate\Bus\PendingBatch` 實例，可用於檢查批次中的 Job：

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

可在擱置批次上使用 `hasJobs` 方法來驗證批次是否包含預期的 Job。此方法接受 Job 實例、類別名稱或 Closure 所組成的陣列：

```php
Bus::assertBatched(function (PendingBatch $batch) {
    return $batch->hasJobs([
        new ProcessCsvRow(row: 1),
        new ProcessCsvRow(row: 2),
        new ProcessCsvRow(row: 3),
    ]);
});
```

使用 Closure 時，該 Closure 將接收 Job 實例。預期的 Job 型別將從 Closure 的型別提示 (Type Hint) 中推斷出來：

```php
Bus::assertBatched(function (PendingBatch $batch) {
    return $batch->hasJobs([
        fn (ProcessCsvRow $job) => $job->row === 1,
        fn (ProcessCsvRow $job) => $job->row === 2,
        fn (ProcessCsvRow $job) => $job->row === 3,
    ]);
});
```

您可以使用 `assertBatchCount` 方法來斷言已分發了指定數量的批次：

```php
Bus::assertBatchCount(3);
```

您可以使用 `assertNothingBatched` 來斷言沒有任何批次被分發：

```php
Bus::assertNothingBatched();
```

<a name="testing-job-batch-interaction"></a>
#### 測試 Job / 批次互動

此外，您偶爾可能需要測試個別 Job 與其所屬批次的互動。例如，您可能需要測試某個 Job 是否取消了其批次的進一步處理。為此，您需要透過 `withFakeBatch` 方法為該 Job 指派一個模擬 (Fake) 批次。`withFakeBatch` 方法會回傳一個包含 Job 實例與模擬批次的元組 (Tuple)：

```php
[$job, $batch] = (new ShipOrder)->withFakeBatch();

$job->handle();

$this->assertTrue($batch->cancelled());
$this->assertEmpty($batch->added);
```

<a name="testing-job-queue-interactions"></a>
### 測試 Job / 佇列互動

有時，您可能需要測試排入佇列的 Job 是否會 [將自己釋放回佇列](#manually-releasing-a-job)，或是測試該 Job 是否刪除了自己。您可以透過實例化該 Job 並呼叫 `withFakeQueueInteractions` 方法來測試這些佇列互動。

一旦模擬了 Job 的佇列互動，您就可以呼叫 Job 的 `handle` 方法。在呼叫 Job 之後，可以使用各種斷言方法來驗證 Job 的佇列互動：

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
## Job 事件

使用 `Queue` [Facade](/docs/{{version}}/facades) 上的 `before` 和 `after` 方法，您可以指定在處理佇列 Job 之前或之後要執行的回呼 (Callback)。這些回呼是進行額外記錄或增加儀表板統計數據的好機會。通常，您應該從 [服務提供者 (Service Provider)](/docs/{{version}}/providers) 的 `boot` 方法中呼叫這些方法。例如，我們可以使用 Laravel 內建的 `AppServiceProvider`：

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

使用 `Queue` [Facade](/docs/{{version}}/facades) 上的 `looping` 方法，您可以指定在工作者 (Worker) 嘗試從佇列中取得 Job 之前執行的回呼。例如，您可以註冊一個 Closure 來回滾 (Rollback) 任何由先前失敗 Job 所留下的未關閉交易：

```php
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Queue;

Queue::looping(function () {
    while (DB::transactionLevel() > 0) {
        DB::rollBack();
    }
});
```