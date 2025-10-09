# 佇列

- [介紹](#introduction)
    - [連線 vs. 佇列](#connections-vs-queues)
    - [驅動程式注意事項與先決條件](#driver-prerequisites)
- [建立工作](#creating-jobs)
    - [產生工作類別](#generating-job-classes)
    - [類別結構](#class-structure)
    - [唯一工作](#unique-jobs)
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
    - [工作鏈結](#job-chaining)
    - [自訂佇列與連線](#customizing-the-queue-and-connection)
    - [指定最大工作嘗試次數/逾時值](#max-job-attempts-and-timeout)
    - [SQS FIFO 和公平佇列](#sqs-fifo-and-fair-queues)
    - [錯誤處理](#error-handling)
- [工作批次處理](#job-batching)
    - [定義可批次處理的工作](#defining-batchable-jobs)
    - [分派批次](#dispatching-batches)
    - [鏈結與批次](#chains-and-batches)
    - [將工作加入批次](#adding-jobs-to-batches)
    - [檢查批次](#inspecting-batches)
    - [取消批次](#cancelling-batches)
    - [批次失敗](#batch-failures)
    - [清除批次](#pruning-batches)
    - [將批次儲存至 DynamoDB](#storing-batches-in-dynamodb)
- [佇列閉包](#queueing-closures)
- [執行佇列工作者](#running-the-queue-worker)
    - [`queue:work` 命令](#the-queue-work-command)
    - [佇列優先權](#queue-priorities)
    - [佇列工作者與部署](#queue-workers-and-deployment)
    - [工作過期與逾時](#job-expirations-and-timeouts)
- [Supervisor 設定](#supervisor-configuration)
- [處理失敗的工作](#dealing-with-failed-jobs)
    - [清理失敗的工作](#cleaning-up-after-failed-jobs)
    - [重試失敗的工作](#retrying-failed-jobs)
    - [忽略遺失的模型](#ignoring-missing-models)
    - [清除失敗的工作](#pruning-failed-jobs)
    - [將失敗的工作儲存至 DynamoDB](#storing-failed-jobs-in-dynamodb)
    - [停用失敗工作儲存](#disabling-failed-job-storage)
    - [失敗工作事件](#failed-job-events)
- [清除佇列中的工作](#clearing-jobs-from-queues)
- [監控佇列](#monitoring-your-queues)
- [測試](#testing)
    - [偽造部分工作](#faking-a-subset-of-jobs)
    - [測試工作鏈結](#testing-job-chains)
    - [測試工作批次](#testing-job-batches)
    - [測試工作/佇列互動](#testing-job-queue-interactions)
- [工作事件](#job-events)

<a name="introduction"></a>
## 介紹

在建構您的 Web 應用程式時，您可能會遇到某些任務，例如解析並儲存上傳的 CSV 檔案，這些任務在一般的 Web 請求期間執行時間過長。幸好，Laravel 讓您可以輕鬆建立可在背景處理的佇列工作 (queued jobs)。透過將耗時的任務移至佇列，您的應用程式可以以極快的速度回應 Web 請求，並為客戶提供更好的使用者體驗。

Laravel 佇列為各種不同的佇列後端提供統一的佇列 API，例如 [Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io)，甚至是關聯式資料庫。

Laravel 的佇列設定選項儲存在您應用程式的 `config/queue.php` 設定檔中。在此檔案中，您將找到框架中包含的每個佇列驅動程式的連線設定，包括 database、[Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 和 [Beanstalkd](https://beanstalkd.github.io/) 驅動程式，以及一個會立即執行工作的同步驅動程式 (用於開發或測試期間)。也包含了一個 `null` 佇列驅動程式，它會捨棄佇列工作。

> [!NOTE]
> Laravel Horizon 是一個美觀的儀表板和設定系統，適用於您的 Redis 驅動佇列。請參閱完整的 [Horizon 文件](/docs/{{version}}/horizon)以獲取更多資訊。


<a name="connections-vs-queues"></a>
### 連線 vs. 佇列

在開始使用 Laravel 佇列之前，了解「連線 (connections)」與「佇列 (queues)」之間的區別非常重要。在您的 `config/queue.php` 設定檔中，有一個 `connections` 設定陣列。此選項定義了與後端佇列服務 (例如 Amazon SQS、Beanstalk 或 Redis) 的連線。然而，任何一個佇列連線都可能有多個「佇列」，這些佇列可以被視為不同堆疊或堆積的佇列工作。

請注意，`queue` 設定檔中的每個連線設定範例都包含一個 `queue` 屬性。這是當工作被傳送到給定連線時，會被分派到的預設佇列。換句話說，如果您分派一個工作而未明確定義應分派到哪個佇列，該工作將會被放置在連線設定的 `queue` 屬性中定義的佇列上：

```php
use App\Jobs\ProcessPodcast;

// This job is sent to the default connection's default queue...
ProcessPodcast::dispatch();

// This job is sent to the default connection's "emails" queue...
ProcessPodcast::dispatch()->onQueue('emails');
```

有些應用程式可能不需要將工作推送到多個佇列，而是偏好只有一個簡單的佇列。然而，將工作推送到多個佇列對於希望對工作處理方式進行優先排序或區分的應用程式來說，可能特別有用，因為 Laravel 佇列工作者允許您根據優先順序指定要處理哪些佇列。例如，如果您將工作推送到 `high` 佇列，您可以執行一個工作者，讓它們獲得更高的處理優先權：

```shell
php artisan queue:work --queue=high,default
```


<a name="driver-prerequisites"></a>
### 驅動程式注意事項與先決條件


<a name="database"></a>
#### 資料庫

為了使用 `database` 佇列驅動程式，您將需要一個資料庫表格來儲存工作。通常，這會包含在 Laravel 預設的 `0001_01_01_000002_create_jobs_table.php` [資料庫遷移](/docs/{{version}}/migrations)中；然而，如果您的應用程式不包含此遷移，您可以使用 `make:queue-table` Artisan 命令來建立它：

```shell
php artisan make:queue-table

php artisan migrate
```


<a name="redis"></a>
#### Redis

為了使用 `redis` 佇列驅動程式，您應該在 `config/database.php` 設定檔中配置一個 Redis 資料庫連線。

> [!WARNING]
> `serializer` 和 `compression` Redis 選項不支援 `redis` 佇列驅動程式。


<a name="redis-cluster"></a>
##### Redis 叢集

如果您的 Redis 佇列連線使用 [Redis Cluster](https://redis.io/docs/latest/operate/rs/databases/durability-ha/clustering)，您的佇列名稱必須包含 [key hash tag](https://redis.io/docs/latest/develop/using-commands/keyspace/#hashtags)。這是為了確保給定佇列的所有 Redis 鍵都放置在相同的 hash slot 中所必需的：

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
##### 阻斷

當使用 Redis 佇列時，您可以使用 `block_for` 設定選項來指定驅動程式應等待工作變得可用多久，然後再迭代工作者循環並重新輪詢 Redis 資料庫。

根據您的佇列負載調整此值，比持續輪詢 Redis 資料庫以獲取新工作更有效率。例如，您可以將此值設定為 `5`，以指示驅動程式應在等待工作變得可用時阻斷五秒鐘：

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
> 將 `block_for` 設定為 `0` 將會導致佇列工作者無限期阻斷，直到有工作可用為止。這也會阻止處理諸如 `SIGTERM` 之類的信號，直到下一個工作被處理為止。


<a name="other-driver-prerequisites"></a>
#### 其他驅動程式先決條件

以下列出的佇列驅動程式需要下列依賴項。這些依賴項可以透過 Composer 套件管理器安裝：

<div class="content-list" markdown="1">

- Amazon SQS: `aws/aws-sdk-php ~3.0`
- Beanstalkd: `pda/pheanstalk ~5.0`
- Redis: `predis/predis ~2.0` 或 phpredis PHP 擴充功能
- [MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/queues/): `mongodb/laravel-mongodb`

</div>

<a name="creating-jobs"></a>
## 建立工作

<a name="generating-job-classes"></a>
### 產生工作類別

預設情況下，應用程式中所有可佇列的工作皆儲存於 `app/Jobs` 目錄中。如果 `app/Jobs` 目錄不存在，則在您執行 `make:job` Artisan 命令時將會自動建立：

```shell
php artisan make:job ProcessPodcast
```

所產生的類別將會實作 `Illuminate\Contracts\Queue\ShouldQueue` 介面，以向 Laravel 指示此工作應被推送到佇列中非同步執行。

> [!NOTE]
> 工作 stub 可以透過 [stub 發佈](/docs/{{version}}/artisan#stub-customization)進行客製化。

<a name="class-structure"></a>
### 類別結構

工作類別非常簡單，通常只包含一個 `handle` 方法，該方法在工作由佇列處理時會被調用。首先，讓我們看看一個範例工作類別。在這個範例中，我們假設管理一個 Podcast 發佈服務，需要在 Podcast 檔案發佈之前處理這些上傳的檔案：

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

在此範例中，請注意我們能夠將一個 [Eloquent 模型](/docs/{{version}}/eloquent)直接傳遞給佇列工作的建構式。由於工作使用了 `Queueable` Trait，Eloquent 模型及其載入的關聯在工作處理時將會被妥善地序列化和反序列化。

如果您的佇列工作在其建構式中接受一個 Eloquent 模型，則只有該模型的識別碼會被序列化到佇列中。當工作實際被處理時，佇列系統將自動從資料庫中重新擷取完整的模型實例及其載入的關聯。這種模型序列化方法允許將更小的工作 Payload 發送到佇列驅動程式。

<a name="handle-method-dependency-injection"></a>
#### `handle` 方法的依賴注入

當工作由佇列處理時，`handle` 方法會被調用。請注意，我們可以在工作的 `handle` 方法上型別提示依賴。Laravel [服務容器](/docs/{{version}}/container)會自動注入這些依賴。

如果您想完全控制容器如何將依賴注入到 `handle` 方法中，您可以使用容器的 `bindMethod` 方法。`bindMethod` 方法接受一個回呼 (callback)，該回呼會接收到工作和容器。在回呼中，您可以隨意調用 `handle` 方法。通常，您應該從 `App\Providers\AppServiceProvider` [服務提供者](/docs/{{version}}/providers)的 `boot` 方法中調用此方法：

```php
use App\Jobs\ProcessPodcast;
use App\Services\AudioProcessor;
use Illuminate\Contracts\Foundation\Application;

$this->app->bindMethod([ProcessPodcast::class, 'handle'], function (ProcessPodcast $job, Application $app) {
    return $job->handle($app->make(AudioProcessor::class));
});
```

> [!WARNING]
> 二進位資料，例如原始圖像內容，應在傳遞給佇列工作之前透過 `base64_encode` 函數進行處理。否則，工作在放入佇列時可能無法正確序列化為 JSON。

<a name="handling-relationships"></a>
#### 佇列中的關聯

由於所有已載入的 Eloquent 模型關聯在工作被佇列時也會被序列化，因此序列化的工作字串有時會變得非常大。此外，當工作被反序列化且模型關聯從資料庫中重新擷取時，它們將會被完整地擷取。在模型序列化到佇列之前所應用的任何先前的關聯約束，在工作反序列化時將不會被應用。因此，如果您希望處理給定關聯的子集，則應在佇列工作中重新約束該關聯。

或者，為防止關聯被序列化，您可以在設定屬性值時，在模型上呼叫 `withoutRelations` 方法。此方法將返回一個沒有載入關聯的模型實例：

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

如果您正在使用 [PHP constructor property promotion](https://www.php.net/manual/en/language.oop5.decon.php#language.oop5.decon.constructor.promotion) 並希望指示 Eloquent 模型不應序列化其關聯，您可以使用 `WithoutRelations` 屬性：

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

為方便起見，如果您希望序列化所有不帶關聯的模型，您可以將 `WithoutRelations` 屬性應用於整個類別，而不是將屬性應用於每個模型：

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

如果一個工作接收的是 Eloquent 模型的集合或陣列，而不是單一模型，則該集合中的模型在工作反序列化和執行時將不會恢復其關聯。這是為了防止處理大量模型的工作造成過多的資源使用。

<a name="unique-jobs"></a>
### 唯一工作

> [!WARNING]
> 唯一工作需要支援 [鎖定](/docs/{{version}}/cache#atomic-locks) 的快取驅動程式。目前，`memcached`、`redis`、`dynamodb`、`database`、`file` 和 `array` 快取驅動程式支援原子鎖定。

> [!WARNING]
> 唯一工作約束不適用於批次內的工作。

有時，您可能希望確保在任何時間點上，佇列中只有一個特定工作的實例。您可以透過在工作類別上實作 `ShouldBeUnique` 介面來實現此目的。此介面不要求您在類別上定義任何額外的方法：

```php
<?php

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Queue\ShouldBeUnique;

class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    // ...
}
```

在上述範例中，`UpdateSearchIndex` 工作是唯一的。因此，如果佇列中已經有該工作的另一個實例且尚未完成處理，則該工作將不會被分派。

在某些情況下，您可能希望定義一個使其成為唯一的「鍵」，或者您可能希望指定一個逾時時間，超過該時間後工作就不再是唯一的。為了實現此目的，您可以在工作類別上定義 `uniqueId` 和 `uniqueFor` 屬性或方法：

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

在上述範例中，`UpdateSearchIndex` 工作透過產品 ID 保持唯一。因此，任何具有相同產品 ID 的新工作分派都將被忽略，直到現有工作完成處理為止。此外，如果現有工作在一小時內未被處理，則唯一鎖定將被釋放，並且另一個具有相同唯一鍵的工作可以被分派到佇列中。

> [!WARNING]
> 如果您的應用程式從多個網頁伺服器或容器分派工作，您應該確保所有伺服器都與同一個中央快取伺服器通訊，以便 Laravel 能夠準確判斷工作是否唯一。

<a name="keeping-jobs-unique-until-processing-begins"></a>
#### 在處理開始前保持工作唯一

預設情況下，唯一工作在完成處理或所有重試嘗試失敗後會「解鎖」。然而，在某些情況下，您可能希望工作在處理之前立即解鎖。為此，您的工作應該實作 `ShouldBeUniqueUntilProcessing` 契約，而非 `ShouldBeUnique` 契約：

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
#### 唯一工作鎖定

在幕後，當 `ShouldBeUnique` 工作被分派時，Laravel 會嘗試使用 `uniqueId` 鍵取得一個 [鎖定](/docs/{{version}}/cache#atomic-locks)。如果鎖定已被持有，則工作不會被分派。當工作完成處理或所有重試嘗試失敗時，此鎖定會被釋放。預設情況下，Laravel 將使用預設快取驅動程式來取得此鎖定。然而，如果您希望使用另一個驅動程式來取得鎖定，您可以定義一個 `uniqueVia` 方法，該方法會回傳應該使用的快取驅動程式：

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
> 如果您只需要限制工作的並行處理，請改用 [WithoutOverlapping](/docs/{{version}}/queues#preventing-job-overlaps) 工作中介層。

<a name="encrypted-jobs"></a>
### 加密工作

Laravel 允許您透過 [加密](/docs/{{version}}/encryption) 來確保工作資料的隱私和完整性。首先，只需將 `ShouldBeEncrypted` 介面新增到工作類別中。一旦此介面被新增到類別中，Laravel 將在將您的工作推送到佇列之前自動加密它：

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

工作中介層允許您在排隊工作的執行周圍包裝自訂邏輯，減少工作本身的樣板程式碼。例如，考慮以下 `handle` 方法，它利用 Laravel 的 Redis 速率限制功能，允許每五秒只處理一個工作：

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

雖然這段程式碼是有效的，但 `handle` 方法的實作變得嘈雜，因為它充滿了 Redis 速率限制邏輯。此外，此速率限制邏輯必須複製到任何其他我們希望進行速率限制的工作。我們可以在工作本身定義一個處理速率限制的工作中介層，而不是在 handle 方法中進行速率限制：

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

如您所見，如同 [路由中介層](/docs/{{version}}/middleware) 一樣，工作中介層會接收正在處理的工作以及一個應調用的回呼，以繼續處理該工作。

您可以使用 `make:job-middleware` Artisan 命令來產生一個新的工作中介層類別。建立工作中介層後，可以透過從工作的 `middleware` 方法回傳它們來將其附加到工作。此方法在透過 `make:job` Artisan 命令生成的職務中不存在，因此您需要手動將其新增到您的工作類別中：

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
> 工作中介層也可以指派給 [可佇列的事件監聽器](/docs/{{version}}/events#queued-event-listeners)、[可郵寄項目](/docs/{{version}}/mail#queueing-mail) 和 [通知](/docs/{{version}}/notifications#queueing-notifications)。

<a name="rate-limiting"></a>
### 速率限制

儘管我們剛才展示了如何編寫自己的速率限制工作中介層，但 Laravel 實際上包含了一個速率限制中介層，您可以用來限制工作的速率。如同 [路由速率限制器](/docs/{{version}}/routing#defining-rate-limiters) 一樣，工作速率限制器是使用 `RateLimiter` Facade 的 `for` 方法定義的。

例如，您可能希望允許使用者每小時備份一次資料，同時對進階客戶不施加此類限制。為了實現這一點，您可以在 `AppServiceProvider` 的 `boot` 方法中定義一個 `RateLimiter`：

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

在上面的範例中，我們定義了每小時的速率限制；但是，您可以使用 `perMinute` 方法輕鬆定義基於分鐘的速率限制。此外，您可以將任何您想要的值傳遞給速率限制的 `by` 方法；但是，這個值最常用於按客戶區分速率限制：

```php
return Limit::perMinute(50)->by($job->user->id);
```

一旦您定義了速率限制，您就可以使用 `Illuminate\Queue\Middleware\RateLimited` 中介層將速率限制器附加到您的工作。每當工作超出速率限制時，此中介層將根據速率限制持續時間以適當的延遲將工作釋放回佇列：

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

將受速率限制的工作釋放回佇列仍會增加工作總 `attempts` 數。您可能希望相應地調整工作類別中的 `tries` 和 `maxExceptions` 屬性。或者，您可能希望使用 [retryUntil 方法](#time-based-attempts) 來定義工作不再嘗試的時間量。

使用 `releaseAfter` 方法，您還可以指定在再次嘗試釋放的工作之前必須經過的秒數：

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

如果您不希望在工作受速率限制時重試，您可以使用 `dontRelease` 方法：

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
> 如果您正在使用 Redis，您可以使用 `Illuminate\Queue\Middleware\RateLimitedWithRedis` 中介層，它針對 Redis 進行了優化，比基本速率限制中介層更高效。

<a name="preventing-job-overlaps"></a>
### 防止工作重疊

Laravel 包含 `Illuminate\Queue\Middleware\WithoutOverlapping` 中介層，它允許您根據任意鍵值防止工作重疊。當佇列工作正在修改一個一次只能由一個工作修改的資源時，這會很有幫助。

例如，假設您有一個佇列工作，它會更新使用者的信用分數，並且您希望防止相同使用者 ID 的信用分數更新工作重疊。要實現這一點，您可以從工作的 `middleware` 方法中返回 `WithoutOverlapping` 中介層：

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

將重疊的工作釋放回佇列仍會增加工作的總嘗試次數。您可能需要相應地調整工作類別中的 `tries` 和 `maxExceptions` 屬性。例如，將 `tries` 屬性保留為預設值 1 將會阻止任何重疊的工作稍後被重試。

任何相同類型的重疊工作都將被釋放回佇列。您也可以指定在釋放的工作再次嘗試之前必須經過的秒數：

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

如果您希望立即刪除任何重疊的工作，使其不會被重試，您可以使用 `dontRelease` 方法：

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

`WithoutOverlapping` 中介層由 Laravel 的原子鎖定功能提供支援。有時，您的工作可能會意外失敗或逾時，導致鎖定未被釋放。因此，您可以透過 `expireAfter` 方法明確定義鎖定過期時間。例如，以下範例將指示 Laravel 在工作開始處理後三分鐘釋放 `WithoutOverlapping` 鎖定：

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
#### 跨工作類別共用鎖定鍵值

預設情況下，`WithoutOverlapping` 中介層只會防止相同類別的重疊工作。因此，儘管兩個不同的工作類別可能使用相同的鎖定鍵值，但它們不會被防止重疊。然而，您可以指示 Laravel 使用 `shared` 方法將鍵值應用於跨工作類別：

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

Laravel 包含一個 `Illuminate\Queue\Middleware\ThrottlesExceptions` 中介層，讓您可以對例外進行節流。一旦工作拋出指定數量的例外，所有後續執行工作的嘗試將被延遲，直到經過指定的時間間隔。此中介層對於與不穩定的第三方服務互動的工作特別有用。

舉例來說，假設一個佇列中的工作與開始拋出例外的第三方 API 互動。若要對例外進行節流，您可以從工作的 `middleware` 方法中回傳 `ThrottlesExceptions` 中介層。通常，此中介層應與實作 [基於時間的嘗試](#time-based-attempts) 的工作配對使用：

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

此中介層接受的第一個建構子參數是工作在被節流之前可以拋出的例外數量，而第二個建構子參數是工作一旦被節流後，在再次嘗試之前應該經過的秒數。在上述程式碼範例中，如果工作連續拋出 10 個例外，我們將等待 5 分鐘後再嘗試該工作，並受 30 分鐘的時間限制。

當工作拋出例外但尚未達到例外閾值時，該工作通常會立即重試。不過，您可以透過在將中介層附加到工作時呼叫 `backoff` 方法，來指定此類工作應延遲的分鐘數：

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

在內部，此中介層使用 Laravel 的快取系統來實作速率限制，並且將工作的類別名稱用作快取「key」。您可以透過在將中介層附加到工作時呼叫 `by` 方法來覆寫此 key。如果您有多個工作與相同的第三方服務互動，並且希望它們共享一個共同的節流「bucket」，確保它們遵守單一的共享限制，這將會很有用：

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

依預設，此中介層將對每個例外進行節流。您可以透過在將中介層附加到工作時呼叫 `when` 方法來修改此行為。只有在提供給 `when` 方法的閉包回傳 `true` 時，該例外才會被節流：

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

與 `when` 方法（該方法將工作釋放回佇列或拋出例外）不同，`deleteWhen` 方法允許您在給定例外發生時完全刪除該工作：

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

如果您希望將節流的例外回報給應用程式的例外處理器，您可以透過在將中介層附加到工作時呼叫 `report` 方法來實現。或者，您可以向 `report` 方法提供一個閉包，只有當給定閉包回傳 `true` 時，該例外才會被回報：

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
> 如果您正在使用 Redis，您可以使用 `Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis` 中介層，它專為 Redis 調整，比基本例外節流中介層更有效率。

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

一旦您撰寫完工作類別，您就可以使用工作本身的 `dispatch` 方法來分派它。傳遞給 `dispatch` 方法的引數將會傳遞給工作的建構式：

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

如果您想條件性地分派工作，可以使用 `dispatchIf` 和 `dispatchUnless` 方法：

```php
ProcessPodcast::dispatchIf($accountActive, $podcast);

ProcessPodcast::dispatchUnless($accountSuspended, $podcast);
```

在新的 Laravel 應用程式中，`database` 驅動程式是預設的佇列驅動程式。您可以在應用程式的 `config/queue.php` 設定檔中指定不同的佇列驅動程式。

<a name="delayed-dispatching"></a>
### 延遲分派

如果您想指定工作不應立即供佇列工作者處理，您可以在分派工作時使用 `delay` 方法。例如，我們指定工作應在分派後 10 分鐘才可供處理：

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

在某些情況下，工作可能已設定預設延遲。如果您需要繞過此延遲並分派工作以立即處理，您可以使用 `withoutDelay` 方法：

```php
ProcessPodcast::dispatch($podcast)->withoutDelay();
```

> [!WARNING]
> Amazon SQS 佇列服務的最大延遲時間為 15 分鐘。

<a name="dispatching-after-the-response-is-sent-to-browser"></a>
#### 響應傳送到瀏覽器後分派

另外，如果您的網頁伺服器正在使用 [FastCGI](https://www.php.net/manual/en/install.fpm.php)，`dispatchAfterResponse` 方法會延遲分派工作，直到 HTTP 響應傳送到使用者的瀏覽器之後。這仍然允許使用者開始使用應用程式，即使佇列工作仍在執行。這通常只應用於大約需要一秒鐘的工作，例如寄送電子郵件。由於這些工作在目前的 HTTP 請求中處理，因此以這種方式分派的工作不需要佇列工作者運行即可處理：

```php
use App\Jobs\SendNotification;

SendNotification::dispatchAfterResponse();
```

您也可以 `dispatch` 閉包並將 `afterResponse` 方法鏈結到 [dispatch 輔助函式](/docs/{{version}}/helpers#method-dispatch)，以在 HTTP 響應傳送到瀏覽器後執行閉包：

```php
use App\Mail\WelcomeMessage;
use Illuminate\Support\Facades\Mail;

dispatch(function () {
    Mail::to('taylor@example.com')->send(new WelcomeMessage);
})->afterResponse();
```

<a name="synchronous-dispatching"></a>
### 同步分派

如果您想立即分派工作 (同步)，您可以使用 `dispatchSync` 方法。當使用此方法時，工作將不會排入佇列，而會在目前的程序中立即執行：

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
### 工作與資料庫交易

雖然在資料庫交易內分派工作是完全沒問題的，但您應特別注意確保您的工作能夠實際成功執行。當在交易內分派工作時，工作可能會在父交易提交之前被工作者處理。當發生這種情況時，您在資料庫交易期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫記錄可能不存在於資料庫中。

幸好，Laravel 提供了幾種解決此問題的方法。首先，您可以在佇列連線的設定陣列中設定 `after_commit` 連線選項：

```php
'redis' => [
    'driver' => 'redis',
    // ...
    'after_commit' => true,
],
```

當 `after_commit` 選項為 `true` 時，您可以在資料庫交易內分派工作；然而，Laravel 將會等待開啟中的父資料庫交易提交後，才會實際分派工作。當然，如果目前沒有開啟的資料庫交易，工作將會立即分派。

如果交易因為期間發生的例外而回溯，在該交易期間分派的工作將會被捨棄。

> [!NOTE]
> 將 `after_commit` 設定選項設為 `true` 也會導致任何排入佇列的事件監聽器、郵件、通知和廣播事件在所有開啟的資料庫交易提交後分派。

<a name="specifying-commit-dispatch-behavior-inline"></a>
#### 行內指定提交分派行為

如果您沒有將 `after_commit` 佇列連線設定選項設為 `true`，您仍然可以指示特定工作應在所有開啟的資料庫交易提交後分派。為此，您可以將 `afterCommit` 方法鏈結到您的分派操作上：

```php
use App\Jobs\ProcessPodcast;

ProcessPodcast::dispatch($podcast)->afterCommit();
```

同樣地，如果 `after_commit` 設定選項設為 `true`，您可以指示特定工作應立即分派，無需等待任何開啟的資料庫交易提交：

```php
ProcessPodcast::dispatch($podcast)->beforeCommit();
```

<a name="job-chaining"></a>
### 工作鏈結

工作鏈結讓您可以指定一系列佇列工作，這些工作將在主要工作成功執行後依序運行。如果序列中的某個工作失敗，其餘的工作將不會被執行。要執行佇列工作鏈結，您可以使用 `Bus` Facade 提供的 `chain` 方法。Laravel 的命令匯流排是一個較低階的元件，佇列工作分派就是建立在其之上：

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

除了鏈結工作類別實例之外，您也可以鏈結閉包：

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
> 在工作中使用 `$this->delete()` 方法刪除工作並不會阻止鏈結中的工作繼續處理。鏈結只會在鏈結中的某個工作失敗時才會停止執行。

<a name="chain-connection-queue"></a>
#### 鏈結連線與佇列

如果您想為鏈結工作指定要使用的連線和佇列，可以使用 `onConnection` 和 `onQueue` 方法。這些方法會指定要使用的佇列連線和佇列名稱，除非佇列工作明確指定了不同的連線/佇列：

```php
Bus::chain([
    new ProcessPodcast,
    new OptimizePodcast,
    new ReleasePodcast,
])->onConnection('redis')->onQueue('podcasts')->dispatch();
```

<a name="adding-jobs-to-the-chain"></a>
#### 將工作加入鏈結

有時，您可能需要在現有的工作鏈結中，從該鏈結中的另一個工作內部，在工作之前加入或之後附加一個工作。您可以使用 `prependToChain` 和 `appendToChain` 方法來完成此操作：

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

當鏈結工作時，您可以使用 `catch` 方法來指定一個閉包，如果鏈結中的工作失敗，該閉包將被呼叫。指定的回呼將收到導致工作失敗的 `Throwable` 實例：

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
> 由於鏈結回呼會被序列化並稍後由 Laravel 佇列執行，因此您不應在鏈結回呼中使用 `$this` 變數。

<a name="customizing-the-queue-and-connection"></a>
### 自訂佇列與連線

<a name="dispatching-to-a-particular-queue"></a>
#### 分派到特定佇列

透過將工作推送到不同的佇列，您可以「分類」您的佇列工作，甚至可以根據您分配給不同佇列的工作者數量來設定優先順序。請記住，這不是將工作推送到佇列設定檔中定義的不同佇列「連線」，而是只推送到單一連線中的特定佇列。要指定佇列，請在分派工作時使用 `onQueue` 方法：

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

此外，您也可以在工作的建構式中呼叫 `onQueue` 方法來指定工作的佇列：

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

您可以將 `onConnection` 和 `onQueue` 方法鏈結在一起，為工作指定連線和佇列：

```php
ProcessPodcast::dispatch($podcast)
    ->onConnection('sqs')
    ->onQueue('processing');
```

此外，您也可以在工作的建構式中呼叫 `onConnection` 方法來指定工作的連線：

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
### 指定最大工作嘗試次數/逾時值

<a name="max-attempts"></a>
#### 最大嘗試次數

工作嘗試次數是 Laravel 佇列系統的核心概念，並驅動許多進階功能。儘管一開始可能令人困惑，但在修改預設設定之前，了解它們的運作方式非常重要。

當一個工作被分派時，它會被推送到佇列中。然後工作者會接收它並嘗試執行。這就是一次工作嘗試。

然而，一次嘗試不一定表示工作者的 `handle` 方法已經被執行。嘗試次數也可能以多種方式「被消耗」：

<div class="content-list" markdown="1">

- 工作在執行期間遇到未處理的例外。
- 工作使用 `$this->release()` 手動釋放回佇列。
- `WithoutOverlapping` 或 `RateLimited` 等中介層未能取得鎖定並釋放工作。
- 工作逾時。
- 工作的 `handle` 方法執行並完成，沒有拋出例外。

</div>

你可能不希望無限期地嘗試一個工作。因此，Laravel 提供了多種方式來指定一個工作可以嘗試多少次或嘗試多久。

> [!NOTE]
> 預設情況下，Laravel 只會嘗試一個工作一次。如果你的工作使用 `WithoutOverlapping` 或 `RateLimited` 等中介層，或者你手動釋放工作，你可能需要透過 `tries` 選項來增加允許的嘗試次數。

指定一個工作最大嘗試次數的一種方法是透過 Artisan 命令列上的 `--tries` 開關。這將適用於工作者處理的所有工作，除非正在處理的工作本身指定了它可以嘗試的次數：

```shell
php artisan queue:work --tries=3
```

如果一個工作超出其最大嘗試次數，它將被視為「失敗」的工作。有關處理失敗工作的更多資訊，請參閱 [失敗工作文件](#dealing-with-failed-jobs)。如果提供 `--tries=0` 給 `queue:work` 命令，該工作將會無限期地重試。

你可以透過在工作類別本身定義最大嘗試次數來採取更細緻的方法。如果工作上指定了最大嘗試次數，它將優先於命令列上提供的 `--tries` 值：

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
#### 基於時間的嘗試次數

除了定義一個工作在失敗之前可以嘗試多少次之外，你還可以定義一個工作不再應該被嘗試的時間。這允許一個工作在給定的時間範圍內被嘗試任意次數。若要定義一個工作不再應該被嘗試的時間，請在你的工作類別中添加 `retryUntil` 方法。此方法應返回一個 `DateTime` 實例：

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
> 你也可以在你的 [佇列事件監聽器](/docs/{{version}}/events#queued-event-listeners) 和 [佇列通知](/docs/{{version}}/notifications#queueing-notifications) 上定義 `tries` 屬性或 `retryUntil` 方法。

<a name="max-exceptions"></a>
#### 最大例外

有時你可能希望指定一個工作可以嘗試多次，但如果重試是由給定數量的未處理例外觸發（而不是直接由 `release` 方法釋放），則該工作應失敗。若要實現此目的，你可以在你的工作類別上定義一個 `maxExceptions` 屬性：

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

在此範例中，如果應用程式無法取得 Redis 鎖定，該工作將被釋放十秒，並將繼續重試最多 25 次。然而，如果工作拋出三個未處理的例外，則該工作將失敗。

<a name="timeout"></a>
#### 逾時

通常，你大約知道你的佇列工作預計需要多長時間。因此，Laravel 允許你指定一個「逾時」值。預設情況下，逾時值為 60 秒。如果一個工作處理時間超過逾時值指定秒數，處理該工作的工作者將會以錯誤結束。通常，工作者將由 [伺服器上配置的程序管理器](#supervisor-configuration) 自動重新啟動。

工作可以運行的最大秒數可以使用 Artisan 命令列上的 `--timeout` 開關來指定：

```shell
php artisan queue:work --timeout=30
```

如果工作因持續逾時而超出其最大嘗試次數，它將被標記為失敗。

你也可以在工作類別本身定義一個工作應允許運行的最大秒數。如果工作上指定了逾時，它將優先於命令列上指定的任何逾時：

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

有時，IO 阻斷程序，例如 sockets 或對外 HTTP 連線，可能不會遵守你指定的逾時。因此，在使用這些功能時，你應該始終嘗試也使用它們的 API 來指定逾時。例如，在使用 [Guzzle](https://docs.guzzlephp.org) 時，你應該始終指定連線和請求逾時值。

> [!WARNING]
> 必須安裝 [PCNTL](https://www.php.net/manual/en/book.pcntl.php) PHP 擴充功能才能指定工作逾時。此外，工作的「timeout」值應該始終小於其 [「retry after」](#job-expiration) 值。否則，工作可能會在實際完成執行或逾時之前被重新嘗試。

<a name="failing-on-timeout"></a>
#### 逾時失敗

如果你希望指示一個工作在逾時時應被標記為 [失敗](#dealing-with-failed-jobs)，你可以在工作類別上定義 `$failOnTimeout` 屬性：

```php
/**
 * Indicate if the job should be marked as failed on timeout.
 *
 * @var bool
 */
public $failOnTimeout = true;
```

> [!NOTE]
> 預設情況下，當一個工作逾時時，它會消耗一次嘗試並被釋放回佇列（如果允許重試）。但是，如果你將工作設定為逾時失敗，則無論嘗試次數設定為何，它都不會被重試。

<a name="sqs-fifo-and-fair-queues"></a>
### SQS FIFO 和公平佇列

Laravel 支援 [Amazon SQS FIFO (先進先出)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html) 佇列，讓您可以依其發送的確切順序處理工作，同時透過訊息去重複確保僅處理一次。

FIFO 佇列需要訊息群組 ID 來決定哪些工作可以平行處理。具有相同群組 ID 的工作會循序處理，而具有不同群組 ID 的訊息則可以同時處理。

Laravel 提供流暢的 `onGroup` 方法，用於在分派工作時指定訊息群組 ID：

```php
ProcessOrder::dispatch($order)
    ->onGroup("customer-{$order->customer_id}");
```

SQS FIFO 佇列支援訊息去重複以確保僅處理一次。請在您的工作類別中實作 `deduplicationId` 方法來提供自訂的去重複 ID：

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

在使用 FIFO 佇列時，您還需要在監聽器、郵件和通知上定義訊息群組。或者，您可以將這些物件的佇列實例分派到非 FIFO 佇列。

若要為 [佇列事件監聽器](/docs/{{version}}/events#queued-event-listeners) 定義訊息群組，請在監聽器上定義 `messageGroup` 方法。您也可以選擇性地定義 `deduplicationId` 方法：

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

當發送將佇列到 FIFO 佇列的 [郵件訊息](/docs/{{version}}/mail) 時，您應該在發送此郵件時呼叫 `onGroup` 方法，並選擇性地呼叫 `withDeduplicator` 方法：

```php
use App\Mail\InvoicePaid;
use Illuminate\Support\Facades\Mail;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

Mail::to($request->user())->send($invoicePaid);
```

當發送將佇列到 FIFO 佇列的 [通知](/docs/{{version}}/notifications) 時，您應該在發送此通知時呼叫 `onGroup` 方法，並選擇性地呼叫 `withDeduplicator` 方法：

```php
use App\Notifications\InvoicePaid;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

$user->notify($invoicePaid);
```

<a name="error-handling"></a>
### 錯誤處理

如果在處理工作時拋出例外，該工作將自動重新釋放到佇列中，以便可以再次嘗試。該工作將繼續重新釋放，直到達到應用程式允許的最大嘗試次數為止。最大嘗試次數由 `queue:work` Artisan 命令上使用的 `--tries` 選項定義。或者，最大嘗試次數可以在工作類別本身上定義。有關執行佇列工作者的更多資訊，請參閱 [下方](#running-the-queue-worker)。

<a name="manually-releasing-a-job"></a>
#### 手動釋放工作

有時您可能希望手動將工作釋放回佇列中，以便稍後可以再次嘗試。您可以透過呼叫 `release` 方法來達成此目的：

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

預設情況下，`release` 方法會將工作釋放回佇列中以立即處理。但是，您可以透過向 `release` 方法傳遞一個整數或日期實例，指示佇列在經過指定秒數後才讓工作可供處理：

```php
$this->release(10);

$this->release(now()->addSeconds(10));
```

<a name="manually-failing-a-job"></a>
#### 手動將工作標記為失敗

有時您可能需要手動將工作標記為「失敗」。為此，您可以呼叫 `fail` 方法：

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

如果您想因為捕獲到的例外而將工作標記為失敗，您可以將例外傳遞給 `fail` 方法。或者，為了方便，您可以傳遞一個字串錯誤訊息，它將為您轉換為例外：

```php
$this->fail($exception);

$this->fail('Something went wrong.');
```

> [!NOTE]
> 有關失敗工作的更多資訊，請查閱 [處理工作失敗的說明文件](#dealing-with-failed-jobs)。

<a name="fail-jobs-on-exceptions"></a>
#### 在特定例外情況下使工作失敗

`FailOnException` [工作中介層](#job-middleware) 允許您在拋出特定例外時短路重試。這允許在暫時性例外（例如外部 API 錯誤）時重試，但在持久性例外（例如使用者權限被撤銷）時永久性地使工作失敗：

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

Laravel 的工作批次處理功能讓您可以輕鬆執行一批工作，並在該批次工作完成執行後執行某些動作。開始之前，您應該先建立一個資料庫遷移，以建立一個包含批次工作中繼資料 (例如完成百分比) 的表格。這個遷移可以使用 `make:queue-batches-table` Artisan 指令來產生：

```shell
php artisan make:queue-batches-table

php artisan migrate
```

<a name="defining-batchable-jobs"></a>
### 定義可批次處理的工作

要定義一個可批次處理的工作，您應該像往常一樣[建立一個佇列工作](#creating-jobs)；但是，您應該將 `Illuminate\Bus\Batchable` trait 新增至工作類別中。這個 trait 提供了 `batch` 方法的存取權限，該方法可用於檢索工作正在其中執行的當前批次：

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

要分派一批工作，您應該使用 `Bus` Facade 的 `batch` 方法。當然，批次處理主要在與完成回呼函式結合時才有用。因此，您可以使用 `then`、`catch` 和 `finally` 方法來定義批次的完成回呼函式。每個回呼函式在被呼叫時都會收到一個 `Illuminate\Bus\Batch` 實例。在這個範例中，我們假設正在將一批工作排入佇列，每個工作都會處理 CSV 檔案中的指定行數：

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

批次的 ID，可以透過 `$batch->id` 屬性存取，可用於在批次分派後[查詢 Laravel 命令匯流排](#inspecting-batches)以獲取批次資訊。

> [!WARNING]
> 由於批次回呼函式由 Laravel 佇列序列化並在稍後執行，因此您不應在回呼函式中使用 `$this` 變數。此外，由於批次工作包裹在資料庫交易中，觸發隱式提交的資料庫語句不應在工作中執行。

<a name="naming-batches"></a>
#### 命名批次

某些工具，例如 [Laravel Horizon](/docs/{{version}}/horizon) 和 [Laravel Telescope](/docs/{{version}}/telescope)，如果批次已命名，可能會為批次提供更友善的偵錯資訊。要為批次指定一個任意名稱，您可以在定義批次時呼叫 `name` 方法：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import CSV')->dispatch();
```

<a name="batch-connection-queue"></a>
#### 批次連線與佇列

如果您想指定用於批次工作的連線和佇列，您可以使用 `onConnection` 和 `onQueue` 方法。所有批次工作都必須在相同的連線和佇列中執行：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->onConnection('redis')->onQueue('imports')->dispatch();
```

<a name="chains-and-batches"></a>
### 鏈結與批次

您可以透過將[鏈結工作](#job-chaining)放在陣列中，在批次內定義一組鏈結工作。例如，我們可以並行執行兩個工作鏈結，並在兩個工作鏈結都完成處理後執行一個回呼函式：

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

反之，您可以透過在鏈結中定義批次，在[鏈結](#job-chaining)中執行工作批次。例如，您可以先執行一批工作來發布多個 Podcast，然後再執行一批工作來傳送發布通知：

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
### 將工作加入批次

有時，從批次工作內部將額外工作加入批次可能很有用。當您需要批次處理數千個工作，這些工作可能在網頁請求期間分派時間過長時，這種模式會很有用。因此，您可以選擇分派一個初始的「載入器」工作批次，以將更多工作注入批次：

```php
$batch = Bus::batch([
    new LoadImportBatch,
    new LoadImportBatch,
    new LoadImportBatch,
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import Contacts')->dispatch();
```

在這個範例中，我們將使用 `LoadImportBatch` 工作來為批次注入額外的工作。為了實現這一點，我們可以使用批次實例上的 `add` 方法，該方法可以透過工作的 `batch` 方法存取：

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
> 您只能從屬於同一批次的工作中，將工作加入批次。

<a name="inspecting-batches"></a>
### 檢查批次

提供給批次完成回呼的 `Illuminate\Bus\Batch` 實例，包含各種屬性與方法，可協助您與特定批次工作互動及檢查：

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

所有 `Illuminate\Bus\Batch` 實例都可 JSON 序列化，這表示您可以直接從應用程式的路由回傳它們，以取得包含批次資訊 (包括其完成進度) 的 JSON payload。這使得在應用程式的使用者介面 (UI) 中顯示批次的完成進度資訊變得方便。

若要透過批次的 ID 取得批次，您可以使用 `Bus` Facade 的 `findBatch` 方法：

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

如您在先前的範例中所見，批次處理的工作通常應在繼續執行前判斷其對應批次是否已取消。然而，為了方便起見，您可以將 `SkipIfBatchCancelled` [中介層](#job-middleware) 指派給工作。顧名思義，此中介層將指示 Laravel，如果工作的對應批次已取消，則不處理該工作：

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

當批次處理的工作失敗時，`catch` 回呼 (如果已指定) 將會被呼叫。此回呼僅針對批次中第一個失敗的工作被呼叫。

<a name="allowing-failures"></a>
#### 允許失敗

當批次中的工作失敗時，Laravel 會自動將批次標記為「已取消」。如果您希望，可以停用此行為，使工作失敗不會自動將批次標記為已取消。這可以透過在分派批次時呼叫 `allowFailures` 方法來達成：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->allowFailures()->dispatch();
```

您可以選擇性地提供一個閉包給 `allowFailures` 方法，這個閉包會在每次工作失敗時執行：

```php
$batch = Bus::batch([
    // ...
])->allowFailures(function (Batch $batch, $exception) {
    // Handle individual job failures...
})->dispatch();
```

<a name="retrying-failed-batch-jobs"></a>
#### 重試失敗的批次工作

為了方便起見，Laravel 提供了 `queue:retry-batch` Artisan 命令，讓您可以輕鬆重試特定批次中所有失敗的工作。此命令接受批次的 UUID，該批次中失敗的工作將會被重試：

```shell
php artisan queue:retry-batch 32dbc76c-4f82-4749-b610-a639fe0099b5
```

<a name="pruning-batches"></a>
### 清除批次

如果不清除，`job_batches` 資料表可能會非常迅速地累積記錄。為了緩解這個問題，您應該[排程](/docs/{{version}}/scheduling) `queue:prune-batches` Artisan 命令使其每天執行：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches')->daily();
```

依預設，所有超過 24 小時的已完成批次都將被清除。您可以在呼叫命令時使用 `hours` 選項來決定保留批次資料的時間長度。例如，以下命令將刪除所有超過 48 小時前完成的批次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48')->daily();
```

有時，您的 `jobs_batches` 資料表可能會累積從未成功完成的批次記錄，例如批次中某個工作失敗且從未成功重試的批次。您可以指示 `queue:prune-batches` 命令使用 `unfinished` 選項來清除這些未完成的批次記錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --unfinished=72')->daily();
```

同樣地，您的 `jobs_batches` 資料表也可能累積已取消批次的記錄。您可以指示 `queue:prune-batches` 命令使用 `cancelled` 選項來清除這些已取消的批次記錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --cancelled=72')->daily();
```

<a name="storing-batches-in-dynamodb"></a>
### 將批次儲存至 DynamoDB

Laravel 也支援將批次中繼資訊儲存至 [DynamoDB](https://aws.amazon.com/dynamodb)，而非關聯式資料庫。但是，您需要手動建立一個 DynamoDB 資料表來儲存所有批次紀錄。

通常，此資料表應命名為 `job_batches`，但您應該根據應用程式 `queue` 設定檔中 `queue.batching.table` 設定選項的值來命名資料表。

<a name="dynamodb-batch-table-configuration"></a>
#### DynamoDB 批次資料表設定

`job_batches` 資料表應具有一個名為 `application` 的字串主要分割鍵和一個名為 `id` 的字串主要排序鍵。鍵的 `application` 部分將包含應用程式 `app` 設定檔中 `name` 設定值所定義的應用程式名稱。由於應用程式名稱是 DynamoDB 資料表鍵的一部分，您可以使用同一個資料表來儲存多個 Laravel 應用程式的工作批次。

此外，如果想利用 [自動清除批次](#pruning-batches-in-dynamodb)，您可以在資料表中定義 `ttl` 屬性。

<a name="dynamodb-configuration"></a>
#### DynamoDB 設定

接著，安裝 AWS SDK，以便您的 Laravel 應用程式可以與 Amazon DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

然後，將 `queue.batching.driver` 設定選項的值設為 `dynamodb`。此外，您應該在 `batching` 設定陣列中定義 `key`、`secret` 和 `region` 設定選項。這些選項將用於向 AWS 進行身份驗證。當使用 `dynamodb` 驅動程式時，`queue.batching.database` 設定選項是不必要的：

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
#### 清除 DynamoDB 中的批次

當利用 [DynamoDB](https://aws.amazon.com/dynamodb) 儲存工作批次資訊時，用於清除儲存在關聯式資料庫中批次的典型清除指令將不起作用。相反地，您可以使用 [DynamoDB 原生 TTL 功能](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html) 自動移除舊批次的紀錄。

如果您使用 `ttl` 屬性定義了 DynamoDB 資料表，您可以定義設定參數來指示 Laravel 如何清除批次紀錄。`queue.batching.ttl_attribute` 設定值定義了儲存 TTL 的屬性名稱，而 `queue.batching.ttl` 設定值則定義了批次紀錄在上次更新後多少秒可從 DynamoDB 資料表移除：

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

除了將工作類別分派到佇列之外，您也可以分派一個閉包。這對於需要脫離當前請求週期執行的快速、簡單任務非常有用。當將閉包分派到佇列時，該閉包的程式碼內容會經過密碼學簽章，以防止在傳輸過程中被修改：

```php
use App\Models\Podcast;

$podcast = Podcast::find(1);

dispatch(function () use ($podcast) {
    $podcast->publish();
});
```

若要為佇列閉包指定名稱，該名稱可用於佇列報告儀表板以及由 `queue:work` 命令顯示，您可以使用 `name` 方法：

```php
dispatch(function () {
    // ...
})->name('Publish Podcast');
```

使用 `catch` 方法，您可以提供一個閉包。如果佇列閉包在耗盡所有佇列的[已設定重試嘗試次數](#max-job-attempts-and-timeout)後仍未能成功完成，則應執行此閉包：

```php
use Throwable;

dispatch(function () use ($podcast) {
    $podcast->publish();
})->catch(function (Throwable $e) {
    // This job has failed...
});
```

> [!WARNING]
> 由於 `catch` 回呼會被序列化，並由 Laravel 佇列在稍後時間執行，您不應該在 `catch` 回呼中使用 `$this` 變數。

<a name="running-the-queue-worker"></a>
## 執行佇列工作者

<a name="the-queue-work-command"></a>
### `queue:work` 命令

Laravel 包含一個 Artisan 命令，它將啟動佇列工作者並處理新的工作，當它們被推送到佇列時。您可以使用 `queue:work` Artisan 命令來執行工作者。請注意，一旦 `queue:work` 命令啟動，它將持續執行，直到您手動停止它或關閉終端機：

```shell
php artisan queue:work
```

> [!NOTE]
> 為了讓 `queue:work` 程序永久在背景執行，您應該使用一個進程監控器，例如 [Supervisor](#supervisor-configuration) 以確保佇列工作者不會停止執行。

您可以包含 `-v` 旗標在調用 `queue:work` 命令時，如果您希望在命令輸出中包含已處理的工作 ID、連線名稱和佇列名稱：

```shell
php artisan queue:work -v
```

請記住，佇列工作者是長時間執行的程序，並將已啟動的應用程式狀態儲存在記憶體中。因此，它們在啟動後不會察覺到程式碼庫中的變更。因此，在您的部署過程中，請務必 [重新啟動佇列工作者](#queue-workers-and-deployment)。此外，請記住，應用程式建立或修改的任何靜態狀態都不會在工作之間自動重設。

或者，您可以執行 `queue:listen` 命令。使用 `queue:listen` 命令時，當您想要重新載入更新的程式碼或重設應用程式狀態時，無需手動重新啟動工作者；然而，此命令的效率明顯低於 `queue:work` 命令：

```shell
php artisan queue:listen
```

<a name="running-multiple-queue-workers"></a>
#### 執行多個佇列工作者

要為佇列分配多個工作者並同時處理工作，您只需啟動多個 `queue:work` 程序。這可以在本機透過終端機中的多個分頁完成，或在生產環境中使用您的程序管理器的設定來完成。[使用 Supervisor](#supervisor-configuration) 時，您可以使用 `numprocs` 配置值。

<a name="specifying-the-connection-queue"></a>
#### 指定連線與佇列

您也可以指定工作者應使用的佇列連線。傳遞給 `work` 命令的連線名稱應與您的 `config/queue.php` 配置檔案中定義的其中一個連線相對應：

```shell
php artisan queue:work redis
```

預設情況下，`queue:work` 命令僅處理指定連線上的預設佇列中的工作。然而，您還可以進一步自訂佇列工作者，使其僅處理指定連線上的特定佇列。例如，如果您的所有電子郵件都在 `redis` 佇列連線上的 `emails` 佇列中處理，您可以發出以下命令來啟動一個僅處理該佇列的工作者：

```shell
php artisan queue:work redis --queue=emails
```

<a name="processing-a-specified-number-of-jobs"></a>
#### 處理指定數量的工作

您可以使用 `--once` 選項來指示工作者只處理佇列中的單一工作：

```shell
php artisan queue:work --once
```

您可以使用 `--max-jobs` 選項來指示工作者處理指定數量的工作然後退出。此選項與 [Supervisor](#supervisor-configuration) 結合使用時可能很有用，這樣您的工作者在處理完指定數量的工作後會自動重新啟動，釋放可能累積的任何記憶體：

```shell
php artisan queue:work --max-jobs=1000
```

<a name="processing-all-queued-jobs-then-exiting"></a>
#### 處理所有佇列中的工作然後退出

您可以使用 `--stop-when-empty` 選項來指示工作者處理所有工作然後優雅地退出。此選項在 Docker 容器中處理 Laravel 佇列時可能很有用，如果您希望在佇列清空後關閉容器：

```shell
php artisan queue:work --stop-when-empty
```

<a name="processing-jobs-for-a-given-number-of-seconds"></a>
#### 處理指定時間的工作

您可以使用 `--max-time` 選項來指示工作者在指定秒數內處理工作然後退出。此選項與 [Supervisor](#supervisor-configuration) 結合使用時可能很有用，這樣您的工作者在處理指定時間的工作後會自動重新啟動，釋放可能累積的任何記憶體：

```shell
# Process jobs for one hour and then exit...
php artisan queue:work --max-time=3600
```

<a name="worker-sleep-duration"></a>
#### 工作者休眠時長

當佇列中有工作時，工作者將持續處理工作而不在工作之間產生延遲。然而，`sleep` 選項決定了如果沒有可用的工作，工作者將「休眠」多少秒。當然，在休眠期間，工作者將不會處理任何新的工作：

```shell
php artisan queue:work --sleep=3
```

<a name="maintenance-mode-queues"></a>
#### 維護模式與佇列

當您的應用程式處於 [維護模式](/docs/{{version}}/configuration#maintenance-mode) 時，所有佇列中的工作將不會被處理。一旦應用程式退出維護模式，工作將繼續正常處理。

即使啟用維護模式，也要強制您的佇列工作者處理工作，您可以使用 `--force` 選項：

```shell
php artisan queue:work --force
```

<a name="resource-considerations"></a>
#### 資源考量

守護進程佇列工作者在處理每個工作之前不會「重新啟動」框架。因此，在每個工作完成後，您應該釋放所有佔用大量資源的項目。例如，如果您正在使用 [GD library](https://www.php.net/manual/en/book.image.php) 進行影像處理，在處理完影像後，您應該使用 `imagedestroy` 釋放記憶體。

<a name="queue-priorities"></a>
### 佇列優先權

有時您可能希望優先處理佇列。例如，在您的 `config/queue.php` 配置檔案中，您可以將 `redis` 連線的預設 `queue` 設定為 `low`。然而，有時您可能希望將工作推送到 `high` 優先級的佇列，例如這樣：

```php
dispatch((new Job)->onQueue('high'));
```

若要啟動一個工作者，確保所有 `high` 佇列中的工作在處理 `low` 佇列中的任何工作之前都被處理，請將以逗號分隔的佇列名稱列表傳遞給 `work` 命令：

```shell
php artisan queue:work --queue=high,low
```

<a name="queue-workers-and-deployment"></a>
### 佇列工作者與部署

由於佇列工作者是長時間執行的程序，如果不重新啟動，它們將不會注意到您程式碼的變更。因此，部署使用佇列工作者的應用程式最簡單的方法是在部署過程中重新啟動工作者。您可以透過發出 `queue:restart` 命令來優雅地重新啟動所有工作者：

```shell
php artisan queue:restart
```

此命令將指示所有佇列工作者在處理完當前工作後優雅地退出，這樣就不會遺失任何現有的工作。由於佇列工作者在執行 `queue:restart` 命令時會退出，您應該執行一個程序管理器，例如 [Supervisor](#supervisor-configuration) 以自動重新啟動佇列工作者。

> [!NOTE]
> 佇列使用 [快取](/docs/{{version}}/cache) 來儲存重新啟動訊號，因此在使用此功能之前，您應該驗證您的應用程式是否已正確配置快取驅動程式。

<a name="job-expirations-and-timeouts"></a>
### 工作過期與逾時


<a name="job-expiration"></a>
#### 工作過期

在您應用程式的 `config/queue.php` 配置檔案中，每個佇列連線都定義了一個 `retry_after` 選項。此選項指定佇列連線在重試一個正在處理中的工作之前，應該等待多少秒。舉例來說，如果 `retry_after` 的值設定為 `90`，那麼如果一個工作已經處理了 90 秒但尚未被釋放或刪除，它將被釋回佇列。通常，您應該將 `retry_after` 的值設定為您的工作合理完成處理所需的最大秒數。

> [!WARNING]
> 唯一不包含 `retry_after` 值的佇列連線是 Amazon SQS。SQS 會根據其 [Default Visibility Timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html) 重試工作，該設定在 AWS console 中管理。


<a name="worker-timeouts"></a>
#### 工作者逾時

`queue:work` Artisan 命令暴露了一個 `--timeout` 選項。預設情況下，`--timeout` 值為 60 秒。如果一個工作處理的時間超過了逾時值所指定的秒數，處理該工作的工作者將會以錯誤退出。通常，工作者會由在您的伺服器上配置的[程序管理員](#supervisor-configuration)自動重新啟動：

```shell
php artisan queue:work --timeout=60
```

`retry_after` 配置選項和 `--timeout` CLI 選項是不同的，但它們協同運作，以確保工作不遺失，且工作僅被成功處理一次。

> [!WARNING]
> `--timeout` 值應始終比您的 `retry_after` 配置值至少短幾秒。這將確保處理停滯的工作的工作者總是在工作被重試之前終止。如果您的 `--timeout` 選項長於您的 `retry_after` 配置值，您的工作可能會被處理兩次。

<a name="supervisor-configuration"></a>
## Supervisor 設定

在生產環境中，您需要一種方法來保持 `queue:work` 處理程序持續運行。`queue:work` 處理程序可能因多種原因停止運行，例如工作者逾時或執行 `queue:restart` 命令。

因此，您需要配置一個進程監控器，以偵測您的 `queue:work` 處理程序何時退出並自動重新啟動它們。此外，進程監控器還可以讓您指定要同時運行多少個 `queue:work` 處理程序。Supervisor 是 Linux 環境中常用的進程監控器，我們將在本文件中討論如何配置它。

<a name="installing-supervisor"></a>
#### 安裝 Supervisor

Supervisor 是一個用於 Linux 作業系統的進程監控器，如果您的 `queue:work` 處理程序失敗，它將自動重新啟動。若要在 Ubuntu 上安裝 Supervisor，您可以使用以下命令：

```shell
sudo apt-get install supervisor
```

> [!NOTE]
> 如果您覺得自行配置和管理 Supervisor 很麻煩，請考慮使用 [Laravel Cloud](https://cloud.laravel.com)，它提供了用於運行 Laravel 佇列工作者的全代管平台。

<a name="configuring-supervisor"></a>
#### 配置 Supervisor

Supervisor 配置檔案通常儲存在 `/etc/supervisor/conf.d` 目錄中。在此目錄中，您可以建立任意數量的配置檔案，以指示 Supervisor 如何監控您的處理程序。例如，讓我們建立一個 `laravel-worker.conf` 檔案，用於啟動和監控 `queue:work` 處理程序：

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

在此範例中，`numprocs` 指令將指示 Supervisor 運行八個 `queue:work` 處理程序並監控所有這些程序，如果它們失敗將自動重新啟動。您應該更改配置中的 `command` 指令，以反映您所需的佇列連線和工作者選項。

> [!WARNING]
> 您應該確保 `stopwaitsecs` 的值大於您最長運行工作所花費的秒數。否則，Supervisor 可能會在工作完成處理之前終止該工作。

<a name="starting-supervisor"></a>
#### 啟動 Supervisor

配置檔案建立後，您可以使用以下命令更新 Supervisor 配置並啟動處理程序：

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start "laravel-worker:*"
```

如需更多關於 Supervisor 的資訊，請查閱 [Supervisor 文件](http://supervisord.org/index.html)。

<a name="dealing-with-failed-jobs"></a>
## 處理失敗的工作

有時候您的佇列工作會失敗。別擔心，事情不總是按計畫進行！Laravel 提供了一個方便的方式來[指定工作應嘗試的最大次數](#max-job-attempts-and-timeout)。在非同步工作超過此嘗試次數後，它將被插入到 `failed_jobs` 資料庫資料表。[同步分派的工作](/docs/{{version}}/queues#synchronous-dispatching)失敗後不會儲存在此資料表，其異常會立即由應用程式處理。

用於建立 `failed_jobs` 資料表的遷移通常已經存在於新的 Laravel 應用程式中。然而，如果您的應用程式不包含此資料表的遷移，您可以使用 `make:queue-failed-table` 命令來建立遷移：

```shell
php artisan make:queue-failed-table

php artisan migrate
```

當執行 [佇列工作者](#running-the-queue-worker)處理程序時，您可以使用 `queue:work` 命令上的 `--tries` 開關來指定工作應嘗試的最大次數。如果您沒有為 `--tries` 選項指定值，工作只會被嘗試一次，或者像工作類別的 `$tries` 屬性所指定的次數：

```shell
php artisan queue:work redis --tries=3
```

使用 `--backoff` 選項，您可以指定 Laravel 應等待多少秒，然後再重試遇到異常的工作。預設情況下，工作會立即釋放回佇列中，以便可以再次嘗試：

```shell
php artisan queue:work redis --tries=3 --backoff=3
```

如果您想設定 Laravel 應等待多少秒，然後再重試遇到異常的工作（每個工作單獨設定），您可以透過在工作類別上定義一個 `backoff` 屬性來實現：

```php
/**
 * The number of seconds to wait before retrying the job.
 *
 * @var int
 */
public $backoff = 3;
```

如果您需要更複雜的邏輯來決定工作的退避時間，您可以在工作類別上定義一個 `backoff` 方法：

```php
/**
 * Calculate the number of seconds to wait before retrying the job.
 */
public function backoff(): int
{
    return 3;
}
```

您可以透過從 `backoff` 方法返回一個退避值陣列，輕鬆設定「指數型」退避。在這個範例中，第一次重試的延遲為 1 秒，第二次重試為 5 秒，第三次重試為 10 秒，如果還有更多嘗試次數，則每次後續重試為 10 秒：

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

當特定工作失敗時，您可能想要向您的使用者發送警報，或還原工作部分完成的任何動作。為此，您可以在工作類別上定義一個 `failed` 方法。導致工作失敗的 `Throwable` 實例將會傳遞給 `failed` 方法：

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
> 在呼叫 `failed` 方法之前，會實例化一個新的工作實例；因此，任何可能在 `handle` 方法中發生的類別屬性修改將會遺失。

失敗的工作不一定是因為遇到未處理的異常。當工作耗盡所有允許的嘗試次數時，也可能被視為失敗。這些嘗試次數可以透過多種方式消耗：

<div class="content-list" markdown="1">

- 工作逾時。
- 工作在執行期間遇到未處理的異常。
- 工作被手動或透過中介層釋放回佇列。

</div>

如果最終嘗試因工作執行期間拋出的異常而失敗，該異常將會傳遞給工作的 `failed` 方法。然而，如果工作因達到最大允許嘗試次數而失敗，則 `$exception` 將會是 `Illuminate\Queue\MaxAttemptsExceededException` 的實例。同樣地，如果工作因超過設定的逾時而失敗，則 `$exception` 將會是 `Illuminate\Queue\TimeoutExceededException` 的實例。

<a name="retrying-failed-jobs"></a>
### 重試失敗的工作

要查看所有已插入到 `failed_jobs` 資料庫資料表中的失敗工作，您可以使用 `queue:failed` Artisan 命令：

```shell
php artisan queue:failed
```

`queue:failed` 命令將列出工作 ID、連線、佇列、失敗時間以及關於工作的其他資訊。工作 ID 可用於重試失敗的工作。例如，要重試 ID 為 `ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece` 的失敗工作，請發出以下命令：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece
```

如果有必要，您可以將多個 ID 傳遞給命令：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece 91401d2c-0784-4f43-824c-34f94a33c24d
```

您也可以重試特定佇列的所有失敗工作：

```shell
php artisan queue:retry --queue=name
```

要重試所有失敗的工作，請執行 `queue:retry` 命令並將 `all` 作為 ID 傳遞：

```shell
php artisan queue:retry all
```

如果您想刪除失敗的工作，您可以使用 `queue:forget` 命令：

```shell
php artisan queue:forget 91401d2c-0784-4f43-824c-34f94a33c24d
```

> [!NOTE]
> 當使用 [Horizon](/docs/{{version}}/horizon) 時，您應該使用 `horizon:forget` 命令來刪除失敗的工作，而不是 `queue:forget` 命令。

要從 `failed_jobs` 資料表刪除所有失敗的工作，您可以使用 `queue:flush` 命令：

```shell
php artisan queue:flush
```

`queue:flush` 命令會從佇列中移除所有失敗的工作記錄，無論失敗的工作有多舊。您可以使用 `--hours` 選項僅刪除在特定小時前或更早失敗的工作：

```shell
php artisan queue:flush --hours=48
```

<a name="ignoring-missing-models"></a>
### 忽略遺失的模型

當將 Eloquent 模型注入到工作中時，模型會在放置到佇列之前自動序列化，且在工作處理時從資料庫中重新擷取。然而，如果模型在工作等待工作者處理時已被刪除，您的工作可能會因 `ModelNotFoundException` 而失敗。

為了方便，您可以選擇自動刪除模型遺失的工作，方法是將工作的 `deleteWhenMissingModels` 屬性設定為 `true`。當此屬性設定為 `true` 時，Laravel 將會悄悄地丟棄工作而不拋出異常：

```php
/**
 * Delete the job if its models no longer exist.
 *
 * @var bool
 */
public $deleteWhenMissingModels = true;
```

<a name="pruning-failed-jobs"></a>
### 清除失敗的工作

您可以透過呼叫 `queue:prune-failed` Artisan 命令來清除應用程式 `failed_jobs` 資料表中的記錄：

```shell
php artisan queue:prune-failed
```

預設情況下，所有超過 24 小時的失敗工作記錄將被清除。如果您為命令提供 `--hours` 選項，只有在過去 N 小時內插入的失敗工作記錄會被保留。例如，以下命令將刪除所有在超過 48 小時前插入的失敗工作記錄：

```shell
php artisan queue:prune-failed --hours=48
```

<a name="storing-failed-jobs-in-dynamodb"></a>
### 將失敗的工作儲存至 DynamoDB

Laravel 也支援將您失敗的工作紀錄儲存到 [DynamoDB](https://aws.amazon.com/dynamodb)，而非關聯式資料庫表。不過，您必須手動建立一個 DynamoDB 資料表來儲存所有失敗的工作紀錄。通常，這個資料表應該命名為 `failed_jobs`，但您應該根據應用程式 `queue` 設定檔中 `queue.failed.table` 設定值的內容來命名資料表。

`failed_jobs` 資料表應包含一個名為 `application` 的字串主分割區鍵，以及一個名為 `uuid` 的字串主排序鍵。該鍵的 `application` 部分將包含您的應用程式名稱，此名稱由應用程式 `app` 設定檔中 `name` 設定值所定義。由於應用程式名稱是 DynamoDB 資料表鍵的一部分，您可以使用同一個資料表來儲存多個 Laravel 應用程式的失敗工作。

此外，請確保您已安裝 AWS SDK，以便您的 Laravel 應用程式可以與 Amazon DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

接著，將 `queue.failed.driver` 設定選項的值設定為 `dynamodb`。此外，您應該在失敗工作設定陣列中定義 `key`、`secret` 和 `region` 設定選項。這些選項將用於向 AWS 進行身份驗證。當使用 `dynamodb` 驅動程式時，`queue.failed.database` 設定選項是不必要的：

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
### 停用失敗工作儲存

您可以透過將 `queue.failed.driver` 設定選項的值設定為 `null`，指示 Laravel 丟棄失敗的工作而不儲存它們。通常，這可以透過 `QUEUE_FAILED_DRIVER` 環境變數來實現：

```ini
QUEUE_FAILED_DRIVER=null
```

<a name="failed-job-events"></a>
### 失敗工作事件

如果您想註冊一個事件監聽器，以便在工作失敗時被呼叫，您可以使用 `Queue` Facade 的 `failing` 方法。例如，我們可以從 Laravel 內建的 `AppServiceProvider` 的 `boot` 方法中，將一個閉包附加到此事件：

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
## 清除佇列中的工作

> [!NOTE]
> 使用 [Horizon](/docs/{{version}}/horizon) 時，您應該使用 `horizon:clear` 命令來清除佇列中的工作，而非 `queue:clear` 命令。

若您想從預設連線的預設佇列中刪除所有工作，可以使用 `queue:clear` Artisan 命令來執行：

```shell
php artisan queue:clear
```

您也可以提供 `connection` 引數和 `queue` 選項，以從特定連線和佇列中刪除工作：

```shell
php artisan queue:clear redis --queue=emails
```

> [!WARNING]
> 清除佇列中的工作僅適用於 SQS、Redis 和資料庫佇列驅動程式。此外，SQS 訊息刪除程序需要長達 60 秒，因此在您清除佇列後 60 秒內傳送至 SQS 佇列的工作也可能會被刪除。

<a name="monitoring-your-queues"></a>
## 監控佇列

如果您的佇列突然湧入大量工作，可能會導致佇列不堪負荷，進而延長工作的完成時間。如果您願意，Laravel 可以在您的佇列工作計數超過指定閾值時向您發出警報。

首先，您應該將 `queue:monitor` 命令排程為 [每分鐘執行](/docs/{{version}}/scheduling)。此命令接受您希望監控的佇列名稱以及您期望的工作計數閾值：

```shell
php artisan queue:monitor redis:default,redis:deployments --max=100
```

僅僅排程此命令並不足以觸發通知，提醒您佇列已不堪負荷。當命令遇到工作計數超過閾值的佇列時，將會分派一個 `Illuminate\Queue\Events\QueueBusy` 事件。您可以在應用程式的 `AppServiceProvider` 中監聽此事件，以便向您或您的開發團隊傳送通知：

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

當測試分派工作的程式碼時，您可能希望指示 Laravel 不要實際執行工作本身，因為工作的程式碼可以直接且獨立於分派它的程式碼進行測試。當然，要測試工作本身，您可以實例化一個工作實例並直接在測試中呼叫其 `handle` 方法。

您可以使用 `Queue` facade 的 `fake` 方法來防止佇列工作實際被推送到佇列中。在呼叫 `Queue` facade 的 `fake` 方法後，您可以斷言應用程式嘗試將工作推送到佇列：

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

您可以將閉包傳遞給 `assertPushed`、`assertNotPushed`、`assertClosurePushed` 或 `assertClosureNotPushed` 方法，以斷言推送到佇列中的工作通過了給定的「真值測試」。如果至少有一個工作通過了給定的真值測試，則斷言將會成功：

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

如果您只需要偽造特定的工作，同時允許其他工作正常執行，您可以將應偽造的工作的類別名稱傳遞給 `fake` 方法：

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

您可以使用 `except` 方法偽造除了指定工作集之外的所有工作：

```php
Queue::fake()->except([
    ShipOrder::class,
]);
```

<a name="testing-job-chains"></a>
### 測試工作鏈結

要測試工作鏈結，您需要利用 `Bus` facade 的偽造功能。`Bus` facade 的 `assertChained` 方法可用於斷言工作鏈結已分派。`assertChained` 方法接受一個鏈結工作的陣列作為其第一個引數：

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

如您在上述範例中看到的，鏈結工作的陣列可以是工作類別名稱的陣列。但是，您也可以提供實際工作實例的陣列。這樣做時，Laravel 將確保工作實例屬於相同的類別，並且具有應用程式分派的鏈結工作的相同屬性值：

```php
Bus::assertChained([
    new ShipOrder,
    new RecordShipment,
    new UpdateInventory,
]);
```

您可以使用 `assertDispatchedWithoutChain` 方法來斷言工作已推送而沒有工作鏈結：

```php
Bus::assertDispatchedWithoutChain(ShipOrder::class);
```

<a name="testing-chain-modifications"></a>
#### 測試鏈結修改

如果鏈結工作將工作[預設或附加到現有鏈結](#adding-jobs-to-the-chain)，您可以使用工作的 `assertHasChain` 方法來斷言工作具有預期的剩餘工作鏈結：

```php
$job = new ProcessPodcast;

$job->handle();

$job->assertHasChain([
    new TranscribePodcast,
    new OptimizePodcast,
    new ReleasePodcast,
]);
```

`assertDoesntHaveChain` 方法可用於斷言工作的剩餘鏈結為空：

```php
$job->assertDoesntHaveChain();
```

<a name="testing-chained-batches"></a>
#### 測試鏈結批次

如果您的工作鏈結[包含工作批次](#chains-and-batches)，您可以透過在鏈結斷言中插入 `Bus::chainedBatch` 定義來斷言鏈結批次符合您的預期：

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

`Bus` facade 的 `assertBatched` 方法可用於斷言[工作批次](#job-batching)已分派。傳遞給 `assertBatched` 方法的閉包會接收一個 `Illuminate\Bus\PendingBatch` 實例，該實例可用於檢查批次中的工作：

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

您可以使用 `assertBatchCount` 方法來斷言已分派了指定數量的批次：

```php
Bus::assertBatchCount(3);
```

您可以使用 `assertNothingBatched` 來斷言沒有批次被分派：

```php
Bus::assertNothingBatched();
```

<a name="testing-job-batch-interaction"></a>
#### 測試工作/批次互動

此外，您可能偶爾需要測試個別工作與其底層批次的互動。例如，您可能需要測試工作是否取消了其批次的後續處理。為此，您需要透過 `withFakeBatch` 方法為工作指派一個偽造批次。`withFakeBatch` 方法會回傳一個包含工作實例和偽造批次的 tuple：

```php
[$job, $batch] = (new ShipOrder)->withFakeBatch();

$job->handle();

$this->assertTrue($batch->cancelled());
$this->assertEmpty($batch->added);
```

<a name="testing-job-queue-interactions"></a>
### 測試工作/佇列互動

有時候，您可能需要測試佇列工作是否[將自身釋回佇列](#manually-releasing-a-job)。或者，您可能需要測試工作是否已自行刪除。您可以透過實例化工作並呼叫 `withFakeQueueInteractions` 方法來測試這些佇列互動。

一旦工作的佇列互動被偽造後，您可以在該工作上呼叫 `handle` 方法。呼叫工作後，提供了各種斷言方法來驗證工作的佇列互動：

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

使用 `Queue` [Facade](/docs/{{version}}/facades) 上的 `before` 和 `after` 方法，您可以指定在佇列工作被處理之前或之後執行的回呼。這些回呼是執行額外日誌記錄或增加儀表板統計數據的好機會。通常，您應該從 [service provider](/docs/{{version}}/providers) 的 `boot` 方法呼叫這些方法。例如，我們可以使用 Laravel 隨附的 `AppServiceProvider`：

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

使用 `Queue` [Facade](/docs/{{version}}/facades) 上的 `looping` 方法，您可以指定回呼，這些回呼會在 worker 嘗試從佇列中提取工作之前執行。例如，您可以註冊一個閉包，以回溯之前失敗的工作遺留的任何未關閉的交易：

```php
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Queue;

Queue::looping(function () {
    while (DB::transactionLevel() > 0) {
        DB::rollBack();
    }
});
```