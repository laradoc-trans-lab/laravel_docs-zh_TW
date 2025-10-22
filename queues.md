# 佇列

- [簡介](#introduction)
    - [連線 vs. 佇列](#connections-vs-queues)
    - [驅動器注意事項與必要條件](#driver-prerequisites)
- [建立 Job](#creating-jobs)
    - [產生 Job Class](#generating-job-classes)
    - [Class 結構](#class-structure)
    - [唯一 Job](#unique-jobs)
    - [加密 Job](#encrypted-jobs)
- [Job 中介層](#job-middleware)
    - [頻率限制](#rate-limiting)
    - [避免 Job 重疊](#preventing-job-overlaps)
    - [例外節流](#throttling-exceptions)
    - [跳過 Job](#skipping-jobs)
- [分派 Job](#dispatching-jobs)
    - [延遲分派](#delayed-dispatching)
    - [同步分派](#synchronous-dispatching)
    - [Job 與資料庫交易](#jobs-and-database-transactions)
    - [Job 鏈](#job-chaining)
    - [自訂佇列與連線](#customizing-the-queue-and-connection)
    - [指定 Job 最大嘗試次數 / 逾時值](#max-job-attempts-and-timeout)
    - [SQS FIFO 與公平佇列](#sqs-fifo-and-fair-queues)
    - [佇列容錯移轉](#queue-failover)
    - [錯誤處理](#error-handling)
- [Job 批次處理](#job-batching)
    - [定義可批次處理的 Job](#defining-batchable-jobs)
    - [分派批次](#dispatching-batches)
    - [鏈與批次](#chains-and-batches)
    - [將 Job 加入批次](#adding-jobs-to-batches)
    - [檢查批次](#inspecting-batches)
    - [取消批次](#cancelling-batches)
    - [批次失敗](#batch-failures)
    - [清除批次](#pruning-batches)
    - [將批次儲存至 DynamoDB](#storing-batches-in-dynamodb)
- [閉包排入佇列](#queueing-closures)
- [執行佇列 Worker](#running-the-queue-worker)
    - [`queue:work` 指令](#the-queue-work-command)
    - [佇列優先權](#queue-priorities)
    - [佇列 Worker 與部署](#queue-workers-and-deployment)
    - [Job 過期與逾時](#job-expirations-and-timeouts)
- [Supervisor 設定](#supervisor-configuration)
- [處理失敗的 Job](#dealing-with-failed-jobs)
    - [清理失敗的 Job](#cleaning-up-after-failed-jobs)
    - [重試失敗的 Job](#retrying-failed-jobs)
    - [忽略遺失的 Model](#ignoring-missing-models)
    - [清除失敗的 Job](#pruning-failed-jobs)
    - [將失敗的 Job 儲存至 DynamoDB](#storing-failed-jobs-in-dynamodb)
    - [停用失敗 Job 儲存](#disabling-failed-job-storage)
    - [失敗 Job 事件](#failed-job-events)
- [從佇列清除 Job](#clearing-jobs-from-queues)
- [監控佇列](#monitoring-your-queues)
- [測試](#testing)
    - [模擬 Job 子集](#faking-a-subset-of-jobs)
    - [測試 Job 鏈](#testing-job-chains)
    - [測試 Job 批次](#testing-job-batches)
    - [測試 Job / 佇列互動](#testing-job-queue-interactions)
- [Job 事件](#job-events)

<a name="introduction"></a>
## 簡介

在建構您的 Web 應用程式時，某些任務 (例如解析並儲存上傳的 CSV 檔案) 可能需要太長時間才能在典型的 Web 請求期間執行。幸運的是，Laravel 讓您可以輕鬆建立可在背景處理的排入佇列的 Job。透過將耗時的任務移至佇列，您的應用程式可以極快的速度回應 Web 請求，並為客戶提供更好的使用者體驗。

Laravel 佇列為各種不同的佇列後端提供統一的佇列 API，例如 [Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 甚至是關聯式資料庫。

Laravel 的佇列設定選項儲存在您應用程式的 `config/queue.php` 設定檔中。在此檔案中，您會找到框架中包含的每個佇列驅動器的連線設定，包括資料庫、[Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 和 [Beanstalkd](https://beanstalkd.github.io/) 驅動器，以及一個會立即執行 Job 的同步驅動器 (用於開發或測試期間)。還包含一個 `null` 佇列驅動器，它會丟棄排入佇列的 Job。

> [!NOTE]
> Laravel Horizon 是一個美觀的儀表板和設定系統，適用於您的 Redis 驅動佇列。請參閱完整的 [Horizon 說明文件](/docs/{{version}}/horizon) 以獲取更多資訊。

<a name="connections-vs-queues"></a>
### 連線 vs. 佇列

在開始使用 Laravel 佇列之前，了解「連線」和「佇列」之間的區別很重要。在您的 `config/queue.php` 設定檔中，有一個 `connections` 設定陣列。此選項定義了與後端佇列服務 (例如 Amazon SQS、Beanstalk 或 Redis) 的連線。但是，任何給定的佇列連線都可能有多個「佇列」，這些佇列可以被視為不同的堆疊或一堆排入佇列的 Job。

請注意，`queue` 設定檔中的每個連線設定範例都包含一個 `queue` 屬性。這是當 Job 被傳送到給定連線時將被分派到的預設佇列。換句話說，如果您分派一個 Job 而沒有明確定義它應該被分派到哪個佇列，那麼該 Job 將被放置在連線設定的 `queue` 屬性中定義的佇列上：

```php
use App\Jobs\ProcessPodcast;

// This job is sent to the default connection's default queue...
ProcessPodcast::dispatch();

// This job is sent to the default connection's "emails" queue...
ProcessPodcast::dispatch()->onQueue('emails');
```

有些應用程式可能不需要將 Job 推送到多個佇列，而是偏好只有一個簡單的佇列。但是，將 Job 推送到多個佇列對於希望優先處理或區分 Job 處理方式的應用程式特別有用，因為 Laravel 佇列 Worker 允許您指定它應該按優先順序處理哪些佇列。例如，如果您將 Job 推送到 `high` 佇列，您可以執行一個 Worker 來賦予它們更高的處理優先順序：

```shell
php artisan queue:work --queue=high,default
```

<a name="driver-prerequisites"></a>
### 驅動器注意事項與必要條件

<a name="database"></a>
#### 資料庫

為了使用 `database` 佇列驅動器，您將需要一個資料庫表來存放 Job。通常，這包含在 Laravel 的預設 `0001_01_01_000002_create_jobs_table.php` [資料庫遷移](/docs/{{version}}/migrations) 中；但是，如果您的應用程式不包含此遷移，您可以使用 `make:queue-table` Artisan 指令來建立它：

```shell
php artisan make:queue-table

php artisan migrate
```

<a name="redis"></a>
#### Redis

為了使用 `redis` 佇列驅動器，您應該在 `config/database.php` 設定檔中設定 Redis 資料庫連線。

> [!WARNING]
> `redis` 佇列驅動器不支援 `serializer` 和 `compression` Redis 選項。

<a name="redis-cluster"></a>
##### Redis Cluster

如果您的 Redis 佇列連線使用 [Redis Cluster](https://redis.io/docs/latest/operate/rs/databases/durability-ha/clustering)，您的佇列名稱必須包含 [鍵值雜湊標籤](https://redis.io/docs/latest/develop/using-commands/keyspace/#hashtags)。這是為了確保給定佇列的所有 Redis 鍵都放置在相同的雜湊槽中：

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
##### 阻擋

使用 Redis 佇列時，您可以使用 `block_for` 設定選項來指定驅動器應該等待 Job 可用多久，然後再透過 Worker 迴圈迭代並重新輪詢 Redis 資料庫。

根據您的佇列負載調整此值比持續輪詢 Redis 資料庫以獲取新 Job 更有效率。例如，您可以將值設定為 `5`，表示驅動器在等待 Job 可用時應該阻擋五秒：

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
> 將 `block_for` 設定為 `0` 將導致佇列 Worker 無限期地阻擋，直到有 Job 可用為止。這也會阻止處理 `SIGTERM` 等訊號，直到下一個 Job 被處理。

<a name="other-driver-prerequisites"></a>
#### 其他驅動器必要條件

以下是列出的佇列驅動器所需的依賴項。這些依賴項可以透過 Composer 套件管理器安裝：

<div class="content-list" markdown="1">

- Amazon SQS: `aws/aws-sdk-php ~3.0`
- Beanstalkd: `pda/pheanstalk ~5.0`
- Redis: `predis/predis ~2.0` 或 phpredis PHP 擴充功能
- [MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/queues/): `mongodb/laravel-mongodb`

</div>

<a name="creating-jobs"></a>
## 建立 Job


<a name="generating-job-classes"></a>
### 產生 Job Class

預設情況下，您應用程式中所有可排入佇列的 Job 都儲存在 `app/Jobs` 目錄中。如果 `app/Jobs` 目錄不存在，執行 `make:job` Artisan 指令時會自動建立：

```shell
php artisan make:job ProcessPodcast
```

產生的 Class 會實作 `Illuminate\Contracts\Queue\ShouldQueue` 介面，表示該 Job 應被推送到佇列中非同步執行。

> [!NOTE]
> Job 樣板可以使用 [樣板發布](/docs/{{version}}/artisan#stub-customization) 功能進行自訂。


<a name="class-structure"></a>
### Class 結構

Job Class 非常簡單，通常只包含一個 `handle` 方法，該方法會在 Job 被佇列處理時被呼叫。首先，讓我們看看一個範例 Job Class。在這個範例中，我們假設我們管理一個 Podcast 發布服務，需要在發布 Podcast 檔案之前對其進行處理：

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

在這個範例中，請注意我們可以將一個 [Eloquent 模型](/docs/{{version}}/eloquent) 直接傳遞給佇列 Job 的建構子。由於 Job 使用了 `Queueable` Trait，Eloquent 模型及其載入的關聯會在 Job 處理時優雅地進行序列化和反序列化。

如果您的佇列 Job 在建構子中接受一個 Eloquent 模型，只有該模型的識別碼會被序列化到佇列中。當 Job 實際被處理時，佇列系統會自動從資料庫中重新取得完整的模型實例及其載入的關聯。這種模型序列化方法允許將更小的 Job 負載傳送到您的佇列驅動器。


<a name="handle-method-dependency-injection"></a>
#### `handle` 方法的依賴注入

當 Job 被佇列處理時，會呼叫 `handle` 方法。請注意，我們可以在 Job 的 `handle` 方法上進行型別提示依賴。Laravel [服務容器](/docs/{{version}}/container) 會自動注入這些依賴。

如果您想完全控制容器如何將依賴注入到 `handle` 方法中，您可以使用容器的 `bindMethod` 方法。`bindMethod` 方法接受一個回呼，該回呼會接收 Job 和容器。在回呼中，您可以自由地以您希望的任何方式呼叫 `handle` 方法。通常，您應該從 `App\Providers\AppServiceProvider` [服務供應者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫此方法：

```php
use App\Jobs\ProcessPodcast;
use App\Services\AudioProcessor;
use Illuminate\Contracts\Foundation\Application;

$this->app->bindMethod([ProcessPodcast::class, 'handle'], function (ProcessPodcast $job, Application $app) {
    return $job->handle($app->make(AudioProcessor::class));
});
```

> [!WARNING]
> 二進位資料 (例如原始圖片內容) 在傳遞給佇列 Job 之前，應透過 `base64_encode` 函數進行處理。否則，Job 在被放入佇列時可能無法正確序列化為 JSON。


<a name="handling-relationships"></a>
#### 佇列關係

由於所有已載入的 Eloquent 模型關聯在 Job 排入佇列時也會被序列化，因此序列化的 Job 字串有時會變得非常大。此外，當 Job 被反序列化並從資料庫中重新取得模型關聯時，它們將會被完整地取得。在模型序列化排入 Job 佇列之前應用的任何先前關聯約束，在 Job 反序列化時將不會被應用。因此，如果您希望處理給定關聯的子集，則應在佇列 Job 中重新約束該關聯。

或者，為了防止關聯被序列化，您可以在設定屬性值時呼叫模型上的 `withoutRelations` 方法。此方法將返回一個沒有已載入關聯的模型實例：

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

如果您正在使用 [PHP 建構子屬性提升](https://www.php.net/manual/en/language.oop5.decon.php#language.oop5.decon.constructor.promotion)，並希望表明 Eloquent 模型不應序列化其關聯，您可以使用 `WithoutRelations` 屬性：

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

為方便起見，如果您希望序列化所有不帶關聯的模型，您可以將 `WithoutRelations` 屬性應用於整個 Class，而不是將屬性應用於每個模型：

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

如果 Job 接收到的是 Eloquent 模型集合或陣列而非單一模型，則該集合中的模型在 Job 反序列化並執行時不會還原其關聯。這是為了避免處理大量模型時造成過度的資源使用。

<a name="unique-jobs"></a>
### 唯一 Job

> [!WARNING]
> 唯一 Job 需要支援[鎖定](/docs/{{version}}/cache#atomic-locks)的快取驅動器。目前，`memcached`、`redis`、`dynamodb`、`database`、`file` 與 `array` 快取驅動器支援原子鎖定。

> [!WARNING]
> 唯一 Job 的限制不適用於批次中的 Job。

有時，您可能希望確保佇列中在任何時間點只有一個特定 Job 的實例。您可以透過在 Job Class 上實作 `ShouldBeUnique` 介面來實現此目的。此介面不要求您在 Class 上定義任何額外方法：

```php
<?php

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Queue\ShouldBeUnique;

class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    // ...
}
```

在上面的範例中，`UpdateSearchIndex` Job 是唯一的。因此，如果佇列中已有另一個 Job 實例且尚未完成處理，則該 Job 將不會被分派。

在某些情況下，您可能希望定義一個特定的「鍵」來使 Job 唯一，或者您可能希望指定一個逾時，在此逾時之後 Job 不再保持唯一。為了實現此目的，您可以在 Job Class 上定義 `uniqueId` 和 `uniqueFor` 屬性或方法：

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

在上面的範例中，`UpdateSearchIndex` Job 透過產品 ID 保持唯一。因此，任何具有相同產品 ID 的新 Job 分派都會被忽略，直到現有 Job 完成處理為止。此外，如果現有 Job 在一小時內未被處理，則唯一鎖定將被釋放，並且另一個具有相同唯一鍵的 Job 可以分派到佇列。

> [!WARNING]
> 如果您的應用程式從多個 Web 伺服器或容器分派 Job，您應該確保所有伺服器都與同一個中央快取伺服器通訊，以便 Laravel 可以準確判斷 Job 是否唯一。

<a name="keeping-jobs-unique-until-processing-begins"></a>
#### 在 Job 開始處理前保持唯一

預設情況下，唯一 Job 會在 Job 完成處理或所有重試嘗試失敗後「解鎖」。然而，在某些情況下，您可能希望 Job 在處理之前立即解鎖。為了實現此目的，您的 Job 應該實作 `ShouldBeUniqueUntilProcessing` 契約而不是 `ShouldBeUnique` 契約：

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

在幕後，當 `ShouldBeUnique` Job 被分派時，Laravel 會嘗試使用 `uniqueId` 鍵取得[鎖定](/docs/{{version}}/cache#atomic-locks)。如果鎖定已被持有，則 Job 不會被分派。當 Job 完成處理或所有重試嘗試失敗時，此鎖定會被釋放。預設情況下，Laravel 將使用預設快取驅動器來取得此鎖定。但是，如果您希望使用另一個驅動器來取得鎖定，您可以定義一個 `uniqueVia` 方法，該方法會回傳應該使用的快取驅動器：

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
> 如果您只需要限制 Job 的併發處理，請改用 [WithoutOverlapping](/docs/{{version}}/queues#preventing-job-overlaps) Job 中介層。

<a name="encrypted-jobs"></a>
### 加密 Job

Laravel 允許您透過[加密](/docs/{{version}}/encryption)來確保 Job 資料的隱私和完整性。要開始使用，只需將 `ShouldBeEncrypted` 介面新增到 Job Class 即可。一旦此介面被新增到 Class，Laravel 就會在將您的 Job 推送到佇列之前自動加密您的 Job：

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

Job 中介層讓您可以將自訂邏輯包裝在排入佇列的 Job 執行周圍，減少 Job 本身的重複性程式碼。舉例來說，考慮以下 `handle` 方法，它利用了 Laravel 的 Redis 頻率限制功能，以允許每五秒鐘只處理一個 Job：

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

儘管這段程式碼有效，但 `handle` 方法的實作變得冗長，因為它充斥著 Redis 頻率限制邏輯。此外，對於任何其他需要頻率限制的 Job，這段頻率限制邏輯都必須重複。與其在 handle 方法中進行頻率限制，不如定義一個 Job 中介層來處理頻率限制：

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

如您所見，如同 [路由中介層](/docs/{{version}}/middleware) 一樣，Job 中介層會接收正在處理的 Job 以及一個應被呼叫以繼續處理 Job 的回呼。

您可以使用 `make:job-middleware` Artisan 指令來產生新的 Job 中介層類別。建立 Job 中介層後，可以透過從 Job 的 `middleware` 方法回傳它們來將它們附加到 Job。`make:job` Artisan 指令所產生的 Job 不包含此方法，因此您需要手動將其新增至您的 Job 類別中：

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
> Job 中介層也可以指派給 [可排入佇列的事件監聽器](/docs/{{version}}/events#queued-event-listeners)、[郵件](/docs/{{version}}/mail#queueing-mail) 和 [通知](/docs/{{version}}/notifications#queueing-notifications)。

<a name="rate-limiting"></a>
### 頻率限制

儘管我們剛才示範了如何編寫自己的頻率限制 Job 中介層，但 Laravel 實際上包含一個頻率限制中介層，您可以用來限制 Job 的頻率。與 [路由頻率限制器](/docs/{{version}}/routing#defining-rate-limiters) 類似，Job 頻率限制器是使用 `RateLimiter` 外觀的 `for` 方法來定義的。

舉例來說，您可能希望允許使用者每小時備份一次資料，但對尊榮客戶不施加此類限制。為實現此目的，您可以在 `AppServiceProvider` 的 `boot` 方法中定義一個 `RateLimiter`：

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

在上面的範例中，我們定義了每小時的頻率限制；但是，您可以使用 `perMinute` 方法輕鬆定義基於分鐘的頻率限制。此外，您可以將任何值傳遞給頻率限制的 `by` 方法；然而，這個值最常用來依客戶區分頻率限制：

```php
return Limit::perMinute(50)->by($job->user->id);
```

一旦您定義了頻率限制，您就可以使用 `Illuminate\Queue\Middleware\RateLimited` 中介層將該頻率限制附加到您的 Job。每當 Job 超出頻率限制時，此中介層將根據頻率限制持續時間，以適當的延遲將 Job 釋放回佇列：

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

將受頻率限制的 Job 釋放回佇列仍會增加 Job 的總 `attempts` 次數。您可能需要相應地調整 Job 類別上的 `tries` 和 `maxExceptions` 屬性。或者，您可能希望使用 [retryUntil 方法](#time-based-attempts) 來定義 Job 不應再嘗試的時間量。

使用 `releaseAfter` 方法，您還可以指定在釋放的 Job 再次嘗試之前必須經過的秒數：

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

如果您不希望 Job 在受到頻率限制時被重試，您可以使用 `dontRelease` 方法：

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
> 如果您正在使用 Redis，您可以使用 `Illuminate\Queue\Middleware\RateLimitedWithRedis` 中介層，它專為 Redis 調優，並且比基本的頻率限制中介層更高效。

<a name="preventing-job-overlaps"></a>
### 避免 Job 重疊

Laravel 包含了一個 `Illuminate\Queue\Middleware\WithoutOverlapping` 中介層，允許你基於任意鍵來避免 Job 重疊。當一個排入佇列的 Job 正在修改某個資源，且該資源應當在同一時間只能由一個 Job 修改時，這會非常有用。

例如，假設你有一個排入佇列的 Job，它會更新使用者的信用分數，並且你希望避免相同使用者 ID 的信用分數更新 Job 重疊。為此，你可以從 Job 的 `middleware` 方法中回傳 `WithoutOverlapping` 中介層：

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

將重疊的 Job 釋放回佇列仍會增加 Job 的總嘗試次數。你可能需要相應地調整 Job Class 上的 `tries` 和 `maxExceptions` 屬性。例如，預設情況下將 `tries` 屬性保留為 1 將會防止任何重疊的 Job 稍後被重試。

任何相同類型的重疊 Job 都會被釋放回佇列。你也可以指定在釋放的 Job 再次被嘗試之前必須經過的秒數：

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

如果你希望立即刪除任何重疊的 Job，使其不會被重試，你可以使用 `dontRelease` 方法：

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

`WithoutOverlapping` 中介層由 Laravel 的原子鎖功能提供支援。有時候，你的 Job 可能會意外失敗或逾時，導致鎖未釋放。因此，你可以使用 `expireAfter` 方法明確定義鎖的過期時間。例如，以下範例將指示 Laravel 在 Job 開始處理後三分鐘釋放 `WithoutOverlapping` 鎖：

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
> `WithoutOverlapping` 中介層需要支援[鎖](/docs/{{version}}/cache#atomic-locks)的快取驅動器。目前，`memcached`、`redis`、`dynamodb`、`database`、`file` 和 `array` 快取驅動器支援原子鎖。

<a name="sharing-lock-keys"></a>
#### 跨 Job Class 共享鎖定鍵

預設情況下，`WithoutOverlapping` 中介層只會防止相同 Class 的重疊 Job。因此，儘管兩個不同的 Job Class 可能使用相同的鎖定鍵，但它們不會被防止重疊。但是，你可以使用 `shared` 方法指示 Laravel 將該鍵應用於所有 Job Class：

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
### 例外節流

Laravel 包含了 `Illuminate\Queue\Middleware\ThrottlesExceptions` 中介層，可讓您節流例外。一旦 Job 拋出指定數量的例外，後續嘗試執行 Job 都將延遲，直到指定的時程間隔過去後。此中介層對於與不穩定的第三方服務互動的 Job 尤其有用。

例如，讓我們想像一個排入佇列的 Job，它與一個開始拋出例外的第三方 API 互動。為了節流例外，您可以從 Job 的 `middleware` 方法回傳 `ThrottlesExceptions` 中介層。通常，此中介層應與實作 [基於時間的嘗試](#time-based-attempts) 的 Job 搭配使用：

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

中介層接受的第一個建構式參數是例外被節流前，Job 可拋出的例外數量，而第二個建構式參數是在例外被節流後，Job 再次嘗試前應經過的秒數。在上述程式碼範例中，如果 Job 拋出 10 個連續例外，我們將等待 5 分鐘後再嘗試 Job，並受 30 分鐘時間限制。

當 Job 拋出例外但尚未達到例外閾值時，Job 通常會立即重試。不過，您可以透過在將中介層附加到 Job 上時呼叫 `backoff` 方法來指定此類 Job 應延遲的分鐘數：

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

在內部，此中介層使用 Laravel 的快取系統來實作頻率限制，並將 Job 的類別名稱用作快取「鍵 (Key)」。您可以透過在將中介層附加到 Job 上時呼叫 `by` 方法來覆寫此鍵。如果您有多個 Job 與同一個第三方服務互動，並且希望它們共用一個共同的節流「桶 (bucket)」，以確保它們遵守單一的共用限制，這會很有用：

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

預設情況下，此中介層將節流每個例外。您可以透過在將中介層附加到 Job 上時呼叫 `when` 方法來修改此行為。只有當提供給 `when` 方法的閉包回傳 `true` 時，例外才會被節流：

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

與 `when` 方法（它將 Job 釋回佇列或拋出例外）不同，`deleteWhen` 方法可讓您在發生指定例外時完全刪除 Job：

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

如果您希望將節流的例外報告給應用程式的例外處理器，您可以透過在將中介層附加到 Job 上時呼叫 `report` 方法來實現。您可以選擇提供一個閉包給 `report` 方法，並且只有在給定的閉包回傳 `true` 時，例外才會被報告：

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
> 如果您正在使用 Redis，您可以使用 `Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis` 中介層，它針對 Redis 進行了優化，比基本的例外節流中介層更高效。

<a name="skipping-jobs"></a>
### 跳過 Job

`Skip` 中介層允許您指定 Job 應被跳過 / 刪除，而無需修改 Job 的邏輯。如果給定的條件評估為 `true`，`Skip::when` 方法將會刪除 Job；而如果條件評估為 `false`，`Skip::unless` 方法將會刪除 Job：

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
## 分派 Job

在您編寫好 Job Class 後，您可以使用 Job 本身的 `dispatch` 方法來分派它。傳遞給 `dispatch` 方法的引數將會被傳遞給 Job 的建構子：

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

在新的 Laravel 應用程式中，`database` 驅動器是預設的佇列驅動器。您可以在應用程式的 `config/queue.php` 配置檔中指定不同的佇列驅動器。

<a name="delayed-dispatching"></a>
### 延遲分派

如果您想指定 Job 不應立即由佇列 Worker 處理，您可以在分派 Job 時使用 `delay` 方法。例如，讓我們指定 Job 在分派後 10 分鐘才可供處理：

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

在某些情況下，Job 可能已配置預設延遲。如果您需要繞過此延遲並立即處理 Job，您可以使用 `withoutDelay` 方法：

```php
ProcessPodcast::dispatch($podcast)->withoutDelay();
```

> [!WARNING]
> Amazon SQS 佇列服務的最大延遲時間為 15 分鐘。

<a name="dispatching-after-the-response-is-sent-to-browser"></a>
#### 在回應傳送至瀏覽器後分派

或者，如果您的 Web 伺服器正在使用 [FastCGI](https://www.php.net/manual/en/install.fpm.php)，`dispatchAfterResponse` 方法會延遲分派 Job，直到 HTTP 回應傳送給使用者瀏覽器之後。即使佇列中的 Job 仍在執行，這仍將允許使用者開始使用應用程式。這通常僅用於大約一秒的 Job，例如發送電子郵件。由於它們在當前 HTTP 請求中處理，以此方式分派的 Job 不需要佇列 Worker 執行即可處理：

```php
use App\Jobs\SendNotification;

SendNotification::dispatchAfterResponse();
```

您也可以 `dispatch` 一個閉包並鏈接 `afterResponse` 方法到 [dispatch 輔助函式](/docs/{{version}}/helpers#method-dispatch)上，以便在 HTTP 回應傳送給瀏覽器後執行該閉包：

```php
use App\Mail\WelcomeMessage;
use Illuminate\Support\Facades\Mail;

dispatch(function () {
    Mail::to('taylor@example.com')->send(new WelcomeMessage);
})->afterResponse();
```

<a name="synchronous-dispatching"></a>
### 同步分派

如果您想立即 (同步) 分派 Job，可以使用 `dispatchSync` 方法。使用此方法時，Job 將不會排入佇列，並會立即在當前程序中執行：

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
#### 延遲分派

使用延遲同步分派，您可以在當前程序中處理 Job，但在 HTTP 回應發送給使用者之後。這允許您同步處理「排入佇列」的 Job，而不會減慢使用者的應用程式體驗。要延遲同步 Job 的執行，請將 Job 分派到 `deferred` 連線：

```php
RecordDelivery::dispatch($order)->onConnection('deferred');
```

`deferred` 連線也作為預設的 [容錯移轉佇列](#queue-failover)。

<a name="jobs-and-database-transactions"></a>
### Job 與資料庫交易

儘管在資料庫交易中分派 Job 完全沒有問題，但您應特別注意確保 Job 實際能夠成功執行。在交易中分派 Job 時，Job 有可能在父交易提交之前就被 Worker 處理。當這種情況發生時，您在資料庫交易期間對 Model 或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何 Model 或資料庫記錄可能不存在於資料庫中。

幸運的是，Laravel 提供了幾種解決此問題的方法。首先，您可以在佇列連線的配置陣列中設定 `after_commit` 連線選項：

```php
'redis' => [
    'driver' => 'redis',
    // ...
    'after_commit' => true,
],
```

當 `after_commit` 選項為 `true` 時，您可以在資料庫交易中分派 Job；但 Laravel 會等到所有開啟的父資料庫交易都提交後，才會實際分派 Job。當然，如果目前沒有開啟的資料庫交易，Job 將會立即分派。

如果交易因交易期間發生的例外而回溯，則在該交易期間分派的 Job 將被丟棄。

> [!NOTE]
> 將 `after_commit` 配置選項設為 `true` 也會導致任何排入佇列的事件監聽器、mailable、通知和廣播事件在所有開啟的資料庫交易提交後才分派。

<a name="specifying-commit-dispatch-behavior-inline"></a>
#### 行內指定提交分派行為

如果您沒有將 `after_commit` 佇列連線配置選項設為 `true`，您仍然可以指示特定 Job 應在所有開啟的資料庫交易提交後分派。為此，您可以將 `afterCommit` 方法鏈接到您的分派操作上：

```php
use App\Jobs\ProcessPodcast;

ProcessPodcast::dispatch($podcast)->afterCommit();
```

同樣地，如果 `after_commit` 配置選項設為 `true`，您可以指示特定 Job 應立即分派，而不等待任何開啟的資料庫交易提交：

```php
ProcessPodcast::dispatch($podcast)->beforeCommit();
```

<a name="job-chaining"></a>
### Job 鏈

Job 鏈結能讓你指定一系列的佇列 Job，在主要 Job 成功執行後依序運行。如果序列中的某個 Job 失敗，其餘的 Job 將不會被執行。為了執行佇列 Job 鏈，你可以使用 `Bus` Facade 提供的 `chain` 方法。Laravel 的命令匯流排是一個底層元件，佇列 Job 的分派就是建立在其之上：

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

除了鏈結 Job 類別實例之外，你也可以鏈結閉包 (Closure)：

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
> 在 Job 內使用 `$this->delete()` 方法刪除 Job 並不會阻止鏈結 Job 被處理。鏈結只會在鏈中的 Job 失敗時停止執行。


<a name="chain-connection-queue"></a>
#### 鏈結連線與佇列

如果你想指定鏈結 Job 所使用的連線與佇列，你可以使用 `onConnection` 和 `onQueue` 方法。這些方法會指定佇列連線與佇列名稱，除非佇列 Job 被明確指定了不同的連線 / 佇列：

```php
Bus::chain([
    new ProcessPodcast,
    new OptimizePodcast,
    new ReleasePodcast,
])->onConnection('redis')->onQueue('podcasts')->dispatch();
```


<a name="adding-jobs-to-the-chain"></a>
#### 將 Job 加入鏈中

有時，你可能需要從鏈中的另一個 Job 內部，在現有的 Job 鏈前置或附加 Job。你可以使用 `prependToChain` 和 `appendToChain` 方法來完成此操作：

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

鏈結 Job 時，你可以使用 `catch` 方法來指定一個閉包，當鏈中的 Job 失敗時，該閉包應被呼叫。給定的回呼 (callback) 將會收到導致 Job 失敗的 `Throwable` 實例：

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
> 由於鏈結的回呼 (callback) 會被 Laravel 佇列序列化並在稍後執行，因此你不應該在鏈結回呼中使用 `$this` 變數。


<a name="customizing-the-queue-and-connection"></a>
### 自訂佇列與連線


<a name="dispatching-to-a-particular-queue"></a>
#### 分派到特定佇列

藉由將 Job 推送到不同的佇列，你可以「分類」你的佇列 Job，甚至優先處理你指派給各個佇列的 worker 數量。請記住，這並非將 Job 推送到你佇列設定檔中定義的不同佇列「連線」，而僅是推送到單一連線中的特定佇列。若要指定佇列，可在分派 Job 時使用 `onQueue` 方法：

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

或者，你也可以在 Job 的建構子中呼叫 `onQueue` 方法來指定 Job 的佇列：

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

如果你的應用程式與多個佇列連線互動，你可以使用 `onConnection` 方法來指定要將 Job 推送到哪個連線：

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

你可以將 `onConnection` 和 `onQueue` 方法鏈結在一起，以指定 Job 的連線和佇列：

```php
ProcessPodcast::dispatch($podcast)
    ->onConnection('sqs')
    ->onQueue('processing');
```

或者，你也可以在 Job 的建構子中呼叫 `onConnection` 方法來指定 Job 的連線：

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

Job 嘗試次數是 Laravel 佇列系統的核心概念，並驅動許多進階功能。雖然它們一開始可能令人困惑，但在修改預設設定之前，了解它們如何運作非常重要。

當 Job 被分派時，它會被推送到佇列中。然後 Worker 會接收它並嘗試執行。這就是一次 Job 嘗試。

然而，一次嘗試並不一定意味著 Job 的 `handle` 方法被執行。嘗試次數也可能以多種方式「被消耗」：

<div class="content-list" markdown="1">

- Job 在執行過程中遇到未處理的例外。
- Job 使用 `$this->release()` 手動釋放回佇列。
- `WithoutOverlapping` 或 `RateLimited` 等中介層未能取得鎖定並釋放 Job。
- Job 逾時。
- Job 的 `handle` 方法執行完成且未拋出例外。

</div>

您可能不希望無限期地嘗試一個 Job。因此，Laravel 提供了多種方式來指定一個 Job 可以嘗試多少次或嘗試多久。

> [!NOTE]
> 預設情況下，Laravel 只會嘗試 Job 一次。如果您的 Job 使用了 `WithoutOverlapping` 或 `RateLimited` 等中介層，或者您手動釋放 Job，您可能需要透過 `tries` 選項增加允許的嘗試次數。

指定 Job 最大嘗試次數的一種方法是透過 Artisan 命令列上的 `--tries` 參數。除非正在處理的 Job 指定了它可以嘗試的次數，否則這將適用於 Worker 處理的所有 Job：

```shell
php artisan queue:work --tries=3
```

如果一個 Job 超過其最大嘗試次數，它將被視為「失敗」的 Job。有關處理失敗 Job 的更多資訊，請參閱[失敗 Job 文件](#dealing-with-failed-jobs)。如果提供 `--tries=0` 給 `queue:work` 指令，該 Job 將無限期重試。

您可以透過在 Job Class 本身定義最大嘗試次數來採用更細緻的方法。如果在 Job 上指定了最大嘗試次數，它將優先於命令列上提供 `--tries` 值：

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

如果您需要對特定 Job 的最大嘗試次數進行動態控制，您可以在 Job 上定義一個 `tries` 方法：

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

除了定義 Job 在失敗前可以嘗試多少次之外，您還可以定義 Job 不應再嘗試的時間。這允許 Job 在給定時間範圍內被嘗試任意次數。要定義 Job 不應再嘗試的時間，請在 Job Class 中添加 `retryUntil` 方法。此方法應回傳一個 `DateTime` 實例：

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

如果同時定義了 `retryUntil` 和 `tries`，Laravel 將優先考慮 `retryUntil` 方法。

> [!NOTE]
> 您也可以在[佇列事件監聽器](/docs/{{version}}/events#queued-event-listeners)和[佇列通知](/docs/{{version}}/notifications#queueing-notifications)上定義 `tries` 屬性或 `retryUntil` 方法。

<a name="max-exceptions"></a>
#### 最大例外數量

有時您可能希望指定一個 Job 可以嘗試多次，但如果重試是由給定數量的未處理例外觸發（而不是直接由 `release` 方法釋放）則應失敗。為此，您可以在 Job Class 上定義一個 `maxExceptions` 屬性：

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

在此範例中，如果應用程式無法取得 Redis 鎖定，Job 將被釋放十秒，並將繼續重試最多 25 次。然而，如果 Job 拋出三個未處理的例外，Job 將失敗。

<a name="timeout"></a>
#### 逾時

通常，您大致知道您的佇列 Job 需要多長時間。為此，Laravel 允許您指定一個「逾時」值。預設情況下，逾時值為 60 秒。如果一個 Job 處理時間超過逾時值指定的秒數，處理該 Job 的 Worker 將會錯誤退出。通常，Worker 將由[您伺服器上設定的程序管理器](#supervisor-configuration)自動重新啟動。

Job 可以執行的最大秒數可以使用 Artisan 命令列上的 `--timeout` 參數來指定：

```shell
php artisan queue:work --timeout=30
```

如果 Job 因持續逾時而超過其最大嘗試次數，它將被標記為失敗。

您也可以在 Job Class 本身定義 Job 應允許執行的最大秒數。如果在 Job 上指定了逾時，它將優先於命令列上指定的任何逾時：

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

有時，IO 阻塞程序，例如 Sockets 或對外 HTTP 連線，可能不遵守您指定的逾時。因此，當使用這些功能時，您應該始終嘗試使用其 API 來指定逾時。例如，當使用 [Guzzle](https://docs.guzzlephp.org) 時，您應該始終指定連線和請求逾時值。

> [!WARNING]
> 必須安裝 [PCNTL](https://www.php.net/manual/en/book.pcntl.php) PHP 擴充功能才能指定 Job 逾時。此外，Job 的「逾時」值應始終小於其[「重試時間」](#job-expiration)值。否則，Job 可能會在實際完成執行或逾時之前再次嘗試。

<a name="failing-on-timeout"></a>
#### 逾時失敗

如果您想指示 Job 在逾時時應被標記為[失敗](#dealing-with-failed-jobs)，您可以在 Job Class 上定義 `$failOnTimeout` 屬性：

```php
/**
 * Indicate if the job should be marked as failed on timeout.
 *
 * @var bool
 */
public $failOnTimeout = true;
```

> [!NOTE]
> 預設情況下，當 Job 逾時時，它會消耗一次嘗試並被釋放回佇列（如果允許重試）。但是，如果您將 Job 設定為在逾時時失敗，則無論為 tries 設定的值為何，它都不會被重試。

<a name="sqs-fifo-and-fair-queues"></a>
### SQS FIFO 與公平佇列

Laravel 支援 [Amazon SQS FIFO (先進先出)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html) 佇列，允許你以 Job 傳送的確切順序處理它們，同時透過訊息去重複確保僅一次處理。

FIFO 佇列需要一個訊息群組 ID 來確定哪些 Job 可以平行處理。具有相同群組 ID 的 Job 會依序處理，而具有不同群組 ID 的訊息則可以同時處理。

Laravel 提供一個流暢的 `onGroup` 方法，用於在分派 Job 時指定訊息群組 ID：

```php
ProcessOrder::dispatch($order)
    ->onGroup("customer-{$order->customer_id}");
```

SQS FIFO 佇列支援訊息去重複，以確保僅一次處理。在你的 Job 類別中實作 `deduplicationId` 方法，以提供自訂去重複 ID：

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
#### FIFO 監聽器、郵件與通知

使用 FIFO 佇列時，你還需要在監聽器、郵件與通知上定義訊息群組。或者，你可以將這些物件的佇列實例分派到非 FIFO 佇列。

若要為 [佇列事件監聽器](/docs/{{version}}/events#queued-event-listeners) 定義訊息群組，請在監聽器上定義 `messageGroup` 方法。你也可以選擇性地定義 `deduplicationId` 方法：

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

當傳送要排入 FIFO 佇列的 [郵件訊息](/docs/{{version}}/mail) 時，你應該在傳送通知時呼叫 `onGroup` 方法，並可選擇性地呼叫 `withDeduplicator` 方法：

```php
use App\Mail\InvoicePaid;
use Illuminate\Support\Facades\Mail;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

Mail::to($request->user())->send($invoicePaid);
```

當傳送要排入 FIFO 佇列的 [通知](/docs/{{version}}/notifications) 時，你應該在傳送通知時呼叫 `onGroup` 方法，並可選擇性地呼叫 `withDeduplicator` 方法：

```php
use App\Notifications\InvoicePaid;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

$user->notify($invoicePaid);
```

<a name="queue-failover"></a>
### 佇列容錯移轉

`failover` 佇列驅動器在將 Job 推送到佇列時提供自動容錯移轉功能。如果主要佇列連線因任何原因失敗，Laravel 將自動嘗試將 Job 推送到清單中下一個設定好的連線。這對於確保佇列可靠性至關重要的生產環境中實現高可用性特別有用。

若要設定容錯移轉佇列連線，請指定 `failover` 驅動器並提供一個依序嘗試的連線名稱陣列。預設情況下，Laravel 在你的應用程式 `config/queue.php` 設定檔中包含一個容錯移轉範例設定：

```php
'failover' => [
    'driver' => 'failover',
    'connections' => [
        'database',
        'sync',
    ],
],
```

一旦你設定了使用 `failover` 驅動器的連線，你可能會想將容錯移轉連線設定為應用程式 `.env` 檔中的預設佇列連線：

```ini
QUEUE_CONNECTION=failover
```

當佇列連線操作失敗並啟用容錯移轉時，Laravel 將分派 `Illuminate\Queue\Events\QueueFailedOver` 事件，允許你回報或記錄佇列連線已失敗。

<a name="error-handling"></a>
### 錯誤處理

如果在 Job 處理期間拋出例外，Job 將自動重新放回佇列，以便稍後再次嘗試。Job 將持續被釋放，直到它達到應用程式允許的最大嘗試次數。最大嘗試次數由 `queue:work` Artisan 指令上使用的 `--tries` 開關定義。或者，最大嘗試次數也可以在 Job 類別本身上定義。有關執行佇列 Worker 的更多資訊 [可以在下方找到](#running-the-queue-worker)。

<a name="manually-releasing-a-job"></a>
#### 手動釋放 Job

有時你可能希望手動將 Job 重新放回佇列，以便稍後再次嘗試。你可以透過呼叫 `release` 方法來實現：

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

預設情況下，`release` 方法會將 Job 重新放回佇列以供立即處理。然而，你可以指示佇列在經過指定秒數之前，不要讓 Job 可供處理，方法是將整數或日期實例傳遞給 `release` 方法：

```php
$this->release(10);

$this->release(now()->addSeconds(10));
```

<a name="manually-failing-a-job"></a>
#### 手動失敗 Job

有時你可能需要手動將 Job 標記為「失敗」。為此，你可以呼叫 `fail` 方法：

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

如果你想因為捕捉到例外而將 Job 標記為失敗，你可以將該例外傳遞給 `fail` 方法。或者，為了方便起見，你可以傳遞一個字串錯誤訊息，它將被轉換為例外：

```php
$this->fail($exception);

$this->fail('Something went wrong.');
```

> [!NOTE]
> 關於失敗 Job 的更多資訊，請查閱 [處理 Job 失敗的說明文件](#dealing-with-failed-jobs)。

<a name="fail-jobs-on-exceptions"></a>
#### 在特定例外發生時 Job 失敗

`FailOnException` [Job 中介層](#job-middleware) 允許你在拋出特定例外時中止重試。這允許在外部 API 錯誤等暫時性例外情況下重試，但在使用者權限被撤銷等持久性例外情況下永久使 Job 失敗：

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

Laravel 的 Job 批次處理功能讓您可以輕鬆執行一批 Job，並在 Job 批次執行完成後執行某些動作。在開始之前，您應該建立資料庫遷移來建立一個資料表，其中將包含有關您 Job 批次的元資料，例如它們的完成百分比。此遷移可以使用 `make:queue-batches-table` Artisan 指令來產生：

```shell
php artisan make:queue-batches-table

php artisan migrate
```


<a name="defining-batchable-jobs"></a>
### 定義可批次處理的 Job

要定義一個可批次處理的 Job，您應該像往常一樣[建立一個可排入佇列的 Job](#creating-jobs)；然而，您應該將 `Illuminate\Bus\Batchable` trait 加入到 Job Class 中。這個 trait 提供了 `batch` 方法的存取權限，該方法可用來擷取 Job 正在執行的當前批次：

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

要分派一批 Job，您應該使用 `Bus` Facade 的 `batch` 方法。當然，批次處理主要在與完成回呼搭配使用時才有用。因此，您可以使用 `then`、`catch` 和 `finally` 方法來為批次定義完成回呼。每個回呼在被叫用時都將收到一個 `Illuminate\Bus\Batch` 實例。

當執行多個佇列 Worker 時，批次中的 Job 將會並行處理。因此，Job 完成的順序可能與它們加入批次的順序不同。有關如何依序執行一系列 Job 的資訊，請查閱我們關於[Job 鏈與批次](#chains-and-batches)的說明文件。

在此範例中，我們將假設我們正在將一批 Job 排入佇列，每個 Job 處理 CSV 檔案中的指定行數：

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

批次的 ID（可透過 `$batch->id` 屬性存取）可用於在分派後[查詢 Laravel 命令匯流排](#inspecting-batches)以獲取有關批次的資訊。

> [!WARNING]
> 由於批次回呼由 Laravel 佇列在稍後序列化並執行，因此您不應在回呼中使用 `$this` 變數。此外，由於批次處理的 Job 包裹在資料庫交易中，因此不應在 Job 中執行觸發隱式提交的資料庫語句。


<a name="naming-batches"></a>
#### 命名批次

如果批次被命名，某些工具，例如 [Laravel Horizon](/docs/{{version}}/horizon) 和 [Laravel Telescope](/docs/{{version}}/telescope)，可能會為批次提供更易於使用的偵錯資訊。要為批次指定一個任意名稱，您可以在定義批次時呼叫 `name` 方法：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import CSV')->dispatch();
```


<a name="batch-connection-queue"></a>
#### 批次連線與佇列

如果您想為批次處理的 Job 指定要使用的連線和佇列，您可以使用 `onConnection` 和 `onQueue` 方法。所有批次處理的 Job 都必須在相同的連線和佇列中執行：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->onConnection('redis')->onQueue('imports')->dispatch();
```


<a name="chains-and-batches"></a>
### 鏈與批次

您可以透過將鏈式 Job 放置在陣列中，來在批次中定義一組[鏈式 Job](#job-chaining)。例如，我們可以並行執行兩個 Job 鏈，並在兩個 Job 鏈都完成處理後執行一個回呼：

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

相反地，您可以透過在 Job 鏈中定義批次，來在[Job 鏈](#job-chaining)中執行 Job 批次。例如，您可以先執行一批 Job 來發布多個 Podcast，然後再執行一批 Job 來傳送發布通知：

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
### 將 Job 加入批次

有時，從批次處理的 Job 內部向批次添加更多 Job 可能會很有用。當您需要批次處理數千個 Job，這些 Job 在 Web 請求期間分派可能需要太長時間時，此模式會很有用。因此，您可以改為分派初始批次的「載入器」Job，這些 Job 將為批次注入更多 Job：

```php
$batch = Bus::batch([
    new LoadImportBatch,
    new LoadImportBatch,
    new LoadImportBatch,
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import Contacts')->dispatch();
```

在此範例中，我們將使用 `LoadImportBatch` Job 為批次注入額外的 Job。為此，我們可以使用批次實例上的 `add` 方法，該方法可透過 Job 的 `batch` 方法存取：

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
> 您只能從屬於同一個批次的 Job 中向該批次添加 Job。

<a name="inspecting-batches"></a>
### 檢查批次

提供給批次完成回呼的 `Illuminate\Bus\Batch` 實例具備多種屬性與方法，協助您與指定 Job 批次互動並進行檢查：

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

所有 `Illuminate\Bus\Batch` 實例皆可被 JSON 序列化，這表示您可以直接從應用程式的路由回傳它們，以取得包含批次資訊 (包括其完成進度) 的 JSON payload。這讓在應用程式 UI 中顯示批次完成進度資訊變得方便。

若要透過其 ID 取得批次，您可以使用 `Bus` Facade 的 `findBatch` 方法：

```php
use Illuminate\Support\Facades\Bus;
use Illuminate\Support\Facades\Route;

Route::get('/batch/{batchId}', function (string $batchId) {
    return Bus::findBatch($batchId);
});
```


<a name="cancelling-batches"></a>
### 取消批次

有時您可能需要取消指定批次的執行。這可以透過在 `Illuminate\Bus\Batch` 實例上呼叫 `cancel` 方法來實現：

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

如您在先前的範例中所見，批次處理的 Job 通常應在繼續執行之前判斷其對應的批次是否已取消。然而，為了方便起見，您可以改為將 `SkipIfBatchCancelled` [中介層](#job-middleware) 指派給該 Job。正如其名稱所示，此中介層將指示 Laravel，如果其對應的批次已取消，則不處理該 Job：

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

當批次處理的 Job 失敗時，`catch` 回呼 (如果已指派) 將會被呼叫。此回呼僅針對批次中第一個失敗的 Job 呼叫。


<a name="allowing-failures"></a>
#### 允許失敗

當批次中的 Job 失敗時，Laravel 會自動將批次標記為「已取消」。如果您希望，您可以停用此行為，讓 Job 失敗時不會自動將批次標記為已取消。這可以透過在分派批次時呼叫 `allowFailures` 方法來實現：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->allowFailures()->dispatch();
```

您可以選擇性地為 `allowFailures` 方法提供一個閉包，該閉包將在每個 Job 失敗時執行：

```php
$batch = Bus::batch([
    // ...
])->allowFailures(function (Batch $batch, $exception) {
    // Handle individual job failures...
})->dispatch();
```


<a name="retrying-failed-batch-jobs"></a>
#### 重試失敗的批次 Job

為了方便起見，Laravel 提供了一個 `queue:retry-batch` Artisan 指令，讓您可以輕鬆重試指定批次中所有失敗的 Job。此指令接受應重試其失敗 Job 的批次的 UUID：

```shell
php artisan queue:retry-batch 32dbc76c-4f82-4749-b610-a639fe0099b5
```


<a name="pruning-batches"></a>
### 清除批次

如果不清除，`job_batches` 資料表會非常快速地累積記錄。為了緩解此問題，您應[排程](/docs/{{version}}/scheduling) `queue:prune-batches` Artisan 指令每日執行：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches')->daily();
```

預設情況下，所有超過 24 小時的已完成批次將被清除。您可以在呼叫指令時使用 `hours` 選項，以決定保留批次資料的時間長度。例如，以下指令將刪除所有在 48 小時前完成的批次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48')->daily();
```

有時，您的 `jobs_batches` 資料表可能會累積從未成功完成的批次記錄，例如，其中一個 Job 失敗且從未成功重試的批次。您可以指示 `queue:prune-batches` 指令使用 `unfinished` 選項來清除這些未完成的批次記錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --unfinished=72')->daily();
```

同樣地，您的 `jobs_batches` 資料表也可能累積已取消批次的記錄。您可以指示 `queue:prune-batches` 指令使用 `cancelled` 選項來清除這些已取消的批次記錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --cancelled=72')->daily();
```

<a name="storing-batches-in-dynamodb"></a>
### 將批次儲存至 DynamoDB

Laravel 也支援將批次中繼資訊儲存在 [DynamoDB](https://aws.amazon.com/dynamodb) 而非關聯式資料庫。但是，您需要手動建立 DynamoDB 資料表來儲存所有批次紀錄。

通常，這個資料表應命名為 `job_batches`，但您可以根據應用程式 `queue` 設定檔中 `queue.batching.table` 設定值的名稱來命名資料表。


<a name="dynamodb-batch-table-configuration"></a>
#### DynamoDB 批次資料表設定

`job_batches` 資料表應具有一個名為 `application` 的字串型別主要分區索引鍵，以及一個名為 `id` 的字串型別主要排序索引鍵。索引鍵中的 `application` 部分將包含您的應用程式名稱，如應用程式 `app` 設定檔中 `name` 設定值所定義。由於應用程式名稱是 DynamoDB 資料表索引鍵的一部分，因此您可以使用相同的資料表來儲存多個 Laravel 應用程式的 Job 批次。

此外，如果想利用[自動清除批次](#pruning-batches-in-dynamodb)的功能，您可以為您的資料表定義 `ttl` 屬性。


<a name="dynamodb-configuration"></a>
#### DynamoDB 設定

接下來，安裝 AWS SDK，讓您的 Laravel 應用程式可以與 Amazon DynamoDB 進行通訊：

```shell
composer require aws/aws-sdk-php
```

然後，將 `queue.batching.driver` 設定選項的值設為 `dynamodb`。此外，您應該在 `batching` 設定陣列中定義 `key`、`secret` 和 `region` 設定選項。這些選項將用於向 AWS 進行身份驗證。使用 `dynamodb` 驅動器時，`queue.batching.database` 設定選項是不必要的：

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
#### 在 DynamoDB 中清除批次

當利用 [DynamoDB](https://aws.amazon.com/dynamodb) 儲存 Job 批次資訊時，用於清除儲存在關聯式資料庫中批次的常見清除指令將無法運作。相反地，您可以利用 [DynamoDB 的原生 TTL 功能](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)來自動移除舊批次的紀錄。

如果您的 DynamoDB 資料表定義了 `ttl` 屬性，您可以定義設定參數來指示 Laravel 如何清除批次紀錄。`queue.batching.ttl_attribute` 設定值定義了儲存 TTL 的屬性名稱，而 `queue.batching.ttl` 設定值定義了批次紀錄可從 DynamoDB 資料表移除的秒數，此秒數是相對於紀錄上次更新的時間：

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
## 閉包排入佇列

除了分派 Job Class 到佇列之外，您也可以分派閉包。這對於需要在當前請求週期之外執行的快速、簡單任務非常有用。當分派閉包到佇列時，閉包的程式碼內容會經過加密簽署，以防止在傳輸過程中被修改：

```php
use App\Models\Podcast;

$podcast = Podcast::find(1);

dispatch(function () use ($podcast) {
    $podcast->publish();
});
```

若要為已排入佇列的閉包指定一個名稱，該名稱可用於佇列報告儀表板，也能由 `queue:work` 指令顯示，您可以使用 `name` 方法：

```php
dispatch(function () {
    // ...
})->name('Publish Podcast');
```

使用 `catch` 方法，您可以提供一個閉包，當已排入佇列的閉包在用盡佇列所有 [配置的重試嘗試次數](#max-job-attempts-and-timeout) 後仍未能成功完成時，該閉包將會被執行：

```php
use Throwable;

dispatch(function () use ($podcast) {
    $podcast->publish();
})->catch(function (Throwable $e) {
    // This job has failed...
});
```

> [!WARNING]
> 由於 `catch` 回呼是由 Laravel 佇列序列化並稍後執行，因此您不應在 `catch` 回呼中使用 `$this` 變數。

<a name="running-the-queue-worker"></a>
## 執行佇列 Worker

<a name="the-queue-work-command"></a>
### `queue:work` 指令

Laravel 包含一個 Artisan 指令，它會啟動一個佇列 Worker 並處理被推送到佇列中的新 Job。您可以使用 `queue:work` Artisan 指令來運行 Worker。請注意，一旦 `queue:work` 指令啟動後，它會持續運行，直到您手動停止它或關閉終端機為止：

```shell
php artisan queue:work
```

> [!NOTE]
> 若要讓 `queue:work` 程序在背景永久運行，您應該使用像 [Supervisor](#supervisor-configuration) 這樣的行程監控器，以確保佇列 Worker 不會停止運行。

當您調用 `queue:work` 指令時，可以包含 `-v` 旗標，以在指令輸出中顯示已處理的 Job ID、連線名稱和佇列名稱：

```shell
php artisan queue:work -v
```

請記住，佇列 Worker 是長時程程序，並將啟動的應用程式狀態儲存在記憶體中。因此，它們在啟動後不會注意到程式碼庫的變更。所以，在您的部署流程中，請務必 [重新啟動佇列 Worker](#queue-workers-and-deployment)。此外，請記住，應用程式建立或修改的任何靜態狀態都不會在 Job 之間自動重設。

或者，您也可以運行 `queue:listen` 指令。當使用 `queue:listen` 指令時，您不需要在重新載入更新後的程式碼或重設應用程式狀態時手動重新啟動 Worker；但是，這個指令的效率明顯低於 `queue:work` 指令：

```shell
php artisan queue:listen
```

<a name="running-multiple-queue-workers"></a>
#### 執行多個佇列 Worker

若要將多個 Worker 分配給一個佇列並同時處理 Job，您只需啟動多個 `queue:work` 程序即可。這可以在本機透過終端機中的多個分頁完成，或者在生產環境中使用您的行程管理器設定來完成。[當使用 Supervisor](#supervisor-configuration) 時，您可以使用 `numprocs` 設定值。

<a name="specifying-the-connection-queue"></a>
#### 指定連線與佇列

您也可以指定 Worker 應該使用哪個佇列連線。傳遞給 `work` 指令的連線名稱應與您的 `config/queue.php` 設定檔中定義的其中一個連線相對應：

```shell
php artisan queue:work redis
```

預設情況下，`queue:work` 指令只會處理指定連線的預設佇列中的 Job。然而，您可以進一步自訂您的佇列 Worker，使其只處理指定連線的特定佇列。例如，如果您所有的電子郵件都在 `redis` 佇列連線上的 `emails` 佇列中處理，您可以執行以下指令來啟動一個只處理該佇列的 Worker：

```shell
php artisan queue:work redis --queue=emails
```

<a name="processing-a-specified-number-of-jobs"></a>
#### 處理指定數量的 Job

`--once` 選項可用於指示 Worker 只從佇列中處理一個 Job：

```shell
php artisan queue:work --once
```

`--max-jobs` 選項可用於指示 Worker 處理指定數量的 Job 後退出。此選項與 [Supervisor](#supervisor-configuration) 結合使用時會很有用，以便您的 Worker 在處理完指定數量的 Job 後自動重新啟動，釋放它們可能累積的任何記憶體：

```shell
php artisan queue:work --max-jobs=1000
```

<a name="processing-all-queued-jobs-then-exiting"></a>
#### 處理所有佇列中的 Job 並退出

`--stop-when-empty` 選項可用於指示 Worker 處理所有 Job 後優雅地退出。此選項在 Docker 容器內處理 Laravel 佇列時很有用，如果您希望在佇列清空後關閉容器：

```shell
php artisan queue:work --stop-when-empty
```

<a name="processing-jobs-for-a-given-number-of-seconds"></a>
#### 在指定秒數內處理 Job

`--max-time` 選項可用於指示 Worker 在指定秒數內處理 Job，然後退出。此選項與 [Supervisor](#supervisor-configuration) 結合使用時會很有用，以便您的 Worker 在處理 Job 指定時間後自動重新啟動，釋放它們可能累積的任何記憶體：

```shell
# Process jobs for one hour and then exit...
php artisan queue:work --max-time=3600
```

<a name="worker-sleep-duration"></a>
#### Worker 睡眠時間

當佇列中有 Job 可用時，Worker 將會持續處理 Job，Job 之間沒有延遲。然而，`sleep` 選項決定了當沒有 Job 可用時，Worker 將「休眠」多少秒。當然，在休眠期間，Worker 不會處理任何新的 Job：

```shell
php artisan queue:work --sleep=3
```

<a name="maintenance-mode-queues"></a>
#### 維護模式與佇列

當您的應用程式處於 [維護模式](/docs/{{version}}/configuration#maintenance-mode) 時，不會處理任何佇列中的 Job。一旦應用程式退出維護模式，這些 Job 將會恢復正常處理。

若要強制佇列 Worker 即使在維護模式啟用時也處理 Job，您可以使用 `--force` 選項：

```shell
php artisan queue:work --force
```

<a name="resource-considerations"></a>
#### 資源考量

守護行程佇列 Worker 在處理每個 Job 之前不會「重啟」框架。因此，您應該在每個 Job 完成後釋放所有大量資源。例如，如果您正在使用 [GD library](https://www.php.net/manual/en/book.image.php) 進行影像處理，您應該在處理完影像後使用 `imagedestroy` 釋放記憶體。

<a name="queue-priorities"></a>
### 佇列優先權

有時您可能希望優先處理您的佇列。例如，在您的 `config/queue.php` 設定檔中，您可以將 `redis` 連線的預設 `queue` 設定為 `low`。然而，偶爾您可能希望將 Job 推送到 `high` 優先權的佇列，如下所示：

```php
dispatch((new Job)->onQueue('high'));
```

若要啟動一個 Worker，確保所有 `high` 佇列中的 Job 都被處理完畢後，才繼續處理 `low` 佇列中的任何 Job，請將一個逗號分隔的佇列名稱列表傳遞給 `work` 指令：

```shell
php artisan queue:work --queue=high,low
```

<a name="queue-workers-and-deployment"></a>
### 佇列 Worker 與部署

由於佇列 Worker 是長時程程序，若不重新啟動，它們將不會察覺程式碼的變更。因此，使用佇列 Worker 部署應用程式最簡單的方法是在部署流程中重新啟動 Worker。您可以透過執行 `queue:restart` 指令來優雅地重新啟動所有 Worker：

```shell
php artisan queue:restart
```

此指令會指示所有佇列 Worker 在完成處理當前 Job 後優雅地退出，以確保現有 Job 不會遺失。由於佇列 Worker 在執行 `queue:restart` 指令時會退出，您應該運行像 [Supervisor](#supervisor-configuration) 這樣的行程管理器來自動重新啟動佇列 Worker。

> [!NOTE]
> 佇列使用 [快取](/docs/{{version}}/cache) 來儲存重啟訊號，因此在使用此功能之前，您應該驗證您的應用程式已正確配置快取驅動器。

<a name="job-expirations-and-timeouts"></a>
### Job 過期與逾時


<a name="job-expiration"></a>
#### Job 過期

在您的 `config/queue.php` 設定檔中，每個佇列連線都定義了一個 `retry_after` 選項。此選項指定佇列連線在重試正在處理的 Job 之前應該等待多少秒。例如，如果 `retry_after` 的值設定為 `90`，則如果 Job 已經處理了 90 秒而沒有被釋出或刪除，它將被釋回佇列。通常，您應該將 `retry_after` 值設定為您的 Job 合理完成處理所需的最大秒數。

> [!WARNING]
> 唯一不包含 `retry_after` 值的佇列連線是 Amazon SQS。SQS 將根據 [預設可見度逾時](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html) 重試 Job，此逾時在 AWS console 中管理。


<a name="worker-timeouts"></a>
#### Worker 逾時

`queue:work` Artisan 指令會提供一個 `--timeout` 選項。預設情況下，`--timeout` 值為 60 秒。如果 Job 的處理時間超過逾時值指定的秒數，則處理 Job 的 Worker 會帶著錯誤結束。通常，Worker 會被[伺服器上設定的程序管理器](#supervisor-configuration)自動重啟：

```shell
php artisan queue:work --timeout=60
```

`retry_after` 設定選項和 `--timeout` CLI 選項是不同的，但它們共同作用以確保 Job 不會遺失並且 Job 僅被成功處理一次。

> [!WARNING]
> `--timeout` 值應該始終比您的 `retry_after` 設定值至少短幾秒。這將確保處理凍結 Job 的 Worker 在 Job 重試之前就會被終止。如果您的 `--timeout` 選項比您的 `retry_after` 設定值長，您的 Job 可能會被處理兩次。

<a name="supervisor-configuration"></a>
## Supervisor 設定

在生產環境中，您需要一種方式來保持您的 `queue:work` 程序持續運行。`queue:work` 程序可能會因多種原因停止運行，例如 Worker 逾時或執行了 `queue:restart` 指令。

為此，您需要設定一個程序監控器，以偵測您的 `queue:work` 程序何時終止並自動重新啟動它們。此外，程序監控器還允許您指定要同時運行多少個 `queue:work` 程序。Supervisor 是一個在 Linux 環境中常用的程序監控器，我們將在本文件中討論如何設定它。


<a name="installing-supervisor"></a>
#### 安裝 Supervisor

Supervisor 是一個 Linux 作業系統的程序監控器，如果您的 `queue:work` 程序失敗，它將自動重新啟動。要在 Ubuntu 上安裝 Supervisor，您可以使用以下指令：

```shell
sudo apt-get install supervisor
```

> [!NOTE]
> 如果自行設定和管理 Supervisor 聽起來很困難，請考慮使用 [Laravel Cloud](https://cloud.laravel.com)，它提供了一個完全託管的平台來運行 Laravel 佇列 Worker。


<a name="configuring-supervisor"></a>
#### 設定 Supervisor

Supervisor 設定檔通常儲存在 `/etc/supervisor/conf.d` 目錄中。在此目錄中，您可以建立任意數量的設定檔，這些檔案會指示 Supervisor 如何監控您的程序。例如，讓我們建立一個 `laravel-worker.conf` 檔案，用於啟動和監控 `queue:work` 程序：

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

在此範例中，`numprocs` 指令將指示 Supervisor 運行八個 `queue:work` 程序並監控它們，如果它們失敗，將自動重新啟動。您應該修改設定中的 `command` 指令，以反映您所需的佇列連線和 Worker 選項。

> [!WARNING]
> 您應該確保 `stopwaitsecs` 的值大於您運行時間最長的 Job 所消耗的秒數。否則，Supervisor 可能會在 Job 完成處理之前將其終止。


<a name="starting-supervisor"></a>
#### 啟動 Supervisor

設定檔建立後，您可以使用以下指令更新 Supervisor 設定並啟動程序：

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start "laravel-worker:*"
```

有關 Supervisor 的更多資訊，請查閱 [Supervisor documentation](http://supervisord.org/index.html)。

<a name="dealing-with-failed-jobs"></a>
## 處理失敗的 Job

有時候您排入佇列的 Job 會失敗。別擔心，事情不總是按計畫進行！Laravel 提供了一種方便的方式來[指定 Job 的最大嘗試次數](#max-job-attempts-and-timeout)。當一個非同步 Job 超過此嘗試次數後，它將會被插入到 `failed_jobs` 資料庫表格中。[同步分派的 Job](/docs/{{version}}/queues#synchronous-dispatching) 失敗後不會儲存在此表格中，其例外會立即由應用程式處理。

建立 `failed_jobs` 表格的遷移 (Migration) 通常已存在於新的 Laravel 應用程式中。然而，如果您的應用程式不包含此表格的遷移，您可以使用 `make:queue-failed-table` 指令來建立此遷移：

```shell
php artisan make:queue-failed-table

php artisan migrate
```

執行 [佇列 Worker](#running-the-queue-worker) 程序時，您可以使用 `queue:work` 指令上的 `--tries` 開關來指定 Job 應嘗試的最大次數。如果您未指定 `--tries` 選項的值，Job 只會嘗試一次，或依照 Job Class 的 `$tries` 屬性指定次數：

```shell
php artisan queue:work redis --tries=3
```

使用 `--backoff` 選項，您可以指定 Laravel 應等待多少秒，然後重試遇到例外的 Job。預設情況下，Job 會立即釋放回佇列以便再次嘗試：

```shell
php artisan queue:work redis --tries=3 --backoff=3
```

如果您想為每個 Job 配置 Laravel 應等待多少秒，然後重試遇到例外的 Job，您可以在 Job Class 上定義一個 `backoff` 屬性：

```php
/**
 * The number of seconds to wait before retrying the job.
 *
 * @var int
 */
public $backoff = 3;
```

如果您需要更複雜的邏輯來決定 Job 的退避時間，您可以在 Job Class 上定義一個 `backoff` 方法：

```php
/**
 * Calculate the number of seconds to wait before retrying the job.
 */
public function backoff(): int
{
    return 3;
}
```

您可以透過從 `backoff` 方法回傳一個退避值陣列，輕鬆配置「指數型」退避。在此範例中，第一次重試的延遲為 1 秒，第二次為 5 秒，第三次為 10 秒，如果還有更多嘗試次數，之後每次重試都將延遲 10 秒：

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
### 清理失敗的 Job

當特定的 Job 失敗時，您可能想要向使用者發送警示，或還原 Job 部分完成的任何動作。為此，您可在 Job Class 上定義 `failed` 方法。導致 Job 失敗的 `Throwable` 實例將會傳遞給 `failed` 方法：

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
> 在呼叫 `failed` 方法之前，會先實例化一個新的 Job 實例；因此，任何在 `handle` 方法中發生的 Class 屬性修改，都將會遺失。

失敗的 Job 不一定是遇到未處理的例外。Job 也可能被視為失敗，當它用盡所有允許的嘗試次數時。這些嘗試次數可以透過多種方式消耗掉：

<div class="content-list" markdown="1">

- Job 逾時。
- Job 在執行期間遇到未處理的例外。
- Job 被手動或透過中介層釋放回佇列。

</div>

如果最後一次嘗試由於 Job 執行期間拋出的例外而失敗，該例外將會傳遞給 Job 的 `failed` 方法。然而，如果 Job 因為達到最大允許嘗試次數而失敗，`$exception` 將會是 `Illuminate\Queue\MaxAttemptsExceededException` 的實例。同樣地，如果 Job 因超出配置的逾時而失敗，`$exception` 將會是 `Illuminate\Queue\TimeoutExceededException` 的實例。

<a name="retrying-failed-jobs"></a>
### 重試失敗的 Job

若要查看所有已插入到您的 `failed_jobs` 資料庫表格中的失敗 Job，您可以使用 `queue:failed` Artisan 指令：

```shell
php artisan queue:failed
```

`queue:failed` 指令將會列出 Job ID、連線、佇列、失敗時間，以及 Job 的其他資訊。Job ID 可用於重試失敗的 Job。例如，要重試 ID 為 `ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece` 的失敗 Job，請執行以下指令：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece
```

如有必要，可向指令傳遞多個 ID：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece 91401d2c-0784-4f43-824c-34f94a33c24d
```

您也可以重試特定佇列的所有失敗 Job：

```shell
php artisan queue:retry --queue=name
```

若要重試所有失敗的 Job，請執行 `queue:retry` 指令並傳遞 `all` 作為 ID：

```shell
php artisan queue:retry all
```

如果您想刪除失敗的 Job，您可以使用 `queue:forget` 指令：

```shell
php artisan queue:forget 91401d2c-0784-4f43-824c-34f94a33c24d
```

> [!NOTE]
> 使用 [Horizon](/docs/{{version}}/horizon) 時，應使用 `horizon:forget` 指令來刪除失敗的 Job，而不是 `queue:forget` 指令。

若要從 `failed_jobs` 表格刪除所有失敗的 Job，您可以使用 `queue:flush` 指令：

```shell
php artisan queue:flush
```

`queue:flush` 指令會從佇列中移除所有失敗的 Job 記錄，無論失敗的 Job 有多舊。您可以使用 `--hours` 選項僅刪除在特定小時數之前或更早失敗的 Job：

```shell
php artisan queue:flush --hours=48
```

<a name="ignoring-missing-models"></a>
### 忽略遺失的 Model

將 Eloquent Model 注入到 Job 中時，Model 會自動序列化，在排入佇列之前，並在 Job 處理時從資料庫中重新取得。然而，如果 Model 在 Job 等待 Worker 處理期間已被刪除，您的 Job 可能會以 `ModelNotFoundException` 失敗。

為了方便，您可以選擇透過將 Job 的 `deleteWhenMissingModels` 屬性設定為 `true` 來自動刪除 Model 遺失的 Job。當此屬性設定為 `true` 時，Laravel 會悄悄地丟棄 Job 而不會拋出例外：

```php
/**
 * Delete the job if its models no longer exist.
 *
 * @var bool
 */
public $deleteWhenMissingModels = true;
```

<a name="pruning-failed-jobs"></a>
### 清除失敗的 Job

您可以透過執行 `queue:prune-failed` Artisan 指令來清除應用程式 `failed_jobs` 表格中的記錄：

```shell
php artisan queue:prune-failed
```

預設情況下，所有超過 24 小時的失敗 Job 記錄都將被清除。如果您向指令提供了 `--hours` 選項，則只會保留在最近 N 小時內插入的失敗 Job 記錄。例如，以下指令將會刪除所有超過 48 小時前插入的失敗 Job 記錄：

```shell
php artisan queue:prune-failed --hours=48
```

<a name="storing-failed-jobs-in-dynamodb"></a>
### 將失敗的 Job 儲存至 DynamoDB

Laravel 也支援將您的失敗 Job 記錄儲存至 [DynamoDB](https://aws.amazon.com/dynamodb)，而非關聯式資料庫表格。不過，您必須手動建立一個 DynamoDB 資料表來儲存所有失敗的 Job 記錄。通常，這個資料表應命名為 `failed_jobs`，但您應根據應用程式 `queue` 設定檔中 `queue.failed.table` 設定值來命名資料表。

`failed_jobs` 資料表應具有名為 `application` 的字串主要分割區鍵 (primary partition key) 與名為 `uuid` 的字串主要排序鍵 (primary sort key)。鍵的 `application` 部分將包含應用程式 `app` 設定檔中 `name` 設定值所定義的應用程式名稱。由於應用程式名稱是 DynamoDB 資料表鍵的一部分，您可以使用同一個資料表來儲存多個 Laravel 應用程式的失敗 Job。

此外，請確保您已安裝 AWS SDK，以便您的 Laravel 應用程式可以與 Amazon DynamoDB 進行通訊：

```shell
composer require aws/aws-sdk-php
```

接著，將 `queue.failed.driver` 設定選項的值設定為 `dynamodb`。此外，您應在失敗 Job 設定陣列中定義 `key`、`secret` 和 `region` 設定選項。這些選項將用於向 AWS 進行身份驗證。當使用 `dynamodb` 驅動器時，`queue.failed.database` 設定選項是不必要的：

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

您可以指示 Laravel 捨棄失敗的 Job 而不儲存它們，方法是將 `queue.failed.driver` 設定選項的值設定為 `null`。通常，這可以透過 `QUEUE_FAILED_DRIVER` 環境變數來達成：

```ini
QUEUE_FAILED_DRIVER=null
```

<a name="failed-job-events"></a>
### 失敗 Job 事件

如果您想註冊一個事件監聽器，以便在 Job 失敗時被呼叫，可以使用 `Queue` Facade 的 `failing` 方法。例如，我們可以從 Laravel 隨附的 `AppServiceProvider` 的 `boot` 方法中，將一個閉包附加到此事件：

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
## 從佇列清除 Job

> [!NOTE]
> 使用 [Horizon](/docs/{{version}}/horizon) 時，應該使用 `horizon:clear` 指令來清除佇列中的 Job，而不是使用 `queue:clear` 指令。

若要刪除預設連線之預設佇列中的所有 Job，可以使用 `queue:clear` Artisan 指令：

```shell
php artisan queue:clear
```

您也可以提供 `connection` 引數與 `queue` 選項來從指定的連線與佇列中刪除 Job：

```shell
php artisan queue:clear redis --queue=emails
```

> [!WARNING]
> 從佇列中清除 Job 僅適用於 SQS、Redis 與資料庫佇列驅動器。此外，SQS 訊息刪除過程最長需 60 秒，因此在清除佇列後 60 秒內傳送至 SQS 佇列的 Job 也可能被刪除。

<a name="monitoring-your-queues"></a>
## 監控佇列

若佇列突然湧入大量 Job，可能會導致佇列不堪重負，造成 Job 完成時間過長。若您希望，當佇列中的 Job 數量超過指定閾值時，Laravel 可以發出警示。

首先，您應該將 `queue:monitor` 指令排程為 [每分鐘執行一次](/docs/{{version}}/scheduling)。此指令接受您希望監控的佇列名稱以及您所需的 Job 數量閾值：

```shell
php artisan queue:monitor redis:default,redis:deployments --max=100
```

單獨排程此指令不足以觸發通知以警示您佇列不堪重負的狀態。當指令遇到 Job 數量超出您閾值的佇列時，將會分派 `Illuminate\Queue\Events\QueueBusy` 事件。您可以在應用程式的 `AppServiceProvider` 中監聽此事件，以便向您或您的開發團隊傳送通知：

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

當測試分派 Job 的程式碼時，您可能希望指示 Laravel 不要實際執行 Job 本身，因為 Job 的程式碼可以直接且獨立於分派它的程式碼進行測試。當然，要測試 Job 本身，您可以直接在測試中實例化一個 Job 實例並呼叫其 `handle` 方法。

您可以使用 `Queue` Facade 的 `fake` 方法來防止排入佇列的 Job 實際被推送到佇列中。呼叫 `Queue` Facade 的 `fake` 方法後，您可以斷言應用程式嘗試將 Job 推送到佇列：

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

您可以傳遞一個閉包給 `assertPushed`、`assertNotPushed`、`assertClosurePushed` 或 `assertClosureNotPushed` 方法，以斷言推送到佇列中的 Job 通過了給定的「真值測試」。如果至少有一個 Job 通過了給定的真值測試，則斷言將會成功：

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
### 模擬 Job 子集

如果您只需要模擬特定的 Job，同時允許其他 Job 正常執行，您可以將需要模擬的 Job Class 名稱傳遞給 `fake` 方法：

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

您可以使用 `except` 方法模擬所有 Job，除了指定的一組 Job 之外：

```php
Queue::fake()->except([
    ShipOrder::class,
]);
```

<a name="testing-job-chains"></a>
### 測試 Job 鏈

為了測試 Job 鏈，您需要利用 `Bus` Facade 的模擬功能。`Bus` Facade 的 `assertChained` 方法可用於斷言一個 Job 鏈已被分派。`assertChained` 方法接受一個 Job 鏈陣列作為其第一個參數：

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

如您在上面的範例中看到的，Job 鏈陣列可以是 Job 的 Class 名稱陣列。但是，您也可以提供實際 Job 實例的陣列。這樣做時，Laravel 將確保 Job 實例與應用程式分派的 Job 鏈擁有相同的 Class 和屬性值：

```php
Bus::assertChained([
    new ShipOrder,
    new RecordShipment,
    new UpdateInventory,
]);
```

您可以使用 `assertDispatchedWithoutChain` 方法斷言一個 Job 被推送到佇列時沒有 Job 鏈：

```php
Bus::assertDispatchedWithoutChain(ShipOrder::class);
```

<a name="testing-chain-modifications"></a>
#### 測試鏈修改

如果鏈中的 Job 在現有的鏈中[預置或附加 Job](#adding-jobs-to-the-chain)，您可以使用該 Job 的 `assertHasChain` 方法來斷言該 Job 擁有所預期的剩餘 Job 鏈：

```php
$job = new ProcessPodcast;

$job->handle();

$job->assertHasChain([
    new TranscribePodcast,
    new OptimizePodcast,
    new ReleasePodcast,
]);
```

`assertDoesntHaveChain` 方法可用於斷言該 Job 的剩餘鏈為空：

```php
$job->assertDoesntHaveChain();
```

<a name="testing-chained-batches"></a>
#### 測試鏈式批次

如果您的 Job 鏈[包含 Job 批次](#chains-and-batches)，您可以透過在鏈斷言中插入一個 `Bus::chainedBatch` 定義，來斷言鏈式批次符合您的預期：

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

`Bus` Facade 的 `assertBatched` 方法可用於斷言一個 [Job 批次](/docs/{{version}}/queues#job-batching) 已被分派。傳遞給 `assertBatched` 方法的閉包會接收一個 `Illuminate\Bus\PendingBatch` 實例，該實例可用於檢查批次內的 Job：

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

您可以使用 `assertBatchCount` 方法斷言指定數量的批次已被分派：

```php
Bus::assertBatchCount(3);
```

您可以使用 `assertNothingBatched` 斷言沒有任何批次被分派：

```php
Bus::assertNothingBatched();
```

<a name="testing-job-batch-interaction"></a>
#### 測試 Job / 批次互動

此外，您可能偶爾需要測試個別 Job 與其底層批次的互動。例如，您可能需要測試 Job 是否取消了其批次的後續處理。為此，您需要透過 `withFakeBatch` 方法為 Job 分配一個模擬批次。`withFakeBatch` 方法會回傳一個包含 Job 實例和模擬批次的 Tuple：

```php
[$job, $batch] = (new ShipOrder)->withFakeBatch();

$job->handle();

$this->assertTrue($batch->cancelled());
$this->assertEmpty($batch->added);
```

<a name="testing-job-queue-interactions"></a>
### 測試 Job / 佇列互動

有時，您可能需要測試已排入佇列的 Job 是否[自行釋放回佇列](#manually-releasing-a-job)。或者，您可能需要測試 Job 是否自行刪除了。您可以透過實例化 Job 並呼叫其 `withFakeQueueInteractions` 方法來測試這些佇列互動。

一旦 Job 的佇列互動被模擬後，您就可以呼叫 Job 上的 `handle` 方法。在呼叫 Job 後，有多種斷言方法可用來驗證 Job 的佇列互動：

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

藉由 `Queue` [Facade](/docs/{{version}}/facades) 上的 `before` 與 `after` 方法，您可以指定在佇列 Job 處理之前或之後執行的回呼。這些回呼是執行額外日誌記錄或增加儀表板統計數據的絕佳機會。通常，您應該從 [Service Provider](/docs/{{version}}/providers) 的 `boot` 方法呼叫這些方法。例如，我們可以使用 Laravel 內建的 `AppServiceProvider`：

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

藉由 `Queue` [Facade](/docs/{{version}}/facades) 上的 `looping` 方法，您可以指定在 Worker 嘗試從佇列中提取 Job 之前執行的回呼。例如，您可以註冊一個閉包來回溯任何由先前失敗的 Job 遺留下來的交易：

```php
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Queue;

Queue::looping(function () {
    while (DB::transactionLevel() > 0) {
        DB::rollBack();
    }
});
```