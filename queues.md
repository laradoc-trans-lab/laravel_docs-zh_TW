# 佇列

- [簡介](#introduction)
    - [連線與佇列的差異](#connections-vs-queues)
    - [驅動器注意事項與先決條件](#driver-prerequisites)
- [建立 Jobs](#creating-jobs)
    - [產生 Job Class](#generating-job-classes)
    - [Class 結構](#class-structure)
    - [Unique Jobs](#unique-jobs)
    - [加密的 Jobs](#encrypted-jobs)
- [Job 中介層](#job-middleware)
    - [速率限制](#rate-limiting)
    - [避免 Job 重疊](#preventing-job-overlaps)
    - [限制例外](#throttling-exceptions)
    - [跳過 Jobs](#skipping-jobs)
- [分派 Jobs](#dispatching-jobs)
    - [延遲分派](#delayed-dispatching)
    - [同步分派](#synchronous-dispatching)
    - [Jobs 與資料庫交易](#jobs-and-database-transactions)
    - [Job 鏈接](#job-chaining)
    - [客製化佇列與連線](#customizing-the-queue-and-connection)
    - [指定 Job 最大嘗試次數 / 逾時值](#max-job-attempts-and-timeout)
    - [SQS FIFO 與公平佇列](#sqs-fifo-and-fair-queues)
    - [錯誤處理](#error-handling)
- [Job 批次處理](#job-batching)
    - [定義可批次處理的 Jobs](#defining-batchable-jobs)
    - [分派批次](#dispatching-batches)
    - [鏈接與批次](#chains-and-batches)
    - [將 Jobs 加入批次](#adding-jobs-to-batches)
    - [檢查批次](#inspecting-batches)
    - [取消批次](#cancelling-batches)
    - [批次失敗](#batch-failures)
    - [清理批次](#pruning-batches)
    - [將批次儲存至 DynamoDB](#storing-batches-in-dynamodb)
- [佇列化閉包](#queueing-closures)
- [執行佇列 Worker](#running-the-queue-worker)
    - [`queue:work` 命令](#the-queue-work-command)
    - [佇列優先級](#queue-priorities)
    - [佇列 Worker 與部署](#queue-workers-and-deployment)
    - [Job 過期與逾時](#job-expirations-and-timeouts)
- [Supervisor 配置](#supervisor-configuration)
- [處理失敗的 Jobs](#dealing-with-failed-jobs)
    - [清理失敗 Jobs 後](#cleaning-up-after-failed-jobs)
    - [重試失敗的 Jobs](#retrying-failed-jobs)
    - [忽略遺失的 Model](#ignoring-missing-models)
    - [清理失敗 Jobs](#pruning-failed-jobs)
    - [將失敗 Jobs 儲存至 DynamoDB](#storing-failed-jobs-in-dynamodb)
    - [禁用失敗 Job 儲存](#disabling-failed-job-storage)
    - [失敗 Job 事件](#failed-job-events)
- [從佇列中清除 Jobs](#clearing-jobs-from-queues)
- [監控您的佇列](#monitoring-your-queues)
- [測試](#testing)
    - [偽造部分 Jobs](#faking-a-subset-of-jobs)
    - [測試 Job 鏈接](#testing-job-chains)
    - [測試 Job 批次](#testing-job-batches)
    - [測試 Job / 佇列互動](#testing-job-queue-interactions)
- [Job 事件](#job-events)

<a name="introduction"></a>
## 簡介

在建構您的網路應用程式時，有些任務，例如解析並儲存上傳的 CSV 檔案，可能會耗費太長時間，無法在一般的網路請求期間執行。幸運的是，Laravel 允許您輕鬆建立可在背景處理的佇列 Job。透過將耗時任務移至佇列，您的應用程式可以極快的速度回應網路請求，並為您的客戶提供更好的使用者體驗。

Laravel 佇列提供了統一的佇列 API，支援各種不同的佇列後端，例如 [Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io)，甚至是關聯式資料庫。

Laravel 的佇列設定選項儲存在您應用程式的 `config/queue.php` 設定檔中。在此檔案中，您會找到框架中包含的每個佇列驅動器的連線設定，包括資料庫、[Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 和 [Beanstalkd](https://beanstalkd.github.io/) 驅動器，以及一個會立即執行 Job 的同步驅動器 (供開發或測試期間使用)。也包含一個 `null` 佇列驅動器，它會丟棄佇列中的 Job。

> [!NOTE]
> Laravel Horizon 是一個為您的 Redis 驅動佇列設計的精美儀表板與設定系統。請查看完整的 [Horizon 文件](/docs/{{version}}/horizon) 以獲取更多資訊。

<a name="connections-vs-queues"></a>
### 連線與佇列的差異

在開始使用 Laravel 佇列之前，了解「連線 (connections)」與「佇列 (queues)」之間的區別非常重要。在您的 `config/queue.php` 設定檔中，有一個 `connections` 設定陣列。此選項定義了與 Amazon SQS、Beanstalk 或 Redis 等後端佇列服務的連線。然而，任何一個佇列連線都可以有多個「佇列」，這些「佇列」可以被視為不同堆疊或堆積的佇列 Job。

請注意，`queue` 設定檔中的每個連線設定範例都包含一個 `queue` 屬性。這是 Job 被傳送到指定連線時將被分派到的預設佇列。換句話說，如果您分派 Job 時沒有明確定義應該分派到哪個佇列，該 Job 將會被放置到連線設定中 `queue` 屬性所定義的佇列上：

```php
use App\Jobs\ProcessPodcast;

// This job is sent to the default connection's default queue...
ProcessPodcast::dispatch();

// This job is sent to the default connection's "emails" queue...
ProcessPodcast::dispatch()->onQueue('emails');
```

有些應用程式可能不需要將 Job 推送到多個佇列，而是偏好使用一個簡單的佇列。然而，將 Job 推送到多個佇列對於希望優先處理或區分 Job 處理方式的應用程式來說特別有用，因為 Laravel 佇列 Worker 允許您指定應根據優先級處理哪些佇列。例如，如果您將 Job 推送到 `high` 佇列，您可以運行一個 Worker 給予它們更高的處理優先級：

```shell
php artisan queue:work --queue=high,default
```

<a name="driver-prerequisites"></a>
### 驅動器注意事項與先決條件

<a name="database"></a>
#### Database

為了使用 `database` 佇列驅動器，您需要一個資料庫表來存放 Job。通常，這會包含在 Laravel 預設的 `0001_01_01_000002_create_jobs_table.php` [資料庫遷移](/docs/{{version}}/migrations)中；但是，如果您的應用程式不包含此遷移，您可以使用 `make:queue-table` Artisan 命令來建立它：

```shell
php artisan make:queue-table

php artisan migrate
```

<a name="redis"></a>
#### Redis

為了使用 `redis` 佇列驅動器，您應該在 `config/database.php` 設定檔中設定 Redis 資料庫連線。

> [!WARNING]
> `serializer` 和 `compression` Redis 選項不支援 `redis` 佇列驅動器。

<a name="redis-cluster"></a>
##### Redis Cluster

如果您的 Redis 佇列連線使用 [Redis Cluster](https://redis.io/docs/latest/operate/rs/databases/durability-ha/clustering)，您的佇列名稱必須包含 [key hash tag](https://redis.io/docs/latest/develop/using-commands/keyspace/#hashtags)。這是為了確保指定佇列的所有 Redis 鍵都放置在相同的 hash slot 中：

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
##### Blocking

使用 Redis 佇列時，您可以使用 `block_for` 設定選項來指定驅動器在 Worker 循環並重新輪詢 Redis 資料庫之前，應該等待 Job 可用多長時間。

根據您的佇列負載調整此值，會比持續輪詢 Redis 資料庫以獲取新 Job 更有效率。例如，您可以將值設定為 `5`，表示驅動器在等待 Job 可用時應阻擋五秒：

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
> 將 `block_for` 設定為 `0` 會導致佇列 Worker 無限期阻擋，直到 Job 可用為止。這也會阻止 `SIGTERM` 等訊號被處理，直到下一個 Job 被處理為止。

<a name="other-driver-prerequisites"></a>
#### 其他驅動器先決條件

以下列出的佇列驅動器需要以下依賴項。這些依賴項可以透過 Composer 套件管理器安裝：

<div class="content-list" markdown="1">

- Amazon SQS: `aws/aws-sdk-php ~3.0`
- Beanstalkd: `pda/pheanstalk ~5.0`
- Redis: `predis/predis ~2.0` or phpredis PHP extension
- [MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/queues/): `mongodb/laravel-mongodb`

</div>

<a name="creating-jobs"></a>
## 建立 Jobs


<a name="generating-job-classes"></a>
### 產生 Job Class

預設情況下，應用程式中所有可佇列的 Jobs 都儲存在 `app/Jobs` 目錄中。如果 `app/Jobs` 目錄不存在，執行 `make:job` Artisan 命令時將會自動建立：

```shell
php artisan make:job ProcessPodcast
```

產生的 class 會實作 `Illuminate\Contracts\Queue\ShouldQueue` 介面，這表示 Laravel 應該將該 Job 推送至佇列中以非同步執行。

> [!NOTE]
> Job 樣板可以使用 [stub publishing](/docs/{{version}}/artisan#stub-customization) 進行客製化。


<a name="class-structure"></a>
### Class 結構

Job class 都非常簡單，通常只包含一個 `handle` 方法，該方法在 Job 被佇列處理時呼叫。首先，讓我們看看一個範例 Job class。在這個範例中，我們假設我們管理一個 Podcast 發佈服務，需要在發佈之前處理上傳的 Podcast 檔案：

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

在這個範例中，請注意，我們可以將 [Eloquent Model](/docs/{{version}}/eloquent) 直接傳遞給佇列 Job 的建構式。由於 Job 使用了 `Queueable` trait，Eloquent Model 及其載入的關聯會在 Job 處理時優雅地序列化與反序列化。

如果佇列 Job 在其建構式中接受一個 Eloquent Model，則只有 Model 的識別碼會被序列化到佇列中。當 Job 實際處理時，佇列系統會自動從資料庫中重新擷取完整的 Model 實例及其載入的關聯。這種 Model 序列化方法允許將更小的 Job 負載傳送到佇列驅動器。


<a name="handle-method-dependency-injection"></a>
#### `handle` 方法的依賴注入

`handle` 方法在 Job 被佇列處理時呼叫。請注意，我們可以在 Job 的 `handle` 方法上進行類型提示依賴。Laravel [服務容器](/docs/{{version}}/container) 會自動注入這些依賴。

如果您想完全控制容器如何將依賴注入到 `handle` 方法中，您可以使用容器的 `bindMethod` 方法。`bindMethod` 方法接受一個回呼，該回呼會接收 Job 和容器。在回呼中，您可以根據需要呼叫 `handle` 方法。通常，您應該從 `App\Providers\AppServiceProvider` [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫此方法：

```php
use App\Jobs\ProcessPodcast;
use App\Services\AudioProcessor;
use Illuminate\Contracts\Foundation\Application;

$this->app->bindMethod([ProcessPodcast::class, 'handle'], function (ProcessPodcast $job, Application $app) {
    return $job->handle($app->make(AudioProcessor::class));
});
```

> [!WARNING]
> 二進位資料（例如原始圖像內容）在傳遞給佇列 Job 之前，應透過 `base64_encode` 函數進行處理。否則，Job 在被放入佇列時可能無法正確地序列化為 JSON。


<a name="handling-relationships"></a>
#### 佇列關聯

由於所有已載入的 Eloquent Model 關聯在 Job 佇列化時也會被序列化，因此序列化後的 Job 字串有時會變得相當龐大。此外，當 Job 被反序列化並從資料庫中重新擷取 Model 關聯時，它們將會被完整地擷取。在 Job 佇列化處理過程中，模型序列化之前套用的任何先前的關聯約束將在 Job 反序列化時不會被套用。因此，如果您希望處理特定關聯的子集，您應該在佇列 Job 中重新約束該關聯。

或者，為了防止關聯被序列化，您可以在設定屬性值時，呼叫 Model 上的 `withoutRelations` 方法。此方法將會回傳一個沒有載入關聯的 Model 實例：

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

如果您使用 [PHP 建構式屬性提升](https://www.php.net/manual/en/language.oop5.decon.php#language.oop5.decon.constructor.promotion)，並且希望指示 Eloquent Model 不應序列化其關聯，您可以使用 `WithoutRelations` 屬性：

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

為方便起見，如果您希望在不帶關聯的情況下序列化所有 Model，您可以將 `WithoutRelations` 屬性套用至整個 class，而不是將其套用至每個 Model：

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

如果 Job 接收的是 Eloquent Model 集合或陣列，而不是單一 Model，那麼該集合中的 Model 在 Job 反序列化並執行時，其關聯將不會被恢復。這是為了防止處理大量 Model 的 Jobs 造成過度資源使用。

<a name="unique-jobs"></a>
### Unique Jobs

> [!WARNING]
> Unique Jobs 需要支援 [鎖定](/docs/{{version}}/cache#atomic-locks) 的快取驅動器。目前，`memcached`、`redis`、`dynamodb`、`database`、`file` 與 `array` 等快取驅動器支援原子鎖定。

> [!WARNING]
> Unique Job 的限制不適用於批次中的 Jobs。

有時，您可能希望確保在任何時間點上，佇列中只有一個特定 Job 的實例。您可以在 Job 類別上實作 `ShouldBeUnique` 介面來達成此目的。此介面不要求您在類別上定義任何額外的方法：

```php
<?php

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Queue\ShouldBeUnique;

class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    // ...
}
```

在上述範例中，`UpdateSearchIndex` Job 是 Unique 的。因此，如果佇列中已存在另一個 Job 實例且尚未完成處理，則該 Job 將不會被分派。

在某些情況下，您可能希望定義一個特定的「鍵」來使 Job Unique，或者您可能希望指定一個逾時時間，超過該時間後 Job 將不再保持 Unique。為此，您可以在 Job 類別上定義 `uniqueId` 和 `uniqueFor` 屬性或方法：

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

在上述範例中，`UpdateSearchIndex` Job 根據產品 ID 保持 Unique。因此，任何具有相同產品 ID 的新 Job 分派都將被忽略，直到現有 Job 完成處理。此外，如果現有 Job 在一小時內未處理完畢，則 Unique 鎖定將被釋放，並且可以將具有相同 Unique 鍵的另一個 Job 分派到佇列。

> [!WARNING]
> 如果您的應用程式從多個網路伺服器或容器分派 Jobs，您應該確保所有伺服器都與相同的中央快取伺服器通訊，以便 Laravel 能夠準確判斷 Job 是否 Unique。

<a name="keeping-jobs-unique-until-processing-begins"></a>
#### 在處理開始前保持 Job 的 Unique

預設情況下，Unique job 在 Job 完成處理或所有重試嘗試失敗後會「解鎖」。然而，在某些情況下，您可能希望 Job 在處理之前立即解鎖。為此，您的 Job 應該實作 `ShouldBeUniqueUntilProcessing` 契約，而不是 `ShouldBeUnique` 契約：

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
#### Unique Job 鎖定

在幕後，當一個 `ShouldBeUnique` Job 被分派時，Laravel 會嘗試使用 `uniqueId` 鍵來取得一個 [鎖定](/docs/{{version}}/cache#atomic-locks)。如果該鎖定已被佔用，則 Job 將不會被分派。當 Job 完成處理或所有重試嘗試失敗後，此鎖定會被釋放。預設情況下，Laravel 會使用預設的快取驅動器來取得此鎖定。但是，如果您希望使用另一個驅動器來取得鎖定，您可以定義一個 `uniqueVia` 方法來回傳應該使用的快取驅動器：

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
> 如果您只需要限制 Job 的並行處理，請改用 [WithoutOverlapping](/docs/{{version}}/queues#preventing-job-overlaps) job 中介層。

<a name="encrypted-jobs"></a>
### 加密的 Jobs

Laravel 允許您透過 [加密](/docs/{{version}}/encryption) 來確保 Job 資料的隱私和完整性。若要開始，只需將 `ShouldBeEncrypted` 介面新增到 Job 類別。一旦此介面已新增到類別中，Laravel 將會自動加密您的 Job，然後再將其推送到佇列：

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

Job 中介層讓您可以在佇列 Jobs 的執行周圍包裝自訂邏輯，減少 Jobs 本身中的樣板程式碼。例如，考慮以下利用 Laravel 的 Redis 速率限制功能，以允許每五秒只能處理一個 Job 的 `handle` 方法：

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

儘管此程式碼是有效的，但 `handle` 方法的實作變得嘈雜，因為它充滿了 Redis 速率限制邏輯。此外，這種速率限制邏輯必須複製到任何其他我們想要進行速率限制的 Jobs。與其在 handle 方法中進行速率限制，我們可以定義一個處理速率限制的 Job 中介層：

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

如您所見，如同 [路由中介層](/docs/{{version}}/middleware) 一樣，Job 中介層接收正在處理的 Job 和一個應該被呼叫以繼續處理 Job 的回調。

您可以使用 `make:job-middleware` Artisan 命令來產生新的 Job 中介層類別。建立 Job 中介層後，可以透過從 Job 的 `middleware` 方法中回傳它們來將其附加到 Job。這個方法不存在於由 `make:job` Artisan 命令生成的 Jobs 中，因此您需要手動將其新增到您的 Job 類別：

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
> Job 中介層也可以指派給 [可佇列事件監聽器](/docs/{{version}}/events#queued-event-listeners)、[Mailables](/docs/{{version}}/mail#queueing-mail) 和 [Notifications](/docs/{{version}}/notifications#queueing-notifications)。

<a name="rate-limiting"></a>
### 速率限制

儘管我們剛才展示了如何編寫自己的速率限制 Job 中介層，但 Laravel 實際上包含了一個速率限制中介層，您可以使用它來限制 Jobs 的速率。就像 [路由速率限制器](/docs/{{version}}/routing#defining-rate-limiters) 一樣，Job 速率限制器是使用 `RateLimiter` Facade 的 `for` 方法來定義的。

例如，您可能希望允許使用者每小時備份一次資料，同時對高階客戶不施加此類限制。為此，您可以在 `AppServiceProvider` 的 `boot` 方法中定義 `RateLimiter`：

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

在上面的範例中，我們定義了每小時的速率限制；但是，您可以使用 `perMinute` 方法輕鬆定義基於分鐘的速率限制。此外，您可以將任何您希望的值傳遞給速率限制的 `by` 方法；然而，這個值最常用於按客戶區分速率限制：

```php
return Limit::perMinute(50)->by($job->user->id);
```

一旦您定義了速率限制，您可以使用 `Illuminate\Queue\Middleware\RateLimited` 中介層將其附加到您的 Job。每當 Job 超過速率限制時，此中介層將根據速率限制的持續時間，以適當的延遲將 Job 釋放回佇列：

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

將速率限制的 Job 釋放回佇列，仍然會增加 Job 的總 `attempts` 次數。您可能希望相應地調整 Job 類別上的 `tries` 和 `maxExceptions` 屬性。或者，您可能希望使用 [retryUntil 方法](#time-based-attempts) 來定義 Job 不再嘗試的時間量。

使用 `releaseAfter` 方法，您還可以指定在重新嘗試已釋放的 Job 之前必須經過的秒數：

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

如果您不希望 Job 在速率限制時被重試，您可以使用 `dontRelease` 方法：

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

> [!NOTE]
> 如果您正在使用 Redis，您可以使用 `Illuminate\Queue\Middleware\RateLimitedWithRedis` 中介層，它針對 Redis 進行了優化，比基本的速率限制中介層更有效率。

<a name="preventing-job-overlaps"></a>
### 避免 Job 重疊

Laravel 包含 `Illuminate\Queue\Middleware\WithoutOverlapping` 中介層，它允許您根據任意金鑰防止 Job 重疊。當佇列 Job 正在修改一次只能由一個 Job 修改的資源時，這會很有幫助。

例如，假設您有一個佇列 Job 會更新使用者的信用評分，並且您希望防止相同使用者 ID 的信用評分更新 Job 重疊。為了實現這一點，您可以從 Job 的 `middleware` 方法中回傳 `WithoutOverlapping` 中介層：

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

將重疊的 Job 釋放回佇列仍會增加 Job 的總嘗試次數。您可能希望相應地調整 Job class 上的 `tries` 和 `maxExceptions` 屬性。例如，將 `tries` 屬性保留為預設值 1 將會阻止任何重疊的 Job 在稍後重試。

任何相同類型的重疊 Job 都將被釋放回佇列。您也可以指定在釋放的 Job 再次被嘗試之前必須經過的秒數：

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

如果您希望立即刪除任何重疊的 Job，使其不再重試，您可以使用 `dontRelease` 方法：

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

`WithoutOverlapping` 中介層由 Laravel 的原子鎖定功能提供支援。有時，您的 Job 可能會意外失敗或逾時，導致鎖定未被釋放。因此，您可以使用 `expireAfter` 方法明確定義鎖定過期時間。例如，以下範例將指示 Laravel 在 Job 開始處理後三分鐘釋放 `WithoutOverlapping` 鎖定：

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
> `WithoutOverlapping` 中介層需要支援 [鎖定](/docs/{{version}}/cache#atomic-locks) 的快取驅動器。目前，`memcached`、`redis`、`dynamodb`、`database`、`file` 和 `array` 快取驅動器支援原子鎖定。


<a name="sharing-lock-keys"></a>
#### 跨 Job Class 共享鎖定金鑰

預設情況下，`WithoutOverlapping` 中介層只會防止相同 class 的 Job 重疊。因此，儘管兩個不同的 Job class 可能使用相同的鎖定金鑰，它們並不會被阻止重疊。然而，您可以指示 Laravel 使用 `shared` 方法將金鑰應用於所有 Job class：

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
### 限制例外

Laravel 包含了 `Illuminate\Queue\Middleware\ThrottlesExceptions` 中介層，讓您可以限制例外。一旦 job 拋出指定數量的例外，所有後續執行該 job 的嘗試都會延遲，直到指定時間間隔過後。此中介層對於與不穩定的第三方服務互動的 job 特別有用。

例如，假設有一個佇列 job 與開始拋出例外的第三方 API 互動。為了限制例外，您可以從 job 的 `middleware` 方法回傳 `ThrottlesExceptions` 中介層。通常，此中介層應該與實作 [基於時間的嘗試](#time-based-attempts) 的 job 搭配使用：

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
    return now()->addMinutes(30);
}
```

此中介層接受的第一個建構子引數是 job 在被限制之前可以拋出的例外數量，而第二個建構子引數是 job 被限制後，在再次嘗試該 job 之前應該經過的秒數。在上述程式碼範例中，如果 job 拋出 10 個連續例外，我們將等待 5 分鐘後再嘗試該 job，並受 30 分鐘的時間限制。

當 job 拋出例外但例外閾值尚未達到時，該 job 通常會立即重試。然而，您可以在將中介層附加到 job 時呼叫 `backoff` 方法，來指定該 job 應該延遲的分鐘數：

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

在內部，此中介層使用 Laravel 的快取系統來實作速率限制，並且 job 的類別名稱被用作快取「鍵 (key)」。您可以在將中介層附加到 job 時呼叫 `by` 方法來覆寫這個鍵。這在您有多個 job 與相同的第三方服務互動，並且希望它們共享一個共同的限制「桶」以確保它們遵守單一的共享限制時可能很有用：

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

預設情況下，此中介層會限制每個例外。您可以在將中介層附加到 job 時呼叫 `when` 方法來修改此行為。只有當提供給 `when` 方法的閉包回傳 `true` 時，例外才會被限制：

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

與 `when` 方法會將 job 釋放回佇列或拋出例外不同，`deleteWhen` 方法允許您在發生指定例外時完全刪除 job：

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

如果您希望將被限制的例外報告給應用程式的例外處理器，您可以在將中介層附加到 job 時呼叫 `report` 方法。可選地，您可以為 `report` 方法提供一個閉包，只有當給定的閉包回傳 `true` 時，例外才會被報告：

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

> [!NOTE]
> 如果您正在使用 Redis，您可以使用 `Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis` 中介層，它針對 Redis 進行了優化，比基本的例外限制中介層更有效率。

<a name="skipping-jobs"></a>
### 跳過 Jobs

`Skip` 中介層允許您指定 job 應該被跳過 / 刪除，而無需修改 job 的邏輯。如果給定的條件評估為 `true`，`Skip::when` 方法將刪除該 job；而如果條件評估為 `false`，`Skip::unless` 方法將刪除該 job：

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

您也可以傳入一個 `Closure` 給 `when` 和 `unless` 方法，進行更複雜的條件判斷：

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
## 分派 Jobs

一旦您編寫好 Job class，就可以使用 Job 自身的 `dispatch` 方法來分派它。傳遞給 `dispatch` 方法的參數將會傳入 Job 的建構函式：

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

如果您想條件式地分派 Job，可以使用 `dispatchIf` 和 `dispatchUnless` 方法：

```php
ProcessPodcast::dispatchIf($accountActive, $podcast);

ProcessPodcast::dispatchUnless($accountSuspended, $podcast);
```

在新的 Laravel 應用程式中，`database` 驅動器是預設的佇列驅動器。您可以在應用程式的 `config/queue.php` 設定檔中指定不同的佇列驅動器。

<a name="delayed-dispatching"></a>
### 延遲分派

如果您想指定 Job 不應立即由佇列 worker 處理，可以在分派 Job 時使用 `delay` 方法。例如，讓我們指定一個 Job 在分派後 10 分鐘內不應可供處理：

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
            ->delay(now()->addMinutes(10));

        return redirect('/podcasts');
    }
}
```

在某些情況下，Jobs 可能已設定了預設延遲。如果您需要繞過此延遲並立即處理 Job，可以使用 `withoutDelay` 方法：

```php
ProcessPodcast::dispatch($podcast)->withoutDelay();
```

> [!WARNING]
> Amazon SQS 佇列服務的最大延遲時間為 15 分鐘。

<a name="dispatching-after-the-response-is-sent-to-browser"></a>
#### 在回應送出至瀏覽器後分派

另外，如果您的 Web 伺服器使用 [FastCGI](https://www.php.net/manual/en/install.fpm.php)，`dispatchAfterResponse` 方法會將 Job 的分派延遲到 HTTP 回應送達使用者瀏覽器之後。這仍然允許使用者開始使用應用程式，即使佇列中的 Job 仍在執行。這通常只適用於耗時約一秒的 Jobs，例如傳送電子郵件。由於它們是在目前的 HTTP 請求中處理的，以這種方式分派的 Jobs 不需要佇列 worker 執行即可處理：

```php
use App\Jobs\SendNotification;

SendNotification::dispatchAfterResponse();
```

您也可以 `dispatch` 一個閉包 (closure) 並將 `afterResponse` 方法鏈接到 [dispatch helper](/docs/{{version}}/helpers#method-dispatch) 上，以便在 HTTP 回應送達瀏覽器後執行該閉包：

```php
use App\Mail\WelcomeMessage;
use Illuminate\Support\Facades\Mail;

dispatch(function () {
    Mail::to('taylor@example.com')->send(new WelcomeMessage);
})->afterResponse();
```

<a name="synchronous-dispatching"></a>
### 同步分派

如果您想立即 (同步) 分派 Job，可以使用 `dispatchSync` 方法。當使用此方法時，Job 將不會被佇列，而是會在目前處理程序中立即執行：

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

<a name="jobs-and-database-transactions"></a>
### Jobs 與資料庫交易

雖然在資料庫交易中分派 Jobs 是完全沒問題的，但您應特別注意確保您的 Job 能夠成功執行。當在交易中分派 Job 時，Job 有可能在父交易提交之前被 worker 處理。當這種情況發生時，您在資料庫交易期間對 Model 或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何 Model 或資料庫記錄可能在資料庫中尚不存在。

幸運的是，Laravel 提供了幾種解決此問題的方法。首先，您可以在佇列連線的設定陣列中設定 `after_commit` 連線選項：

```php
'redis' => [
    'driver' => 'redis',
    // ...
    'after_commit' => true,
],
```

當 `after_commit` 選項為 `true` 時，您可以在資料庫交易中分派 Jobs；然而，Laravel 會等到開啟的父資料庫交易提交後，才會實際分派 Job。當然，如果目前沒有開啟的資料庫交易，Job 將會立即分派。

如果交易因為在交易期間發生的例外而回滾，則在該交易期間分派的 Jobs 將會被丟棄。

> [!NOTE]
> 將 `after_commit` 設定選項設定為 `true` 也會導致所有佇列事件監聽器、mailables、notifications 和廣播事件在所有開啟的資料庫交易提交後分派。

<a name="specifying-commit-dispatch-behavior-inline"></a>
#### 行內指定提交分派行為

如果您未將 `after_commit` 佇列連線設定選項設定為 `true`，您仍然可以指出某個特定的 Job 應在所有開啟的資料庫交易提交後分派。為此，您可以將 `afterCommit` 方法鏈接到您的分派操作上：

```php
use App\Jobs\ProcessPodcast;

ProcessPodcast::dispatch($podcast)->afterCommit();
```

同樣地，如果 `after_commit` 設定選項設定為 `true`，您可以指出某個特定的 Job 應立即分派，而無需等待任何開啟的資料庫交易提交：

```php
ProcessPodcast::dispatch($podcast)->beforeCommit();
```

<a name="job-chaining"></a>
### Job 鏈接

Job 鏈接允許您指定一系列佇列 Jobs，這些 Jobs 在主要 Job 成功執行後應按順序運行。如果序列中的某個 Job 失敗，其餘 Jobs 將不會運行。要執行佇列 Job 鏈接，您可以使用 `Bus` Facade 提供的 `chain` 方法。Laravel 的命令 Bus 是一個底層元件，佇列 Job 分派是建立在其之上：

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

除了鏈接 Job Class 實例之外，您也可以鏈接閉包：

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
> 在 Job 中使用 `$this->delete()` 方法刪除 Jobs 並不會阻止鏈接中的 Jobs 繼續處理。只有當鏈接中的 Job 失敗時，鏈接才會停止執行。

<a name="chain-connection-queue"></a>
#### 鏈接的連線與佇列

如果您想為鏈接的 Jobs 指定應使用的連線與佇列，您可以使用 `onConnection` 與 `onQueue` 方法。這些方法會指定應使用的佇列連線與佇列名稱，除非佇列 Job 被明確指定了不同的連線/佇列：

```php
Bus::chain([
    new ProcessPodcast,
    new OptimizePodcast,
    new ReleasePodcast,
])->onConnection('redis')->onQueue('podcasts')->dispatch();
```

<a name="adding-jobs-to-the-chain"></a>
#### 將 Jobs 加入鏈接

有時，您可能需要從鏈接中的另一個 Job 內部，將 Job 預置或附加到現有的 Job 鏈接。您可以使用 `prependToChain` 與 `appendToChain` 方法來實現此目的：

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
#### 鏈接失敗

在鏈接 Jobs 時，您可以使用 `catch` 方法指定一個閉包，當鏈接中的 Job 失敗時應調用此閉包。此回呼會接收導致 Job 失敗的 `Throwable` 實例：

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
> 由於鏈接回呼會被 Laravel 佇列序列化並在稍後執行，因此您不應在鏈接回呼中使用 `$this` 變數。

<a name="customizing-the-queue-and-connection"></a>
### 客製化佇列與連線

<a name="dispatching-to-a-particular-queue"></a>
#### 分派至特定佇列

透過將 Jobs 推送到不同的佇列，您可以「分類」您的佇列 Jobs，甚至可以優先處理您分配給不同佇列的 Worker 數量。請記住，這並非將 Jobs 推送到佇列設定檔中定義的不同佇列「連線」，而僅是推送到單一連線中的特定佇列。要指定佇列，請在分派 Job 時使用 `onQueue` 方法：

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

或者，您可以在 Job 的建構函式中調用 `onQueue` 方法來指定 Job 的佇列：

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
#### 分派至特定連線

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

您可以將 `onConnection` 與 `onQueue` 方法鏈接在一起，以指定 Job 的連線和佇列：

```php
ProcessPodcast::dispatch($podcast)
    ->onConnection('sqs')
    ->onQueue('processing');
```

或者，您可以在 Job 的建構函式中調用 `onConnection` 方法來指定 Job 的連線：

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
### 指定 Job 最大嘗試次數 / 逾時值


<a name="max-attempts"></a>
#### 最大嘗試次數

Job 嘗試是 Laravel 佇列系統的核心概念，並支援許多進階功能。儘管它們一開始可能看起來令人困惑，但在修改預設設定之前，了解它們的運作方式非常重要。

當一個 job 被分派時，它會被推送到佇列中。然後 worker 會接收它並嘗試執行它。這就是一次 job 嘗試。

然而，一次嘗試不一定表示 job 的 `handle` 方法已經執行。嘗試也可能透過多種方式「被消耗」：

<div class="content-list" markdown="1">

- Job 在執行期間遇到未處理的例外。
- Job 使用 `$this->release()` 手動釋放回佇列。
- `WithoutOverlapping` 或 `RateLimited` 等中介層未能取得鎖定並釋放 Job。
- Job 逾時。
- Job 的 `handle` 方法執行完成且未拋出例外。

</div>

您可能不希望無限期地嘗試一個 job。因此，Laravel 提供了多種方式來指定一個 job 可以被嘗試多少次或嘗試多長時間。

> [!NOTE]
> By default, Laravel will only attempt a job once. If your job uses middleware like `WithoutOverlapping` or `RateLimited`, or if you're manually releasing jobs, you will likely need to increase the number of allowed attempts via the `tries` option.

指定 job 最大嘗試次數的一種方法是透過 Artisan 命令列上的 `--tries` 參數。這將適用於 worker 處理的所有 job，除非正在處理的 job 指定了它可能被嘗試的次數：

```shell
php artisan queue:work --tries=3
```

如果一個 job 超過其最大嘗試次數，它將被視為一個「失敗」的 job。有關處理失敗 job 的更多資訊，請查閱 [失敗 job 文件](#dealing-with-failed-jobs)。如果提供 `--tries=0` 給 `queue:work` 命令，該 job 將會無限期地重試。

您可以透過在 job class 本身定義 job 的最大嘗試次數來採用更細緻的方法。如果在 job 上指定了最大嘗試次數，它將優先於命令列上提供的 `--tries` 值：

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

如果您需要對特定 job 的最大嘗試次數進行動態控制，您可以在 job 上定義一個 `tries` 方法：

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
#### 基於時間的嘗試次數

作為定義 job 在失敗前可嘗試多少次的替代方案，您可以定義 job 不再應被嘗試的時間點。這允許一個 job 在給定時間範圍內被嘗試任意次數。要定義 job 不再應被嘗試的時間，請在您的 job class 中添加一個 `retryUntil` 方法。此方法應返回一個 `DateTime` 實例：

```php
use DateTime;

/**
 * Determine the time at which the job should timeout.
 */
public function retryUntil(): DateTime
{
    return now()->addMinutes(10);
}
```

如果同時定義了 `retryUntil` 和 `tries`，Laravel 將優先採用 `retryUntil` 方法。

> [!NOTE]
> You may also define a `tries` property or `retryUntil` method on your [queued event listeners](/docs/{{version}}/events#queued-event-listeners) and [queued notifications](/docs/{{version}}/notifications#queueing-notifications).


<a name="max-exceptions"></a>
#### 最大例外數

有時您可能希望指定一個 job 可以被嘗試多次，但如果重試是由給定數量的未處理例外觸發（而不是直接由 `release` 方法釋放）則應該失敗。為此，您可以在 job class 上定義一個 `maxExceptions` 屬性：

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

在此範例中，如果應用程式無法取得 Redis 鎖定，則該 job 將被釋放十秒，並將繼續重試最多 25 次。然而，如果 job 拋出三個未處理的例外，該 job 將會失敗。


<a name="timeout"></a>
#### 逾時

通常，您大約知道您的佇列 job 預計需要多長時間。因此，Laravel 允許您指定一個「逾時」值。預設情況下，逾時值為 60 秒。如果一個 job 處理的時間超過逾時值所指定的秒數，處理該 job 的 worker 將會因錯誤而退出。通常，worker 將會由 [您伺服器上配置的處理程序管理器](#supervisor-configuration) 自動重新啟動。

job 可執行的最大秒數可以透過 Artisan 命令列上的 `--timeout` 參數來指定：

```shell
php artisan queue:work --timeout=30
```

如果 job 因持續逾時而超過其最大嘗試次數，它將被標記為失敗。

您也可以在 job class 本身定義一個 job 允許執行的最大秒數。如果在 job 上指定了逾時，它將優先於命令列上指定的任何逾時值：

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

有時，IO 阻塞的程序，例如 sockets 或對外 HTTP 連線，可能不會遵守您指定的逾時。因此，使用這些功能時，您應始終嘗試使用其 API 來指定逾時。例如，使用 [Guzzle](https://docs.guzzlephp.org) 時，您應始終指定連線和請求的逾時值。

> [!WARNING]
> The [PCNTL](https://www.php.net/manual/en/book.pcntl.php) PHP extension must be installed in order to specify job timeouts. In addition, a job's "timeout" value should always be less than its ["retry after"](#job-expiration) value. Otherwise, the job may be re-attempted before it has actually finished executing or timed out.


<a name="failing-on-timeout"></a>
#### 逾時失敗

如果您希望在逾時時將 job 標記為 [失敗](#dealing-with-failed-jobs)，您可以在 job class 上定義 `$failOnTimeout` 屬性：

```php
/**
 * Indicate if the job should be marked as failed on timeout.
 *
 * @var bool
 */
public $failOnTimeout = true;
```

> [!NOTE]
> By default, when a job times out, it consumes one attempt and is released back to the queue (if retries are allowed). However, if you configure the job to fail on timeout, it will not be retried, regardless of the value set for tries.

<a name="sqs-fifo-and-fair-queues"></a>
### SQS FIFO 與公平佇列

Laravel 支援 [Amazon SQS FIFO (First-In-First-Out)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html) 佇列，讓您可以依照 Jobs 發送的確切順序處理 Jobs，同時透過訊息去重複確保只會處理一次。

FIFO 佇列需要一個訊息群組 ID 來判斷哪些 Jobs 可以平行處理。擁有相同群組 ID 的 Jobs 會依序處理，而擁有不同群組 ID 的訊息則可以並行處理。

Laravel 提供一個流暢的 `onGroup` 方法來指定分派 Jobs 時的訊息群組 ID：

```php
ProcessOrder::dispatch($order)
    ->onGroup("customer-{$order->customer_id}");
```

SQS FIFO 佇列支援訊息去重複，以確保只處理一次。在您的 Job class 中實作一個 `deduplicationId` 方法來提供一個客製化的去重複 ID：

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
#### FIFO Listeners、Mail 與 Notifications

當使用 FIFO 佇列時，您也需要定義 listeners、mail 與 notifications 的訊息群組。或者，您可以將這些物件的佇列實例分派到非 FIFO 佇列。

若要為 [queued event listener](/docs/{{version}}/events#queued-event-listeners) 定義訊息群組，請在 listener 上定義一個 `messageGroup` 方法。您也可以選擇性地定義一個 `deduplicationId` 方法：

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;

class SendShipmentNotification
{
    // ...

    /**
     * Get the job's message group.
     */
    public function messageGroup(): string
    {
        return "shipments";
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

當發送一個準備在 FIFO 佇列中佇列化的 [mail message](/docs/{{version}}/mail) 時，您應該在發送 notification 時呼叫 `onGroup` 方法，並選擇性地呼叫 `withDeduplicator` 方法：

```php
use App\Mail\InvoicePaid;
use Illuminate\Support\Facades\Mail;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

Mail::to($request->user())->send($invoicePaid);
```

當發送一個準備在 FIFO 佇列中佇列化的 [notification](/docs/{{version}}/notifications) 時，您應該在發送 notification 時呼叫 `onGroup` 方法，並選擇性地呼叫 `withDeduplicator` 方法：

```php
use App\Notifications\InvoicePaid;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

$user->notify($invoicePaid);
```

<a name="error-handling"></a>
### 錯誤處理

如果 Job 在處理期間拋出 exception，Job 將會自動釋放回佇列中，以便再次嘗試執行。Job 將會持續釋放，直到達到應用程式允許的最大嘗試次數為止。最大嘗試次數由 `queue:work` Artisan 命令中使用的 `--tries` 開關定義。或者，最大嘗試次數也可以在 Job class 本身中定義。有關執行佇列 worker 的更多資訊，[請參閱下方](#running-the-queue-worker)。

<a name="manually-releasing-a-job"></a>
#### 手動釋放 Job

有時您可能希望手動將 Job 釋放回佇列，以便稍後再次嘗試。您可以透過呼叫 `release` 方法來達成此目的：

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

預設情況下，`release` 方法會將 Job 釋放回佇列以立即處理。然而，您可以指示佇列在經過指定秒數後才讓 Job 可供處理，透過傳遞一個整數或日期實例給 `release` 方法：

```php
$this->release(10);

$this->release(now()->addSeconds(10));
```

<a name="manually-failing-a-job"></a>
#### 手動讓 Job 失敗

有時您可能需要手動將 Job 標記為「失敗」。若要這麼做，您可以呼叫 `fail` 方法：

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

如果您想因為捕獲到的 exception 而將 Job 標記為失敗，您可以將 exception 傳遞給 `fail` 方法。或者，為了方便，您可以傳遞一個字串錯誤訊息，它會自動轉換為一個 exception：

```php
$this->fail($exception);

$this->fail('Something went wrong.');
```

> [!NOTE]
> 有關失敗 Jobs 的更多資訊，請查閱 [處理 Job 失敗的文件](#dealing-with-failed-jobs)。

<a name="fail-jobs-on-exceptions"></a>
#### 讓 Jobs 在特定例外時失敗

`FailOnException` [job 中介層](#job-middleware) 允許您在拋出特定 exceptions 時短路重試機制。這使得可以在暫時性 exceptions (例如外部 API 錯誤) 時重試，但對於持久性 exceptions (例如使用者的權限被撤銷) 則會讓 Job 永久失敗：

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
        $user->authorize('sync-chat-history');

        $response = Http::throw()->get(
            "https://chat.laravel.test/?user={$user->uuid}"
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

Laravel 的 Job 批次處理功能讓您可以輕鬆地執行一批 Job，並在 Job 批次完成執行後執行一些操作。在開始之前，您應該建立一個資料庫 Migration 來建立一個資料表，其中將包含有關您 Job 批次的元資訊，例如它們的完成百分比。此 Migration 可以使用 `make:queue-batches-table` Artisan 命令產生：

```shell
php artisan make:queue-batches-table

php artisan migrate
```

<a name="defining-batchable-jobs"></a>
### 定義可批次處理的 Jobs

要定義一個可批次處理的 Job，您應該像平常一樣[建立一個可佇列化的 Job](#creating-jobs)；但是，您應該將 `Illuminate\Bus\Batchable` trait 新增到 Job class。此 trait 提供了 `batch` 方法的存取權限，該方法可用於檢索 Job 正在其中執行的當前批次：

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
### 分派批次

要分派一批 Jobs，您應該使用 `Bus` Facade 的 `batch` 方法。當然，批次處理主要在結合完成回呼時才有用。因此，您可以使用 `then`、`catch` 和 `finally` 方法來為批次定義完成回呼。這些回呼在被叫用時都將接收一個 `Illuminate\Bus\Batch` 實例。在這個範例中，我們將假設我們正在佇列一批 Job，每個 Job 處理 CSV 檔案中的給定數量的資料列：

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
    // First batch job failure detected...
})->finally(function (Batch $batch) {
    // The batch has finished executing...
})->dispatch();

return $batch->id;
```

批次的 ID (可透過 `$batch->id` 屬性存取) 可用於在分派後 [向 Laravel 命令匯流排查詢](#inspecting-batches) 有關批次的資訊。

> [!WARNING]
> 由於批次回呼會被序列化並在稍後由 Laravel 佇列執行，因此您不應在回呼中使用 `$this` 變數。此外，由於批次處理的 Jobs 會被包裝在資料庫交易中，因此不應在 Jobs 內部執行觸發隱式提交的資料庫語句。

<a name="naming-batches"></a>
#### 命名批次

如果批次有命名，某些工具 (例如 [Laravel Horizon](/docs/{{version}}/horizon) 和 [Laravel Telescope](/docs/{{version}}/telescope)) 可以為批次提供更友善的偵錯資訊。要為批次指派任意名稱，您可以在定義批次時叫用 `name` 方法：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import CSV')->dispatch();
```

<a name="batch-connection-queue"></a>
#### 批次連線與佇列

如果您想指定用於批次處理 Jobs 的連線和佇列，您可以使用 `onConnection` 和 `onQueue` 方法。所有批次處理的 Jobs 都必須在相同的連線和佇列中執行：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->onConnection('redis')->onQueue('imports')->dispatch();
```

<a name="chains-and-batches"></a>
### 鏈接與批次

您可以透過將鏈接 Jobs 放置在陣列中，在批次內定義一組[鏈接 Jobs](#job-chaining)。例如，我們可以並行執行兩個 Job 鏈接，並在兩個 Job 鏈接都完成處理時執行回呼：

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

反之，您可以透過在[鏈接](#job-chaining)內定義批次來在鏈接內執行批次 Jobs。例如，您可以先執行一批 Jobs 來發布多個 Podcast，然後再執行一批 Jobs 來發送發布通知：

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
### 將 Jobs 加入批次

有時，從批次處理的 Job 內部向批次新增額外的 Jobs 可能會很有用。當您需要批次處理數千個 Job，而這些 Job 在 Web 請求期間分派可能需要太長時間時，此模式會很有用。因此，您可以分派一個初始的「載入器」Job 批次，該批次將使用更多的 Jobs 來豐富該批次：

```php
$batch = Bus::batch([
    new LoadImportBatch,
    new LoadImportBatch,
    new LoadImportBatch,
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import Contacts')->dispatch();
```

在此範例中，我們將使用 `LoadImportBatch` Job 來豐富批次，使其包含額外的 Jobs。要實現這一點，我們可以使用可透過 Job 的 `batch` 方法存取的批次實例上的 `add` 方法：

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
> 您只能從屬於同一批次的 Job 內部向批次新增 Jobs。

<a name="inspecting-batches"></a>
### 檢查批次

提供給批次完成回呼 (callbacks) 的 `Illuminate\Bus\Batch` 實例具有各種屬性與方法，可協助您與特定 Job 批次互動及檢查：

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

所有 `Illuminate\Bus\Batch` 實例都是可 JSON 序列化的，這表示您可以直接從應用程式的路由回傳它們，以取得包含批次相關資訊 (包括其完成進度) 的 JSON 負載。這使得在應用程式的 UI 中顯示批次的完成進度資訊非常方便。

要依據批次的 ID 擷取批次，您可以使用 `Bus` Facade 的 `findBatch` 方法：

```php
use Illuminate\Support\Facades\Bus;
use Illuminate\Support\Facades\Route;

Route::get('/batch/{batchId}', function (string $batchId) {
    return Bus::findBatch($batchId);
});
```


<a name="cancelling-batches"></a>
### 取消批次

有時您可能需要取消特定批次的執行。這可以透過呼叫 `Illuminate\Bus\Batch` 實例上的 `cancel` 方法來實現：

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

正如您可能在前面的範例中注意到的，批次 Jobs 通常應該在繼續執行之前判斷其對應的批次是否已取消。然而，為了方便起見，您可以將 `SkipIfBatchCancelled` [中介層](#job-middleware) 分派給 Job。顧名思義，此中介層將指示 Laravel，如果 Job 對應的批次已取消，則不處理該 Job：

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

當批次 Job 失敗時，會呼叫 `catch` 回呼 (如果已分派)。此回呼只會在批次中第一個失敗的 Job 失敗時才呼叫。


<a name="allowing-failures"></a>
#### 允許失敗

當批次中的 Job 失敗時，Laravel 會自動將該批次標記為「已取消」。如果您願意，可以禁用此行為，這樣 Job 失敗就不會自動將批次標記為已取消。這可以透過在分派批次時呼叫 `allowFailures` 方法來實現：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->allowFailures()->dispatch();
```

您可以選擇性地為 `allowFailures` 方法提供一個閉包，該閉包將在每次 Job 失敗時執行：

```php
$batch = Bus::batch([
    // ...
])->allowFailures(function (Batch $batch, $exception) {
    // Handle individual job failures...
})->dispatch();
```


<a name="retrying-failed-batch-jobs"></a>
#### 重試失敗的批次 Jobs

為了方便起見，Laravel 提供了 `queue:retry-batch` Artisan 命令，可讓您輕鬆重試給定批次中所有失敗的 Jobs。此命令接受應重試其失敗 Jobs 的批次的 UUID：

```shell
php artisan queue:retry-batch 32dbc76c-4f82-4749-b610-a639fe0099b5
```


<a name="pruning-batches"></a>
### 清理批次

如果不進行清理，`job_batches` 資料表可能會很快累積大量的記錄。為此，您應該 [排程](/docs/{{version}}/scheduling) `queue:prune-batches` Artisan 命令每天執行：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches')->daily();
```

預設情況下，所有超過 24 小時的已完成批次都會被清理。您可以在呼叫命令時使用 `hours` 選項來決定保留批次資料的時間長度。例如，以下命令將刪除所有超過 48 小時前完成的批次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48')->daily();
```

有時，您的 `jobs_batches` 資料表可能會累積未成功完成的批次記錄，例如 Job 失敗且從未成功重試的批次。您可以使用 `unfinished` 選項指示 `queue:prune-batches` 命令清理這些未完成的批次記錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --unfinished=72')->daily();
```

同樣地，您的 `jobs_batches` 資料表也可能累積已取消批次的記錄。您可以使用 `cancelled` 選項指示 `queue:prune-batches` 命令清理這些已取消的批次記錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --cancelled=72')->daily();
```

<a name="storing-batches-in-dynamodb"></a>
### 將批次儲存至 DynamoDB

Laravel 也支援將批次的中繼資訊儲存在 [DynamoDB](https://aws.amazon.com/dynamodb) 中，而非關聯式資料庫。然而，您需要手動建立一個 DynamoDB 資料表來儲存所有的批次記錄。

通常，此資料表應命名為 `job_batches`，但您應根據應用程式 `queue` 設定檔中 `queue.batching.table` 設定值的內容來命名資料表。

<a name="dynamodb-batch-table-configuration"></a>
#### DynamoDB 批次資料表設定

`job_batches` 資料表應具有一個名為 `application` 的字串主分割區鍵和一個名為 `id` 的字串主排序鍵。鍵的 `application` 部分將包含您的應用程式名稱，該名稱由應用程式 `app` 設定檔中的 `name` 設定值所定義。由於應用程式名稱是 DynamoDB 資料表鍵的一部分，您可以使用同一個資料表來儲存多個 Laravel 應用程式的 Job 批次。

此外，如果您希望利用 [自動批次清理](#pruning-batches-in-dynamodb)，您也可以為資料表定義 `ttl` 屬性。

<a name="dynamodb-configuration"></a>
#### DynamoDB 設定

接下來，安裝 AWS SDK，讓您的 Laravel 應用程式可以與 Amazon DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

然後，將 `queue.batching.driver` 設定選項的值設定為 `dynamodb`。此外，您應在 `batching` 設定陣列中定義 `key`、`secret` 和 `region` 設定選項。這些選項將用於 AWS 身份驗證。當使用 `dynamodb` 驅動器時，`queue.batching.database` 設定選項是不必要的：

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

當利用 [DynamoDB](https://aws.amazon.com/dynamodb) 儲存 Job 批次資訊時，用於清理儲存在關聯式資料庫中批次的傳統清理命令將不起作用。相反地，您可以利用 [DynamoDB 原生的 TTL 功能](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html) 來自動移除舊批次的記錄。

如果您為 DynamoDB 資料表定義了 `ttl` 屬性，您可以定義設定參數來指示 Laravel 如何清理批次記錄。`queue.batching.ttl_attribute` 設定值定義了持有 TTL 的屬性名稱，而 `queue.batching.ttl` 設定值定義了批次記錄從 DynamoDB 資料表移除前，相對於上次記錄更新的時間，所需的秒數：

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
## 佇列化閉包

您可以分派一個閉包到佇列，而非分派一個 job class。這對於需要在目前請求週期之外執行的快速、簡單任務非常有用。當分派閉包到佇列時，閉包的程式碼內容會經過加密簽章，以確保其在傳輸過程中不會被修改：

```php
use App\Models\Podcast;

$podcast = Podcast::find(1);

dispatch(function () use ($podcast) {
    $podcast->publish();
});
```

若要為佇列化的閉包指定一個名稱，以便佇列報告儀表板使用，以及在 `queue:work` 命令中顯示，您可以使用 `name` 方法：

```php
dispatch(function () {
    // ...
})->name('Publish Podcast');
```

透過 `catch` 方法，您可以提供一個閉包，該閉包應在佇列化閉包耗盡所有佇列的 [配置重試嘗試次數](#max-job-attempts-and-timeout) 仍未能成功完成時執行：

```php
use Throwable;

dispatch(function () use ($podcast) {
    $podcast->publish();
})->catch(function (Throwable $e) {
    // This job has failed...
});
```

> [!WARNING]
> 由於 `catch` 回呼是由 Laravel 佇列序列化並在稍後執行的，因此您不應在 `catch` 回呼中使用 `$this` 變數。


<a name="running-the-queue-worker"></a>
## 執行佇列 Worker


<a name="the-queue-work-command"></a>
### `queue:work` 命令

Laravel 包含一個 Artisan 命令，它會啟動一個佇列 worker，並處理被推送到佇列中的新 job。您可以使用 `queue:work` Artisan 命令來執行 worker。請注意，一旦 `queue:work` 命令啟動，它將持續運行，直到您手動停止或關閉終端機為止：

```shell
php artisan queue:work
```

> [!NOTE]
> 為了讓 `queue:work` 程序在背景永久運行，您應該使用程序監控器，例如 [Supervisor](#supervisor-configuration)，以確保佇列 worker 不會停止運行。

如果您希望命令的輸出包含已處理的 job ID、連線名稱和佇列名稱，可以在呼叫 `queue:work` 命令時加入 `-v` 旗標：

```shell
php artisan queue:work -v
```

請記住，佇列 worker 是長生命週期的程序，並將啟動的應用程式狀態儲存在記憶體中。因此，一旦它們啟動後，將不會察覺到程式碼基底的變化。因此，在您的部署過程中，請務必 [重新啟動佇列 worker](#queue-workers-and-deployment)。此外，請記住，由您的應用程式建立或修改的任何靜態狀態都不會在 job 之間自動重設。

或者，您可以執行 `queue:listen` 命令。當使用 `queue:listen` 命令時，當您需要重新載入更新後的程式碼或重設應用程式狀態時，無需手動重新啟動 worker；然而，此命令的效率明顯低於 `queue:work` 命令：

```shell
php artisan queue:listen
```


<a name="running-multiple-queue-workers"></a>
#### 執行多個佇列 Worker

若要為一個佇列指派多個 worker 並同時處理多個 job，您只需啟動多個 `queue:work` 程序。這可以在本地透過終端機的多個分頁完成，或在生產環境中使用您的程序管理器的配置設定。[當使用 Supervisor](#supervisor-configuration) 時，您可以使用 `numprocs` 配置值。


<a name="specifying-the-connection-queue"></a>
#### 指定連線與佇列

您也可以指定 worker 應使用的佇列連線。傳遞給 `work` 命令的連線名稱應與您 `config/queue.php` 配置檔中定義的連線之一相對應：

```shell
php artisan queue:work redis
```

預設情況下，`queue:work` 命令只處理指定連線上的預設佇列中的 job。然而，您可以透過僅處理指定連線上的特定佇列，進一步客製化您的佇列 worker。例如，如果您的所有電子郵件都在 `redis` 佇列連線上的 `emails` 佇列中處理，您可以發出以下命令來啟動一個只處理該佇列的 worker：

```shell
php artisan queue:work redis --queue=emails
```


<a name="processing-a-specified-number-of-jobs"></a>
#### 處理指定數量的 Jobs

您可以使用 `--once` 選項來指示 worker 僅從佇列中處理一個 job：

```shell
php artisan queue:work --once
```

您可以使用 `--max-jobs` 選項來指示 worker 處理指定數量的 job，然後退出。此選項與 [Supervisor](#supervisor-configuration) 結合使用時可能很有用，如此一來，您的 worker 在處理指定數量的 job 後會自動重新啟動，釋放可能累積的記憶體：

```shell
php artisan queue:work --max-jobs=1000
```


<a name="processing-all-queued-jobs-then-exiting"></a>
#### 處理所有佇列中的 Jobs 然後退出

您可以使用 `--stop-when-empty` 選項來指示 worker 處理所有 job，然後優雅地退出。當在 Docker 容器中處理 Laravel 佇列時，如果您希望在佇列為空後關閉容器，此選項會很有用：

```shell
php artisan queue:work --stop-when-empty
```


<a name="processing-jobs-for-a-given-number-of-seconds"></a>
#### 處理指定秒數的 Jobs

您可以使用 `--max-time` 選項來指示 worker 處理指定秒數的 job，然後退出。此選項與 [Supervisor](#supervisor-configuration) 結合使用時可能很有用，如此一來，您的 worker 在處理指定時間的 job 後會自動重新啟動，釋放可能累積的記憶體：

```shell
# Process jobs for one hour and then exit...
php artisan queue:work --max-time=3600
```


<a name="worker-sleep-duration"></a>
#### Worker 睡眠持續時間

當佇列中有 jobs 可用時，worker 會不間斷地持續處理 jobs。不過，`sleep` 選項會決定如果沒有可用的 jobs 時，worker 將「睡眠」多少秒。當然，在睡眠期間，worker 將不會處理任何新的 jobs：

```shell
php artisan queue:work --sleep=3
```


<a name="maintenance-mode-queues"></a>
#### 維護模式與佇列

當您的應用程式處於 [維護模式](/docs/{{version}}/configuration#maintenance-mode) 時，將不會處理任何佇列中的 jobs。一旦應用程式脫離維護模式，jobs 將會照常被處理。

若要強制您的佇列 worker 在啟用維護模式時仍處理 jobs，您可以使用 `--force` 選項：

```shell
php artisan queue:work --force
```


<a name="resource-considerations"></a>
#### 資源考量

Daemon 佇列 worker 在處理每個 job 之前不會「重啟」框架。因此，您應該在每個 job 完成後釋放任何佔用大量資源。例如，如果您正在使用 [GD 函式庫](https://www.php.net/manual/en/book.image.php) 進行圖像處理，您應該在處理完圖像後使用 `imagedestroy` 釋放記憶體。


<a name="queue-priorities"></a>
### 佇列優先級

有時您可能希望對佇列的處理方式進行優先級排序。例如，在您的 `config/queue.php` 配置檔中，您可以將 `redis` 連線的預設 `queue` 設定為 `low`。不過，有時您可能希望將 job 推送到 `high` 優先級佇列，如下所示：

```php
dispatch((new Job)->onQueue('high'));
```

若要啟動一個 worker，確保所有 `high` 佇列中的 jobs 都處理完畢後才繼續處理 `low` 佇列中的任何 jobs，請將逗號分隔的佇列名稱列表傳遞給 `work` 命令：

```shell
php artisan queue:work --queue=high,low
```


<a name="queue-workers-and-deployment"></a>
### 佇列 Worker 與部署

由於佇列 worker 是長時間運行的程序，若不重新啟動，它們將不會注意到程式碼的變更。因此，使用佇列 worker 部署應用程式最簡單的方法是在部署過程中重新啟動 worker。您可以透過發出 `queue:restart` 命令來優雅地重新啟動所有 worker：

```shell
php artisan queue:restart
```

此命令將指示所有佇列 worker 在完成處理當前 job 後優雅地退出，以確保不會遺失任何現有 jobs。由於佇列 worker 在執行 `queue:restart` 命令時會退出，您應該運行一個程序管理器，例如 [Supervisor](#supervisor-configuration)，來自動重新啟動佇列 worker。

> [!NOTE]
> 佇列使用 [快取](/docs/{{version}}/cache) 來儲存重新啟動訊號，因此在使用此功能之前，您應該驗證您的應用程式是否已正確配置快取驅動器。


<a name="job-expirations-and-timeouts"></a>
### Job 過期與逾時


<a name="job-expiration"></a>
#### Job 過期

在您的 `config/queue.php` 配置檔中，每個佇列連線都定義了一個 `retry_after` 選項。此選項指定佇列連線在重試正在處理的 job 之前應該等待多少秒。例如，如果 `retry_after` 的值設定為 `90`，如果 job 已經處理了 90 秒但沒有被釋放或刪除，它將被釋放回佇列中。通常，您應該將 `retry_after` 值設定為您的 jobs 合理完成處理所需的最大秒數。

> [!WARNING]
> 唯一不包含 `retry_after` 值的佇列連線是 Amazon SQS。SQS 將根據在 AWS 控制台中管理的 [Default Visibility Timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html) 重試 job。


<a name="worker-timeouts"></a>
#### Worker 逾時

`queue:work` Artisan 命令公開了一個 `--timeout` 選項。預設情況下，`--timeout` 值為 60 秒。如果 job 處理時間超過逾時值所指定的秒數，處理該 job 的 worker 將會以錯誤退出。通常，worker 會由在您伺服器上配置的 [程序管理器](#supervisor-configuration) 自動重新啟動：

```shell
php artisan queue:work --timeout=60
```

`retry_after` 配置選項和 `--timeout` CLI 選項是不同的，但它們協同工作以確保 jobs 不會遺失，並且 jobs 只會成功處理一次。

> [!WARNING]
> `--timeout` 值應始終比 `retry_after` 配置值短至少幾秒。這將確保處理凍結 job 的 worker 在 job 重試之前總是終止。如果您的 `--timeout` 選項長於 `retry_after` 配置值，您的 jobs 可能會被處理兩次。


<a name="supervisor-configuration"></a>
## Supervisor 配置

在生產環境中，您需要一種方法來保持 `queue:work` 程序運行。`queue:work` 程序可能會因各種原因停止運行，例如 worker 逾時超出限制或執行 `queue:restart` 命令。

因此，您需要配置一個程序監控器，它可以檢測 `queue:work` 程序何時退出並自動重新啟動它們。此外，程序監控器可以讓您指定要同時運行多少個 `queue:work` 程序。Supervisor 是一個在 Linux 環境中常用的程序監控器，我們將在接下來的文件中討論如何配置它。


<a name="installing-supervisor"></a>
#### 安裝 Supervisor

Supervisor 是一個適用於 Linux 作業系統的程序監控器，如果 `queue:work` 程序失敗，它將自動重新啟動。若要在 Ubuntu 上安裝 Supervisor，您可以使用以下命令：

```shell
sudo apt-get install supervisor
```

> [!NOTE]
> 如果自行配置和管理 Supervisor 聽起來令人卻步，請考慮使用 [Laravel Cloud](https://cloud.laravel.com)，它提供了一個完全託管的平台來運行 Laravel 佇列 worker。


<a name="configuring-supervisor"></a>
#### 配置 Supervisor

Supervisor 配置檔通常儲存在 `/etc/supervisor/conf.d` 目錄中。在此目錄中，您可以建立任意數量的配置檔，指示 Supervisor 如何監控您的程序。例如，讓我們建立一個 `laravel-worker.conf` 檔，它啟動並監控 `queue:work` 程序：

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /home/forge/app.com/artisan queue:work sqs --sleep=3 --tries=3 --max-time=3600
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

在此範例中，`numprocs` 指令將指示 Supervisor 運行八個 `queue:work` 程序並監控所有這些程序，如果它們失敗，則自動重新啟動。您應該更改配置中的 `command` 指令，以反映您所需的佇列連線和 worker 選項。

> [!WARNING]
> 您應該確保 `stopwaitsecs` 的值大於您運行時間最長的 job 所消耗的秒數。否則，Supervisor 可能會在 job 完成處理之前將其終止。


<a name="starting-supervisor"></a>
#### 啟動 Supervisor

配置檔建立後，您可以使用以下命令更新 Supervisor 配置並啟動程序：

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start "laravel-worker:*"
```

有關 Supervisor 的更多資訊，請查閱 [Supervisor 文件](http://supervisord.org/index.html)。

<a name="dealing-with-failed-jobs"></a>
## 處理失敗的 Jobs

有時您的佇列 Job 會失敗。別擔心，事情不總是按計畫進行！Laravel 提供了一個便捷的方法來[指定 Job 最大嘗試次數](#max-job-attempts-and-timeout)。當非同步 Job 超過此嘗試次數後，它將會被插入到 `failed_jobs` 資料庫 table 中。[同步分派的 Jobs](/docs/{{version}}/queues#synchronous-dispatching) 失敗時不會儲存在此 table 中，其例外狀況會立即由應用程式處理。

建立 `failed_jobs` table 的 Migration 通常已存在於新的 Laravel 應用程式中。然而，如果您的應用程式不包含此 table 的 Migration，您可以使用 `make:queue-failed-table` 命令來建立 Migration：

```shell
php artisan make:queue-failed-table

php artisan migrate
```

執行[佇列 Worker](#running-the-queue-worker) 程序時，您可以使用 `queue:work` 命令上的 `--tries` 開關來指定 Job 應嘗試的最大次數。如果您沒有為 `--tries` 選項指定值，Job 只會嘗試一次，或者依照 Job class 的 `$tries` 屬性所指定的次數：

```shell
php artisan queue:work redis --tries=3
```

使用 `--backoff` 選項，您可以指定 Laravel 應等待多少秒，然後再重試遇到例外狀況的 Job。預設情況下，Job 會立即釋放回佇列中，以便可以再次嘗試：

```shell
php artisan queue:work redis --tries=3 --backoff=3
```

如果您想設定 Laravel 應等待多少秒，以便在每個 Job 遇到例外狀況後重試，您可以在您的 Job class 上定義一個 `backoff` 屬性來實現：

```php
/**
 * The number of seconds to wait before retrying the job.
 *
 * @var int
 */
public $backoff = 3;
```

如果您需要更複雜的邏輯來決定 Job 的退避時間，您可以在您的 Job class 上定義一個 `backoff` 方法：

```php
/**
 * Calculate the number of seconds to wait before retrying the job.
 */
public function backoff(): int
{
    return 3;
}
```

您可以透過從 `backoff` 方法傳回退避值陣列，輕鬆地配置「指數型」退避。在此範例中，第一次重試的延遲將為 1 秒，第二次重試為 5 秒，第三次重試為 10 秒，如果還有更多嘗試次數，則之後每次重試都為 10 秒：

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
### 清理失敗 Jobs 後

當某個 Job 失敗時，您可能想向使用者發送警示，或復原 Job 部分完成的任何操作。為了實現這一點，您可以在 Job class 上定義一個 `failed` 方法。導致 Job 失敗的 `Throwable` 實例將會傳遞給 `failed` 方法：

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
> 在呼叫 `failed` 方法之前，會實例化一個新的 Job 實例；因此，在 `handle` 方法中可能發生的任何 class 屬性修改都將遺失。

失敗的 Job 不一定是遇到未處理的例外狀況。Job 也可能在耗盡所有允許的嘗試次數時被視為失敗。這些嘗試次數可以透過幾種方式消耗：

<div class="content-list" markdown="1">

- Job 逾時。
- Job 在執行過程中遇到未處理的例外狀況。
- Job 被手動或透過中介層釋放回佇列中。

</div>

如果最終嘗試因 Job 執行期間拋出的例外狀況而失敗，該例外狀況將會傳遞給 Job 的 `failed` 方法。然而，如果 Job 因達到最大允許嘗試次數而失敗，則 `$exception` 將會是 `Illuminate\Queue\MaxAttemptsExceededException` 的實例。同樣地，如果 Job 因超出設定的逾時時間而失敗，則 `$exception` 將會是 `Illuminate\Queue\TimeoutExceededException` 的實例。

<a name="retrying-failed-jobs"></a>
### 重試失敗的 Jobs

要查看已插入到 `failed_jobs` 資料庫 table 中的所有失敗 Job，您可以使用 `queue:failed` Artisan 命令：

```shell
php artisan queue:failed
```

`queue:failed` 命令將列出 Job ID、連線、佇列、失敗時間以及其他關於 Job 的資訊。Job ID 可用於重試失敗的 Job。例如，要重試 ID 為 `ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece` 的失敗 Job，請執行以下命令：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece
```

如果需要，您可以向命令傳遞多個 ID：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece 91401d2c-0784-4f43-824c-34f94a33c24d
```

您也可以重試特定佇列的所有失敗 Job：

```shell
php artisan queue:retry --queue=name
```

要重試所有失敗的 Job，請執行 `queue:retry` 命令並將 `all` 作為 ID 傳遞：

```shell
php artisan queue:retry all
```

如果您想刪除失敗的 Job，您可以使用 `queue:forget` 命令：

```shell
php artisan queue:forget 91401d2c-0784-4f43-824c-34f94a33c24d
```

> [!NOTE]
> 當使用 [Horizon](/docs/{{version}}/horizon) 時，您應該使用 `horizon:forget` 命令來刪除失敗的 Job，而不是 `queue:forget` 命令。

要從 `failed_jobs` table 中刪除所有失敗的 Job，您可以使用 `queue:flush` 命令：

```shell
php artisan queue:flush
```

`queue:flush` 命令會從您的佇列中移除所有失敗的 Job 記錄，無論失敗 Job 的歷史有多長。您可以使用 `--hours` 選項，僅刪除在特定小時數前或更早失敗的 jobs：

```shell
php artisan queue:flush --hours=48
```

<a name="ignoring-missing-models"></a>
### 忽略遺失的 Model

當將一個 Eloquent model 注入到 Job 中時，該 model 會在被放入佇列之前自動序列化，並在 Job 處理時從資料庫中重新取得。然而，如果該 model 在 Job 等待 Worker 處理期間已被刪除，您的 Job 可能會因為 `ModelNotFoundException` 而失敗。

為了方便，您可以選擇自動刪除含有遺失 model 的 Job，透過將 Job 的 `deleteWhenMissingModels` 屬性設定為 `true`。當此屬性設定為 `true` 時，Laravel 將會靜默地丟棄該 Job，而不會拋出例外狀況：

```php
/**
 * Delete the job if its models no longer exist.
 *
 * @var bool
 */
public $deleteWhenMissingModels = true;
```

<a name="pruning-failed-jobs"></a>
### 清理失敗 Jobs

您可以透過呼叫 `queue:prune-failed` Artisan 命令來清理應用程式 `failed_jobs` table 中的記錄：

```shell
php artisan queue:prune-failed
```

預設情況下，所有超過 24 小時的失敗 Job 記錄都將被清理。如果您向命令提供 `--hours` 選項，將只保留在過去 N 小時內插入的失敗 Job 記錄。例如，以下命令將刪除所有在 48 小時前插入的失敗 Job 記錄：

```shell
php artisan queue:prune-failed --hours=48
```

<a name="storing-failed-jobs-in-dynamodb"></a>
### 將失敗 Jobs 儲存至 DynamoDB

Laravel 還支援將您的失敗 Job 記錄儲存在 [DynamoDB](https://aws.amazon.com/dynamodb) 中，而不是關係型資料庫 table。然而，您必須手動建立一個 DynamoDB table 來儲存所有失敗的 Job 記錄。通常，這個 table 應該命名為 `failed_jobs`，但您應該根據應用程式 `queue` 設定檔中 `queue.failed.table` 設定值來命名 table。

`failed_jobs` table 應該有一個名為 `application` 的字串主要 partition key 和一個名為 `uuid` 的字串主要 sort key。Key 的 `application` 部分將包含您的應用程式名稱，如應用程式 `app` 設定檔中 `name` 設定值所定義。由於應用程式名稱是 DynamoDB table key 的一部分，您可以使用相同的 table 來儲存多個 Laravel 應用程式的失敗 Job。

此外，請確保您安裝 AWS SDK，以便您的 Laravel 應用程式可以與 Amazon DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

接下來，將 `queue.failed.driver` 設定選項的值設定為 `dynamodb`。此外，您應該在失敗 Job 設定陣列中定義 `key`、`secret` 和 `region` 設定選項。這些選項將用於向 AWS 進行身份驗證。當使用 `dynamodb` 驅動器時，`queue.failed.database` 設定選項是不必要的：

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
### 禁用失敗 Job 儲存

您可以指示 Laravel 丟棄失敗的 Job 而不儲存它們，透過將 `queue.failed.driver` 設定選項的值設定為 `null`。通常，這可以透過 `QUEUE_FAILED_DRIVER` 環境變數來實現：

```ini
QUEUE_FAILED_DRIVER=null
```

<a name="failed-job-events"></a>
### 失敗 Job 事件

如果您想註冊一個在 Job 失敗時會被呼叫的事件 Listener，您可以使用 `Queue` Facade 的 `failing` 方法。例如，我們可以從 Laravel 隨附的 `AppServiceProvider` 的 `boot` 方法中，將一個閉包附加到此事件：

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
## 從佇列中清除 Jobs

> [!NOTE]
> 當使用 [Horizon](/docs/{{version}}/horizon) 時，您應該使用 `horizon:clear` 命令來清除佇列中的 jobs，而不是 `queue:clear` 命令。

如果您想刪除預設連線中預設佇列的所有 jobs，您可以使用 `queue:clear` Artisan 命令來執行此操作：

```shell
php artisan queue:clear
```

您也可以提供 `connection` 參數和 `queue` 選項，以從特定的連線和佇列中刪除 jobs：

```shell
php artisan queue:clear redis --queue=emails
```

> [!WARNING]
> 從佇列中清除 jobs 僅適用於 SQS、Redis 和 database 佇列驅動器。此外，SQS 訊息刪除過程最長需要 60 秒，因此在您清除佇列後 60 秒內發送到 SQS 佇列的 jobs 也可能會被刪除。

<a name="monitoring-your-queues"></a>
## 監控您的佇列

如果您的佇列突然湧入大量 jobs，它可能會不堪負荷，導致 jobs 完成的等待時間過長。如果您願意，Laravel 可以在您的佇列 job 數量超過指定閾值時向您發出警報。

首先，您應該將 `queue:monitor` 命令排程為 [每分鐘執行一次](/docs/{{version}}/scheduling)。該命令接受您希望監控的佇列名稱以及您期望的 job 數量閾值：

```shell
php artisan queue:monitor redis:default,redis:deployments --max=100
```

僅僅排程此命令不足以觸發通知，提醒您佇列已不堪負荷的狀態。當命令遇到 job 數量超過閾值的佇列時，將會分派一個 `Illuminate\Queue\Events\QueueBusy` 事件。您可以在應用程式的 `AppServiceProvider` 中監聽此事件，以便向您或您的開發團隊傳送通知：

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

當您測試分派 Jobs 的程式碼時，您可能希望指示 Laravel 不要實際執行 Job 本身，因為 Job 的程式碼可以直接測試，並與分派它的程式碼分開。當然，要測試 Job 本身，您可以在測試中直接實例化一個 Job 實例並呼叫 `handle` 方法。

您可以使用 `Queue` facade 的 `fake` 方法來防止佇列化的 Jobs 實際被推送到佇列中。呼叫 `Queue` facade 的 `fake` 方法後，您可以斷言應用程式嘗試將 Jobs 推送到佇列中：

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

    // Assert a job was pushed twice...
    Queue::assertPushed(ShipOrder::class, 2);

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

        // Assert a job was pushed twice...
        Queue::assertPushed(ShipOrder::class, 2);

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

您可以將閉包傳遞給 `assertPushed`、`assertNotPushed`、`assertClosurePushed` 或 `assertClosureNotPushed` 方法，以斷言有 Job 被推送到佇列中並通過了給定的「真實性測試」。如果至少有一個 Job 被推送到佇列中並通過了給定的真實性測試，則該斷言將會成功：

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
### 偽造部分 Jobs

如果您只需要偽造特定的 Jobs，同時允許其他 Jobs 正常執行，您可以將需要偽造的 Jobs 的 class 名稱傳遞給 `fake` 方法：

```php tab=Pest
test('orders can be shipped', function () {
    Queue::fake([
        ShipOrder::class,
    ]);

    // Perform order shipping...

    // Assert a job was pushed twice...
    Queue::assertPushed(ShipOrder::class, 2);
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
    Queue::assertPushed(ShipOrder::class, 2);
}
```

您可以使用 `except` 方法偽造所有 Jobs，除了指定的一組 Jobs：

```php
Queue::fake()->except([
    ShipOrder::class,
]);
```


<a name="testing-job-chains"></a>
### 測試 Job 鏈接

要測試 Job 鏈接，您需要利用 `Bus` facade 的偽造功能。`Bus` facade 的 `assertChained` 方法可用於斷言 Job 鏈接已被分派。`assertChained` 方法接受一個 Job 鏈接的陣列作為其第一個參數：

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

如您在上面的範例中看到的，Job 鏈接的陣列可以是 Job 的 class 名稱陣列。然而，您也可以提供一個實際 Job 實例的陣列。當這樣做時，Laravel 將確保這些 Job 實例與您的應用程式分派的鏈接 Job 具有相同的 class 和相同的屬性值：

```php
Bus::assertChained([
    new ShipOrder,
    new RecordShipment,
    new UpdateInventory,
]);
```

您可以使用 `assertDispatchedWithoutChain` 方法斷言 Job 在沒有 Job 鏈接的情況下被推送：

```php
Bus::assertDispatchedWithoutChain(ShipOrder::class);
```


<a name="testing-chain-modifications"></a>
#### 測試鏈接修改

如果一個鏈接 Job 在現有鏈接前置或後置 Jobs，您可以使用 Job 的 `assertHasChain` 方法來斷言 Job 具有預期的剩餘 Job 鏈接：

```php
$job = new ProcessPodcast;

$job->handle();

$job->assertHasChain([
    new TranscribePodcast,
    new OptimizePodcast,
    new ReleasePodcast,
]);
```

`assertDoesntHaveChain` 方法可用於斷言 Job 的剩餘鏈接為空：

```php
$job->assertDoesntHaveChain();
```


<a name="testing-chained-batches"></a>
#### 測試鏈接批次

如果您的 Job 鏈接包含一個批次的 Jobs，您可以在您的鏈接斷言中插入一個 `Bus::chainedBatch` 定義來斷言鏈接的批次符合您的預期：

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

`Bus` facade 的 `assertBatched` 方法可用於斷言一個 Jobs 批次已被分派。傳遞給 `assertBatched` 方法的閉包會收到一個 `Illuminate\Bus\PendingBatch` 實例，該實例可用於檢查批次中的 Jobs：

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

您可以使用 `assertBatchCount` 方法來斷言已分派了給定數量的批次：

```php
Bus::assertBatchCount(3);
```

您可以使用 `assertNothingBatched` 來斷言沒有分派任何批次：

```php
Bus::assertNothingBatched();
```


<a name="testing-job-batch-interaction"></a>
#### 測試 Job / 批次互動

此外，您可能偶爾需要測試單個 Job 與其底層批次的互動。例如，您可能需要測試 Job 是否取消了其批次的後續處理。為此，您需要透過 `withFakeBatch` 方法為 Job 分配一個偽造的批次。`withFakeBatch` 方法會回傳一個包含 Job 實例和偽造批次的 tuple：

```php
[$job, $batch] = (new ShipOrder)->withFakeBatch();

$job->handle();

$this->assertTrue($batch->cancelled());
$this->assertEmpty($batch->added);
```


<a name="testing-job-queue-interactions"></a>
### 測試 Job / 佇列互動

有時，您可能需要測試佇列化的 Job 是否將自身釋放回佇列。或者，您可能需要測試 Job 是否已自行刪除。您可以透過實例化 Job 並呼叫 `withFakeQueueInteractions` 方法來測試這些佇列互動。

一旦 Job 的佇列互動被偽造，您就可以呼叫 Job 上的 `handle` 方法。呼叫 Job 後，各種斷言方法可用來驗證 Job 的佇列互動：

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

透過在 `Queue` [Facade](/docs/{{version}}/facades) 上使用 `before` 和 `after` 方法，您可以指定在佇列化 Job 處理之前或之後執行的回呼。這些回呼是執行額外日誌記錄或增加儀表板統計資料的絕佳機會。通常，您應該在 [Service Provider](/docs/{{version}}/providers) 的 `boot` 方法中呼叫這些方法。例如，我們可以使用 Laravel 內建的 `AppServiceProvider`：

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

透過在 `Queue` [Facade](/docs/{{version}}/facades) 上使用 `looping` 方法，您可以指定在 Worker 嘗試從佇列中取得 Job 之前執行的回呼。例如，您可以註冊一個閉包來回溯任何由先前失敗的 Job 所遺留的開放交易：

```php
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Queue;

Queue::looping(function () {
    while (DB::transactionLevel() > 0) {
        DB::rollBack();
    }
});
```