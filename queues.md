# 佇列

- [簡介](#introduction)
    - [連線 vs. 佇列](#connections-vs-queues)
    - [驅動程式注意事項與前置需求](#driver-prerequisites)
- [建立工作 (Jobs)](#creating-jobs)
    - [產生工作類別](#generating-job-classes)
    - [類別結構](#class-structure)
    - [唯一工作 (Unique Jobs)](#unique-jobs)
    - [加密工作 (Encrypted Jobs)](#encrypted-jobs)
- [工作中介層 (Job Middleware)](#job-middleware)
    - [速率限制](#rate-limiting)
    - [防止工作重疊](#preventing-job-overlaps)
    - [限制例外狀況 (Throttling Exceptions)](#throttling-exceptions)
    - [跳過工作](#skipping-jobs)
- [派發工作](#dispatching-jobs)
    - [延遲派發](#delayed-dispatching)
    - [同步派發](#synchronous-dispatching)
    - [工作與資料庫交易](#jobs-and-database-transactions)
    - [工作鏈 (Job Chaining)](#job-chaining)
    - [自訂佇列與連線](#customizing-the-queue-and-connection)
    - [指定最大嘗試次數 / 超時值](#max-job-attempts-and-timeout)
    - [SQS FIFO 與公平佇列](#sqs-fifo-and-fair-queues)
    - [佇列故障轉移 (Queue Failover)](#queue-failover)
    - [錯誤處理](#error-handling)
- [工作批處理 (Job Batching)](#job-batching)
    - [定義可批處理的工作](#defining-batchable-jobs)
    - [派發批處理](#dispatching-batches)
    - [鏈與批處理](#chains-and-batches)
    - [將工作加入批處理](#adding-jobs-to-batches)
    - [檢查批處理](#inspecting-batches)
    - [取消批處理](#cancelling-batches)
    - [批處理失敗](#batch-failures)
    - [修剪批處理](#pruning-batches)
    - [將批處理儲存在 DynamoDB 中](#storing-batches-in-dynamodb)
- [將閉包加入佇列](#queueing-closures)
- [執行佇列工作者 (Queue Worker)](#running-the-queue-worker)
    - [`queue:work` 指令](#the-queue-work-command)
    - [佇列優先權](#queue-priorities)
    - [佇列工作者與部署](#queue-workers-and-deployment)
    - [工作過期與超時](#job-expirations-and-timeouts)
    - [暫停與恢復佇列工作者](#pausing-and-resuming-queue-workers)
- [Supervisor 設定](#supervisor-configuration)
- [處理失敗的工作](#dealing-with-failed-jobs)
    - [清理失敗的工作](#cleaning-up-after-failed-jobs)
    - [重試失敗的工作](#retrying-failed-jobs)
    - [忽略缺失的模型](#ignoring-missing-models)
    - [修剪失敗的工作](#pruning-failed-jobs)
    - [將失敗的工作儲存在 DynamoDB 中](#storing-failed-jobs-in-dynamodb)
    - [停用失敗工作儲存](#disabling-failed-job-storage)
    - [失敗工作事件](#failed-job-events)
- [從佇列中清除工作](#clearing-jobs-from-queues)
- [監控您的佇列](#monitoring-your-queues)
- [測試](#testing)
    - [偽造部分工作](#faking-a-subset-of-jobs)
    - [測試工作鏈](#testing-job-chains)
    - [測試工作批處理](#testing-job-batches)
    - [測試工作 / 佇列互動](#testing-job-queue-interactions)
- [工作事件](#job-events)

<a name="introduction"></a>
## 簡介

在建構您的網頁應用程式時，您可能會有一些任務（例如解析並儲存上傳的 CSV 檔案）在一般的網頁請求過程中需要花費太長時間才能完成。幸好，Laravel 允許您輕鬆地建立可放入佇列的工作 (jobs)，這些工作可以在背景中處理。透過將耗時的任務移至佇列，您的應用程式可以以極快的速度回應網頁請求，並為您的客戶提供更好的使用者體驗。

Laravel 佇列為各種不同的佇列後端提供了一個統一的佇列 API，例如 [Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 甚至是關聯式資料庫。

Laravel 的佇列設定選項儲存在應用程式的 `config/queue.php` 設定檔中。在此檔案中，您會發現框架內含的每個佇列驅動程式的連線設定，包括 database、[Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 和 [Beanstalkd](https://beanstalkd.github.io/) 驅動程式，以及一個會立即執行工作的同步驅動程式（用於開發或測試期間）。此外還包含一個 `null` 佇列驅動程式，它會直接捨棄佇列工作。

> [!NOTE]
> Laravel Horizon 是一個為 Redis 驅動的佇列所設計的精美儀表板與設定系統。請參閱完整的 [Horizon 文件](/docs/{{version}}/horizon) 以獲取更多資訊。


<a name="connections-vs-queues"></a>
### 連線 vs. 佇列

在開始使用 Laravel 佇列之前，了解「連線 (connections)」與「佇列 (queues)」之間的區別非常重要。在您的 `config/queue.php` 設定檔中，有一個 `connections` 設定陣列。此選項定義了與後端佇列服務（如 Amazon SQS、Beanstalk 或 Redis）的連線。然而，任何給定的佇列連線可能具有多個「佇列」，您可以將其視為不同的工作堆疊或工作集。

請注意，`queue` 設定檔中的每個連線設定範例都包含一個 `queue` 屬性。這是工作在被發送到給定連線時，預設會被派發到的佇列。換句話說，如果您在派發工作時沒有明確定義它應該被派發到哪個佇列，該工作將被放置在連線設定中 `queue` 屬性所定義的佇列中：

```php
use App\Jobs\ProcessPodcast;

// This job is sent to the default connection's default queue...
ProcessPodcast::dispatch();

// This job is sent to the default connection's "emails" queue...
ProcessPodcast::dispatch()->onQueue('emails');
```

有些應用程式可能不需要將工作推送到多個佇列，而傾向於使用一個簡單的佇列。然而，將工作推送到多個佇列對於希望優先排序或區分工作處理方式的應用程式特別有用，因為 Laravel 佇列工作者允許您指定它應該按優先權處理哪些佇列。例如，如果您將工作推送到 `high` 佇列，您可以執行一個給予該佇列較高處理優先權的工作者：

```shell
php artisan queue:work --queue=high,default
```


<a name="driver-prerequisites"></a>
### 驅動程式注意事項與前置需求


<a name="database"></a>
#### Database

為了使用 `database` 佇列驅動程式，您需要一個資料庫資料表來儲存工作。通常，這已包含在 Laravel 預設的 `0001_01_01_000002_create_jobs_table.php` [資料庫遷移](/docs/{{version}}/migrations) 中；然而，如果您的應用程式不包含此遷移，您可以使用 `make:queue-table` Artisan 指令來建立它：

```shell
php artisan make:queue-table

php artisan migrate
```


<a name="redis"></a>
#### Redis

為了使用 `redis` 佇列驅動程式，您應該在 `config/database.php` 設定檔中配置 Redis 資料庫連線。

> [!WARNING]
> `redis` 佇列驅動程式不支援 Redis 的 `serializer` 和 `compression` 選項。


<a name="redis-cluster"></a>
##### Redis Cluster

如果您的 Redis 佇列連線使用 [Redis 叢集](https://redis.io/docs/latest/operate/rs/databases/durability-ha/clustering)，您的佇列名稱必須包含 [金鑰雜湊標記 (key hash tag)](https://redis.io/docs/latest/develop/using-commands/keyspace/#hashtags)。這是為了確保給定佇列的所有 Redis 金鑰都被放置在相同的雜湊槽 (hash slot) 中：

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

使用 Redis 佇列時，您可以使用 `block_for` 設定選項來指定驅動程式在進入工作者迴圈並重新輪詢 Redis 資料庫之前，應該等待工作可用之時間長度。

根據您的佇列負載調整此值，會比持續輪詢 Redis 資料庫以獲取新工作更有效率。例如，您可以將值設定為 `5`，表示驅動程式在等待工作可用時應阻塞五秒鐘：

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
> 將 `block_for` 設定為 `0` 會導致佇列工作者無限期地阻塞，直到有工作可用。這也會導致在下一個工作被處理之前，無法處理如 `SIGTERM` 之類的訊號。


<a name="other-driver-prerequisites"></a>
#### 其他驅動程式前置需求

下列佇列驅動程式需要以下依賴套件。這些依賴套件可以透過 Composer 套件管理工具安裝：

<div class="content-list" markdown="1">

- Amazon SQS: `aws/aws-sdk-php ~3.0`
- Beanstalkd: `pda/pheanstalk ~5.0`
- Redis: `predis/predis ~2.0` 或 phpredis PHP 擴展
- [MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/queues/): `mongodb/laravel-mongodb`

</div>

<a name="creating-jobs"></a>
## 建立工作 (Jobs)


<a name="generating-job-classes"></a>
### 產生工作類別

預設情況下，應用程式中所有可進入佇列的工作都儲存在 `app/Jobs` 目錄中。如果 `app/Jobs` 目錄不存在，當您執行 `make:job` Artisan 指令時會自動建立：

```shell
php artisan make:job ProcessPodcast
```

產生的類別會實作 `Illuminate\Contracts\Queue\ShouldQueue` 介面，這會告訴 Laravel 該工作應該被推入佇列中以非同步執行。

> [!NOTE]
> 您可以使用 [stub 發布](/docs/{{version}}/artisan#stub-customization) 來自訂工作的 stub。


<a name="class-structure"></a>
### 類別結構

工作類別非常簡單，通常只包含一個在工作被佇列處理時被呼叫的 `handle` 方法。首先，讓我們來看看一個工作類別的範例。在這個範例中，我們假設我們經營一個 Podcast 發布服務，且需要在發布之前處理上傳的 Podcast 檔案：

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

在這個範例中，請注意我們可以直接將 [Eloquent 模型](/docs/{{version}}/eloquent) 傳遞到佇列工作的建構子中。由於工作使用了 `Queueable` trait，Eloquent 模型及其已載入的關聯在工作處理時會被優雅地序列化與反序列化。

如果您的佇列工作在建構子中接收一個 Eloquent 模型，只有該模型的識別碼會被序列化到佇列中。當工作實際被處理時，佇列系統會自動從資料庫中重新取得完整的模型實體及其載入的關聯。這種模型序列化方式可以讓傳送到佇列驅動程式的工作負載 (payload) 變得更小。


<a name="handle-method-dependency-injection"></a>
#### `handle` 方法的依賴注入

`handle` 方法會在工作被佇列處理時被呼叫。請注意，我們可以在工作的 `handle` 方法中使用型別提示 (type-hint) 來定義依賴項。Laravel 的 [服務容器](/docs/{{version}}/container) 會自動注入這些依賴項。

如果您想要完全控制容器如何將依賴項注入到 `handle` 方法中，可以使用容器的 `bindMethod` 方法。`bindMethod` 方法接收一個回呼函數 (callback)，該函數會接收工作與容器。在回呼函數中，您可以隨意呼叫 `handle` 方法。通常，您應該在 `App\Providers\AppServiceProvider` [服務提供者(Service Providers)](/docs/{{version}}/providers) 的 `boot` 方法中呼叫此方法：

```php
use App\Jobs\ProcessPodcast;
use App\Services\AudioProcessor;
use Illuminate\Contracts\Foundation\Application;

$this->app->bindMethod([ProcessPodcast::class, 'handle'], function (ProcessPodcast $job, Application $app) {
    return $job->handle($app->make(AudioProcessor::class));
});
```

> [!WARNING]
> 二進位資料（例如原始圖片內容）在傳遞給佇列工作之前，應先通過 `base64_encode` 函數處理。否則，工作在被放入佇列時可能無法正確地序列化為 JSON。


<a name="handling-relationships"></a>
#### 佇列中的關聯

由於所有已載入的 Eloquent 模型關聯在工作進入佇列時也會被序列化，因此序列化後的工作字串有時會變得非常大。此外，當工作被反序列化且模型關聯從資料庫重新取得時，它們將被完整地取得。在工作進入佇列過程中，模型被序列化之前所套用的任何關聯限制，在工作反序列化時將不會被套用。因此，如果您希望處理特定關聯的子集，應該在佇列工作中重新定義該關聯的限制。

或者，為了防止關聯被序列化，您可以在設定屬性值時，對模型呼叫 `withoutRelations` 方法。此方法將返回一個不含已載入關聯的模型實體：

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

如果您使用 [PHP 建構子屬性提升 (constructor property promotion)](https://www.php.net/manual/en/language.oop5.decon.php#language.oop5.decon.constructor.promotion) 且想要指定某個 Eloquent 模型不應序列化其關聯，可以使用 `WithoutRelations` 屬性 (attribute)：

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

為了方便起見，如果您希望序列化所有模型且不包含關聯，您可以將 `WithoutRelations` 屬性套用到整個類別，而不需要套用到每個模型上：

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

如果工作接收的是 Eloquent 模型的集合 (collection) 或陣列而非單一模型，則該集合中的模型在工作反序列化並執行時，其關聯將不會被還原。這是為了防止處理大量模型的工作消耗過多資源。

<a name="unique-jobs"></a>
### 唯一工作 (Unique Jobs)

> [!WARNING]
> 唯一工作需要支援 [鎖定](/docs/{{version}}/cache#atomic-locks) 的快取驅動程式。目前，`memcached`、`redis`、`dynamodb`、`database`、`file` 與 `array` 快取驅動程式皆支援原子鎖定 (atomic locks)。

> [!WARNING]
> 唯一工作的限制不適用於批處理中的工作。

有時候，您可能希望確保在任何時間點，佇列中僅存在一個特定工作的實例。您可以在工作類別中實作 `ShouldBeUnique` 介面來達成此目的。此介面不需要您在類別中定義任何額外的方法：

```php
<?php

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Queue\ShouldBeUnique;

class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    // ...
}
```

在上述範例中，`UpdateSearchIndex` 工作是唯一的。因此，如果該工作的另一個實例已經在佇列中且尚未完成處理，則此工作將不會被派發。

在某些情況下，您可能想要定義一個特定的「鍵 (key)」來使工作具有唯一性，或者您可能想要指定一個超時時間，超過該時間後工作將不再保持唯一。為了達成此目的，您可以在工作類別中定義 `uniqueId` 與 `uniqueFor` 屬性或方法：

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

在上述範例中，`UpdateSearchIndex` 工作是根據產品 ID 來決定唯一性的。因此，任何具有相同產品 ID 的新派發工作都將被忽略，直到現有的工作完成處理為止。此外，如果現有的工作在一小時內未被處理，唯一鎖定將被釋放，而另一個具有相同唯一鍵的工作則可以被派發到佇列中。

> [!WARNING]
> 如果您的應用程式從多台 Web 伺服器或容器派發工作，您應確保所有伺服器都與同一個中央快取伺服器通訊，以便 Laravel 能準確判斷工作是否唯一。


<a name="keeping-jobs-unique-until-processing-begins"></a>
#### 在處理開始前保持工作唯一

預設情況下，唯一工作會在完成處理或所有重試嘗試皆失敗後被「解鎖」。然而，在某些情況下，您可能希望工作在開始處理之前立即解鎖。為了達成此目的，您的工作應該實作 `ShouldBeUniqueUntilProcessing` 契約而非 `ShouldBeUnique` 契約：

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

在後台，當 `ShouldBeUnique` 工作被派發時，Laravel 會嘗試獲取一個使用 `uniqueId` 鍵的 [鎖定](/docs/{{version}}/cache#atomic-locks)。如果鎖定已被佔用，則工作不會被派發。此鎖定會在工作完成處理或所有重試嘗試皆失敗後被釋放。預設情況下，Laravel 將使用預設的快取驅動程式來獲取此鎖定。然而，如果您希望使用另一個驅動程式來獲取鎖定，您可以定義一個 `uniqueVia` 方法，該方法應回傳應使用的快取驅動程式：

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
> 如果您只需要限制工作的同時執行數量，請改用 [WithoutOverlapping](/docs/{{version}}/queues#preventing-job-overlaps) 工作中介層。


<a name="encrypted-jobs"></a>
### 加密工作 (Encrypted Jobs)

Laravel 允許您透過 [加密](/docs/{{version}}/encryption) 來確保工作資料的隱私性與完整性。要開始使用，只需將 `ShouldBeEncrypted` 介面新增至工作類別即可。一旦將此介面新增至類別，Laravel 會在將工作推送到佇列之前自動對其進行加密：

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
## 工作中介層 (Job Middleware)

工作中介層讓您可以將自訂邏輯封裝在佇列工作的執行過程周圍，從而減少工作類別本身的冗餘程式碼。例如，請參考以下利用 Laravel 的 Redis 速率限制功能，讓每五秒僅允許處理一個工作的 `handle` 方法：

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

雖然這段程式碼是正確的，但 `handle` 方法的實作會變得相當雜亂，因為其中充斥著 Redis 的速率限制邏輯。此外，對於任何我們想要限制速率的工作，都必須重複撰寫這段速率限制邏輯。與其在 handle 方法中進行速率限制，我們可以定義一個專門處理速率限制的工作中介層：

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

如您所見，與 [路由中介層](/docs/{{version}}/middleware) 類似，工作中介層會接收到目前正在處理的工作以及一個用於繼續處理該工作且應被呼叫的回呼函式。

您可以使用 `make:job-middleware` Artisan 指令來產生新的工作中介層類別。建立工作中介層後，可以透過在工作的 `middleware` 方法中回傳它們，將中介層附加到工作中。此方法在由 `make:job` Artisan 指令產生的工作中並不存在，因此您需要手動將其新增到工作類別中：

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
> 工作中介層也可以指派給 [可佇列的事件監聽器](/docs/{{version}}/events#queued-event-listeners)、[可郵寄類別 (mailables)](/docs/{{version}}/mail#queueing-mail) 以及 [通知](/docs/{{version}}/notifications#queueing-notifications)。


<a name="rate-limiting"></a>
### 速率限制

雖然我們剛才演示了如何編寫自訂的速率限制工作中介層，但 Laravel 實際上已經內建了可用於限制工作速率的速率限制中介層。與 [路由速率限制器](/docs/{{version}}/routing#defining-rate-limiters) 類似，工作的速率限制器是使用 `RateLimiter` Facade 的 `for` 方法來定義的。

例如，您可能希望允許使用者每小時備份一次資料，而對高級客戶則不設此限制。為了實現這一點，您可以在 `AppServiceProvider` 的 `boot` 方法中定義一個 `RateLimiter`：

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

在上述範例中，我們定義了每小時的速率限制；然而，您也可以使用 `perMinute` 方法輕鬆定義以分鐘為單位的速率限制。此外，您可以將任何您想要的值傳遞給速率限制的 `by` 方法；不過，此值最常用於根據客戶來區分速率限制：

```php
return Limit::perMinute(50)->by($job->user->id);
```

定義好速率限制後，您可以使用 `Illuminate\Queue\Middleware\RateLimited` 中介層將速率限制器附加到工作中。每當工作超過速率限制時，此中介層會根據速率限制的持續時間，將工作以適當的延遲重新放回佇列：

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

將受到速率限制的工作重新放回佇列仍會增加工作的總 `attempts` 次數。您可能需要相應地調整工作類別中的 `tries` 與 `maxExceptions` 屬性。或者，您可以使用 [retryUntil 方法](#time-based-attempts) 來定義工作不再嘗試之前的時間期限。

使用 `releaseAfter` 方法，您還可以指定在重新嘗試被放回的工作之前必須經過的秒數：

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

如果您不希望工作在受到速率限制時被重試，可以使用 `dontRelease` 方法：

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

如果您使用 Redis，可以使用 `Illuminate\Queue\Middleware\RateLimitedWithRedis` 中介層，它針對 Redis 進行了優化，比基礎的速率限制中介層更有效率：

```php
use Illuminate\Queue\Middleware\RateLimitedWithRedis;

public function middleware(): array
{
    return [new RateLimitedWithRedis('backups')];
}
```

可以使用 `connection` 方法來指定中介層應使用的 Redis 連線：

```php
return [(new RateLimitedWithRedis('backups'))->connection('limiter')];
```

<a name="preventing-job-overlaps"></a>
### 防止工作重疊

Laravel 包含一個 `Illuminate\Queue\Middleware\WithoutOverlapping` 中介層，允許您根據任意的金鑰來防止工作重疊。當一個佇列工作正在修改某個資源，且該資源一次只能由一個工作修改時，這會非常有用。

例如，假設您有一個佇列工作用於更新使用者的信用分數，且您希望防止同一個使用者 ID 的信用分數更新工作發生重疊。為了實現這一點，您可以從工作的 `middleware` 方法中回傳 `WithoutOverlapping` 中介層：

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

將重疊的工作釋放回佇列仍會增加該工作的總嘗試次數。您可能需要相應地調整工作類別上的 `tries` 和 `maxExceptions` 屬性。例如，將 `tries` 屬性維持預設值 1，將防止任何重疊的工作在稍後被重試。

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

如果您希望立即刪除任何重疊的工作以便不再重試，可以使用 `dontRelease` 方法：

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

`WithoutOverlapping` 中介層是由 Laravel 的原子鎖 (atomic lock) 功能驅動的。有時，您的工作可能會意外失敗或超時，導致鎖未被釋放。因此，您可以使用 `expireAfter` 方法明確定義鎖的過期時間。例如，下方的範例將指示 Laravel 在工作開始處理三分鐘後釋放 `WithoutOverlapping` 鎖：

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
> `WithoutOverlapping` 中介層需要一個支援 [鎖](/docs/{{version}}/cache#atomic-locks) 的快取驅動程式。目前，`memcached`、`redis`、`dynamodb`、`database`、`file` 和 `array` 快取驅動程式支援原子鎖。


<a name="sharing-lock-keys"></a>
#### 在工作類別之間共用鎖金鑰

預設情況下，`WithoutOverlapping` 中介層僅會防止相同類別的工作重疊。因此，即使兩個不同的工作類別使用相同的鎖金鑰，它們也不會被防止重疊。不過，您可以使用 `shared` 方法指示 Laravel 將金鑰應用於多個工作類別：

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
### 限制例外狀況 (Throttling Exceptions)

Laravel 包含了一個 `Illuminate\Queue\Middleware\ThrottlesExceptions` 中介層，讓您可以限制例外狀況。一旦工作拋出指定次數的例外狀況，之後所有執行該工作的嘗試都將被延遲，直到指定的時間間隔結束。此中介層對於與不穩定第三方服務互動的工作特別有用。

例如，假設有一個與第三方 API 互動且開始拋出例外狀況的佇列工作。要限制例外狀況，您可以從工作的 `middleware` 方法中回傳 `ThrottlesExceptions` 中介層。通常，此中介層應與實作了[以時間為基礎的嘗試](#time-based-attempts)的工作搭配使用：

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

此中介層接受的第一個建構子引數是工作在被限制前可以拋出的例外狀況次數，而第二個建構子引數則是工作被限制後，在再次嘗試之前應該經過的秒數。在上面的程式碼範例中，如果工作連續拋出 10 次例外狀況，我們將在 5 分鐘後再次嘗試執行該工作，但仍受限於 30 分鐘的時間限制。

當工作拋出例外狀況但尚未達到例外狀況閾值時，工作通常會立即被重試。然而，您可以在將中介層附加到工作時，透過呼叫 `backoff` 方法來指定此類工作應延遲的分鐘數：

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

在內部，此中介層使用 Laravel 的快取系統來實作速率限制，並將工作的類別名稱作為快取「金鑰」(key)。您可以在將中介層附加到工作時，透過呼叫 `by` 方法來覆寫此金鑰。如果您有多個工作與同一個第三方服務互動，並希望它們共用一個共同的限制「桶」(bucket) 以確保它們遵守單一的共用限制，這將會非常有用：

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

預設情況下，此中介層會限制所有的例外狀況。您可以在將中介層附加到工作時，透過呼叫 `when` 方法來修改此行為。只有當提供給 `when` 方法的閉包回傳 `true` 時，該例外狀況才會被限制：

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

與 `when` 方法（將工作釋放回佇列或拋出例外狀況）不同，`deleteWhen` 方法允許您在發生特定例外狀況時完全刪除該工作：

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

如果您希望將被限制的例外狀況回報給應用程式的例外狀況處理程式，您可以在將中介層附加到工作時，透過呼叫 `report` 方法來達成。您可以選擇性地為 `report` 方法提供一個閉包，且僅在該閉包回傳 `true` 時才會回報該例外狀況：

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
#### 使用 Redis 限制例外狀況

如果您使用 Redis，可以使用 `Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis` 中介層，它針對 Redis 進行了優化，比基本的例外狀況限制中介層更有效率：

```php
use Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis;

public function middleware(): array
{
    return [new ThrottlesExceptionsWithRedis(10, 10 * 60)];
}
```

`connection` 方法可用於指定中介層應使用哪一個 Redis 連線：

```php
return [(new ThrottlesExceptionsWithRedis(10, 10 * 60))->connection('limiter')];
```


<a name="skipping-jobs"></a>
### 跳過工作

`Skip` 中介層允許您指定工作應該被跳過或刪除，而不需要修改工作的邏輯。`Skip::when` 方法會在給定條件為 `true` 時刪除工作，而 `Skip::unless` 方法則會在條件為 `false` 時刪除工作：

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
## 派發工作

當您寫好工作類別後，可以使用工作本身的 `dispatch` 方法來派發它。傳遞給 `dispatch` 方法的引數將會被傳交給工作的建構子：

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

如果您想要根據條件派發工作，可以使用 `dispatchIf` 與 `dispatchUnless` 方法：

```php
ProcessPodcast::dispatchIf($accountActive, $podcast);

ProcessPodcast::dispatchUnless($accountSuspended, $podcast);
```

在新的 Laravel 應用程式中，`database` 連線被定義為預設佇列。您可以透過修改應用程式 `.env` 檔案中的 `QUEUE_CONNECTION` 環境變數，來指定不同的預設佇列連線。


<a name="delayed-dispatching"></a>
### 延遲派發

如果您想要指定工作不應立即被佇列工作者處理，可以在派發工作時使用 `delay` 方法。例如，我們指定工作在派發 10 分鐘後才可供處理：

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

在某些情況下，工作可能配置了預設延遲。如果您需要繞過此延遲並立即派發工作以供處理，可以使用 `withoutDelay` 方法：

```php
ProcessPodcast::dispatch($podcast)->withoutDelay();
```

> [!WARNING]
> Amazon SQS 佇列服務的最大延遲時間為 15 分鐘。


<a name="synchronous-dispatching"></a>
### 同步派發

如果您想要立即（同步）派發工作，可以使用 `dispatchSync` 方法。使用此方法時，工作不會被放入佇列，而是在目前的行程中立即執行：

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
#### 遞延派發

使用遞延同步派發，您可以派發一個工作在目前的行程中處理，但是在 HTTP 回應傳送給使用者之後才執行。這讓您能夠同步處理「佇列」工作，而不會降低使用者的應用程式體驗。若要遞延同步工作的執行，請將工作派發到 `deferred` 連線：

```php
RecordDelivery::dispatch($order)->onConnection('deferred');
```

`deferred` 連線同時也作為預設的 [故障轉移佇列](#queue-failover)。

同樣地，`background` 連線會在 HTTP 回應傳送給使用者之後處理工作；不過，該工作是在獨立產生的 PHP 行程中處理，讓 PHP-FPM / 應用程式工作者可以立即處理另一個傳入的 HTTP 請求：

```php
RecordDelivery::dispatch($order)->onConnection('background');
```


<a name="jobs-and-database-transactions"></a>
### 工作與資料庫交易

雖然在資料庫交易中派發工作是完全沒問題的，但您應該特別小心，以確保工作能夠成功執行。在交易中派發工作時，工作有可能在父交易提交（commit）之前就被工作者處理。當這種情況發生時，您在資料庫交易期間對模型或資料庫紀錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫紀錄可能還不存在於資料庫中。

幸運的是，Laravel 提供了幾種解決此問題的方法。首先，您可以在佇列連線的設定陣列中設定 `after_commit` 連線選項：

```php
'redis' => [
    'driver' => 'redis',
    // ...
    'after_commit' => true,
],
```

當 `after_commit` 選項為 `true` 時，您可以在資料庫交易中派發工作；不過，Laravel 會等到開啟的父資料庫交易提交之後，才會真正派發該工作。當然，如果目前沒有開啟任何資料庫交易，工作將立即被派發。

如果交易因期間發生的例外狀況而回滾（rolled back），則在該交易期間派發的工作將被捨棄。

> [!NOTE]
> 將 `after_commit` 設定選項設為 `true`，也會導致所有佇列化的事件監聽器、郵件類別 (mailables)、通知以及廣播事件在所有開啟的資料庫交易提交後才被派發。


<a name="specifying-commit-dispatch-behavior-inline"></a>
#### 行內指定提交派發行為

如果您沒有將 `after_commit` 佇列連線設定選項設為 `true`，您仍然可以指定某個特定工作應在所有開啟的資料庫交易提交後才被派發。要達成此目的，您可以將 `afterCommit` 方法鏈接到派發操作上：

```php
use App\Jobs\ProcessPodcast;

ProcessPodcast::dispatch($podcast)->afterCommit();
```

同樣地，如果 `after_commit` 設定選項已設為 `true`，您可以指定某個特定工作應立即派發，而無需等待任何開啟的資料庫交易提交：

```php
ProcessPodcast::dispatch($podcast)->beforeCommit();
```

<a name="job-chaining"></a>
### 工作鏈 (Job Chaining)

工作鏈 (Job Chaining) 允許您指定一組佇列工作，讓它們在主工作成功執行後依序執行。如果序列中的任何一個工作失敗，其餘的工作將不會被執行。若要執行佇列工作鏈，您可以使用 `Bus` Facade 提供的 `chain` 方法。Laravel 的指令匯流排 (command bus) 是一個底層元件，佇列工作的派發正是建立在其之上：

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
> 在工作內部使用 `$this->delete()` 方法刪除工作，並不會阻止鏈接的工作被處理。工作鏈只有在其中一個工作失敗時才會停止執行。


<a name="chain-connection-queue"></a>
#### 工作鏈連線與佇列

如果您想指定工作鏈應使用的連線與佇列，可以使用 `onConnection` 和 `onQueue` 方法。除非佇列工作被明確地指定了不同的連線或佇列，否則這些方法會指定該工作鏈所使用的佇列連線與佇列名稱：

```php
Bus::chain([
    new ProcessPodcast,
    new OptimizePodcast,
    new ReleasePodcast,
])->onConnection('redis')->onQueue('podcasts')->dispatch();
```


<a name="adding-jobs-to-the-chain"></a>
#### 將工作加入工作鏈

有時候，您可能需要在該鏈中的另一個工作內，將工作新增到現有工作鏈的前端或末端。您可以使用 `prependToChain` 和 `appendToChain` 方法來達成此目的：

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
#### 工作鏈失敗

在建立工作鏈時，您可以使用 `catch` 方法來指定一個閉包，當工作鏈中的工作失敗時即會調用該閉包。該回呼函數將接收到導致工作失敗的 `Throwable` 實例：

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
> 由於工作鏈的回呼函數會被序列化並由 Laravel 佇列在稍後時間執行，因此您不應該在工作鏈的回呼函數中使用 `$this` 變數。


<a name="customizing-the-queue-and-connection"></a>
### 自訂佇列與連線


<a name="dispatching-to-a-particular-queue"></a>
#### 派發至特定佇列

透過將工作推送至不同的佇列，您可以對佇列工作進行「分類」，甚至能優先決定分配多少個工作者 (worker) 給各個佇列。請注意，這並不是將工作推送至佇列設定檔中定義的不同佇列「連線」，而僅是單一連線內的不同特定佇列。若要指定佇列，請在派發工作時使用 `onQueue` 方法：

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

或者，您也可以在工作的建構子中呼叫 `onQueue` 方法來指定該工作的佇列：

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

> [!WARNING]
> 透過 `onQueue` 進行的建構子佇列指定僅適用於工作類別。對於 [佇列事件監聽器](/docs/{{version}}/events#customizing-the-queue-connection-queue-name)，請在監聽器類別中定義 `viaQueue` 方法或 `$queue` 屬性。


<a name="dispatching-to-a-particular-connection"></a>
#### 派發至特定連線

如果您的應用程式與多個佇列連線互動，您可以使用 `onConnection` 方法來指定要將工作推送至哪個連線：

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

您可以將 `onConnection` 和 `onQueue` 方法鏈接在一起，以同時指定工作的連線與佇列：

```php
ProcessPodcast::dispatch($podcast)
    ->onConnection('sqs')
    ->onQueue('processing');
```

或者，您也可以在工作的建構子中呼叫 `onConnection` 方法來指定該工作的連線：

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
### 指定最大嘗試次數 / 超時值


<a name="max-attempts"></a>
#### 最大嘗試次數

工作嘗試是 Laravel 佇列系統的核心概念，並為許多進階功能提供支援。雖然剛開始可能會覺得有些令人困惑，但在修改預設設定之前，理解其運作方式至關重要。

當工作被派發時，它會被推入佇列。接著工作者會取出該工作並嘗試執行它。這就稱為一次工作嘗試。

然而，一次嘗試並不一定意味著工作的 `handle` 方法被執行了。嘗試次數也可以透過以下幾種方式被「消耗」：

<div class="content-list" markdown="1">

- 工作在執行過程中遇到未處理的例外狀況。
- 使用 `$this->release()` 手動將工作重新放回佇列。
- 例如 `WithoutOverlapping` 或 `RateLimited` 等中介層無法取得鎖而將工作釋放。
- 工作超時。
- 工作的 `handle` 方法執行並完成且未拋出例外狀況。

</div>

您可能不希望無限次地嘗試執行某個工作。因此，Laravel 提供了多種方式來指定工作可以被嘗試的次數或時長。

> [!NOTE]
> 預設情況下，Laravel 只會嘗試執行工作一次。如果您的工作使用了 `WithoutOverlapping` 或 `RateLimited` 等中介層，或者您在手動釋放工作，您可能需要透過 `tries` 選項來增加允許的嘗試次數。

指定工作最大嘗試次數的一種方法是透過 Artisan 命令列的 `--tries` 選項。除非被處理的工作本身指定了可嘗試的次數，否則這將適用於該工作者處理的所有工作：

```shell
php artisan queue:work --tries=3
```

如果工作超過了最大嘗試次數，它將被視為「失敗」的工作。有關處理失敗工作的更多資訊，請參閱 [失敗工作文件](#dealing-with-failed-jobs)。如果向 `queue:work` 指令提供 `--tries=0`，則工作將被無限次重試。

您也可以採取更細緻的方法，直接在工作類別上定義最大嘗試次數。如果工作中指定了最大嘗試次數，它將優先於命令列提供的 `--tries` 值：

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

如果您需要對特定工作的最大嘗試次數進行動態控制，可以在工作中定義一個 `tries` 方法：

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

除了定義工作在失敗前可嘗試的次數外，您還可以定義一個工作不再被嘗試的時間點。這允許工作在給定的時間範圍內被嘗試任意次數。要定義工作不再被嘗試的時間，請在工作類別中加入 `retryUntil` 方法。此方法應回傳一個 `DateTime` 實例：

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
> 您也可以在 [佇列事件監聽器](/docs/{{version}}/events#queued-event-listeners) 和 [佇列通知](/docs/{{version}}/notifications#queueing-notifications) 上定義 `tries` 屬性或 `retryUntil` 方法。


<a name="max-exceptions"></a>
#### 最大例外狀況次數

有時您可能希望指定工作可以被嘗試多次，但如果重試是由於達到一定數量的未處理例外狀況所觸發（而非直接由 `release` 方法釋放），則應將其標記為失敗。要實現此功能，您可以在工作類別中定義 `maxExceptions` 屬性：

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

在這個範例中，如果應用程式無法取得 Redis 鎖，工作將被釋放十秒鐘，並將繼續重試最多 25 次。然而，如果工作拋出了三個未處理的例外狀況，該工作將失敗。


<a name="timeout"></a>
#### 超時

通常，您大致知道佇列工作預計需要花多少時間。因此，Laravel 允許您指定「超時」值。預設情況下，超時值為 60 秒。如果工作處理時間超過超時值指定的秒數，處理該工作的工作者將帶著錯誤退出。通常，工作者會由 [伺服器上設定的行程管理員](#supervisor-configuration) 自動重新啟動。

工作可執行的最大秒數可以使用 Artisan 命令列的 `--timeout` 選項來指定：

```shell
php artisan queue:work --timeout=30
```

如果工作因持續超時而超過最大嘗試次數，它將被標記為失敗。

您也可以在工作類別本身定義允許工作執行的最大秒數。如果在工作中指定了超時時間，它將優先於命令列指定的任何超時值：

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

有時，如通訊端 (sockets) 或對外 HTTP 連線等 IO 阻塞行程可能不會遵守您指定的超時時間。因此，在使用這些功能時，您應該始終嘗試透過它們的 API 同時指定超時時間。例如，使用 [Guzzle](https://docs.guzzlephp.org) 時，您應該始終指定連線和請求的超時值。

> [!WARNING]
> 必須安裝 [PCNTL](https://www.php.net/manual/en/book.pcntl.php) PHP 擴充功能才能指定工作超時。此外，工作的「超時」值應始終小於其 [「重試後」(#job-expiration)](#job-expiration) 的值。否則，工作可能會在實際執行完成或超時之前就被重新嘗試。


<a name="failing-on-timeout"></a>
#### 超時時標記為失敗

如果您希望在超時時將工作標記為 [失敗](#dealing-with-failed-jobs)，可以在工作類別中定義 `$failOnTimeout` 屬性：

```php
/**
 * Indicate if the job should be marked as failed on timeout.
 *
 * @var bool
 */
public $failOnTimeout = true;
```

> [!NOTE]
> 預設情況下，當工作超時時，它會消耗一次嘗試並重新放回佇列（如果允許重試）。然而，如果您將工作設定為超時時失敗，則無論 tries 設定為多少，它都不會被重試。

<a name="sqs-fifo-and-fair-queues"></a>
### SQS FIFO 與公平佇列

Laravel 支援 [Amazon SQS FIFO (First-In-First-Out)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html) 佇列，允許您按照工作被送出的確切順序來處理工作，同時透過訊息重複排除 (message deduplication) 來確保「精確一次 (exactly-once)」的處理。

FIFO 佇列需要一個訊息群組 ID，以決定哪些工作可以平行處理。具有相同群組 ID 的工作將依序處理，而具有不同群組 ID 的訊息則可以同時處理。

Laravel 提供了一個流暢的 `onGroup` 方法，用於在派發工作時指定訊息群組 ID：

```php
ProcessOrder::dispatch($order)
    ->onGroup("customer-{$order->customer_id}");
```

SQS FIFO 佇列支援訊息重複排除以確保精確一次處理。您可以在工作類別中實作 `deduplicationId` 方法，以提供自訂的重複排除 ID：

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

當使用 FIFO 佇列時，您還需要在監聽器、郵件和通知中定義訊息群組。或者，您可以將這些物件的佇列實例派發到非 FIFO 佇列中。

要為 [佇列事件監聽器](/docs/{{version}}/events#queued-event-listeners) 定義訊息群組，請在監聽器上定義 `messageGroup` 方法。您也可以選擇性地定義 `deduplicationId` 方法：

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

當發送將被加入 FIFO 佇列的 [郵件訊息](/docs/{{version}}/mail) 時，您應該在發送通知時呼叫 `onGroup` 方法，以及選擇性地呼叫 `withDeduplicator` 方法：

```php
use App\Mail\InvoicePaid;
use Illuminate\Support\Facades\Mail;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

Mail::to($request->user())->send($invoicePaid);
```

當發送將被加入 FIFO 佇列的 [通知](/docs/{{version}}/notifications) 時，您應該在發送通知時呼叫 `onGroup` 方法，以及選擇性地呼叫 `withDeduplicator` 方法：

```php
use App\Notifications\InvoicePaid;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

$user->notify($invoicePaid);
```


<a name="queue-failover"></a>
### 佇列故障轉移 (Queue Failover)

`failover` 佇列驅動程式在將工作推送到佇列時提供自動故障轉移功能。如果 `failover` 設定中的主佇列連線因任何原因失敗，Laravel 將自動嘗試將工作推送到清單中的下一個設定連線。這對於確保正式環境中佇列可靠性至關重要的高可用性特別有用。

要設定故障轉移佇列連線，請指定 `failover` 驅動程式並提供一個要依序嘗試的連線名稱陣列。預設情況下，Laravel 在您的應用程式 `config/queue.php` 設定檔中包含了一個故障轉移設定範例：

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

一旦您設定了使用 `failover` 驅動程式的連線，您需要將該故障轉移連線設定為 `.env` 檔案中的預設佇列連線，以使用故障轉移功能：

```ini
QUEUE_CONNECTION=failover
```

接下來，為故障轉移連線清單中的每個連線啟動至少一個工作者：

```bash
php artisan queue:work redis
php artisan queue:work database
```

> [!NOTE]
> 您不需要為使用 `sync`、`background` 或 `deferred` 佇列驅動程式的連線執行工作者，因為這些驅動程式在目前的 PHP 行程中處理工作。

當佇列連線操作失敗且啟動故障轉移時，Laravel 將派發 `Illuminate\Queue\Events\QueueFailedOver` 事件，讓您可以回報或記錄佇列連線已失敗。

> [!NOTE]
> 如果您使用 Laravel Horizon，請記得 Horizon 僅管理 Redis 佇列。如果您的故障轉移清單包含 `database`，您應該在 Horizon 之外執行一般的 `php artisan queue:work database` 行程。

<a name="error-handling"></a>
### 錯誤處理

如果工作在處理過程中拋出例外狀況，該工作將自動重新放回佇列，以便再次嘗試執行。工作將持續被重新放回，直到達到您的應用程式所允許的最大嘗試次數。最大嘗試次數是由 `queue:work` Artisan 指令中的 `--tries` 選項定義的。或者，也可以在工作類別本身定義最大嘗試次數。關於執行佇列工作者的更多資訊，[請參閱下方](#running-the-queue-worker)。


<a name="manually-releasing-a-job"></a>
#### 手動釋放工作

有時您可能希望手動將工作重新放回佇列，以便在稍後的時間再次嘗試執行。您可以透過呼叫 `release` 方法來達成此目的：

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

預設情況下，`release` 方法會將工作重新放回佇列以進行立即處理。然而，您可以透過向 `release` 方法傳遞整數或日期實例，來指示佇列在經過指定秒數後才允許處理該工作：

```php
$this->release(10);

$this->release(now()->plus(seconds: 10));
```


<a name="manually-failing-a-job"></a>
#### 手動將工作標記為失敗

有時您可能需要手動將工作標記為「失敗」。若要如此執行，您可以呼叫 `fail` 方法：

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

如果您想因為捕捉到某個例外狀況而將工作標記為失敗，您可以將該例外狀況傳遞給 `fail` 方法。或者，為了方便起見，您可以傳遞一個字串形式的錯誤訊息，系統會為您將其轉換為例外狀況：

```php
$this->fail($exception);

$this->fail('Something went wrong.');
```

> [!NOTE]
> 關於失敗工作的更多資訊，請參閱 [處理失敗工作的文件](#dealing-with-failed-jobs)。


<a name="fail-jobs-on-exceptions"></a>
#### 針對特定例外狀況將工作標記為失敗

`FailOnException` [工作中介層](#job-middleware) 允許您在拋出特定例外狀況時直接終止重試。這讓您可以在遇到暫時性例外狀況（例如外部 API 錯誤）時進行重試，但在遇到持續性例外狀況（例如使用者的權限被撤銷）時，將工作永久標記為失敗：

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
## 工作批處理 (Job Batching)

Laravel 的工作批處理功能讓您可以輕鬆地平行執行一組工作，並在該批工作完成執行後執行某些動作。

在開始之前，您應該建立一個資料庫遷移來建立一張資料表，用於儲存工作批處理的元資訊 (meta information)，例如完成百分比。您可以使用 `make:queue-batches-table` Artisan 指令來產生此遷移：

```shell
php artisan make:queue-batches-table

php artisan migrate
```


<a name="defining-batchable-jobs"></a>
### 定義可批處理的工作

要定義一個可批處理的工作，您應該像平常一樣[建立一個可加入佇列的工作](#creating-jobs)；但您必須在工作類別中加入 `Illuminate\Bus\Batchable` trait。這個 trait 提供了 `batch` 方法，可用於取得該工作目前執行所在的批處理：

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
### 派發批處理

要派發一批工作，您應該使用 `Bus` Facade 的 `batch` 方法。當然，批處理在與完成回呼 (completion callbacks) 結合使用時最為有用。因此，您可以使用 `then`、`catch` 和 `finally` 方法來定義批處理的完成回呼。這些回呼在被調用時，都會接收到一個 `Illuminate\Bus\Batch` 實例。

當執行多個佇列工作者時，批處理中的工作將會平行處理。因此，工作完成的順序可能與它們被加入批處理的順序不同。請參閱我們的[工作鏈與批處理](#chains-and-batches)文件，以了解如何按順序執行一系列工作。

在此範例中，我們假設正在將一批工作加入佇列，每個工作處理 CSV 檔案中的指定行數：

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

批處理的 ID（可透過 `$batch->id` 屬性存取）可用於在派發後[查詢 Laravel 命令匯流排 (command bus)](#inspecting-batches) 以獲取關於該批處理的資訊。

> [!WARNING]
> 由於批處理回呼會被序列化並在稍後由 Laravel 佇列執行，因此您不應在回呼中使用 `$this` 變數。此外，由於批處理工作被封裝在資料庫交易中，因此不應在工作中執行會觸發隱式提交 (implicit commits) 的資料庫陳述式。


<a name="naming-batches"></a>
#### 為批處理命名

如果為批處理命名，某些工具（例如 [Laravel Horizon](/docs/{{version}}/horizon) 和 [Laravel Telescope](/docs/{{version}}/telescope)）可以提供更人性化的批處理偵錯資訊。要為批處理指定一個任意名稱，您可以在定義批處理時調用 `name` 方法：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import CSV')->dispatch();
```


<a name="batch-connection-queue"></a>
#### 批處理連線與佇列

如果您想要指定批處理工作應使用的連線與佇列，可以使用 `onConnection` 和 `onQueue` 方法。所有批處理工作必須在相同的連線與佇列中執行：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->onConnection('redis')->onQueue('imports')->dispatch();
```


<a name="chains-and-batches"></a>
### 鏈與批處理

您可以透過將鏈接工作放入陣列中，在批處理內定義一組[工作鏈](#job-chaining)。例如，我們可以平行執行兩個工作鏈，並在兩個工作鏈都完成處理後執行一個回呼：

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

相反地，您也可以透過在[工作鏈](#job-chaining)中定義批處理，來在鏈中執行批處理工作。例如，您可以先執行一批工作來發布多個 Podcast，然後再執行一批工作來發送發布通知：

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
### 將工作加入批處理

有時候，從批處理工作中將額外的工作加入到該批處理中會很有用。當您需要批處理數千個工作，且在一次 Web 請求中派發這些工作可能耗時過長時，這種模式非常有用。因此，您可以選擇先派發一批「載入 (loader)」工作，由這些工作在批處理中填充更多的工作：

```php
$batch = Bus::batch([
    new LoadImportBatch,
    new LoadImportBatch,
    new LoadImportBatch,
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import Contacts')->dispatch();
```

在此範例中，我們將使用 `LoadImportBatch` 工作來為批處理填充額外的工作。要實現這一點，我們可以使用批處理實例上的 `add` 方法，而該實例可透過工作的 `batch` 方法取得：

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
> 您只能從屬於同一批處理的工作中，將新工作加入到該批處理。

<a name="inspecting-batches"></a>
### 檢查批處理

提供給批處理完成回呼 (callback) 的 `Illuminate\Bus\Batch` 實例具有多種屬性與方法，可協助您與特定的工作批處理進行互動並檢查其狀態：

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
#### 從路由回傳批處理

所有 `Illuminate\Bus\Batch` 實例皆可 JSON 序列化，這意味著您可以直接從應用程式的路由中回傳它們，以獲取包含批處理資訊（包括完成進度）的 JSON 載荷 (payload)。這讓您能方便地在應用程式的 UI 中顯示批處理的完成進度資訊。

若要透過 ID 檢索批處理，您可以使用 `Bus` Facade 的 `findBatch` 方法：

```php
use Illuminate\Support\Facades\Bus;
use Illuminate\Support\Facades\Route;

Route::get('/batch/{batchId}', function (string $batchId) {
    return Bus::findBatch($batchId);
});
```


<a name="cancelling-batches"></a>
### 取消批處理

有時您可能需要取消特定批處理的執行。您可以透過呼叫 `Illuminate\Bus\Batch` 實例上的 `cancel` 方法來實現：

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

正如您在之前的範例中所看到的，批處理工作在繼續執行之前，通常應該先判斷其對應的批處理是否已被取消。不過，為了方便起見，您可以改為將 `SkipIfBatchCancelled` [中介層](#job-middleware) 分配給該工作。正如其名稱所示，如果對應的批處理已被取消，此中介層將指示 Laravel 不要處理該工作：

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
### 批處理失敗

當批處理工作失敗時，會觸發 `catch` 回呼（如果已指定）。此回呼僅會在批處理中第一個失敗的工作時被觸發。


<a name="allowing-failures"></a>
#### 允許失敗

當批處理中的工作失敗時，Laravel 會自動將該批處理標記為「已取消 (cancelled)」。如果您希望工作失敗時不會自動將批處理標記為已取消，可以透過在派發批處理時呼叫 `allowFailures` 方法來停用此行為：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->allowFailures()->dispatch();
```

您也可以選擇向 `allowFailures` 方法提供一個閉包，該閉包將在每次工作失敗時執行：

```php
$batch = Bus::batch([
    // ...
])->allowFailures(function (Batch $batch, $exception) {
    // Handle individual job failures...
})->dispatch();
```


<a name="retrying-failed-batch-jobs"></a>
#### 重試失敗的批處理工作

為了方便起見，Laravel 提供了 `queue:retry-batch` Artisan 指令，讓您可以輕鬆地重試特定批處理中所有失敗的工作。此指令接收需要重試失敗工作的批處理 UUID：

```shell
php artisan queue:retry-batch 32dbc76c-4f82-4749-b610-a639fe0099b5
```


<a name="pruning-batches"></a>
### 修剪批處理

如果不進行修剪，`job_batches` 資料表會迅速累積紀錄。為了緩解這個問題，您應該[排程](/docs/{{version}}/scheduling) `queue:prune-batches` Artisan 指令每日執行：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches')->daily();
```

預設情況下，所有完成且超過 24 小時的批處理都將被修剪。您可以在呼叫指令時使用 `hours` 選項來決定保留批處理資料的時間。例如，下列指令將刪除所有在 48 小時前完成的批處理：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48')->daily();
```

有時，您的 `job_batches` 資料表可能會累積從未成功完成的批處理紀錄，例如其中某個工作失敗且該工作從未被成功重試。您可以使用 `unfinished` 選項指示 `queue:prune-batches` 指令修剪這些未完成的批處理紀錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --unfinished=72')->daily();
```

同樣地，您的 `job_batches` 資料表也可能累積已取消批處理的紀錄。您可以使用 `cancelled` 選項指示 `queue:prune-batches` 指令修剪這些已取消的批處理紀錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --cancelled=72')->daily();
```

<a name="storing-batches-in-dynamodb"></a>
### 將批處理儲存在 DynamoDB 中

Laravel 同樣提供將批處理的元數據（meta information）儲存在 [DynamoDB](https://aws.amazon.com/dynamodb) 而非關聯式資料庫的支持。然而，您需要手動建立一個 DynamoDB 資料表來儲存所有的批處理紀錄。

通常，這個資料表應命名為 `job_batches`，但您應該根據應用程式 `queue` 設定檔中 `queue.batching.table` 設定值來命名該資料表。


<a name="dynamodb-batch-table-configuration"></a>
#### DynamoDB 批處理資料表設定

`job_batches` 資料表應該具有一個名為 `application` 的字串主分區鍵（primary partition key）以及一個名為 `id` 的字串主排序鍵（primary sort key）。鍵中的 `application` 部分將包含您的應用程式名稱，該名稱由應用程式 `app` 設定檔中的 `name` 設定值定義。由於應用程式名稱是 DynamoDB 資料表鍵的一部分，因此您可以使用同一個資料表來儲存多個 Laravel 應用程式的工作批處理。

此外，如果您想利用 [自動批處理修剪](#pruning-batches-in-dynamodb)，可以為您的資料表定義 `ttl` 屬性。


<a name="dynamodb-configuration"></a>
#### DynamoDB 設定

接下來，安裝 AWS SDK 以便您的 Laravel 應用程式能與 Amazon DynamoDB 通訊：

```shell
composer require aws/aws-sdk-php
```

然後，將 `queue.batching.driver` 設定選項的值設定為 `dynamodb`。此外，您應該在 `batching` 設定陣列中定義 `key`、`secret` 和 `region` 設定選項。這些選項將用於 AWS 認證。使用 `dynamodb` 驅動程式時，不需要 `queue.batching.database` 設定選項：

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
#### 在 DynamoDB 中修剪批處理

當利用 [DynamoDB](https://aws.amazon.com/dynamodb) 儲存工作批處理資訊時，用於修剪儲存在關聯式資料庫中批處理的典型修剪指令將無法運作。相反地，您可以利用 [DynamoDB 的原生 TTL 功能](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html) 來自動移除舊批處理的紀錄。

如果您在定義 DynamoDB 資料表時設定了 `ttl` 屬性，您可以定義設定參數來指示 Laravel 如何修剪批處理紀錄。`queue.batching.ttl_attribute` 設定值定義了持有 TTL 的屬性名稱，而 `queue.batching.ttl` 設定值則定義了相對於紀錄最後更新時間後，多少秒後該批處理紀錄可以從 DynamoDB 資料表中移除：

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
## 將閉包加入佇列

您可以選擇派發閉包到佇列，而不是派發工作類別。這對於需要在目前請求週期之外執行的快速且簡單的任務非常有用。當將閉包派發到佇列時，閉包的程式碼內容會經過加密簽署，以確保在傳輸過程中不會被修改：

```php
use App\Models\Podcast;

$podcast = Podcast::find(1);

dispatch(function () use ($podcast) {
    $podcast->publish();
});
```

若要為加入佇列的閉包指定一個名稱（可用於佇列報告儀表板或由 `queue:work` 指令顯示），您可以使用 `name` 方法：

```php
dispatch(function () {
    // ...
})->name('Publish Podcast');
```

您可以使用 `catch` 方法提供一個閉包，當加入佇列的閉包在耗盡所有[已設定的重試次數](#max-job-attempts-and-timeout)後仍無法成功完成時，將執行該閉包：

```php
use Throwable;

dispatch(function () use ($podcast) {
    $podcast->publish();
})->catch(function (Throwable $e) {
    // This job has failed...
});
```

> [!WARNING]
> 由於 `catch` 回呼函數會被序列化並在稍後由 Laravel 佇列執行，因此您不應在 `catch` 回呼函數中使用 `$this` 變數。

<a name="running-the-queue-worker"></a>
## 執行佇列工作者 (Queue Worker)


<a name="the-queue-work-command"></a>
### `queue:work` 指令

Laravel 提供了一個 Artisan 指令，可用於啟動佇列工作者並在工作被推送到佇列時對其進行處理。您可以使用 `queue:work` Artisan 指令來執行工作者。請注意，一旦 `queue:work` 指令啟動，它將持續執行，直到被手動停止或您關閉終端機為止：

```shell
php artisan queue:work
```

> [!NOTE]
> 若要讓 `queue:work` 行程永久在背景執行，您應該使用如 [Supervisor](#supervisor-configuration) 之類的行程監控工具，以確保佇列工作者不會停止執行。

如果您希望在指令的輸出中包含處理的工作 ID、連線名稱和佇列名稱，可以在調用 `queue:work` 指令時加上 `-v` 旗標：

```shell
php artisan queue:work -v
```

請記住，佇列工作者是長生命週期的行程，且會將啟動後的應用程式狀態儲存在記憶體中。因此，在工作者啟動後，它們不會注意到程式碼庫中的變更。因此，在部署過程中，請務必[重新啟動您的佇列工作者](#queue-workers-and-deployment)。此外，請記住應用程式建立或修改的任何靜態狀態，在工作之間不會自動重設。

或者，您也可以執行 `queue:listen` 指令。使用 `queue:listen` 指令時，當您想要重新載入更新的程式碼或重設應用程式狀態時，不需要手動重新啟動工作者；然而，此指令的效率顯著低於 `queue:work` 指令：

```shell
php artisan queue:listen
```


<a name="running-multiple-queue-workers"></a>
#### 執行多個佇列工作者

若要為一個佇列分配多個工作者並同時處理工作，您只需啟動多個 `queue:work` 行程即可。這可以在本地透過終端機的多個分頁完成，或在正式環境中使用行程管理工具的設定來達成。[使用 Supervisor](#supervisor-configuration) 時，您可以使用 `numprocs` 設定值。


<a name="specifying-the-connection-queue"></a>
#### 指定連線與佇列

您也可以指定工作者應該使用哪個佇列連線。傳遞給 `work` 指令的連線名稱應該與 `config/queue.php` 設定檔中定義的其中一個連線對應：

```shell
php artisan queue:work redis
```

預設情況下，`queue:work` 指令僅處理給定連線上預設佇列的工作。然而，您可以進一步自訂佇列工作者，使其僅處理給定連線中的特定佇列。例如，如果您的所有電子郵件都在 `redis` 佇列連線的 `emails` 佇列中處理，您可以發出以下指令來啟動僅處理該佇列的工作者：

```shell
php artisan queue:work redis --queue=emails
```


<a name="processing-a-specified-number-of-jobs"></a>
#### 處理指定數量的工作

`--once` 選項可用於指示工作者僅從佇列中處理單個工作：

```shell
php artisan queue:work --once
```

`--max-jobs` 選項可用於指示工作者處理指定數量的工作後退出。當與 [Supervisor](#supervisor-configuration) 結合使用時，此選項非常有用，因為工作者在處理指定數量的工作後會自動重新啟動，從而釋放它們可能累積的記憶體：

```shell
php artisan queue:work --max-jobs=1000
```


<a name="processing-all-queued-jobs-then-exiting"></a>
#### 處理所有佇列工作後退出

`--stop-when-empty` 選項可用於指示工作者處理所有工作後優雅地退出。如果您在 Docker 容器中處理 Laravel 佇列，並希望在佇列清空後關閉容器，此選項會非常有用：

```shell
php artisan queue:work --stop-when-empty
```


<a name="processing-jobs-for-a-given-number-of-seconds"></a>
#### 處理指定秒數的工作

`--max-time` 選項可用於指示工作者處理工作指定秒數後退出。當與 [Supervisor](#supervisor-configuration) 結合使用時，此選項非常有用，因為工作者在處理工作指定時間後會自動重新啟動，從而釋放它們可能累積的記憶體：

```shell
# Process jobs for one hour and then exit...
php artisan queue:work --max-time=3600
```


<a name="worker-sleep-duration"></a>
#### 工作者睡眠時間

當佇列中有可用工作時，工作者將在工作之間不延遲地持續處理工作。然而，`sleep` 選項決定了在沒有可用工作時，工作者將「睡眠」多少秒。當然，在睡眠期間，工作者不會處理任何新工作：

```shell
php artisan queue:work --sleep=3
```


<a name="maintenance-mode-queues"></a>
#### 維護模式與佇列

當您的應用程式處於[維護模式](/docs/{{version}}/configuration#maintenance-mode) 時，不會處理任何佇列工作。一旦應用程式退出維護模式，工作將恢復正常處理。

若要強制佇列工作者即使在維護模式啟用時也處理工作，您可以使用 `--force` 選項：

```shell
php artisan queue:work --force
```


<a name="resource-considerations"></a>
#### 資源考量

守護行程(Daemon)佇列工作者在處理每個工作之前不會「重新啟動」框架。因此，您應該在每個工作完成後釋放任何沉重的資源。例如，如果您使用 [GD 函式庫](https://www.php.net/manual/en/book.image.php) 進行影像處理，在完成影像處理後，您應該使用 `imagedestroy` 釋放記憶體。


<a name="queue-priorities"></a>
### 佇列優先權

有時您可能希望優先處理某些佇列。例如，在 `config/queue.php` 設定檔中，您可以將 `redis` 連線的預設 `queue` 設定為 `low`。然而，偶爾您可能希望將工作推送到 `high` 優先權佇列，如下所示：

```php
dispatch((new Job)->onQueue('high'));
```

若要啟動一個在繼續處理 `low` 佇列工作之前，先確保所有 `high` 佇列工作都已處理的工作者，請向 `work` 指令傳遞以逗號分隔的佇列名稱列表：

```shell
php artisan queue:work --queue=high,low
```


<a name="queue-workers-and-deployment"></a>
### 佇列工作者與部署

由於佇列工作者是長生命週期的行程，如果不重新啟動，它們不會注意到程式碼的變更。因此，部署使用佇列工作者的應用程式最簡單的方法是在部署過程中重新啟動工作者。您可以使用 `queue:restart` 指令來優雅地重新啟動所有工作者：

```shell
php artisan queue:restart
```

此指令將指示所有佇列工作者在完成目前處理的工作後優雅地退出，以確保沒有現有的工作丟失。由於執行 `queue:restart` 指令時佇列工作者會退出，您應該執行如 [Supervisor](#supervisor-configuration) 之類的行程管理工具，以自動重新啟動佇列工作者。

> [!NOTE]
> 佇列使用[快取](/docs/{{version}}/cache)來儲存重新啟動訊號，因此在使用此功能之前，您應該驗證應用程式是否已正確設定快取驅動程式。

<a name="job-expirations-and-timeouts"></a>
### 工作過期與超時


<a name="job-expiration"></a>
#### 工作過期

在您的 `config/queue.php` 設定檔中，每個佇列連線都定義了一個 `retry_after` 選項。此選項指定了佇列連線在重試正在處理的工作之前應該等待多少秒。例如，如果 `retry_after` 的值設定為 `90`，則該工作在處理 90 秒且未被釋放或刪除時，將被重新放回佇列中。通常，您應該將 `retry_after` 的值設定為您的工作在合理情況下完成處理所需的最長秒數。

> [!WARNING]
> 唯一不包含 `retry_after` 值的佇列連線是 Amazon SQS。SQS 將根據在 AWS 控制台管理的 [Default Visibility Timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html) 來重試工作。


<a name="worker-timeouts"></a>
#### 工作者超時

`queue:work` Artisan 指令提供了一個 `--timeout` 選項。預設情況下，`--timeout` 的值為 60 秒。如果工作處理的時間超過了超時值指定的秒數，處理該工作的工作者將以錯誤結束。通常，工作者會由您伺服器上[設定的行程管理員](#supervisor-configuration)自動重新啟動：

```shell
php artisan queue:work --timeout=60
```

`retry_after` 設定選項與 `--timeout` CLI 選項雖然不同，但它們共同運作以確保工作不會遺失，且每個工作僅被成功處理一次。

> [!WARNING]
> `--timeout` 的值應始終比您的 `retry_after` 設定值至少短幾秒。這將確保處理凍結工作的工作者在工作被重試之前總是會被終止。如果您的 `--timeout` 選項比 `retry_after` 設定值長，您的工作可能會被處理兩次。


<a name="pausing-and-resuming-queue-workers"></a>
### 暫停與恢復佇列工作者

有時您可能需要暫時防止佇列工作者處理新工作，而不需要完全停止工作者。例如，您可能希望在系統維護期間暫停工作處理。Laravel 提供了 `queue:pause` 和 `queue:continue` Artisan 指令來暫停和恢復佇列工作者。

要暫停特定的佇列，請提供佇列連線名稱和佇列名稱：

```shell
php artisan queue:pause database:default
```

在這個範例中，`database` 是佇列連線名稱，而 `default` 是佇列名稱。一旦佇列被暫停，任何從該佇列處理工作的工作者將繼續完成目前的工作，但在佇列恢復之前不會獲取任何新工作。

要恢復處理已暫停佇列中的工作，請使用 `queue:continue` 指令：

```shell
php artisan queue:continue database:default
```

恢復佇列後，工作者將立即開始處理該佇列中的新工作。請注意，暫停佇列並不會停止工作者行程本身 - 它僅是防止工作者從指定的佇列中處理新工作。


<a name="worker-restart-and-pause-signals"></a>
#### 工作者重新啟動與暫停訊號

預設情況下，佇列工作者在每次工作迭代時都會輪詢快取驅動程式以獲取重新啟動和暫停訊號。雖然這種輪詢對於回應 `queue:restart` 和 `queue:pause` 指令至關重要，但它確實會帶來少量的效能開銷。

如果您需要最佳化效能且不需要這些中斷功能，可以透過呼叫 `Queue` Facade 的 `withoutInterruptionPolling` 方法來全域停用此輪詢。這通常應該在 `AppServiceProvider` 的 `boot` 方法中執行：

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

或者，您也可以透過設定 `Illuminate\Queue\Worker` 類別上的靜態 `$restartable` 或 `$pausable` 屬性，來單獨停用重新啟動或暫停輪詢：

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
> 當中斷輪詢被停用時，工作者將不會回應 `queue:restart` 或 `queue:pause` 指令（取決於停用了哪些功能）。

<a name="supervisor-configuration"></a>
## Supervisor 設定

在生產環境中，您需要一種方式來讓 `queue:work` 行程持續執行。`queue:work` 行程可能會因為各種原因停止執行，例如工作者超時或執行了 `queue:restart` 指令。

因此，您需要設定一個行程監控程式，以便在 `queue:work` 行程結束時能偵測到並自動重新啟動它們。此外，行程監控程式還允許您指定要同時執行多少個 `queue:work` 行程。Supervisor 是 Linux 環境中常用的行程監控程式，我們將在接下來的文件中討論如何設定它。


<a name="installing-supervisor"></a>
#### 安裝 Supervisor

Supervisor 是 Linux 作業系統的行程監控程式，如果您的 `queue:work` 行程失敗，它會自動重新啟動。若要在 Ubuntu 上安裝 Supervisor，可以使用以下指令：

```shell
sudo apt-get install supervisor
```

> [!NOTE]
> 如果您覺得自行設定與管理 Supervisor 太過繁瑣，可以考慮使用 [Laravel Cloud](https://cloud.laravel.com)，它提供了一個完全託管的平台來執行 Laravel 佇列工作者。


<a name="configuring-supervisor"></a>
#### 設定 Supervisor

Supervisor 的設定檔通常儲存在 `/etc/supervisor/conf.d` 目錄中。在此目錄中，您可以建立任意數量的設定檔，用來指示 Supervisor 應如何監控您的行程。例如，讓我們建立一個 `laravel-worker.conf` 檔案來啟動並監控 `queue:work` 行程：

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

在此範例中，`numprocs` 指令將指示 Supervisor 執行 8 個 `queue:work` 行程並監控所有行程，若其失敗則自動重新啟動。您應該更改設定中的 `command` 指令，以符合您想要的佇列連線與工作者選項。

> [!WARNING]
> 您應該確保 `stopwaitsecs` 的值大於執行時間最長的工作所耗費的秒數。否則，Supervisor 可能會在工作完成處理之前就將其強制終止。


<a name="starting-supervisor"></a>
#### 啟動 Supervisor

建立設定檔後，您可以使用以下指令來更新 Supervisor 設定並啟動行程：

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start "laravel-worker:*"
```

如需更多關於 Supervisor 的資訊，請參閱 [Supervisor 官方文件](http://supervisord.org/index.html)。

<a name="dealing-with-failed-jobs"></a>
## 處理失敗的工作

有時候您的佇列工作會失敗。別擔心，事情並不總是按計劃進行！Laravel 提供了一種便捷的方法來 [指定工作應該嘗試的最大次數](#max-job-attempts-and-timeout)。當一個非同步工作超過此嘗試次數後，它將被插入到 `failed_jobs` 資料庫資料表中。[同步派發的工作](/docs/{{version}}/queues#synchronous-dispatching) 如果失敗，則不會儲存在此表中，其例外狀況將立即由應用程式處理。

在新的 Laravel 應用程式中，通常已經包含了一個建立 `failed_jobs` 表的遷移檔。然而，如果您的應用程式中沒有這個表的遷移檔，您可以使用 `make:queue-failed-table` Artisan 指令來建立它：

```shell
php artisan make:queue-failed-table

php artisan migrate
```

在執行 [佇列工作者](#running-the-queue-worker) 行程時，您可以在 `queue:work` 指令中使用 `--tries` 選項來指定工作應該嘗試的最大次數。如果您沒有為 `--tries` 選項指定數值，工作將僅嘗試一次，或者按照工作類別的 `$tries` 屬性所指定的次數進行嘗試：

```shell
php artisan queue:work redis --tries=3
```

使用 `--backoff` 選項，您可以指定 Laravel 在重試遇到例外狀況的工作之前應該等待多少秒。預設情況下，工作會立即被釋放回佇列中以便再次嘗試：

```shell
php artisan queue:work redis --tries=3 --backoff=3
```

如果您想針對單一工作設定 Laravel 在重試遇到例外狀況的工作之前應等待的秒數，您可以在工作類別中定義 `backoff` 屬性：

```php
/**
 * The number of seconds to wait before retrying the job.
 *
 * @var int
 */
public $backoff = 3;
```

如果您需要更複雜的邏輯來決定工作的退避時間 (backoff time)，您可以在工作類別中定義 `backoff` 方法：

```php
/**
 * Calculate the number of seconds to wait before retrying the job.
 */
public function backoff(): int
{
    return 3;
}
```

您可以透過從 `backoff` 方法回傳一個退避值陣列，輕鬆地設定「指數型」退避。在此範例中，第一次重試的延遲為 1 秒，第二次為 5 秒，第三次為 10 秒，如果還有剩餘嘗試次數，則之後的每次重試延遲均為 10 秒：

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

當特定工作失敗時，您可能想要向使用者發送警報，或者還原該工作部分完成的任何操作。為了實現這一點，您可以在工作類別中定義一個 `failed` 方法。導致工作失敗的 `Throwable` 實例將被傳遞給 `failed` 方法：

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
> 在呼叫 `failed` 方法之前會重新實例化一個新的工作實例；因此，在 `handle` 方法中可能發生的任何類別屬性修改都將會遺失。

失敗的工作並不一定是指遇到了未處理的例外狀況。當工作用盡了所有允許的嘗試次數時，也會被視為失敗。這些嘗試次數可能透過以下幾種方式被消耗：

<div class="content-list" markdown="1">

- 工作超時。
- 工作在執行期間遇到未處理的例外狀況。
- 工作被手動或透過中介層釋放回佇列。

</div>

如果最後一次嘗試因為執行工作期間拋出的例外狀況而失敗，該例外狀況將被傳遞到工作的 `failed` 方法中。然而，如果工作是因為達到最大允許嘗試次數而失敗，則 `$exception` 將會是 `Illuminate\Queue\MaxAttemptsExceededException` 的實例。同樣地，如果工作是因為超過設定的超時時間而失敗，則 `$exception` 將會是 `Illuminate\Queue\TimeoutExceededException` 的實例。


<a name="retrying-failed-jobs"></a>
### 重試失敗的工作

若要查看所有已插入 `failed_jobs` 資料庫表中的失敗工作，您可以使用 `queue:failed` Artisan 指令：

```shell
php artisan queue:failed
```

`queue:failed` 指令將列出工作 ID、連線、佇列、失敗時間以及關於該工作的其他資訊。工作 ID 可用於重試失敗的工作。例如，要重試一個 ID 為 `ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece` 的失敗工作，請執行以下指令：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece
```

如有必要，您可以向該指令傳遞多個 ID：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece 91401d2c-0784-4f43-824c-34f94a33c24d
```

您也可以重試特定佇列的所有失敗工作：

```shell
php artisan queue:retry --queue=name
```

若要重試所有失敗的工作，請執行 `queue:retry` 指令並傳遞 `all` 作為 ID：

```shell
php artisan queue:retry all
```

如果您想要刪除一個失敗的工作，可以使用 `queue:forget` 指令：

```shell
php artisan queue:forget 91401d2c-0784-4f43-824c-34f94a33c24d
```

> [!NOTE]
> 當使用 [Horizon](/docs/{{version}}/horizon) 時，您應該使用 `horizon:forget` 指令來刪除失敗的工作，而不是使用 `queue:forget` 指令。

若要從 `failed_jobs` 表中刪除所有失敗的工作，可以使用 `queue:flush` 指令：

```shell
php artisan queue:flush
```

`queue:flush` 指令會從您的佇列中移除所有失敗的工作記錄，無論該失敗工作有多久。您可以使用 `--hours` 選項來僅刪除在特定小時前或更早失敗的工作：

```shell
php artisan queue:flush --hours=48
```


<a name="ignoring-missing-models"></a>
### 忽略缺失的模型

在將 Eloquent 模型注入工作時，模型在被放入佇列之前會自動被序列化，並在工作被處理時從資料庫中重新擷取。然而，如果模型在工作等待工作者處理期間被刪除了，您的工作可能會因為 `ModelNotFoundException` 而失敗。

為了方便起見，您可以選擇透過將工作的 `deleteWhenMissingModels` 屬性設定為 `true` 來自動刪除缺失模型的工作。當此屬性設定為 `true` 時，Laravel 將會悄悄地捨棄該工作而不會拋出例外狀況：

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

您可以透過呼叫 `queue:prune-failed` Artisan 指令來修剪應用程式 `failed_jobs` 表中的記錄：

```shell
php artisan queue:prune-failed
```

預設情況下，所有超過 24 小時的失敗工作記錄都將被修剪。如果您向該指令提供 `--hours` 選項，則僅會保留在過去 N 小時內插入的失敗工作記錄。例如，以下指令將刪除所有在 48 小時前插入的失敗工作記錄：

```shell
php artisan queue:prune-failed --hours=48
```

<a name="storing-failed-jobs-in-dynamodb"></a>
### 將失敗的工作儲存在 DynamoDB 中

Laravel 同樣提供將失敗工作記錄儲存在 [DynamoDB](https://aws.amazon.com/dynamodb) 而非關聯式資料庫表格的支持。然而，您必須手動建立一個 DynamoDB 表格來儲存所有失敗的工作記錄。通常，此表格應命名為 `failed_jobs`，但您應該根據應用程式 `queue` 設定檔中 `queue.failed.table` 設定值來命名表格。

`failed_jobs` 表格應該有一個名為 `application` 的字串主分區鍵 (primary partition key) 以及一個名為 `uuid` 的字串主排序鍵 (primary sort key)。鍵中的 `application` 部分將包含您的應用程式名稱，該名稱由應用程式 `app` 設定檔中的 `name` 設定值定義。由於應用程式名稱是 DynamoDB 表格鍵的一部分，因此您可以使用同一個表格來儲存多個 Laravel 應用程式的失敗工作。

此外，請確保安裝 AWS SDK，以便您的 Laravel 應用程式能與 Amazon DynamoDB 進行通訊：

```shell
composer require aws/aws-sdk-php
```

接著，將 `queue.failed.driver` 設定選項的值設定為 `dynamodb`。此外，您應該在失敗工作設定陣列中定義 `key`、`secret` 與 `region` 設定選項。這些選項將用於 AWS 認證。使用 `dynamodb` 驅動程式時，不需要 `queue.failed.database` 設定選項：

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

您可以將 `queue.failed.driver` 設定選項的值設定為 `null`，以指示 Laravel 在不儲存的情況下捨棄失敗的工作。通常，這可以透過 `QUEUE_FAILED_DRIVER` 環境變數來達成：

```ini
QUEUE_FAILED_DRIVER=null
```


<a name="failed-job-events"></a>
### 失敗工作事件

如果您想註冊一個在工作失敗時會被觸發的事件監聽器，可以使用 `Queue` Facade 的 `failing` 方法。例如，我們可以在 Laravel 提供的 `AppServiceProvider` 的 `boot` 方法中，將一個閉包附加到此事件上：

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
## 從佇列中清除工作

> [!NOTE]
> 當使用 [Horizon](/docs/{{version}}/horizon) 時，您應該使用 `horizon:clear` 指令來從佇列中清除工作，而不是使用 `queue:clear` 指令。

如果您想要從預設連線的預設佇列中刪除所有工作，可以使用 `queue:clear` Artisan 指令來達成：

```shell
php artisan queue:clear
```

您也可以提供 `connection` 引數與 `queue` 選項，以從特定的連線與佇列中刪除工作：

```shell
php artisan queue:clear redis --queue=emails
```

> [!WARNING]
> 從佇列中清除工作僅適用於 SQS、Redis 與 database 佇列驅動程式。此外，SQS 的訊息刪除過程最多需要 60 秒，因此在您清除佇列後 60 秒內傳送到 SQS 佇列的工作也可能會被刪除。


<a name="monitoring-your-queues"></a>
## 監控您的佇列

如果您的佇列突然湧入大量工作，可能會導致負荷過重，進而使工作完成的等待時間增加。如果您需要，Laravel 可以在佇列的工作數量超過指定閾值時向您發出警報。

首先，您應該將 `queue:monitor` 指令排程為 [每分鐘執行一次](/docs/{{version}}/scheduling)。該指令接收您想要監控的佇列名稱以及您設定的工作數量閾值：

```shell
php artisan queue:monitor redis:default,redis:deployments --max=100
```

僅僅排程此指令不足以觸發通知來警示您佇列負荷過重的狀態。當指令遇到工作數量超過閾值的佇列時，會派發一個 `Illuminate\Queue\Events\QueueBusy` 事件。您可以在應用程式的 `AppServiceProvider` 中監聽此事件，以便向您或您的開發團隊發送通知：

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

在測試派發工作的程式碼時，您可能希望指示 Laravel 不要實際執行工作本身，因為工作的程式碼可以獨立於派發它的程式碼直接進行測試。當然，要測試工作本身，您可以在測試中實例化一個工作實例並直接呼叫 `handle` 方法。

您可以使用 `Queue` Facade 的 `fake` 方法來防止佇列工作被實際推送到佇列中。呼叫 `Queue` Facade 的 `fake` 方法後，您可以斷言應用程式是否嘗試將工作推送到佇列中：

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

您可以將閉包傳遞給 `assertPushed`、`assertNotPushed`、`assertClosurePushed` 或 `assertClosureNotPushed` 方法，以斷言推送到佇列的工作是否通過特定的「真值測試」。如果至少有一個工作通過了該真值測試，則斷言成功：

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

如果您只需要偽造特定的工作，而允許其他工作正常執行，您可以將需要偽造的工作類別名稱傳遞給 `fake` 方法：

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

您可以使用 `except` 方法來偽造除指定工作集以外的所有工作：

```php
Queue::fake()->except([
    ShipOrder::class,
]);
```


<a name="testing-job-chains"></a>
### 測試工作鏈

要測試工作鏈，您需要利用 `Bus` Facade 的偽造功能。`Bus` Facade 的 `assertChained` 方法可用於斷言是否派發了[工作鏈](/docs/{{version}}/queues#job-chaining)。`assertChained` 方法接收一個包含鏈接工作的陣列作為第一個引數：

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

如上面的範例所示，鏈接工作的陣列可以是工作類別名稱的陣列。此外，您也可以提供實際工作實例的陣列。這樣做時，Laravel 會確保工作實例與應用程式派發的鏈接工作屬於相同的類別且具有相同的屬性值：

```php
Bus::assertChained([
    new ShipOrder,
    new RecordShipment,
    new UpdateInventory,
]);
```

您可以使用 `assertDispatchedWithoutChain` 方法來斷言一個工作是在沒有工作鏈的情況下被推送的：

```php
Bus::assertDispatchedWithoutChain(ShipOrder::class);
```


<a name="testing-chain-modifications"></a>
#### 測試鏈修改

如果一個鏈接工作在現有鏈中[前置或後置工作](#adding-jobs-to-the-chain)，您可以使用工作的 `assertHasChain` 方法來斷言該工作是否具有預期的剩餘工作鏈：

```php
$job = new ProcessPodcast;

$job->handle();

$job->assertHasChain([
    new TranscribePodcast,
    new OptimizePodcast,
    new ReleasePodcast,
]);
```

`assertDoesntHaveChain` 方法可用於斷言工作的剩餘鏈為空：

```php
$job->assertDoesntHaveChain();
```


<a name="testing-chained-batches"></a>
#### 測試鏈接批處理

如果您的工作鏈[包含一個工作批處理](#chains-and-batches)，您可以在鏈斷言中插入 `Bus::chainedBatch` 定義，以斷言鏈接的批處理符合您的預期：

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
### 測試工作批處理

`Bus` 門面 (Facade) 的 `assertBatched` 方法可用於斷言 [工作批處理](/docs/{{version}}/queues#job-batching) 已被派發。傳遞給 `assertBatched` 方法的閉包會接收一個 `Illuminate\Bus\PendingBatch` 實例，可用於檢查批處理中的工作：

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

在待定批處理 (pending batch) 上可以使用 `hasJobs` 方法來驗證批處理是否包含預期的工作。該方法接受工作實例、類別名稱或閉包的陣列：

```php
Bus::assertBatched(function (PendingBatch $batch) {
    return $batch->hasJobs([
        new ProcessCsvRow(row: 1),
        new ProcessCsvRow(row: 2),
        new ProcessCsvRow(row: 3),
    ]);
});
```

使用閉包時，閉包將接收工作實例。預期的工作型別將從閉包的型別提示 (type hint) 中推斷：

```php
Bus::assertBatched(function (PendingBatch $batch) {
    return $batch->hasJobs([
        fn (ProcessCsvRow $job) => $job->row === 1,
        fn (ProcessCsvRow $job) => $job->row === 2,
        fn (ProcessCsvRow $job) => $job->row === 3,
    ]);
});
```

您可以使用 `assertBatchCount` 方法來斷言已派發指定數量的批處理：

```php
Bus::assertBatchCount(3);
```

您可以使用 `assertNothingBatched` 來斷言沒有任何批處理被派發：

```php
Bus::assertNothingBatched();
```


<a name="testing-job-batch-interaction"></a>
#### 測試工作 / 批處理互動

此外，您偶爾可能需要測試單個工作與其底層批處理的互動。例如，您可能需要測試工作是否取消了其批處理的後續處理。要實現此目的，您需要透過 `withFakeBatch` 方法將偽造的批處理分配給工作。`withFakeBatch` 方法會返回一個包含工作實例與偽造批處理的元組 (tuple)：

```php
[$job, $batch] = (new ShipOrder)->withFakeBatch();

$job->handle();

$this->assertTrue($batch->cancelled());
$this->assertEmpty($batch->added);
```


<a name="testing-job-queue-interactions"></a>
### 測試工作 / 佇列互動

有時候，您可能需要測試佇列工作是否 [將自身重新釋放回佇列](#manually-releasing-a-job)，或者測試工作是否刪除了自身。您可以透過實例化工作並調用 `withFakeQueueInteractions` 方法來測試這些佇列互動。

一旦工作的佇列互動被偽造後，您就可以調用工作的 `handle` 方法。在調用工作後，可以使用各種斷言方法來驗證工作的佇列互動：

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

使用 `Queue` [Facade](/docs/{{version}}/facades) 的 `before` 與 `after` 方法，您可以指定在佇列工作被處理之前或之後執行的回呼 (callbacks)。這些回呼是執行額外記錄 (logging) 或增加儀表板統計數據的絕佳機會。通常，您應該在 [服務提供者(Service Providers)](/docs/{{version}}/providers) 的 `boot` 方法中呼叫這些方法。例如，我們可以使用 Laravel 內建的 `AppServiceProvider`：

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

使用 `Queue` [Facade](/docs/{{version}}/facades) 的 `looping` 方法，您可以指定在工作者嘗試從佇列中擷取工作之前執行的回呼。例如，您可以註冊一個閉包來回滾 (rollback) 任何由先前失敗的工作而未關閉的交易：

```php
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Queue;

Queue::looping(function () {
    while (DB::transactionLevel() > 0) {
        DB::rollBack();
    }
});
```