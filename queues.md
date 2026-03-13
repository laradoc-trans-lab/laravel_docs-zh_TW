# Queues

- [簡介](#introduction)
    - [連接 vs. Queues](#connections-vs-queues)
    - [驅動程式說明與先決條件](#driver-prerequisites)
- [建立 Jobs](#creating-jobs)
    - [產生 Job 類別](#generating-job-classes)
    - [類別結構](#class-structure)
    - [Unique Jobs](#unique-jobs)
    - [加密的 Jobs](#encrypted-jobs)
- [Job 中介層](#job-middleware)
    - [速率限制](#rate-limiting)
    - [防止 Job 重疊](#preventing-job-overlaps)
    - [節流異常](#throttling-exceptions)
    - [跳過 Jobs](#skipping-jobs)
- [派送 Jobs](#dispatching-jobs)
    - [延遲派送](#delayed-dispatching)
    - [同步派送](#synchronous-dispatching)
    - [Jobs 與資料庫交易](#jobs-and-database-transactions)
    - [Job 鏈接](#job-chaining)
    - [自訂 Queue 與連接](#customizing-the-queue-and-connection)
    - [指定最大 Job 嘗試次數 / 超時值](#max-job-attempts-and-timeout)
    - [SQS FIFO 與公平 Queues](#sqs-fifo-and-fair-queues)
    - [Queue 故障轉移](#queue-failover)
    - [錯誤處理](#error-handling)
- [Job 批次](#job-batching)
    - [定義可批次的 Jobs](#defining-batchable-jobs)
    - [派送批次](#dispatching-batches)
    - [鏈接與批次](#chains-and-batches)
    - [將 Jobs 新增至批次](#adding-jobs-to-batches)
    - [檢查批次](#inspecting-batches)
    - [取消批次](#cancelling-batches)
    - [批次失敗](#batch-failures)
    - [修剪批次](#pruning-batches)
    - [將批次儲存在 DynamoDB](#storing-batches-in-dynamodb)
- [Queue 閉包](#queueing-closures)
- [執行 Queue Worker](#running-the-queue-worker)
    - [`queue:work` 命令](#the-queue-work-command)
    - [Queue 優先級](#queue-priorities)
    - [Queue Workers 與佈署](#queue-workers-and-deployment)
    - [Job 過期與超時](#job-expirations-and-timeouts)
    - [暫停與恢復 Queue Workers](#pausing-and-resuming-queue-workers)
- [Supervisor 設定](#supervisor-configuration)
- [處理失敗的 Jobs](#dealing-with-failed-jobs)
    - [失敗 Jobs 後的清理](#cleaning-up-after-failed-jobs)
    - [重試失敗的 Jobs](#retrying-failed-jobs)
    - [忽略遺失的 Models](#ignoring-missing-models)
    - [修剪失敗的 Jobs](#pruning-failed-jobs)
    - [將失敗的 Jobs 儲存在 DynamoDB](#storing-failed-jobs-in-dynamodb)
    - [停用失敗 Job 儲存](#disabling-failed-job-storage)
    - [失敗 Job 事件](#failed-job-events)
- [清除 Queue 中的 Jobs](#clearing-jobs-from-queues)
- [監控您的 Queues](#monitoring-your-queues)
- [測試](#testing)
    - [模擬部分 Jobs](#faking-a-subset-of-jobs)
    - [測試 Job 鏈接](#testing-job-chains)
    - [測試 Job 批次](#testing-job-batches)
    - [測試 Job / Queue 互動](#testing-job-queue-interactions)
- [Job 事件](#job-events)

<a name="introduction"></a>
## 簡介

在開發網頁應用程式時，您可能會遇到一些任務（例如解析並儲存上傳的 CSV 檔案），這些任務在一般的網頁請求中執行會耗費太長時間。幸運的是，Laravel 允許您輕鬆地建立排程 Jobs，並在背景處理這些任務。藉由將耗時的任務移至 Queue，您的應用程式能以極快的速度回應網頁請求，並為客戶提供更好的使用者體驗。

Laravel Queues 在各種不同的 Queue 後端（如 [Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io)，甚至是關聯式資料庫）之間提供了統一的 Queueing API。

Laravel 的 Queue 設定選項儲存在您的應用程式 `config/queue.php` 設定檔中。在此檔案中，您會發現框架包含的每個 Queue 驅動程式的連接設定，包括資料庫、[Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 和 [Beanstalkd](https://beanstalkd.github.io/) 驅動程式，以及一個會立即執行 Jobs 的同步驅動程式（用於開發或測試）。同時也包含了一個 `null` Queue 驅動程式，用於丟棄已排入 Queue 的 Jobs。

> [!NOTE]
> Laravel Horizon 是一個為 Redis 驅動的 Queues 所設計的美觀儀表板與設定系統。請參考完整的 [Horizon 文件](/docs/{{version}}/horizon) 以取得更多資訊。


<a name="connections-vs-queues"></a>
### 連接 vs. Queues

在開始使用 Laravel Queues 之前，了解「連接 (Connections)」與「Queues」之間的區別非常重要。在您的 `config/queue.php` 設定檔中，有一個 `connections` 設定陣列。此選項定義了與後端 Queue 服務（如 Amazon SQS、Beanstalk 或 Redis）的連接。然而，任何特定的 Queue 連接都可能具有多個「Queues」，您可以將其視為不同堆疊或成堆的已排程 Jobs。

請注意，`queue` 設定檔中的每個連接設定範例都包含一個 `queue` 屬性。這是當 Jobs 被發送到特定連接時，預設會派送到的 Queue。換句話說，如果您在派送 Job 時沒有明確定義應該派送到哪個 Queue，則該 Job 將被放置在連接設定的 `queue` 屬性中定義的 Queue：

```php
use App\Jobs\ProcessPodcast;

// This job is sent to the default connection's default queue...
ProcessPodcast::dispatch();

// This job is sent to the default connection's "emails" queue...
ProcessPodcast::dispatch()->onQueue('emails');
```

某些應用程式可能永遠不需要將 Jobs 推送到多個 Queues，而是傾向於使用單一的 Queue。然而，將 Jobs 推送到多個 Queues 對於想要優先處理或區分 Jobs 處理方式的應用程式特別有用，因為 Laravel Queue Worker 允許您指定應按優先順序處理哪些 Queues。例如，如果您將 Jobs 推送到 `high` Queue，則可以執行一個賦予它們更高處理優先順序的 Worker：

```shell
php artisan queue:work --queue=high,default
```


<a name="driver-prerequisites"></a>
### 驅動程式說明與先決條件


<a name="database"></a>
#### 資料庫

為了使用 `database` Queue 驅動程式，您需要一個資料庫資料表來存放 Jobs。通常，這已包含在 Laravel 預設的 `0001_01_01_000002_create_jobs_table.php` [資料庫遷移](/docs/{{version}}/migrations)中；然而，如果您的應用程式不包含此遷移，您可以使用 `make:queue-table` Artisan 命令來建立它：

```shell
php artisan make:queue-table

php artisan migrate
```


<a name="redis"></a>
#### Redis

為了使用 `redis` Queue 驅動程式，您應該在 `config/database.php` 設定檔中設定一個 Redis 資料庫連接。

> [!WARNING]
> `redis` Queue 驅動程式不支援 `serializer` 和 `compression` Redis 選項。


<a name="redis-cluster"></a>
##### Redis 叢集

如果您的 Redis Queue 連接使用了 [Redis 叢集](https://redis.io/docs/latest/operate/rs/databases/durability-ha/clustering)，您的 Queue 名稱必須包含一個 [Key Hash Tag](https://redis.io/docs/latest/develop/using-commands/keyspace/#hashtags)。這是為了確保特定 Queue 的所有 Redis Keys 都被放置在同一個 Hash Slot 中：

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

當使用 Redis Queue 時，您可以使用 `block_for` 設定選項來指定驅動程式在循環 Worker 迴圈並重新輪詢 Redis 資料庫之前，應等待 Job 變為可用的時間。

根據您的 Queue 負載調整此值，會比持續輪詢 Redis 資料庫以獲取新 Jobs 更有效率。例如，您可以將該值設定為 `5`，表示驅動程式在等待 Job 變為可用時應阻塞五秒：

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
> 將 `block_for` 設定為 `0` 會導致 Queue Workers 無限期阻塞，直到 Job 可用為止。這也會導致 `SIGTERM` 等訊號在下一個 Job 被處理之前無法被處理。


<a name="other-driver-prerequisites"></a>
#### 其他驅動程式先決條件

列出的 Queue 驅動程式需要以下依賴項目。這些依賴項目可以透過 Composer 套件管理器安裝：

<div class="content-list" markdown="1">

- Amazon SQS: `aws/aws-sdk-php ~3.0`
- Beanstalkd: `pda/pheanstalk ~5.0`
- Redis: `predis/predis ~2.0` 或 phpredis PHP 擴充功能
- [MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/queues/): `mongodb/laravel-mongodb`

</div>

<a name="creating-jobs"></a>
## 建立 Jobs


<a name="generating-job-classes"></a>
### 產生 Job 類別

預設情況下，應用程式中所有可佇列的 Jobs 都儲存在 `app/Jobs` 目錄中。如果 `app/Jobs` 目錄不存在，則會在您執行 `make:job` Artisan 命令時建立：

```shell
php artisan make:job ProcessPodcast
```

產生的類別將實作 `Illuminate\Contracts\Queue\ShouldQueue` 介面，這向 Laravel 表明該 Job 應該被推送到 Queue 中以非同步執行。

> [!NOTE]
> Job Stub 可以使用 [Stub 發佈](/docs/{{version}}/artisan#stub-customization)進行自訂。


<a name="class-structure"></a>
### 類別結構

Job 類別非常簡單，通常只包含一個在 Queue 處理 Job 時會被調用的 `handle` 方法。首先，讓我們看一個 Job 類別的範例。在這個範例中，假設我們管理一個播客 (Podcast) 發佈服務，並且需要在上傳的播客檔案發佈之前對其進行處理：

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

在這個範例中，請注意我們可以將 Eloquent Model 直接傳遞給佇列 Job 的建構函式。由於 Job 使用了 `Queueable` trait，Eloquent Model 及其已載入的關聯在 Job 處理時會被優雅地序列化與反序列化。

如果您的佇列 Job 在建構函式中接收一個 Eloquent Model，則只有該 Model 的識別碼會被序列化到 Queue 中。當 Job 實際被處理時，Queue 系統會自動從資料庫中重新取得完整的 Model 實例及其已載入的關聯。這種 Model 序列化方法可以讓發送到 Queue 驅動程式的 Job Payload 變得更小。


<a name="handle-method-dependency-injection"></a>
#### `handle` 方法的依賴注入

`handle` 方法在 Queue 處理 Job 時被調用。請注意，我們可以在 Job 的 `handle` 方法中對依賴項進行型別提示 (Type-hint)。Laravel [服務容器 (Service Container)](/docs/{{version}}/container) 會自動注入這些依賴項。

如果您想完全控制容器如何將依賴項注入 `handle` 方法，可以使用容器的 `bindMethod` 方法。`bindMethod` 方法接收一個回呼 (Callback)，該回呼接收 Job 與容器。在回呼中，您可以隨意調用 `handle` 方法。通常，您應該在 `App\Providers\AppServiceProvider` [服務提供者](/docs/{{version}}/providers)的 `boot` 方法中調用此方法：

```php
use App\Jobs\ProcessPodcast;
use App\Services\AudioProcessor;
use Illuminate\Contracts\Foundation\Application;

$this->app->bindMethod([ProcessPodcast::class, 'handle'], function (ProcessPodcast $job, Application $app) {
    return $job->handle($app->make(AudioProcessor::class));
});
```

> [!WARNING]
> 二進位資料（如原始圖片內容）在傳遞給佇列 Job 之前，應先通過 `base64_encode` 函式處理。否則，Job 在放入 Queue 時可能無法正確地序列化為 JSON。


<a name="handling-relationships"></a>
#### 佇列中的關聯

由於所有已載入的 Eloquent Model 關聯在 Job 進入佇列時也會被序列化，序列化後的 Job 字串有時會變得很長。此外，當 Job 被反序列化並從資料庫重新取得 Model 關聯時，它們將會被完整地取得。在 Job 佇列化過程中、Model 被序列化之前所套用的任何先前關聯約束，在 Job 被反序列化時都不會被套用。因此，如果您希望處理特定關聯的子集，則應在佇列 Job 中重新約束該關聯。

或者，為了防止關聯被序列化，您可以在設定屬性值時在 Model 上調用 `withoutRelations` 方法。此方法將回傳一個不含已載入關聯的 Model 實例：

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

如果您正在使用 [PHP 建構函式屬性提升 (Constructor Property Promotion)](https://www.php.net/manual/en/language.oop5.decon.php#language.oop5.decon.constructor.promotion)，並且希望指示 Eloquent Model 不應序列化其關聯，您可以使用 `WithoutRelations` 屬性 (Attribute)：

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

為了方便起見，如果您希望將所有 Model 在不含關聯的情況下進行序列化，您可以將 `WithoutRelations` 屬性套用於整個類別，而不是套用於每個 Model：

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

如果 Job 接收的是 Eloquent Model 的集合 (Collection) 或陣列，而不是單一 Model，則在 Job 被反序列化並執行時，該集合中的 Model 將不會恢復其關聯。這是為了防止在處理大量 Model 的 Job 時消耗過多資源。

<a name="unique-jobs"></a>
### Unique Jobs

> [!WARNING]
> Unique jobs 需要一個支援 [鎖定 (locks)](/docs/{{version}}/cache#atomic-locks) 的快取驅動程式。目前 `memcached`、`redis`、`dynamodb`、`database`、`file` 與 `array` 快取驅動程式皆支援不可分割鎖定 (atomic locks)。

> [!WARNING]
> Unique job 限制不適用於批次 (batches) 內的 Jobs。

有時您可能希望確保在任何時間點，佇列中只有一個特定 Job 的實例。您可以透過在 Job 類別中實作 `ShouldBeUnique` 介面來做到這一點。此介面不需要您在類別中定義任何額外的方法：

```php
<?php

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Queue\ShouldBeUnique;

class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    // ...
}
```

在上面的範例中，`UpdateSearchIndex` Job 是唯一的。因此，如果該 Job 的另一個實例已在佇列中且尚未完成處理，則不會派送該 Job。

在某些情況下，您可能希望定義一個使 Job 唯一的特定「鍵 (key)」，或者您可能希望指定一個超時時間，超過該時間後 Job 將不再保持唯一。為此，您可以在 Job 類別中定義 `uniqueId` 和 `uniqueFor` 屬性或方法：

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

在上面的範例中，`UpdateSearchIndex` Job 透過產品 ID 保持唯一。因此，在現有 Job 完成處理之前，任何具有相同產品 ID 的新派送 Job 都將被忽略。此外，如果現有 Job 在一小時內未處理完成，唯一鎖定將被釋放，另一個具有相同唯一鍵的 Job 即可被派送到佇列。

> [!WARNING]
> 如果您的應用程式從多個網頁伺服器或容器派送 Jobs，您應該確保所有伺服器都在與同一個中央快取伺服器通訊，以便 Laravel 可以精確判斷 Job 是否唯一。


<a name="keeping-jobs-unique-until-processing-begins"></a>
#### Keeping Jobs Unique Until Processing Begins

預設情況下，Unique Jobs 會在 Job 完成處理或所有重試嘗試皆失敗後「解鎖」。然而，在某些情況下，您可能希望您的 Job 在開始處理前立即解鎖。為此，您的 Job 應該實作 `ShouldBeUniqueUntilProcessing` 合約，而不是 `ShouldBeUnique` 合約：

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
#### Unique Job Locks

在幕後，當一個 `ShouldBeUnique` 的 Job 被派送時，Laravel 會嘗試獲取一個帶有 `uniqueId` 鍵的 [鎖定 (lock)](/docs/{{version}}/cache#atomic-locks)。如果鎖定已被持有，則不會派送該 Job。當 Job 完成處理或所有重試嘗試皆失敗時，此鎖定會被釋放。預設情況下，Laravel 將使用預設的快取驅動程式來獲取此鎖定。但是，如果您希望使用另一個驅動程式來獲取鎖定，您可以定義一個 `uniqueVia` 方法來回傳應該使用的快取驅動程式：

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
### 加密的 Jobs

Laravel 允許您透過 [加密](/docs/{{version}}/encryption) 來確保 Job 資料的隱私與完整性。要開始使用，只需將 `ShouldBeEncrypted` 介面新增至 Job 類別即可。一旦此介面被新增至類別，Laravel 將在將您的 Job 推送到佇列之前自動對其進行加密：

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

Job 中介層允許您在排隊的 Job 執行周圍封裝自訂邏輯，從而減少 Job 本身的樣板程式碼。例如，考慮以下 `handle` 方法，它利用 Laravel 的 Redis 速率限制功能，每五秒僅允許處理一個 Job：

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

雖然這段程式碼是有效的，但 `handle` 方法的實作會變得雜亂，因為它充斥著 Redis 的速率限制邏輯。此外，我們必須為任何其他想要限制速率的 Job 複製這段速率限制邏輯。與其在 handle 方法中進行速率限制，我們可以定義一個處理速率限制的 Job 中介層：

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

如您所見，就像[路由中介層](/docs/{{version}}/middleware)一樣，Job 中介層接收正在處理的 Job 以及一個應被呼叫以繼續處理該 Job 的回呼 (Callback)。

您可以使用 `make:job-middleware` Artisan 命令產生新的 Job 中介層類別。建立 Job 中介層後，可以透過 Job 的 `middleware` 方法回傳它們，將其附加到 Job 上。此方法在透過 `make:job` Artisan 命令建構的 Job 中並不存在，因此您需要手動將其新增到您的 Job 類別中：

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
> Job 中介層也可以分配給[可排隊的事件監聽器](/docs/{{version}}/events#queued-event-listeners)、[Mailables](/docs/{{version}}/mail#queueing-mail) 和[通知](/docs/{{version}}/notifications#queueing-notifications)。


<a name="rate-limiting"></a>
### 速率限制

雖然我們剛剛示範了如何編寫自己的速率限制 Job 中介層，但 Laravel 實際上包含了一個您可以用來限制 Job 速率的中介層。就像[路由速率限制器](/docs/{{version}}/routing#defining-rate-limiters)一樣，Job 速率限制器是使用 `RateLimiter` Facade 的 `for` 方法定義的。

例如，您可能希望允許使用者每小時備份一次數據，同時對高級客戶不設此限制。要達成此目的，您可以在 `AppServiceProvider` 的 `boot` 方法中定義一個 `RateLimiter`：

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

在上面的範例中，我們定義了每小時的速率限制；然而，您可以使用 `perMinute` 方法輕鬆定義基於分鐘的速率限制。此外，您可以將任何您希望的值傳遞給速率限制的 `by` 方法；但是，此值最常用於按客戶劃分速率限制：

```php
return Limit::perMinute(50)->by($job->user->id);
```

定義速率限制後，您可以使用 `Illuminate\Queue\Middleware\RateLimited` 中介層將速率限制器附加到您的 Job。每當 Job 超過速率限制時，此中介層將根據速率限制持續時間，以適當的延遲將 Job 釋放回 Queue：

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

將受速率限制的 Job 釋放回 Queue 仍會增加 Job 的總 `attempts` 次數。您可能需要相應地調整 Job 類別上的 `tries` 和 `maxExceptions` 屬性。或者，您可能希望使用 [retryUntil 方法](#time-based-attempts)來定義在不再嘗試該 Job 之前所經過的時間。

使用 `releaseAfter` 方法，您還可以指定在再次嘗試已釋放的 Job 之前必須經過的秒數：

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

如果您不希望在 Job 受到速率限制時重試，可以使用 `dontRelease` 方法：

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
#### 使用 Redis 的速率限制

如果您正在使用 Redis，則可以使用 `Illuminate\Queue\Middleware\RateLimitedWithRedis` 中介層，它針對 Redis 進行了微調，並且比基礎的速率限制中介層更有效率：

```php
use Illuminate\Queue\Middleware\RateLimitedWithRedis;

public function middleware(): array
{
    return [new RateLimitedWithRedis('backups')];
}
```

`connection` 方法可用於指定中介層應使用的 Redis 連接：

```php
return [(new RateLimitedWithRedis('backups'))->connection('limiter')];
```

<a name="preventing-job-overlaps"></a>
### 防止 Job 重疊

Laravel 包含了一個 `Illuminate\Queue\Middleware\WithoutOverlapping` 中介層，讓您可以根據任意金鑰防止 Job 重疊。這在 Queue 中的 Job 正在修改應當一次僅由一個 Job 修改的資源時非常有用。

例如，假設您有一個更新使用者信用評分的 Queue Job，且您希望防止同一個使用者 ID 的信用評分更新 Job 重疊。為此，您可以從 Job 的 `middleware` 方法回傳 `WithoutOverlapping` 中介層：

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

將重疊的 Job 重新釋放回 Queue 仍會增加 Job 的總嘗試次數。您可能需要相應地調整 Job 類別上的 `tries` 和 `maxExceptions` 屬性。例如，將 `tries` 屬性保持為預設值 1，將會防止任何重疊的 Job 在稍後重試。

任何相同類型的重疊 Job 都將被釋放回 Queue。您也可以指定在再次嘗試已釋放的 Job 之前必須經過的秒數：

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

`WithoutOverlapping` 中介層由 Laravel 的原子鎖功能驅動。有時，您的 Job 可能會意外失敗或超時，導致鎖定未被釋放。因此，您可以使用 `expireAfter` 方法明確定義鎖定過期時間。例如，下方的範例將指示 Laravel 在 Job 開始處理三分鐘後釋放 `WithoutOverlapping` 鎖定：

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
> `WithoutOverlapping` 中介層需要支援 [鎖定](/docs/{{version}}/cache#atomic-locks) 的快取驅動程式。目前 `memcached`、`redis`、`dynamodb`、`database`、`file` 和 `array` 快取驅動程式皆支援原子鎖。

<a name="sharing-lock-keys"></a>
#### 跨 Job 類別共用鎖定金鑰

預設情況下，`WithoutOverlapping` 中介層僅會防止相同類別的 Job 重疊。因此，儘管兩個不同的 Job 類別使用相同的鎖定金鑰，它們也不會被防止重疊。然而，您可以使用 `shared` 方法指示 Laravel 跨 Job 類別套用金鑰：

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
### 節流異常

Laravel 包含了一個 `Illuminate\Queue\Middleware\ThrottlesExceptions` 中介層，讓您可以對異常進行節流。一旦 Job 拋出特定次數的異常，後續所有執行該 Job 的嘗試都將延遲，直到指定的間隔時間過去為止。此中介層對於與不穩定第三方服務互動的 Job 特別有用。

例如，想像一個與第三方 API 互動的佇列 Job 開始拋出異常。為了對異常進行節流，您可以從 Job 的 `middleware` 方法中回傳 `ThrottlesExceptions` 中介層。通常，此中介層應與實作了 [基於時間的嘗試](#time-based-attempts) 的 Job 搭配使用：

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

中介層接受的第一個建構子參數是 Job 在被節流前可以拋出的異常次數，而第二個建構子參數則是 Job 被節流後，再次嘗試執行前應經過的秒數。在上面的程式碼範例中，如果 Job 連續拋出 10 次異常，我們將等待 5 分鐘後再嘗試執行 Job，並受限於 30 分鐘的時間限制。

當 Job 拋出異常但尚未達到異常閾值時，Job 通常會立即重試。然而，您可以在將中介層附加到 Job 時，透過呼叫 `backoff` 方法來指定此類 Job 應延遲的分鐘數：

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

在內部，此中介層使用 Laravel 的快取系統來實作速率限制，並以 Job 的類別名稱作為快取「鍵 (key)」。您可以在將中介層附加到 Job 時，透過呼叫 `by` 方法來覆蓋此鍵。如果您有多個與同一個第三方服務互動的 Job，且希望它們共享同一個節流「貯槽 (bucket)」以確保它們遵循單一共享限制，這將非常有用：

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

預設情況下，此中介層會對每個異常進行節流。您可以在將中介層附加到 Job 時呼叫 `when` 方法來修改此行為。只有在提供給 `when` 方法的閉包傳回 `true` 時，異常才會被節流：

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

與會將 Job 釋放回 Queue 或拋出異常的 `when` 方法不同，`deleteWhen` 方法讓您可以在發生特定異常時完全刪除 Job：

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

如果您希望將被節流的異常回報給應用程式的異常處理器，您可以在將中介層附加到 Job 時呼叫 `report` 方法。或者，您可以提供一個閉包給 `report` 方法，只有當給定的閉包傳回 `true` 時，異常才會被回報：

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
#### 使用 Redis 進行異常節流

如果您正在使用 Redis，可以使用 `Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis` 中介層，它針對 Redis 進行了微調，比基本的異常節流中介層更有效率：

```php
use Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis;

public function middleware(): array
{
    return [new ThrottlesExceptionsWithRedis(10, 10 * 60)];
}
```

`connection` 方法可用於指定中介層應使用的 Redis 連接：

```php
return [(new ThrottlesExceptionsWithRedis(10, 10 * 60))->connection('limiter')];
```

<a name="skipping-jobs"></a>
### 跳過 Jobs

`Skip` 中介層讓您可以指定應跳過 / 刪除 Job，而無需修改 Job 的邏輯。若給定條件評估為 `true`，`Skip::when` 方法將會刪除該 Job；而若條件評估為 `false`，`Skip::unless` 方法則會刪除該 Job：

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
## 派送 Jobs

一旦編寫完 Job 類別，您可以使用 Job 本身的 `dispatch` 方法來派送它。傳遞給 `dispatch` 方法的參數將被提供給 Job 的建構函式：

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

如果您想要有條件地派送 Job，可以使用 `dispatchIf` 和 `dispatchUnless` 方法：

```php
ProcessPodcast::dispatchIf($accountActive, $podcast);

ProcessPodcast::dispatchUnless($accountSuspended, $podcast);
```

在新的 Laravel 應用程式中，`database` 連接被定義為預設的 Queue。您可以透過更改應用程式 `.env` 檔案中的 `QUEUE_CONNECTION` 環境變數來指定不同的預設 Queue 連接。


<a name="delayed-dispatching"></a>
### 延遲派送

如果您想要指定一個 Job 不應立即供 Queue Worker 處理，可以在派送 Job 時使用 `delay` 方法。例如，讓我們指定一個 Job 在派送 10 分鐘後才能處理：

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

在某些情況下，Jobs 可能設定了預設的延遲。如果您需要繞過此延遲並派送 Job 進行立即處理，可以使用 `withoutDelay` 方法：

```php
ProcessPodcast::dispatch($podcast)->withoutDelay();
```

> [!WARNING]
> Amazon SQS Queue 服務的最大延遲時間為 15 分鐘。


<a name="synchronous-dispatching"></a>
### 同步派送

如果您想立即 (同步) 派送 Job，可以使用 `dispatchSync` 方法。使用此方法時，Job 不會進入 Queue，而是會在當前程序中立即執行：

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
#### 延後派送

使用延後同步派送，您可以將 Job 派送為在當前程序中處理，但在 HTTP 回應發送給使用者之後執行。這讓您可以同步處理「已排隊」的 Job，而不會減慢使用者的應用程式體驗。若要延後執行同步 Job，請將 Job 派送到 `deferred` 連接：

```php
RecordDelivery::dispatch($order)->onConnection('deferred');
```

`deferred` 連接也可作為預設的 [Queue 故障轉移](#queue-failover)。

同樣地，`background` 連接會在 HTTP 回應發送給使用者後處理 Jobs；然而，Job 會在一個單獨生成的 PHP 程序中處理，讓 PHP-FPM / 應用程式 Worker 可以用於處理另一個傳入的 HTTP 請求：

```php
RecordDelivery::dispatch($order)->onConnection('background');
```


<a name="jobs-and-database-transactions"></a>
### Jobs 與資料庫交易

雖然在資料庫交易中派送 Jobs 是完全可以的，但您應該特別注意確保您的 Job 實際上能夠成功執行。在交易中派送 Job 時，Job 可能會在父交易提交之前就被 Worker 處理。當這種情況發生時，您在資料庫交易期間對 Models 或資料庫紀錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何 Models 或資料庫紀錄可能在資料庫中尚不存在。

幸好，Laravel 提供了幾種解決此問題的方法。首先，您可以在 Queue 連接的設定陣列中設定 `after_commit` 連接選項：

```php
'redis' => [
    'driver' => 'redis',
    // ...
    'after_commit' => true,
],
```

當 `after_commit` 選項為 `true` 時，您可以在資料庫交易內派送 Jobs；然而，Laravel 會等到開啟的父資料庫交易被提交後，才實際派送該 Job。當然，如果目前沒有開啟任何資料庫交易，Job 將會立即派送。

如果交易因交易期間發生的異常而回滾，在該交易期間派送的 Jobs 將被捨棄。

> [!NOTE]
> 將 `after_commit` 設定選項設為 `true` 也會導致任何已排隊的事件監聽器、Mailables、通知和廣播事件在所有開啟的資料庫交易提交後才被派送。


<a name="specifying-commit-dispatch-behavior-inline"></a>
#### 內嵌指定提交派送行為

如果您未將 `after_commit` Queue 連接設定選項設為 `true`，您仍然可以指示特定 Job 應在所有開啟的資料庫交易提交後才被派送。若要實現此目的，您可以在派送操作中鏈接 `afterCommit` 方法：

```php
use App\Jobs\ProcessPodcast;

ProcessPodcast::dispatch($podcast)->afterCommit();
```

同理，如果 `after_commit` 設定選項被設為 `true`，您可以指示特定 Job 應立即派送，而不必等待任何開啟的資料庫交易提交：

```php
ProcessPodcast::dispatch($podcast)->beforeCommit();
```

<a name="job-chaining"></a>
### Job 鏈接

Job 鏈接允許你指定一連串的 Queue Jobs，這些 Jobs 將在主要 Job 執行成功後依序執行。如果序列中的其中一個 Job 失敗，剩餘的 Jobs 將不會執行。要執行一個 Queue Job 鏈接，你可以使用 `Bus` Facade 提供的 `chain` 方法。Laravel 的 Command Bus 是一個底層元件，Queue Job 的派送功能就是建立在其之上：

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

除了鏈接 Job 類別的實體，你也可以鏈接閉包 (Closures)：

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
> 使用 Job 內的 `$this->delete()` 方法刪除 Job 並不會阻止鏈接的 Job 被處理。只有當鏈接中的 Job 失敗時，鏈接才會停止執行。


<a name="chain-connection-queue"></a>
#### 鏈接連接與 Queue

如果你想指定用於鏈接 Jobs 的連接與 Queue，可以使用 `onConnection` 與 `onQueue` 方法。除非該 Queue Job 被明確分配了不同的連接或 Queue，否則這些方法會指定要使用的 Queue 連接與 Queue 名稱：

```php
Bus::chain([
    new ProcessPodcast,
    new OptimizePodcast,
    new ReleasePodcast,
])->onConnection('redis')->onQueue('podcasts')->dispatch();
```


<a name="adding-jobs-to-the-chain"></a>
#### 將 Jobs 新增至鏈接

有時，你可能需要在鏈接中的某個 Job 內，將一個 Job 加到現有鏈接的前端或末端。你可以使用 `prependToChain` 與 `appendToChain` 方法來達成此目的：

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

鏈接 Jobs 時，你可以使用 `catch` 方法指定一個閉包，當鏈接中的 Job 失敗時會叫用該閉包。給定的回呼 (Callback) 將會接收到導致 Job 失敗的 `Throwable` 實體：

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
> 由於鏈接的回呼會被序列化並在稍後由 Laravel Queue 執行，因此你不應該在鏈接回呼中使用 `$this` 變數。


<a name="customizing-the-queue-and-connection"></a>
### 自訂 Queue 與連接


<a name="dispatching-to-a-particular-queue"></a>
#### 派送至特定的 Queue

透過將 Jobs 推送到不同的 Queues，你可以對 Queue Jobs 進行「分類」，甚至可以根據優先級為各個 Queues 分配不同數量的 Worker。請記住，這並不會將 Jobs 推送到你的 Queue 設定檔中定義的不同 Queue 「連接 (Connections)」，而只是推送到單一連接中的特定 Queues。若要指定 Queue，請在派送 Job 時使用 `onQueue` 方法：

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

或者，你也可以透過在 Job 的建構子中呼叫 `onQueue` 方法來指定 Job 的 Queue：

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
#### 派送至特定的連接

如果你的應用程式與多個 Queue 連接互動，你可以使用 `onConnection` 方法指定要將 Job 推送到哪個連接：

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

你可以將 `onConnection` 與 `onQueue` 方法鏈接在一起，以指定 Job 的連接與 Queue：

```php
ProcessPodcast::dispatch($podcast)
    ->onConnection('sqs')
    ->onQueue('processing');
```

或者，你也可以透過在 Job 的建構子中呼叫 `onConnection` 方法來指定 Job 的連接：

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
### 指定最大 Job 嘗試次數 / 超時值


<a name="max-attempts"></a>
#### 最大嘗試次數

Job 嘗試是 Laravel Queue 系統的核心概念，並支援許多進階功能。雖然一開始可能會令人困惑，但在修改預設設定之前，瞭解它們的工作原理非常重要。

當 Job 被派送時，它會被推送到 Queue 中。接著 Worker 會取得它並嘗試執行。這就是一次 Job 嘗試。

然而，一次嘗試不一定代表 Job 的 `handle` 方法已經執行。嘗試也可能透過以下幾種方式被「消耗」：

<div class="content-list" markdown="1">

- Job 在執行期間遇到未處理的異常。
- Job 使用 `$this->release()` 手動釋放回 Queue。
- 如 `WithoutOverlapping` 或 `RateLimited` 等中介層無法取得鎖定並釋放 Job。
- Job 超時。
- Job 的 `handle` 方法執行並完成，且未拋出異常。

</div>

您可能不想無限期地持續嘗試某個 Job。因此，Laravel 提供了多種方式來指定 Job 可以嘗試的次數或時間長度。

> [!NOTE]
> 預設情況下，Laravel 只會嘗試一次 Job。如果您 Job 使用了如 `WithoutOverlapping` 或 `RateLimited` 之類的中介層，或是您正在手動釋放 Job，您可能需要透過 `tries` 選項來增加允許的嘗試次數。

指定 Job 最大嘗試次數的一種方法是透過 Artisan 命令列上的 `--tries` 切換參數。這將適用於 Worker 處理的所有 Job，除非正在處理的 Job 自行指定了可嘗試的次數：

```shell
php artisan queue:work --tries=3
```

如果 Job 超過其最大嘗試次數，它將被視為「失敗」的 Job。有關處理失敗 Job 的更多資訊，請參閱 [失敗 Job 文件](#dealing-with-failed-jobs)。如果為 `queue:work` 命令提供 `--tries=0`，Job 將無限期地重試。

您可以透過在 Job 類別本身定義 Job 最大嘗試次數，來採取更細粒度的方法。如果在 Job 上指定了最大嘗試次數，它將優先於命令列上提供的 `--tries` 值：

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

如果您需要動態控制特定 Job 的最大嘗試次數，可以在 Job 中定義 `tries` 方法：

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
#### 基於時間的嘗試

除了定義 Job 在失敗前可以嘗試的次數外，您還可以定義 Job 不應再被嘗試的時間點。這允許 Job 在指定的時限內嘗試任意次數。要定義 Job 不應再被嘗試的時間，請在您的 Job 類別中新增 `retryUntil` 方法。此方法應回傳一個 `DateTime` 實例：

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
> 您也可以在 [queued event listeners](/docs/{{version}}/events#queued-event-listeners) 與 [queued notifications](/docs/{{version}}/notifications#queueing-notifications) 上定義 `tries` 屬性或 `retryUntil` 方法。


<a name="max-exceptions"></a>
#### 最大異常次數

有時您可能希望指定 Job 可以嘗試多次，但如果重試是由特定數量的未處理異常觸發的（而不是直接透過 `release` 方法釋放），則該 Job 應失敗。要達成此目的，您可以在 Job 類別中定義 `maxExceptions` 屬性：

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

在此範例中，如果應用程式無法取得 Redis 鎖定， Job 將釋放 10 秒，並將繼續重試最多 25 次。然而，如果 Job 拋出了三個未處理的異常，該 Job 將會失敗。


<a name="timeout"></a>
#### 超時

通常，您大致知道您預期佇列 Job 所花費的時間。因此，Laravel 允許您指定一個「超時」值。預設情況下，超時值為 60 秒。如果 Job 處理的時間超過超時值指定的秒數，處理該 Job 的 Worker 將會出錯並退出。通常，Worker 會由您 [伺服器上設定的程序管理員](#supervisor-configuration) 自動重啟。

Job 可以執行的最大秒數可以使用 Artisan 命令列上的 `--timeout` 切換參數來指定：

```shell
php artisan queue:work --timeout=30
```

如果 Job 因為持續超時而超過其最大嘗試次數，它將被標記為失敗。

您也可以在 Job 類別本身定義 Job 允許執行的最大秒數。如果在 Job 上指定了超時值，它將優先於命令列上指定的任何超時值：

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

有時，通訊端 (Socket) 或傳出的 HTTP 連接等 IO 阻塞程序可能不會遵守您指定的超時值。因此，在使用這些功能時，您應該始終嘗試使用它們的 API 來指定超時值。例如，使用 [Guzzle](https://docs.guzzlephp.org) 時，您應該始終指定連接和請求的超時值。

> [!WARNING]
> 必須安裝 [PCNTL](https://www.php.net/manual/en/book.pcntl.php) PHP 擴充功能才能指定 Job 超時。此外，Job 的「超時 (timeout)」值應始終小於其 [「重試等待時間 (retry after)」](#job-expiration) 值。否則，Job 可能會在實際完成執行或超時之前就重新嘗試。


<a name="failing-on-timeout"></a>
#### 超時時失敗

如果您想指示 Job 在超時時應標記為 [失敗](#dealing-with-failed-jobs)，您可以在 Job 類別中定義 `$failOnTimeout` 屬性：

```php
/**
 * Indicate if the job should be marked as failed on timeout.
 *
 * @var bool
 */
public $failOnTimeout = true;
```

> [!NOTE]
> 預設情況下，當 Job 超時時，它會消耗一次嘗試並被釋放回 Queue (如果允許重試)。然而，如果您將 Job 設定為在超時時失敗，則無論 tries 設定為何，它都不會再重試。

<a name="sqs-fifo-and-fair-queues"></a>
### SQS FIFO 與公平 Queues

Laravel 支援 [Amazon SQS FIFO (先進先出)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html) Queues，讓您可以按照發送的確切順序處理 Job，同時透過訊息去重 (Deduplication) 確保精確一次 (Exactly-once) 的處理。

FIFO Queue 需要訊息群組 ID (Message Group ID) 來判斷哪些 Job 可以平行處理。具有相同群組 ID 的 Job 會依序處理，而具有不同群組 ID 的訊息則可以併發處理。

Laravel 提供了一個流暢的 `onGroup` 方法，用於在派送 Job 時指定訊息群組 ID：

```php
ProcessOrder::dispatch($order)
    ->onGroup("customer-{$order->customer_id}");
```

SQS FIFO Queue 支援訊息去重，以確保精確一次的處理。請在您的 Job 類別中實作 `deduplicationId` 方法以提供自訂的去重 ID：

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

使用 FIFO Queue 時，您也需要在監聽器、郵件與通知上定義訊息群組。或者，您可以將這些物件的佇列實例派送到非 FIFO 的 Queue。

若要為 [Queue 事件監聽器](/docs/{{version}}/events#queued-event-listeners) 定義訊息群組，請在監聽器中定義 `messageGroup` 方法。您也可以選擇性地定義 `deduplicationId` 方法：

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

當發送要排入 FIFO Queue 的 [郵件訊息](/docs/{{version}}/mail) 時，您應該在發送通知時呼叫 `onGroup` 方法，並選擇性地呼叫 `withDeduplicator` 方法：

```php
use App\Mail\InvoicePaid;
use Illuminate\Support\Facades\Mail;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

Mail::to($request->user())->send($invoicePaid);
```

當發送要排入 FIFO Queue 的 [通知](/docs/{{version}}/notifications) 時，您應該在發送通知時呼叫 `onGroup` 方法，並選擇性地呼叫 `withDeduplicator` 方法：

```php
use App\Notifications\InvoicePaid;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

$user->notify($invoicePaid);
```

<a name="queue-failover"></a>
### Queue 故障轉移

`failover` Queue 驅動程式提供在將 Job 推送到 Queue 時的自動故障轉移功能。如果 `failover` 設定中的主要 Queue 連接因任何原因失敗，Laravel 會自動嘗試將 Job 推送到清單中的下一個已設定連接。這對於確保在 Queue 可靠性至關重要的正式環境中的高可用性特別有用。

若要設定故障轉移 Queue 連接，請指定 `failover` 驅動程式並提供一個要按順序嘗試的連接名稱陣列。預設情況下，Laravel 在您的應用程式 `config/queue.php` 設定檔中包含了一個故障轉移設定範例：

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

設定好使用 `failover` 驅動程式的連接後，您需要將應用程式 `.env` 檔案中的預設 Queue 連接設定為該故障轉移連接，以便使用故障轉移功能：

```ini
QUEUE_CONNECTION=failover
```

接著，為故障轉移連接清單中的每個連接啟動至少一個 Worker：

```bash
php artisan queue:work redis
php artisan queue:work database
```

> [!NOTE]
> 您不需要為使用 `sync`、`background` 或 `deferred` Queue 驅動程式的連接執行 Worker，因為這些驅動程式會在當前的 PHP 程序中處理 Job。

當 Queue 連接操作失敗且啟動故障轉移時，Laravel 會派送 `Illuminate\Queue\Events\QueueFailedOver` 事件，讓您可以回報或記錄 Queue 連接失敗。

> [!NOTE]
> 如果您使用 Laravel Horizon，請記住 Horizon 僅管理 Redis Queue。如果您的故障轉移清單包含 `database`，您應該在 Horizon 之外執行一般的 `php artisan queue:work database` 程序。

<a name="error-handling"></a>
### 錯誤處理

若 Job 在處理過程中拋出例外，該 Job 將自動被釋放回 Queue 中，以便再次嘗試。Job 將持續被釋放，直到達到應用程式所允許的最大嘗試次數。最大嘗試次數是透過 `queue:work` Artisan 命令使用的 `--tries` 切換參數來定義。或者，也可以在 Job 類別本身定義最大嘗試次數。更多關於執行 Queue Worker 的資訊[可以在下方找到](#running-the-queue-worker)。


<a name="manually-releasing-a-job"></a>
#### 手動釋放 Job

有時您可能希望手動將 Job 釋放回 Queue，以便稍後再次嘗試。您可以透過呼叫 `release` 方法來完成此操作：

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

預設情況下，`release` 方法會將 Job 釋放回 Queue 以立即進行處理。然而，您可以透過向 `release` 方法傳遞一個整數或日期實例，來指示 Queue 在經過指定的秒數後才讓該 Job 可供處理：

```php
$this->release(10);

$this->release(now()->plus(seconds: 10));
```


<a name="manually-failing-a-job"></a>
#### 手動使 Job 失敗

偶爾您可能需要手動將 Job 標記為「失敗」。若要這樣做，您可以呼叫 `fail` 方法：

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

如果您是因為擷取到例外而想將 Job 標記為失敗，可以將該例外傳遞給 `fail` 方法。或者，為了方便起見，您可以傳遞一個字串錯誤訊息，它將為您轉換為一個例外：

```php
$this->fail($exception);

$this->fail('Something went wrong.');
```

> [!NOTE]
> 關於失敗 Jobs 的更多資訊，請參閱[處理失敗 Jobs 的文件](#dealing-with-failed-jobs)。


<a name="fail-jobs-on-exceptions"></a>
#### 針對特定例外使 Jobs 失敗

`FailOnException` [Job 中介層](#job-middleware) 允許您在拋出特定例外時短路 (Short-circuit) 重試程序。這允許在發生暫時性例外（如外部 API 錯誤）時進行重試，但在發生持續性例外（如使用者的權限被撤銷）時讓 Job 永久失敗：

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
## Job 批次

Laravel 的 Job 批次功能讓您能輕鬆地執行一批 Jobs，並在該批 Jobs 執行完成後執行某些操作。在開始之前，您應該建立一個資料庫遷移來構建一個資料表，該資料表將包含有關 Job 批次的中繼資訊 (Meta Information)，例如其完成百分比。可以使用 `make:queue-batches-table` Artisan 命令產生此遷移：

```shell
php artisan make:queue-batches-table

php artisan migrate
```


<a name="defining-batchable-jobs"></a>
### 定義可批次的 Jobs

要定義一個可批次的 Job，您應該像平常一樣 [建立一個 Queueable Job](#creating-jobs)；但是，您應該將 `Illuminate\Bus\Batchable` trait 新增至該 Job 類別。此 trait 提供了對 `batch` 方法的存取，該方法可用於檢索該 Job 正在其中執行的當前批次：

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
### 派送批次

要派送一批 Jobs，您應該使用 `Bus` Facade 的 `batch` 方法。當然，當批次與完成回呼 (Callback) 結合使用時最有用。因此，您可以使用 `then`、`catch` 和 `finally` 方法來為批次定義完成回呼。當這些回呼被叫用時，每個回呼都會接收一個 `Illuminate\Bus\Batch` 實例。

當執行多個 Queue Workers 時，批次中的 Jobs 將會並行處理。因此，Jobs 完成的順序可能與它們被新增至批次的順序不同。有關如何按順序執行一系列 Job 的資訊，請參閱我們關於 [Job 鏈接與批次](#chains-and-batches) 的文件。

在此範例中，我們假設我們正在排隊一個 Jobs 批次，每個 Job 都處理 CSV 檔案中給定數量的資料列：

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

批次的 ID 可以透過 `$batch->id` 屬性存取，可用於在派送批次後 [向 Laravel 命令匯流排查詢](#inspecting-batches) 有關批次的資訊。

> [!WARNING]
> 由於批次回呼是由 Laravel Queue 序列化並在稍後執行的，因此您不應在回呼中使用 `$this` 變數。此外，由於批次 Jobs 被封裝在資料庫交易中，因此不應在 Jobs 中執行會觸發隱含提交 (Implicit Commits) 的資料庫語句。


<a name="naming-batches"></a>
#### 命名批次

如果為批次命名，某些工具（如 [Laravel Horizon](/docs/{{version}}/horizon) 和 [Laravel Telescope](/docs/{{version}}/telescope)）可能會提供更人性化的除錯資訊。要為批次分配一個任意名稱，您可以在定義批次時呼叫 `name` 方法：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import CSV')->dispatch();
```


<a name="batch-connection-queue"></a>
#### 批次連接與 Queue

如果您想指定批次 Jobs 應使用的連接和 Queue，可以使用 `onConnection` 和 `onQueue` 方法。所有批次 Jobs 必須在同一個連接和 Queue 中執行：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->onConnection('redis')->onQueue('imports')->dispatch();
```


<a name="chains-and-batches"></a>
### 鏈接與批次

您可以透過將鏈接的 Jobs 放在陣列中，在批次中定義一組 [鏈接的 Jobs](#job-chaining)。例如，我們可以並行執行兩個 Job 鏈接，並在兩個 Job 鏈接都完成處理後執行一個回呼：

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

相反地，您可以透過在鏈接中定義批次，在 [鏈接](#job-chaining) 中執行 Jobs 批次。例如，您可以先執行一批 Jobs 來發佈多個 Podcast，然後再執行一批 Jobs 來發送發佈通知：

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
### 將 Jobs 新增至批次

有時，從批次 Job 中向批次新增額外的 Jobs 可能很有用。當您需要批次處理數千個 Jobs，而這些 Jobs 在網頁請求期間派送可能耗時太長時，此模式非常有用。因此，您可能希望派送一個初始的「載入器 (Loader)」Jobs 批次，該批次用更多的 Jobs 來填充 (Hydrate) 該批次：

```php
$batch = Bus::batch([
    new LoadImportBatch,
    new LoadImportBatch,
    new LoadImportBatch,
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import Contacts')->dispatch();
```

在此範例中，我們將使用 `LoadImportBatch` Job 為批次填充額外的 Jobs。要達成此目的，我們可以使用可透過 Job 的 `batch` 方法存取的批次實例上的 `add` 方法：

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
> 您只能從屬於同一批次的 Job 中向批次新增 Jobs。

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

所有 `Illuminate\Bus\Batch` 實例都是 JSON 可序列化的，這意味著您可以直接從應用程式的其中一個路由回傳它們，以取得包含該批次相關資訊的 JSON 酬載 (payload)，包括其完成進度。這使得在應用程式的 UI 中顯示批次完成進度變得非常方便。

若要透過 ID 取得批次，您可以使用 `Bus` facade 的 `findBatch` 方法：

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

正如您在之前的範例中所注意到的，批次 Jobs 通常應該在繼續執行之前確定其對應的批次是否已被取消。然而，為了方便起見，您可以改為將 `SkipIfBatchCancelled` [中介層](#job-middleware) 分配給該 Job。正如其名稱所示，如果對應的批次已被取消，此中介層將指示 Laravel 不要處理該 Job：

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

當批次 Job 失敗時，將會呼叫 `catch` 回呼（如果已分配）。此回呼僅針對批次中第一個失敗的 Job 被呼叫。


<a name="allowing-failures"></a>
#### 允許失敗

當批次中的 Job 失敗時，Laravel 會自動將該批次標記為「已取消」。如果您願意，可以停用此行為，以便 Job 失敗時不會自動將批次標記為已取消。這可以透過在派送批次時呼叫 `allowFailures` 方法來實現：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->allowFailures()->dispatch();
```

您可以選擇為 `allowFailures` 方法提供一個閉包，該閉包將在每次 Job 失敗時執行：

```php
$batch = Bus::batch([
    // ...
])->allowFailures(function (Batch $batch, $exception) {
    // Handle individual job failures...
})->dispatch();
```


<a name="retrying-failed-batch-jobs"></a>
#### 重試失敗的批次 Jobs

為了方便起見，Laravel 提供了 `queue:retry-batch` Artisan 命令，讓您可以輕鬆地重試指定批次的所有失敗 Jobs。此命令接受要重試失敗 Jobs 的批次 UUID：

```shell
php artisan queue:retry-batch 32dbc76c-4f82-4749-b610-a639fe0099b5
```


<a name="pruning-batches"></a>
### 修剪批次

如果不進行修剪，`job_batches` 資料表會非常快速地累積紀錄。為了減輕這種情況，您應該 [排程](/docs/{{version}}/scheduling) `queue:prune-batches` Artisan 命令每天執行：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches')->daily();
```

預設情況下，所有完成超過 24 小時的批次都將被修剪。您可以在呼叫命令時使用 `hours` 選項來決定保留批次資料的時間。例如，以下命令將刪除所有在 48 小時前完成的批次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48')->daily();
```

有時，您的 `job_batches` 資料表可能會累積那些從未成功完成的批次紀錄，例如批次中某個 Job 失敗且該 Job 從未重試成功。您可以指示 `queue:prune-batches` 命令使用 `unfinished` 選項來修剪這些未完成的批次紀錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --unfinished=72')->daily();
```

同樣地，您的 `job_batches` 資料表也可能累積已取消批次的紀錄。您可以指示 `queue:prune-batches` 命令使用 `cancelled` 選項來修剪這些已取消的批次紀錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --cancelled=72')->daily();
```

<a name="storing-batches-in-dynamodb"></a>
### 將批次儲存在 DynamoDB

Laravel 也支援將批次元資訊儲存在 [DynamoDB](https://aws.amazon.com/dynamodb) 中，而非關聯式資料庫。然而，您需要手動建立一個 DynamoDB 資料表來儲存所有的批次紀錄。

通常這個資料表應該命名為 `job_batches`，但您應該根據應用程式 `queue` 設定檔中的 `queue.batching.table` 設定值來命名該資料表。


<a name="dynamodb-batch-table-configuration"></a>
#### DynamoDB 批次資料表設定

`job_batches` 資料表應包含一個名為 `application` 的字串型別主要分割鍵 (Partition Key)，以及一個名為 `id` 的字串型別主要排序鍵 (Sort Key)。該金鑰的 `application` 部分將包含您的應用程式名稱，這定義在應用程式 `app` 設定檔中的 `name` 設定值中。由於應用程式名稱是 DynamoDB 資料表金鑰的一部分，因此您可以使用同一個資料表來儲存多個 Laravel 應用程式的 Job 批次。

此外，如果您想利用 [自動批次修剪](#pruning-batches-in-dynamodb) 功能，您可以為資料表定義 `ttl` 屬性。


<a name="dynamodb-configuration"></a>
#### DynamoDB 設定

接著，安裝 AWS SDK 以便您的 Laravel 應用程式能與 Amazon DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

然後，將 `queue.batching.driver` 設定選項的值設為 `dynamodb`。此外，您應該在 `batching` 設定陣列中定義 `key`、`secret` 以及 `region` 設定選項。這些選項將用於 AWS 的身份驗證。在使用 `dynamodb` 驅動程式時，不需要 `queue.batching.database` 設定選項：

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
#### 在 DynamoDB 中修剪批次

當使用 [DynamoDB](https://aws.amazon.com/dynamodb) 來儲存 Job 批次資訊時，用於修剪儲存在關聯式資料庫中批次的典型修剪命令將無法運作。取而代之的是，您可以利用 [DynamoDB 原生的 TTL 功能](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html) 來自動移除舊批次的紀錄。

如果您為您的 DynamoDB 資料表定義了 `ttl` 屬性，您可以定義設定參數來指示 Laravel 如何修剪批次紀錄。`queue.batching.ttl_attribute` 設定值定義了存放 TTL 的屬性名稱，而 `queue.batching.ttl` 設定值則定義了在最後一次更新紀錄後，經過多少秒後可以從 DynamoDB 資料表中移除批次紀錄：

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
## Queue 閉包

除了將 Job 類別派送至 Queue 之外，您也可以派送一個閉包。這對於需要在目前請求週期之外執行的快速、簡單任務非常有用。將閉包派送至 Queue 時，閉包的程式碼內容會經過加密簽署，以便在傳輸過程中不會被修改：

```php
use App\Models\Podcast;

$podcast = Podcast::find(1);

dispatch(function () use ($podcast) {
    $podcast->publish();
});
```

若要為排入 Queue 的閉包指定一個名稱，以便於 Queue 報告儀表板中使用，並顯示在 `queue:work` 命令中，您可以使用 `name` 方法：

```php
dispatch(function () {
    // ...
})->name('Publish Podcast');
```

透過使用 `catch` 方法，您可以提供一個閉包，當排入 Queue 的閉包在耗盡所有 Queue [設定的重試嘗試次數](#max-job-attempts-and-timeout) 後仍無法成功完成時，該閉包將被執行：

```php
use Throwable;

dispatch(function () use ($podcast) {
    $podcast->publish();
})->catch(function (Throwable $e) {
    // This job has failed...
});
```

> [!WARNING]
> 由於 `catch` 回呼會被序列化並在稍後由 Laravel Queue 執行，因此您不應在 `catch` 回呼內使用 `$this` 變數。

<a name="running-the-queue-worker"></a>
## 執行 Queue Worker


<a name="the-queue-work-command"></a>
### `queue:work` 命令

Laravel 包含一個 Artisan 命令，它將啟動一個 Queue Worker 並處理被推送到 Queue 的新 Jobs。您可以使用 `queue:work` Artisan 命令來執行 Worker。請注意，一旦 `queue:work` 命令啟動後，它將持續執行，直到被手動停止或您關閉終端機為止：

```shell
php artisan queue:work
```

> [!NOTE]
> 若要讓 `queue:work` 進程永久地在背景執行，您應該使用如 [Supervisor](#supervisor-configuration) 之類的進程監控器，以確保 Queue Worker 不會停止運作。

如果您希望在命令輸出中包含已處理的 Job ID、連接名稱和 Queue 名稱，可以在調用 `queue:work` 命令時加入 `-v` 旗標：

```shell
php artisan queue:work -v
```

請記住，Queue Workers 是長駐型進程，會將啟動後的應用程式狀態儲存在記憶體中。因此，在它們啟動後，它們將不會察覺到您程式碼庫中的變動。所以，在您的佈署過程中，請務必 [重新啟動您的 Queue Workers](#queue-workers-and-deployment)。此外，請記住，應用程式所建立或修改的任何靜態狀態都不會在 Job 之間自動重置。

或者，您也可以執行 `queue:listen` 命令。使用 `queue:listen` 命令時，當您想要重新載入更新後的程式碼或重置應用程式狀態時，不需要手動重新啟動 Worker；但是，此命令的效率明顯低於 `queue:work` 命令：

```shell
php artisan queue:listen
```


<a name="running-multiple-queue-workers"></a>
#### 執行多個 Queue Workers

要為一個 Queue 分配多個 Workers 並並行處理 Jobs，您只需啟動多個 `queue:work` 進程即可。這可以透過在終端機中開啟多個分頁在本地完成，或者在正式環境中使用進程管理器的設定來完成。[當使用 Supervisor 時](#supervisor-configuration)，您可以使用 `numprocs` 設定值。


<a name="specifying-the-connection-queue"></a>
#### 指定連接與 Queue

您也可以指定 Worker 應該利用哪個 Queue 連接。傳遞給 `work` 命令的連接名稱應對應於 `config/queue.php` 設定檔中定義的其中一個連接：

```shell
php artisan queue:work redis
```

預設情況下，`queue:work` 命令僅處理給定連接上預設 Queue 的 Jobs。但是，您可以透過僅處理給定連接的特定 Queues 來進一步自訂您的 Queue Worker。例如，如果您所有的電子郵件都在 `redis` Queue 連接的 `emails` Queue 中處理，則可以發送以下命令來啟動一個僅處理該 Queue 的 Worker：

```shell
php artisan queue:work redis --queue=emails
```


<a name="processing-a-specified-number-of-jobs"></a>
#### 處理指定數量的 Jobs

`--once` 選項可用於指示 Worker 僅處理來自 Queue 的單個 Job：

```shell
php artisan queue:work --once
```

`--max-jobs` 選項可用於指示 Worker 處理給定數量的 Jobs 後結束。此選項在與 [Supervisor](#supervisor-configuration) 結合使用時非常有用，以便您的 Workers 在處理給定數量的 Jobs 後自動重新啟動，釋放它們可能累積的任何記憶體：

```shell
php artisan queue:work --max-jobs=1000
```


<a name="processing-all-queued-jobs-then-exiting"></a>
#### 處理所有佇列中的 Jobs 後結束

`--stop-when-empty` 選項可用於指示 Worker 處理所有 Jobs 然後優雅地結束。如果您在 Docker 容器中處理 Laravel Queues，且希望在 Queue 清空後關閉容器，則此選項非常有用：

```shell
php artisan queue:work --stop-when-empty
```


<a name="processing-jobs-for-a-given-number-of-seconds"></a>
#### 在指定秒數內處理 Jobs

`--max-time` 選項可用於指示 Worker 在處理 Jobs 指定秒數後結束。此選項在與 [Supervisor](#supervisor-configuration) 結合使用時非常有用，以便您的 Workers 在處理 Jobs 一段時間後自動重新啟動，釋放它們可能累積的任何記憶體：

```shell
# Process jobs for one hour and then exit...
php artisan queue:work --max-time=3600
```


<a name="worker-sleep-duration"></a>
#### Worker 睡眠時間

當 Queue 中有可用的 Jobs 時，Worker 將持續處理 Jobs，Job 之間沒有任何延遲。然而，`sleep` 選項決定了如果沒有可用的 Jobs，Worker 將「睡眠」多少秒。當然，在睡眠期間，Worker 不會處理任何新 Job：

```shell
php artisan queue:work --sleep=3
```


<a name="maintenance-mode-queues"></a>
#### 維護模式與 Queues

當您的應用程式處於 [維護模式](/docs/{{version}}/configuration#maintenance-mode) 時，將不會處理任何佇列中的 Jobs。一旦應用程式脫離維護模式，Jobs 將照常繼續處理。

要強制您的 Queue Workers 即使啟用了維護模式也處理 Jobs，您可以使用 `--force` 選項：

```shell
php artisan queue:work --force
```


<a name="resource-considerations"></a>
#### 資源考量

Daemon Queue Workers 在處理每個 Job 之前不會「重啟」框架。因此，您應該在每個 Job 完成後釋放任何耗費資源的項目。例如，如果您正在使用 [GD 函式庫](https://www.php.net/manual/en/book.image.php) 進行影像處理，則在影像處理完畢後，您應該使用 `imagedestroy` 釋放記憶體。


<a name="queue-priorities"></a>
### Queue 優先級

有時您可能希望優先處理您的 Queues。例如，在您的 `config/queue.php` 設定檔中，您可以將 `redis` 連接的預設 `queue` 設定為 `low`。然而，偶爾您可能希望將一個 Job 推送到 `high` 優先級的 Queue，如下所示：

```php
dispatch((new Job)->onQueue('high'));
```

要啟動一個 Worker，確保在繼續處理 `low` Queue 上的任何 Jobs 之前，先處理完所有 `high` Queue 的 Jobs，請向 `work` 命令傳遞一個以逗號分隔的 Queue 名稱清單：

```shell
php artisan queue:work --queue=high,low
```


<a name="queue-workers-and-deployment"></a>
### Queue Workers 與佈署

由於 Queue Workers 是長駐型進程，如果不重新啟動，它們將不會察覺到程式碼的變動。因此，部署使用 Queue Workers 應用程式的最簡單方法是在部署過程中重新啟動 Workers。您可以透過發送 `queue:restart` 命令來優雅地重新啟動所有 Workers：

```shell
php artisan queue:restart
```

此命令將指示所有 Queue Workers 在處理完目前 Job 後優雅地結束，以免遺失任何現有的 Jobs。由於 Queue Workers 將在執行 `queue:restart` 命令時結束，因此您應該執行如 [Supervisor](#supervisor-configuration) 之類的進程管理器來自動重新啟動 Queue Workers。

> [!NOTE]
> Queue 使用 [快取](/docs/{{version}}/cache) 來儲存重新啟動訊號，因此在使用此功能之前，您應該驗證是否為您的應用程式正確設定了快取驅動程式。

<a name="job-expirations-and-timeouts"></a>
### Job 過期與超時

<a name="job-expiration"></a>
#### Job 過期

在您的 `config/queue.php` 設定檔中，每個 Queue 連接都定義了一個 `retry_after` 選項。此選項指定了 Queue 連接在重試正在處理的 Job 之前應等待多少秒。例如，如果 `retry_after` 的值設為 `90`，那麼如果該 Job 已處理 90 秒而未被釋放或刪除，它將被放回 Queue 中。通常，您應該將 `retry_after` 的值設定為您的 Job 預計完成處理所需的最大秒數。

> [!WARNING]
> 唯一不包含 `retry_after` 值的 Queue 連接是 Amazon SQS。SQS 將根據在 AWS 控制台中管理的 [預設可見性超時 (Default Visibility Timeout)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html) 來重試 Job。

<a name="worker-timeouts"></a>
#### Worker 超時

`queue:work` Artisan 命令提供了一個 `--timeout` 選項。預設情況下，`--timeout` 的值為 60 秒。如果一個 Job 的處理時間超過了超時值指定的秒數，處理該 Job 的 Worker 將會出錯並退出。通常，Worker 會由[伺服器上設定的程序管理器](#supervisor-configuration)自動重新啟動：

```shell
php artisan queue:work --timeout=60
```

`retry_after` 設定選項和 `--timeout` CLI 選項不同，但它們會協同工作，以確保 Job 不會遺失，且 Job 只會成功處理一次。

> [!WARNING]
> `--timeout` 的值應該始終比您的 `retry_after` 設定值短至少幾秒鐘。這將確保處理凍結 Job 的 Worker 始終在 Job 被重試之前被終止。如果您的 `--timeout` 選項比您的 `retry_after` 設定值長，您的 Job 可能會被處理兩次。

<a name="pausing-and-resuming-queue-workers"></a>
### 暫停與恢復 Queue Workers

有時您可能需要暫時防止 Queue Worker 處理新 Job，而不完全停止 Worker。例如，您可能希望在系統維護期間暫停 Job 處理。Laravel 提供了 `queue:pause` 和 `queue:continue` Artisan 命令來暫停與恢復 Queue Workers。

若要暫停特定的 Queue，請提供 Queue 連接名稱與 Queue 名稱：

```shell
php artisan queue:pause database:default
```

在此範例中，`database` 是 Queue 連接名稱，而 `default` 是 Queue 名稱。一旦 Queue 被暫停，任何從該 Queue 處理 Job 的 Worker 將繼續完成其目前的 Job，但在該 Queue 恢復之前不會取得任何新 Job。

若要恢復處理已暫停 Queue 中的 Job，請使用 `queue:continue` 命令：

```shell
php artisan queue:continue database:default
```

恢復 Queue 後，Worker 將立即開始處理該 Queue 中的新 Job。請注意，暫停 Queue 不會停止 Worker 程序本身 —— 它僅防止 Worker 從指定的 Queue 處理新 Job。

<a name="worker-restart-and-pause-signals"></a>
#### Worker 重啟與暫停訊號

預設情況下，Queue Workers 會在每次 Job 迭代時輪詢快取驅動程式以獲取重啟與暫停訊號。雖然這種輪詢對於回應 `queue:restart` 和 `queue:pause` 命令至關重要，但它確實會引入一小部分效能開銷。

如果您需要優化效能且不需要這些中斷功能，您可以透過在 `Queue` Facade 上呼叫 `withoutInterruptionPolling` 方法來全域停用此輪詢。這通常應該在您的 `AppServiceProvider` 的 `boot` 方法中完成：

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

或者，您可以透過在 `Illuminate\Queue\Worker` 類別上設定靜態屬性 `$restartable` 或 `$pausable` 來分別停用重啟或暫停輪詢：

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
> 當中斷輪詢被停用時，Workers 將不會回應 `queue:restart` 或 `queue:pause` 命令（取決於停用了哪些功能）。

<a name="supervisor-configuration"></a>
## Supervisor 設定

在正式環境中，您需要一種方法來保持 `queue:work` 程序持續執行。`queue:work` 程序可能會因為各種原因停止運作，例如超過 worker 超時時間或執行了 `queue:restart` 命令。

出於這個原因，您需要設定一個程序監控器，它可以偵測您的 `queue:work` 程序何時結束並自動重新啟動它們。此外，程序監控器可以讓您指定想要同時執行多少個 `queue:work` 程序。Supervisor 是 Linux 環境中常用的程序監控器，我們將在接下來的文件中討論如何設定它。

<a name="installing-supervisor"></a>
#### 安裝 Supervisor

Supervisor 是 Linux 作業系統的程序監控器，如果您的 `queue:work` 程序失敗，它將會自動重新啟動它們。要在 Ubuntu 上安裝 Supervisor，您可以使用以下命令：

```shell
sudo apt-get install supervisor
```

> [!NOTE]
> 如果自行設定與管理 Supervisor 讓您感到不知所措，請考慮使用 [Laravel Cloud](https://cloud.laravel.com)，它提供了一個用於執行 Laravel queue workers 的全代管平台。

<a name="configuring-supervisor"></a>
#### 設定 Supervisor

Supervisor 設定檔通常儲存在 `/etc/supervisor/conf.d` 目錄中。在此目錄中，您可以建立任意數量的設定檔，指示 supervisor 應該如何監控您的程序。例如，讓我們建立一個 `laravel-worker.conf` 檔案來啟動並監控 `queue:work` 程序：

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

在此範例中，`numprocs` 指令將指示 Supervisor 執行八個 `queue:work` 程序並監控所有程序，如果它們失敗則自動重新啟動。您應該更改設定中的 `command` 指令，以反映您所需的 queue 連接和 worker 選項。

> [!WARNING]
> 您應該確保 `stopwaitsecs` 的值大於執行時間最長的 job 所消耗的秒數。否則，Supervisor 可能會在 job 處理完成之前將其強制終止。

<a name="starting-supervisor"></a>
#### 啟動 Supervisor

設定檔建立完成後，您可以使用以下命令更新 Supervisor 設定並啟動程序：

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start "laravel-worker:*"
```

有關 Supervisor 的更多資訊，請參閱 [Supervisor 說明文件](http://supervisord.org/index.html)。

<a name="dealing-with-failed-jobs"></a>
## 處理失敗的 Jobs

有時您的佇列 Job 會失敗。別擔心，事情並不總是按計劃進行！Laravel 提供了一種簡便的方法來[指定 Job 應嘗試的最大次數](#max-job-attempts-and-timeout)。在非同步 Job 超過此嘗試次數後，它將被插入 `failed_jobs` 資料庫資料表中。失敗的[同步派送的 Jobs](/docs/{{version}}/queues#synchronous-dispatching) 不會儲存在此資料表中，其異常會立即由應用程式處理。

在新的 Laravel 應用程式中，通常已經存在建立 `failed_jobs` 資料表的遷移。然而，如果您的應用程式不包含此資料表的遷移，您可以使用 `make:queue-failed-table` 命令來建立遷移：

```shell
php artisan make:queue-failed-table

php artisan migrate
```

執行 [Queue Worker](#running-the-queue-worker) 程序時，您可以使用 `queue:work` 命令上的 `--tries` 選項來指定 Job 應嘗試的最大次數。如果您沒有為 `--tries` 選項指定值，則 Job 僅會嘗試一次，或按 Job 類別的 `$tries` 屬性所指定的次數進行嘗試：

```shell
php artisan queue:work redis --tries=3
```

使用 `--backoff` 選項，您可以指定 Laravel 在重試遇到異常的 Job 之前應等待多少秒。預設情況下，Job 會立即釋放回佇列中，以便可以再次嘗試：

```shell
php artisan queue:work redis --tries=3 --backoff=3
```

如果您想針對每個 Job 設定 Laravel 在重試遇到異常的 Job 之前應等待多少秒，您可以透過在 Job 類別上定義 `backoff` 屬性來實現：

```php
/**
 * The number of seconds to wait before retrying the job.
 *
 * @var int
 */
public $backoff = 3;
```

如果您需要更複雜的邏輯來決定 Job 的重試延遲時間，您可以在 Job 類別上定義 `backoff` 方法：

```php
/**
 * Calculate the number of seconds to wait before retrying the job.
 */
public function backoff(): int
{
    return 3;
}
```

您可以透過從 `backoff` 方法回傳一個重試延遲值的陣列，來輕鬆設定「指數級」的延遲。在此範例中，第一次重試的延遲為 1 秒，第二次為 5 秒，第三次為 10 秒，如果還有剩餘的嘗試次數，則之後的每次重試均為 10 秒：

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
### 失敗 Jobs 後的清理

當特定的 Job 失敗時，您可能希望向使用者傳送警報，或還原由 Job 部分完成的任何操作。為此，您可以在 Job 類別上定義一個 `failed` 方法。導致 Job 失敗的 `Throwable` 執行個體將被傳遞給 `failed` 方法：

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
> 在調用 `failed` 方法之前，會先實例化 Job 的新執行個體；因此，在 `handle` 方法中可能發生的任何類別屬性修改都將遺失。

失敗的 Job 並不一定是指遇到未處理異常的 Job。當 Job 耗盡所有允許的嘗試次數時，也可以被視為失敗。這些嘗試次數可以透過以下幾種方式消耗：

<div class="content-list" markdown="1">

- Job 超時。
- Job 在執行過程中遇到未處理的異常。
- Job 被手動或由中介層釋放回佇列。

</div>

如果最後一次嘗試是因為 Job 執行期間拋出的異常而失敗，該異常將傳遞給 Job 的 `failed` 方法。然而，如果 Job 是因為達到最大允許嘗試次數而失敗，則 `$exception` 將會是 `Illuminate\Queue\MaxAttemptsExceededException` 的執行個體。同樣地，如果 Job 是因為超過設定的超時時間而失敗，則 `$exception` 將會是 `Illuminate\Queue\TimeoutExceededException` 的執行個體。


<a name="retrying-failed-jobs"></a>
### 重試失敗的 Jobs

要查看已插入 `failed_jobs` 資料庫資料表中的所有失敗 Job，您可以使用 `queue:failed` Artisan 命令：

```shell
php artisan queue:failed
```

`queue:failed` 命令將列出 Job ID、連接、佇列、失敗時間以及有關該 Job 的其他資訊。Job ID 可用於重試失敗的 Job。例如，要重試 ID 為 `ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece` 的失敗 Job，請執行以下命令：

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

要重試所有失敗的 Job，請執行 `queue:retry` 命令並將 `all` 作為 ID 傳入：

```shell
php artisan queue:retry all
```

如果您想刪除一個失敗的 Job，可以使用 `queue:forget` 命令：

```shell
php artisan queue:forget 91401d2c-0784-4f43-824c-34f94a33c24d
```

> [!NOTE]
> 使用 [Horizon](/docs/{{version}}/horizon) 時，應使用 `horizon:forget` 命令來刪除失敗的 Job，而不是 `queue:forget` 命令。

要從 `failed_jobs` 資料表中刪除所有失敗的 Job，可以使用 `queue:flush` 命令：

```shell
php artisan queue:flush
```

`queue:flush` 命令會從您的佇列中移除所有失敗的 Job 紀錄，無論失敗的 Job 多久。您可以使用 `--hours` 選項僅刪除特定小時數之前或更早失敗的 Job：

```shell
php artisan queue:flush --hours=48
```


<a name="ignoring-missing-models"></a>
### 忽略遺失的 Models

將 Eloquent 模型注入 Job 時，該模型在放入佇列之前會自動被序列化，並在 Job 處理時從資料庫中重新檢索。然而，如果模型在 Job 等待 Worker 處理期間被刪除，您的 Job 可能會失敗並拋出 `ModelNotFoundException`。

為方便起見，您可以透過將 Job 的 `deleteWhenMissingModels` 屬性設置為 `true`，來選擇自動刪除遺失模型的 Job。當此屬性設置為 `true` 時，Laravel 將安靜地丟棄該 Job 而不引發異常：

```php
/**
 * Delete the job if its models no longer exist.
 *
 * @var bool
 */
public $deleteWhenMissingModels = true;
```


<a name="pruning-failed-jobs"></a>
### 修剪失敗的 Jobs

您可以透過調用 `queue:prune-failed` Artisan 命令來修剪應用程式 `failed_jobs` 資料表中的紀錄：

```shell
php artisan queue:prune-failed
```

預設情況下，所有超過 24 小時的失敗 Job 紀錄都將被修剪。如果您為命令提供 `--hours` 選項，則僅保留過去 N 小時內插入的失敗 Job 紀錄。例如，以下命令將刪除所有在 48 小時以前插入的失敗 Job 紀錄：

```shell
php artisan queue:prune-failed --hours=48
```

<a name="storing-failed-jobs-in-dynamodb"></a>
### 將失敗的 Jobs 儲存在 DynamoDB

Laravel 也提供支援將失敗的 Job 紀錄儲存在 [DynamoDB](https://aws.amazon.com/dynamodb) 而非關聯式資料庫資料表中。但是，您必須手動建立一個 DynamoDB 資料表來儲存所有失敗的 Job 紀錄。通常，此資料表應命名為 `failed_jobs`，但您應該根據應用程式 `queue` 設定檔中 `queue.failed.table` 設定值來命名資料表。

`failed_jobs` 資料表應具有一個名為 `application` 的字串主要分割鍵 (Partition Key) 以及一個名為 `uuid` 的字串主要排序鍵 (Sort Key)。鍵的 `application` 部分將包含您的應用程式名稱，如應用程式 `app` 設定檔中 `name` 設定值所定義。由於應用程式名稱是 DynamoDB 資料表鍵的一部分，因此您可以使用同一個資料表來儲存多個 Laravel 應用程式的失敗 Jobs。

此外，請確保您安裝了 AWS SDK，以便您的 Laravel 應用程式可以與 Amazon DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

接著，將 `queue.failed.driver` 設定選項的值設為 `dynamodb`。此外，您應該在失敗 Job 設定陣列中定義 `key`、`secret` 與 `region` 設定選項。這些選項將用於 AWS 認證。使用 `dynamodb` 驅動程式時，不需要 `queue.failed.database` 設定選項：

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

您可以透過將 `queue.failed.driver` 設定選項的值設為 `null`，來指示 Laravel 捨棄失敗的 Jobs 而不進行儲存。通常，這可以透過 `QUEUE_FAILED_DRIVER` 環境變數來完成：

```ini
QUEUE_FAILED_DRIVER=null
```

<a name="failed-job-events"></a>
### 失敗 Job 事件

如果您想註冊一個在 Job 失敗時叫用的事件監聽器，可以使用 `Queue` Facade 的 `failing` 方法。例如，我們可以從 Laravel 內建的 `AppServiceProvider` 的 `boot` 方法中將一個閉包附加到此事件：

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
## 清除 Queue 中的 Jobs

> [!NOTE]
> 當使用 [Horizon](/docs/{{version}}/horizon) 時，您應該使用 `horizon:clear` 命令來清除 Queue 中的 Jobs，而不是使用 `queue:clear` 命令。

如果您想從預設連接的預設 Queue 中刪除所有 Jobs，可以使用 `queue:clear` Artisan 命令：

```shell
php artisan queue:clear
```

您也可以提供 `connection` 參數與 `queue` 選項，以從特定的連接和 Queue 中刪除 Jobs：

```shell
php artisan queue:clear redis --queue=emails
```

> [!WARNING]
> 清除 Queue 中的 Jobs 僅適用於 SQS、Redis 和資料庫 Queue 驅動程式。此外，SQS 訊息刪除程序最多需要 60 秒，因此在您清除 Queue 後 60 秒內發送到 SQS Queue 的 Jobs 也可能會被刪除。

<a name="monitoring-your-queues"></a>
## 監控您的 Queues

如果您的 Queue 突然湧入大量 Jobs，它可能會變得應接不暇，導致 Jobs 完成的等待時間過長。如果您願意，Laravel 可以在您的 Queue Job 數量超過指定閾值時提醒您。

首先，您應該將 `queue:monitor` 命令排定為[每分鐘執行一次](/docs/{{version}}/scheduling)。此命令接受您想要監控的 Queue 名稱以及您期望的 Job 數量閾值：

```shell
php artisan queue:monitor redis:default,redis:deployments --max=100
```

僅排定此命令不足以觸發通知來提醒您 Queue 的繁忙狀態。當命令遇到 Job 數量超過閾值的 Queue 時，將會派送一個 `Illuminate\Queue\Events\QueueBusy` 事件。您可以在應用程式的 `AppServiceProvider` 中監聽此事件，以便向您或您的開發團隊發送通知：

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

當測試會派送 jobs 的程式碼時，您可能希望指示 Laravel 不要實際執行 job 本身，因為 job 的程式碼可以與派送它的程式碼分開來直接進行測試。當然，若要測試 job 本身，您可以實例化 job 並在測試中直接調用 `handle` 方法。

您可以使用 `Queue` facade 的 `fake` 方法來防止已排入隊列的 jobs 實際被推送到 queue。在調用 `Queue` facade 的 `fake` 方法後，您就可以斷言應用程式曾嘗試將 jobs 推送到 queue：

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

您可以將閉包傳遞給 `assertPushed`、`assertNotPushed`、`assertClosurePushed` 或 `assertClosureNotPushed` 方法，以便斷言已推送的 job 是否通過給定的「真值測試 (Truth Test)」。如果至少有一個推送的 job 通過給定的真值測試，則斷言將成功：

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
### 模擬部分 Jobs

如果您只需要模擬特定的 jobs，同時允許其他 jobs 正常執行，您可以將應被模擬的 job 類別名稱傳遞給 `fake` 方法：

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

您可以使用 `except` 方法來模擬除了指定的一組 jobs 之外的所有 jobs：

```php
Queue::fake()->except([
    ShipOrder::class,
]);
```


<a name="testing-job-chains"></a>
### 測試 Job 鏈接

要測試 Job 鏈接，您需要利用 `Bus` facade 的模擬功能。`Bus` facade 的 `assertChained` 方法可用於斷言已派送了一個 [Job 鏈接](/docs/{{version}}/queues#job-chaining)。`assertChained` 方法的第一個參數接受一個鏈接的 jobs 陣列：

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

如上面的範例所示，鏈接的 jobs 陣列可以是 job 的類別名稱陣列。然而，您也可以提供實際的 job 實例陣列。這樣做時，Laravel 將確保 job 實例與應用程式派送的鏈接 jobs 屬於相同的類別且具有相同的屬性值：

```php
Bus::assertChained([
    new ShipOrder,
    new RecordShipment,
    new UpdateInventory,
]);
```

您可以使用 `assertDispatchedWithoutChain` 方法來斷言一個 job 被推送時不帶有 Job 鏈接：

```php
Bus::assertDispatchedWithoutChain(ShipOrder::class);
```


<a name="testing-chain-modifications"></a>
#### 測試鏈接修改

如果鏈接的 job [將 jobs 前置或後置到現有鏈接](#adding-jobs-to-the-chain)，您可以使用該 job 的 `assertHasChain` 方法來斷言該 job 具有預期的剩餘鏈接：

```php
$job = new ProcessPodcast;

$job->handle();

$job->assertHasChain([
    new TranscribePodcast,
    new OptimizePodcast,
    new ReleasePodcast,
]);
```

`assertDoesntHaveChain` 方法可用於斷言 job 的剩餘鏈接為空：

```php
$job->assertDoesntHaveChain();
```


<a name="testing-chained-batches"></a>
#### 測試鏈接批次

如果您的 Job 鏈接[包含一個 Job 批次](#chains-and-batches)，您可以透過在鏈接斷言中插入 `Bus::chainedBatch` 定義來斷言鏈接中的批次是否符合您的預期：

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

`Bus` Facade 的 `assertBatched` 方法可用於斷言 [Job 批次](/docs/{{version}}/queues#job-batching) 已被派送。提供給 `assertBatched` 方法的閉包會接收一個 `Illuminate\Bus\PendingBatch` 實例，可用於檢查批次中的 Jobs：

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

`hasJobs` 方法可用於待處理批次上，以驗證批次是否包含預期的 Jobs。此方法接受 Job 實例、類別名稱或閉包的陣列：

```php
Bus::assertBatched(function (PendingBatch $batch) {
    return $batch->hasJobs([
        new ProcessCsvRow(row: 1),
        new ProcessCsvRow(row: 2),
        new ProcessCsvRow(row: 3),
    ]);
});
```

使用閉包時，閉包將接收 Job 實例。預期的 Job 型別將從閉包的型別提示 (Type Hint) 中推斷出來：

```php
Bus::assertBatched(function (PendingBatch $batch) {
    return $batch->hasJobs([
        fn (ProcessCsvRow $job) => $job->row === 1,
        fn (ProcessCsvRow $job) => $job->row === 2,
        fn (ProcessCsvRow $job) => $job->row === 3,
    ]);
});
```

您可以使用 `assertBatchCount` 方法來斷言已派送了指定數量的批次：

```php
Bus::assertBatchCount(3);
```

您可以使用 `assertNothingBatched` 來斷言沒有派送任何批次：

```php
Bus::assertNothingBatched();
```


<a name="testing-job-batch-interaction"></a>
#### 測試 Job / 批次互動

此外，您偶爾可能需要測試單個 Job 與其底層批次的互動。例如，您可能需要測試一個 Job 是否取消了其批次的後續處理。要達成此目的，您需要透過 `withFakeBatch` 方法為 Job 分配一個模擬批次。`withFakeBatch` 方法會傳回一個包含 Job 實例與模擬批次的元組 (Tuple)：

```php
[$job, $batch] = (new ShipOrder)->withFakeBatch();

$job->handle();

$this->assertTrue($batch->cancelled());
$this->assertEmpty($batch->added);
```


<a name="testing-job-queue-interactions"></a>
### 測試 Job / Queue 互動

有時，您可能需要測試排隊的 Job 是否將 [自身釋放回 Queue](#manually-releasing-a-job)。或者，您可能需要測試 Job 是否刪除了自身。您可以透過實例化 Job 並呼叫 `withFakeQueueInteractions` 方法來測試這些 Queue 互動。

一旦模擬了 Job 的 Queue 互動，您就可以在 Job 上呼叫 `handle` 方法。在執行 Job 之後，有多種斷言方法可用於驗證 Job 的 Queue 互動：

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

透過 `Queue` [Facade](/docs/{{version}}/facades) 的 `before` 與 `after` 方法，您可以指定在處理佇列 Job 之前或之後執行的回呼。這些回呼是進行額外日誌記錄或為儀表板累加統計數據的好機會。通常，您應該在 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫這些方法。例如，我們可以使用 Laravel 內建的 `AppServiceProvider`：

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

透過 `Queue` [Facade](/docs/{{version}}/facades) 的 `looping` 方法，您可以指定在 Worker 嘗試從 Queue 抓取 Job 之前執行的回呼。例如，您可能會註冊一個閉包來還原任何由先前失敗 Job 所留下的開啟交易：

```php
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Queue;

Queue::looping(function () {
    while (DB::transactionLevel() > 0) {
        DB::rollBack();
    }
});
```