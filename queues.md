# 佇列

- [簡介](#introduction)
    - [連線與佇列的區別](#connections-vs-queues)
    - [驅動程式注意事項與先決條件](#driver-prerequisites)
- [建立 Job](#creating-jobs)
    - [產生 Job Class](#generating-job-classes)
    - [Class 結構](#class-structure)
    - [唯一 Job](#unique-jobs)
    - [加密 Job](#encrypted-jobs)
- [Job Middleware](#job-middleware)
    - [速率限制](#rate-limiting)
    - [避免 Job 重疊](#preventing-job-overlaps)
    - [節流例外](#throttling-exceptions)
    - [跳過 Job](#skipping-jobs)
- [分派 Job](#dispatching-jobs)
    - [延遲分派](#delayed-dispatching)
    - [同步分派](#synchronous-dispatching)
    - [Job 與資料庫交易](#jobs-and-database-transactions)
    - [Job 鏈](#job-chaining)
    - [自訂佇列與連線](#customizing-the-queue-and-connection)
    - [指定 Job 最大嘗試次數 / 逾時值](#max-job-attempts-and-timeout)
    - [錯誤處理](#error-handling)
- [Job 批次處理](#job-batching)
    - [定義可批次處理的 Job](#defining-batchable-jobs)
    - [分派批次](#dispatching-batches)
    - [鏈與批次](#chains-and-batches)
    - [將 Job 加入批次](#adding-jobs-to-batches)
    - [檢查批次](#inspecting-batches)
    - [取消批次](#cancelling-batches)
    - [批次失敗](#batch-failures)
    - [修剪批次](#pruning-batches)
    - [將批次儲存至 DynamoDB](#storing-batches-in-dynamodb)
- [佇列 Closure](#queueing-closures)
- [執行佇列 Worker](#running-the-queue-worker)
    - [`queue:work` 命令](#the-queue-work-command)
    - [佇列優先順序](#queue-priorities)
    - [佇列 Worker 與部署](#queue-workers-and-deployment)
    - [Job 過期與逾時](#job-expirations-and-timeouts)
- [Supervisor 設定](#supervisor-configuration)
- [處理失敗的 Job](#dealing-with-failed-jobs)
    - [失敗 Job 後的清理](#cleaning-up-after-failed-jobs)
    - [重試失敗的 Job](#retrying-failed-jobs)
    - [忽略遺失的 Model](#ignoring-missing-models)
    - [修剪失敗的 Job](#pruning-failed-jobs)
    - [將失敗的 Job 儲存至 DynamoDB](#storing-failed-jobs-in-dynamodb)
    - [停用失敗 Job 儲存](#disabling-failed-job-storage)
    - [失敗 Job 事件](#failed-job-events)
- [從佇列中清除 Job](#clearing-jobs-from-queues)
- [監控佇列](#monitoring-your-queues)
- [測試](#testing)
    - [模擬部分 Job](#faking-a-subset-of-jobs)
    - [測試 Job 鏈](#testing-job-chains)
    - [測試 Job 批次](#testing-job-batches)
    - [測試 Job / 佇列互動](#testing-job-queue-interactions)
- [Job 事件](#job-events)

<a name="introduction"></a>
## 簡介

在建構 Web 應用程式時，您可能會遇到一些任務，例如解析並儲存上傳的 CSV 檔案，這些任務在一般的 Web 請求期間執行時間過長。幸運的是，Laravel 讓您可以輕鬆建立可在背景處理的佇列 Job。透過將耗時的任務移至佇列，您的應用程式可以極快的速度回應 Web 請求，並為您的客戶提供更好的使用者體驗。

Laravel 佇列為各種不同的佇列後端提供統一的佇列 API，例如 [Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io)，甚至是關聯式資料庫。

Laravel 的佇列設定選項儲存在應用程式的 `config/queue.php` 設定檔中。在此檔案中，您會找到框架中包含的每個佇列驅動程式的連線設定，包括 database、[Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 和 [Beanstalkd](https://beanstalkd.github.io/) 驅動程式，以及一個將立即執行 Job 的同步驅動程式 (用於本地開發期間)。還包含一個 `null` 佇列驅動程式，它會丟棄佇列中的 Job。

> [!NOTE]
> Laravel 現在提供 Horizon，一個用於 Redis 驅動佇列的精美儀表板和設定系統。請查看完整的 [Horizon 說明文件](/docs/{{version}}/horizon) 以獲取更多資訊。

<a name="connections-vs-queues"></a>
### 連線與佇列的區別

在開始使用 Laravel 佇列之前，了解「連線 (connections)」和「佇列 (queues)」之間的區別很重要。在您的 `config/queue.php` 設定檔中，有一個 `connections` 設定陣列。此選項定義了與後端佇列服務 (例如 Amazon SQS、Beanstalk 或 Redis) 的連線。然而，任何給定的佇列連線都可能有多個「佇列」，這些佇列可以被視為不同的堆疊或堆積的佇列 Job。

請注意，`queue` 設定檔中的每個連線設定範例都包含一個 `queue` 屬性。這是 Job 在分派到給定連線時將被分派到的預設佇列。換句話說，如果您分派一個 Job 而沒有明確定義它應該分派到哪個佇列，該 Job 將被放置在連線設定的 `queue` 屬性中定義的佇列上：

    use App\Jobs\ProcessPodcast;

    // This job is sent to the default connection's default queue...
    ProcessPodcast::dispatch();

    // This job is sent to the default connection's "emails" queue...
    ProcessPodcast::dispatch()->onQueue('emails');

有些應用程式可能不需要將 Job 推送到多個佇列，而是傾向於只有一個簡單的佇列。然而，將 Job 推送到多個佇列對於希望優先處理或區分 Job 處理方式的應用程式來說特別有用，因為 Laravel 佇列 Worker 允許您根據優先順序指定它應該處理哪些佇列。例如，如果您將 Job 推送到 `high` 佇列，您可以執行一個 Worker 來賦予它們更高的處理優先順序：

```shell
php artisan queue:work --queue=high,default
```

<a name="driver-prerequisites"></a>
### 驅動程式注意事項與先決條件

<a name="database"></a>
#### Database

為了使用 `database` 佇列驅動程式，您將需要一個資料庫表格來儲存 Job。通常，這包含在 Laravel 的預設 `0001_01_01_000002_create_jobs_table.php` [資料庫遷移](/docs/{{version}}/migrations) 中；但是，如果您的應用程式不包含此遷移，您可以使用 `make:queue-table` Artisan 命令來建立它：

```shell
php artisan make:queue-table

php artisan migrate
```

<a name="redis"></a>
#### Redis

為了使用 `redis` 佇列驅動程式，您應該在 `config/database.php` 設定檔中設定 Redis 資料庫連線。

> [!WARNING]
> `redis` 佇列驅動程式不支援 `serializer` 和 `compression` Redis 選項。

**Redis Cluster**

如果您的 Redis 佇列連線使用 Redis Cluster，您的佇列名稱必須包含一個 [key hash tag](https://redis.io/docs/reference/cluster-spec/#hash-tags)。這是為了確保給定佇列的所有 Redis key 都放置在相同的 hash slot 中：

    'redis' => [
        'driver' => 'redis',
        'connection' => env('REDIS_QUEUE_CONNECTION', 'default'),
        'queue' => env('REDIS_QUEUE', '{default}'),
        'retry_after' => env('REDIS_QUEUE_RETRY_AFTER', 90),
        'block_for' => null,
        'after_commit' => false,
    ],

**Blocking**

使用 Redis 佇列時，您可以使用 `block_for` 設定選項來指定驅動程式應該等待 Job 可用多長時間，然後再迭代 Worker 迴圈並重新輪詢 Redis 資料庫。

根據您的佇列負載調整此值可以比持續輪詢 Redis 資料庫以獲取新 Job 更有效率。例如，您可以將值設定為 `5`，表示驅動程式應該阻塞五秒鐘，等待 Job 可用：

    'redis' => [
        'driver' => 'redis',
        'connection' => env('REDIS_QUEUE_CONNECTION', 'default'),
        'queue' => env('REDIS_QUEUE', 'default'),
        'retry_after' => env('REDIS_QUEUE_RETRY_AFTER', 90),
        'block_for' => 5,
        'after_commit' => false,
    ],

> [!WARNING]
> 將 `block_for` 設定為 `0` 將導致佇列 Worker 無限期阻塞，直到 Job 可用。這也將阻止處理 `SIGTERM` 等訊號，直到下一個 Job 被處理。

<a name="other-driver-prerequisites"></a>
#### 其他驅動程式先決條件

以下是列出的佇列驅動程式所需的依賴項。這些依賴項可以透過 Composer 套件管理器安裝：

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

預設情況下，應用程式的所有可佇列 Job 都儲存在 `app/Jobs` 目錄中。如果 `app/Jobs` 目錄不存在，它將在您執行 `make:job` Artisan 命令時建立：

```shell
php artisan make:job ProcessPodcast
```

產生的 Class 將實作 `Illuminate\Contracts\Queue\ShouldQueue` 介面，向 Laravel 指示該 Job 應該被推送到佇列以非同步執行。

> [!NOTE]
> Job stub 可以使用 [stub publishing](/docs/{{version}}/artisan#stub-customization) 進行自訂。

<a name="class-structure"></a>
### Class 結構

Job Class 非常簡單，通常只包含一個 `handle` 方法，該方法在 Job 被佇列處理時被呼叫。首先，讓我們看看一個 Job Class 範例。在此範例中，我們將假裝我們管理一個 Podcast 發布服務，並且需要在發布之前處理上傳的 Podcast 檔案：

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

在此範例中，請注意我們能夠將 [Eloquent model](/docs/{{version}}/eloquent) 直接傳遞到佇列 Job 的建構函式中。由於 Job 使用的 `Queueable` trait，Eloquent model 及其載入的關聯將在 Job 處理時優雅地序列化和反序列化。

如果您的佇列 Job 在其建構函式中接受一個 Eloquent model，則只有該 model 的識別碼會被序列化到佇列中。當 Job 實際被處理時，佇列系統將自動從資料庫中重新擷取完整的 model 實例及其載入的關聯。這種 model 序列化方法允許將更小的 Job payload 發送到您的佇列驅動程式。

<a name="handle-method-dependency-injection"></a>
#### `handle` 方法依賴注入

`handle` 方法在 Job 被佇列處理時被呼叫。請注意，我們可以在 Job 的 `handle` 方法上型別提示依賴項。Laravel [服務容器](/docs/{{version}}/container) 會自動注入這些依賴項。

如果您想完全控制容器如何將依賴項注入 `handle` 方法，您可以使用容器的 `bindMethod` 方法。`bindMethod` 方法接受一個回呼，該回呼接收 Job 和容器。在回呼中，您可以隨意呼叫 `handle` 方法。通常，您應該從 `App\Providers\AppServiceProvider` [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫此方法：

    use App\Jobs\ProcessPodcast;
    use App\Services\AudioProcessor;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->bindMethod([ProcessPodcast::class, 'handle'], function (ProcessPodcast $job, Application $app) {
        return $job->handle($app->make(AudioProcessor::class));
    });

> [!WARNING]
> 二進位資料，例如原始圖片內容，在傳遞給佇列 Job 之前應透過 `base64_encode` 函數進行處理。否則，Job 在放入佇列時可能無法正確序列化為 JSON。

<a name="handling-relationships"></a>
#### 佇列關聯

由於所有載入的 Eloquent model 關聯在 Job 佇列時也會被序列化，因此序列化的 Job 字串有時會變得相當大。此外，當 Job 被反序列化並從資料庫中重新擷取 model 關聯時，它們將被完整擷取。在 Job 佇列處理期間，在 model 序列化之前應用的任何先前關聯約束將在 Job 反序列化時不被應用。因此，如果您希望處理給定關聯的子集，您應該在佇列 Job 中重新約束該關聯。

或者，為了防止關聯被序列化，您可以在設定屬性值時呼叫 model 上的 `withoutRelations` 方法。此方法將返回一個沒有載入關聯的 model 實例：

    /**
     * Create a new job instance.
     */
    public function __construct(
        Podcast $podcast,
    ) {
        $this->podcast = $podcast->withoutRelations();
    }

如果您正在使用 PHP 建構函式屬性提升，並希望指示 Eloquent model 不應序列化其關聯，您可以使用 `WithoutRelations` 屬性：

    use Illuminate\Queue\Attributes\WithoutRelations;

    /**
     * Create a new job instance.
     */
    public function __construct(
        #[WithoutRelations]
        public Podcast $podcast,
    ) {}

如果一個 Job 接收一個 Eloquent model 的集合或陣列，而不是單個 model，則該集合中的 model 在 Job 反序列化和執行時將不會恢復其關聯。這是為了防止處理大量 model 的 Job 產生過多的資源使用。

<a name="unique-jobs"></a>
### 唯一 Job

> [!WARNING]
> 唯一 Job 需要支援 [鎖定](/docs/{{version}}/cache#atomic-locks) 的快取驅動程式。目前，`memcached`、`redis`、`dynamodb`、`database`、`file` 和 `array` 快取驅動程式支援原子鎖定。此外，唯一 Job 限制不適用於批次中的 Job。

有時，您可能希望確保在任何時間點，佇列中只有一個特定 Job 的實例。您可以透過在 Job Class 上實作 `ShouldBeUnique` 介面來實現此目的。此介面不需要您在 Class 上定義任何額外的方法：

    <?php

    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Contracts\Queue\ShouldBeUnique;

    class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
    {
        ...
    }

在上面的範例中，`UpdateSearchIndex` Job 是唯一的。因此，如果另一個 Job 實例已在佇列中且尚未完成處理，則該 Job 將不會被分派。

在某些情況下，您可能希望定義一個特定的「key」來使 Job 唯一，或者您可能希望指定一個逾時時間，超過該時間後 Job 不再保持唯一。為此，您可以在 Job Class 上定義 `uniqueId` 和 `uniqueFor` 屬性或方法：

    <?php

    use App\Models\Product;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Contracts\Queue\ShouldBeUnique;

    class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
    {
        /**
         * The product instance.
         *
         * @var \App\Product
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

在上面的範例中，`UpdateSearchIndex` Job 透過產品 ID 保持唯一。因此，任何具有相同產品 ID 的新 Job 分派都將被忽略，直到現有 Job 完成處理。此外，如果現有 Job 在一小時內未處理，則唯一鎖將被釋放，並且可以將另一個具有相同唯一 key 的 Job 分派到佇列。

> [!WARNING]
> 如果您的應用程式從多個 Web 伺服器或容器分派 Job，您應該確保所有伺服器都與相同的中央快取伺服器通訊，以便 Laravel 可以準確判斷 Job 是否唯一。

<a name="keeping-jobs-unique-until-processing-begins"></a>
#### 在處理開始前保持 Job 唯一

預設情況下，唯一 Job 在 Job 完成處理或所有重試嘗試失敗後會「解鎖」。但是，在某些情況下，您可能希望 Job 在處理之前立即解鎖。為此，您的 Job 應該實作 `ShouldBeUniqueUntilProcessing` 契約，而不是 `ShouldBeUnique` 契約：

    <?php

    use App\Models\Product;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Contracts\Queue\ShouldBeUniqueUntilProcessing;

    class UpdateSearchIndex implements ShouldQueue, ShouldBeUniqueUntilProcessing
    {
        // ...
    }

<a name="unique-job-locks"></a>
#### 唯一 Job 鎖定

在幕後，當一個 `ShouldBeUnique` Job 被分派時，Laravel 會嘗試獲取一個帶有 `uniqueId` key 的 [鎖定](/docs/{{version}}/cache#atomic-locks)。如果未獲取鎖定，則 Job 不會被分派。此鎖定在 Job 完成處理或所有重試嘗試失敗時釋放。預設情況下，Laravel 將使用預設快取驅動程式來獲取此鎖定。但是，如果您希望使用另一個驅動程式來獲取鎖定，您可以定義一個 `uniqueVia` 方法，該方法返回應該使用的快取驅動程式：

    use Illuminate\Contracts\Cache\Repository;
    use Illuminate\Support\Facades\Cache;

    class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
    {
        ...

        /**
         * Get the cache driver for the unique job lock.
         */
        public function uniqueVia(): Repository
        {
            return Cache::driver('redis');
        }
    }

> [!NOTE]
> 如果您只需要限制 Job 的並行處理，請改用 [`WithoutOverlapping`](/docs/{{version}}/queues#preventing-job-overlaps) Job middleware。

<a name="encrypted-jobs"></a>
### 加密 Job

Laravel 允許您透過 [加密](/docs/{{version}}/encryption) 來確保 Job 資料的隱私和完整性。首先，只需將 `ShouldBeEncrypted` 介面新增到 Job Class。一旦此介面已新增到 Class，Laravel 將在將其推送到佇列之前自動加密您的 Job：

    <?php

    use Illuminate\Contracts\Queue\ShouldBeEncrypted;
    use Illuminate\Contracts\Queue\ShouldQueue;

    class UpdateSearchIndex implements ShouldQueue, ShouldBeEncrypted
    {
        // ...
    }

<a name="job-middleware"></a>
## Job Middleware

Job middleware 允許您在佇列 Job 的執行周圍包裝自訂邏輯，從而減少 Job 本身中的樣板程式碼。例如，考慮以下 `handle` 方法，它利用 Laravel 的 Redis 速率限制功能，允許每五秒只處理一個 Job：

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

雖然這段程式碼是有效的，但 `handle` 方法的實作變得嘈雜，因為它被 Redis 速率限制邏輯所混淆。此外，對於我們想要進行速率限制的任何其他 Job，此速率限制邏輯必須重複。

與其在 `handle` 方法中進行速率限制，我們可以定義一個處理速率限制的 Job middleware。Laravel 沒有 Job middleware 的預設位置，因此您可以將 Job middleware 放置在應用程式中的任何位置。在此範例中，我們將 middleware 放置在 `app/Jobs/Middleware` 目錄中：

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

如您所見，與 [路由 middleware](/docs/{{version}}/middleware) 一樣，Job middleware 接收正在處理的 Job 和一個應該被呼叫以繼續處理 Job 的回呼。

建立 Job middleware 後，可以透過從 Job 的 `middleware` 方法返回它們來將它們附加到 Job。此方法不存在於由 `make:job` Artisan 命令 scaffold 的 Job 上，因此您需要手動將其新增到您的 Job Class：

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

> [!NOTE]
> Job middleware 也可以分配給可佇列的事件監聽器、mailable 和通知。

<a name="rate-limiting"></a>
### 速率限制

儘管我們剛剛示範了如何編寫自己的速率限制 Job middleware，但 Laravel 實際上包含了一個您可以利用的速率限制 middleware 來限制 Job 的速率。與 [路由速率限制器](/docs/{{version}}/routing#defining-rate-limiters) 一樣，Job 速率限制器是使用 `RateLimiter` Facade 的 `for` 方法定義的。

例如，您可能希望允許使用者每小時備份一次資料，同時對高級客戶不施加此類限制。為此，您可以在 `AppServiceProvider` 的 `boot` 方法中定義一個 `RateLimiter`：

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

在上面的範例中，我們定義了一個每小時的速率限制；但是，您可以使用 `perMinute` 方法輕鬆定義基於分鐘的速率限制。此外，您可以將任何您希望的值傳遞給速率限制的 `by` 方法；但是，此值最常用於按客戶區分速率限制：

    return Limit::perMinute(50)->by($job->user->id);

定義速率限制後，您可以使用 `Illuminate\Queue\Middleware\RateLimited` middleware 將速率限制器附加到您的 Job。每次 Job 超過速率限制時，此 middleware 都會根據速率限制持續時間，以適當的延遲將 Job 釋放回佇列。

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

將速率限制的 Job 釋放回佇列仍會增加 Job 的總 `attempts` 次數。您可能希望相應地調整 Job Class 上的 `tries` 和 `maxExceptions` 屬性。或者，您可能希望使用 [`retryUntil` 方法](#time-based-attempts) 來定義 Job 不應再嘗試的時間量。

如果您不希望 Job 在速率限制時重試，您可以使用 `dontRelease` 方法：

    /**
     * Get the middleware the job should pass through.
     *
     * @return array<int, object>
     */
    public function middleware(): array
    {
        return [(new RateLimited('backups'))->dontRelease()];
    }

> [!NOTE]
> 如果您正在使用 Redis，您可以使用 `Illuminate\Queue\Middleware\RateLimitedWithRedis` middleware，它針對 Redis 進行了優化，並且比基本速率限制 middleware 更有效率。

<a name="preventing-job-overlaps"></a>
### 避免 Job 重疊

Laravel 包含一個 `Illuminate\Queue\Middleware\WithoutOverlapping` middleware，它允許您根據任意 key 防止 Job 重疊。當佇列 Job 正在修改一個應該一次只由一個 Job 修改的資源時，這會很有幫助。

例如，讓我們想像您有一個佇列 Job，它更新使用者的信用評分，並且您希望防止相同使用者 ID 的信用評分更新 Job 重疊。為此，您可以從 Job 的 `middleware` 方法返回 `WithoutOverlapping` middleware：

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

任何相同型別的重疊 Job 都將被釋放回佇列。您還可以指定在釋放的 Job 再次嘗試之前必須經過的秒數：

    /**
     * Get the middleware the job should pass through.
     *
     * @return array<int, object>
     */
    public function middleware(): array
    {
        return [(new WithoutOverlapping($this->order->id))->releaseAfter(60)];
    }

如果您希望立即刪除任何重疊的 Job，以便它們不會被重試，您可以使用 `dontRelease` 方法：

    /**
     * Get the middleware the job should pass through.
     *
     * @return array<int, object>
     */
    public function middleware(): array
    {
        return [(new WithoutOverlapping($this->order->id))->dontRelease()];
    }

`WithoutOverlapping` middleware 由 Laravel 的原子鎖定功能提供支援。有時，您的 Job 可能會意外失敗或逾時，導致鎖定未釋放。因此，您可以透過 `expireAfter` 方法明確定義鎖定過期時間。例如，下面的範例將指示 Laravel 在 Job 開始處理後三分鐘釋放 `WithoutOverlapping` 鎖定：

    /**
     * Get the middleware the job should pass through.
     *
     * @return array<int, object>
     */
    public function middleware(): array
    {
        return [(new WithoutOverlapping($this->order->id))->expireAfter(180)];
    }

> [!WARNING]
> `WithoutOverlapping` middleware 需要支援 [鎖定](/docs/{{version}}/cache#atomic-locks) 的快取驅動程式。目前，`memcached`、`redis`、`dynamodb`、`database`、`file` 和 `array` 快取驅動程式支援原子鎖定。

<a name="sharing-lock-keys-across-job-classes"></a>
#### 跨 Job Class 共用鎖定 Key

預設情況下，`WithoutOverlapping` middleware 只會阻止相同 Class 的 Job 重疊。因此，儘管兩個不同的 Job Class 可能使用相同的鎖定 key，但它們不會被阻止重疊。但是，您可以指示 Laravel 使用 `shared` 方法將 key 應用於跨 Job Class：

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
### 節流例外

Laravel 包含一個 `Illuminate\Queue\Middleware\ThrottlesExceptions` middleware，它允許您節流例外。一旦 Job 拋出給定數量的例外，所有進一步執行 Job 的嘗試都將延遲，直到指定的時間間隔過去。此 middleware 對於與不穩定的第三方服務互動的 Job 特別有用。

例如，讓我們想像一個佇列 Job 與開始拋出例外的第三方 API 互動。為了節流例外，您可以從 Job 的 `middleware` 方法返回 `ThrottlesExceptions` middleware。通常，此 middleware 應該與實作 [基於時間的嘗試](#time-based-attempts) 的 Job 配對：

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

middleware 接受的第一個建構函式參數是 Job 在被節流之前可以拋出的例外數量，而第二個建構函式參數是 Job 被節流後再次嘗試之前應該經過的秒數。在上面的程式碼範例中，如果 Job 連續拋出 10 個例外，我們將等待 5 分鐘，然後再嘗試 Job，受 30 分鐘的時間限制。

當 Job 拋出例外但尚未達到例外閾值時，Job 通常會立即重試。但是，您可以透過在將 middleware 附加到 Job 時呼叫 `backoff` 方法來指定此類 Job 應該延遲的分鐘數：

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

在內部，此 middleware 使用 Laravel 的快取系統來實作速率限制，並且 Job 的 Class 名稱被用作快取「key」。您可以透過在將 middleware 附加到 Job 時呼叫 `by` 方法來覆寫此 key。如果您有多個 Job 與相同的第三方服務互動，並且您希望它們共用一個共同的節流「bucket」，這會很有用：

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

預設情況下，此 middleware 將節流每個例外。您可以透過在將 middleware 附加到 Job 時呼叫 `when` 方法來修改此行為。然後，只有當提供給 `when` 方法的 closure 返回 `true` 時，例外才會被節流：

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

如果您希望將節流的例外報告給應用程式的例外處理程式，您可以透過在將 middleware 附加到 Job 時呼叫 `report` 方法來實現。或者，您可以向 `report` 方法提供一個 closure，並且只有當給定的 closure 返回 `true` 時，例外才會被報告：

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

> [!NOTE]
> 如果您正在使用 Redis，您可以使用 `Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis` middleware，它針對 Redis 進行了優化，並且比基本例外節流 middleware 更有效率。

<a name="skipping-jobs"></a>
### 跳過 Job

`Skip` middleware 允許您指定應該跳過/刪除 Job，而無需修改 Job 的邏輯。`Skip::when` 方法將在給定條件評估為 `true` 時刪除 Job，而 `Skip::unless` 方法將在條件評估為 `false` 時刪除 Job：

    use Illuminate\Queue\Middleware\Skip;

    /**
    * Get the middleware the job should pass through.
    */
    public function middleware(): array
    {
        return [
            Skip::when($someCondition),
        ];
    }

您還可以將 `Closure` 傳遞給 `when` 和 `unless` 方法，以進行更複雜的條件評估：

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

<a name="dispatching-jobs"></a>
## 分派 Job

一旦您編寫了 Job Class，您就可以使用 Job 本身上的 `dispatch` 方法來分派它。傳遞給 `dispatch` 方法的參數將提供給 Job 的建構函式：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
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

如果您想有條件地分派 Job，您可以使用 `dispatchIf` 和 `dispatchUnless` 方法：

    ProcessPodcast::dispatchIf($accountActive, $podcast);

    ProcessPodcast::dispatchUnless($accountSuspended, $podcast);

在新的 Laravel 應用程式中，`sync` 驅動程式是預設的佇列驅動程式。此驅動程式在當前請求的前景中同步執行 Job，這在本地開發期間通常很方便。如果您想實際開始將 Job 佇列以進行背景處理，您可以在應用程式的 `config/queue.php` 設定檔中指定不同的佇列驅動程式。

<a name="delayed-dispatching"></a>
### 延遲分派

如果您想指定 Job 不應立即供佇列 Worker 處理，您可以在分派 Job 時使用 `delay` 方法。例如，讓我們指定 Job 在分派後 10 分鐘內不應供處理：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
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

在某些情況下，Job 可能已設定預設延遲。如果您需要繞過此延遲並分派 Job 以立即處理，您可以使用 `withoutDelay` 方法：

    ProcessPodcast::dispatch($podcast)->withoutDelay();

> [!WARNING]
> Amazon SQS 佇列服務的最大延遲時間為 15 分鐘。

<a name="dispatching-after-the-response-is-sent-to-browser"></a>
#### 在回應傳送至瀏覽器後分派

或者，如果您的 Web 伺服器使用 FastCGI，`dispatchAfterResponse` 方法會延遲分派 Job，直到 HTTP 回應傳送給使用者瀏覽器之後。這仍然允許使用者開始使用應用程式，即使佇列 Job 仍在執行。這通常只應用於大約一秒鐘的 Job，例如傳送電子郵件。由於它們在當前 HTTP 請求中處理，因此以這種方式分派的 Job 不需要佇列 Worker 運行即可處理：

    use App\Jobs\SendNotification;

    SendNotification::dispatchAfterResponse();

您還可以 `dispatch` 一個 closure 並將 `afterResponse` 方法鏈接到 `dispatch` 輔助函數上，以在 HTTP 回應傳送給瀏覽器後執行 closure：

    use App\Mail\WelcomeMessage;
    use Illuminate\Support\Facades\Mail;

    dispatch(function () {
        Mail::to('taylor @example.com')->send(new WelcomeMessage);
    })->afterResponse();

<a name="synchronous-dispatching"></a>
### 同步分派

如果您想立即 (同步) 分派 Job，您可以使用 `dispatchSync` 方法。使用此方法時，Job 將不會被佇列，並將在當前程序中立即執行：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
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

<a name="jobs-and-database-transactions"></a>
### Job 與資料庫交易

雖然在資料庫交易中分派 Job 完全沒問題，但您應該特別注意確保您的 Job 實際上能夠成功執行。在交易中分派 Job 時，Job 可能會在父交易提交之前被 Worker 處理。發生這種情況時，您在資料庫交易期間對 Model 或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何 Model 或資料庫記錄可能不存在於資料庫中。

幸運的是，Laravel 提供了幾種解決此問題的方法。首先，您可以在佇列連線的設定陣列中設定 `after_commit` 連線選項：

    'redis' => [
        'driver' => 'redis',
        // ...
        'after_commit' => true,
    ],

當 `after_commit` 選項為 `true` 時，您可以在資料庫交易中分派 Job；但是，Laravel 將等到開放的父資料庫交易已提交後才實際分派 Job。當然，如果目前沒有開放的資料庫交易，Job 將立即分派。

如果由於交易期間發生的例外而導致交易回滾，則在該交易期間分派的 Job 將被丟棄。

> [!NOTE]
> 將 `after_commit` 設定選項設定為 `true` 也會導致所有開放的資料庫交易提交後，任何佇列的事件監聽器、mailable、通知和廣播事件都會被分派。

<a name="specifying-commit-dispatch-behavior-inline"></a>
#### 行內指定提交分派行為

如果您沒有將 `after_commit` 佇列連線設定選項設定為 `true`，您仍然可以指示特定 Job 應在所有開放的資料庫交易提交後分派。為此，您可以將 `afterCommit` 方法鏈接到您的分派操作上：

    use App\Jobs\ProcessPodcast;

    ProcessPodcast::dispatch($podcast)->afterCommit();

同樣，如果 `after_commit` 設定選項設定為 `true`，您可以指示特定 Job 應立即分派，而無需等待任何開放的資料庫交易提交：

    ProcessPodcast::dispatch($podcast)->beforeCommit();

<a name="job-chaining"></a>
### Job 鏈

Job 鏈允許您指定一系列佇列 Job，這些 Job 應在主要 Job 成功執行後按順序運行。如果序列中的一個 Job 失敗，其餘 Job 將不會運行。要執行佇列 Job 鏈，您可以使用 `Bus` Facade 提供的 `chain` 方法。Laravel 的命令 Bus 是佇列 Job 分派的底層組件：

    use App\Jobs\OptimizePodcast;
    use App\Jobs\ProcessPodcast;
    use App\Jobs\ReleasePodcast;
    use Illuminate\Support\Facades\Bus;

    Bus::chain([
        new ProcessPodcast,
        new OptimizePodcast,
        new ReleasePodcast,
    ])->dispatch();

除了鏈接 Job Class 實例之外，您還可以鏈接 closure：

    Bus::chain([
        new ProcessPodcast,
        new OptimizePodcast,
        function () {
            Podcast::update(/* ... */);
        },
    ])->dispatch();

> [!WARNING]
> 在 Job 中使用 `$this->delete()` 方法刪除 Job 不會阻止鏈接的 Job 被處理。鏈接只會在鏈接中的 Job 失敗時停止執行。

<a name="chain-connection-queue"></a>
#### 鏈接連線與佇列

如果您想指定鏈接 Job 應該使用的連線和佇列，您可以使用 `onConnection` 和 `onQueue` 方法。這些方法指定了應該使用的佇列連線和佇列名稱，除非佇列 Job 被明確分配了不同的連線/佇列：

    Bus::chain([
        new ProcessPodcast,
        new OptimizePodcast,
        new ReleasePodcast,
    ])->onConnection('redis')->onQueue('podcasts')->dispatch();

<a name="adding-jobs-to-the-chain"></a>
#### 將 Job 加入鏈中

有時，您可能需要從鏈中的另一個 Job 內部，在現有 Job 鏈的前面或後面新增 Job。您可以使用 `prependToChain` 和 `appendToChain` 方法來實現此目的：

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

鏈接 Job 時，您可以使用 `catch` 方法來指定一個 closure，該 closure 應在鏈接中的 Job 失敗時被呼叫。給定的回呼將接收導致 Job 失敗的 `Throwable` 實例：

    use Illuminate\Support\Facades\Bus;
    use Throwable;

    Bus::chain([
        new ProcessPodcast,
        new OptimizePodcast,
        new ReleasePodcast,
    ])->catch(function (Throwable $e) {
        // A job within the chain has failed...
    })->dispatch();

> [!WARNING]
> 由於鏈接回呼由 Laravel 佇列序列化並在稍後執行，因此您不應在鏈接回呼中使用 `$this` 變數。

<a name="customizing-the-queue-and-connection"></a>
### 自訂佇列與連線

<a name="dispatching-to-a-particular-queue"></a>
#### 分派到特定佇列

透過將 Job 推送到不同的佇列，您可以「分類」您的佇列 Job，甚至可以優先處理您分配給各種佇列的 Worker 數量。請記住，這不會將 Job 推送到佇列設定檔中定義的不同佇列「連線」，而只會推送到單一連線中的特定佇列。要指定佇列，請在分派 Job 時使用 `onQueue` 方法：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
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

或者，您可以透過在 Job 的建構函式中呼叫 `onQueue` 方法來指定 Job 的佇列：

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

<a name="dispatching-to-a-particular-connection"></a>
#### 分派到特定連線

如果您的應用程式與多個佇列連線互動，您可以使用 `onConnection` 方法指定要將 Job 推送到哪個連線：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
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

您可以將 `onConnection` 和 `onQueue` 方法鏈接在一起，以指定 Job 的連線和佇列：

    ProcessPodcast::dispatch($podcast)
        ->onConnection('sqs')
        ->onQueue('processing');

或者，您可以透過在 Job 的建構函式中呼叫 `onConnection` 方法來指定 Job 的連線：

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

<a name="max-job-attempts-and-timeout"></a>
### 指定 Job 最大嘗試次數 / 逾時值

<a name="max-attempts"></a>
#### 最大嘗試次數

如果您的佇列 Job 遇到錯誤，您可能不希望它無限期地重試。因此，Laravel 提供了多種方法來指定 Job 可以嘗試的次數或時間長度。

指定 Job 最大嘗試次數的一種方法是透過 Artisan 命令列上的 `--tries` 開關。這將適用於 Worker 處理的所有 Job，除非正在處理的 Job 指定了它可以嘗試的次數：

```shell
php artisan queue:work --tries=3
```

如果 Job 超過其最大嘗試次數，它將被視為「失敗」Job。有關處理失敗 Job 的更多資訊，請參閱 [失敗 Job 說明文件](#dealing-with-failed-jobs)。如果向 `queue:work` 命令提供了 `--tries=0`，則 Job 將無限期重試。

您可以透過在 Job Class 本身定義 Job 最大嘗試次數來採用更細粒度的方法。如果在 Job 上指定了最大嘗試次數，它將優先於命令列上提供的 `--tries` 值：

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

如果您需要對特定 Job 的最大嘗試次數進行動態控制，您可以在 Job 上定義一個 `tries` 方法：

    /**
     * Determine number of times the job may be attempted.
     */
    public function tries(): int
    {
        return 5;
    }

<a name="time-based-attempts"></a>
#### 基於時間的嘗試

作為定義 Job 在失敗之前可以嘗試多少次的一種替代方法，您可以定義 Job 不應再嘗試的時間。這允許 Job 在給定時間範圍內嘗試任意次數。要定義 Job 不應再嘗試的時間，請在 Job Class 中新增一個 `retryUntil` 方法。此方法應返回一個 `DateTime` 實例：

    use DateTime;

    /**
     * Determine the time at which the job should timeout.
     */
    public function retryUntil(): DateTime
    {
        return now()->addMinutes(10);
    }

> [!NOTE]
> 您也可以在 [佇列事件監聽器](/docs/{{version}}/events#queued-event-listeners) 上定義 `tries` 屬性或 `retryUntil` 方法。

<a name="max-exceptions"></a>
#### 最大例外數

有時您可能希望指定 Job 可以嘗試多次，但如果重試是由給定數量的未處理例外觸發 (而不是直接由 `release` 方法釋放)，則應失敗。為此，您可以在 Job Class 上定義一個 `maxExceptions` 屬性：

    <?php

    namespace App\Jobs;

    use Illuminate\Support\Facades\Redis;

    class ProcessPodcast implements ShouldQueue
    {
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

在此範例中，如果應用程式無法獲取 Redis 鎖定，則 Job 將被釋放十秒鐘，並將繼續重試最多 25 次。但是，如果 Job 拋出三個未處理的例外，則 Job 將失敗。

<a name="timeout"></a>
#### 逾時

通常，您大致知道您預期佇列 Job 需要多長時間。因此，Laravel 允許您指定一個「逾時」值。預設情況下，逾時值為 60 秒。如果 Job 處理時間超過逾時值指定的秒數，則處理 Job 的 Worker 將以錯誤退出。通常，Worker 將由 [伺服器上設定的程序管理器](#supervisor-configuration) 自動重新啟動。

Job 可以運行的最大秒數可以使用 Artisan 命令列上的 `--timeout` 開關指定：

```shell
php artisan queue:work --timeout=30
```

如果 Job 因持續逾時而超過其最大嘗試次數，它將被標記為失敗。

您也可以在 Job Class 本身定義 Job 應該允許運行的最大秒數。如果在 Job 上指定了逾時，它將優先於命令列上指定的任何逾時：

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

有時，IO 阻塞程序 (例如 socket 或傳出 HTTP 連線) 可能不遵守您指定的逾時。因此，在使用這些功能時，您應該始終嘗試使用其 API 指定逾時。例如，使用 Guzzle 時，您應該始終指定連線和請求逾時值。

> [!WARNING]
> 必須安裝 `pcntl` PHP 擴充功能才能指定 Job 逾時。此外，Job 的「逾時」值應始終小於其 [「重試後」](#job-expiration) 值。否則，Job 可能會在實際完成執行或逾時之前重新嘗試。

<a name="failing-on-timeout"></a>
#### 逾時失敗

如果您想指示 Job 應在逾時時被標記為 [失敗](#dealing-with-failed-jobs)，您可以在 Job Class 上定義 `$failOnTimeout` 屬性：

```php
/**
 * Indicate if the job should be marked as failed on timeout.
 *
 * @var bool
 */
public $failOnTimeout = true;
```

<a name="error-handling"></a>
### 錯誤處理

如果在處理 Job 時拋出例外，Job 將自動釋放回佇列，以便稍後再次嘗試。Job 將繼續釋放，直到它已嘗試應用程式允許的最大次數。最大嘗試次數由 `queue:work` Artisan 命令上使用的 `--tries` 開關定義。或者，最大嘗試次數可以在 Job Class 本身定義。有關運行佇列 Worker 的更多資訊 [可以在下面找到](#running-the-queue-worker)。

<a name="manually-releasing-a-job"></a>
#### 手動釋放 Job

有時您可能希望手動將 Job 釋放回佇列，以便稍後再次嘗試。您可以透過呼叫 `release` 方法來實現此目的：

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        // ...

        $this->release();
    }

預設情況下，`release` 方法會將 Job 釋放回佇列以立即處理。但是，您可以指示佇列在給定秒數過去之前不讓 Job 可用於處理，方法是將整數或日期實例傳遞給 `release` 方法：

    $this->release(10);

    $this->release(now()->addSeconds(10));

<a name="manually-failing-a-job"></a>
#### 手動使 Job 失敗

有時您可能需要手動將 Job 標記為「失敗」。為此，您可以呼叫 `fail` 方法：

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        // ...

        $this->fail();
    }

如果您想因為您捕獲的例外而將 Job 標記為失敗，您可以將例外傳遞給 `fail` 方法。或者，為了方便起見，您可以傳遞一個字串錯誤訊息，該訊息將為您轉換為例外：

    $this->fail($exception);

    $this->fail('Something went wrong.');

> [!NOTE]
> 有關失敗 Job 的更多資訊，請查看 [處理 Job 失敗的說明文件](#dealing-with-failed-jobs)。

<a name="job-batching"></a>
## Job 批次處理

Laravel 的 Job 批次處理功能允許您輕鬆執行一批 Job，然後在該批 Job 完成執行後執行某些操作。在開始之前，您應該建立一個資料庫遷移來建立一個表格，該表格將包含有關您的 Job 批次的元資訊，例如它們的完成百分比。此遷移可以使用 `make:queue-batches-table` Artisan 命令產生：

```shell
php artisan make:queue-batches-table

php artisan migrate
```

<a name="defining-batchable-jobs"></a>
### 定義可批次處理的 Job

要定義可批次處理的 Job，您應該像往常一樣 [建立一個可佇列的 Job](#creating-jobs)；但是，您應該將 `Illuminate\Bus\Batchable` trait 新增到 Job Class。此 trait 提供了對 `batch` 方法的存取，該方法可用於擷取 Job 正在執行的當前批次：

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

<a name="dispatching-batches"></a>
### 分派批次

要分派一批 Job，您應該使用 `Bus` Facade 的 `batch` 方法。當然，批次處理主要在與完成回呼結合使用時才有用。因此，您可以使用 `then`、`catch` 和 `finally` 方法來定義批次的完成回呼。這些回呼中的每一個在被呼叫時都將接收一個 `Illuminate\Bus\Batch` 實例。在此範例中，我們將想像我們正在佇列一批 Job，每個 Job 都處理 CSV 檔案中的給定行數：

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

批次的 ID，可以透過 `$batch->id` 屬性存取，可用於在分派後 [查詢 Laravel 命令 Bus](#inspecting-batches) 以獲取有關批次的資訊。

> [!WARNING]
> 由於批次回呼由 Laravel 佇列序列化並在稍後執行，因此您不應在回呼中使用 `$this` 變數。此外，由於批次 Job 包裝在資料庫交易中，因此不應在 Job 中執行觸發隱式提交的資料庫語句。

<a name="naming-batches"></a>
#### 命名批次

某些工具 (例如 Laravel Horizon 和 Laravel Telescope) 可能會為批次提供更友善的偵錯資訊，如果批次已命名。要為批次分配任意名稱，您可以在定義批次時呼叫 `name` 方法：

    $batch = Bus::batch([
        // ...
    ])->then(function (Batch $batch) {
        // All jobs completed successfully...
    })->name('Import CSV')->dispatch();

<a name="batch-connection-queue"></a>
#### 批次連線與佇列

如果您想指定批次 Job 應該使用的連線和佇列，您可以使用 `onConnection` 和 `onQueue` 方法。所有批次 Job 都必須在相同的連線和佇列中執行：

    $batch = Bus::batch([
        // ...
    ])->then(function (Batch $batch) {
        // All jobs completed successfully...
    })->onConnection('redis')->onQueue('imports')->dispatch();

<a name="chains-and-batches"></a>
### 鏈與批次

您可以透過將鏈接 Job 放置在陣列中來定義批次中的一組 [鏈接 Job](#job-chaining)。例如，我們可以並行執行兩個 Job 鏈，並在兩個 Job 鏈都完成處理後執行回呼：

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
        // ...
    })->dispatch();

相反，您可以透過在鏈中定義批次來在 [鏈](#job-chaining) 中運行 Job 批次。例如，您可以首先運行一批 Job 來發布多個 Podcast，然後運行一批 Job 來發送發布通知：

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

<a name="adding-jobs-to-batches"></a>
### 將 Job 加入批次

有時，從批次 Job 內部向批次新增額外 Job 可能會很有用。當您需要批次處理數千個 Job，而這些 Job 在 Web 請求期間可能需要太長時間才能分派時，這種模式會很有用。因此，您可以改為分派一批初始的「載入器」Job，這些 Job 會用更多 Job 填充批次：

    $batch = Bus::batch([
        new LoadImportBatch,
        new LoadImportBatch,
        new LoadImportBatch,
    ])->then(function (Batch $batch) {
        // All jobs completed successfully...
    })->name('Import Contacts')->dispatch();

在此範例中，我們將使用 `LoadImportBatch` Job 來用額外 Job 填充批次。為此，我們可以使用批次實例上的 `add` 方法，該方法可以透過 Job 的 `batch` 方法存取：

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

> [!WARNING]
> 您只能從屬於同一批次的 Job 內部向批次新增 Job。

<a name="inspecting-batches"></a>
### 檢查批次

提供給批次完成回呼的 `Illuminate\Bus\Batch` 實例具有各種屬性和方法，可協助您與給定 Job 批次互動和檢查：

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

<a name="returning-batches-from-routes"></a>
#### 從路由返回批次

所有 `Illuminate\Bus\Batch` 實例都是 JSON 可序列化的，這表示您可以直接從應用程式的路由返回它們，以擷取包含批次資訊 (包括其完成進度) 的 JSON payload。這使得在應用程式的 UI 中顯示有關批次完成進度的資訊變得方便。

要透過其 ID 擷取批次，您可以使用 `Bus` Facade 的 `findBatch` 方法：

    use Illuminate\Support\Facades\Bus;
    use Illuminate\Support\Facades\Route;

    Route::get('/batch/{batchId}', function (string $batchId) {
        return Bus::findBatch($batchId);
    });

<a name="cancelling-batches"></a>
### 取消批次

有時您可能需要取消給定批次的執行。這可以透過呼叫 `Illuminate\Bus\Batch` 實例上的 `cancel` 方法來實現：

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        if ($this->user->exceedsImportLimit()) {
            return $this->batch()->cancel();
        }

        if ($this->batch()->cancelled()) {
            return;
        }
    }

如您在前面的範例中可能已經注意到，批次 Job 通常應在繼續執行之前判斷其對應的批次是否已取消。但是，為了方便起見，您可以改為將 `SkipIfBatchCancelled` [middleware](#job-middleware) 分配給 Job。顧名思義，此 middleware 將指示 Laravel 在其對應的批次已取消時不處理 Job：

    use Illuminate\Queue\Middleware\SkipIfBatchCancelled;

    /**
     * Get the middleware the job should pass through.
     */
    public function middleware(): array
    {
        return [new SkipIfBatchCancelled];
    }

<a name="batch-failures"></a>
### 批次失敗

當批次 Job 失敗時，`catch` 回呼 (如果已分配) 將被呼叫。此回呼僅針對批次中第一個失敗的 Job 呼叫。

<a name="allowing-failures"></a>
#### 允許失敗

當批次中的 Job 失敗時，Laravel 將自動將批次標記為「已取消」。如果您願意，您可以停用此行為，以便 Job 失敗不會自動將批次標記為已取消。這可以透過在分派批次時呼叫 `allowFailures` 方法來實現：

    $batch = Bus::batch([
        // ...
    ])->then(function (Batch $batch) {
        // All jobs completed successfully...
    })->allowFailures()->dispatch();

<a name="retrying-failed-batch-jobs"></a>
#### 重試失敗的批次 Job

為了方便起見，Laravel 提供了一個 `queue:retry-batch` Artisan 命令，允許您輕鬆重試給定批次的所有失敗 Job。`queue:retry-batch` 命令接受應重試其失敗 Job 的批次的 UUID：

```shell
php artisan queue:retry-batch 32dbc76c-4f82-4749-b610-a639fe0099b5
```

<a name="pruning-batches"></a>
### 修剪批次

如果不修剪，`job_batches` 表格可能會非常快速地累積記錄。為了緩解此問題，您應該 [排程](/docs/{{version}}/scheduling) `queue:prune-batches` Artisan 命令每天運行：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('queue:prune-batches')->daily();

預設情況下，所有超過 24 小時的已完成批次都將被修剪。您可以在呼叫命令時使用 `hours` 選項來確定保留批次資料的時間長度。例如，以下命令將刪除所有超過 48 小時的已完成批次：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('queue:prune-batches --hours=48')->daily();

有時，您的 `jobs_batches` 表格可能會累積未成功完成的批次記錄，例如 Job 失敗且該 Job 從未成功重試的批次。您可以使用 `unfinished` 選項指示 `queue:prune-batches` 命令修剪這些未完成的批次記錄：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('queue:prune-batches --hours=48 --unfinished=72')->daily();

同樣，您的 `jobs_batches` 表格也可能累積已取消批次的批次記錄。您可以使用 `cancelled` 選項指示 `queue:prune-batches` 命令修剪這些已取消的批次記錄：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('queue:prune-batches --hours=48 --cancelled=72')->daily();

<a name="storing-batches-in-dynamodb"></a>
### 將批次儲存至 DynamoDB

Laravel 還支援將批次元資訊儲存在 [DynamoDB](https://aws.amazon.com/dynamodb) 而不是關聯式資料庫中。但是，您需要手動建立一個 DynamoDB 表格來儲存所有批次記錄。

通常，此表格應命名為 `job_batches`，但您應根據應用程式 `queue` 設定檔中 `queue.batching.table` 設定值來命名表格。

<a name="dynamodb-batch-table-configuration"></a>
#### DynamoDB 批次表格設定

`job_batches` 表格應具有一個名為 `application` 的字串主分割區 key 和一個名為 `id` 的字串主排序 key。key 的 `application` 部分將包含應用程式 `app` 設定檔中 `name` 設定值定義的應用程式名稱。由於應用程式名稱是 DynamoDB 表格 key 的一部分，因此您可以使用相同的表格來儲存多個 Laravel 應用程式的 Job 批次。

此外，如果您想利用 [DynamoDB 中的自動批次修剪](#pruning-batches-in-dynamodb)，您可以為表格定義 `ttl` 屬性。

<a name="dynamodb-configuration"></a>
#### DynamoDB 設定

接下來，安裝 AWS SDK，以便您的 Laravel 應用程式可以與 Amazon DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

然後，將 `queue.batching.driver` 設定選項的值設定為 `dynamodb`。此外，您應該在 `batching` 設定陣列中定義 `key`、`secret` 和 `region` 設定選項。這些選項將用於向 AWS 進行身份驗證。使用 `dynamodb` 驅動程式時，`queue.batching.database` 設定選項是不必要的：

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
#### DynamoDB 中的批次修剪

當利用 [DynamoDB](https://aws.amazon.com/dynamodb) 儲存 Job 批次資訊時，用於修剪儲存在關聯式資料庫中的批次的典型修剪命令將不起作用。相反，您可以利用 [DynamoDB 的原生 TTL 功能](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html) 來自動刪除舊批次的記錄。

如果您使用 `ttl` 屬性定義了 DynamoDB 表格，您可以定義設定參數來指示 Laravel 如何修剪批次記錄。`queue.batching.ttl_attribute` 設定值定義了儲存 TTL 的屬性名稱，而 `queue.batching.ttl` 設定值定義了批次記錄可以從 DynamoDB 表格中刪除的秒數，相對於記錄上次更新的時間：

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
## 佇列 Closure

除了將 Job Class 分派到佇列之外，您還可以分派 closure。這對於需要執行在當前請求週期之外的快速、簡單任務非常有用。將 closure 分派到佇列時，closure 的程式碼內容會經過加密簽名，因此在傳輸過程中無法修改：

    $podcast = App\Podcast::find(1);

    dispatch(function () use ($podcast) {
        $podcast->publish();
    });

使用 `catch` 方法，您可以提供一個 closure，該 closure 應在佇列 closure 在耗盡所有佇列的 [設定重試嘗試次數](#max-job-attempts-and-timeout) 後未能成功完成時執行：

    use Throwable;

    dispatch(function () use ($podcast) {
        $podcast->publish();
    })->catch(function (Throwable $e) {
        // This job has failed...
    });

> [!WARNING]
> 由於 `catch` 回呼由 Laravel 佇列序列化並在稍後執行，因此您不應在 `catch` 回呼中使用 `$this` 變數。

<a name="running-the-queue-worker"></a>
## 執行佇列 Worker

<a name="the-queue-work-command"></a>
### `queue:work` 命令

Laravel 包含一個 Artisan 命令，它將啟動一個佇列 Worker 並處理推送到佇列中的新 Job。您可以使用 `queue:work` Artisan 命令運行 Worker。請注意，一旦 `queue:work` 命令啟動，它將繼續運行，直到手動停止或您關閉終端機：

```shell
php artisan queue:work
```

> [!NOTE]
> 為了讓 `queue:work` 程序永久在背景運行，您應該使用 [Supervisor](#supervisor-configuration) 等程序監視器來確保佇列 Worker 不會停止運行。

如果您希望將已處理的 Job ID 包含在命令的輸出中，您可以在呼叫 `queue:work` 命令時包含 `-v` 旗標：

```shell
php artisan queue:work -v
```

請記住，佇列 Worker 是長期運行的程序，並將啟動的應用程式狀態儲存在記憶體中。因此，它們在啟動後不會注意到程式碼庫中的更改。因此，在您的部署過程中，請務必 [重新啟動您的佇列 Worker](#queue-workers-and-deployment)。此外，請記住，應用程式建立或修改的任何靜態狀態都不會在 Job 之間自動重置。

或者，您可以運行 `queue:listen` 命令。使用 `queue:listen` 命令時，您無需在要重新載入更新的程式碼或重置應用程式狀態時手動重新啟動 Worker；但是，此命令的效率明顯低於 `queue:work` 命令：

```shell
php artisan queue:listen
```

<a name="running-multiple-queue-workers"></a>
#### 運行多個佇列 Worker

要為佇列分配多個 Worker 並並行處理 Job，您只需啟動多個 `queue:work` 程序。這可以在本地透過終端機中的多個選項卡完成，也可以在生產環境中使用程序管理器的設定完成。[使用 Supervisor 時](#supervisor-configuration)，您可以使用 `numprocs` 設定值。

<a name="specifying-the-connection-queue"></a>
#### 指定連線與佇列

您還可以指定 Worker 應該使用的佇列連線。傳遞給 `work` 命令的連線名稱應與 `config/queue.php` 設定檔中定義的其中一個連線相對應：

```shell
php artisan queue:work redis
```

預設情況下，`queue:work` 命令只處理給定連線的預設佇列的 Job。但是，您可以透過僅處理給定連線的特定佇列來進一步自訂您的佇列 Worker。例如，如果您的所有電子郵件都在 `redis` 佇列連線上的 `emails` 佇列中處理，您可以發出以下命令來啟動一個只處理該佇列的 Worker：

```shell
php artisan queue:work redis --queue=emails
```

<a name="processing-a-specified-number-of-jobs"></a>
#### 處理指定數量的 Job

`--once` 選項可用於指示 Worker 只處理佇列中的單個 Job：

```shell
php artisan queue:work --once
```

`--max-jobs` 選項可用於指示 Worker 處理給定數量的 Job，然後退出。此選項與 [Supervisor](#supervisor-configuration) 結合使用時可能很有用，以便您的 Worker 在處理給定數量的 Job 後自動重新啟動，釋放它們可能累積的任何記憶體：

```shell
php artisan queue:work --max-jobs=1000
```

<a name="processing-all-queued-jobs-then-exiting"></a>
#### 處理所有佇列 Job 然後退出

`--stop-when-empty` 選項可用於指示 Worker 處理所有 Job，然後優雅地退出。此選項在 Docker 容器中處理 Laravel 佇列時很有用，如果您希望在佇列為空後關閉容器：

```shell
php artisan queue:work --stop-when-empty
```

<a name="processing-jobs-for-a-given-number-of-seconds"></a>
#### 處理 Job 指定秒數

`--max-time` 選項可用於指示 Worker 處理 Job 指定秒數，然後退出。此選項與 [Supervisor](#supervisor-configuration) 結合使用時可能很有用，以便您的 Worker 在處理 Job 指定時間後自動重新啟動，釋放它們可能累積的任何記憶體：

```shell
# Process jobs for one hour and then exit...
php artisan queue:work --max-time=3600
```

<a name="worker-sleep-duration"></a>
#### Worker 睡眠時間

當佇列中有 Job 可用時，Worker 將繼續處理 Job，Job 之間沒有延遲。但是，`sleep` 選項決定了如果沒有 Job 可用，Worker 將「睡眠」多少秒。當然，在睡眠期間，Worker 將不會處理任何新 Job：

```shell
php artisan queue:work --sleep=3
```

<a name="maintenance-mode-queues"></a>
#### 維護模式與佇列

當您的應用程式處於 [維護模式](/docs/{{version}}/configuration#maintenance-mode) 時，不會處理任何佇列 Job。一旦應用程式退出維護模式，Job 將繼續正常處理。

要強制您的佇列 Worker 處理 Job，即使維護模式已啟用，您可以使用 `--force` 選項：

```shell
php artisan queue:work --force
```

<a name="resource-considerations"></a>
#### 資源考量

Daemon 佇列 Worker 在處理每個 Job 之前不會「重新啟動」框架。因此，您應該在每個 Job 完成後釋放任何大量資源。例如，如果您正在使用 GD 函式庫進行圖片處理，您應該在處理完圖片後使用 `imagedestroy` 釋放記憶體。

<a name="queue-priorities"></a>
### 佇列優先順序

有時您可能希望優先處理佇列。例如，在您的 `config/queue.php` 設定檔中，您可以將 `redis` 連線的預設 `queue` 設定為 `low`。但是，有時您可能希望將 Job 推送到 `high` 優先順序佇列，如下所示：

    dispatch((new Job)->onQueue('high'));

要啟動一個 Worker，該 Worker 會驗證所有 `high` 佇列 Job 都已處理，然後再繼續處理 `low` 佇列上的任何 Job，請將逗號分隔的佇列名稱列表傳遞給 `work` 命令：

```shell
php artisan queue:work --queue=high,low
```

<a name="queue-workers-and-deployment"></a>
### 佇列 Worker 與部署

由於佇列 Worker 是長期運行的程序，它們在不重新啟動的情況下不會注意到程式碼的更改。因此，部署使用佇列 Worker 的應用程式最簡單的方法是在部署過程中重新啟動 Worker。您可以透過發出 `queue:restart` 命令來優雅地重新啟動所有 Worker：

```shell
php artisan queue:restart
```

此命令將指示所有佇列 Worker 在完成處理當前 Job 後優雅地退出，以便不會丟失任何現有 Job。由於佇列 Worker 在執行 `queue:restart` 命令時將退出，因此您應該運行一個程序管理器，例如 [Supervisor](#supervisor-configuration) 來自動重新啟動佇列 Worker。

> [!NOTE]
> 佇列使用 [快取](/docs/{{version}}/cache) 來儲存重新啟動訊號，因此在使用此功能之前，您應該驗證您的應用程式是否已正確設定快取驅動程式。

<a name="job-expirations-and-timeouts"></a>
### Job 過期與逾時

<a name="job-expiration"></a>
#### Job 過期

在您的 `config/queue.php` 設定檔中，每個佇列連線都定義了一個 `retry_after` 選項。此選項指定佇列連線在重試正在處理的 Job 之前應該等待多少秒。例如，如果 `retry_after` 的值設定為 `90`，則如果 Job 已處理 90 秒而未釋放或刪除，它將被釋放回佇列。通常，您應該將 `retry_after` 值設定為您的 Job 合理完成處理所需的最大秒數。

> [!WARNING]
> 唯一不包含 `retry_after` 值的佇列連線是 Amazon SQS。SQS 將根據 [預設可見性逾時](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html) 重試 Job，該逾時在 AWS 控制台中管理。

<a name="worker-timeouts"></a>
#### Worker 逾時

`queue:work` Artisan 命令公開了一個 `--timeout` 選項。預設情況下，`--timeout` 值為 60 秒。如果 Job 處理時間超過逾時值指定的秒數，則處理 Job 的 Worker 將以錯誤退出。通常，Worker 將由 [伺服器上設定的程序管理器](#supervisor-configuration) 自動重新啟動：

```shell
php artisan queue:work --timeout=60
```

`retry_after` 設定選項和 `--timeout` CLI 選項是不同的，但它們協同工作以確保 Job 不會丟失，並且 Job 只成功處理一次。

> [!WARNING]
> `--timeout` 值應始終比您的 `retry_after` 設定值至少短幾秒鐘。這將確保處理凍結 Job 的 Worker 始終在 Job 重試之前終止。如果您的 `--timeout` 選項長於您的 `retry_after` 設定值，您的 Job 可能會被處理兩次。

<a name="supervisor-configuration"></a>
## Supervisor 設定

在生產環境中，您需要一種方法來保持 `queue:work` 程序運行。`queue:work` 程序可能因各種原因停止運行，例如 Worker 逾時或執行 `queue:restart` 命令。

因此，您需要設定一個程序監視器，該監視器可以檢測您的 `queue:work` 程序何時退出並自動重新啟動它們。此外，程序監視器可以讓您指定要同時運行多少個 `queue:work` 程序。Supervisor 是 Linux 環境中常用的程序監視器，我們將在以下說明文件中討論如何設定它。

<a name="installing-supervisor"></a>
#### 安裝 Supervisor

Supervisor 是 Linux 作業系統的程序監視器，如果您的 `queue:work` 程序失敗，它將自動重新啟動它們。要在 Ubuntu 上安裝 Supervisor，您可以使用以下命令：

```shell
sudo apt-get install supervisor
```

> [!NOTE]
> 如果自行設定和管理 Supervisor 聽起來很麻煩，請考慮使用 [Laravel Forge](https://forge.laravel.com)，它將自動為您的生產 Laravel 專案安裝和設定 Supervisor。

<a name="configuring-supervisor"></a>
#### 設定 Supervisor

Supervisor 設定檔通常儲存在 `/etc/supervisor/conf.d` 目錄中。在此目錄中，您可以建立任意數量的設定檔，這些設定檔指示 Supervisor 應如何監視您的程序。例如，讓我們建立一個 `laravel-worker.conf` 檔案，該檔案啟動並監視 `queue:work` 程序：

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

在此範例中，`numprocs` 指令將指示 Supervisor 運行八個 `queue:work` 程序並監視所有這些程序，如果它們失敗則自動重新啟動它們。您應該更改設定的 `command` 指令以反映您所需的佇列連線和 Worker 選項。

> [!WARNING]
> 您應該確保 `stopwaitsecs` 的值大於您運行時間最長的 Job 所消耗的秒數。否則，Supervisor 可能會在 Job 完成處理之前將其終止。

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

有時您的佇列 Job 會失敗。別擔心，事情並不總是按計劃進行！Laravel 包含一種方便的方法來 [指定 Job 應該嘗試的最大次數](#max-job-attempts-and-timeout)。非同步 Job 超過此嘗試次數後，它將被插入到 `failed_jobs` 資料庫表格中。[同步分派的 Job](/docs/{{version}}/queues#synchronous-dispatching) 失敗後不會儲存在此表格中，其例外會立即由應用程式處理。

建立 `failed_jobs` 表格的遷移通常已存在於新的 Laravel 應用程式中。但是，如果您的應用程式不包含此表格的遷移，您可以使用 `make:queue-failed-table` 命令來建立遷移：

```shell
php artisan make:queue-failed-table

php artisan migrate
```

運行 [佇列 Worker](#running-the-queue-worker) 程序時，您可以使用 `queue:work` 命令上的 `--tries` 開關指定 Job 應該嘗試的最大次數。如果您沒有為 `--tries` 選項指定值，則 Job 只會嘗試一次，或者 Job Class 的 `$tries` 屬性指定的次數：

```shell
php artisan queue:work redis --tries=3
```

使用 `--backoff` 選項，您可以指定 Laravel 在重試遇到例外的 Job 之前應該等待多少秒。預設情況下，Job 會立即釋放回佇列，以便可以再次嘗試：

```shell
php artisan queue:work redis --tries=3 --backoff=3
```

如果您想為每個 Job 設定 Laravel 在重試遇到例外的 Job 之前應該等待多少秒，您可以透過在 Job Class 上定義 `backoff` 屬性來實現：

    /**
     * The number of seconds to wait before retrying the job.
     *
     * @var int
     */
    public $backoff = 3;

如果您需要更複雜的邏輯來確定 Job 的退避時間，您可以在 Job Class 上定義一個 `backoff` 方法：

    /**
    * Calculate the number of seconds to wait before retrying the job.
    */
    public function backoff(): int
    {
        return 3;
    }

您可以透過從 `backoff` 方法返回退避值陣列來輕鬆設定「指數」退避。在此範例中，第一次重試的重試延遲為 1 秒，第二次重試為 5 秒，第三次重試為 10 秒，如果還有更多嘗試，則每次後續重試為 10 秒：

    /**
    * Calculate the number of seconds to wait before retrying the job.
    *
    * @return array<int, int>
    */
    public function backoff(): array
    {
        return [1, 5, 10];
    }

<a name="cleaning-up-after-failed-jobs"></a>
### 失敗 Job 後的清理

當特定 Job 失敗時，您可能希望向使用者發送警報或還原 Job 部分完成的任何操作。為此，您可以在 Job Class 上定義一個 `failed` 方法。導致 Job 失敗的 `Throwable` 實例將傳遞給 `failed` 方法：

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

> [!WARNING]
> 在呼叫 `failed` 方法之前會實例化一個新的 Job 實例；因此，在 `handle` 方法中可能發生的任何 Class 屬性修改都將丟失。

<a name="retrying-failed-jobs"></a>
### 重試失敗的 Job

要查看已插入到 `failed_jobs` 資料庫表格中的所有失敗 Job，您可以使用 `queue:failed` Artisan 命令：

```shell
php artisan queue:failed
```

`queue:failed` 命令將列出 Job ID、連線、佇列、失敗時間以及有關 Job 的其他資訊。Job ID 可用於重試失敗的 Job。例如，要重試 ID 為 `ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece` 的失敗 Job，請發出以下命令：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece
```

如有必要，您可以將多個 ID 傳遞給命令：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece 91401d2c-0784-4f43-824c-34f94a33c24d
```

您還可以重試特定佇列的所有失敗 Job：

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
> 使用 [Horizon](/docs/{{version}}/horizon) 時，您應該使用 `horizon:forget` 命令刪除失敗的 Job，而不是 `queue:forget` 命令。

要從 `failed_jobs` 表格中刪除所有失敗的 Job，您可以使用 `queue:flush` 命令：

```shell
php artisan queue:flush
```

<a name="ignoring-missing-models"></a>
### 忽略遺失的 Model

將 Eloquent model 注入 Job 時，model 會在放入佇列之前自動序列化，並在 Job 處理時從資料庫中重新擷取。但是，如果 Job 在等待 Worker 處理時 model 已被刪除，您的 Job 可能會因 `ModelNotFoundException` 而失敗。

為了方便起見，您可以選擇透過將 Job 的 `deleteWhenMissingModels` 屬性設定為 `true` 來自動刪除缺少 model 的 Job。當此屬性設定為 `true` 時，Laravel 將悄悄地丟棄 Job 而不會引發例外：

    /**
     * Delete the job if its models no longer exist.
     *
     * @var bool
     */
    public $deleteWhenMissingModels = true;

<a name="pruning-failed-jobs"></a>
### 修剪失敗的 Job

您可以透過呼叫 `queue:prune-failed` Artisan 命令來修剪應用程式 `failed_jobs` 表格中的記錄：

```shell
php artisan queue:prune-failed
```

預設情況下，所有超過 24 小時的失敗 Job 記錄都將被修剪。如果您向命令提供 `--hours` 選項，則只會保留在過去 N 小時內插入的失敗 Job 記錄。例如，以下命令將刪除所有在 48 小時前插入的失敗 Job 記錄：

```shell
php artisan queue:prune-failed --hours=48
```

<a name="storing-failed-jobs-in-dynamodb"></a>
### 將失敗的 Job 儲存至 DynamoDB

Laravel 還支援將失敗的 Job 記錄儲存在 [DynamoDB](https://aws.amazon.com/dynamodb) 而不是關聯式資料庫表格中。但是，您必須手動建立一個 DynamoDB 表格來儲存所有失敗的 Job 記錄。通常，此表格應命名為 `failed_jobs`，但您應根據應用程式 `queue` 設定檔中 `queue.failed.table` 設定值來命名表格。

`failed_jobs` 表格應具有一個名為 `application` 的字串主分割區 key 和一個名為 `uuid` 的字串主排序 key。key 的 `application` 部分將包含應用程式 `app` 設定檔中 `name` 設定值定義的應用程式名稱。由於應用程式名稱是 DynamoDB 表格 key 的一部分，因此您可以使用相同的表格來儲存多個 Laravel 應用程式的失敗 Job。

此外，請確保您安裝 AWS SDK，以便您的 Laravel 應用程式可以與 Amazon DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

接下來，將 `queue.failed.driver` 設定選項的值設定為 `dynamodb`。此外，您應該在失敗 Job 設定陣列中定義 `key`、`secret` 和 `region` 設定選項。這些選項將用於向 AWS 進行身份驗證。使用 `dynamodb` 驅動程式時，`queue.failed.database` 設定選項是不必要的：

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

您可以透過將 `queue.failed.driver` 設定選項的值設定為 `null` 來指示 Laravel 丟棄失敗的 Job 而不儲存它們。通常，這可以透過 `QUEUE_FAILED_DRIVER` 環境變數來實現：

```ini
QUEUE_FAILED_DRIVER=null
```

<a name="failed-job-events"></a>
### 失敗 Job 事件

如果您想註冊一個事件監聽器，該監聽器將在 Job 失敗時被呼叫，您可以使用 `Queue` Facade 的 `failing` 方法。例如，我們可以從 Laravel 隨附的 `AppServiceProvider` 的 `boot` 方法中將一個 closure 附加到此事件：

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

<a name="clearing-jobs-from-queues"></a>
## 從佇列中清除 Job

> [!NOTE]
> 使用 [Horizon](/docs/{{version}}/horizon) 時，您應該使用 `horizon:clear` 命令從佇列中清除 Job，而不是 `queue:clear` 命令。

如果您想從預設連線的預設佇列中刪除所有 Job，您可以使用 `queue:clear` Artisan 命令：

```shell
php artisan queue:clear
```

您還可以提供 `connection` 參數和 `queue` 選項，以從特定連線和佇列中刪除 Job：

```shell
php artisan queue:clear redis --queue=emails
```

> [!WARNING]
> 從佇列中清除 Job 僅適用於 SQS、Redis 和 database 佇列驅動程式。此外，SQS 訊息刪除過程最多需要 60 秒，因此在您清除佇列後 60 秒內發送到 SQS 佇列的 Job 也可能被刪除。

<a name="monitoring-your-queues"></a>
## 監控佇列

如果您的佇列突然湧入大量 Job，它可能會不堪重負，導致 Job 完成的等待時間很長。如果您願意，當您的佇列 Job 數量超過指定閾值時，Laravel 可以提醒您。

首先，您應該 [排程](/docs/{{version}}/scheduling) `queue:monitor` 命令每分鐘運行一次。該命令接受您希望監控的佇列名稱以及您所需的 Job 數量閾值：

```shell
php artisan queue:monitor redis:default,redis:deployments --max=100
```

僅排程此命令不足以觸發通知，提醒您佇列已不堪重負。當命令遇到 Job 數量超過閾值的佇列時，將分派 `Illuminate\Queue\Events\QueueBusy` 事件。您可以在應用程式的 `AppServiceProvider` 中監聽此事件，以便向您或您的開發團隊發送通知：

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
        Notification::route('mail', 'dev @example.com')
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

測試分派 Job 的程式碼時，您可能希望指示 Laravel 實際上不執行 Job 本身，因為 Job 的程式碼可以直接且獨立於分派它的程式碼進行測試。當然，要測試 Job 本身，您可以實例化一個 Job 實例並直接在測試中呼叫 `handle` 方法。

您可以使用 `Queue` Facade 的 `fake` 方法來防止佇列 Job 實際被推送到佇列。呼叫 `Queue` Facade 的 `fake` 方法後，您可以斷言應用程式嘗試將 Job 推送到佇列：

```php tab=Pest
<?php

use App\Jobs\AnotherJob;
use App\Jobs\FinalJob;
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

    // Assert that a Closure was pushed to the queue...
    Queue::assertClosurePushed();

    // Assert the total number of jobs that were pushed...
    Queue::assertCount(3);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Jobs\AnotherJob;
use App\Jobs\FinalJob;
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

        // Assert that a Closure was pushed to the queue...
        Queue::assertClosurePushed();

        // Assert the total number of jobs that were pushed...
        Queue::assertCount(3);
    }
}
```

您可以將 closure 傳遞給 `assertPushed` 或 `assertNotPushed` 方法，以斷言推送到佇列的 Job 通過給定的「真值測試」。如果至少有一個 Job 通過給定的真值測試，則斷言將成功：

    Queue::assertPushed(function (ShipOrder $job) use ($order) {
        return $job->order->id === $order->id;
    });

<a name="faking-a-subset-of-jobs"></a>
### 模擬部分 Job

如果您只需要模擬特定 Job，同時允許其他 Job 正常執行，您可以將應該模擬的 Job 的 Class 名稱傳遞給 `fake` 方法：

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

您可以使用 `except` 方法模擬除一組指定 Job 之外的所有 Job：

    Queue::fake()->except([
        ShipOrder::class,
    ]);

<a name="testing-job-chains"></a>
### 測試 Job 鏈

要測試 Job 鏈，您需要利用 `Bus` Facade 的模擬功能。`Bus` Facade 的 `assertChained` 方法可用於斷言已分派 [Job 鏈](/docs/{{version}}/queues#job-chaining)。`assertChained` 方法將鏈接 Job 陣列作為其第一個參數：

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

如您在上面的範例中看到的，鏈接 Job 陣列可以是 Job Class 名稱的陣列。但是，您也可以提供實際的 Job 實例陣列。這樣做時，Laravel 將確保 Job 實例與應用程式分派的鏈接 Job 具有相同的 Class 和相同的屬性值：

    Bus::assertChained([
        new ShipOrder,
        new RecordShipment,
        new UpdateInventory,
    ]);

您可以使用 `assertDispatchedWithoutChain` 方法來斷言 Job 在沒有 Job 鏈的情況下被推送：

    Bus::assertDispatchedWithoutChain(ShipOrder::class);

<a name="testing-chain-modifications"></a>
#### 測試鏈修改

如果鏈接 Job [在現有鏈的前面或後面新增 Job](#adding-jobs-to-the-chain)，您可以使用 Job 的 `assertHasChain` 方法來斷言 Job 具有預期的剩餘 Job 鏈：

```php
$job = new ProcessPodcast;

$job->handle();

$job->assertHasChain([
    new TranscribePodcast,
    new OptimizePodcast,
    new ReleasePodcast,
]);
```

`assertDoesntHaveChain` 方法可用於斷言 Job 的剩餘鏈為空：

```php
$job->assertDoesntHaveChain();
```

<a name="testing-chained-batches"></a>
#### 測試鏈接批次

如果您的 Job 鏈 [包含一批 Job](#chains-and-batches)，您可以透過在鏈斷言中插入 `Bus::chainedBatch` 定義來斷言鏈接批次符合您的預期：

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

<a name="testing-job-batches"></a>
### 測試 Job 批次

`Bus` Facade 的 `assertBatched` 方法可用於斷言已分派 [Job 批次](#job-batching)。提供給 `assertBatched` 方法的 closure 接收 `Illuminate\Bus\PendingBatch` 實例，該實例可用於檢查批次中的 Job：

    use Illuminate\Bus\PendingBatch;
    use Illuminate\Support\Facades\Bus;

    Bus::fake();

    // ...

    Bus::assertBatched(function (PendingBatch $batch) {
        return $batch->name == 'import-csv' &&
               $batch->jobs->count() === 10;
    });

您可以使用 `assertBatchCount` 方法來斷言已分派給定數量的批次：

    Bus::assertBatchCount(3);

您可以使用 `assertNothingBatched` 來斷言沒有分派任何批次：

    Bus::assertNothingBatched();

<a name="testing-job-batch-interaction"></a>
#### 測試 Job / 批次互動

此外，您可能偶爾需要測試個別 Job 與其底層批次的互動。例如，您可能需要測試 Job 是否取消了其批次的進一步處理。為此，您需要透過 `withFakeBatch` 方法為 Job 分配一個模擬批次。`withFakeBatch` 方法返回一個包含 Job 實例和模擬批次的 tuple：

    [$job, $batch] = (new ShipOrder)->withFakeBatch();

    $job->handle();

    $this->assertTrue($batch->cancelled());
    $this->assertEmpty($batch->added);

<a name="testing-job-queue-interactions"></a>
### 測試 Job / 佇列互動

有時，您可能需要測試佇列 Job [將自身釋放回佇列](#manually-releasing-a-job)。或者，您可能需要測試 Job 是否刪除了自身。您可以透過實例化 Job 並呼叫 `withFakeQueueInteractions` 方法來測試這些佇列互動。

一旦 Job 的佇列互動被模擬，您可以呼叫 Job 上的 `handle` 方法。呼叫 Job 後，可以使用 `assertReleased`、`assertDeleted`、`assertNotDeleted`、`assertFailed`、`assertFailedWith` 和 `assertNotFailed` 方法來對 Job 的佇列互動進行斷言：

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

使用 `Queue` [Facade](/docs/{{version}}/facades) 上的 `before` 和 `after` 方法，您可以指定在佇列 Job 處理之前或之後執行的回呼。這些回呼是執行額外日誌記錄或增加儀表板統計資料的絕佳機會。通常，您應該從 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫這些方法。例如，我們可以使用 Laravel 隨附的 `AppServiceProvider`：

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

使用 `Queue` [Facade](/docs/{{version}}/facades) 上的 `looping` 方法，您可以指定在 Worker 嘗試從佇列中獲取 Job 之前執行的回呼。例如，您可以註冊一個 closure 來回滾先前失敗的 Job 留下的任何交易：

    use Illuminate\Support\Facades\DB;
    use Illuminate\Support\Facades\Queue;

    Queue::looping(function () {
        while (DB::transactionLevel() > 0) {
            DB::rollBack();
        }
    });
