# 隊列 (Queues)

- [簡介](#introduction)
    - [連線 vs. 隊列](#connections-vs-queues)
    - [驅動程式備註與先決條件](#driver-prerequisites)
- [建立 Job](#creating-jobs)
    - [生成 Job 類別](#generating-job-classes)
    - [類別結構](#class-structure)
    - [唯一 Job](#unique-jobs)
    - [加密 Job](#encrypted-jobs)
- [Job 中介層](#job-middleware)
    - [速率限制](#rate-limiting)
    - [防止 Job 重疊](#preventing-job-overlaps)
    - [節流例外狀況](#throttling-exceptions)
    - [跳過 Job](#skipping-jobs)
- [派遣 Job](#dispatching-jobs)
    - [延遲派遣](#delayed-dispatching)
    - [同步派遣](#synchronous-dispatching)
    - [Job 與資料庫交易](#jobs-and-database-transactions)
    - [Job 鏈結](#job-chaining)
    - [自定義隊列與連線](#customizing-the-queue-and-connection)
    - [指定 Job 最大重試次數 / 超時數值](#max-job-attempts-and-timeout)
    - [SQS FIFO 與公平隊列](#sqs-fifo-and-fair-queues)
    - [隊列容錯移轉 (Failover)](#queue-failover)
    - [錯誤處理](#error-handling)
- [Job 批次處理](#job-batching)
    - [定義可批次處理的 Job](#defining-batchable-jobs)
    - [派遣批次](#dispatching-batches)
    - [鏈結與批次](#chains-and-batches)
    - [向批次添加 Job](#adding-jobs-to-batches)
    - [檢查批次](#inspecting-batches)
    - [取消批次](#cancelling-batches)
    - [批次失敗處理](#batch-failures)
    - [修剪批次](#pruning-batches)
    - [在 DynamoDB 中儲存批次](#storing-batches-in-dynamodb)
- [隊列化閉包](#queueing-closures)
- [執行隊列 Worker](#running-the-queue-worker)
    - [`queue:work` 指令](#the-queue-work-command)
    - [隊列優先權](#queue-priorities)
    - [隊列 Worker 與部署](#queue-workers-and-deployment)
    - [Job 過期與超時](#job-expirations-and-timeouts)
    - [暫停與恢復隊列 Worker](#pausing-and-resuming-queue-workers)
- [Supervisor 設定](#supervisor-configuration)
- [處理失敗的 Job](#dealing-with-failed-jobs)
    - [失敗 Job 的後續清理](#cleaning-up-after-failed-jobs)
    - [重試失敗的 Job](#retrying-failed-jobs)
    - [忽略缺失的模型](#ignoring-missing-models)
    - [修剪失敗的 Job](#pruning-failed-jobs)
    - [在 DynamoDB 中儲存失敗的 Job](#storing-failed-jobs-in-dynamodb)
    - [停用失敗 Job 儲存](#disabling-failed-job-storage)
    - [失敗 Job 事件](#failed-job-events)
- [清除隊列中的 Job](#clearing-jobs-from-queues)
- [監控您的隊列](#monitoring-your-queues)
- [測試](#testing)
    - [模擬 Job 子集](#faking-a-subset-of-jobs)
    - [測試 Job 鏈結](#testing-job-chains)
    - [測試 Job 批次](#testing-job-batches)
    - [測試 Job / 隊列互動](#testing-job-queue-interactions)
- [Job 事件](#job-events)

<a name="introduction"></a>
## 簡介

在建構 Web 應用程式時，您可能會遇到一些任務（例如解析並儲存上傳的 CSV 檔案），這些任務在典型的 Web 請求期間執行會耗費太長時間。幸運的是，Laravel 允許您輕鬆建立可在背景處理的隊列 Job。透過將耗時的任務移至隊列，您的應用程式可以極快的速度回應 Web 請求，並為您的客戶提供更好的使用者體驗。

Laravel 隊列為各種不同的隊列後端（例如 [Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 甚至關聯式資料庫）提供了統一的隊列 API。

Laravel 的隊列設定選項儲存在應用程式的 `config/queue.php` 設定檔中。在此檔案中，您將找到框架內含的每個隊列驅動程式的連線設定，包括資料庫、[Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 和 [Beanstalkd](https://beanstalkd.github.io/) 驅動程式，以及一個會立即執行 Job 的同步 (synchronous) 驅動程式（用於開發或測試期間）。同時也包含了一個 `null` 隊列驅動程式，用於捨棄隊列中的 Job。

> [!NOTE]
> Laravel Horizon 是一個為 Redis 驅動的隊列提供美觀儀表板與設定系統的工具。請查看完整的 [Horizon 文件](/docs/{{version}}/horizon) 以取得更多資訊。


<a name="connections-vs-queues"></a>
### 連線 vs. 隊列

在開始使用 Laravel 隊列之前，了解「連線 (connections)」與「隊列 (queues)」之間的區別非常重要。在您的 `config/queue.php` 設定檔中，有一個 `connections` 設定陣列。此選項定義了與後端隊列服務（如 Amazon SQS、Beanstalk 或 Redis）的連線。然而，任何給定的隊列連線都可能擁有多個「隊列」，可以將其視為隊列 Job 的不同堆疊或堆集。

請注意，`queue` 設定檔中的每個連線設定範例都包含一個 `queue` 屬性。這是 Job 在傳送到給定連線時，若未明確定義要傳送到哪個隊列，則會被派遣到的預設隊列。換句話說，如果您在派遣 Job 時沒有明確定義應將其派遣到哪個隊列，該 Job 將被放置在連線設定中 `queue` 屬性所定義的隊列上：

```php
use App\Jobs\ProcessPodcast;

// This job is sent to the default connection's default queue...
ProcessPodcast::dispatch();

// This job is sent to the default connection's "emails" queue...
ProcessPodcast::dispatch()->onQueue('emails');
```

某些應用程式可能永遠不需要將 Job 推送到多個隊列，而是傾向於使用一個簡單的隊列。然而，對於希望對 Job 處理方式進行優先排序或分割的應用程式來說，將 Job 推送到多個隊列特別有用，因為 Laravel 隊列 Worker 允許您指定應按優先順序處理哪些隊列。例如，如果您將 Job 推送到 `high` 隊列，則可以執行一個賦予它們較高處理優先權的 Worker：

```shell
php artisan queue:work --queue=high,default
```


<a name="driver-prerequisites"></a>
### 驅動程式備註與先決條件


<a name="database"></a>
#### 資料庫

為了使用 `database` 隊列驅動程式，您需要一個資料庫資料表來存放 Job。通常，這已包含在 Laravel 預設的 `0001_01_01_000002_create_jobs_table.php` [資料庫遷移 (Database Migration)](/docs/{{version}}/migrations) 中；然而，如果您的應用程式不包含此遷移，您可以使用 `make:queue-table` Artisan 指令來建立它：

```shell
php artisan make:queue-table

php artisan migrate
```


<a name="redis"></a>
#### Redis

為了使用 `redis` 隊列驅動程式，您應該在 `config/database.php` 設定檔中設定一個 Redis 資料庫連線。

> [!WARNING]
> `redis` 隊列驅動程式不支援 Redis 的 `serializer` 和 `compression` 選項。


<a name="redis-cluster"></a>
##### Redis Cluster

如果您的 Redis 隊列連線使用 [Redis Cluster](https://redis.io/docs/latest/operate/rs/databases/durability-ha/clustering)，您的隊列名稱必須包含 [金鑰雜湊標記 (key hash tag)](https://redis.io/docs/latest/develop/using-commands/keyspace/#hashtags)。這是必要的，以確保給定隊列的所有 Redis 金鑰都放置在同一個雜湊槽 (hash slot) 中：

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

使用 Redis 隊列時，您可以使用 `block_for` 設定選項來指定驅動程式在迭代 Worker 迴圈並重新輪詢 Redis 資料庫之前，應等待 Job 變為可用狀態的時間。

根據您的隊列負載調整此值，會比持續輪詢 Redis 資料庫以獲取新 Job 更有效率。例如，您可以將該值設定為 `5`，表示驅動程式在等待 Job 變為可用時應阻塞五秒鐘：

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
> 將 `block_for` 設定為 `0` 將導致隊列 Worker 無限期地阻塞，直到 Job 可用為止。這也會防止 `SIGTERM` 等訊號被處理，直到下一個 Job 處理完畢。


<a name="other-driver-prerequisites"></a>
#### 其他驅動程式先決條件

列出的隊列驅動程式需要以下依賴項目。這些依賴項目可以透過 Composer 套件管理員安裝：

<div class="content-list" markdown="1">

- Amazon SQS: `aws/aws-sdk-php ~3.0`
- Beanstalkd: `pda/pheanstalk ~5.0`
- Redis: `predis/predis ~2.0` 或 phpredis PHP 擴充功能
- [MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/queues/): `mongodb/laravel-mongodb`

</div>

<a name="creating-jobs"></a>
## 建立 Job

<a name="generating-job-classes"></a>
### 生成 Job 類別

預設情況下，應用程式中所有可隊列化的 Job 都儲存在 `app/Jobs` 目錄中。如果 `app/Jobs` 目錄不存在，當您執行 `make:job` Artisan 指令時，該目錄將會被建立：

```shell
php artisan make:job ProcessPodcast
```

生成的類別將實作 `Illuminate\Contracts\Queue\ShouldQueue` 介面，這向 Laravel 表明該 Job 應該被推送到隊列中非同步執行。

> [!NOTE]
> Job Stub 可以使用 [Stub 發布](/docs/{{version}}/artisan#stub-customization) 進行自定義。

<a name="class-structure"></a>
### 類別結構

Job 類別非常簡單，通常只包含一個 `handle` 方法，該方法在 Job 被隊列處理時調用。首先，讓我們看一個 Job 類別範例。在這個範例中，我們假設管理一個 Podcast 發布服務，並需要在上傳的 Podcast 檔案發布前進行處理：

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

在這個範例中，請注意我們可以直接將 [Eloquent 模型](/docs/{{version}}/eloquent) 傳遞給隊列 Job 的建構函式。由於 Job 使用了 `Queueable` trait，Eloquent 模型及其載入的關聯在處理 Job 時將會被優雅地序列化與反序列化。

如果您的隊列 Job 在建構函式中接受一個 Eloquent 模型，則只有模型的識別碼會被序列化到隊列中。當 Job 實際被處理時，隊列系統會自動從資料庫中重新取得完整的模型實例及其載入的關聯。這種模型序列化的方法可以讓發送到隊列驅動程式的 Job 負載 (Payload) 小得多。

<a name="handle-method-dependency-injection"></a>
#### `handle` 方法相依性注入

`handle` 方法在 Job 被隊列處理時調用。請注意，我們可以在 Job 的 `handle` 方法中對相依性進行型別提示。Laravel [服務容器](/docs/{{version}}/container) 會自動注入這些相依性。

如果您想完全控制容器如何將相依性注入到 `handle` 方法中，可以使用容器的 `bindMethod` 方法。`bindMethod` 方法接受一個回呼，該回呼接收 Job 與容器。在回呼中，您可以隨心所欲地調用 `handle` 方法。通常，您應該在 `App\Providers\AppServiceProvider` [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中調用此方法：

```php
use App\Jobs\ProcessPodcast;
use App\Services\AudioProcessor;
use Illuminate\Contracts\Foundation\Application;

$this->app->bindMethod([ProcessPodcast::class, 'handle'], function (ProcessPodcast $job, Application $app) {
    return $job->handle($app->make(AudioProcessor::class));
});
```

> [!WARNING]
> 二進位資料（例如原始圖片內容）在傳遞給隊列 Job 之前，應先通過 `base64_encode` 函式處理。否則，Job 在被放入隊列時可能無法正確序列化為 JSON。

<a name="handling-relationships"></a>
#### 隊列化關聯

由於所有已載入的 Eloquent 模型關聯在 Job 進入隊列時也會被序列化，序列化後的 Job 字串有時會變得非常大。此外，當 Job 被反序列化並從資料庫重新取得模型關聯時，它們將會被完整地取得。在 Job 隊列化過程中，模型被序列化前套用的任何先前關聯約束，在 Job 反序列化時都不會被套用。因此，如果您希望處理特定關聯的子集，則應在隊列 Job 中重新約束該關聯。

或者，為了防止關聯被序列化，您可以在設定屬性值時對模型調用 `withoutRelations` 方法。此方法將返回不包含已載入關聯的模型實例：

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

如果您使用的是 [PHP 建構函式屬性提升 (Constructor Property Promotion)](https://www.php.net/manual/en/language.oop5.decon.php#language.oop5.decon.constructor.promotion)，並且希望指明一個 Eloquent 模型不應序列化其關聯，您可以使用 `WithoutRelations` 屬性：

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

為了方便起見，如果您希望序列化所有模型且不包含關聯，可以將 `WithoutRelations` 屬性套用於整個類別，而不是套用於每個模型：

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

如果 Job 接收的是 Eloquent 模型的集合或陣列而非單個模型，則在 Job 被反序列化並執行時，該集合中的模型將不會恢復其關聯。這是為了防止在處理大量模型的 Job 上消耗過多資源。

<a name="unique-jobs"></a>
### 唯一 Job

> [!WARNING]
> 唯一 Job 需要支援 [鎖 (Locks)](/docs/{{version}}/cache#atomic-locks) 的快取驅動程式。目前 `memcached`、`redis`、`dynamodb`、`database`、`file` 及 `array` 快取驅動程式皆支援原子鎖。

> [!WARNING]
> 唯一 Job 的限制不適用於批次 (Batches) 內的 Job。

有時候，您可能希望確保在任何時間點，隊列中只有一個特定 Job 的實例。您可以透過在 Job 類別實作 `ShouldBeUnique` 介面來達成此目的。此介面不需要您在類別中定義任何額外的方法：

```php
<?php

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Queue\ShouldBeUnique;

class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    // ...
}
```

在上述範例中，`UpdateSearchIndex` 這個 Job 是唯一的。因此，如果隊列中已經存在另一個相同的 Job 實例且尚未處理完成，則該 Job 將不會被派遣。

在某些情況下，您可能想要定義一個特定的「鍵 (Key)」來讓 Job 變得唯一，或者您可能想要指定一個超時時間，超過該時間後 Job 將不再保持唯一。為了達成此目的，您可以在 Job 類別中定義 `uniqueId` 與 `uniqueFor` 屬性或方法：

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

在上述範例中，`UpdateSearchIndex` Job 根據產品 ID 保持唯一。因此，在現有 Job 完成處理之前，任何具有相同產品 ID 的新 Job 派遣請求都將被忽略。此外，如果現有的 Job 在一小時內未被處理，唯一鎖將會被釋放，屆時另一個具有相同唯一鍵的 Job 即可被派遣到隊列中。

> [!WARNING]
> 如果您的應用程式從多個網頁伺服器或容器派遣 Job，您應確保所有伺服器都與同一個中央快取伺服器通訊，以便 Laravel 能夠準確判斷 Job 是否為唯一。

<a name="keeping-jobs-unique-until-processing-begins"></a>
#### 保持 Job 唯一直到處理開始

預設情況下，唯一 Job 會在處理完成或所有重試嘗試皆失敗後才「解鎖」。然而，在某些情況下，您可能希望 Job 在開始處理之前就立即解鎖。為了達成此目的，您的 Job 應該實作 `ShouldBeUniqueUntilProcessing` 契約 (Contract)，而非 `ShouldBeUnique` 契約：

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
#### 唯一 Job 鎖

在幕後，當派遣一個 `ShouldBeUnique` Job 時，Laravel 會嘗試以 `uniqueId` 作為鍵名來獲取一個 [鎖 (Lock)](/docs/{{version}}/cache#atomic-locks)。如果該鎖已被佔用，則不會派遣該 Job。此鎖會在 Job 處理完成或所有重試嘗試皆失敗後釋放。預設情況下，Laravel 會使用預設的快取驅動程式來獲取此鎖。然而，如果您希望使用另一個驅動程式來獲取鎖，您可以定義一個 `uniqueVia` 方法，並回傳應使用的快取驅動程式：

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

Laravel 允許您透過 [加密 (Encryption)](/docs/{{version}}/encryption) 來確保 Job 資料的隱私與完整性。要開始使用，只需將 `ShouldBeEncrypted` 介面新增到 Job 類別即可。一旦將此介面新增到類別，Laravel 會在將 Job 推送到隊列之前自動對其進行加密：

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

Job 中介層允許您在執行隊列 Job 的邏輯周圍封裝自定義邏輯，從而減少 Job 本身的樣板程式碼。例如，考慮以下 `handle` 方法，它利用 Laravel 的 Redis 速率限制功能，每五秒僅允許處理一個 Job：

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

雖然這段程式碼是有效的，但 `handle` 方法的實作會變得臃腫，因為它充斥著 Redis 速率限制邏輯。此外，對於我們想要限制速率的任何其他 Job，都必須重複這段速率限制邏輯。與其在 handle 方法中進行速率限制，我們可以定義一個處理速率限制的 Job 中介層：

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

如您所見，與[路由中介層](/docs/{{version}}/middleware)一樣，Job 中介層接收正在處理的 Job 以及一個應被呼叫以繼續處理 Job 的回呼。

您可以使用 `make:job-middleware` Artisan 指令生成新的 Job 中介層類別。建立 Job 中介層後，可以透過從 Job 的 `middleware` 方法回傳它們來將其附加到 Job。在使用 `make:job` Artisan 指令生成的 Job 中並不存在此方法，因此您需要手動將其加入到 Job 類別中：

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
> Job 中介層也可以分配給[隊列化事件監聽器](/docs/{{version}}/events#queued-event-listeners)、mailables 與 notifications。


<a name="rate-limiting"></a>
### 速率限制

雖然我們剛才示範了如何編寫自己的速率限制 Job 中介層，但 Laravel 實際上包含了一個您可以利用來限制 Job 速率的中介層。與[路由速率限制器](/docs/{{version}}/routing#defining-rate-limiters)一樣，Job 速率限制器是使用 `RateLimiter` Facade 的 `for` 方法定義的。

例如，您可能希望允許使用者每小時備份一次數據，而對頂級客戶則不設此限制。為了達成這個目標，您可以在 `AppServiceProvider` 的 `boot` 方法中定義一個 `RateLimiter`：

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

在上面的範例中，我們定義了每小時的速率限制；然而，您可以使用 `perMinute` 方法輕鬆定義基於分鐘的速率限制。此外，您可以將任何您想要的值傳遞給速率限制的 `by` 方法；不過，這個值最常用於根據客戶劃分速率限制：

```php
return Limit::perMinute(50)->by($job->user->id);
```

一旦定義了速率限制，就可以使用 `Illuminate\Queue\Middleware\RateLimited` 中介層將速率限制器附加到 Job 上。每當 Job 超過速率限制時，此中介層會根據速率限制持續時間，以適當的延遲將 Job 釋放回隊列中：

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

將受速率限制的 Job 釋放回隊列仍會增加 Job 的總 `attempts` 次數。您可能需要相應地調整 Job 類別上的 `tries` 和 `maxExceptions` 屬性。或者，您可能希望使用 [retryUntil 方法](#time-based-attempts)來定義在不再嘗試該 Job 之前的時間量。

使用 `releaseAfter` 方法，您還可以指定在再次嘗試釋放的 Job 之前必須經過的秒數：

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
#### Rate Limiting With Redis

如果您正在使用 Redis，可以使用 `Illuminate\Queue\Middleware\RateLimitedWithRedis` 中介層，該中介層專為 Redis 進行了優化，比基礎速率限制中介層效率更高：

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

Laravel 包含了一個 `Illuminate\Queue\Middleware\WithoutOverlapping` 中介層，讓您可以根據任意鍵值來防止 Job 重疊。當一個隊列化 Job 正在修改某個一次只能由一個 Job 修改的資源時，這非常有用。

例如，假設您有一個更新使用者信用評分的隊列化 Job，且您希望防止同一個使用者 ID 的信用評分更新 Job 發生重疊。要達成此目的，您可以從 Job 的 `middleware` 方法中回傳 `WithoutOverlapping` 中介層：

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

將重疊的 Job 釋放回隊列仍會增加該 Job 的總嘗試次數。您可能需要相應地調整 Job 類別中的 `tries` 與 `maxExceptions` 屬性。例如，若保持預設將 `tries` 屬性設為 1，則會防止任何重疊的 Job 在稍後重試。

任何相同類型的重疊 Job 都將會被釋放回隊列。您也可以指定被釋放的 Job 在再次嘗試前必須經過的秒數：

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

如果您希望立即刪除任何重疊的 Job 以便不再重試，您可以使用 `dontRelease` 方法：

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

`WithoutOverlapping` 中介層是由 Laravel 的原子鎖 (Atomic Lock) 功能所驅動的。有時候，您的 Job 可能會意外失敗或超時，導致鎖沒有被釋放。因此，您可以使用 `expireAfter` 方法明確定義鎖的過期時間。例如，下方的範例將指示 Laravel 在 Job 開始處理三分鐘後釋放 `WithoutOverlapping` 鎖：

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
> `WithoutOverlapping` 中介層需要支援 [鎖 (Locks)](/docs/{{version}}/cache#atomic-locks) 的快取驅動程式。目前 `memcached`、`redis`、`dynamodb`、`database`、`file` 與 `array` 快取驅動程式皆支援原子鎖。


<a name="sharing-lock-keys"></a>
#### 跨 Job 類別共享鎖定鍵 (Lock Keys)

根據預設，`WithoutOverlapping` 中介層只會防止相同類別的 Job 重疊。因此，即使兩個不同的 Job 類別使用相同的鎖定鍵，它們也不會被阻止重疊。然而，您可以使用 `shared` 方法指示 Laravel 跨 Job 類別套用該鍵值：

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
### 節流例外狀況

Laravel 包含一個 `Illuminate\Queue\Middleware\ThrottlesExceptions` 中介層，允許您節流例外狀況。一旦 Job 拋出指定數量的例外狀況，所有進一步執行該 Job 的嘗試都將延遲，直到指定的時間間隔過後為止。此中介層對於與不穩定的第三方服務互動的 Job 特別有用。

例如，讓我們想像一個與第三方 API 互動的隊列化 Job，而該 API 開始拋出例外狀況。若要節流例外狀況，您可以從 Job 的 `middleware` 方法回傳 `ThrottlesExceptions` 中介層。通常，此中介層應與實作 [基於時間的嘗試](#time-based-attempts) 的 Job 搭配使用：

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

中介層接受的第一個建構構參數是 Job 在被節流前可以拋出的例外狀況數量，而第二個建構參數是 Job 被節流後，再次嘗試執行前應經過的秒數。在上述程式碼範例中，如果 Job 連續拋出 10 次例外狀況，我們將等待 5 分鐘後再嘗試執行 Job，並受 30 分鐘的時間限制。

當 Job 拋出例外狀況但尚未達到例外狀況閾值時，Job 通常會立即重試。但是，您可以在將中介層附加到 Job 時，透過呼叫 `backoff` 方法來指定此類 Job 應延遲的分鐘數：

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

在內部，此中介層使用 Laravel 的快取系統來實作速率限制，並將 Job 的類別名稱用作快取的「鍵 (Key)」。您可以在將中介層附加到 Job 時透過呼叫 `by` 方法來覆蓋此鍵。如果您有多個 Job 與同一個第三方服務互動，並且希望它們共享一個共同的節流「貯體 (Bucket)」以確保它們遵循單一共享限制時，這會非常有用：

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

預設情況下，此中介層會節流每一個例外狀況。您可以透過在將中介層附加到 Job 時呼叫 `when` 方法來修改此行為。僅當提供給 `when` 方法的閉包回傳 `true` 時，該例外狀況才會被節流：

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

與 `when` 方法（會將 Job 釋放回隊列或拋出例外狀況）不同，`deleteWhen` 方法允許您在發生特定例外狀況時完全刪除 Job：

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

如果您希望將節流的例外狀況回報給應用程式的例外狀況處理器，您可以在將中介層附加到 Job 時呼叫 `report` 方法。或者，您可以為 `report` 方法提供一個閉包，且僅當該閉包回傳 `true` 時，例外狀況才會被回報：

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
#### 使用 Redis 節流例外狀況

如果您正在使用 Redis，可以使用 `Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis` 中介層，它針對 Redis 進行了最佳化，且比基礎的例外狀況節流中介層更有效率：

```php
use Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis;

public function middleware(): array
{
    return [new ThrottlesExceptionsWithRedis(10, 10 * 60)];
}
```

可以使用 `connection` 方法指定中介層應使用的 Redis 連線：

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
## 派遣 Job

撰寫好 Job 類別後，你可以使用 Job 本身的 `dispatch` 方法來派遣它。傳遞給 `dispatch` 方法的參數將會被傳遞給 Job 的建構子：

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

如果你想根據條件派遣 Job，可以使用 `dispatchIf` 與 `dispatchUnless` 方法：

```php
ProcessPodcast::dispatchIf($accountActive, $podcast);

ProcessPodcast::dispatchUnless($accountSuspended, $podcast);
```

在新的 Laravel 應用程式中，`database` 連線被定義為預設隊列。你可以透過更改應用程式 `.env` 檔案中的 `QUEUE_CONNECTION` 環境變數來指定不同的預設隊列連線。


<a name="delayed-dispatching"></a>
### 延遲派遣

如果你想指定 Job 不應立即由隊列 Worker 處理，可以在派遣 Job 時使用 `delay` 方法。例如，讓我們指定一個 Job 在派遣 10 分鐘後才能處理：

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

在某些情況下，Job 可能已設定了預設延遲。如果你需要繞過此延遲並立即派遣 Job 處理，可以使用 `withoutDelay` 方法：

```php
ProcessPodcast::dispatch($podcast)->withoutDelay();
```

> [!WARNING]
> Amazon SQS 隊列服務的最大延遲時間為 15 分鐘。


<a name="synchronous-dispatching"></a>
### 同步派遣

如果你想立即（同步）派遣 Job，可以使用 `dispatchSync` 方法。使用此方法時，Job 將不會進入隊列，而是會立即在目前程序中執行：

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
#### 延遲派遣 (Deferred Dispatching)

透過使用延遲同步派遣，你可以將 Job 派遣到目前的程序中處理，但會在 HTTP 回應傳送給使用者之後才執行。這讓你可以同步處理「隊列」Job，而不會減慢使用者的應用程式體驗。若要延遲執行同步 Job，請將 Job 派遣至 `deferred` 連線：

```php
RecordDelivery::dispatch($order)->onConnection('deferred');
```

`deferred` 連線同時也作為預設的 [隊列容錯移轉 (Failover)](#queue-failover)。

同樣地，`background` 連線會在 HTTP 回應傳送給使用者後處理 Job；然而，Job 會在獨立生成的 PHP 程序中處理，讓 PHP-FPM / 應用程式 Worker 能繼續處理另一個傳入的 HTTP 請求：

```php
RecordDelivery::dispatch($order)->onConnection('background');
```


<a name="jobs-and-database-transactions"></a>
### Job 與資料庫交易

雖然在資料庫交易中派遣 Job 是完全沒問題的，但你應該特別小心，以確保你的 Job 實際上能夠成功執行。在交易中派遣 Job 時，Worker 可能會在父交易提交 (Commit) 之前就嘗試執行該 Job。發生這種情況時，你在資料庫交易中所做的模型或資料庫紀錄更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫紀錄也可能不存在於資料庫中。

慶幸的是，Laravel 提供了多種解決此問題的方法。首先，你可以在隊列連線的設定陣列中設定 `after_commit` 連線選項：

```php
'redis' => [
    'driver' => 'redis',
    // ...
    'after_commit' => true,
],
```

當 `after_commit` 選項為 `true` 時，你可以在資料庫交易中派遣 Job；然而，Laravel 會等到所有開啟的父資料庫交易都提交後，才會實際派遣該 Job。當然，如果目前沒有開啟任何資料庫交易，該 Job 將會立即被派遣。

如果交易因交易過程中發生的例外狀況而回滾 (Roll back)，在該交易期間派遣的 Job 將會被丟棄。

> [!NOTE]
> 將 `after_commit` 設定選項設為 `true` 也會導致任何隊列化事件監聽器、Mailable、通知及廣播事件在所有開啟的資料庫交易提交後才被派遣。


<a name="specifying-commit-dispatch-behavior-inline"></a>
#### 在行內指定提交派遣行為

如果你沒有將 `after_commit` 隊列連線設定選項設為 `true`，你仍然可以指示特定 Job 應在所有開啟的資料庫交易提交後才被派遣。若要達成此目的，你可以在派遣操作後鏈結 `afterCommit` 方法：

```php
use App\Jobs\ProcessPodcast;

ProcessPodcast::dispatch($podcast)->afterCommit();
```

同樣地，如果 `after_commit` 設定選項被設為 `true`，你可以指示特定 Job 應立即派遣，而無需等待任何開啟的資料庫交易提交：

```php
ProcessPodcast::dispatch($podcast)->beforeCommit();
```

<a name="job-chaining"></a>
### Job 鏈結

Job 鏈結允許您指定一系列在主 Job 執行成功後依序執行的隊列 Job。如果序列中的某個 Job 失敗，則其餘 Job 將不會執行。若要執行隊列 Job 鏈結，您可以使用 `Bus` Facade 提供的 `chain` 方法。Laravel 的指令匯流排 (Command Bus) 是建構隊列 Job 派遣功能之下的底層組件：

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

除了鏈結 Job 類別實例之外，您也可以鏈結閉包：

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
> 使用 Job 內的 `$this->delete()` 方法刪除 Job 並不會阻止鏈結中的後續 Job 被處理。只有在鏈結中的某個 Job 失敗時，鏈結才會停止執行。


<a name="chain-connection-queue"></a>
#### 鏈結連線與隊列

如果您想要指定鏈結 Job 應使用的連線與隊列，可以使用 `onConnection` 與 `onQueue` 方法。除非隊列 Job 已明確分配了不同的連線 / 隊列，否則這些方法會指定應使用的隊列連線與隊列名稱：

```php
Bus::chain([
    new ProcessPodcast,
    new OptimizePodcast,
    new ReleasePodcast,
])->onConnection('redis')->onQueue('podcasts')->dispatch();
```


<a name="adding-jobs-to-the-chain"></a>
#### 向鏈結添加 Job

有時，您可能需要從鏈結中的某個 Job 內部，向現有的 Job 鏈結開頭或結尾添加 Job。您可以使用 `prependToChain` 與 `appendToChain` 方法來完成此操作：

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
#### 鏈結失敗處理

鏈結 Job 時，您可以使用 `catch` 方法指定一個在鏈結中的 Job 失敗時應調用的閉包。該回呼函式將接收導致 Job 失敗的 `Throwable` 實例：

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
> 由於鏈結回呼會被序列化並由 Laravel 隊列在稍後執行，因此您不應在鏈結回呼中使用 `$this` 變數。


<a name="customizing-the-queue-and-connection"></a>
### 自定義隊列與連線


<a name="dispatching-to-a-particular-queue"></a>
#### 派遣至特定隊列

藉由將 Job 推送到不同的隊列，您可以將隊列 Job 進行「分類」，甚至可以設定分配給各個隊列的 Worker 優先權。請記住，這並不會將 Job 推送到隊列設定檔中定義的不同隊列「連線 (Connections)」，而僅是推送到單一連線中的特定隊列。若要指定隊列，請在派遣 Job 時使用 `onQueue` 方法：

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

或者，您也可以在 Job 的建構子內調用 `onQueue` 方法來指定 Job 的隊列：

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
#### 派遣至特定連線

如果您的應用程式與多個隊列連線進行互動，您可以使用 `onConnection` 方法來指定要將 Job 推送到哪個連線：

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

您可以將 `onConnection` 與 `onQueue` 方法鏈結在一起，為 Job 指定連線與隊列：

```php
ProcessPodcast::dispatch($podcast)
    ->onConnection('sqs')
    ->onQueue('processing');
```

或者，您也可以在 Job 的建構子內調用 `onConnection` 方法來指定 Job 的連線：

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
### 指定 Job 最大重試次數 / 超時數值

<a name="max-attempts"></a>
#### 最大嘗試次數

Job 嘗試次數是 Laravel 隊列系統的核心概念，並支援許多進階功能。雖然起初可能看起來令人困惑，但在修改預設設定之前，了解它們的工作原理非常重要。

當一個 Job 被派遣時，它會被推送到隊列中。接著一個 Worker 會取得它並嘗試執行它。這就是一次 Job 嘗試 (Attempt)。

然而，一次嘗試並不一定意味著 Job 的 `handle` 方法已被執行。嘗試也可能透過以下幾種方式被「消耗」：

<div class="content-list" markdown="1">

- Job 在執行期間遇到未處理的例外狀況。
- 使用 `$this->release()` 手動將 Job 釋放回隊列。
- 中介層（如 `WithoutOverlapping` 或 `RateLimited`）未能取得鎖定並釋放 Job。
- Job 超時。
- Job 的 `handle` 方法運行並完成，且未拋出例外狀況。

</div>

您可能不希望無限期地持續嘗試某個 Job。因此，Laravel 提供了多種方式來指定一個 Job 可以嘗試的次數或時間長度。

> [!NOTE]
> 預設情況下，Laravel 只會嘗試 Job 一次。如果您的 Job 使用了像 `WithoutOverlapping` 或 `RateLimited` 這樣的中介層，或者如果您手動釋放 Job，您可能需要透過 `tries` 選項增加允許的嘗試次數。

指定 Job 最大嘗試次數的一種方法是透過 Artisan 命令列上的 `--tries` 切換開關。除非正在處理的 Job 指定了它可以嘗試的次數，否則這將套用於該 Worker 處理的所有 Job：

```shell
php artisan queue:work --tries=3
```

如果一個 Job 超過了其最大嘗試次數，它將被視為「失敗」的 Job。有關處理失敗 Job 的更多資訊，請參閱[處理失敗的 Job 文件](#dealing-with-failed-jobs)。如果為 `queue:work` 指令提供了 `--tries=0`，則 Job 將無限期重試。

您可以透過在 Job 類別本身定義最大嘗試次數，來採取更細粒度的方法。如果在 Job 上指定了最大嘗試次數，它的優先級將高於命令列上提供的 `--tries` 數值：

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

如果您需要對特定 Job 的最大嘗試次數進行動態控制，可以在 Job 上定義一個 `tries` 方法：

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

除了定義 Job 在失敗前可以嘗試多少次之外，您還可以定義一個不再嘗試 Job 的時間點。這允許在給定的時間範圍內嘗試 Job 任意次數。要定義不再嘗試 Job 的時間，請在您的 Job 類別中新增一個 `retryUntil` 方法。此方法應回傳一個 `DateTime` 執行個體：

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

如果同時定義了 `retryUntil` 和 `tries`，Laravel 會優先使用 `retryUntil` 方法。

> [!NOTE]
> 您也可以在[隊列化事件監聽器](/docs/{{version}}/events#queued-event-listeners)和[隊列化通知](/docs/{{version}}/notifications#queueing-notifications)中定義 `tries` 屬性或 `retryUntil` 方法。

<a name="max-exceptions"></a>
#### 最大例外狀況次數

有時您可能希望指定 Job 可以嘗試多次，但如果重試是由於給定數量的未處理例外狀況觸發的（而不是直接由 `release` 方法釋放），則該 Job 應該失敗。要達成此目的，您可以在 Job 類別中定義 `maxExceptions` 屬性：

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

在此範例中，如果應用程式無法取得 Redis 鎖定，Job 將被釋放 10 秒，並將繼續重試最多 25 次。但是，如果 Job 拋出三個未處理的例外狀況，則 Job 將失敗。

<a name="timeout"></a>
#### 超時

通常，您大致知道預期的隊列 Job 需要多長時間。因此，Laravel 允許您指定一個「超時 (timeout)」值。預設情況下，超時值為 60 秒。如果 Job 處理的時間超過超時值指定的秒數，處理該 Job 的 Worker 將因錯誤而結束。通常，Worker 會由[伺服器上設定的程序管理器](#supervisor-configuration)自動重啟。

可以使用 Artisan 命令列上的 `--timeout` 切換開關來指定 Job 可運行的最大秒數：

```shell
php artisan queue:work --timeout=30
```

如果 Job 由於持續超時而超過其最大嘗試次數，它將被標記為失敗。

您也可以在 Job 類別本身定義 Job 允許運行的最大秒數。如果在 Job 上指定了超時，它的優先級將高於命令列上指定的任何超時：

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

有時，IO 阻塞程序（如 socket 或外部 HTTP 連線）可能不會遵守您指定的超時。因此，在使用這些功能時，您也應該始終嘗試使用它們的 API 來指定超時。例如，使用 [Guzzle](https://docs.guzzlephp.org) 時，您應該始終指定連線和請求的超時值。

> [!WARNING]
> 必須安裝 [PCNTL](https://www.php.net/manual/en/book.pcntl.php) PHP 擴充功能才能指定 Job 超時。此外，Job 的 「timeout」 值應始終小於其 [「retry after」](#job-expiration) 值。否則，Job 可能在實際執行完成或超時之前就已經被重試。

<a name="failing-on-timeout"></a>
#### 超時時失敗

如果您想指明 Job 在超時時應被標記為[失敗](#dealing-with-failed-jobs)，您可以在 Job 類別中定義 `$failOnTimeout` 屬性：

```php
/**
 * Indicate if the job should be marked as failed on timeout.
 *
 * @var bool
 */
public $failOnTimeout = true;
```

> [!NOTE]
> 預設情況下，當 Job 超時時，它會消耗一次嘗試並被釋放回隊列（如果允許重試）。但是，如果您將 Job 設定為在超時時失敗，則無論 tries 設定為多少，它都不會被重試。

<a name="sqs-fifo-and-fair-queues"></a>
### SQS FIFO 與公平隊列

Laravel 支援 [Amazon SQS FIFO (先入先出)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html) 隊列，讓您可以按照 Job 發送的精確順序進行處理，同時透過訊息去重 (Message deduplication) 確保僅限一次 (Exactly-once) 的處理。

FIFO 隊列需要一個訊息群組識別碼 (Message Group ID) 來判斷哪些 Job 可以並行處理。具有相同群組識別碼的 Job 會依序處理，而具有不同群組識別碼的訊息則可以同時處理。

Laravel 提供了一個流暢的 `onGroup` 方法，用於在派遣 Job 時指定訊息群組識別碼：

```php
ProcessOrder::dispatch($order)
    ->onGroup("customer-{$order->customer_id}");
```

SQS FIFO 隊列支援訊息去重以確保僅限一次的處理。請在您的 Job 類別中實作 `deduplicationId` 方法以提供自定義的去重識別碼：

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

使用 FIFO 隊列時，您還需要在監聽器、郵件與通知上定義訊息群組。或者，您可以將這些物件的隊列化實例派遣到非 FIFO 隊列。

若要為 [隊列化事件監聽器](/docs/{{version}}/events#queued-event-listeners) 定義訊息群組，請在監聽器上定義 `messageGroup` 方法。您也可以選擇定義 `deduplicationId` 方法：

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

當發送要推送到 FIFO 隊列的 [郵件訊息](/docs/{{version}}/mail) 時，您應該在發送通知時呼叫 `onGroup` 方法，並可選擇呼叫 `withDeduplicator` 方法：

```php
use App\Mail\InvoicePaid;
use Illuminate\Support\Facades\Mail;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

Mail::to($request->user())->send($invoicePaid);
```

當發送要推送到 FIFO 隊列的 [通知](/docs/{{version}}/notifications) 時，您應該在發送通知時呼叫 `onGroup` 方法，並可選擇呼叫 `withDeduplicator` 方法：

```php
use App\Notifications\InvoicePaid;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

$user->notify($invoicePaid);
```

<a name="queue-failover"></a>
### 隊列容錯移轉 (Failover)

`failover` 隊列驅動程式在將 Job 推送到隊列時提供自動容錯移轉功能。如果 `failover` 設定中的主要隊列連線因任何原因失敗，Laravel 將自動嘗試將 Job 推送到清單中的下一個設定連線。這對於確保正式環境中的高可用性特別有用，因為在這些環境中隊列的可靠性至關重要。

若要設定容錯移轉隊列連線，請指定 `failover` 驅動程式並提供一個要依序嘗試的連線名稱陣列。預設情況下，Laravel 在應用程式的 `config/queue.php` 設定檔中包含了一個範例容錯移轉設定：

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

設定好使用 `failover` 驅動程式的連線後，您需要將應用程式 `.env` 檔案中的預設隊列連線設置為該容錯移轉連線，以使用容錯移轉功能：

```ini
QUEUE_CONNECTION=failover
```

接下來，為容錯移轉連線清單中的每個連線啟動至少一個 Worker：

```bash
php artisan queue:work redis
php artisan queue:work database
```

> [!NOTE]
> 您不需要為使用 `sync`、`background` 或 `deferred` 隊列驅動程式的連線執行 Worker，因為這些驅動程式會在當前的 PHP 程序中處理 Job。

當隊列連線操作失敗且啟動容錯移轉時，Laravel 將發送 `Illuminate\Queue\Events\QueueFailedOver` 事件，讓您可以回報或記錄隊列連線已失敗。

> [!NOTE]
> 如果您使用 Laravel Horizon，請記住 Horizon 僅管理 Redis 隊列。如果您的容錯移轉清單包含 `database`，您應該在 Horizon 之外執行常規的 `php artisan queue:work database` 程序。

<a name="error-handling"></a>
### 錯誤處理

若在處理 Job 時拋出例外狀況，該 Job 會自動釋放回隊列中以便再次嘗試。Job 會持續被釋放，直到達到應用程式允許的最大嘗試次數為止。最大嘗試次數是由 `queue:work` Artisan 指令使用的 `--tries` 開關定義的。或者，也可以在 Job 類別本身定義最大嘗試次數。更多關於執行隊列 Worker 的資訊[可以在下方找到](#running-the-queue-worker)。


<a name="manually-releasing-a-job"></a>
#### 手動釋放 Job

有時您可能希望手動將 Job 釋放回隊列，以便稍後再次嘗試。您可以透過呼叫 `release` 方法來完成此操作：

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

預設情況下，`release` 方法會將 Job 釋放回隊列以供立即處理。不過，您可以透過向 `release` 方法傳遞一個整數或日期實例，指示隊列在經過指定的秒數之前不要讓該 Job 可供處理：

```php
$this->release(10);

$this->release(now()->plus(seconds: 10));
```


<a name="manually-failing-a-job"></a>
#### 手動標記 Job 為失敗

偶爾您可能需要手動將 Job 標記為「失敗」。為此，您可以呼叫 `fail` 方法：

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

如果您想因為擷取到的例外狀況而將 Job 標記為失敗，可以將例外狀況傳遞給 `fail` 方法。或者為了方便起見，您可以傳遞一個字串錯誤訊息，它將為您轉換為一個例外狀況：

```php
$this->fail($exception);

$this->fail('Something went wrong.');
```

> [!NOTE]
> 關於失敗 Job 的更多資訊，請查看[處理失敗 Job 的文件](#dealing-with-failed-jobs)。


<a name="fail-jobs-on-exceptions"></a>
#### 在特定例外狀況下使 Job 失敗

`FailOnException` [Job 中介層](#job-middleware) 允許您在拋出特定例外狀況時提早結束 (Short-circuit) 重試。這允許在發生暫時性例外狀況（如外部 API 錯誤）時進行重試，但在發生持續性例外狀況（如使用者的權限被撤銷）時讓 Job 永久失敗：

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

Laravel 的 Job 批次處理功能讓您可以輕鬆地執行一組 Job，並在該批次 Job 執行完成後執行某些操作。在開始之前，您應該建立一個資料庫遷移來建立一張資料表，該表將包含有關 Job 批次的 meta 資訊，例如它們的完成百分比。可以使用 `make:queue-batches-table` Artisan 指令生成此遷移：

```shell
php artisan make:queue-batches-table

php artisan migrate
```


<a name="defining-batchable-jobs"></a>
### 定義可批次處理的 Job

要定義一個可批次處理的 Job，您應該像平常一樣[建立一個可隊列化的 Job](#creating-jobs)；但是，您應該在 Job 類別中加入 `Illuminate\Bus\Batchable` trait。這個 trait 提供了對 `batch` 方法的存取，該方法可用於檢索 Job 目前正在其中執行的批次：

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
### 派遣批次

要派遣一個 Job 批次，您應該使用 `Bus` Facade 的 `batch` 方法。當然，批次處理在與完成回呼 (Callback) 結合使用時最為有用。因此，您可以使用 `then`、`catch` 和 `finally` 方法來為批次定義完成回呼。這些回呼在被調用時都將接收一個 `Illuminate\Bus\Batch` 實例。

當執行多個隊列 Worker 時，批次中的 Job 將會並行處理。因此，Job 完成的順序可能與它們被添加到批次的順序不同。請參考我們關於 [Job 鏈結與批次](#chains-and-batches) 的文件，以瞭解如何依序執行一系列 Job。

在下面這個範例中，我們假設正在排隊一個 Job 批次，每個 Job 都處理 CSV 檔案中給定數量的資料列：

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

批次的 ID（可以透過 `$batch->id` 屬性存取）可用於在批次派遣後，向 Laravel 命令匯流排 (Command Bus) [查詢批次相關資訊](#inspecting-batches)。

> [!WARNING]
> 由於批次回呼會被序列化並由 Laravel 隊列在稍後執行，因此您不應在回呼中使用 `$this` 變數。此外，由於批次 Job 是在資料庫交易中包裝的，因此不應在 Job 中執行會觸發隱式提交 (Implicit Commit) 的資料庫陳述式。


<a name="naming-batches"></a>
#### 命名批次

如果為批次命名，某些工具（如 [Laravel Horizon](/docs/{{version}}/horizon) 和 [Laravel Telescope](/docs/{{version}}/telescope)）可能會為批次提供更易於閱讀的偵錯資訊。要為批次分配一個任意名稱，您可以在定義批次時調用 `name` 方法：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import CSV')->dispatch();
```


<a name="batch-connection-queue"></a>
#### 批次連線與隊列

如果您想指定用於批次 Job 的連線和隊列，可以使用 `onConnection` 和 `onQueue` 方法。所有批次 Job 必須在相同的連線和隊列中執行：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->onConnection('redis')->onQueue('imports')->dispatch();
```


<a name="chains-and-batches"></a>
### 鏈結與批次

您可以透過將鏈結的 Job 放在陣列中，在批次內定義一組 [鏈結 Job](#job-chaining)。例如，我們可以並行執行兩個 Job 鏈結，並在兩個 Job 鏈結都處理完成時執行回呼：

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

相反地，您可以透過在鏈結中定義批次，在一個 [鏈結](#job-chaining) 中執行多個 Job 批次。例如，您可以先執行一個 Job 批次來發佈多個 Podcast，然後執行另一個 Job 批次來發送發佈通知：

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
### 向批次添加 Job

有時從批次 Job 內部向批次添加額外的 Job 會很有用。當您需要批次處理數千個 Job，而這些 Job 在 Web 請求期間派遣耗時過長時，這種模式會非常有用。因此，您可能希望先派遣一個初始的「載入器 (Loader)」Job 批次，由它為批次填充更多的 Job：

```php
$batch = Bus::batch([
    new LoadImportBatch,
    new LoadImportBatch,
    new LoadImportBatch,
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import Contacts')->dispatch();
```

在這個範例中，我們將使用 `LoadImportBatch` Job 來為批次填充額外的 Job。為了實現這一點，我們可以使用批次實例上的 `add` 方法，該實例可以透過 Job 的 `batch` 方法取得：

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
> 您只能從屬於同一個批次的 Job 內部向該批次添加 Job。

<a name="inspecting-batches"></a>
### 檢查批次

提供給批次完成回呼 (Callback) 的 `Illuminate\Bus\Batch` 執行個體擁有多種屬性與方法，可協助您與指定的 Job 批次進行互動與檢查：

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

所有的 `Illuminate\Bus\Batch` 執行個體都是可 JSON 序列化的，這意味著您可以直接從應用程式的其中一個路由回傳它們，以取得包含該批次資訊（包括其完成進度）的 JSON 酬載。這使得在應用程式的 UI 中顯示批次完成進度的資訊變得非常方便。

要透過 ID 取得批次，您可以使用 `Bus` Facade 的 `findBatch` 方法：

```php
use Illuminate\Support\Facades\Bus;
use Illuminate\Support\Facades\Route;

Route::get('/batch/{batchId}', function (string $batchId) {
    return Bus::findBatch($batchId);
});
```


<a name="cancelling-batches"></a>
### 取消批次

有時您可能需要取消特定批次的執行。這可以透過呼叫 `Illuminate\Bus\Batch` 執行個體上的 `cancel` 方法來達成：

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

正如您在之前的範例中所注意到的，批次處理的 Job 通常應該在繼續執行之前判斷其對應的批次是否已被取消。然而，為了方便起見，您可以改為將 `SkipIfBatchCancelled` [中介層](#job-middleware) 指派給該 Job。顧名思義，此中介層將指示 Laravel 在其對應的批次已被取消時不要處理該 Job：

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

當批次 Job 失敗時，會調用 `catch` 回呼（如果已指派）。此回呼僅針對批次中第一個失敗的 Job 調用。


<a name="allowing-failures"></a>
#### 允許失敗

當批次中的 Job 失敗時，Laravel 會自動將該批次標記為「已取消」。如果您願意，可以停用此行為，使 Job 失敗不會自動將批次標記為已取消。這可以透過在派遣批次時呼叫 `allowFailures` 方法來達成：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->allowFailures()->dispatch();
```

您可以選擇性地向 `allowFailures` 方法提供一個閉包 (Closure)，該閉包將在每次 Job 失敗時執行：

```php
$batch = Bus::batch([
    // ...
])->allowFailures(function (Batch $batch, $exception) {
    // Handle individual job failures...
})->dispatch();
```


<a name="retrying-failed-batch-jobs"></a>
#### 重試失敗的批次 Job

為了方便起見，Laravel 提供了一個 `queue:retry-batch` Artisan 指令，讓您可以輕鬆地為指定批次重試所有失敗的 Job。此指令接受應重試其失敗 Job 的批次 UUID：

```shell
php artisan queue:retry-batch 32dbc76c-4f82-4749-b610-a639fe0099b5
```


<a name="pruning-batches"></a>
### 修剪批次

如果不進行修剪，`job_batches` 資料表會非常迅速地累積紀錄。為了減輕這種情況，您應該[排程](/docs/{{version}}/scheduling)每日執行 `queue:prune-batches` Artisan 指令：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches')->daily();
```

預設情況下，所有已完成且超過 24 小時的批次都將被修剪。您可以在呼叫指令時使用 `hours` 選項來決定保留批次資料的時間。例如，以下指令將刪除所有在 48 小時前完成的批次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48')->daily();
```

有時，您的 `jobs_batches` 資料表可能會累積那些從未成功完成的批次紀錄，例如某個 Job 失敗且該 Job 從未成功重試的批次。您可以透過 `unfinished` 選項指示 `queue:prune-batches` 指令修剪這些未完成的批次紀錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --unfinished=72')->daily();
```

同樣地，您的 `jobs_batches` 資料表也可能累積已取消批次的紀錄。您可以透過 `cancelled` 選項指示 `queue:prune-batches` 指令修剪這些已取消的批次紀錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --cancelled=72')->daily();
```

<a name="storing-batches-in-dynamodb"></a>
### 在 DynamoDB 中儲存批次

Laravel 也支援將批次的元數據 (meta information) 儲存在 [DynamoDB](https://aws.amazon.com/dynamodb) 中，而非關聯式資料庫。然而，您需要手動建立一個 DynamoDB 資料表來儲存所有的批次紀錄。

通常，這個資料表應命名為 `job_batches`，但您應該根據應用程式 `queue` 設定檔中的 `queue.batching.table` 設定值來命名該資料表。


<a name="dynamodb-batch-table-configuration"></a>
#### DynamoDB 批次資料表設定

`job_batches` 資料表應該有一個名為 `application` 的字串型別主分割鍵 (Primary Partition Key) 以及一個名為 `id` 的字串型別主排序鍵 (Primary Sort Key)。鍵值的 `application` 部分將包含您的應用程式名稱，該名稱定義於應用程式 `app` 設定檔中的 `name` 設定值。由於應用程式名稱是 DynamoDB 資料表鍵的一部分，因此您可以使用同一個資料表來儲存多個 Laravel 應用程式的 Job 批次。

此外，如果您想利用 [自動批次修剪](#pruning-batches-in-dynamodb) 功能，您可以為資料表定義 `ttl` 屬性。


<a name="dynamodb-configuration"></a>
#### DynamoDB 設定

接著，安裝 AWS SDK 以便您的 Laravel 應用程式能與 Amazon DynamoDB 進行通訊：

```shell
composer require aws/aws-sdk-php
```

然後，將 `queue.batching.driver` 設定選項的值設為 `dynamodb`。此外，您應該在 `batching` 設定陣列中定義 `key`、`secret` 與 `region` 設定選項。這些選項將用於 AWS 的身分驗證。當使用 `dynamodb` 驅動程式時，`queue.batching.database` 設定選項是不需要的：

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

當使用 [DynamoDB](https://aws.amazon.com/dynamodb) 儲存 Job 批次資訊時，用於修剪關聯式資料庫中批次的典型修剪指令將無法運作。相反地，您可以利用 [DynamoDB 原生的 TTL 功能](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html) 來自動移除舊批次的紀錄。

如果您為 DynamoDB 資料表定義了 `ttl` 屬性，您可以定義設定參數來指示 Laravel 如何修剪批次紀錄。`queue.batching.ttl_attribute` 設定值定義了持有 TTL 的屬性名稱，而 `queue.batching.ttl` 設定值則定義了批次紀錄從上次更新後，經過多少秒可以從 DynamoDB 資料表中移除：

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
## 隊列化閉包

除了將 Job 類別派遣到隊列之外，您也可以派遣一個閉包。這對於需要在當前請求週期之外執行的快速、簡單任務非常有用。當將閉包派遣到隊列時，閉包的程式碼內容會經過加密簽署，因此在傳輸過程中無法被修改：

```php
use App\Models\Podcast;

$podcast = Podcast::find(1);

dispatch(function () use ($podcast) {
    $podcast->publish();
});
```

若要為隊列化閉包指定一個名稱（可用於隊列報表儀表板，也會顯示在 `queue:work` 指令中），您可以使用 `name` 方法：

```php
dispatch(function () {
    // ...
})->name('Publish Podcast');
```

使用 `catch` 方法，您可以提供一個閉包，當隊列化閉包在用盡所有隊列[設定的重試次數](#max-job-attempts-and-timeout)後仍未能成功完成時，該閉包將被執行：

```php
use Throwable;

dispatch(function () use ($podcast) {
    $podcast->publish();
})->catch(function (Throwable $e) {
    // This job has failed...
});
```

> [!WARNING]
> 由於 `catch` 回呼會被序列化並在稍後由 Laravel 隊列執行，因此您不應在 `catch` 回呼中使用 `$this` 變數。

<a name="running-the-queue-worker"></a>
## 執行隊列 Worker


<a name="the-queue-work-command"></a>
### `queue:work` 指令

Laravel 包含一個 Artisan 指令，它將啟動一個隊列 Worker 並處理被推送到隊列的新 Job。您可以使用 `queue:work` Artisan 指令來執行 Worker。請注意，一旦 `queue:work` 指令啟動，它將持續運行，直到手動停止或關閉終端機：

```shell
php artisan queue:work
```

> [!NOTE]
> 為了讓 `queue:work` 處理程序永久在背景運行，您應該使用諸如 [Supervisor](#supervisor-configuration) 之類的處理程序監控器，以確保隊列 Worker 不會停止運行。

如果您希望在指令的輸出中包含已處理的 Job ID、連線名稱和隊列名稱，可以在調用 `queue:work` 指令時加入 `-v` 旗標：

```shell
php artisan queue:work -v
```

請記住，隊列 Worker 是長時間運行的處理程序，並將已啟動的應用程式狀態儲存在記憶體中。因此，它們在啟動後不會察覺到程式碼庫中的變動。所以，在您的部署過程中，請務必 [重新啟動您的隊列 Worker](#queue-workers-and-deployment)。此外，請記住，應用程式建立或修改的任何靜態狀態都不會在 Job 之間自動重設。

或者，您可以執行 `queue:listen` 指令。使用 `queue:listen` 指令時，當您想要重新載入更新後的程式碼或重設應用程式狀態時，不需要手動重新啟動 Worker；然而，此指令的效率明顯低於 `queue:work` 指令：

```shell
php artisan queue:listen
```


<a name="running-multiple-queue-workers"></a>
#### 執行多個隊列 Worker

要為隊列分配多個 Worker 並同時處理 Job，您只需啟動多個 `queue:work` 處理程序即可。這可以在本地端透過終端機的多個分頁來完成，或者在正式環境中使用處理程序管理員的設定來完成。[使用 Supervisor](#supervisor-configuration) 時，您可以使用 `numprocs` 設定值。


<a name="specifying-the-connection-queue"></a>
#### 指定連線與隊列

您也可以指定 Worker 應使用的隊列連線。傳遞給 `work` 指令的連線名稱應對應於 `config/queue.php` 設定檔中定義的其中一個連線：

```shell
php artisan queue:work redis
```

預設情況下，`queue:work` 指令僅處理給定連線上預設隊列的 Job。然而，您可以透過僅處理給定連線上的特定隊列來進一步自定義您的隊列 Worker。例如，如果您所有的電子郵件都在 `redis` 隊列連線上的 `emails` 隊列中處理，您可以發布以下指令來啟動一個僅處理該隊列的 Worker：

```shell
php artisan queue:work redis --queue=emails
```


<a name="processing-a-specified-number-of-jobs"></a>
#### 處理指定數量的 Job

`--once` 選項可用於指示 Worker 僅處理隊列中的單個 Job：

```shell
php artisan queue:work --once
```

`--max-jobs` 選項可用於指示 Worker 處理給定數量的 Job 後退出。此選項在與 [Supervisor](#supervisor-configuration) 結合使用時非常有用，這樣您的 Worker 就可以在處理完給定數量的 Job 後自動重新啟動，從而釋放它們可能累積的任何記憶體：

```shell
php artisan queue:work --max-jobs=1000
```


<a name="processing-all-queued-jobs-then-exiting"></a>
#### 處理所有已入隊列的 Job 後結束

`--stop-when-empty` 選項可用於指示 Worker 處理完所有 Job 後優雅地退出。如果您希望在隊列為空後關閉容器，則此選項在 Docker 容器內處理 Laravel 隊列時非常有用：

```shell
php artisan queue:work --stop-when-empty
```


<a name="processing-jobs-for-a-given-number-of-seconds"></a>
#### 處理 Job 達指定的秒數後結束

`--max-time` 選項可用於指示 Worker 處理 Job 達給定的秒數後退出。此選項在與 [Supervisor](#supervisor-configuration) 結合使用時非常有用，這樣您的 Worker 就可以在處理 Job 達給定的時間後自動重新啟動，從而釋放它們可能累積的任何記憶體：

```shell
# Process jobs for one hour and then exit...
php artisan queue:work --max-time=3600
```


<a name="worker-sleep-duration"></a>
#### Worker 休眠時間

當隊列中有 Job 可用時，Worker 將持續處理 Job，且 Job 之間沒有延遲。然而，`sleep` 選項決定了如果沒有可用的 Job，Worker 將「休眠」多少秒。當然，在休眠期間，Worker 不會處理任何新 Job：

```shell
php artisan queue:work --sleep=3
```


<a name="maintenance-mode-queues"></a>
#### 維護模式與隊列

當您的應用程式處於 [維護模式](/docs/{{version}}/configuration#maintenance-mode) 時，將不會處理任何已入隊列的 Job。一旦應用程式脫離維護模式，Job 將照常繼續處理。

要強制隊列 Worker 即使在啟用了維護模式的情況下也處理 Job，您可以使用 `--force` 選項：

```shell
php artisan queue:work --force
```


<a name="resource-considerations"></a>
#### 資源考量

常駐型隊列 Worker 在處理每個 Job 之前不會「重新啟動」框架。因此，您應該在每個 Job 完成後釋放任何沉重的資源。例如，如果您正在使用 [GD 函式庫](https://www.php.net/manual/en/book.image.php) 進行圖片操作，則應在處理完圖片後使用 `imagedestroy` 釋放記憶體。


<a name="queue-priorities"></a>
### 隊列優先權

有時您可能希望決定隊列處理的優先順序。例如，在您的 `config/queue.php` 設定檔中，您可以將 `redis` 連線的預設 `queue` 設置為 `low`。然而，有時您可能希望將一個 Job 推送到 `high` 優先權隊列，如下所示：

```php
dispatch((new Job)->onQueue('high'));
```

要啟動一個 Worker，確保在繼續處理 `low` 隊列上的任何 Job 之前處理所有的 `high` 隊列 Job，請向 `work` 指令傳遞以逗號分隔的隊列名稱列表：

```shell
php artisan queue:work --queue=high,low
```


<a name="queue-workers-and-deployment"></a>
### 隊列 Worker 與部署

由於隊列 Worker 是長時間運行的處理程序，因此如果不重新啟動，它們將不會察覺到程式碼的變動。因此，部署使用隊列 Worker 的應用程式最簡單的方法是在部署過程中重新啟動 Worker。您可以透過發布 `queue:restart` 指令來優雅地重新啟動所有 Worker：

```shell
php artisan queue:restart
```

此指令將指示所有隊列 Worker 在完成處理當前 Job 後優雅地退出，以免丟失任何現有的 Job。由於隊列 Worker 將在執行 `queue:restart` 指令時退出，因此您應該執行諸如 [Supervisor](#supervisor-configuration) 之類的處理程序管理員來自動重新啟動隊列 Worker。

> [!NOTE]
> 隊列使用 [快取](/docs/{{version}}/cache) 來儲存重新啟動訊號，因此在使用此功能之前，您應該確認已為應用程式正確設定了快取驅動程式。

<a name="job-expirations-and-timeouts"></a>
### Job 過期與超時


<a name="job-expiration"></a>
#### Job 過期

在您的 `config/queue.php` 設定檔中，每個隊列連線都定義了一個 `retry_after` 選項。此選項指定了隊列連線在重試正在處理的 Job 之前應該等待多少秒。例如，如果 `retry_after` 的值設定為 `90`，那麼如果該 Job 已處理 90 秒且尚未被釋放或刪除，它將被放回隊列中。通常，您應該將 `retry_after` 的值設定為您的 Job 完成處理合理所需的最大秒數。

> [!WARNING]
> 唯一不包含 `retry_after` 值的隊列連線是 Amazon SQS。SQS 將根據在 AWS 控制台中管理的 [預設可見性超時 (Default Visibility Timeout)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html) 來重試 Job。


<a name="worker-timeouts"></a>
#### Worker 超時

`queue:work` Artisan 指令提供了一個 `--timeout` 選項。預設情況下，`--timeout` 的值為 60 秒。如果 Job 處理的時間超過超時數值所指定的秒數，處理該 Job 的 Worker 將會出錯並退出。通常，Worker 會被您[伺服器上設定的行程管理員](#supervisor-configuration)自動重啟：

```shell
php artisan queue:work --timeout=60
```

`retry_after` 設定選項和 `--timeout` CLI 選項是不同的，但它們共同運作以確保 Job 不會遺失，並且 Job 僅會被成功處理一次。

> [!WARNING]
> `--timeout` 的值應始終比您的 `retry_after` 設定值短至少幾秒鐘。這將確保處理凍結 Job 的 Worker 始終在 Job 被重試之前終止。如果您的 `--timeout` 選項比您的 `retry_after` 設定值長，您的 Job 可能會被處理兩次。


<a name="pausing-and-resuming-queue-workers"></a>
### 暫停與恢復隊列 Worker

有時您可能需要暫時防止隊列 Worker 處理新 Job，而不需要完全停止 Worker。例如，您可能希望在系統維護期間暫停 Job 處理。Laravel 提供了 `queue:pause` 和 `queue:continue` Artisan 指令來暫停和恢復隊列 Worker。

要暫停特定的隊列，請提供隊列連線名稱和隊列名稱：

```shell
php artisan queue:pause database:default
```

在此範例中，`database` 是隊列連線名稱，而 `default` 是隊列名稱。一旦隊列暫停，處理該隊列 Job 的任何 Worker 將繼續完成其當前的 Job，但在隊列恢復之前不會領取任何新 Job。

要恢復處理已暫停隊列上的 Job，請使用 `queue:continue` 指令：

```shell
php artisan queue:continue database:default
```

恢復隊列後，Worker 將立即開始處理該隊列的新 Job。請注意，暫停隊列並不會停止 Worker 行程本身——它只是阻止 Worker 處理來自指定隊列的新 Job。


<a name="worker-restart-and-pause-signals"></a>
#### Worker 重啟與暫停訊號

預設情況下，隊列 Worker 會在每次 Job 迭代時輪詢快取驅動程式以獲取重啟和暫停訊號。雖然這種輪詢對於回應 `queue:restart` 和 `queue:pause` 指令至關重要，但它確實會帶來少量的效能開銷。

如果您需要優化效能且不需要這些中斷功能，您可以透過呼叫 `Queue` Facade 上的 `withoutInterruptionPolling` 方法來全域停用此輪詢。這通常應該在您的 `AppServiceProvider` 的 `boot` 方法中完成：

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
> 當中斷輪詢被停用時，Worker 將不會回應 `queue:restart` 或 `queue:pause` 指令（取決於停用了哪些功能）。

<a name="supervisor-configuration"></a>
## Supervisor 設定

在正式環境中，您需要一種方法來保持 `queue:work` 程序持續執行。`queue:work` 程序可能會因為各種原因停止執行，例如超出 Worker 超時時間或執行了 `queue:restart` 指令。

因此，您需要設定一個程序監控器，用來偵測您的 `queue:work` 程序何時結束並自動重啟它們。此外，程序監控器還可以讓您指定要同時執行多少個 `queue:work` 程序。Supervisor 是 Linux 環境中常用的程序監控器，我們將在接下來的文件中討論如何設定它。


<a name="installing-supervisor"></a>
#### 安裝 Supervisor

Supervisor 是 Linux 作業系統的程序監控器，如果您的 `queue:work` 程序失敗，它將自動重啟它們。若要在 Ubuntu 上安裝 Supervisor，您可以使用以下指令：

```shell
sudo apt-get install supervisor
```

> [!NOTE]
> 如果自行設定與管理 Supervisor 讓您感到不知所措，請考慮使用 [Laravel Cloud](https://cloud.laravel.com)，它提供了一個全代管平台來執行 Laravel 隊列 Worker。


<a name="configuring-supervisor"></a>
#### 設定 Supervisor

Supervisor 設定檔通常儲存在 `/etc/supervisor/conf.d` 目錄中。在此目錄內，您可以建立任意數量的設定檔，指示 Supervisor 如何監控您的程序。例如，讓我們建立一個 `laravel-worker.conf` 檔案來啟動並監控 `queue:work` 程序：

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

在此範例中，`numprocs` 指令將指示 Supervisor 執行八個 `queue:work` 程序並監控所有程序，如果它們失敗，則自動重啟。您應該更改設定中的 `command` 指令，以反映您所需的隊列連線和 Worker 選項。

> [!WARNING]
> 您應該確保 `stopwaitsecs` 的值大於執行時間最長的 Job 所消耗的秒數。否則，Supervisor 可能會在 Job 處理完成之前將其強制終止。


<a name="starting-supervisor"></a>
#### 啟動 Supervisor

設定檔建立完成後，您可以使用以下指令更新 Supervisor 設定並啟動程序：

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start "laravel-worker:*"
```

有關 Supervisor 的更多資訊，請參閱 [Supervisor 說明文件](http://supervisord.org/index.html)。

<a name="dealing-with-failed-jobs"></a>
## 處理失敗的 Job

有時您的隊列 Job 會失敗。別擔心，事情並不總是按計劃進行！Laravel 包含了一種便利的方式來[指定 Job 應嘗試的最大次數](#max-job-attempts-and-timeout)。在非同步 Job 超過此嘗試次數後，它將被插入到 `failed_jobs` 資料庫資料表中。[同步派遣 Job](/docs/{{version}}/queues#synchronous-dispatching) 若失敗則不會儲存在此資料表中，其例外狀況會立即由應用程式處理。

建立 `failed_jobs` 資料表的遷移通常已存在於新的 Laravel 應用程式中。然而，如果您的應用程式不包含此資料表的遷移，您可以使用 `make:queue-failed-table` 指令來建立遷移：

```shell
php artisan make:queue-failed-table

php artisan migrate
```

執行[隊列 Worker](#running-the-queue-worker) 處理程序時，您可以使用 `queue:work` 指令上的 `--tries` 切換參數來指定 Job 應嘗試的最大次數。如果您沒有為 `--tries` 選項指定值，Job 將僅嘗試一次，或嘗試次數與 Job 類別的 `$tries` 屬性所指定的次數相同：

```shell
php artisan queue:work redis --tries=3
```

使用 `--backoff` 選項，您可以指定 Laravel 在重試遇到例外狀況的 Job 之前應等待多少秒。預設情況下，Job 會立即被釋放回隊列中，以便再次嘗試：

```shell
php artisan queue:work redis --tries=3 --backoff=3
```

如果您想針對個別 Job 設定 Laravel 在重試遇到例外狀況的 Job 之前應等待多少秒，您可以透過在 Job 類別中定義 `backoff` 屬性來達成：

```php
/**
 * The number of seconds to wait before retrying the job.
 *
 * @var int
 */
public $backoff = 3;
```

如果您需要更複雜的邏輯來決定 Job 的退避 (Backoff) 時間，您可以在 Job 類別中定義 `backoff` 方法：

```php
/**
 * Calculate the number of seconds to wait before retrying the job.
 */
public function backoff(): int
{
    return 3;
}
```

您可以透過從 `backoff` 方法傳回一個退避值陣列來輕鬆設定「指數級」退避。在此範例中，第一次重試的重試延遲將為 1 秒，第二次為 5 秒，第三次為 10 秒，如果還有剩餘嘗試次數，則之後的每次重試均為 10 秒：

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
### 失敗 Job 的後續清理

當特定 Job 失敗時，您可能想向使用者發送警示，或還原由該 Job 部分完成的任何動作。為了達成此目的，您可以在 Job 類別中定義一個 `failed` 方法。導致 Job 失敗的 `Throwable` 實例將被傳遞給 `failed` 方法：

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
> 在呼叫 `failed` 方法之前會實例化一個新的 Job 實例；因此，在 `handle` 方法中可能發生的任何類別屬性修改都將丟失。

失敗的 Job 不一定是指遇到未處理例外狀況的 Job。當 Job 用盡所有允許的嘗試次數時，也可以被視為失敗。這些嘗試可以透過幾種方式消耗：

<div class="content-list" markdown="1">

- Job 超時。
- Job 在執行期間遇到未處理的例外狀況。
- Job 被手動或由中介層釋放回隊列。

</div>

如果最後一次嘗試是因為 Job 執行期間拋出的例外狀況而失敗，該例外狀況將被傳遞給 Job 的 `failed` 方法。然而，如果 Job 是因為達到最大允許嘗試次數而失敗，`$exception` 將是 `Illuminate\Queue\MaxAttemptsExceededException` 的實例。同樣地，如果 Job 是因為超過設定的超時時間而失敗，`$exception` 將是 `Illuminate\Queue\TimeoutExceededException` 的實例。


<a name="retrying-failed-jobs"></a>
### 重試失敗的 Job

要查看已插入到 `failed_jobs` 資料庫資料表中的所有失敗 Job，您可以使用 `queue:failed` Artisan 指令：

```shell
php artisan queue:failed
```

`queue:failed` 指令將列出 Job ID、連線、隊列、失敗時間以及關於該 Job 的其他資訊。Job ID 可用於重試失敗的 Job。例如，要重試一個 ID 為 `ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece` 的失敗 Job，請執行以下指令：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece
```

如有必要，您可以向指令傳遞多個 ID：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece 91401d2c-0784-4f43-824c-34f94a33c24d
```

您也可以重試特定隊列的所有失敗 Job：

```shell
php artisan queue:retry --queue=name
```

要重試所有失敗的 Job，執行 `queue:retry` 指令並將 `all` 作為 ID 傳遞：

```shell
php artisan queue:retry all
```

如果您想刪除失敗的 Job，可以使用 `queue:forget` 指令：

```shell
php artisan queue:forget 91401d2c-0784-4f43-824c-34f94a33c24d
```

> [!NOTE]
> 當使用 [Horizon](/docs/{{version}}/horizon) 時，您應該使用 `horizon:forget` 指令來刪除失敗的 Job，而不是 `queue:forget` 指令。

要從 `failed_jobs` 資料表中刪除所有失敗的 Job，可以使用 `queue:flush` 指令：

```shell
php artisan queue:flush
```

`queue:flush` 指令會從您的隊列中移除所有失敗的 Job 紀錄，無論失敗的 Job 有多舊。您可以使用 `--hours` 選項僅刪除特定小時數之前或更早失敗的 Job：

```shell
php artisan queue:flush --hours=48
```


<a name="ignoring-missing-models"></a>
### 忽略缺失的模型

當將 Eloquent 模型注入 Job 時，該模型在放入隊列之前會自動被序列化，並在處理 Job 時從資料庫重新檢索。然而，如果在 Job 等待 Worker 處理期間模型已被刪除，您的 Job 可能會失敗並拋出 `ModelNotFoundException`。

為了方便起見，您可以透過將 Job 的 `deleteWhenMissingModels` 屬性設置為 `true` 來選擇自動刪除缺失模型的 Job。當此屬性設置為 `true` 時，Laravel 將悄悄地丟棄該 Job 而不引發例外狀況：

```php
/**
 * Delete the job if its models no longer exist.
 *
 * @var bool
 */
public $deleteWhenMissingModels = true;
```


<a name="pruning-failed-jobs"></a>
### 修剪失敗的 Job

您可以透過呼叫 `queue:prune-failed` Artisan 指令來修剪應用程式 `failed_jobs` 資料表中的紀錄：

```shell
php artisan queue:prune-failed
```

預設情況下，所有超過 24 小時的失敗 Job 紀錄都將被修剪。如果您向指令提供 `--hours` 選項，則僅保留過去 N 小時內插入的失敗 Job 紀錄。例如，以下指令將刪除所有在 48 小時前插入的失敗 Job 紀錄：

```shell
php artisan queue:prune-failed --hours=48
```

<a name="storing-failed-jobs-in-dynamodb"></a>
### 在 DynamoDB 中儲存失敗的 Job

Laravel 也支援將失敗的 Job 紀錄儲存在 [DynamoDB](https://aws.amazon.com/dynamodb) 中，而非關聯式資料庫資料表。然而，您必須手動建立一個 DynamoDB 資料表來儲存所有的失敗 Job 紀錄。通常，此資料表應命名為 `failed_jobs`，但您應該根據應用程式 `queue` 設定檔中的 `queue.failed.table` 設定值來命名。

`failed_jobs` 資料表應具有一個名為 `application` 的字串主要分割鍵 (Partition Key)，以及一個名為 `uuid` 的字串主要排序鍵 (Sort Key)。索引鍵中的 `application` 部分將包含您的應用程式名稱，該名稱定義於應用程式 `app` 設定檔中的 `name` 設定值。由於應用程式名稱是 DynamoDB 資料表索引鍵的一部分，因此您可以使用同一個資料表來儲存多個 Laravel 應用程式的失敗 Job。

此外，請確保安裝 AWS SDK，以便您的 Laravel 應用程式可以與 Amazon DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

接著，將 `queue.failed.driver` 設定選項的值設為 `dynamodb`。此外，您應該在失敗 Job 的設定陣列中定義 `key`、`secret` 與 `region` 設定選項。這些選項將用於與 AWS 進行驗證。使用 `dynamodb` 驅動程式時，不需要 `queue.failed.database` 設定選項：

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

您可以透過將 `queue.failed.driver` 設定選項的值設為 `null`，來指示 Laravel 捨棄失敗的 Job 而不進行儲存。通常，這可以透過 `QUEUE_FAILED_DRIVER` 環境變數來達成：

```ini
QUEUE_FAILED_DRIVER=null
```


<a name="failed-job-events"></a>
### 失敗 Job 事件

如果您想要註冊一個在 Job 失敗時叫用的事件監聽器，可以使用 `Queue` Facade 的 `failing` 方法。例如，我們可以從 Laravel 內建的 `AppServiceProvider` 的 `boot` 方法中為此事件附加一個閉包：

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
## 清除隊列中的 Job

> [!NOTE]
> 使用 [Horizon](/docs/{{version}}/horizon) 時，您應該使用 `horizon:clear` 指令來清除隊列中的 Job，而非 `queue:clear` 指令。

如果您想從預設連線的預設隊列中刪除所有 Job，可以使用 `queue:clear` Artisan 指令：

```shell
php artisan queue:clear
```

您也可以提供 `connection` 參數與 `queue` 選項來刪除特定連線與隊列中的 Job：

```shell
php artisan queue:clear redis --queue=emails
```

> [!WARNING]
> 清除隊列中的 Job 僅適用於 SQS、Redis 與 database 隊列驅動程式。此外，SQS 訊息刪除程序最多需要 60 秒，因此在您清除隊列後 60 秒內發送到 SQS 隊列的 Job 也可能會被刪除。


<a name="monitoring-your-queues"></a>
## 監控您的隊列

如果您的隊列突然湧入大量 Job，它可能會變得不堪重負，導致 Job 完成的等待時間變長。如果您願意，Laravel 可以在您的隊列 Job 數量超過指定門檻時向您發出警示。

要開始使用，您應該排程 `queue:monitor` 指令[每分鐘執行一次](/docs/{{version}}/scheduling)。此指令接受您希望監控的隊列名稱以及您期望的 Job 數量門檻：

```shell
php artisan queue:monitor redis:default,redis:deployments --max=100
```

僅排程此指令不足以觸發通知來提醒您隊列處於不堪重負的狀態。當指令遇到 Job 數量超過您門檻的隊列時，將會派遣一個 `Illuminate\Queue\Events\QueueBusy` 事件。您可以在應用程式的 `AppServiceProvider` 中監聽此事件，以便向您或您的開發團隊發送通知：

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

在測試派遣 Job 的程式碼時，您可能希望指示 Laravel 不要實際執行 Job 本身，因為 Job 的程式碼可以與派遣它的程式碼分開直接進行測試。當然，若要測試 Job 本身，您可以實例化一個 Job 實例並在測試中直接呼叫 `handle` 方法。

您可以使用 `Queue` Facade 的 `fake` 方法來防止隊列 Job 被實際推送到隊列中。在呼叫 `Queue` Facade 的 `fake` 方法後，您就可以斷言 (Assert) 應用程式是否有嘗試將 Job 推送到隊列：

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

您可以將閉包傳遞給 `assertPushed`、`assertNotPushed`、`assertClosurePushed` 或 `assertClosureNotPushed` 方法，以便斷言被推送的 Job 是否通過特定的「真值測試 (Truth Test)」。如果至少有一個被推送的 Job 通過了指定的真值測試，則斷言將會成功：

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

如果您只需要模擬特定的 Job，同時允許其他 Job 正常執行，您可以將應模擬的 Job 類別名稱傳遞給 `fake` 方法：

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

您可以使用 `except` 方法來模擬除了指定 Job 以外的所有 Job：

```php
Queue::fake()->except([
    ShipOrder::class,
]);
```


<a name="testing-job-chains"></a>
### 測試 Job 鏈結

若要測試 Job 鏈結，您需要利用 `Bus` Facade 的模擬功能。`Bus` Facade 的 `assertChained` 方法可用於斷言一個 [Job 鏈結](/docs/{{version}}/queues#job-chaining) 已被派遣。`assertChained` 方法接受一個鏈結 Job 陣列作為其第一個參數：

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

如上例所示，鏈結 Job 陣列可以是 Job 的類別名稱陣列。但是，您也可以提供實際的 Job 實例陣列。這樣做時，Laravel 將確保 Job 實例與應用程式派遣的鏈結 Job 具有相同的類別和相同的屬性值：

```php
Bus::assertChained([
    new ShipOrder,
    new RecordShipment,
    new UpdateInventory,
]);
```

您可以使用 `assertDispatchedWithoutChain` 方法來斷言一個 Job 在沒有鏈結的情況下被推送：

```php
Bus::assertDispatchedWithoutChain(ShipOrder::class);
```


<a name="testing-chain-modifications"></a>
#### 測試鏈結修改

如果鏈結中的 Job [將 Job 預置 (Prepend) 或附加 (Append) 到現有鏈結中](#adding-jobs-to-the-chain)，您可以使用 Job 的 `assertHasChain` 方法來斷言該 Job 具有預期的剩餘鏈結：

```php
$job = new ProcessPodcast;

$job->handle();

$job->assertHasChain([
    new TranscribePodcast,
    new OptimizePodcast,
    new ReleasePodcast,
]);
```

`assertDoesntHaveChain` 方法可用於斷言 Job 的剩餘鏈結為空：

```php
$job->assertDoesntHaveChain();
```


<a name="testing-chained-batches"></a>
#### 測試鏈結中的批次

如果您的 Job 鏈結 [包含一個 Job 批次](#chains-and-batches)，您可以透過在鏈結斷言中插入 `Bus::chainedBatch` 定義，來斷言該鏈結批次是否符合您的預期：

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

`Bus` Facade 的 `assertBatched` 方法可用於斷言一個 [Job 批次](/docs/{{version}}/queues#job-batching) 已被派遣。傳遞給 `assertBatched` 方法的閉包會接收一個 `Illuminate\Bus\PendingBatch` 實例，可用於檢查批次中的 Job：

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

可在 Pending Batch 上使用 `hasJobs` 方法來驗證批次是否包含預期的 Job。此方法接受 Job 實例、類別名稱或閉包組成的陣列：

```php
Bus::assertBatched(function (PendingBatch $batch) {
    return $batch->hasJobs([
        new ProcessCsvRow(row: 1),
        new ProcessCsvRow(row: 2),
        new ProcessCsvRow(row: 3),
    ]);
});
```

使用閉包時，閉包將接收 Job 實例。預期的 Job 類型將從閉包的型別提示 (Type Hint) 中推斷出來：

```php
Bus::assertBatched(function (PendingBatch $batch) {
    return $batch->hasJobs([
        fn (ProcessCsvRow $job) => $job->row === 1,
        fn (ProcessCsvRow $job) => $job->row === 2,
        fn (ProcessCsvRow $job) => $job->row === 3,
    ]);
});
```

您可以使用 `assertBatchCount` 方法來斷言已派遣了給定數量的批次：

```php
Bus::assertBatchCount(3);
```

您可以使用 `assertNothingBatched` 來斷言沒有派遣任何批次：

```php
Bus::assertNothingBatched();
```

<a name="testing-job-batch-interaction"></a>
#### 測試 Job / 批次互動

此外，有時您可能需要測試個別 Job 與其底層批次的互動。例如，您可能需要測試 Job 是否取消了其批次的後續處理。為了實現這一點，您需要透過 `withFakeBatch` 方法為 Job 分配一個模擬批次。`withFakeBatch` 方法會回傳一個包含 Job 實例和模擬批次的元組 (Tuple)：

```php
[$job, $batch] = (new ShipOrder)->withFakeBatch();

$job->handle();

$this->assertTrue($batch->cancelled());
$this->assertEmpty($batch->added);
```

<a name="testing-job-queue-interactions"></a>
### 測試 Job / 隊列互動

有時，您可能需要測試排隊的 Job 是否 [將自身釋放回隊列](#manually-releasing-a-job)。或者，您可能需要測試該 Job 是否刪除了自身。您可以透過實例化 Job 並呼叫 `withFakeQueueInteractions` 方法來測試這些隊列互動。

一旦模擬了 Job 的隊列互動，您就可以在 Job 上呼叫 `handle` 方法。在呼叫 Job 之後，可以使用各種斷言方法來驗證 Job 的隊列互動：

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

透過 `Queue` [Facade](/docs/{{version}}/facades) 上的 `before` 與 `after` 方法，您可以指定在處理隊列化的 Job 之前或之後執行的回呼 (Callback)。這些回呼是執行額外日誌記錄或為儀表板增加統計數據的絕佳機會。通常，您應該在 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫這些方法。例如，我們可以使用 Laravel 內建的 `AppServiceProvider`：

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

透過 `Queue` [Facade](/docs/{{version}}/facades) 上的 `looping` 方法，您可以指定在 Worker 嘗試從隊列中取得 Job 之前執行的回呼。例如，您可以註冊一個閉包，以回滾任何因先前失敗的 Job 而未關閉的交易：

```php
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Queue;

Queue::looping(function () {
    while (DB::transactionLevel() > 0) {
        DB::rollBack();
    }
});
```