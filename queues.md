# 佇列

- [簡介](#introduction)
    - [連線與佇列的區別](#connections-vs-queues)
    - [驅動程式說明與先決條件](#driver-prerequisites)
- [建立任務](#creating-jobs)
    - [產生任務類別](#generating-job-classes)
    - [類別結構](#class-structure)
    - [唯一任務](#unique-jobs)
    - [加密任務](#encrypted-jobs)
- [任務中介層](#job-middleware)
    - [速率限制](#rate-limiting)
    - [防止任務重疊](#preventing-job-overlaps)
    - [限制例外](#throttling-exceptions)
    - [跳過任務](#skipping-jobs)
- [分派任務](#dispatching-jobs)
    - [延遲分派](#delayed-dispatching)
    - [同步分派](#synchronous-dispatching)
    - [任務與資料庫交易](#jobs-and-database-transactions)
    - [任務鏈結](#job-chaining)
    - [自訂佇列與連線](#customizing-the-queue-and-connection)
    - [指定任務最大嘗試次數 / 逾時值](#max-job-attempts-and-timeout)
    - [錯誤處理](#error-handling)
- [任務批次處理](#job-batching)
    - [定義可批次處理的任務](#defining-batchable-jobs)
    - [分派批次](#dispatching-batches)
    - [鏈結與批次](#chains-and-batches)
    - [將任務新增至批次](#adding-jobs-to-batches)
    - [檢查批次](#inspecting-batches)
    - [取消批次](#cancelling-batches)
    - [批次失敗](#batch-failures)
    - [清理批次](#pruning-batches)
    - [在 DynamoDB 中儲存批次](#storing-batches-in-dynamodb)
- [佇列閉包](#queueing-closures)
- [執行佇列處理器](#running-the-queue-worker)
    - [`queue:work` 命令](#the-queue-work-command)
    - [佇列優先權](#queue-priorities)
    - [佇列處理器與部署](#queue-workers-and-deployment)
    - [任務過期與逾時](#job-expirations-and-timeouts)
- [Supervisor 配置](#supervisor-configuration)
- [處理失敗任務](#dealing-with-failed-jobs)
    - [清理失敗任務後的善後工作](#cleaning-up-after-failed-jobs)
    - [重試失敗任務](#retrying-failed-jobs)
    - [忽略缺失的模型](#ignoring-missing-models)
    - [清理失敗任務](#pruning-failed-jobs)
    - [在 DynamoDB 中儲存失敗任務](#storing-failed-jobs-in-dynamodb)
    - [停用失敗任務儲存](#disabling-failed-job-storage)
    - [失敗任務事件](#failed-job-events)
- [從佇列中清除任務](#clearing-jobs-from-queues)
- [監控佇列](#monitoring-your-queues)
- [測試](#testing)
    - [模擬部分任務](#faking-a-subset-of-jobs)
    - [測試任務鏈結](#testing-job-chains)
    - [測試任務批次](#testing-job-batches)
    - [測試任務 / 佇列互動](#testing-job-queue-interactions)
- [任務事件](#job-events)

<a name="introduction"></a>
## 簡介

在建構網頁應用程式時，您可能會遇到某些任務，例如解析並儲存上傳的 CSV 檔案，這些任務在一般的網頁請求期間執行時間過長。幸運的是，Laravel 讓您可以輕鬆建立可在背景處理的佇列任務 (queued jobs)。透過將耗時任務移至佇列，您的應用程式可以極快的速度回應網頁請求，並為客戶提供更好的使用者體驗。

Laravel 佇列提供一個統一的佇列 API，支援多種不同的佇列後端，例如 [Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io)，甚至是關聯式資料庫。

Laravel 的佇列設定選項儲存在應用程式的 `config/queue.php` 設定檔中。在此檔案中，您會找到框架所包含的每個佇列驅動程式的連線設定，包括 database、[Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 和 [Beanstalkd](https://beanstalkd.github.io/) 驅動程式，以及一個會立即執行任務的同步驅動程式（用於本機開發）。還包含一個 `null` 佇列驅動程式，它會丟棄佇列中的任務。

> [!NOTE]  
> Laravel 現在提供 Horizon，一個用於 Redis 驅動佇列的精美儀表板和設定系統。請查看完整的 [Horizon 文件](/docs/{{version}}/horizon) 以獲取更多資訊。

<a name="connections-vs-queues"></a>
### 連線與佇列的區別

在開始使用 Laravel 佇列之前，了解「連線 (connections)」和「佇列 (queues)」之間的區別很重要。在您的 `config/queue.php` 設定檔中，有一個 `connections` 設定陣列。此選項定義了與後端佇列服務（例如 Amazon SQS、Beanstalk 或 Redis）的連線。然而，任何給定的佇列連線都可能有多個「佇列」，這些佇列可以被視為不同堆疊或任務堆積。

請注意，`queue` 設定檔中的每個連線設定範例都包含一個 `queue` 屬性。這是任務被傳送到給定連線時，預設會被分派到的佇列。換句話說，如果您分派任務時沒有明確定義應將其分派到哪個佇列，任務將會被放置在連線設定的 `queue` 屬性中定義的佇列上：

    use App\Jobs\ProcessPodcast;

    // This job is sent to the default connection's default queue...
    ProcessPodcast::dispatch();

    // This job is sent to the default connection's "emails" queue...
    ProcessPodcast::dispatch()->onQueue('emails');

有些應用程式可能不需要將任務推送到多個佇列，而是偏好使用一個簡單的佇列。然而，將任務推送到多個佇列對於希望優先處理或區隔任務處理方式的應用程式來說特別有用，因為 Laravel 佇列處理器允許您按優先順序指定它應該處理哪些佇列。例如，如果您將任務推送到 `high` 佇列，您可以運行一個處理器，賦予它們更高的處理優先權：

```shell
php artisan queue:work --queue=high,default
```

<a name="driver-prerequisites"></a>
### 驅動程式說明與先決條件

<a name="database"></a>
#### Database

為了使用 `database` 佇列驅動程式，您需要一個資料庫資料表來儲存任務。通常，這會包含在 Laravel 預設的 `0001_01_01_000002_create_jobs_table.php` [資料庫遷移檔](/docs/{{version}}/migrations)中；但是，如果您的應用程式不包含此遷移檔，您可以使用 `make:queue-table` Artisan 命令來建立它：

```shell
php artisan make:queue-table

php artisan migrate
```

<a name="redis"></a>
#### Redis

為了使用 `redis` 佇列驅動程式，您應該在 `config/database.php` 設定檔中配置一個 Redis 資料庫連線。

> [!WARNING]  
> `serializer` 和 `compression` Redis 選項不支援 `redis` 佇列驅動程式。

**Redis Cluster**

如果您的 Redis 佇列連線使用 Redis Cluster，您的佇列名稱必須包含 [key hash tag](https://redis.io/docs/reference/cluster-spec/#hash-tags)。這是為了確保給定佇列的所有 Redis 鍵都放置在相同的雜湊槽中：

    'redis' => [
        'driver' => 'redis',
        'connection' => env('REDIS_QUEUE_CONNECTION', 'default'),
        'queue' => env('REDIS_QUEUE', '{default}'),
        'retry_after' => env('REDIS_QUEUE_RETRY_AFTER', 90),
        'block_for' => null,
        'after_commit' => false,
    ],

**阻塞**

使用 Redis 佇列時，您可以使用 `block_for` 設定選項來指定驅動程式在循環處理器並重新輪詢 Redis 資料庫之前，應該等待多長時間直到任務可用。

根據佇列負載調整此值，可以比持續輪詢 Redis 資料庫以獲取新任務更有效率。例如，您可以將值設定為 `5`，表示驅動程式在等待任務可用時應該阻塞五秒鐘：

    'redis' => [
        'driver' => 'redis',
        'connection' => env('REDIS_QUEUE_CONNECTION', 'default'),
        'queue' => env('REDIS_QUEUE', 'default'),
        'retry_after' => env('REDIS_QUEUE_RETRY_AFTER', 90),
        'block_for' => 5,
        'after_commit' => false,
    ],

> [!WARNING]  
> 將 `block_for` 設定為 `0` 會導致佇列處理器無限期地阻塞，直到有任務可用。這也會阻止處理諸如 `SIGTERM` 之類的訊號，直到下一個任務處理完成。

<a name="other-driver-prerequisites"></a>
#### 其他驅動程式先決條件

以下列出的佇列驅動程式需要以下依賴項。這些依賴項可以透過 Composer 套件管理器安裝：

<div class="content-list" markdown="1">

- Amazon SQS: `aws/aws-sdk-php ~3.0`
- Beanstalkd: `pda/pheanstalk ~5.0`
- Redis: `predis/predis ~2.0` 或 phpredis PHP 擴充功能
- [MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/queues/): `mongodb/laravel-mongodb`

</div>

<a name="creating-jobs"></a>
## 建立任務

<a name="generating-job-classes"></a>
### 產生任務類別

預設情況下，您應用程式中所有可佇列的任務都儲存在 `app/Jobs` 目錄中。如果 `app/Jobs` 目錄不存在，當您執行 `make:job` Artisan 命令時，它將會被建立：

```shell
php artisan make:job ProcessPodcast
```

產生的類別將實作 `Illuminate\Contracts\Queue\ShouldQueue` 介面，向 Laravel 表明該任務應被推送到佇列中以非同步方式執行。

> [!NOTE]
> 任務樣板可透過 [樣板發佈](/docs/{{version}}/artisan#stub-customization) 來進行自訂。

<a name="class-structure"></a>
### 類別結構

任務類別非常簡單，通常只包含一個 `handle` 方法，當佇列處理任務時會呼叫它。首先，讓我們看看一個任務類別範例。在此範例中，我們假設我們管理一個 podcast 發布服務，需要在 podcast 檔案發布之前處理已上傳的檔案：

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

在此範例中，請注意，我們可以將 [Eloquent 模型](/docs/{{version}}/eloquent) 直接傳遞給佇列任務的建構函式。由於任務使用了 `Queueable` trait，Eloquent 模型及其載入的關聯會在任務處理時被優雅地序列化與反序列化。

如果您的佇列任務在其建構函式中接受一個 Eloquent 模型，只有該模型的識別碼會被序列化到佇列中。當任務實際被處理時，佇列系統會自動從資料庫重新擷取完整的模型實例及其載入的關聯。這種模型序列化方法允許將更小的任務酬載傳送到您的佇列驅動程式。

<a name="handle-method-dependency-injection"></a>
#### `handle` 方法的依賴注入

當佇列處理任務時，會呼叫 `handle` 方法。請注意，我們可以在任務的 `handle` 方法上進行依賴型別提示。Laravel [服務容器](/docs/{{version}}/container) 會自動注入這些依賴。

如果您想完全控制容器如何將依賴注入到 `handle` 方法中，您可以使用容器的 `bindMethod` 方法。`bindMethod` 方法接受一個回呼，該回呼會接收任務和容器。在回呼中，您可以自由地以您希望的任何方式呼叫 `handle` 方法。通常，您應該從 `App\Providers\AppServiceProvider` [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫此方法：

    use App\Jobs\ProcessPodcast;
    use App\Services\AudioProcessor;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->bindMethod([ProcessPodcast::class, 'handle'], function (ProcessPodcast $job, Application $app) {
        return $job->handle($app->make(AudioProcessor::class));
    });

> [!WARNING]
> 二進位資料，例如原始影像內容，在傳遞給佇列任務之前應透過 `base64_encode` 函數傳遞。否則，當任務被放入佇列時，可能無法正確序列化為 JSON。

<a name="handling-relationships"></a>
#### 佇列關聯

因為所有載入的 Eloquent 模型關聯在任務入佇列時也會被序列化，序列化的任務字串有時會變得非常大。此外，當任務被反序列化並從資料庫重新擷取模型關聯時，它們將會被完整擷取。在任務入佇列過程中，模型被序列化之前應用的任何先前的關聯約束，在任務反序列化時將不會被應用。因此，如果您希望處理給定關聯的子集，您應該在佇列任務中重新約束該關聯。

或者，為了防止關聯被序列化，您可以在設定屬性值時，在模型上呼叫 `withoutRelations` 方法。此方法將返回一個不帶有其載入關聯的模型實例：

    /**
     * Create a new job instance.
     */
    public function __construct(
        Podcast $podcast,
    ) {
        $this->podcast = $podcast->withoutRelations();
    }

如果您正在使用 PHP 建構函式屬性提升，並希望指出 Eloquent 模型不應序列化其關聯，您可以使用 `WithoutRelations` 屬性：

    use Illuminate\Queue\Attributes\WithoutRelations;

    /**
     * Create a new job instance.
     */
    public function __construct(
        #[WithoutRelations]
        public Podcast $podcast,
    ) {}

如果任務接收的是 Eloquent 模型的集合或陣列，而不是單一模型，該集合中的模型將不會在任務反序列化和執行時恢復其關聯。這是為了防止處理大量模型的任務消耗過多資源。

<a name="unique-jobs"></a>
### 唯一任務

> [!WARNING]
> 唯一任務需要支援 [鎖定](/docs/{{version}}/cache#atomic-locks) 的快取驅動程式。目前，`memcached`、`redis`、`dynamodb`、`database`、`file` 和 `array` 快取驅動程式支援原子鎖定。此外，唯一任務的限制不適用於批次處理中的任務。

有時候，您可能想要確保在任何時間點，佇列中只有一個特定任務的實例存在。您可以透過在任務類別中實作 `ShouldBeUnique` 介面來達成此目的。此介面不需要您在類別上定義任何額外方法：

    <?php

    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Contracts\Queue\ShouldBeUnique;

    class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
    {
        ...
    }

在上面的範例中，`UpdateSearchIndex` 任務是唯一的。因此，如果佇列中已經存在該任務的另一個實例且尚未完成處理，則該任務將不會被分派。

在某些情況下，您可能想要定義一個特定的「鍵」來使任務成為唯一，或者您可能想要指定一個逾時時間，超過該時間後任務將不再保持唯一。為此，您可以在任務類別上定義 `uniqueId` 和 `uniqueFor` 屬性或方法：

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

在上面的範例中，`UpdateSearchIndex` 任務透過產品 ID 保持唯一。因此，任何具有相同產品 ID 的新任務分派將被忽略，直到現有任務完成處理為止。此外，如果現有任務在一小時內未被處理，則唯一鎖定將被釋放，另一個具有相同唯一鍵的任務即可被分派到佇列。

> [!WARNING]
> 如果您的應用程式從多個網路伺服器或容器分派任務，您應該確保所有伺服器都與相同的中央快取伺服器通訊，以便 Laravel 能夠準確判斷任務是否唯一。

<a name="keeping-jobs-unique-until-processing-begins"></a>
#### 保持任務唯一直到開始處理

預設情況下，唯一任務在完成處理或所有重試嘗試失敗後會「解除鎖定」。然而，在某些情況下，您可能希望任務在開始處理前立即解除鎖定。為此，您的任務應該實作 `ShouldBeUniqueUntilProcessing` 契約，而不是 `ShouldBeUnique` 契約：

    <?php

    use App\Models\Product;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Contracts\Queue\ShouldBeUniqueUntilProcessing;

    class UpdateSearchIndex implements ShouldQueue, ShouldBeUniqueUntilProcessing
    {
        // ...
    }

<a name="unique-job-locks"></a>
#### 唯一任務鎖定

在幕後，當一個 `ShouldBeUnique` 任務被分派時，Laravel 會嘗試使用 `uniqueId` 鍵來取得一個 [鎖定](/docs/{{version}}/cache#atomic-locks)。如果未能取得鎖定，該任務將不會被分派。此鎖定會在任務完成處理或所有重試嘗試失敗時釋放。預設情況下，Laravel 會使用預設的快取驅動程式來取得此鎖定。然而，如果您希望使用另一個驅動程式來取得鎖定，您可以定義一個 `uniqueVia` 方法，該方法會回傳應該使用的快取驅動程式：

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
> 如果您只需要限制任務的併發處理，請改用 [`WithoutOverlapping`](/docs/{{version}}/queues#preventing-job-overlaps) 任務中介層。

<a name="encrypted-jobs"></a>
### 加密任務

Laravel 允許您透過 [加密](/docs/{{version}}/encryption) 來確保任務資料的隱私和完整性。要開始使用，只需將 `ShouldBeEncrypted` 介面新增到任務類別即可。一旦此介面新增到類別後，Laravel 將在將您的任務推送到佇列之前自動加密它：

    <?php

    use Illuminate\Contracts\Queue\ShouldBeEncrypted;
    use Illuminate\Contracts\Queue\ShouldQueue;

    class UpdateSearchIndex implements ShouldQueue, ShouldBeEncrypted
    {
        // ...
    }

<a name="job-middleware"></a>
## 任務中介層

任務中介層允許您在佇列任務的執行周圍包裹自訂邏輯，減少任務本身的重複程式碼。例如，考慮以下利用 Laravel 的 Redis 速率限制功能，允許每五秒只處理一個任務的 `handle` 方法：

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

儘管這段程式碼是有效的，但 `handle` 方法的實作變得冗雜，因為它充斥著 Redis 速率限制邏輯。此外，這種速率限制邏輯必須針對任何其他我們想要進行速率限制的任務進行重複。

我們可以在任務中介層中定義一個處理速率限制的邏輯，而不是在 handle 方法中進行速率限制。Laravel 沒有為任務中介層提供預設位置，因此您可以將任務中介層放在應用程式的任何位置。在這個範例中，我們將中介層放在 `app/Jobs/Middleware` 目錄中：

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

如您所見，如同 [路由中介層](/docs/{{version}}/middleware) 一樣，任務中介層會接收正在處理的任務以及一個應被呼叫以繼續處理任務的回呼。

建立任務中介層後，可以從任務的 `middleware` 方法中回傳它們來將其附加到任務。這個方法不存在於由 `make:job` Artisan 命令生成的任務中，因此您需要手動將其新增到您的任務類別：

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
> 任務中介層也可以分配給可佇列的事件監聽器、郵件和通知。


<a name="rate-limiting"></a>
### 速率限制

儘管我們剛剛演示了如何編寫自己的速率限制任務中介層，但 Laravel 實際上包含了一個速率限制中介層，您可以用來對任務進行速率限制。如同 [路由速率限制器](/docs/{{version}}/routing#defining-rate-limiters) 一樣，任務速率限制器是使用 `RateLimiter` Facade 的 `for` 方法來定義的。

例如，您可能希望允許使用者每小時備份一次資料，而對付費客戶沒有此類限制。為了實現此目的，您可以在 `AppServiceProvider` 的 `boot` 方法中定義一個 `RateLimiter`：

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

在上面的範例中，我們定義了每小時的速率限制；然而，您可以使用 `perMinute` 方法輕鬆定義基於分鐘的速率限制。此外，您可以將任何您想要的值傳遞給速率限制的 `by` 方法；然而，這個值最常用於按客戶區分速率限制：

    return Limit::perMinute(50)->by($job->user->id);

一旦您定義了速率限制，就可以使用 `Illuminate\Queue\Middleware\RateLimited` 中介層將該速率限制器附加到您的任務。每次任務超過速率限制時，此中介層將根據速率限制的持續時間，以適當的延遲將任務釋放回佇列。

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

將受速率限制的任務釋放回佇列仍會增加任務的總嘗試次數 (`attempts`)。您可能希望相應地調整任務類別上的 `tries` 和 `maxExceptions` 屬性。或者，您可能希望使用 [`retryUntil` 方法](#time-based-attempts) 來定義任務不應再嘗試的時間量。

如果您不希望在任務受速率限制時重試，您可以使用 `dontRelease` 方法：

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
> 如果您正在使用 Redis，您可以使用 `Illuminate\Queue\Middleware\RateLimitedWithRedis` 中介層，它針對 Redis 進行了優化，比基本速率限制中介層更有效率。

<a name="preventing-job-overlaps"></a>
### 防止任務重疊

Laravel 包含一個 `Illuminate\Queue\Middleware\WithoutOverlapping` 中介層，允許您根據任意鍵防止任務重疊。當佇列任務正在修改應僅由一個任務一次修改的資源時，這會很有幫助。

例如，假設您有一個佇列任務會更新使用者的信用分數，並且您希望防止相同使用者 ID 的信用分數更新任務重疊。為此，您可以從任務的 `middleware` 方法中回傳 `WithoutOverlapping` 中介層：

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

任何相同類型的重疊任務將被釋回佇列。您也可以指定釋回任務在再次嘗試之前必須經過的秒數：

    /**
     * Get the middleware the job should pass through.
     *
     * @return array<int, object>
     */
    public function middleware(): array
    {
        return [(new WithoutOverlapping($this->order->id))->releaseAfter(60)];
    }

如果您希望立即刪除任何重疊任務，使其不再重試，您可以使用 `dontRelease` 方法：

    /**
     * Get the middleware the job should pass through.
     *
     * @return array<int, object>
     */
    public function middleware(): array
    {
        return [(new WithoutOverlapping($this->order->id))->dontRelease()];
    }

`WithoutOverlapping` 中介層由 Laravel 的原子鎖定功能提供支援。有時，您的任務可能會意外失敗或逾時，導致鎖定未被釋放。因此，您可以明確定義鎖定過期時間，方法是使用 `expireAfter` 方法。例如，以下範例將指示 Laravel 在任務開始處理三分鐘後釋放 `WithoutOverlapping` 鎖定：

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
> `WithoutOverlapping` 中介層需要一個支援 [鎖定](/docs/{{version}}/cache#atomic-locks) 的快取驅動程式。目前，`memcached`、`redis`、`dynamodb`、`database`、`file` 和 `array` 快取驅動程式都支援原子鎖定。

<a name="sharing-lock-keys"></a>
#### 跨任務類別共用鎖定鍵

預設情況下，`WithoutOverlapping` 中介層只會防止相同類別的任務重疊。因此，即使兩個不同的任務類別使用相同的鎖定鍵，它們也不會被防止重疊。但是，您可以指示 Laravel 使用 `shared` 方法將該鍵應用於所有任務類別：

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

Laravel 包含一個 `Illuminate\Queue\Middleware\ThrottlesExceptions` 中介層，允許您限制例外。一旦任務拋出指定數量的例外，所有進一步的任務執行嘗試將被延遲，直到經過指定的時間間隔。此中介層對於與不穩定的第三方服務互動的任務特別有用。

例如，假設一個佇列任務與一個開始拋出例外的第三方 API 互動。為了限制例外，您可以從任務的 `middleware` 方法中回傳 `ThrottlesExceptions` 中介層。通常，此中介層應與實作 [基於時間的嘗試](#time-based-attempts) 的任務搭配使用：

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

中介層接受的第一個建構函式參數是任務在被限制之前可以拋出的例外數量，而第二個建構函式參數是任務被限制後再次嘗試之前應經過的秒數。在上面的程式碼範例中，如果任務拋出 10 個連續例外，我們將等待 5 分鐘再嘗試該任務，並受限於 30 分鐘的時間限制。

當任務拋出例外但尚未達到例外閾值時，任務通常會立即重試。但是，您可以在將中介層附加到任務時，透過呼叫 `backoff` 方法來指定此類任務應延遲的分鐘數：

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

在內部，此中介層使用 Laravel 的快取系統來實作速率限制，並且任務的類別名稱被用作快取「鍵」。您可以在將中介層附加到任務時，透過呼叫 `by` 方法來覆寫此鍵。如果您有多個任務與相同的第三方服務互動，並且希望它們共用一個共同的限制「桶」，這會很有用：

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

預設情況下，此中介層將限制每個例外。您可以透過在將中介層附加到任務時呼叫 `when` 方法來修改此行為。然後，只有當提供給 `when` 方法的閉包回傳 `true` 時，例外才會被限制：

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

如果您希望將受限制的例外報告給應用程式的例外處理器，您可以透過在將中介層附加到任務時呼叫 `report` 方法來實現。此外，您可以提供一個閉包給 `report` 方法，並且只有當給定的閉包回傳 `true` 時，例外才會被報告：

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
> 如果您正在使用 Redis，您可以使用 `Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis` 中介層，它針對 Redis 進行了優化，比基本的例外限制中介層更有效率。

<a name="skipping-jobs"></a>
### 跳過任務

`Skip` 中介層允許您指定應跳過/刪除某個任務，而無需修改任務本身的邏輯。`Skip::when` 方法會在給定條件評估為 `true` 時刪除任務，而 `Skip::unless` 方法則會在條件評估為 `false` 時刪除任務：

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

您也可以將 `Closure` 傳遞給 `when` 和 `unless` 方法，以進行更複雜的條件判斷：

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
## 分派任務

一旦您編寫好任務類別，就可以使用任務本身的 `dispatch` 方法來分派它。傳遞給 `dispatch` 方法的參數將會傳遞給任務的建構子：

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

如果您想條件式地分派任務，可以使用 `dispatchIf` 和 `dispatchUnless` 方法：

    ProcessPodcast::dispatchIf($accountActive, $podcast);

    ProcessPodcast::dispatchUnless($accountSuspended, $podcast);

在新建立的 Laravel 應用程式中，`sync` 驅動程式是預設的佇列驅動程式。此驅動程式會同步地在目前請求的前景中執行任務，這在本地開發期間通常很方便。如果您想真正開始佇列任務進行背景處理，可以在應用程式的 `config/queue.php` 設定檔中指定不同的佇列驅動程式。

<a name="delayed-dispatching"></a>
### 延遲分派

如果您想指定任務不應立即供佇列處理器處理，可以在分派任務時使用 `delay` 方法。例如，讓我們指定一個任務在分派後 10 分鐘才能供處理：

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

在某些情況下，任務可能已經設定了預設延遲。如果您需要繞過此延遲並分派任務以立即處理，可以使用 `withoutDelay` 方法：

    ProcessPodcast::dispatch($podcast)->withoutDelay();

[!WARNING]  
Amazon SQS 佇列服務的最大延遲時間為 15 分鐘。

<a name="dispatching-after-the-response-is-sent-to-browser"></a>
#### HTTP 回應傳送至瀏覽器後分派

或者，如果您的網頁伺服器正在使用 FastCGI，`dispatchAfterResponse` 方法會將任務的分派延遲到 HTTP 回應已送達使用者瀏覽器之後。這仍然允許使用者開始使用應用程式，即使佇列任務仍在執行中。這通常只應用於大約一秒鐘的任務，例如發送電子郵件。由於它們是在目前的 HTTP 請求中處理的，因此以這種方式分派的任務不需要佇列處理器執行即可進行處理：

    use App\Jobs\SendNotification;

    SendNotification::dispatchAfterResponse();

您也可以 `dispatch` 一個閉包，並將 `afterResponse` 方法鏈結到 `dispatch` 輔助函數上，以在 HTTP 回應傳送至瀏覽器後執行該閉包：

    use App\Mail\WelcomeMessage;
    use Illuminate\Support\Facades\Mail;

    dispatch(function () {
        Mail::to('taylor@example.com')->send(new WelcomeMessage);
    })->afterResponse();

<a name="synchronous-dispatching"></a>
### 同步分派

如果您想立即 (同步地) 分派任務，可以使用 `dispatchSync` 方法。使用此方法時，任務不會被佇列，而是會在目前的程序中立即執行：

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
### 任務與資料庫交易

儘管在資料庫交易中分派任務完全沒有問題，但您應該特別注意確保您的任務能夠成功執行。在交易中分派任務時，任務有可能在父交易提交之前就被處理器處理。當這種情況發生時，您在資料庫交易期間對模型或資料庫紀錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫紀錄可能也不存在於資料庫中。

幸運的是，Laravel 提供了幾種解決這個問題的方法。首先，您可以在佇列連線的設定陣列中設定 `after_commit` 連線選項：

    'redis' => [
        'driver' => 'redis',
        // ...
        'after_commit' => true,
    ],

當 `after_commit` 選項設定為 `true` 時，您可以在資料庫交易中分派任務；但是，Laravel 會等到所有開啟的父資料庫交易都提交後，才會實際分派任務。當然，如果目前沒有開啟的資料庫交易，任務將會立即分派。

如果交易由於交易期間發生的例外而回溯，則在該交易期間分派的任務將會被丟棄。

[!NOTE]  
將 `after_commit` 設定選項設定為 `true` 也會導致所有已佇列的事件監聽器、郵件、通知和廣播事件在所有開啟的資料庫交易提交後才分派。

<a name="specifying-commit-dispatch-behavior-inline"></a>
#### 行內指定提交分派行為

如果您沒有將 `after_commit` 佇列連線設定選項設定為 `true`，您仍然可以指示特定任務應該在所有開啟的資料庫交易提交後才分派。為了實現這一點，您可以將 `afterCommit` 方法鏈結到您的分派操作上：

    use App\Jobs\ProcessPodcast;

    ProcessPodcast::dispatch($podcast)->afterCommit();

同樣地，如果 `after_commit` 設定選項設定為 `true`，您可以指示特定任務應該立即分派，而無需等待任何開啟的資料庫交易提交：

    ProcessPodcast::dispatch($podcast)->beforeCommit();

<a name="job-chaining"></a>
### 任務鏈結

任務鏈結讓您能夠指定一個排入佇列的任務列表，這些任務將在主要任務成功執行後依序執行。如果序列中的一個任務失敗，其餘任務將不會被執行。為了執行排入佇列的任務鏈結，您可以使用 `Bus` Facade 提供的 `chain` 方法。Laravel 的命令匯流排是一個底層組件，排入佇列的任務分派功能便是建立於其上：

    use App\Jobs\OptimizePodcast;
    use App\Jobs\ProcessPodcast;
    use App\Jobs\ReleasePodcast;
    use Illuminate\Support\Facades\Bus;

    Bus::chain([
        new ProcessPodcast,
        new OptimizePodcast,
        new ReleasePodcast,
    ])->dispatch();

除了鏈結任務類別實例之外，您也可以鏈結閉包：

    Bus::chain([
        new ProcessPodcast,
        new OptimizePodcast,
        function () {
            Podcast::update(/* ... */);
        },
    ])->dispatch();

> [!WARNING]  
> 在任務中使用 `$this->delete()` 方法刪除任務不會阻止鏈結任務被處理。鏈結只會在鏈結中的某個任務失敗時停止執行。


<a name="chain-connection-queue"></a>
#### 鏈結連線與佇列

如果您希望為鏈結任務指定使用的連線和佇列，您可以使用 `onConnection` 和 `onQueue` 方法。這些方法指定了應該使用的佇列連線和佇列名稱，除非排入佇列的任務明確指定了不同的連線/佇列：

    Bus::chain([
        new ProcessPodcast,
        new OptimizePodcast,
        new ReleasePodcast,
    ])->onConnection('redis')->onQueue('podcasts')->dispatch();


<a name="adding-jobs-to-the-chain"></a>
#### 將任務新增至鏈結

有時，您可能需要在鏈結中的另一個任務內部，將任務前置或附加到現有的任務鏈結中。您可以使用 `prependToChain` 和 `appendToChain` 方法來完成此操作：

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

當鏈結任務時，您可以使用 `catch` 方法指定一個閉包，該閉包將在鏈結中的任務失敗時被調用。給定的回呼將接收導致任務失敗的 `Throwable` 實例：

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
> 由於鏈結回呼會被 Laravel 佇列序列化並在稍後執行，因此您不應在鏈結回呼中使用 `$this` 變數。


<a name="customizing-the-queue-and-connection"></a>
### 自訂佇列與連線


<a name="dispatching-to-a-particular-queue"></a>
#### 分派至特定佇列

透過將任務推送到不同的佇列，您可以「分類」您的排入佇列任務，甚至可以優先處理您分配給不同佇列的工作者數量。請記住，這不是將任務推送到佇列設定檔中定義的不同佇列「連線」，而是在單一連線內推送到特定的佇列。要指定佇列，請在分派任務時使用 `onQueue` 方法：

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

或者，您可以在任務的建構函式中呼叫 `onQueue` 方法來指定任務的佇列：

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
#### 分派至特定連線

如果您的應用程式與多個佇列連線互動，您可以使用 `onConnection` 方法指定將任務推送到哪個連線：

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

您可以將 `onConnection` 和 `onQueue` 方法鏈結在一起，以指定任務的連線和佇列：

    ProcessPodcast::dispatch($podcast)
        ->onConnection('sqs')
        ->onQueue('processing');

或者，您可以在任務的建構函式中呼叫 `onConnection` 方法來指定任務的連線：

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
### 指定任務最大嘗試次數 / 逾時值


<a name="max-attempts"></a>
#### 最大嘗試次數

如果您的佇列任務遇到錯誤，您可能不希望它無限期地重試。因此，Laravel 提供多種方式來指定任務可以嘗試的次數或持續時間。

指定任務最大嘗試次數的一種方法是透過 Artisan 命令列上的 `--tries` 開關。這將適用於所有由處理器處理的任務，除非正在處理的任務本身指定了其嘗試次數：

```shell
php artisan queue:work --tries=3
```

如果任務超過其最大嘗試次數，它將被視為「失敗」的任務。有關處理失敗任務的更多資訊，請參閱[失敗任務說明文件](#dealing-with-failed-jobs)。如果向 `queue:work` 命令提供 `--tries=0`，該任務將無限期重試。

您可以透過在任務類別本身定義任務的最大嘗試次數來採取更細粒度的方法。如果在任務上指定了最大嘗試次數，它將優先於命令列上提供的 `--tries` 值：

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

如果您需要動態控制特定任務的最大嘗試次數，您可以在任務上定義一個 `tries` 方法：

    /**
     * Determine number of times the job may be attempted.
     */
    public function tries(): int
    {
        return 5;
    }


<a name="time-based-attempts"></a>
#### 基於時間的嘗試

作為定義任務失敗前嘗試次數的替代方案，您可以定義任務不應再嘗試的時間點。這允許任務在給定時間範圍內嘗試任意次數。若要定義任務不應再嘗試的時間點，請在您的任務類別中新增一個 `retryUntil` 方法。此方法應回傳一個 `DateTime` 實例：

    use DateTime;

    /**
     * Determine the time at which the job should timeout.
     */
    public function retryUntil(): DateTime
    {
        return now()->addMinutes(10);
    }

> [!NOTE]  
> 您也可以在您的[佇列事件監聽器](/docs/{{version}}/events#queued-event-listeners)上定義 `tries` 屬性或 `retryUntil` 方法。


<a name="max-exceptions"></a>
#### 最大例外次數

有時您可能希望指定任務可以嘗試多次，但如果重試是由特定數量的未處理例外觸發（而不是直接透過 `release` 方法釋放），則應失敗。為此，您可以在您的任務類別上定義一個 `maxExceptions` 屬性：

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

在這個範例中，如果應用程式無法取得 Redis 鎖定，任務將釋放十秒，並將繼續重試最多 25 次。然而，如果任務拋出三個未處理例外，則任務將失敗。


<a name="timeout"></a>
#### 逾時

通常，您大致知道佇列任務預計需要多長時間。因此，Laravel 允許您指定「逾時」值。預設情況下，逾時值為 60 秒。如果任務處理時間超過逾時值所指定的秒數，處理該任務的處理器將會以錯誤結束。通常，處理器會由[伺服器上配置的程序管理器](#supervisor-configuration)自動重啟。

任務可執行的最大秒數可以透過 Artisan 命令列上的 `--timeout` 開關指定：

```shell
php artisan queue:work --timeout=30
```

如果任務因持續逾時而超過其最大嘗試次數，它將被標記為失敗。

您也可以在任務類別本身定義任務允許執行的最大秒數。如果在任務上指定了逾時，它將優先於命令列上指定的任何逾時：

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

有時，I/O 阻塞程序（例如 sockets 或對外 HTTP 連線）可能不會遵守您指定的逾時。因此，當使用這些功能時，您應始終嘗試使用其 API 來指定逾時。例如，當使用 Guzzle 時，您應始終指定連線和請求逾時值。

> [!WARNING]  
> 必須安裝 `pcntl` PHP 擴充功能才能指定任務逾時。此外，任務的「逾時」值應始終小於其[「重試等待」](#job-expiration)值。否則，任務可能在其實際執行完成或逾時之前再次被嘗試。


<a name="failing-on-timeout"></a>
#### 逾時時失敗

如果您希望在逾時時將任務標記為[失敗](#dealing-with-failed-jobs)，您可以在任務類別上定義 `$failOnTimeout` 屬性：

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

如果任務在處理過程中拋出例外，該任務將會自動釋放回佇列，以便再次嘗試處理。該任務將會持續釋放，直到達到應用程式允許的最大嘗試次數為止。最大嘗試次數由 `queue:work` Artisan 命令使用的 `--tries` 開關定義。或者，最大嘗試次數也可以在任務類別本身中定義。有關執行佇列處理器的更多資訊，[請參閱下方](#running-the-queue-worker)。

<a name="manually-releasing-a-job"></a>
#### 手動釋放任務

有時您可能希望手動將任務釋放回佇列，以便稍後再次嘗試。您可以透過呼叫 `release` 方法來實現：

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        // ...

        $this->release();
    }

預設情況下，`release` 方法會將任務釋放回佇列以供立即處理。但是，您可以透過向 `release` 方法傳遞一個整數或日期實例，來指示佇列在經過指定秒數之前不要使任務可供處理：

    $this->release(10);

    $this->release(now()->addSeconds(10));

<a name="manually-failing-a-job"></a>
#### 手動失敗任務

有時您可能需要手動將任務標記為「失敗」。為此，您可以呼叫 `fail` 方法：

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        // ...

        $this->fail();
    }

如果您想因為捕獲的例外而將任務標記為失敗，您可以將例外傳遞給 `fail` 方法。或者，為方便起見，您可以傳遞一個字串錯誤訊息，它將被轉換為例外：

    $this->fail($exception);

    $this->fail('Something went wrong.');

> [!NOTE]  
> 有關失敗任務的更多資訊，請查閱 [處理任務失敗的說明文件](#dealing-with-failed-jobs)。

<a name="job-batching"></a>
## 任務批次處理

Laravel 的任務批次處理功能讓您可以輕鬆執行一批任務，然後在批次任務完成執行後執行某些操作。在開始之前，您應該建立一個資料庫遷移來建立一個表格，其中將包含有關您的任務批次的中繼資訊，例如它們的完成百分比。此遷移可以使用 `make:queue-batches-table` Artisan 命令產生：

```shell
php artisan make:queue-batches-table

php artisan migrate
```


<a name="defining-batchable-jobs"></a>
### 定義可批次處理的任務

若要定義一個可批次處理的任務，您應該像往常一樣[建立一個可佇列任務](#creating-jobs)；但是，您應該將 `Illuminate\Bus\Batchable` trait 新增至任務類別。這個 trait 提供對 `batch` 方法的存取，該方法可用於檢索任務正在執行的當前批次：

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

若要分派一批任務，您應該使用 `Bus` facade 的 `batch` 方法。當然，批次處理主要與完成回呼結合使用時才有用。因此，您可以使用 `then`、`catch` 和 `finally` 方法來定義批次的完成回呼。這些回呼在被調用時，每個都將接收一個 `Illuminate\Bus\Batch` 實例。在此範例中，我們假設我們正在佇列一批任務，每個任務都處理 CSV 檔案中指定數量的列：

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

批次的 ID，可以透過 `$batch->id` 屬性存取，可用於在分派後[查詢 Laravel 命令匯流排](#inspecting-batches)以獲取有關該批次的資訊。

> [!WARNING]  
> 由於批次回呼由 Laravel 佇列序列化並稍後執行，因此您不應在回呼中使用 `$this` 變數。此外，由於批次處理的任務被包裝在資料庫交易中，因此不應在任務中執行會觸發隱式提交的資料庫語句。


<a name="naming-batches"></a>
#### 命名批次

某些工具，例如 Laravel Horizon 和 Laravel Telescope，如果批次已命名，可能會為批次提供更友善的偵錯資訊。若要為批次指定任意名稱，您可以在定義批次時呼叫 `name` 方法：

    $batch = Bus::batch([
        // ...
    ])->then(function (Batch $batch) {
        // All jobs completed successfully...
    })->name('Import CSV')->dispatch();


<a name="batch-connection-queue"></a>
#### 批次連線與佇列

若要指定批次任務應使用的連線和佇列，您可以使用 `onConnection` 和 `onQueue` 方法。所有批次任務必須在相同的連線和佇列中執行：

    $batch = Bus::batch([
        // ...
    ])->then(function (Batch $batch) {
        // All jobs completed successfully...
    })->onConnection('redis')->onQueue('imports')->dispatch();


<a name="chains-and-batches"></a>
### 鏈結與批次

您可以透過將鏈結任務放置在陣列中，在批次內定義一組[鏈結任務](#job-chaining)。例如，我們可以平行執行兩個任務鏈結，並在兩個任務鏈結都處理完成後執行回呼：

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

反之，您可以透過在鏈結中定義批次，在[鏈結](#job-chaining)內執行批次任務。例如，您可以先執行一批任務來發布多個 Podcast，然後再執行一批任務來傳送發布通知：

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
### 將任務新增至批次

有時，從批次任務內部將額外任務新增到批次中可能會很有用。當您需要批次處理數千個任務，而這些任務在 Web 請求期間分派可能需要太長時間時，此模式會很有用。因此，您可以選擇分派一批初始的「載入器」任務，以向批次中注入更多任務：

    $batch = Bus::batch([
        new LoadImportBatch,
        new LoadImportBatch,
        new LoadImportBatch,
    ])->then(function (Batch $batch) {
        // All jobs completed successfully...
    })->name('Import Contacts')->dispatch();

在此範例中，我們將使用 `LoadImportBatch` 任務來向批次中注入額外任務。若要實現此目的，我們可以透過任務的 `batch` 方法存取批次實例上的 `add` 方法：

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
> 您只能從屬於相同批次的任務中將任務新增到批次。

<a name="inspecting-batches"></a>
### 檢查批次

提供給批次完成回呼的 `Illuminate\Bus\Batch` 實例，具有多種屬性與方法，可協助您與特定批次任務互動並檢查其資訊：

    // 批次的 UUID...
    $batch->id;

    // 批次的名稱 (如果適用)...
    $batch->name;

    // 指派給批次的任務數量...
    $batch->totalJobs;

    // 尚未被佇列處理的任務數量...
    $batch->pendingJobs;

    // 已失敗的任務數量...
    $batch->failedJobs;

    // 到目前為止已處理的任務數量...
    $batch->processedJobs();

    // 批次的完成百分比 (0-100)...
    $batch->progress();

    // 指示批次是否已完成執行...
    $batch->finished();

    // 取消批次的執行...
    $batch->cancel();

    // 指示批次是否已被取消...
    $batch->cancelled();


<a name="returning-batches-from-routes"></a>
#### 在路由中回傳批次

所有 `Illuminate\Bus\Batch` 實例都是可 JSON 序列化的，這表示您可以直接從應用程式的路由回傳它們，以取得包含批次資訊（包括其完成進度）的 JSON Payload。這使得在應用程式的 UI 中顯示批次完成進度資訊變得非常方便。

若要根據批次 ID 擷取批次，您可以使用 `Bus` Facade 的 `findBatch` 方法：

    use Illuminate\Support\Facades\Bus;
    use Illuminate\Support\Facades\Route;

    Route::get('/batch/{batchId}', function (string $batchId) {
        return Bus::findBatch($batchId);
    });


<a name="cancelling-batches"></a>
### 取消批次

有時您可能需要取消特定批次的執行。這可以透過呼叫 `Illuminate\Bus\Batch` 實例上的 `cancel` 方法來完成：

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

正如您在之前的範例中可能注意到的，批次處理的任務通常應在繼續執行之前判斷其對應批次是否已被取消。然而，為方便起見，您可以將 `SkipIfBatchCancelled` [中介層](#job-middleware) 指派給任務。顧名思義，此中介層將指示 Laravel 在其對應批次已被取消的情況下，不處理該任務：

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

當批次處理的任務失敗時，`catch` 回呼（如果已指派）將會被呼叫。此回呼只會在批次中第一個失敗的任務發生時被呼叫。


<a name="allowing-failures"></a>
#### 允許失敗

當批次中的任務失敗時，Laravel 將自動將該批次標記為「已取消」。如果您願意，可以停用此行為，這樣任務失敗就不會自動將批次標記為已取消。這可以透過在分派批次時呼叫 `allowFailures` 方法來完成：

    $batch = Bus::batch([
        // ...
    ])->then(function (Batch $batch) {
        // All jobs completed successfully...
    })->allowFailures()->dispatch();


<a name="retrying-failed-batch-jobs"></a>
#### 重試失敗的批次任務

為方便起見，Laravel 提供了 `queue:retry-batch` Artisan 命令，讓您可以輕鬆地重試指定批次的所有失敗任務。`queue:retry-batch` 命令接受該批次的 UUID，其失敗任務應被重試：

```shell
php artisan queue:retry-batch 32dbc76c-4f82-4749-b610-a639fe0099b5
```


<a name="pruning-batches"></a>
### 清理批次

若不進行清理，`job_batches` 資料表可能會非常快速地累積記錄。為緩解此問題，您應該[排程](/docs/{{version}}/scheduling) `queue:prune-batches` Artisan 命令每日執行：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('queue:prune-batches')->daily();

依預設，所有超過 24 小時的已完成批次都將被清理。您可以在呼叫命令時使用 `hours` 選項來決定要保留批次資料多長時間。例如，以下命令將刪除所有在 48 小時前完成的批次：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('queue:prune-batches --hours=48')->daily();

有時，您的 `jobs_batches` 資料表可能會累積那些從未成功完成的批次記錄，例如任務失敗且從未成功重試的批次。您可以使用 `unfinished` 選項指示 `queue:prune-batches` 命令清理這些未完成的批次記錄：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('queue:prune-batches --hours=48 --unfinished=72')->daily();

同樣地，您的 `jobs_batches` 資料表也可能累積已取消批次的記錄。您可以使用 `cancelled` 選項指示 `queue:prune-batches` 命令清理這些已取消的批次記錄：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('queue:prune-batches --hours=48 --cancelled=72')->daily();

<a name="storing-batches-in-dynamodb"></a>
### 在 DynamoDB 中儲存批次

Laravel 也支援將批次的中繼資訊儲存到 [DynamoDB](https://aws.amazon.com/dynamodb)，而非關聯式資料庫。然而，您需要手動建立一個 DynamoDB 資料表來儲存所有批次記錄。

通常，此資料表應命名為 `job_batches`，但您應該根據應用程式的 `queue` 設定檔中 `queue.batching.table` 設定值來命名資料表。

<a name="dynamodb-batch-table-configuration"></a>
#### DynamoDB 批次資料表配置

`job_batches` 資料表應包含一個名為 `application` 的字串主分割區索引鍵 (primary partition key) 和一個名為 `id` 的字串主排序索引鍵 (primary sort key)。索引鍵的 `application` 部分將包含您的應用程式名稱，該名稱由應用程式 `app` 設定檔中的 `name` 設定值定義。由於應用程式名稱是 DynamoDB 資料表索引鍵的一部分，因此您可以使用相同的資料表來儲存多個 Laravel 應用程式的任務批次。

此外，如果您想利用 [自動清理批次](#pruning-batches-in-dynamodb) 功能，您可以為資料表定義 `ttl` 屬性。

<a name="dynamodb-configuration"></a>
#### DynamoDB 配置

接下來，安裝 AWS SDK，讓您的 Laravel 應用程式可以與 Amazon DynamoDB 進行通訊：

```shell
composer require aws/aws-sdk-php
```

然後，將 `queue.batching.driver` 配置選項的值設定為 `dynamodb`。此外，您應該在 `batching` 配置陣列中定義 `key`、`secret` 和 `region` 配置選項。這些選項將用於向 AWS 進行身份驗證。當使用 `dynamodb` 驅動程式時，`queue.batching.database` 配置選項是不必要的：

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

當利用 [DynamoDB](https://aws.amazon.com/dynamodb) 儲存任務批次資訊時，用於清理儲存在關聯式資料庫中的批次的典型清理命令將不起作用。相反地，您可以利用 [DynamoDB 原生的 TTL 功能](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html) 自動移除舊批次的記錄。

如果您使用 `ttl` 屬性定義了您的 DynamoDB 資料表，您可以定義配置參數來指示 Laravel 如何清理批次記錄。`queue.batching.ttl_attribute` 配置值定義了儲存 TTL 的屬性名稱，而 `queue.batching.ttl` 配置值定義了批次記錄在資料表最後更新時間後，多少秒可以從 DynamoDB 資料表中移除的秒數：

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
## 佇列閉包

除了將任務類別分派到佇列之外，您也可以分派閉包。這對於需要在目前請求週期之外執行的快速、簡單任務非常有用。將閉包分派到佇列時，該閉包的程式碼內容會以加密方式簽署，以確保其在傳輸中無法被修改：

    $podcast = App\Podcast::find(1);

    dispatch(function () use ($podcast) {
        $podcast->publish();
    });

使用 `catch` 方法，您可以提供一個閉包，該閉包應在佇列閉包耗盡所有 [設定的重試次數](#max-job-attempts-and-timeout) 後未能成功完成時執行：

    use Throwable;

    dispatch(function () use ($podcast) {
        $podcast->publish();
    })->catch(function (Throwable $e) {
        // This job has failed...
    });

> [!WARNING]  
> 由於 `catch` 回呼被 Laravel 佇列序列化並在稍後執行，因此您不應在 `catch` 回呼中使用 `$this` 變數。

<a name="running-the-queue-worker"></a>
## 執行佇列處理器

<a name="the-queue-work-command"></a>
### `queue:work` 命令

Laravel 包含一個 Artisan 命令，它會啟動佇列處理器並處理被推送到佇列中的新任務。您可以使用 `queue:work` Artisan 命令來執行處理器。請注意，一旦 `queue:work` 命令啟動，它將會持續執行，直到您手動停止它或關閉終端機：

```shell
php artisan queue:work
```

> [!NOTE]  
> 為了讓 `queue:work` 程序永久在背景執行，您應該使用像 [Supervisor](#supervisor-configuration) 這樣的程序監控器，以確保佇列處理器不會停止執行。

如果您希望處理過的任務 ID 包含在命令的輸出中，可以在呼叫 `queue:work` 命令時加上 `-v` 旗標：

```shell
php artisan queue:work -v
```

請記住，佇列處理器是長時間執行的程序，並將啟動的應用程式狀態儲存在記憶體中。因此，一旦啟動，它們將不會注意到程式碼庫中的變更。所以，在您的部署過程中，請務必[重新啟動佇列處理器](#queue-workers-and-deployment)。此外，請記住，應用程式建立或修改的任何靜態狀態都不會在任務之間自動重置。

或者，您可以執行 `queue:listen` 命令。使用 `queue:listen` 命令時，當您想要重新載入更新後的程式碼或重置應用程式狀態時，無需手動重新啟動處理器；然而，此命令的效率遠低於 `queue:work` 命令：

```shell
php artisan queue:listen
```

<a name="running-multiple-queue-workers"></a>
#### 執行多個佇列處理器

若要為單一佇列分配多個處理器並同時處理任務，您只需啟動多個 `queue:work` 程序即可。這可以在本機上透過終端機的多個分頁完成，或者在生產環境中使用您的程序管理器的配置設定來完成。[使用 Supervisor 時](#supervisor-configuration)，您可以使用 `numprocs` 配置值。

<a name="specifying-the-connection-queue"></a>
#### 指定連線與佇列

您也可以指定處理器應使用的佇列連線。傳遞給 `work` 命令的連線名稱應與您的 `config/queue.php` 配置檔中定義的其中一個連線相對應：

```shell
php artisan queue:work redis
```

預設情況下，`queue:work` 命令只處理指定連線上的預設佇列中的任務。然而，您可以透過只處理指定連線上的特定佇列來進一步自訂佇列處理器。例如，如果您的所有電子郵件都在 `redis` 佇列連線上的 `emails` 佇列中處理，您可以發出以下命令來啟動一個只處理該佇列的處理器：

```shell
php artisan queue:work redis --queue=emails
```

<a name="processing-a-specified-number-of-jobs"></a>
#### 處理指定數量的任務

`--once` 選項可用於指示處理器僅處理佇列中的單一任務：

```shell
php artisan queue:work --once
```

`--max-jobs` 選項可用於指示處理器處理指定數量的任務然後退出。此選項結合 [Supervisor](#supervisor-configuration) 使用時可能很有用，這樣您的處理器在處理完指定數量的任務後會自動重新啟動，釋放可能累積的任何記憶體：

```shell
php artisan queue:work --max-jobs=1000
```

<a name="processing-all-queued-jobs-then-exiting"></a>
#### 處理所有佇列任務然後退出

`--stop-when-empty` 選項可用於指示處理器處理所有任務，然後優雅地退出。當您希望在佇列清空後關閉 Docker 容器時，此選項在 Docker 容器中處理 Laravel 佇列時可能很有用：

```shell
php artisan queue:work --stop-when-empty
```

<a name="processing-jobs-for-a-given-number-of-seconds"></a>
#### 處理指定秒數的任務

`--max-time` 選項可用於指示處理器處理指定秒數的任務，然後退出。此選項結合 [Supervisor](#supervisor-configuration) 使用時可能很有用，這樣您的處理器在處理指定時間的任務後會自動重新啟動，釋放可能累積的任何記憶體：

```shell
# Process jobs for one hour and then exit...
php artisan queue:work --max-time=3600
```

<a name="worker-sleep-duration"></a>
#### 處理器休眠時間

當佇列中有任務可用時，處理器將持續處理任務，任務之間沒有延遲。然而，`sleep` 選項決定了當沒有任務可用時，處理器將「休眠」多少秒。當然，在休眠期間，處理器將不會處理任何新任務：

```shell
php artisan queue:work --sleep=3
```

<a name="maintenance-mode-queues"></a>
#### 維護模式與佇列

當您的應用程式處於[維護模式](/docs/{{version}}/configuration#maintenance-mode)時，不會處理任何佇列任務。一旦應用程式解除維護模式，任務將繼續正常處理。

若要強制您的佇列處理器處理任務，即使維護模式已啟用，您可以使用 `--force` 選項：

```shell
php artisan queue:work --force
```

<a name="resource-considerations"></a>
#### 資源考量

Daemon 佇列處理器在處理每個任務之前不會「重啟」框架。因此，您應該在每個任務完成後釋放任何佔用大量資源的物件。例如，如果您正在使用 GD 函式庫進行圖像處理，您應該在處理完圖像後使用 `imagedestroy` 釋放記憶體。

<a name="queue-priorities"></a>
### 佇列優先權

有時您可能希望優先處理您的佇列。例如，在您的 `config/queue.php` 配置檔中，您可以將 `redis` 連線的預設 `queue` 設定為 `low`。然而，您偶爾可能希望將任務推送到 `high` 優先級佇列，如下所示：

    dispatch((new Job)->onQueue('high'));

若要啟動一個處理器，使其確保所有 `high` 佇列的任務在處理 `low` 佇列的任何任務之前都被處理，請將以逗號分隔的佇列名稱列表傳遞給 `work` 命令：

```shell
php artisan queue:work --queue=high,low
```

<a name="queue-workers-and-deployment"></a>
### 佇列處理器與部署

由於佇列處理器是長時間執行的程序，它們在未重新啟動的情況下不會注意到程式碼的變更。因此，部署使用佇列處理器的應用程式最簡單的方法是在部署過程中重新啟動處理器。您可以透過發出 `queue:restart` 命令來優雅地重新啟動所有處理器：

```shell
php artisan queue:restart
```

此命令將指示所有佇列處理器在完成處理當前任務後優雅地退出，以確保沒有現有任務遺失。由於佇列處理器在執行 `queue:restart` 命令時會退出，您應該運行一個程序管理器，例如 [Supervisor](#supervisor-configuration)，以自動重新啟動佇列處理器。

> [!NOTE]  
> 佇列使用[快取](/docs/{{version}}/cache)來儲存重新啟動訊號，因此在使用此功能之前，您應該驗證您的應用程式是否已正確配置快取驅動程式。

<a name="job-expirations-and-timeouts"></a>
### 任務過期與逾時

<a name="job-expiration"></a>
#### 任務過期

在您的 `config/queue.php` 設定檔中，每個佇列連線都會定義一個 `retry_after` 選項。此選項指定佇列連線在重試一個正在處理中的 job 之前應等待多少秒。例如，如果 `retry_after` 的值設定為 `90`，那麼如果該 job 在沒有被釋放或刪除的情況下已經處理了 90 秒，它將會被釋放回佇列。通常，您應該將 `retry_after` 的值設定為您的 job 合理完成處理所需的最大秒數。

> [!WARNING]  
> 唯一不包含 `retry_after` 值的佇列連線是 Amazon SQS。SQS 將根據 [預設可見性逾時](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html) 來重試 job，此逾時由 AWS console 管理。

<a name="worker-timeouts"></a>
#### 處理器逾時

`queue:work` Artisan 命令公開了一個 `--timeout` 選項。預設情況下，`--timeout` 的值為 60 秒。如果一個 job 的處理時間超過了逾時值所指定的秒數，處理該 job 的處理器將會因錯誤而終止。通常，處理器會由您伺服器上設定的 [程序管理器](#supervisor-configuration) 自動重新啟動：

```shell
php artisan queue:work --timeout=60
```

`retry_after` 設定選項和 `--timeout` CLI 選項是不同的，但它們協同工作以確保 job 不會丟失，並且 job 只會被成功處理一次。

> [!WARNING]  
> `--timeout` 值應始終比您的 `retry_after` 設定值至少短幾秒。這將確保處理凍結 job 的處理器在 job 被重試之前總是會終止。如果您的 `--timeout` 選項長於您的 `retry_after` 設定值，您的 job 可能會被處理兩次。

<a name="supervisor-configuration"></a>
## Supervisor 配置

在生產環境中，您需要一種方法來保持 `queue:work` 程序持續運行。`queue:work` 程序可能會因多種原因停止運行，例如超出處理器逾時或執行 `queue:restart` 命令。

為此，您需要配置一個程序監視器，它能偵測到您的 `queue:work` 程序何時終止並自動重新啟動它們。此外，程序監視器可以讓您指定要同時運行多少個 `queue:work` 程序。Supervisor 是一個在 Linux 環境中常用的程序監視器，我們將在以下文件中討論如何配置它。

<a name="installing-supervisor"></a>
#### 安裝 Supervisor

Supervisor 是一個適用於 Linux 作業系統的程序監視器，如果您的 `queue:work` 程序失敗，它將自動重新啟動它們。要在 Ubuntu 上安裝 Supervisor，您可以使用以下命令：

```shell
sudo apt-get install supervisor
```

> [!NOTE]  
> 如果自己配置和管理 Supervisor 聽起來很麻煩，可以考慮使用 [Laravel Forge](https://forge.laravel.com)，它將自動為您的生產環境 Laravel 專案安裝並配置 Supervisor。

<a name="configuring-supervisor"></a>
#### 配置 Supervisor

Supervisor 配置文件通常儲存在 `/etc/supervisor/conf.d` 目錄中。在此目錄中，您可以建立任意數量的配置文件，以指示 Supervisor 如何監視您的程序。例如，讓我們建立一個 `laravel-worker.conf` 文件，它會啟動並監視 `queue:work` 程序：

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

在此範例中，`numprocs` 指令將指示 Supervisor 運行八個 `queue:work` 程序，並監視所有這些程序，如果它們失敗會自動重新啟動。您應該更改配置的 `command` 指令，以反映您所需的佇列連線和處理器選項。

> [!WARNING]  
> 您應該確保 `stopwaitsecs` 的值大於您運行時間最長的任務所消耗的秒數。否則，Supervisor 可能會在任務完成處理之前終止該任務。

<a name="starting-supervisor"></a>
#### 啟動 Supervisor

配置文件建立後，您可以更新 Supervisor 配置並使用以下命令啟動程序：

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start "laravel-worker:*"
```

有關 Supervisor 的更多資訊，請參閱 [Supervisor 文件](http://supervisord.org/index.html)。

<a name="dealing-with-failed-jobs"></a>
## 處理失敗任務

有時你的佇列任務會失敗。別擔心，事情不總是按計畫進行！Laravel 提供一種方便的方法來[指定任務應嘗試的最大次數](#max-job-attempts-and-timeout)。在非同步任務超過此嘗試次數後，它將被插入到 `failed_jobs` 資料庫資料表。[同步分派的任務](/docs/{{version}}/queues#synchronous-dispatching)如果失敗，則不會儲存到此資料表，其例外會立即由應用程式處理。

建立 `failed_jobs` 資料表的遷移通常已存在於新的 Laravel 應用程式中。然而，如果你的應用程式不包含此資料表的遷移，你可以使用 `make:queue-failed-table` 命令來建立該遷移：

```shell
php artisan make:queue-failed-table

php artisan migrate
```

執行 [佇列處理器](#running-the-queue-worker) 程序時，你可以使用 `queue:work` 命令上的 `--tries` 選項來指定任務的最大嘗試次數。如果你不為 `--tries` 選項指定值，任務只會嘗試一次，或依任務類別的 `$tries` 屬性所指定的次數進行嘗試：

```shell
php artisan queue:work redis --tries=3
```

使用 `--backoff` 選項，你可以指定 Laravel 在重試遇到例外的任務之前應等待的秒數。預設情況下，任務會立即被釋放回佇列，以便再次嘗試：

```shell
php artisan queue:work redis --tries=3 --backoff=3
```

如果你希望以每個任務為基礎，設定 Laravel 在重試遇到例外的任務之前應等待的秒數，你可以在任務類別上定義一個 `backoff` 屬性來實現：

    /**
     * The number of seconds to wait before retrying the job.
     *
     * @var int
     */
    public $backoff = 3;

如果你需要更複雜的邏輯來決定任務的回溯時間，你可以在任務類別上定義一個 `backoff` 方法：

    /**
    * Calculate the number of seconds to wait before retrying the job.
    */
    public function backoff(): int
    {
        return 3;
    }

你可以透過從 `backoff` 方法回傳一個回溯值陣列，輕鬆地配置「指數型」回溯。在這個範例中，如果還有剩餘嘗試次數，第一次重試的延遲時間將是 1 秒，第二次重試是 5 秒，第三次重試是 10 秒，之後的每次重試都是 10 秒：

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
### 清理失敗任務後的善後工作

當某個任務失敗時，你可能會想向使用者傳送通知，或是還原任務部分完成的任何操作。為此，你可以在任務類別上定義一個 `failed` 方法。導致任務失敗的 `Throwable` 實例將會傳遞給 `failed` 方法：

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
> 在呼叫 `failed` 方法之前，會實例化任務的新實例；因此，在 `handle` 方法中可能發生的任何類別屬性修改將會遺失。


<a name="retrying-failed-jobs"></a>
### 重試失敗任務

要檢視已插入 `failed_jobs` 資料庫資料表中的所有失敗任務，你可以使用 `queue:failed` Artisan 命令：

```shell
php artisan queue:failed
```

`queue:failed` 命令將列出任務 ID、連線、佇列、失敗時間以及關於任務的其他資訊。任務 ID 可用於重試失敗的任務。例如，要重試 ID 為 `ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece` 的失敗任務，請發出以下命令：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece
```

如有必要，你可以向命令傳遞多個 ID：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece 91401d2c-0784-4f43-824c-34f94a33c24d
```

你也可以重試特定佇列的所有失敗任務：

```shell
php artisan queue:retry --queue=name
```

要重試所有失敗任務，執行 `queue:retry` 命令並傳遞 `all` 作為 ID：

```shell
php artisan queue:retry all
```

如果你希望刪除失敗任務，你可以使用 `queue:forget` 命令：

```shell
php artisan queue:forget 91401d2c-0784-4f43-824c-34f94a33c24d
```

> [!NOTE]  
> 當使用 [Horizon](/docs/{{version}}/horizon) 時，你應該使用 `horizon:forget` 命令來刪除失敗任務，而不是 `queue:forget` 命令。

要從 `failed_jobs` 資料表刪除所有失敗任務，你可以使用 `queue:flush` 命令：

```shell
php artisan queue:flush
```


<a name="ignoring-missing-models"></a>
### 忽略缺失的模型

當將一個 Eloquent 模型注入到任務中時，該模型會在被放入佇列之前自動序列化，並在任務處理時從資料庫重新取回。然而，如果該模型在任務等待處理器處理時已被刪除，你的任務可能會因為 `ModelNotFoundException` 而失敗。

為了方便起見，你可以透過將任務的 `deleteWhenMissingModels` 屬性設定為 `true`，來自動刪除模型缺失的任務。當此屬性設定為 `true` 時，Laravel 將會靜默地丟棄該任務，而不會拋出例外：

    /**
     * Delete the job if its models no longer exist.
     *
     * @var bool
     */
    public $deleteWhenMissingModels = true;


<a name="pruning-failed-jobs"></a>
### 清理失敗任務

你可以透過呼叫 `queue:prune-failed` Artisan 命令來清理應用程式 `failed_jobs` 資料表中的記錄：

```shell
php artisan queue:prune-failed
```

預設情況下，所有超過 24 小時的失敗任務記錄將會被清理。如果你為命令提供 `--hours` 選項，則只會保留在過去 N 小時內插入的失敗任務記錄。例如，以下命令將刪除所有超過 48 小時前插入的失敗任務記錄：

```shell
php artisan queue:prune-failed --hours=48
```

<a name="storing-failed-jobs-in-dynamodb"></a>
### 在 DynamoDB 中儲存失敗任務

Laravel 也支援將您的失敗任務紀錄儲存在 [DynamoDB](https://aws.amazon.com/dynamodb) 中，而不是關聯式資料庫表格。但是，您必須手動建立一個 DynamoDB 表格來儲存所有失敗任務紀錄。通常，此表格應命名為 `failed_jobs`，但您應根據應用程式 `queue` 設定檔中 `queue.failed.table` 設定值來命名表格。

`failed_jobs` 表格應具有一個名為 `application` 的字串主分割區鍵 (primary partition key) 和一個名為 `uuid` 的字串主排序鍵 (primary sort key)。鍵的 `application` 部分將包含應用程式 `app` 設定檔中 `name` 設定值所定義的應用程式名稱。由於應用程式名稱是 DynamoDB 表格鍵的一部分，因此您可以使用相同的表格來儲存多個 Laravel 應用程式的失敗任務。

此外，請確保您已安裝 AWS SDK，以便您的 Laravel 應用程式可以與 Amazon DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

接下來，將 `queue.failed.driver` 設定選項的值設定為 `dynamodb`。此外，您應該在失敗任務設定陣列中定義 `key`、`secret` 和 `region` 設定選項。這些選項將用於向 AWS 進行認證。當使用 `dynamodb` 驅動程式時，`queue.failed.database` 設定選項是不必要的：

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

您可以指示 Laravel 捨棄失敗任務而不儲存它們，方法是將 `queue.failed.driver` 設定選項的值設定為 `null`。通常，這可以透過 `QUEUE_FAILED_DRIVER` 環境變數來實現：

```ini
QUEUE_FAILED_DRIVER=null
```


<a name="failed-job-events"></a>
### 失敗任務事件

如果您想註冊一個在任務失敗時被呼叫的事件監聽器，可以使用 `Queue` Facade 的 `failing` 方法。例如，我們可以從 Laravel 附帶的 `AppServiceProvider` 的 `boot` 方法中將閉包附加到此事件：

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
## 從佇列中清除任務

> [!NOTE]  
> When using [Horizon](/docs/{{version}}/horizon), you should use the `horizon:clear` command to clear jobs from the queue instead of the `queue:clear` command.

如果您想從預設連線的預設佇列中刪除所有任務，您可以使用 `queue:clear` Artisan 命令來完成：

```shell
php artisan queue:clear
```

您也可以提供 `connection` 引數和 `queue` 選項，從特定連線和佇列中刪除任務：

```shell
php artisan queue:clear redis --queue=emails
```

> [!WARNING]  
> 從佇列中清除任務僅適用於 SQS、Redis 和 database 佇列驅動程式。此外，SQS 訊息刪除程序最多需要 60 秒，因此在您清除佇列後 60 秒內發送到 SQS 佇列的任務也可能被刪除。

<a name="monitoring-your-queues"></a>
## 監控佇列

如果您的佇列接收到突如其來的任務量，它可能會不堪負荷，導致任務完成的漫長等待時間。如果您願意，Laravel 可以在您的佇列任務數量超過指定閾值時通知您。

首先，您應該將 `queue:monitor` 命令排程為 [每分鐘執行一次](/docs/{{version}}/scheduling)。此命令接受您希望監控的佇列名稱以及您期望的任務計數閾值：

```shell
php artisan queue:monitor redis:default,redis:deployments --max=100
```

單獨排程此命令並不足以觸發通知，提醒您佇列已不堪負荷。當命令遇到任務數量超過您閾值的佇列時，將會分派一個 `Illuminate\Queue\Events\QueueBusy` 事件。您可以在應用程式的 `AppServiceProvider` 中監聽此事件，以便向您或您的開發團隊發送通知：

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

當您測試分派任務的程式碼時，您可能希望指示 Laravel 不要實際執行任務本身，因為任務的程式碼可以獨立於分派程式碼直接進行測試。當然，要測試任務本身，您可以實例化一個任務實例，並在您的測試中直接呼叫 `handle` 方法。

您可以使用 `Queue` Facade 的 `fake` 方法來阻止佇列任務被實際推送到佇列。呼叫 `Queue` Facade 的 `fake` 方法後，您可以斷言應用程式嘗試將任務推送到佇列：

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

您可以將一個閉包傳遞給 `assertPushed` 或 `assertNotPushed` 方法，以斷言一個通過了指定「驗證測試」的任務已被推送到佇列。如果至少有一個任務通過了給定的驗證測試，那麼斷言將會成功：

    Queue::assertPushed(function (ShipOrder $job) use ($order) {
        return $job->order->id === $order->id;
    });


<a name="faking-a-subset-of-jobs"></a>
### 模擬部分任務

如果您只需要模擬特定的任務，同時允許其他任務正常執行，您可以將需要模擬的任務類別名稱傳遞給 `fake` 方法：

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

您可以使用 `except` 方法來模擬所有任務，除了指定的任務集：

    Queue::fake()->except([
        ShipOrder::class,
    ]);


<a name="testing-job-chains"></a>
### 測試任務鏈結

要測試任務鏈結，您需要利用 `Bus` Facade 的模擬功能。`Bus` Facade 的 `assertChained` 方法可用於斷言任務鏈結已分派。`assertChained` 方法接受一個任務鏈結陣列作為其第一個參數：

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

如您在上面的範例中所見，任務鏈結陣列可以是任務的類別名稱陣列。然而，您也可以提供一個實際的任務實例陣列。在這種情況下，Laravel 將確保任務實例屬於相同的類別，並具有與您的應用程式分派的鏈結任務相同的屬性值：

    Bus::assertChained([
        new ShipOrder,
        new RecordShipment,
        new UpdateInventory,
    ]);

您可以使用 `assertDispatchedWithoutChain` 方法來斷言任務被推送到佇列時沒有任務鏈結：

    Bus::assertDispatchedWithoutChain(ShipOrder::class);


<a name="testing-chain-modifications"></a>
#### 測試鏈結修改

如果鏈結任務[前置或附加任務到現有鏈結](#adding-jobs-to-the-chain)，您可以使用任務的 `assertHasChain` 方法來斷言任務具有預期的剩餘任務鏈結：

```php
$job = new ProcessPodcast;

$job->handle();

$job->assertHasChain([
    new TranscribePodcast,
    new OptimizePodcast,
    new ReleasePodcast,
]);
```

`assertDoesntHaveChain` 方法可用於斷言任務的剩餘鏈結為空：

```php
$job->assertDoesntHaveChain();
```


<a name="testing-chained-batches"></a>
#### 測試鏈結批次

如果您的任務鏈結[包含任務批次](#chains-and-batches)，您可以在您的鏈結斷言中插入一個 `Bus::chainedBatch` 定義，以斷言鏈結批次符合您的預期：

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
### 測試任務批次

`Bus` Facade 的 `assertBatched` 方法可用於斷言任務批次已分派。傳遞給 `assertBatched` 方法的閉包會收到一個 `Illuminate\Bus\PendingBatch` 實例，該實例可用於檢查批次中的任務：

    use Illuminate\Bus\PendingBatch;
    use Illuminate\Support\Facades\Bus;

    Bus::fake();

    // ...

    Bus::assertBatched(function (PendingBatch $batch) {
        return $batch->name == 'import-csv' &&
               $batch->jobs->count() === 10;
    });

您可以使用 `assertBatchCount` 方法來斷言分派了指定數量的批次：

    Bus::assertBatchCount(3);

您可以使用 `assertNothingBatched` 來斷言沒有分派任何批次：

    Bus::assertNothingBatched();


<a name="testing-job-batch-interaction"></a>
#### 測試任務 / 批次互動

此外，您可能偶爾需要測試個別任務與其底層批次的互動。例如，您可能需要測試任務是否取消了其批次的後續處理。為此，您需要通過 `withFakeBatch` 方法為任務指派一個模擬批次。`withFakeBatch` 方法返回一個包含任務實例和模擬批次的元組：

    [$job, $batch] = (new ShipOrder)->withFakeBatch();

    $job->handle();

    $this->assertTrue($batch->cancelled());
    $this->assertEmpty($batch->added);

<a name="testing-job-queue-interactions"></a>
### 測試任務 / 佇列互動

有時，您可能需要測試一個佇列中的任務[將自身釋出回佇列](#manually-releasing-a-job)。或者，您可能需要測試該任務是否已刪除自身。您可以透過實例化任務並呼叫 `withFakeQueueInteractions` 方法來測試這些佇列互動。

一旦任務的佇列互動被模擬後，您可以呼叫該任務的 `handle` 方法。呼叫任務後，`assertReleased`、`assertDeleted`、`assertNotDeleted`、`assertFailed`、`assertFailedWith` 和 `assertNotFailed` 方法可用於針對任務的佇列互動進行斷言：

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

您可以使用 `Queue` [外觀](/docs/{{version}}/facades) 上的 `before` 和 `after` 方法，指定在處理佇列任務之前或之後執行的回呼函式。這些回呼函式是執行額外日誌記錄或為儀表板增加統計數據的絕佳機會。通常，您應該從 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫這些方法。例如，我們可以使用 Laravel 內建的 `AppServiceProvider`：

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

您可以使用 `Queue` [外觀](/docs/{{version}}/facades) 上的 `looping` 方法，指定在處理器嘗試從佇列中取出任務之前執行的回呼函式。例如，您可能會註冊一個閉包函式，用於回溯先前失敗任務留下的任何開啟交易：

    use Illuminate\Support\Facades\DB;
    use Illuminate\Support\Facades\Queue;

    Queue::looping(function () {
        while (DB::transactionLevel() > 0) {
            DB::rollBack();
        }
    });