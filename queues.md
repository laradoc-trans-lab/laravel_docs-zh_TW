# 佇列

- [簡介](#introduction)
    - [連線與佇列](#connections-vs-queues)
    - [驅動程式說明與先決條件](#driver-prerequisites)
- [建立工作](#creating-jobs)
    - [產生工作類別](#generating-job-classes)
    - [類別結構](#class-structure)
    - [唯一性工作](#unique-jobs)
    - [加密工作](#encrypted-jobs)
- [工作中介層](#job-middleware)
    - [速率限制](#rate-limiting)
    - [防止工作重疊](#preventing-job-overlaps)
    - [節流例外](#throttling-exceptions)
    - [跳過工作](#skipping-jobs)
- [分派工作](#dispatching-jobs)
    - [延遲分派](#delayed-dispatching)
    - [同步分派](#synchronous-dispatching)
    - [工作與資料庫交易](#jobs-and-database-transactions)
    - [工作鏈](#job-chaining)
    - [自訂佇列與連線](#customizing-the-queue-and-connection)
    - [指定工作最大嘗試次數/逾時值](#max-job-attempts-and-timeout)
    - [SQS FIFO 與公平佇列](#sqs-fifo-and-fair-queues)
    - [佇列容錯移轉](#queue-failover)
    - [錯誤處理](#error-handling)
- [工作批次處理](#job-batching)
    - [定義可批次處理的工作](#defining-batchable-jobs)
    - [分派批次](#dispatching-batches)
    - [鏈與批次](#chains-and-batches)
    - [將工作新增至批次](#adding-jobs-to-batches)
    - [檢查批次](#inspecting-batches)
    - [取消批次](#cancelling-batches)
    - [批次失敗](#batch-failures)
    - [修剪批次](#pruning-batches)
    - [在 DynamoDB 中儲存批次](#storing-batches-in-dynamodb)
- [佇列閉包](#queueing-closures)
- [執行佇列工作者](#running-the-queue-worker)
    - [`queue:work` 命令](#the-queue-work-command)
    - [佇列優先級](#queue-priorities)
    - [佇列工作者與部署](#queue-workers-and-deployment)
    - [工作到期與逾時](#job-expirations-and-timeouts)
    - [暫停與恢復佇列工作者](#pausing-and-resuming-queue-workers)
- [Supervisor 配置](#supervisor-configuration)
- [處理失敗的工作](#dealing-with-failed-jobs)
    - [清理失敗的工作](#cleaning-up-after-failed-jobs)
    - [重試失敗的工作](#retrying-failed-jobs)
    - [忽略遺失的模型](#ignoring-missing-models)
    - [修剪失敗的工作](#pruning-failed-jobs)
    - [在 DynamoDB 中儲存失敗的工作](#storing-failed-jobs-in-dynamodb)
    - [停用失敗的工作儲存](#disabling-failed-job-storage)
    - [失敗的工作事件](#failed-job-events)
- [從佇列清除工作](#clearing-jobs-from-queues)
- [監控你的佇列](#monitoring-your-queues)
- [測試](#testing)
    - [偽造部分工作](#faking-a-subset-of-jobs)
    - [測試工作鏈](#testing-job-chains)
    - [測試工作批次](#testing-job-batches)
    - [測試工作/佇列互動](#testing-job-queue-interactions)
- [工作事件](#job-events)

<a name="introduction"></a>
## 簡介

在建構您的網路應用程式時，您可能會遇到一些在一般網路請求期間執行時間過長的工作，例如解析並儲存上傳的 CSV 檔案。幸運的是，Laravel 讓您可以輕鬆建立可在背景處理的佇列工作。藉由將耗時的工作移至佇列，您的應用程式可以極快的速度回應網路請求，並為客戶提供更好的使用者體驗。

Laravel 佇列在多種不同的佇列後端提供統一的佇列 API，例如 [Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io)，甚至是關聯式資料庫。

Laravel 的佇列設定選項儲存在您應用程式的 `config/queue.php` 設定檔中。在此檔案中，您會找到框架中包含的每個佇列驅動程式的連線設定，包括 database、[Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 和 [Beanstalkd](https://beanstalkd.github.io/) 驅動程式，以及一個會立即執行工作的同步驅動程式 (用於開發或測試期間)。還包含一個 `null` 佇列驅動程式，它會捨棄佇列中的工作。

> [!NOTE]
> Laravel Horizon 是一個美觀的儀表板和設定系統，專為您的 Redis 驅動佇列而設計。請參閱完整的 [Horizon 文件](/docs/{{version}}/horizon) 以獲取更多資訊。

<a name="connections-vs-queues"></a>
### 連線與佇列

在開始使用 Laravel 佇列之前，了解「連線 (connections)」和「佇列 (queues)」之間的區別很重要。在您的 `config/queue.php` 設定檔中，有一個 `connections` 設定陣列。此選項定義了與後端佇列服務 (如 Amazon SQS、Beanstalk 或 Redis) 的連線。然而，任何一個佇列連線都可以有多個「佇列 (queues)」，這些佇列可以被視為不同的堆疊或一堆佇列工作。

請注意，`queue` 設定檔中的每個連線設定範例都包含一個 `queue` 屬性。這是當工作被傳送到給定連線時，工作將被分派到的預設佇列。換句話說，如果您在分派工作時沒有明確定義它應該被分派到哪個佇列，那麼該工作將被放置在連線設定的 `queue` 屬性中定義的佇列上：

```php
use App\Jobs\ProcessPodcast;

// This job is sent to the default connection's default queue...
ProcessPodcast::dispatch();

// This job is sent to the default connection's "emails" queue...
ProcessPodcast::dispatch()->onQueue('emails');
```

有些應用程式可能不需要將工作推送到多個佇列，而是偏好只有一個簡單的佇列。然而，將工作推送到多個佇列對於希望優先處理或區分工作處理方式的應用程式來說特別有用，因為 Laravel 佇列工作者允許您指定它應該依優先級處理哪些佇列。例如，如果您將工作推送到 `high` 佇列，您可以運行一個工作者，賦予它們更高的處理優先級：

```shell
php artisan queue:work --queue=high,default
```

<a name="driver-prerequisites"></a>
### 驅動程式說明與先決條件

<a name="database"></a>
#### 資料庫

為了使用 `database` 佇列驅動程式，您將需要一個資料庫表格來儲存工作。通常，這包含在 Laravel 的預設 `0001_01_01_000002_create_jobs_table.php` [資料庫遷移](/docs/{{version}}/migrations) 中；然而，如果您的應用程式不包含此遷移，您可以使用 `make:queue-table` Artisan 命令來建立它：

```shell
php artisan make:queue-table

php artisan migrate
```

<a name="redis"></a>
#### Redis

為了使用 `redis` 佇列驅動程式，您應該在您的 `config/database.php` 設定檔中配置一個 Redis 資料庫連線。

> [!WARNING]
> `redis` 佇列驅動程式不支援 `serializer` 和 `compression` Redis 選項。

<a name="redis-cluster"></a>
##### Redis Cluster

如果您的 Redis 佇列連線使用 [Redis Cluster](https://redis.io/docs/latest/operate/rs/databases/durability-ha/clustering)，您的佇列名稱必須包含 [key hash tag](https://redis.io/docs/latest/develop/using-commands/keyspace/#hashtags)。這是為了確保給定佇列的所有 Redis 鍵都放置在同一個雜湊槽中：

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
##### 阻塞

當使用 Redis 佇列時，您可以使用 `block_for` 設定選項來指定驅動程式在遍歷工作者迴圈並重新輪詢 Redis 資料庫之前，應該等待工作變為可用狀態的時間長度。

根據您的佇列負載調整此值，可以比持續輪詢 Redis 資料庫以獲取新工作更有效率。例如，您可以將值設定為 `5`，表示驅動程式應在等待工作變為可用狀態時阻塞五秒：

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
> 將 `block_for` 設定為 `0` 將導致佇列工作者無限期阻塞，直到有工作可用為止。這也會阻止處理諸如 `SIGTERM` 等訊號，直到下一個工作被處理為止。

<a name="other-driver-prerequisites"></a>
#### 其他驅動程式先決條件

以下列出的佇列驅動程式需要以下依賴項。這些依賴項可以透過 Composer 套件管理器安裝：

<div class="content-list" markdown="1">

- Amazon SQS: `aws/aws-sdk-php ~3.0`
- Beanstalkd: `pda/pheanstalk ~5.0`
- Redis: `predis/predis ~2.0` or phpredis PHP extension
- [MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/queues/): `mongodb/laravel-mongodb`

</div>

<a name="creating-jobs"></a>
## 建立工作

<a name="generating-job-classes"></a>
### 產生工作類別

預設情況下，您的應用程式中所有可佇列的工作都儲存在 `app/Jobs` 目錄中。如果 `app/Jobs` 目錄不存在，當您執行 `make:job` Artisan 命令時將會建立它：

```shell
php artisan make:job ProcessPodcast
```

產生的類別將實作 `Illuminate\Contracts\Queue\ShouldQueue` 介面，表示該工作應被推送到佇列中以非同步執行。

> [!NOTE]
> 工作 Stub 可以透過 [Stub 發布](/docs/{{version}}/artisan#stub-customization)進行自訂。

<a name="class-structure"></a>
### 類別結構

工作類別非常簡單，通常只包含一個 `handle` 方法，該方法在工作被佇列處理時被呼叫。讓我們從一個範例工作類別開始。在這個範例中，我們假設我們管理一個 Podcast 發布服務，需要在發布之前處理上傳的 Podcast 檔案：

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

在這個範例中，請注意我們能夠直接將一個 [Eloquent model](/docs/{{version}}/eloquent) 傳遞給佇列工作的建構子。由於工作使用了 `Queueable` Trait，Eloquent model 及其載入的關聯會在工作處理時優雅地序列化與反序列化。

如果您的佇列工作在其建構子中接受一個 Eloquent model，那麼只有該模型的識別碼會被序列化到佇列中。當工作實際處理時，佇列系統將自動從資料庫中重新檢索完整的模型實例及其載入的關聯。這種模型序列化方法可以讓傳送給佇列驅動程式的工作酬載小得多。

<a name="handle-method-dependency-injection"></a>
#### `handle` 方法的依賴注入

`handle` 方法在工作被佇列處理時被呼叫。請注意，我們能夠在工作的 `handle` 方法上型別提示依賴項。Laravel [服務容器](/docs/{{version}}/container)會自動注入這些依賴項。

如果您希望完全控制容器如何將依賴項注入 `handle` 方法，您可以使用容器的 `bindMethod` 方法。`bindMethod` 方法接受一個回呼，該回呼會接收工作與容器。在回呼中，您可以自由地以您希望的方式呼叫 `handle` 方法。通常，您應該從您的 `App\Providers\AppServiceProvider` [服務提供者](/docs/{{version}}/providers)的 `boot` 方法中呼叫此方法：

```php
use App\Jobs\ProcessPodcast;
use App\Services\AudioProcessor;
use Illuminate\Contracts\Foundation\Application;

$this->app->bindMethod([ProcessPodcast::class, 'handle'], function (ProcessPodcast $job, Application $app) {
    return $job->handle($app->make(AudioProcessor::class));
});
```

> [!WARNING]
> 二進位資料，例如原始圖片內容，在傳遞給佇列工作之前，應透過 `base64_encode` 函數處理。否則，工作在放入佇列時可能無法正確地序列化為 JSON。

<a name="handling-relationships"></a>
#### 佇列關聯

由於所有已載入的 Eloquent Model 關聯在工作佇列化時也會被序列化，序列化的工作字串有時會變得非常大。此外，當工作被反序列化且模型關聯從資料庫中重新檢索時，它們將會被完全檢索。在模型於工作佇列處理期間被序列化之前所套用的任何先前關聯約束，將在工作反序列化時不被套用。因此，如果您希望處理給定關聯的子集，則應在佇列工作中重新約束該關聯。

或者，為了防止關聯被序列化，您可以在設定屬性值時對模型呼叫 `withoutRelations` 方法。此方法將返回一個不帶載入關聯的模型實例：

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

如果您使用 [PHP 建構子屬性提升](https://www.php.net/manual/en/language.oop5.decon.php#language.oop5.decon.constructor.promotion)，並希望表明一個 Eloquent Model 不應序列化其關聯，您可以使用 `WithoutRelations` 屬性：

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

為方便起見，如果您希望序列化所有不帶關聯的模型，您可以將 `WithoutRelations` 屬性套用到整個類別，而不是將屬性套用到每個模型：

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

如果一個工作接收的是 Eloquent Model 的集合或陣列而不是單一模型，那麼該集合中的模型將不會在工作被反序列化和執行時恢復其關聯。這是為了防止處理大量模型的工作造成過度的資源使用。

<a name="unique-jobs"></a>
### 唯一性工作

> [!WARNING]
> 唯一性工作需要支援[鎖定](/docs/{{version}}/cache#atomic-locks)的快取驅動程式。目前，`memcached`、`redis`、`dynamodb`、`database`、`file` 和 `array` 快取驅動程式支援原子鎖定。

> [!WARNING]
> 唯一性工作限制不適用於批次內的工作。

有時候，您可能會希望確保在任何時間點，佇列中只有一個特定工作的實例。您可以透過在工作類別上實作 `ShouldBeUnique` 介面來實現此目的。此介面不要求您在類別上定義任何額外的方法：

```php
<?php

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Queue\ShouldBeUnique;

class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    // ...
}
```

在上面的範例中，`UpdateSearchIndex` 工作是唯一的。因此，如果佇列中已經有另一個該工作的實例且尚未完成處理，則該工作將不會被分派。

在某些情況下，您可能希望定義一個特定的「鍵」來使工作唯一，或者您可能希望指定一個逾時時間，超過該時間後工作就不再保持唯一。為此，您可以在工作類別上定義 `uniqueId` 和 `uniqueFor` 屬性或方法：

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

在上面的範例中，`UpdateSearchIndex` 工作透過產品 ID 保持唯一。因此，任何具有相同產品 ID 的新工作分派都將被忽略，直到現有的工作完成處理。此外，如果現有工作在一小時內未處理完畢，則唯一的鎖定將被釋放，另一個具有相同唯一鍵的工作可以被分派到佇列中。

> [!WARNING]
> 如果您的應用程式從多個網路伺服器或容器分派工作，您應該確保所有伺服器都與同一個中央快取伺服器通訊，以便 Laravel 能夠準確判斷工作是否唯一。

<a name="keeping-jobs-unique-until-processing-begins"></a>
#### 保持工作唯一直到處理開始

預設情況下，唯一性工作會在工作完成處理或所有重試嘗試失敗後「解除鎖定」。然而，在某些情況下，您可能希望您的工作在處理之前立即解除鎖定。為此，您的工作應該實作 `ShouldBeUniqueUntilProcessing` 契約，而不是 `ShouldBeUnique` 契約：

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
#### 唯一性工作鎖定

在幕後，當一個 `ShouldBeUnique` 工作被分派時，Laravel 會嘗試使用 `uniqueId` 鍵來取得[鎖定](/docs/{{version}}/cache#atomic-locks)。如果鎖定已被持有，該工作將不會被分派。當工作完成處理或所有重試嘗試失敗時，此鎖定將被釋放。預設情況下，Laravel 將使用預設的快取驅動程式來取得此鎖定。但是，如果您希望使用另一個驅動程式來取得鎖定，您可以定義一個 `uniqueVia` 方法，該方法會回傳應該使用的快取驅動程式：

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
> 如果您只需要限制工作的併發處理，請改用 [WithoutOverlapping](/docs/{{version}}/queues#preventing-job-overlaps) 工作中介層。

<a name="encrypted-jobs"></a>
### 加密工作

Laravel 允許您透過[加密](/docs/{{version}}/encryption)來確保工作資料的隱私和完整性。首先，只需將 `ShouldBeEncrypted` 介面新增到工作類別中。一旦此介面新增到類別後，Laravel 將在將工作推送到佇列之前自動加密您的工作：

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
## 工作中介層

工作中介層允許您將自訂邏輯包裝在已佇列工作的執行周圍，減少工作本身的樣板程式碼。例如，考慮以下 `handle` 方法，它利用 Laravel 的 Redis 速率限制功能，允許每五秒只處理一個工作：

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

儘管這段程式碼是有效的，但 `handle` 方法的實作會變得雜亂，因為它充斥著 Redis 速率限制邏輯。此外，這種速率限制邏輯必須複製到任何我們想要限制速率的其他工作。與其在 handle 方法中進行速率限制，我們可以定義一個處理速率限制的工作中介層：

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

如您所見，如同 [路由中介層](/docs/{{version}}/middleware)，工作中介層會接收正在處理的工作以及一個應調用以繼續處理工作的 callback。

您可以使用 `make:job-middleware` Artisan 命令產生新的工作中介層類別。建立工作中介層後，可以從工作的 `middleware` 方法中回傳它們，將它們附加到工作中。這個方法不存在於由 `make:job` Artisan 命令生成的工作中，因此您需要手動將其新增到您的工作類別中：

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
> 工作中介層也可以分配給 [可佇列的事件監聽器](/docs/{{version}}/events#queued-event-listeners)、[郵件](/docs/{{version}}/mail#queueing-mail) 和 [通知](/docs/{{version}}/notifications#queueing-notifications)。

<a name="rate-limiting"></a>
### 速率限制

儘管我們剛剛展示了如何編寫自己的速率限制工作中介層，但 Laravel 實際上包含了一個速率限制中介層，您可以用它來限制工作的速率。如同 [路由速率限制器](/docs/{{version}}/routing#defining-rate-limiters) 一樣，工作速率限制器是使用 `RateLimiter` Facade 的 `for` 方法定義的。

例如，您可能希望允許使用者每小時備份一次資料，同時不對付費客戶施加此類限制。為此，您可以在 `AppServiceProvider` 的 `boot` 方法中定義一個 `RateLimiter`：

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

在上面的範例中，我們定義了一個每小時的速率限制；但是，您可以使用 `perMinute` 方法輕鬆定義基於分鐘的速率限制。此外，您可以將任何您希望的值傳遞給速率限制的 `by` 方法；然而，這個值最常用於按客戶區分速率限制：

```php
return Limit::perMinute(50)->by($job->user->id);
```

一旦您定義了速率限制，您可以使用 `Illuminate\Queue\Middleware\RateLimited` 中介層將速率限制器附加到您的工作上。每次工作超出速率限制時，此中介層將根據速率限制的持續時間，以適當的延遲將工作釋放回佇列：

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

將速率受限的工作釋放回佇列仍會增加工作的總 `attempts` 次數。您可能需要相應地調整工作類別中的 `tries` 和 `maxExceptions` 屬性。或者，您可能希望使用 [retryUntil 方法](#time-based-attempts) 來定義工作應不再嘗試的時間長度。

使用 `releaseAfter` 方法，您還可以指定在釋放的工作再次嘗試之前必須經過的秒數：

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

如果您不希望在工作受到速率限制時重試，您可以使用 `dontRelease` 方法：

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
> 如果您正在使用 Redis，您可以使用 `Illuminate\Queue\Middleware\RateLimitedWithRedis` 中介層，它針對 Redis 進行了微調，並且比基本速率限制中介層更高效。

<a name="preventing-job-overlaps"></a>
### 防止工作重疊

Laravel 包含一個 `Illuminate\Queue\Middleware\WithoutOverlapping` 中介層，它允許您根據任意鍵來防止工作重疊。當一個佇列工作正在修改應僅由一個工作一次修改的資源時，這會很有幫助。

例如，假設您有一個佇列工作會更新使用者的信用分數，並且您希望防止相同使用者 ID 的信用分數更新工作重疊。為此，您可以從工作的 `middleware` 方法中回傳 `WithoutOverlapping` 中介層：

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

將重疊的工作釋放回佇列仍會增加工作的總嘗試次數。您可能需要相應地調整工作類別中的 `tries` 和 `maxExceptions` 屬性。例如，預設情況下將 `tries` 屬性保留為 1 將會阻止任何重疊的工作稍後再次重試。

任何相同類型的重疊工作都將被釋放回佇列。您也可以指定必須經過多少秒後，被釋放的工作才會再次嘗試：

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

如果您希望立即刪除任何重疊的工作，使其不再重試，您可以使用 `dontRelease` 方法：

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

`WithoutOverlapping` 中介層由 Laravel 的原子鎖定功能提供支援。有時，您的工作可能會意外失敗或逾時，導致鎖定未被釋放。因此，您可以透過 `expireAfter` 方法明確定義鎖定的過期時間。例如，以下範例將指示 Laravel 在工作開始處理三分鐘後釋放 `WithoutOverlapping` 鎖：

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
> `WithoutOverlapping` 中介層需要支援 [鎖定](/docs/{{version}}/cache#atomic-locks) 的快取驅動程式。目前，`memcached`、`redis`、`dynamodb`、`database`、`file` 和 `array` 快取驅動程式都支援原子鎖定。

<a name="sharing-lock-keys"></a>
#### 在工作類別之間共享鎖定鍵

預設情況下，`WithoutOverlapping` 中介層只會阻止相同類別的工作重疊。因此，儘管兩個不同的工作類別可能使用相同的鎖定鍵，但它們不會被阻止重疊。但是，您可以透過使用 `shared` 方法指示 Laravel 將鍵應用於所有工作類別：

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

Laravel 包含一個 `Illuminate\Queue\Middleware\ThrottlesExceptions` 中介層，允許您節流例外。一旦工作拋出給定數量的例外，所有進一步執行工作的嘗試將被延遲，直到經過指定的時間間隔。此中介層對於與不穩定的第三方服務互動的工作特別有用。

例如，假設有一個佇列工作與開始拋出例外的第三方 API 互動。為了節流例外，您可以從工作的 `middleware` 方法中返回 `ThrottlesExceptions` 中介層。通常，此中介層應與實作[基於時間的嘗試](#time-based-attempts)的工作配對使用：

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

中介層接受的第一個建構函式參數是工作在被節流之前可以拋出的例外數量，而第二個建構函式參數是工作在被節流後再次嘗試之前應經過的秒數。在上述程式碼範例中，如果工作連續拋出 10 個例外，我們將等待 5 分鐘後再嘗試該工作，受限於 30 分鐘的時間限制。

當工作拋出例外但尚未達到例外閾值時，通常會立即重試工作。但是，您可以透過在將中介層附加到工作時呼叫 `backoff` 方法來指定此類工作應延遲的分鐘數：

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

在內部，此中介層使用 Laravel 的快取系統來實作速率限制，並且將工作的類別名稱用作快取「鍵」。您可以透過在將中介層附加到工作時呼叫 `by` 方法來覆寫此鍵。如果您有多個工作與相同的第三方服務互動，並且希望它們共享一個共同的節流「桶」，以確保它們遵守單一的共享限制，這可能會很有用：

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

預設情況下，此中介層將節流所有例外。您可以透過在將中介層附加到工作時呼叫 `when` 方法來修改此行為。只有在提供給 `when` 方法的閉包返回 `true` 時，例外才會被節流：

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

與 `when` 方法不同（該方法將工作釋放回佇列或拋出例外），`deleteWhen` 方法允許您在發生給定例外時完全刪除工作：

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

如果您希望將節流的例外報告給應用程式的例外處理器，您可以透過在將中介層附加到工作時呼叫 `report` 方法來實現。您可以選擇提供一個閉包給 `report` 方法，只有在給定的閉包返回 `true` 時，例外才會被報告：

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
> 如果您正在使用 Redis，您可以使用 `Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis` 中介層，該中介層針對 Redis 進行了微調，並且比基本例外節流中介層更有效率。

<a name="skipping-jobs"></a>
### 跳過工作

`Skip` 中介層允許您指定應跳過/刪除工作，而無需修改工作的邏輯。如果給定條件評估為 `true`，`Skip::when` 方法將刪除工作；而如果條件評估為 `false`，`Skip::unless` 方法將刪除工作：

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
## 分派工作

一旦您編寫完工作類別，您就可以使用工作本身上的 `dispatch` 方法來分派它。傳遞給 `dispatch` 方法的參數將會提供給工作的建構函式：

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

如果您想條件式地分派工作，可以使用 `dispatchIf` 和 `dispatchUnless` 方法：

```php
ProcessPodcast::dispatchIf($accountActive, $podcast);

ProcessPodcast::dispatchUnless($accountSuspended, $podcast);
```

在新的 Laravel 應用程式中，`database` 連線被定義為預設佇列。您可以透過更改應用程式 `.env` 檔案中的 `QUEUE_CONNECTION` 環境變數來指定不同的預設佇列連線。

<a name="delayed-dispatching"></a>
### 延遲分派

如果您想指定某個工作不應立即供佇列工作者處理，您可以在分派工作時使用 `delay` 方法。例如，讓我們指定某個工作應在分派後 10 分鐘才可供處理：

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

在某些情況下，工作可能已配置了預設延遲。如果您需要繞過此延遲並立即分派工作進行處理，可以使用 `withoutDelay` 方法：

```php
ProcessPodcast::dispatch($podcast)->withoutDelay();
```

> [!WARNING]
> Amazon SQS 佇列服務的最大延遲時間為 15 分鐘。

<a name="synchronous-dispatching"></a>
### 同步分派

如果您想立即 (同步地) 分派工作，可以使用 `dispatchSync` 方法。當使用此方法時，工作不會被佇列，而會在當前程序中立即執行：

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
#### 延遲同步分派

使用延遲同步分派 (deferred synchronous dispatching)，您可以分派一個工作在當前程序中處理，但在 HTTP 回應已發送給使用者之後。這允許您同步處理「佇列中的」工作，而不會降低使用者應用程式的體驗。要延遲同步工作的執行，請將工作分派到 `deferred` 連線：

```php
RecordDelivery::dispatch($order)->onConnection('deferred');
```

`deferred` 連線也作為預設的 [容錯移轉佇列](#queue-failover)。

同樣地，`background` 連線會在 HTTP 回應已發送給使用者之後處理工作；然而，該工作是在一個單獨產生的 PHP 進程中處理的，這允許 PHP-FPM / 應用程式工作者可以處理另一個傳入的 HTTP 請求：

```php
RecordDelivery::dispatch($order)->onConnection('background');
```

<a name="jobs-and-database-transactions"></a>
### 工作與資料庫交易

雖然在資料庫交易中分派工作是完全沒問題的，但您應該特別注意確保您的工作能夠成功執行。當在交易中分派工作時，工作者有可能在父交易提交之前就處理該工作。當發生這種情況時，您在資料庫交易期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中創建的任何模型或資料庫記錄可能不存在於資料庫中。

幸運的是，Laravel 提供了幾種解決此問題的方法。首先，您可以在佇列連線的配置陣列中設定 `after_commit` 連線選項：

```php
'redis' => [
    'driver' => 'redis',
    // ...
    'after_commit' => true,
],
```

當 `after_commit` 選項設定為 `true` 時，您可以在資料庫交易中分派工作；然而，Laravel 會等到所有開啟的父資料庫交易都已提交後，才會實際分派工作。當然，如果當前沒有開啟的資料庫交易，工作將會立即分派。

如果交易因交易期間發生的例外而回滾，則在該交易期間分派的工作將會被丟棄。

> [!NOTE]
> 將 `after_commit` 配置選項設定為 `true` 也會導致任何佇列的事件監聽器、郵件、通知和廣播事件在所有開啟的資料庫交易提交後才分派。

<a name="specifying-commit-dispatch-behavior-inline"></a>
#### 行內指定提交分派行為

如果您沒有將 `after_commit` 佇列連線配置選項設定為 `true`，您仍然可以指定某個特定工作應在所有開啟的資料庫交易提交後分派。要實現此目的，您可以將 `afterCommit` 方法鏈接到您的分派操作上：

```php
use App\Jobs\ProcessPodcast;

ProcessPodcast::dispatch($podcast)->afterCommit();
```

同樣地，如果 `after_commit` 配置選項已設定為 `true`，您可以指定某個特定工作應立即分派，而無需等待任何開啟的資料庫交易提交：

```php
ProcessPodcast::dispatch($podcast)->beforeCommit();
```

<a name="job-chaining"></a>
### 工作鏈

工作鏈讓您可以指定一個佇列工作列表，這些工作會在主要工作成功執行後依序運行。如果序列中的其中一個工作失敗，其餘的工作將不會被執行。要執行佇列工作鏈，您可以使用 `Bus` Facade 提供的 `chain` 方法。Laravel 的命令匯流排是佇列工作分派所建立的底層元件：

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

除了鏈接工作類別實例外，您也可以鏈接閉包：

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
> 在工作中使用 `$this->delete()` 方法刪除工作，並不會阻止鏈式工作繼續處理。只有當鏈中的工作失敗時，鏈才會停止執行。

<a name="chain-connection-queue"></a>
#### 鏈連線與佇列

如果您想為鏈式工作指定要使用的連線和佇列，可以使用 `onConnection` 和 `onQueue` 方法。這些方法會指定要使用的佇列連線和佇列名稱，除非佇列工作明確地指定了不同的連線/佇列：

```php
Bus::chain([
    new ProcessPodcast,
    new OptimizePodcast,
    new ReleasePodcast,
])->onConnection('redis')->onQueue('podcasts')->dispatch();
```

<a name="adding-jobs-to-the-chain"></a>
#### 將工作新增至鏈

有時，您可能需要從鏈中另一個工作的內部，將工作前置或附加到現有的工作鏈中。您可以使用 `prependToChain` 和 `appendToChain` 方法來完成此操作：

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
#### 鏈失敗

鏈接工作時，您可以使用 `catch` 方法來指定一個閉包，當鏈中的工作失敗時，該閉包應被調用。給定的回呼函式將會收到導致工作失敗的 `Throwable` 實例：

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
> 由於鏈回呼函式會被序列化並由 Laravel 佇列在稍後執行，因此您不應該在鏈回呼函式中使用 `$this` 變數。

<a name="customizing-the-queue-and-connection"></a>
### 自訂佇列與連線

<a name="dispatching-to-a-particular-queue"></a>
#### 分派到特定佇列

透過將工作推送到不同的佇列，您可以「分類」您的佇列工作，甚至可以優先處理您分配給各種佇列的工作者數量。請記住，這並非將工作推送到佇列設定檔中定義的不同佇列「連線」，而僅是推送到單一連線中的特定佇列。要指定佇列，請在分派工作時使用 `onQueue` 方法：

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

或者，您也可以透過在工作的建構式中呼叫 `onQueue` 方法來指定工作的佇列：

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

如果您的應用程式與多個佇列連線互動，您可以使用 `onConnection` 方法來指定要將工作推送到哪個連線：

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

您可以將 `onConnection` 和 `onQueue` 方法鏈接在一起，以指定工作的連線和佇列：

```php
ProcessPodcast::dispatch($podcast)
    ->onConnection('sqs')
    ->onQueue('processing');
```

或者，您也可以透過在工作的建構式中呼叫 `onConnection` 方法來指定工作的連線：

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
### 指定工作最大嘗試次數/逾時值


<a name="max-attempts"></a>
#### 最大嘗試次數

工作嘗試是 Laravel 佇列系統的核心概念，並驅動許多進階功能。儘管它們一開始可能令人困惑，但在修改預設配置之前，了解它們的運作方式非常重要。

當一個工作被分派時，它會被推送到佇列中。然後工作者會接收它並嘗試執行它。這就是一次工作嘗試。

然而，一次嘗試並不一定意味著工作中的 `handle` 方法已被執行。嘗試也可以透過幾種方式「被消耗掉」：

<div class="content-list" markdown="1">

- 工作在執行期間遇到未處理的例外。
- 工作使用 `$this->release()` 手動釋放回佇列。
- Middleware (中介層)，例如 `WithoutOverlapping` 或 `RateLimited` 未能取得鎖定並釋放工作。
- 工作逾時。
- 工作的 `handle` 方法執行並完成，沒有拋出例外。

</div>

你可能不希望無限期地嘗試一個工作。因此，Laravel 提供了多種方式來指定一個工作可以嘗試的次數或持續的時間。

> [!NOTE]
> 預設情況下，Laravel 只會嘗試一次工作。如果你的工作使用 `WithoutOverlapping` 或 `RateLimited` 等中介層，或者你手動釋放工作，你可能需要透過 `tries` 選項來增加允許的嘗試次數。

指定工作最大嘗試次數的一種方法是透過 Artisan 命令列上的 `--tries` 開關。這將應用於工作者處理的所有工作，除非正在處理的工作指定了它可以嘗試的次數：

```shell
php artisan queue:work --tries=3
```

如果一個工作超過其最大嘗試次數，它將被視為「失敗」的工作。有關處理失敗工作的更多資訊，請查閱[失敗工作文件](#dealing-with-failed-jobs)。如果提供 `--tries=0` 給 `queue:work` 命令，該工作將無限期地重試。

你可以透過在工作類別本身定義工作最大嘗試次數來採用更精細的方法。如果在工作上指定了最大嘗試次數，它將優先於命令列上提供的 `--tries` 值：

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

如果你需要對特定工作的最大嘗試次數進行動態控制，你可以在工作上定義一個 `tries` 方法：

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
#### 以時間為基礎的嘗試

除了定義工作在失敗前可以嘗試的次數之外，你還可以定義一個工作不再被嘗試的時間點。這允許工作在給定時間範圍內被嘗試任意次數。要定義工作不再被嘗試的時間點，請在你的工作類別中新增一個 `retryUntil` 方法。此方法應返回一個 `DateTime` 實例：

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
> 你也可以在你的[佇列事件監聽器](/docs/{{version}}/events#queued-event-listeners)和[佇列通知](/docs/{{version}}/notifications#queueing-notifications)上定義 `tries` 屬性或 `retryUntil` 方法。


<a name="max-exceptions"></a>
#### 最大例外次數

有時你可能希望指定一個工作可以嘗試多次，但如果重試是由給定數量的未處理例外觸發的（而不是直接由 `release` 方法釋放），則該工作應該失敗。要實現這一點，你可以在工作類別上定義一個 `maxExceptions` 屬性：

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

在此範例中，如果應用程式無法取得 Redis 鎖定，工作將被釋放十秒，並將繼續重試最多 25 次。然而，如果工作拋出三個未處理的例外，則工作將會失敗。


<a name="timeout"></a>
#### 逾時

通常，你知道你的佇列工作大約需要多長時間。因此，Laravel 允許你指定一個「逾時」值。預設情況下，逾時值為 60 秒。如果一個工作的處理時間超過逾時值指定的秒數，則處理該工作的工作者將會因錯誤而退出。通常，工作者將會由[伺服器上配置的進程管理器](#supervisor-configuration)自動重啟。

工作可以運行的最大秒數可以使用 Artisan 命令列上的 `--timeout` 開關來指定：

```shell
php artisan queue:work --timeout=30
```

如果工作因持續逾時而超出其最大嘗試次數，它將被標記為失敗。

你也可以在工作類別本身定義工作應該允許運行的最大秒數。如果在工作上指定了逾時，它將優先於命令列上指定的任何逾時：

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

有時，IO 阻塞進程（例如 socket 或 outgoing HTTP 連線）可能不會遵守你指定的逾時。因此，在使用這些功能時，你應該始終嘗試使用其 API 來指定逾時。例如，當使用 [Guzzle](https://docs.guzzlephp.org) 時，你應該始終指定連線和請求的逾時值。

> [!WARNING]
> 必須安裝 [PCNTL](https://www.php.net/manual/en/book.pcntl.php) PHP 擴充功能才能指定工作逾時。此外，工作的「逾時」值應始終小於其[「重試時間」](#job-expiration)值。否則，工作可能會在其實際完成執行或逾時之前就被重新嘗試。


<a name="failing-on-timeout"></a>
#### 逾時失敗

如果你想指示一個工作在逾時時應被標記為[失敗](#dealing-with-failed-jobs)，你可以在工作類別上定義 `$failOnTimeout` 屬性：

```php
/**
 * Indicate if the job should be marked as failed on timeout.
 *
 * @var bool
 */
public $failOnTimeout = true;
```

> [!NOTE]
> 預設情況下，當工作逾時時，它會消耗一次嘗試並被釋放回佇列（如果允許重試）。然而，如果你將工作設定為逾時失敗，則無論 tries 設定為何值，該工作都不會被重試。

<a name="sqs-fifo-and-fair-queues"></a>
### SQS FIFO 與公平佇列

Laravel 支援 [Amazon SQS FIFO (先進先出)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html) 佇列，讓你能以精確的順序處理被傳送的工作，同時透過訊息去重複確保只處理一次。

FIFO 佇列需要訊息群組 ID 來確定哪些工作可以平行處理。具有相同群組 ID 的工作會依序處理，而具有不同群組 ID 的訊息可以同步處理。

Laravel 提供流暢的 `onGroup` 方法，用於在分派工作時指定訊息群組 ID：

```php
ProcessOrder::dispatch($order)
    ->onGroup("customer-{$order->customer_id}");
```

SQS FIFO 佇列支援訊息去重複，以確保只處理一次。在您的工作類別中實作 `deduplicationId` 方法，以提供自訂的去重複 ID：

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

使用 FIFO 佇列時，您還需要定義監聽器、郵件與通知的訊息群組。或者，您可以將這些物件的佇列實例分派到非 FIFO 佇列。

若要為 [佇列事件監聽器](/docs/{{version}}/events#queued-event-listeners) 定義訊息群組，請在監聽器上定義一個 `messageGroup` 方法。您也可以選擇性地定義 `deduplicationId` 方法：

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

當傳送即將被佇列到 FIFO 佇列的 [郵件訊息](/docs/{{version}}/mail) 時，您應該在傳送通知時呼叫 `onGroup` 方法，並可選地呼叫 `withDeduplicator` 方法：

```php
use App\Mail\InvoicePaid;
use Illuminate\Support\Facades\Mail;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

Mail::to($request->user())->send($invoicePaid);
```

當傳送即將被佇列到 FIFO 佇列的 [通知](/docs/{{version}}/notifications) 時，您應該在傳送通知時呼叫 `onGroup` 方法，並可選地呼叫 `withDeduplicator` 方法：

```php
use App\Notifications\InvoicePaid;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

$user->notify($invoicePaid);
```

<a name="queue-failover"></a>
### 佇列容錯移轉

`failover` 佇列驅動程式在將工作推送到佇列時提供自動容錯移轉功能。如果 `failover` 設定中的主要佇列連線因任何原因失敗，Laravel 會自動嘗試將工作推送到清單中的下一個設定連線。這對於確保佇列可靠性至關重要的生產環境中的高可用性特別有用。

若要配置容錯移轉佇列連線，請指定 `failover` 驅動程式並提供一個依序嘗試的連線名稱陣列。預設情況下，Laravel 在您的應用程式的 `config/queue.php` 配置檔中包含一個容錯移轉配置範例：

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

一旦您配置了一個使用 `failover` 驅動程式的連線，您將需要將該容錯移轉連線設定為您應用程式的 `.env` 檔中的預設佇列連線，以利用容錯移轉功能：

```ini
QUEUE_CONNECTION=failover
```

接下來，為您的容錯移轉連線清單中的每個連線啟動至少一個工作者：

```bash
php artisan queue:work redis
php artisan queue:work database
```

> [!NOTE]
> 您不需要為使用 `sync`、`background` 或 `deferred` 佇列驅動程式的連線執行工作者，因為這些驅動程式在當前的 PHP 程序中處理工作。

當佇列連線操作失敗並啟用容錯移轉時，Laravel 將會分派 `Illuminate\Queue\Events\QueueFailedOver` 事件，讓您可以報告或記錄佇列連線已失敗。

> [!NOTE]
> 如果您使用 Laravel Horizon，請記住 Horizon 只管理 Redis 佇列。如果您的容錯移轉清單包含 `database`，您應該在 Horizon 旁執行常規的 `php artisan queue:work database` 程序。

<a name="error-handling"></a>
### 錯誤處理

如果工作在處理過程中拋出例外，該工作將自動被釋放回佇列，以便再次嘗試執行。工作將持續被釋放，直到它達到應用程式允許的最大嘗試次數。最大嘗試次數由 `queue:work` Artisan 命令使用的 `--tries` 選項定義。此外，最大嘗試次數也可以在工作類別本身中定義。有關執行佇列工作者的更多資訊，請參閱[下方](#running-the-queue-worker)。

<a name="manually-releasing-a-job"></a>
#### 手動釋放工作

有時您可能希望手動將工作釋放回佇列，以便稍後再次嘗試執行。您可以透過呼叫 `release` 方法來達成此目的：

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

預設情況下，`release` 方法會將工作釋放回佇列以立即處理。但是，您可以透過向 `release` 方法傳遞整數或日期實例，指示佇列在經過指定秒數之前不讓工作可用於處理：

```php
$this->release(10);

$this->release(now()->plus(seconds: 10));
```

<a name="manually-failing-a-job"></a>
#### 手動使工作失敗

有時您可能需要手動將工作標記為「失敗」。您可以透過呼叫 `fail` 方法來達成此目的：

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

如果您想將工作標記為失敗，因為您已捕獲一個例外，您可以將該例外傳遞給 `fail` 方法。或者，為了方便，您可以傳遞一個字串錯誤訊息，它將為您轉換為例外：

```php
$this->fail($exception);

$this->fail('Something went wrong.');
```

> [!NOTE]
> 有關失敗工作的更多資訊，請參閱[處理工作失敗的文檔](#dealing-with-failed-jobs)。

<a name="fail-jobs-on-exceptions"></a>
#### 在特定例外情況下使工作失敗

`FailOnException` [工作中介層](#job-middleware) 允許您在拋出特定例外時中斷重試。這使得在面對暫時性例外（例如外部 API 錯誤）時可以重試，但在面對持久性例外（例如使用者的權限被撤銷）時，則永久使工作失敗：

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
## 工作批次處理

Laravel 的工作批次處理功能讓您輕鬆執行一批工作，並在批次工作完成執行後執行某些動作。在開始之前，您應該建立一個資料庫遷移來建立一個表格，該表格將包含有關您工作批次的元資訊，例如它們的完成百分比。此遷移可以使用 `make:queue-batches-table` Artisan 命令產生：

```shell
php artisan make:queue-batches-table

php artisan migrate
```

<a name="defining-batchable-jobs"></a>
### 定義可批次處理的工作

要定義一個可批次處理的工作，您應該像往常一樣[建立一個佇列工作](#creating-jobs)；但您應該將 `Illuminate\Bus\Batchable` trait 新增到工作類別中。這個 trait 提供了一個 `batch` 方法的存取權限，該方法可用於檢索工作目前正在執行的批次：

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

要分派一批工作，您應該使用 `Bus` facade 的 `batch` 方法。當然，批次處理主要在與完成回呼結合使用時才有用。因此，您可以使用 `then`、`catch` 和 `finally` 方法來定義批次的完成回呼。每個回呼在被調用時都將收到一個 `Illuminate\Bus\Batch` 實例。

當執行多個佇列工作者時，批次中的工作將會平行處理。因此，工作完成的順序可能與它們被新增到批次中的順序不同。請查閱我們關於[工作鏈與批次](#chains-and-batches)的說明文件，以了解如何依序執行一系列工作。

在這個範例中，我們將想像正在將一批工作加入佇列，每個工作都處理 CSV 檔案中的指定行數：

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

該批次的 ID，可以透過 `$batch->id` 屬性存取，可用於在分派後[查詢 Laravel 命令總線](#inspecting-batches)以獲取批次資訊。

> [!WARNING]
> 由於批次回呼會被序列化並由 Laravel 佇列在稍後執行，因此您不應在回呼中使用 `$this` 變數。此外，由於批次處理的工作包裹在資料庫交易中，觸發隱式提交的資料庫語句不應在工作中執行。

<a name="naming-batches"></a>
#### 命名批次

某些工具，例如 [Laravel Horizon](/docs/{{version}}/horizon) 和 [Laravel Telescope](/docs/{{version}}/telescope)，如果批次有命名，可以為批次提供更友善的偵錯資訊。要為批次指定任意名稱，您可以在定義批次時呼叫 `name` 方法：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import CSV')->dispatch();
```

<a name="batch-connection-queue"></a>
#### 批次連線與佇列

如果您想要為批次工作指定要使用的連線和佇列，您可以使用 `onConnection` 和 `onQueue` 方法。所有批次處理的工作都必須在相同的連線和佇列中執行：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->onConnection('redis')->onQueue('imports')->dispatch();
```

<a name="chains-and-batches"></a>
### 鏈與批次

您可以透過將鏈接的工作放在陣列中，在批次內定義一組[鏈接的工作](#job-chaining)。例如，我們可以平行執行兩個工作鏈，並在兩個工作鏈都完成處理後執行一個回呼：

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

反之，您也可以在[工作鏈](#job-chaining)中執行批次工作，透過在鏈中定義批次。例如，您可以先執行一批工作來發布多個 Podcast，然後再執行一批工作來傳送發布通知：

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
### 將工作新增至批次

有時，從批次處理的工作中向批次新增額外的工作可能會很有用。當您需要批次處理數千個工作，而這些工作在網頁請求期間分派可能花費太長時間時，此模式會很有用。因此，您可以選擇分派一批最初的「載入器」工作，以使用更多工作來填充該批次：

```php
$batch = Bus::batch([
    new LoadImportBatch,
    new LoadImportBatch,
    new LoadImportBatch,
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import Contacts')->dispatch();
```

在這個範例中，我們將使用 `LoadImportBatch` 工作來填充批次，新增額外的工作。為此，我們可以利用從工作 `batch` 方法存取的批次實例上的 `add` 方法：

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
> 您只能從屬於同一批次的工作中向該批次新增工作。

<a name="inspecting-batches"></a>
### 檢查批次

提供給批次完成回呼的 `Illuminate\Bus\Batch` 實例，具有多種屬性和方法，可協助您與指定的批次工作互動及檢查：

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
#### 從路由返回批次

所有 `Illuminate\Bus\Batch` 實例都是 JSON 序列化（JSON serializable）的，這表示您可以直接從應用程式的路由返回它們，以取得包含批次資訊（包括其完成進度）的 JSON Payload。這使得在應用程式 UI 中顯示批次完成進度資訊變得方便。

要透過其 ID 檢索批次，您可以使用 `Bus` Facade 的 `findBatch` 方法：

```php
use Illuminate\Support\Facades\Bus;
use Illuminate\Support\Facades\Route;

Route::get('/batch/{batchId}', function (string $batchId) {
    return Bus::findBatch($batchId);
});
```

<a name="cancelling-batches"></a>
### 取消批次

有時候，您可能需要取消指定批次的執行。這可以透過呼叫 `Illuminate\Bus\Batch` 實例上的 `cancel` 方法來實現：

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

如您在之前的範例中所見，批次工作通常應在繼續執行之前，判斷其對應的批次是否已被取消。然而，為方便起見，您可以將 `SkipIfBatchCancelled` [中介層](#job-middleware)指派給工作。顧名思義，此中介層將指示 Laravel，如果其對應的批次已取消，則不要處理該工作：

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

當批次工作失敗時，將會呼叫 `catch` 回呼（如果已指派）。此回呼只會針對批次中第一個失敗的工作被呼叫。

<a name="allowing-failures"></a>
#### 允許失敗

當批次中的工作失敗時，Laravel 會自動將該批次標記為「已取消」。如果您願意，您可以停用此行為，這樣工作失敗就不會自動將批次標記為已取消。這可以透過在分派批次時呼叫 `allowFailures` 方法來實現：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->allowFailures()->dispatch();
```

您可以選擇性地向 `allowFailures` 方法提供一個閉包，該閉包將在每次工作失敗時執行：

```php
$batch = Bus::batch([
    // ...
])->allowFailures(function (Batch $batch, $exception) {
    // Handle individual job failures...
})->dispatch();
```

<a name="retrying-failed-batch-jobs"></a>
#### 重試失敗的批次工作

為方便起見，Laravel 提供了一個 `queue:retry-batch` Artisan 命令，可讓您輕鬆重試指定批次的所有失敗工作。此命令接受要重試的批次工作的 UUID：

```shell
php artisan queue:retry-batch 32dbc76c-4f82-4749-b610-a639fe0099b5
```

<a name="pruning-batches"></a>
### 修剪批次

如果不進行修剪，`job_batches` 資料表會很快累積大量紀錄。為緩解此問題，您應該[排程](/docs/{{version}}/scheduling) `queue:prune-batches` Artisan 命令每天執行：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches')->daily();
```

預設情況下，所有超過 24 小時的已完成批次都將被修剪。您可以在呼叫命令時使用 `hours` 選項來決定保留批次資料的時間長度。例如，以下命令將刪除所有超過 48 小時前完成的批次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48')->daily();
```

有時候，您的 `jobs_batches` 資料表可能會累積從未成功完成的批次紀錄，例如工作失敗且該工作從未成功重試的批次。您可以指示 `queue:prune-batches` 命令使用 `unfinished` 選項來修剪這些未完成的批次紀錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --unfinished=72')->daily();
```

同樣地，您的 `jobs_batches` 資料表也可能累積已取消批次的紀錄。您可以指示 `queue:prune-batches` 命令使用 `cancelled` 選項來修剪這些已取消的批次紀錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --cancelled=72')->daily();
```

<a name="storing-batches-in-dynamodb"></a>
### 在 DynamoDB 中儲存批次

Laravel 也支援將批次的中繼資訊儲存在 [DynamoDB](https://aws.amazon.com/dynamodb) 而非關聯式資料庫中。不過，你需要手動建立一個 DynamoDB 資料表來儲存所有批次記錄。

通常，這個資料表應命名為 `job_batches`，但你應根據應用程式的 `queue` 配置檔中 `queue.batching.table` 設定值來命名資料表。

<a name="dynamodb-batch-table-configuration"></a>
#### DynamoDB 批次資料表設定

`job_batches` 資料表應具有一個名為 `application` 的字串主要分區鍵 (primary partition key) 和一個名為 `id` 的字串主要排序鍵 (primary sort key)。鍵的 `application` 部分將包含應用程式的名稱，此名稱由應用程式的 `app` 配置檔中 `name` 設定值所定義。由於應用程式名稱是 DynamoDB 資料表鍵的一部分，因此你可以使用相同的資料表來儲存多個 Laravel 應用程式的工作批次。

此外，你也可以為你的資料表定義 `ttl` 屬性，如果你想利用 [自動修剪批次](#pruning-batches-in-dynamodb) 功能。

<a name="dynamodb-configuration"></a>
#### DynamoDB 配置

接下來，安裝 AWS SDK，以便你的 Laravel 應用程式可以與 Amazon DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

然後，將 `queue.batching.driver` 配置選項的值設定為 `dynamodb`。此外，你應在 `batching` 配置陣列中定義 `key`、`secret` 和 `region` 配置選項。這些選項將用於向 AWS 進行身份驗證。當使用 `dynamodb` 驅動程式時，`queue.batching.database` 配置選項是不必要的：

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
#### 修剪 DynamoDB 中的批次

當利用 [DynamoDB](https://aws.amazon.com/dynamodb) 儲存工作批次資訊時，用於修剪儲存在關聯式資料庫中批次的典型修剪命令將無法運作。取而代之的是，你可以利用 [DynamoDB 原生的 TTL 功能](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html) 來自動移除舊批次的記錄。

如果你為 DynamoDB 資料表定義了 `ttl` 屬性，你可以定義配置參數來指示 Laravel 如何修剪批次記錄。`queue.batching.ttl_attribute` 配置值定義了儲存 TTL 的屬性名稱，而 `queue.batching.ttl` 配置值定義了批次記錄從 DynamoDB 資料表移除的秒數，此秒數是相對於記錄上次更新的時間：

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

與其分派一個工作類別到佇列，您也可以分派一個閉包。這對於需要在當前請求週期之外執行的快速、簡單任務非常有用。當分派閉包到佇列時，閉包的程式碼內容會經過密碼學簽署，使其在傳輸過程中無法被修改：

```php
use App\Models\Podcast;

$podcast = Podcast::find(1);

dispatch(function () use ($podcast) {
    $podcast->publish();
});
```

若要為佇列閉包指定一個名稱，該名稱可用於佇列報告儀表板，並可透過 `queue:work` 命令顯示，您可以使用 `name` 方法：

```php
dispatch(function () {
    // ...
})->name('Publish Podcast');
```

透過使用 `catch` 方法，您可以提供一個閉包，該閉包將在佇列閉包耗盡所有 [已設定的重試嘗試次數](#max-job-attempts-and-timeout) 仍未能成功完成時執行：

```php
use Throwable;

dispatch(function () use ($podcast) {
    $podcast->publish();
})->catch(function (Throwable $e) {
    // This job has failed...
});
```

> [!WARNING]
> 由於 `catch` 回呼會被 Laravel 佇列序列化並在稍後執行，因此您不應在 `catch` 回呼內使用 `$this` 變數。

<a name="running-the-queue-worker"></a>
## 執行佇列工作者

<a name="the-queue-work-command"></a>
### `queue:work` 命令

Laravel 包含一個 Artisan 命令，它將啟動一個佇列工作者並處理推送至佇列中的新任務。您可以使用 `queue:work` Artisan 命令來運行工作者。請注意，一旦 `queue:work` 命令啟動，它將持續運行直到您手動停止或關閉終端機：

```shell
php artisan queue:work
```

> [!NOTE]
> 為了讓 `queue:work` 程式永久在背景運行，您應該使用像 [Supervisor](#supervisor-configuration) 這樣的程序監控器，以確保佇列工作者不會停止運行。

您可以在調用 `queue:work` 命令時包含 `-v` 標誌，如果您希望在命令輸出中包含已處理的任務 ID、連線名稱和佇列名稱：

```shell
php artisan queue:work -v
```

請記住，佇列工作者是長期運行的程序，並將啟動的應用程式狀態儲存在記憶體中。因此，它們在啟動後不會注意到程式碼庫的變更。所以，在您的部署過程中，請務必 [重新啟動佇列工作者](#queue-workers-and-deployment)。此外，請記住，您的應用程式所建立或修改的任何靜態狀態都不會在任務之間自動重置。

或者，您可以運行 `queue:listen` 命令。當使用 `queue:listen` 命令時，您無需在需要重新載入更新的程式碼或重置應用程式狀態時手動重新啟動工作者；但是，此命令的效率遠不如 `queue:work` 命令：

```shell
php artisan queue:listen
```

<a name="running-multiple-queue-workers"></a>
#### 執行多個佇列工作者

要為一個佇列指派多個工作者並同時處理任務，您只需啟動多個 `queue:work` 程序即可。這可以在本機透過終端機的多個分頁完成，或在生產環境中使用您的程序管理器的配置設定。 [當使用 Supervisor](#supervisor-configuration) 時，您可以使用 `numprocs` 配置值。

<a name="specifying-the-connection-queue"></a>
#### 指定連線與佇列

您也可以指定工作者應使用的佇列連線。傳遞給 `work` 命令的連線名稱應與您的 `config/queue.php` 設定檔中定義的其中一個連線相對應：

```shell
php artisan queue:work redis
```

預設情況下，`queue:work` 命令只處理給定連線上預設佇列的任務。但是，您還可以透過只處理給定連線上的特定佇列來進一步自訂佇列工作者。例如，如果您所有的電子郵件都在 `redis` 佇列連線上的 `emails` 佇列中處理，您可以發出以下命令來啟動一個只處理該佇列的工作者：

```shell
php artisan queue:work redis --queue=emails
```

<a name="processing-a-specified-number-of-jobs"></a>
#### 處理指定數量的任務

`--once` 選項可用於指示工作者只處理佇列中的單一任務：

```shell
php artisan queue:work --once
```

`--max-jobs` 選項可用於指示工作者處理給定數量的任務後退出。此選項與 [Supervisor](#supervisor-configuration) 結合使用時可能很有用，以便您的工作者在處理完指定數量的任務後自動重新啟動，釋放可能累積的任何記憶體：

```shell
php artisan queue:work --max-jobs=1000
```

<a name="processing-all-queued-jobs-then-exiting"></a>
#### 處理所有佇列中的任務並退出

`--stop-when-empty` 選項可用於指示工作者處理所有任務，然後優雅地退出。當在 Docker 容器中處理 Laravel 佇列時，如果您希望在佇列清空後關閉容器，此選項可能很有用：

```shell
php artisan queue:work --stop-when-empty
```

<a name="processing-jobs-for-a-given-number-of-seconds"></a>
#### 在指定秒數內處理任務

`--max-time` 選項可用於指示工作者在給定秒數內處理任務，然後退出。此選項與 [Supervisor](#supervisor-configuration) 結合使用時可能很有用，以便您的工作者在處理任務一段時間後自動重新啟動，釋放可能累積的任何記憶體：

```shell
# Process jobs for one hour and then exit...
php artisan queue:work --max-time=3600
```

<a name="worker-sleep-duration"></a>
#### 工作者休眠時間

當佇列中有任務時，工作者將會不間斷地處理任務。但是，`sleep` 選項決定了如果沒有可用的任務，工作者將「休眠」多少秒。當然，在休眠期間，工作者將不會處理任何新任務：

```shell
php artisan queue:work --sleep=3
```

<a name="maintenance-mode-queues"></a>
#### 維護模式與佇列

當您的應用程式處於 [維護模式](/docs/{{version}}/configuration#maintenance-mode) 時，所有佇列任務將不會被處理。一旦應用程式退出維護模式，這些任務將會照常處理。

要強制佇列工作者處理任務，即使維護模式已啟用，您可以使用 `--force` 選項：

```shell
php artisan queue:work --force
```

<a name="resource-considerations"></a>
#### 資源考量

常駐佇列工作者在處理每個任務之前不會「重新啟動」框架。因此，您應該在每個任務完成後釋放任何佔用大量資源的項目。例如，如果您正在使用 [GD 函式庫](https://www.php.net/manual/en/book.image.php) 進行影像處理，您應該在處理完影像後使用 `imagedestroy` 釋放記憶體。

<a name="queue-priorities"></a>
### 佇列優先級

有時您可能希望優先處理您的佇列。例如，在您的 `config/queue.php` 設定檔中，您可以將 `redis` 連線的預設 `queue` 設定為 `low`。然而，偶爾您可能希望像這樣將任務推送到 `high` 優先級佇列：

```php
dispatch((new Job)->onQueue('high'));
```

要啟動一個工作者，確保所有 `high` 佇列任務在繼續處理 `low` 佇列中的任何任務之前都已處理完畢，請將以逗號分隔的佇列名稱列表傳遞給 `work` 命令：

```shell
php artisan queue:work --queue=high,low
```

<a name="queue-workers-and-deployment"></a>
### 佇列工作者與部署

由於佇列工作者是長期運行的程序，它們在未重新啟動的情況下不會注意到程式碼的變更。因此，部署使用佇列工作者的應用程式最簡單的方法是在部署過程中重新啟動工作者。您可以透過發出 `queue:restart` 命令來優雅地重新啟動所有工作者：

```shell
php artisan queue:restart
```

此命令將指示所有佇列工作者在處理完當前任務後優雅地退出，以便不會遺失任何現有任務。由於 `queue:restart` 命令執行時佇列工作者將會退出，您應該運行一個程序管理器，例如 [Supervisor](#supervisor-configuration)，以自動重新啟動佇列工作者。

> [!NOTE]
> 佇列使用 [快取](/docs/{{version}}/cache) 來儲存重新啟動訊號，因此在使用此功能之前，您應該驗證您的應用程式是否已正確配置快取驅動程式。

<a name="job-expirations-and-timeouts"></a>
### 工作到期與逾時


<a name="job-expiration"></a>
#### 工作到期

在您的 `config/queue.php` 設定檔中，每個佇列連線都定義了 `retry_after` 選項。此選項指定佇列連線在重試正在處理中的工作之前應等待多少秒。舉例來說，如果 `retry_after` 的值設為 `90`，那麼如果該工作已處理 90 秒而未被釋放或刪除，該工作將會被釋放回佇列中。通常，您應該將 `retry_after` 的值設為您的工作合理完成處理所需的最大秒數。

> [!WARNING]
> 唯一不包含 `retry_after` 值的佇列連線是 Amazon SQS。SQS 會根據 [Default Visibility Timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html) 重試工作，此逾時設定在 AWS 主控台中管理。


<a name="worker-timeouts"></a>
#### 工作者逾時

`queue:work` Artisan 命令公開了 `--timeout` 選項。預設情況下，`--timeout` 的值為 60 秒。如果工作處理時間超過逾時值所指定的秒數，處理該工作的工作者將會因錯誤而終止。通常，工作者將由[伺服器上配置的程序管理器](#supervisor-configuration)自動重新啟動：

```shell
php artisan queue:work --timeout=60
```

`retry_after` 設定選項和 `--timeout` CLI 選項是不同的，但它們協同運作以確保工作不會遺失，並且工作只會成功處理一次。

> [!WARNING]
> `--timeout` 值應始終比您的 `retry_after` 設定值至少短幾秒。這將確保處理凍結工作的執行緒在工作被重試之前終止。如果您的 `--timeout` 選項比您的 `retry_after` 設定值長，您的工作可能會被處理兩次。


<a name="pausing-and-resuming-queue-workers"></a>
### 暫停與恢復佇列工作者

有時您可能需要暫時阻止佇列工作者處理新工作，而不完全停止工作者。例如，您可能希望在系統維護期間暫停工作處理。Laravel 提供了 `queue:pause` 和 `queue:continue` Artisan 命令來暫停和恢復佇列工作者。

要暫停特定佇列，請提供佇列連線名稱和佇列名稱：

```shell
php artisan queue:pause database:default
```

在此範例中，`database` 是佇列連線名稱，`default` 是佇列名稱。一旦佇列被暫停，任何正在處理該佇列中工作的工作者將會繼續完成其目前的工作，但在佇列恢復之前，不會接收任何新工作。

要恢復處理已暫停佇列中的工作，請使用 `queue:continue` 命令：

```shell
php artisan queue:continue database:default
```

恢復佇列後，工作者將立即開始處理該佇列中的新工作。請注意，暫停佇列並不會停止工作者程序本身 — 它只會阻止工作者處理指定佇列中的新工作。


<a name="worker-restart-and-pause-signals"></a>
#### 工作者重啟與暫停訊號

預設情況下，佇列工作者在每次工作迭代時會輪詢快取驅動程式以獲取重啟和暫停訊號。雖然這種輪詢對於響應 `queue:restart` 和 `queue:pause` 命令至關重要，但它確實會引入輕微的效能開銷。

如果您需要最佳化效能且不需要這些中斷功能，您可以透過呼叫 `Queue` Facade 上的 `withoutInterruptionPolling` 方法來全域停用此輪詢。這通常應該在您的 `AppServiceProvider` 的 `boot` 方法中完成：

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

或者，您可以透過在 `Illuminate\Queue\Worker` 類別上設定靜態 `$restartable` 或 `$pausable` 屬性來個別停用重啟或暫停輪詢：

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
> 當中斷輪詢被停用時，工作者將不會響應 `queue:restart` 或 `queue:pause` 命令（取決於停用了哪些功能）。

<a name="supervisor-configuration"></a>
## Supervisor 配置

在生產環境中，您需要一種方法來維持 `queue:work` 處理程序的運行。`queue:work` 處理程序可能會因多種原因停止運行，例如工作者逾時或執行 `queue:restart` 命令。

因此，您需要配置一個處理程序監控器，它可以偵測到您的 `queue:work` 處理程序退出時自動重新啟動它們。此外，處理程序監控器還可以讓您指定要同時運行多少個 `queue:work` 處理程序。Supervisor 是 Linux 環境中常用的處理程序監控器，我們將在以下文件中討論如何配置它。

<a name="installing-supervisor"></a>
#### 安裝 Supervisor

Supervisor 是 Linux 作業系統的處理程序監控器，如果您的 `queue:work` 處理程序失敗，它將自動重新啟動它們。要在 Ubuntu 上安裝 Supervisor，您可以使用以下命令：

```shell
sudo apt-get install supervisor
```

> [!NOTE]
> 如果自行配置和管理 Supervisor 聽起來很繁瑣，考慮使用 [Laravel Cloud](https://cloud.laravel.com)，它提供了一個完全託管的平台來運行 Laravel 佇列工作者。

<a name="configuring-supervisor"></a>
#### 配置 Supervisor

Supervisor 配置檔案通常儲存在 `/etc/supervisor/conf.d` 目錄中。在此目錄中，您可以建立任意數量的配置檔案，指示 supervisor 如何監控您的處理程序。例如，讓我們建立一個 `laravel-worker.conf` 檔案，啟動並監控 `queue:work` 處理程序：

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

在此範例中，`numprocs` 指令將指示 Supervisor 運行八個 `queue:work` 處理程序並監控所有這些處理程序，如果它們失敗，將自動重新啟動。您應該更改配置中的 `command` 指令，以反映您所需的佇列連線和工作者選項。

> [!WARNING]
> 您應該確保 `stopwaitsecs` 的值大於您運行時間最長的工作所消耗的秒數。否則，Supervisor 可能會在工作完成處理之前終止該工作。

<a name="starting-supervisor"></a>
#### 啟動 Supervisor

配置檔案建立後，您可以使用以下命令更新 Supervisor 配置並啟動處理程序：

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start "laravel-worker:*"
```

有關 Supervisor 的更多資訊，請參閱 [Supervisor 文件](http://supervisord.org/index.html)。

<a name="dealing-with-failed-jobs"></a>
## 處理失敗的工作

有時候，你佇列中的工作可能會失敗。別擔心，事情不總會如計畫般順利！Laravel 提供了一種便利的方式來 [指定工作最大嘗試次數/逾時值](#max-job-attempts-and-timeout)。當一個非同步工作超過這個嘗試次數後，它將被插入到 `failed_jobs` 資料庫表格中。[同步分派的工作](/docs/{{version}}/queues#synchronous-dispatching) 如果失敗，則不會儲存在此表格中，其例外會立即由應用程式處理。

通常，在新的 Laravel 應用程式中，用於建立 `failed_jobs` 表格的遷移檔案已經存在。但是，如果你的應用程式沒有包含此表格的遷移檔案，你可以使用 `make:queue-failed-table` 命令來建立它：

```shell
php artisan make:queue-failed-table

php artisan migrate
```

當執行 [佇列工作者](#running-the-queue-worker) 程序時，你可以使用 `queue:work` 命令上的 `--tries` 選項來指定工作應嘗試的最大次數。如果你未為 `--tries` 選項指定值，工作只會被嘗試一次，或按照工作類別的 `$tries` 屬性所指定的次數進行嘗試：

```shell
php artisan queue:work redis --tries=3
```

使用 `--backoff` 選項，你可以指定 Laravel 在重試遇到例外狀況的工作前應該等待多少秒。預設情況下，工作會立即釋放回佇列中，以便再次嘗試：

```shell
php artisan queue:work redis --tries=3 --backoff=3
```

如果你想為每個工作設定 Laravel 在重試遇到例外狀況的工作前應等待多少秒，你可以在工作類別本身定義一個 `backoff` 屬性：

```php
/**
 * The number of seconds to wait before retrying the job.
 *
 * @var int
 */
public $backoff = 3;
```

如果你需要更複雜的邏輯來決定工作的退避時間，你可以在工作類別上定義一個 `backoff` 方法：

```php
/**
 * Calculate the number of seconds to wait before retrying the job.
 */
public function backoff(): int
{
    return 3;
}
```

你可以透過從 `backoff` 方法返回一個退避值陣列來輕鬆設定「指數型」退避。在此範例中，第一次重試的延遲為 1 秒，第二次重試為 5 秒，第三次重試為 10 秒，如果還有更多嘗試次數，則之後每次重試都為 10 秒：

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
### 清理失敗的工作

當特定工作失敗時，你可能希望向你的使用者發送警報，或還原工作中部分完成的任何操作。為了實現這一點，你可以在工作類別上定義一個 `failed` 方法。導致工作失敗的 `Throwable` 實例將被傳遞給 `failed` 方法：

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
> 在調用 `failed` 方法之前，會實例化一個新的工作實例；因此，在 `handle` 方法中可能發生的任何類別屬性修改都將會遺失。

失敗的工作不一定是指遇到未處理例外狀況的工作。當工作耗盡了所有允許的嘗試次數時，也可能被視為失敗。這些嘗試次數可以透過以下幾種方式消耗：

<div class="content-list" markdown="1">

- 工作逾時。
- 工作在執行期間遇到未處理例外狀況。
- 工作被手動或透過中介層釋放回佇列中。

</div>

如果最後一次嘗試因工作執行期間拋出的例外狀況而失敗，該例外狀況將會傳遞給工作的 `failed` 方法。然而，如果工作因達到允許的最大嘗試次數而失敗，`$exception` 將是 `Illuminate\Queue\MaxAttemptsExceededException` 的實例。同樣地，如果工作因超過設定的逾時時間而失敗，`$exception` 將是 `Illuminate\Queue\TimeoutExceededException` 的實例。


<a name="retrying-failed-jobs"></a>
### 重試失敗的工作

要查看所有已插入到你的 `failed_jobs` 資料庫表格中的失敗工作，你可以使用 `queue:failed` Artisan 命令：

```shell
php artisan queue:failed
```

`queue:failed` 命令將列出工作 ID、連線、佇列、失敗時間以及其他有關工作的資訊。工作 ID 可以用來重試失敗的工作。例如，要重試一個 ID 為 `ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece` 的失敗工作，請執行以下命令：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece
```

如有必要，你可以向該命令傳遞多個 ID：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece 91401d2c-0784-4f43-824c-34f94a33c24d
```

你也可以重試特定佇列的所有失敗工作：

```shell
php artisan queue:retry --queue=name
```

要重試你所有的失敗工作，請執行 `queue:retry` 命令並將 `all` 作為 ID 傳遞：

```shell
php artisan queue:retry all
```

如果你想刪除一個失敗的工作，你可以使用 `queue:forget` 命令：

```shell
php artisan queue:forget 91401d2c-0784-4f43-824c-34f94a33c24d
```

> [!NOTE]
> 當使用 [Horizon](/docs/{{version}}/horizon) 時，你應該使用 `horizon:forget` 命令來刪除失敗的工作，而不是 `queue:forget` 命令。

要從 `failed_jobs` 表格中刪除你所有的失敗工作，你可以使用 `queue:flush` 命令：

```shell
php artisan queue:flush
```

`queue:flush` 命令會從佇列中移除所有失敗的工作記錄，無論失敗工作有多舊。你可以使用 `--hours` 選項只刪除在特定小時數之前失敗的工作。例如，以下命令將刪除所有在 48 小時前或更早插入的失敗工作記錄：

```shell
php artisan queue:flush --hours=48
```


<a name="ignoring-missing-models"></a>
### 忽略遺失的模型

將 Eloquent 模型注入工作中時，模型會在放置到佇列之前自動序列化，並在工作處理時從資料庫中重新檢索。然而，如果模型在工作等待工作者處理期間已被刪除，你的工作可能會因為 `ModelNotFoundException` 而失敗。

為方便起見，你可以選擇透過將工作的 `deleteWhenMissingModels` 屬性設定為 `true`，來自動刪除遺失模型的任務。當此屬性設定為 `true` 時，Laravel 將會悄悄地丟棄該工作，而不會引發例外狀況：

```php
/**
 * Delete the job if its models no longer exist.
 *
 * @var bool
 */
public $deleteWhenMissingModels = true;
```


<a name="pruning-failed-jobs"></a>
### 修剪失敗的工作

你可以透過調用 `queue:prune-failed` Artisan 命令來修剪應用程式 `failed_jobs` 表格中的記錄：

```shell
php artisan queue:prune-failed
```

預設情況下，所有超過 24 小時的失敗工作記錄將會被修剪。如果你為命令提供了 `--hours` 選項，則只會保留在過去 N 小時內插入的失敗工作記錄。例如，以下命令將刪除所有在 48 小時前或更早插入的失敗工作記錄：

```shell
php artisan queue:prune-failed --hours=48
```

<a name="storing-failed-jobs-in-dynamodb"></a>
### 在 DynamoDB 中儲存失敗的工作

Laravel 也支援將失敗的工作記錄儲存在 [DynamoDB](https://aws.amazon.com/dynamodb) 而非關聯式資料庫表格中。不過，您必須手動建立一個 DynamoDB 表格來儲存所有失敗的工作記錄。通常，此表格應命名為 `failed_jobs`，但您應該根據應用程式 `queue` 配置檔中的 `queue.failed.table` 配置值來命名表格。

`failed_jobs` 表格應該有一個名為 `application` 的字串主要分割區鍵，以及一個名為 `uuid` 的字串主要排序鍵。鍵的 `application` 部分將包含應用程式 `app` 配置檔中 `name` 配置值所定義的應用程式名稱。由於應用程式名稱是 DynamoDB 表格鍵的一部分，您可以使用同一個表格為多個 Laravel 應用程式儲存失敗的工作。

此外，請確保您已安裝 AWS SDK，以便您的 Laravel 應用程式可以與 Amazon DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

接下來，將 `queue.failed.driver` 配置選項的值設定為 `dynamodb`。此外，您應該在失敗工作配置陣列中定義 `key`、`secret` 和 `region` 配置選項。這些選項將用於 AWS 認證。當使用 `dynamodb` 驅動程式時，`queue.failed.database` 配置選項不再需要：

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
### 停用失敗的工作儲存

您可以指示 Laravel 捨棄失敗的工作而不儲存它們，方法是將 `queue.failed.driver` 配置選項的值設定為 `null`。通常，這可以透過 `QUEUE_FAILED_DRIVER` 環境變數來實現：

```ini
QUEUE_FAILED_DRIVER=null
```


<a name="failed-job-events"></a>
### 失敗的工作事件

如果您想註冊一個在工作失敗時將被呼叫的事件監聽器，可以使用 `Queue` Facade 的 `failing` 方法。例如，我們可以從 Laravel 內建的 `AppServiceProvider` 服務提供者的 `boot` 方法中將閉包附加到此事件：

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
## 從佇列清除工作

> [!NOTE]
> 當使用 [Horizon](/docs/{{version}}/horizon) 時，您應該使用 `horizon:clear` 命令從佇列中清除工作，而不是使用 `queue:clear` 命令。

如果您想刪除預設連線的預設佇列中的所有工作，可以使用 `queue:clear` Artisan 命令來完成：

```shell
php artisan queue:clear
```

您也可以提供 `connection` 引數和 `queue` 選項來刪除特定連線和佇列中的工作：

```shell
php artisan queue:clear redis --queue=emails
```

> [!WARNING]
> 從佇列清除工作僅適用於 SQS、Redis 和資料庫佇列驅動程式。此外，SQS 訊息刪除過程最長需要 60 秒，因此在您清除佇列後 60 秒內傳送至 SQS 佇列的工作也可能被刪除。

<a name="monitoring-your-queues"></a>
## 監控你的佇列

如果您的佇列突然湧入大量工作，可能會不堪重負，導致工作完成的等待時間變長。如果您願意，Laravel 可以在佇列工作數量超過指定閾值時向您發出警報。

首先，您應該將 `queue:monitor` 命令排程為 [每分鐘執行一次](/docs/{{version}}/scheduling)。此命令接受您希望監控的佇列名稱以及您希望的工作數量閾值：

```shell
php artisan queue:monitor redis:default,redis:deployments --max=100
```

單獨排程此命令並不足以觸發通知，提醒您佇列已不堪重負。當命令遇到工作數量超出閾值的佇列時，將會分派一個 `Illuminate\Queue\Events\QueueBusy` 事件。您可以在應用程式的 `AppServiceProvider` 中監聽此事件，以便向您或您的開發團隊傳送通知：

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

當測試分派工作的程式碼時，你可能希望指示 Laravel 不要實際執行工作本身，因為工作的程式碼可以獨立於分派它的程式碼進行測試。當然，要測試工作本身，你可以實例化一個工作實例，並直接在你的測試中呼叫 `handle` 方法。

你可以使用 `Queue` facade 的 `fake` 方法來防止佇列中的工作實際被推送到佇列。呼叫 `Queue` facade 的 `fake` 方法後，你可以斷言應用程式嘗試將工作推送到佇列：

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

你可以將閉包傳遞給 `assertPushed`、`assertNotPushed`、`assertClosurePushed` 或 `assertClosureNotPushed` 方法，以斷言已推送的工作通過了給定的「真實性測試」。如果至少有一個工作通過了給定的真實性測試，則斷言將成功：

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
### 偽造部分工作

如果你只需要偽造特定工作，同時允許其他工作正常執行，你可以將應偽造的工作類別名稱傳遞給 `fake` 方法：

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

你可以使用 `except` 方法來偽造所有工作，除了指定的工作集：

```php
Queue::fake()->except([
    ShipOrder::class,
]);
```


<a name="testing-job-chains"></a>
### 測試工作鏈

要測試工作鏈，你需要利用 `Bus` facade 的偽造功能。`Bus` facade 的 `assertChained` 方法可用於斷言已分派一個 [工作鏈](#job-chaining)。`assertChained` 方法接受一個工作鏈陣列作為其第一個參數：

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

如上例所示，工作鏈陣列可以是工作類別名稱陣列。但是，你也可以提供實際的工作實例陣列。這樣做時，Laravel 將確保工作實例與你的應用程式分派的工作鏈具有相同的類別和屬性值：

```php
Bus::assertChained([
    new ShipOrder,
    new RecordShipment,
    new UpdateInventory,
]);
```

你可以使用 `assertDispatchedWithoutChain` 方法來斷言工作已推送但沒有工作鏈：

```php
Bus::assertDispatchedWithoutChain(ShipOrder::class);
```


<a name="testing-chain-modifications"></a>
#### 測試鏈修改

如果鏈式工作 [預設或附加工作到現有鏈](#adding-jobs-to-the-chain)，你可以使用工作的 `assertHasChain` 方法來斷言工作具有預期的剩餘工作鏈：

```php
$job = new ProcessPodcast;

$job->handle();

$job->assertHasChain([
    new TranscribePodcast,
    new OptimizePodcast,
    new ReleasePodcast,
]);
```

`assertDoesntHaveChain` 方法可用於斷言工作剩餘的鏈為空：

```php
$job->assertDoesntHaveChain();
```


<a name="testing-chained-batches"></a>
#### 測試鏈式批次

如果你的工作鏈 [包含一批工作](#chains-and-batches)，你可以藉由在你的鏈斷言中插入 `Bus::chainedBatch` 定義來斷言鏈式批次符合你的預期：

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
### 測試工作批次

`Bus` facade 的 `assertBatched` 方法可用於斷言已分派一個 [工作批次](#job-batching)。傳遞給 `assertBatched` 方法的閉包會收到一個 `Illuminate\Bus\PendingBatch` 實例，該實例可用於檢查批次中的工作：

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

你可以使用 `assertBatchCount` 方法來斷言已分派了給定數量的批次：

```php
Bus::assertBatchCount(3);
```

你可以使用 `assertNothingBatched` 來斷言沒有分派任何批次：

```php
Bus::assertNothingBatched();
```


<a name="testing-job-batch-interaction"></a>
#### 測試工作/批次互動

此外，你可能偶爾需要測試個別工作與其底層批次的互動。例如，你可能需要測試工作是否取消了其批次的後續處理。為了達成此目的，你需要透過 `withFakeBatch` 方法為工作指派一個偽造的批次。`withFakeBatch` 方法返回一個包含工作實例和偽造批次的元組：

```php
[$job, $batch] = (new ShipOrder)->withFakeBatch();

$job->handle();

$this->assertTrue($batch->cancelled());
$this->assertEmpty($batch->added);
```

<a name="testing-job-queue-interactions"></a>
### 測試工作/佇列互動

有時候，您可能需要測試一個佇列中的工作[將自己釋放回佇列](#manually-releasing-a-job)。或者，您可能需要測試工作自行刪除。您可以透過實例化工作並呼叫 `withFakeQueueInteractions` 方法來測試這些佇列互動。

一旦工作與佇列的互動被偽造後，您可以呼叫工作上的 `handle` 方法。呼叫工作之後，各種斷言方法可用來驗證工作與佇列的互動：

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
## 工作事件

透過 `Queue` [facade](/docs/{{version}}/facades) 上的 `before` 和 `after` 方法，您可以指定在佇列工作處理之前或之後執行的回呼。這些回呼是執行額外記錄或增加儀表板統計數據的絕佳機會。通常，您應該從 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫這些方法。例如，我們可以使用 Laravel 內建的 `AppServiceProvider`：

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

透過 `Queue` [facade](/docs/{{version}}/facades) 上的 `looping` 方法，您可以指定在工作者嘗試從佇列中獲取工作之前執行的回呼。例如，您可以註冊一個閉包來回溯先前失敗的工作所留下的任何交易：

```php
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Queue;

Queue::looping(function () {
    while (DB::transactionLevel() > 0) {
        DB::rollBack();
    }
});
```