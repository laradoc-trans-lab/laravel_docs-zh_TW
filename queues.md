# 佇列

- [簡介](#introduction)
    - [連線 vs. 佇列](#connections-vs-queues)
    - [驅動說明與前置準備](#driver-prerequisites)
- [建立 Job](#creating-jobs)
    - [產生 Job 類別](#generating-job-classes)
    - [類別結構](#class-structure)
    - [唯一 Job](#unique-jobs)
    - [Debounced Job](#debounced-jobs)
    - [加密 Job](#encrypted-jobs)
- [Job 中介層](#job-middleware)
    - [速率限制](#rate-limiting)
    - [防止 Job 重疊](#preventing-job-overlaps)
    - [限制 Exception 頻率](#throttling-exceptions)
    - [釋放 Job](#releasing-jobs)
    - [略過 Job](#skipping-jobs)
- [分派 Job](#dispatching-jobs)
    - [延遲分派](#delayed-dispatching)
    - [同步分派](#synchronous-dispatching)
    - [大量分派](#bulk-dispatching)
    - [在分派前準備 Job](#preparing-jobs-before-dispatch)
    - [Job 與資料庫交易](#jobs-and-database-transactions)
    - [Job 鏈結](#job-chaining)
    - [自訂佇列與連線](#customizing-the-queue-and-connection)
    - [指定最大 Job 嘗試次數 / 超時時間](#max-job-attempts-and-timeout)
    - [SQS FIFO 與公平佇列](#sqs-fifo-and-fair-queues)
    - [佇列故障移轉](#queue-failover)
    - [錯誤處理](#error-handling)
- [Job 批次處理](#job-batching)
    - [定義可批次處理的 Job](#defining-batchable-jobs)
    - [分派批次](#dispatching-batches)
    - [鏈結與批次](#chains-and-batches)
    - [新增 Job 至批次](#adding-jobs-to-batches)
    - [檢視批次](#inspecting-batches)
    - [取消批次](#cancelling-batches)
    - [批次失敗處理](#batch-failures)
    - [修剪批次記錄](#pruning-batches)
    - [將批次儲存於 DynamoDB](#storing-batches-in-dynamodb)
- [佇列化 Closure](#queueing-closures)
- [執行佇列 Worker](#running-the-queue-worker)
    - [`queue:work` 指令](#the-queue-work-command)
    - [佇列優先順序](#queue-priorities)
    - [佇列 Worker 與部署](#queue-workers-and-deployment)
    - [回應 Worker 信號](#reacting-to-worker-signals)
    - [Job 過期與超時](#job-expirations-and-timeouts)
    - [暫停與恢復佇列 Worker](#pausing-and-resuming-queue-workers)
- [Supervisor 設定](#supervisor-configuration)
- [處理失敗的 Job](#dealing-with-failed-jobs)
    - [在 Job 失敗後清理](#cleaning-up-after-failed-jobs)
    - [重試失敗的 Job](#retrying-failed-jobs)
    - [忽略遺失的模型](#ignoring-missing-models)
    - [修剪失敗的 Job](#pruning-failed-jobs)
    - [將失敗的 Job 儲存於 DynamoDB](#storing-failed-jobs-in-dynamodb)
    - [停用失敗 Job 的儲存](#disabling-failed-job-storage)
    - [失敗 Job 事件](#failed-job-events)
- [清除佇列中的 Job](#clearing-jobs-from-queues)
- [監控您的佇列](#monitoring-your-queues)
- [測試](#testing)
    - [模擬部分 Job](#faking-a-subset-of-jobs)
    - [測試 Job 鏈結](#testing-job-chains)
    - [測試 Job 批次](#testing-job-batches)
    - [測試 Job / 佇列互動](#testing-job-queue-interactions)
- [Job 事件](#job-events)

<a name="introduction"></a>
## 簡介

在建置 Web 應用程式時，您可能會遇到某些任務（例如解析與儲存上傳的 CSV 檔案）需要耗費較長時間，若在一般的 Web 請求過程中執行會導致回應過慢。幸好，Laravel 讓您能輕鬆建立可於背景處理的佇列任務 (Queued Jobs)。透過將耗時的任務移至佇列中，您的應用程式便能以極快的速度回應 Web 請求，為您的客戶提供更好的使用者體驗。

Laravel 佇列為多種不同的佇列後端服務（如 [Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 甚至關聯式資料庫）提供了統一的佇列 API。

Laravel 的佇列設定選項儲存於應用程式的 `config/queue.php` 設定檔中。在此檔案中，您可以找到框架內建之各個佇列驅動的連線設定，包含資料庫、[Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 及 [Beanstalkd](https://beanstalkd.github.io/) 驅動，以及用於立即執行 Job 的同步 (synchronous) 驅動（供開發或測試時使用）。此外，也包含一個會直接丟棄佇列 Job 的 `null` 佇列驅動。

> [!NOTE]
> Laravel Horizon 為您基於 Redis 的佇列提供了美觀的儀表板與設定系統。請參考完整的 [Horizon 文件](/docs/{{version}}/horizon) 以取得更多資訊。


<a name="connections-vs-queues"></a>
### 連線 vs. 佇列

在開始使用 Laravel 佇列之前，瞭解「連線 (connections)」與「佇列 (queues)」之間的區別非常重要。在您的 `config/queue.php` 設定檔中，有一個 `connections` 設定陣列。此選項定義了與後端佇列服務（如 Amazon SQS、Beanstalk 或 Redis）的連線。然而，任何給定的佇列連線都可以擁有多個「佇列」，這些佇列可以被視為不同堆疊或堆放佇列 Job 的地方。

請注意，`queue` 設定檔中的每個連線設定範例都包含一個 `queue` 屬性。當 Job 被發送到給定的連線時，這是預設分派到的佇列。換句話說，如果您在分派 Job 時未明確定義應分派到哪個佇列，該 Job 將會被放置在連線設定中 `queue` 屬性所定義的佇列上：

```php
use App\Jobs\ProcessPodcast;

// This job is sent to the default connection's default queue...
ProcessPodcast::dispatch();

// This job is sent to the default connection's "emails" queue...
ProcessPodcast::dispatch()->onQueue('emails');
```

某些應用程式可能永遠不需要將 Job 推送到多個佇列，而傾向於只使用一個簡單的佇列。然而，對於希望對 Job 處理方式進行優先順序劃分或分流的應用程式來說，將 Job 推送到多個佇列會特別有用，因為 Laravel 佇列 Worker 允許您依據優先順序指定要處理哪些佇列。例如，如果您將 Job 推送到 `high` 佇列，您可以執行一個 Worker 並給予它們更高的處理優先權：

```shell
php artisan queue:work --queue=high,default
```


<a name="driver-prerequisites"></a>
### 驅動說明與前置準備


<a name="database"></a>
#### 資料庫

為了使用 `database` 佇列驅動，您需要一個資料庫表單來存放 Job。通常這已經包含在 Laravel 預設的 `0001_01_01_000002_create_jobs_table.php` [資料庫遷移](/docs/{{version}}/migrations)中；然而，如果您的應用程式不包含此遷移檔，您可以使用 `make:queue-table` Artisan 指令來建立它：

```shell
php artisan make:queue-table

php artisan migrate
```


<a name="redis"></a>
#### Redis

為了使用 `redis` 佇列驅動，您應該在 `config/database.php` 設定檔中設定 Redis 資料庫連線。

> [!WARNING]
> `redis` 佇列驅動不支援 `serializer` 與 `compression` 這兩個 Redis 選項。


<a name="redis-cluster"></a>
##### Redis 叢集

如果您的 Redis 佇列連線使用的是 [Redis 叢集 (Redis Cluster)](https://redis.io/docs/latest/operate/rs/databases/durability-ha/clustering)，您的佇列名稱必須包含 [Key Hash Tag](https://redis.io/docs/latest/develop/using-commands/keyspace/#hashtags)。這是為了確保給定佇列的所有 Redis Key 都會被放置在相同的 Hash Slot 中：

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

使用 Redis 佇列時，您可以使用 `block_for` 設定選項來指定驅動在重新迴圈 Worker 流程並再次輪詢 (Re-poll) Redis 資料庫之前，應該等待 Job 變為可用狀態的時間。

根據佇列負載調整此數值，會比不斷輪詢 Redis 資料庫以取得新 Job 更有效率。例如，您可以將值設定為 `5`，表示驅動在等待 Job 可用時應阻塞 5 秒：

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
> 將 `block_for` 設定為 `0` 會導致佇列 Worker 無限期阻塞，直到有可用 Job 為止。這也會導致像是 `SIGTERM` 這類的訊號在下一個 Job 處理完成之前無法被處理。


<a name="sqs-overflow-storage"></a>
#### SQS 溢出儲存空間

Amazon SQS 限制了佇列訊息 Payload 的最大容量。如果您需要分派 Payload 可能會超出此限制的 Job，您可以將 Laravel 設定為將過大的 SQS Payload 儲存在快取存放區中，並改為透過 SQS 傳送指標。若要啟用此功能，請在您的 SQS 佇列連線設定中新增 `overflow` 陣列：

```php
'sqs' => [
    'driver' => 'sqs',
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'prefix' => env('SQS_PREFIX', 'https://sqs.us-east-1.amazonaws.com/your-account-id'),
    'queue' => env('SQS_QUEUE', 'default'),
    'suffix' => env('SQS_SUFFIX'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    'after_commit' => false,
    'overflow' => [
        'enabled' => env('SQS_OVERFLOW_ENABLED', false),
        'store' => env('SQS_OVERFLOW_STORE'),
        'always' => false,
        'delete_after_processing' => true,
        'flush_on_clear' => env('SQS_OVERFLOW_FLUSH_ON_CLEAR', false),
    ],
],
```

當啟用溢出儲存空間時，Laravel 將會把至少 1 MB 的 Payload 儲存在設定的快取存放區中。若 `always` 選項為 `true`，無論大小為何，每個 SQS Payload 都將儲存在快取存放區中。由於佇列 Job 在被處理時需要從快取存放區中檢索其 Payload，因此您應該選擇一個能夠保留 Payload 直到 Worker 處理完畢的存放區。預設情況下，已儲存的 Payload 會在 Job 成功處理並從 SQS 刪除後被刪除。

若 `flush_on_clear` 選項為 `true`，當執行 `queue:clear` 指令清除 SQS 佇列時，設定的溢出快取存放區將會被清除 (Flush)。由於清除快取存放區可能會移除該存放區中的所有項目，因此啟用此選項時，您應該將 SQS 溢出儲存空間設定為使用專用的快取存放區。


<a name="other-driver-prerequisites"></a>
#### 其他驅動前置準備

列出的佇列驅動需要以下依賴套件。這些依賴套件可透過 Composer 套件包管理器進行安裝：

<div class="content-list" markdown="1">

- Amazon SQS: `aws/aws-sdk-php ~3.0`
- Beanstalkd: `pda/pheanstalk ~5.0`
- Redis: `predis/predis ~3.0` 或 phpredis PHP 擴充套件
- [MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/queues/): `mongodb/laravel-mongodb`

</div>

<a name="creating-jobs"></a>
## 建立 Job


<a name="generating-job-classes"></a>
### 產生 Job 類別

預設情況下，應用程式中所有可佇列化的 Job 都儲存在 `app/Jobs` 目錄中。若 `app/Jobs` 目錄不存在，當您執行 `make:job` Artisan 指令時會自動建立該目錄：

```shell
php artisan make:job ProcessPodcast
```

產生的類別會實作 `Illuminate\Contracts\Queue\ShouldQueue` 介面，告訴 Laravel 該 Job 應該被推送到佇列中非同步執行。

> [!NOTE]
> Job 的存根 (Stub) 可以透過[存根發布 (Stub Publishing)](/docs/{{version}}/artisan#stub-customization)進行自訂。


<a name="class-structure"></a>
### 類別結構

Job 類別非常簡單，通常只包含一個在佇列處理該 Job 時被呼叫的 `handle` 方法。首先，讓我們看一個 Job 類別的範例。在這個範例中，假設我們管理一個播客 (Podcast) 發布服務，需要在上傳的播客檔案發布前對其進行處理：

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

在這個範例中，請注意我們可以將 [Eloquent Model](/docs/{{version}}/eloquent) 直接傳入佇列 Job 的建構子。由於該 Job 使用了 `Queueable` trait，Eloquent Model 以及其已載入的關聯會在 Job 處理時優雅地進行序列化與反序列化。

若您的佇列 Job 在其建構子中接受一個 Eloquent Model，則只有該 Model 的識別碼會被序列化到佇列中。當 Job 實際被處理時，佇列系統會自動從資料庫重新取得完整的 Model 實例及其已載入的關聯。這種 Model 序列化方式可以讓發送到佇列驅動的 Job 負載 (Payload) 小得多。


<a name="handle-method-dependency-injection"></a>
#### `handle` 方法的依賴注入

當 Job 被佇列處理時，會呼叫 `handle` 方法。請注意，我們可以在 Job 的 `handle` 方法上對依賴進行型態提示 (Type-hint)。Laravel [服務容器](/docs/{{version}}/container) 會自動注入這些依賴。

如果您想完全控制容器如何將依賴注入到 `handle` 方法中，可以使用容器的 `bindMethod` 方法。`bindMethod` 方法接收一個回呼函式，該函式會接收 Job 與容器。在回呼函式中，您可以依需求自由呼叫 `handle` 方法。通常，您應該在 `App\Providers\AppServiceProvider` [服務提供者(Service Providers)](/docs/{{version}}/providers) 的 `boot` 方法中呼叫此方法：

```php
use App\Jobs\ProcessPodcast;
use App\Services\AudioProcessor;
use Illuminate\Contracts\Foundation\Application;

$this->app->bindMethod([ProcessPodcast::class, 'handle'], function (ProcessPodcast $job, Application $app) {
    return $job->handle($app->make(AudioProcessor::class));
});
```

> [!WARNING]
> 二進位資料（例如原始圖片內容）在傳遞給佇列 Job 之前，應該先通過 `base64_encode` 函式處理。否則，Job 在放入佇列時可能無法正確序列化為 JSON。


<a name="handling-relationships"></a>
#### 佇列化關聯

因為所有已載入的 Eloquent Model 關聯也會在 Job 被放入佇列時被序列化，所以序列化後的 Job 字串有時會變得非常龐大。此外，當 Job 被反序列化並從資料庫重新取得 Model 關聯時，這些關聯將被完整取得。在 Job 進入佇列過程前、Model 被序列化之前所套用的任何前置關聯約束條件，在 Job 反序列化時都不會被重新套用。因此，若您希望處理指定關聯的子集，您應該在佇列 Job 內部重新為該關聯加上約束條件。

或者，若要防止關聯被序列化，您可以在設定屬性值時呼叫 Model 上的 `withoutRelations` 方法。此方法將回傳不包含已載入關聯的 Model 實例：

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

如果您只需要移除特定的關聯並保留其餘關聯，可以使用 `withoutRelation` 方法：

```php
$this->podcast = $podcast->withoutRelation('comments');
```

如果您使用的是 [PHP 建構子屬性提升 (Constructor Property Promotion)](https://www.php.net/manual/en/language.oop5.decon.php#language.oop5.decon.constructor.promotion)，並希望指定 Eloquent Model 不應將其關聯進行序列化，可以使用 `WithoutRelations` 屬性：

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

為了方便起見，如果您希望將所有 Model 進行序列化而不帶任何關聯，可以將 `WithoutRelations` 屬性套用到整個類別，而不是將屬性分別套用到每個 Model 上：

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

如果 Job 接收的是 Eloquent Model 的集合 (Collection) 或陣列，而非單一 Model，則當 Job 反序列化並執行時，該集合內的 Model 將不會還原其關聯。這是為了防止處理大量 Model 的 Job 消耗過多資源。

<a name="unique-jobs"></a>
### 唯一 Job

> [!WARNING]
> 唯一 Job 需要支援[鎖定](/docs/{{version}}/cache#atomic-locks)的快取驅動。目前 `memcached`、`redis`、`dynamodb`、`database`、`file` 與 `array` 快取驅動支援原子鎖定。

> [!WARNING]
> 唯一 Job 限制並不適用於批次內的 Job。

有時候，您可能希望確保在任何時間點，佇列中都只有一個特定 Job 的實例。您可以在 Job 類別上實作 `ShouldBeUnique` 介面來達到此目的。此介面不需要您在類別中定義任何額外的方法：

```php
<?php

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Queue\ShouldBeUnique;

class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    // ...
}
```

在上面的範例中，`UpdateSearchIndex` Job 是唯一的。因此，如果該 Job 的另一個實例已經在佇列中且尚未完成處理，則不會分派該 Job。

在某些情況下，您可能希望定義使 Job 唯一的特定「鍵值 (Key)」，或者您可能希望指定一個超時時間，超過該時間後 Job 將不再保持唯一。若要做到這一點，您可以使用 `UniqueFor` 屬性並在 Job 類別上定義 `uniqueId` 方法：

```php
<?php

namespace App\Jobs;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Queue\ShouldBeUnique;
use Illuminate\Queue\Attributes\UniqueFor;

#[UniqueFor(3600)]
class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    /**
     * The product instance.
     *
     * @var \App\Models\Product
     */
    public $product;

    /**
     * Get the unique ID for the job.
     */
    public function uniqueId(): string
    {
        return $this->product->id;
    }
}
```
在上面的範例中，`UpdateSearchIndex` Job 會根據商品 ID 保持唯一。因此，在現有的 Job 完成處理之前，任何具有相同商品 ID 的新 Job 分派都會被忽略。此外，如果現有的 Job 在一小時內未處理完成，則唯一鎖定將被釋放，並且可以將具有相同唯一鍵值的另一個 Job 分派到佇列中。

> [!WARNING]
> 如果您的應用程式從多個 Web 伺服器或容器分派 Job，您應該確保所有伺服器都與同一個中央快取伺服器進行通訊，以便 Laravel 可以精確地判斷 Job 是否唯一。


<a name="keeping-jobs-unique-until-processing-begins"></a>
#### 保持 Job 唯一直到開始處理

預設情況下，唯一 Job 會在 Job 完成處理或所有重試嘗試均失敗後「解鎖」。然而，在某些情況下，您可能希望 Job 在開始處理之前就立即解鎖。若要實現此目的，您的 Job 應該實作 `ShouldBeUniqueUntilProcessing` 契約(Contracts)，而非 `ShouldBeUnique` 契約：

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

在幕後，當分派 `ShouldBeUnique` Job 時，Laravel 會嘗試取得帶有 `uniqueId` 鍵值的[鎖定](/docs/{{version}}/cache#atomic-locks)。如果該鎖定已被佔用，則不會分派該 Job。當 Job 完成處理或所有重試嘗試均失敗時，此鎖定會被釋放。預設情況下，Laravel 會使用預設的快取驅動來取得此鎖定。但是，如果您希望使用另一個驅動來取得鎖定，可以定義一個 `uniqueVia` 方法來傳回應使用的快取驅動：

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
> 如果您只需要限制 Job 的同時處理數量，請改用 [WithoutOverlapping](/docs/{{version}}/queues#preventing-job-overlaps) Job 中介層。


<a name="debounced-jobs"></a>
### Debounced Job

有時候，您可能希望確保在短時間內多次分派相同的 Job 時，只有最後一次分派才會真正執行。您可以透過在 Job 中新增 `DebounceFor` 屬性來做到這一點：

```php
<?php

namespace App\Jobs;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;
use Illuminate\Queue\Attributes\DebounceFor;

#[DebounceFor(30)]
class UpdateSearchIndex implements ShouldQueue
{
    use Queueable;

    /**
     * Create a new job instance.
     */
    public function __construct(public int $productId)
    {
    }

    /**
     * Get the debounce ID for the job.
     */
    public function debounceId(): string
    {
        return (string) $this->productId;
    }
}
```

在上面的範例中，如果在 `30` 秒內對同一個商品重複分派 `UpdateSearchIndex`，將會防抖動 (Debounce) 該 Job，從而僅執行最新的分派。

如果您想限制頻繁重新分派的 Job 可以被延遲的最長時間，您可以為 `DebounceFor` 屬性提供 `maxWait` 引數：

```php
#[DebounceFor(30, maxWait: 120)]
class UpdateSearchIndex implements ShouldQueue
{
    use Queueable;

    // ...
}
```

您可以透過在 Job 上定義 `debounceVia` 方法來自訂用於 Debounce 追蹤的快取儲存區：

```php
use Illuminate\Contracts\Cache\Repository;
use Illuminate\Support\Facades\Cache;

public function debounceVia(): Repository
{
    return Cache::driver('redis');
}
```

如果 Debounced Job 被更新的分派所取代，Laravel 將會分派 `Illuminate\Queue\Events\JobDebounced` 事件並從佇列中移除被取代的 Job。

> [!WARNING]
> Debounced Job 與唯一 Job 是互斥的。使用 `DebounceFor` 屬性的 Job 不應實作 `ShouldBeUnique`。

> [!WARNING]
> 如果您的應用程式從多個 Web 伺服器或容器分派 Debounced Job，您應該確保所有伺服器都與同一個中央快取伺服器進行通訊。


<a name="encrypted-jobs"></a>
### 加密 Job

Laravel 允許您透過[加密](/docs/{{version}}/encryption)來確保 Job 資料的隱私與完整性。要開始使用，只需將 `ShouldBeEncrypted` 介面新增至 Job 類別即可。一旦將此介面新增至類別，Laravel 將會在將 Job 推送到佇列之前自動對其進行加密：

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

Job 中介層 (Job middleware) 允許您在佇列化的 Job 執行前後包覆自訂邏輯，從而減少 Job 本身的重複樣板程式碼。例如，請參考下方利用 Laravel 的 Redis 速率限制功能、讓每五秒鐘僅允許處理一個 Job 的 `handle` 方法：

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

雖然這段程式碼是有效的，但 `handle` 方法的實作變得相當雜亂，因為其中充斥著 Redis 速率限制的邏輯。此外，對於任何其他我們想要進行速率限制的 Job，這段速率限制邏輯都必須重複撰寫。與其在 handle 方法中進行速率限制，我們不如定義一個專門處理速率限制的 Job 中介層：

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

如您所見，就像[路由中介層](/docs/{{version}}/middleware)一樣，Job 中介層會接收正在處理的 Job 以及一個應該被呼叫以繼續處理 Job 的回呼 (Callback)。

您可以使用 `make:job-middleware` Artisan 指令來產生一個新的 Job 中介層類別。建立 Job 中介層之後，可以透過從 Job 的 `middleware` 方法回傳該中介層來將其附加至 Job 上。使用 `make:job` Artisan 指令所建立的 Job 底稿預設並不包含此方法，因此您需要手動將其新增至 Job 類別中：

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
> Job 中介層也可以被指定給[可佇列化的事件監聽器](/docs/{{version}}/events#queued-event-listeners)、[Mailable](/docs/{{version}}/mail#queueing-mail) 與[通知](/docs/{{version}}/notifications#queueing-notifications)。


<a name="rate-limiting"></a>
### 速率限制

雖然我們剛剛示範了如何撰寫自己的速率限制 Job 中介層，但 Laravel 實際上已經內建了一個速率限制中介層，您可以用它來限制 Job 的執行速率。就像[路由速率限制器](/docs/{{version}}/routing#defining-rate-limiters)一樣，Job 速率限制器也是使用 `RateLimiter` Facade 的 `for` 方法來定義的。

例如，您可能希望允許一般使用者每小時備份一次資料，而對尊榮客戶則不設此限。若要達到這個目的，您可以在 `AppServiceProvider` 的 `boot` 方法中定義一個 `RateLimiter`：

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

在上述範例中，我們定義了一個每小時的速率限制；然而，您可以使用 `perMinute` 方法輕鬆地定義以分鐘為單位的速率限制。此外，您可以將任何想要的值傳遞給速率限制的 `by` 方法；不過，這個值最常用於按客戶分段限制速率：

```php
return Limit::perMinute(50)->by($job->user->id);
```

定義好速率限制後，您可以使用 `Illuminate\Queue\Middleware\RateLimited` 中介層將該速率限制器附加到您的 Job。每當 Job 超過速率限制時，此中介層會根據速率限制的時間長度，以適當的延遲時間將 Job 釋放回佇列：

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

將受到速率限制的 Job 釋放回佇列仍然會增加該 Job 的總 `attempts` 次數。您可能需要相應地調整 Job 類別上的 `Tries` 與 `MaxExceptions` 屬性。或者，您也可以使用 [retryUntil 方法](#time-based-attempts)來定義不再嘗試該 Job 之前的這段時間。

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

如果您不希望 Job 在受到速率限制時重新嘗試，可以使用 `dontRelease` 方法：

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

如果您使用的是 Redis，您可以使用 `Illuminate\Queue\Middleware\RateLimitedWithRedis` 中介層，該中介層針對 Redis 進行了微調，且比基本速率限制中介層更有效率：

```php
use Illuminate\Queue\Middleware\RateLimitedWithRedis;

public function middleware(): array
{
    return [new RateLimitedWithRedis('backups')];
}
```

`connection` 方法可用於指定中介層應該使用哪個 Redis 連線：

```php
return [(new RateLimitedWithRedis('backups'))->connection('limiter')];
```

<a name="preventing-job-overlaps"></a>
### 防止 Job 重疊

Laravel 包含一個 `Illuminate\Queue\Middleware\WithoutOverlapping` 中介層，允許您根據任意鍵值來防止 Job 重疊執行。當佇列中的 Job 正在修改同時間只能由一個 Job 修改的資源時，這會非常有幫助。

舉例來說，假設您有一個更新使用者信用分數的佇列 Job，且您想防止相同使用者 ID 的信用分數更新 Job 重疊執行。若要達成此目的，您可以從 Job 的 `middleware` 方法中回傳 `WithoutOverlapping` 中介層：

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

將重疊的 Job 釋放回佇列仍會增加該 Job 的總嘗試次數。您可能需要相應地調整 Job 類別上的 `Tries` 和 `MaxExceptions` 屬性。例如，保持預設的 `Tries` 為 1，將會防止任何重疊的 Job 在稍後重試。

任何相同類型的重疊 Job 都會被釋放回佇列。您也可以指定釋放後的 Job 在再次嘗試前必須經過的秒數：

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

若您希望立即刪除任何重疊的 Job 以便不再重試，您可以使用 `dontRelease` 方法：

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

`WithoutOverlapping` 中介層底層是由 Laravel 的原子鎖 (Atomic Lock) 功能支援。有時，您的 Job 可能會意外失敗或超時，導致鎖定未被釋放。因此，您可以使用 `expireAfter` 方法明確定義鎖定過期時間。例如，以下範例將指示 Laravel 在 Job 開始處理 3 分鐘後釋放 `WithoutOverlapping` 鎖定：

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
> `WithoutOverlapping` 中介層需要支援[鎖定](/docs/{{version}}/cache#atomic-locks)的快取驅動程式。目前 `memcached`、`redis`、`dynamodb`、`database`、`file` 及 `array` 快取驅動程式皆支援原子鎖。


<a name="sharing-lock-keys"></a>
#### 跨 Job 類別共用鎖定金鑰

預設情況下，`WithoutOverlapping` 中介層只會防止相同類別的 Job 重疊。因此，雖然兩個不同的 Job 類別可能使用相同的鎖定金鑰，但它們不會被防止重疊。不過，您可以透過使用 `shared` 方法來指示 Laravel 將金鑰套用至跨 Job 類別：

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
### 限制 Exception 頻率

Laravel 包含一個 `Illuminate\Queue\Middleware\ThrottlesExceptions` 中介層，允許您限制 Exception 的觸發頻率。一旦 Job 拋出指定的 Exception 次數後，後續所有執行該 Job 的嘗試都將被延遲，直到指定的時間間隔過去為止。此中介層對於與不穩定第三方服務互動的 Job 特別有用。

例如，假設有一個佇列 Job 與第三方 API 進行互動，而該 API 開始拋出 Exception。若要限制 Exception 頻率，您可以從 Job 的 `middleware` 方法回傳 `ThrottlesExceptions` 中介層。通常，此中介層應與實作[基於時間的嘗試](#time-based-attempts)的 Job 搭配使用：

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

該中介層接受的第一個建構子引數是 Job 被限制頻率前可拋出的 Exception 次數，而第二個建構子引數則是 Job 一旦被限制頻率後，再次嘗試執行 Job 前必須經過的秒數。在上述程式碼範例中，如果 Job 連續拋出 10 次 Exception，我們將等待 5 分鐘後才再次嘗試該 Job，並受限於 30 分鐘的時間限制。

當 Job 拋出 Exception 但尚未達到 Exception 門檻時，通常會立即重試該 Job。然而，您可以在將中介層附加到 Job 時呼叫 `backoff` 方法，來指定該 Job 應該延遲的分鐘數：

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

`backoff` 方法也接受一個接收拋出 Exception 的 Closure，從而能動態決定延遲時間：

```php
use App\Exceptions\RateLimitedException;
use Illuminate\Queue\Middleware\ThrottlesExceptions;
use Throwable;

/**
 * Get the middleware the job should pass through.
 *
 * @return array<int, object>
 */
public function middleware(): array
{
    return [(new ThrottlesExceptions(10, 5 * 60))->backoff(
        fn (Throwable $throwable) => $throwable instanceof RateLimitedException
            ? $throwable->retryAfterMinutes()
            : 5
    )];
}
```

在內部，此中介層使用 Laravel 的快取系統來實作速率限制，並將 Job 的類別名稱用作快取的 "key"。您可以在將中介層附加到 Job 時，透過呼叫 `by` 方法來覆寫此 key。如果您有多個與同一個第三方服務互動的 Job，且希望它們共享同一個限制「區塊 (Bucket)」以確保遵循單一共享限制時，這會非常實用：

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

預設情況下，此中介層會限制所有 Exception 的頻率。您可以透過在附加中介層到 Job 時呼叫 `when` 方法來修改此行為。如此一來，只有傳遞給 `when` 方法的 Closure 回傳 `true` 時，才會限制 Exception 頻率：

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

與將 Job 釋放回佇列或拋出 Exception 的 `when` 方法不同，`deleteWhen` 方法允許您在發生指定的 Exception 時直接刪除該 Job：

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

如果您希望能將被限制頻率的 Exception 回報給應用程式的 Exception 處理常式 (Exception Handler)，可以在將中介層附加到 Job 時呼叫 `report` 方法。另外，您也可以提供 Closure 給 `report` 方法，這樣該 Exception 只會在給定的 Closure 回傳 `true` 時才被回報：

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
#### 使用 Redis 限制 Exception 頻率

如果您使用的是 Redis，可以使用 `Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis` 中介層，該中介層針對 Redis 進行了特別調校，比基礎的 Exception 頻率限制中介層效率更高：

```php
use Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis;

public function middleware(): array
{
    return [new ThrottlesExceptionsWithRedis(10, 10 * 60)];
}
```

`connection` 方法可用於指定該中介層應使用的 Redis 連線：

```php
return [(new ThrottlesExceptionsWithRedis(10, 10 * 60))->connection('limiter')];
```


<a name="releasing-jobs"></a>
### 釋放 Job

`Release` 中介層允許您在不執行 Job 的情況下將其釋放回佇列。若給定條件求值為 `true`，`Release::when` 方法將會釋放 Job；而若條件求值為 `false`，`Release::unless` 方法則會釋放 Job：

```php
use Illuminate\Queue\Middleware\Release;

/**
 * Get the middleware the job should pass through.
 */
public function middleware(): array
{
    return [
        Release::when($condition, releaseAfter: 60),
    ];
}
```

將 Job 釋放回佇列仍然會增加該 Job 的總嘗試次數。您可能需要相應地調整 Job 類別上的 `Tries` 和 `MaxExceptions` 屬性。

您也可以傳遞 `Closure` 給 `when` 和 `unless` 方法，進行更複雜的條件求值：

```php
use Illuminate\Queue\Middleware\Release;

/**
 * Get the middleware the job should pass through.
 */
public function middleware(): array
{
    return [
        Release::when(function (): bool {
            return ! $this->order->isPaid();
        }, releaseAfter: 60),
    ];
}
```


<a name="skipping-jobs"></a>
### 略過 Job

`Skip` 中介層允許您指定略過 / 刪除 Job，而無需修改 Job 的邏輯。若給定條件求值為 `true`，`Skip::when` 方法將會刪除 Job；而若條件求值為 `false`，`Skip::unless` 方法則會刪除 Job：

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

您也可以傳遞 `Closure` 給 `when` 和 `unless` 方法，進行更複雜的條件求值：

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

一旦撰寫好 Job 類別後，您就可以使用 Job 類別本身的 `dispatch` 方法來分派它。傳遞給 `dispatch` 方法的引數將會被傳入該 Job 的建構子：

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

若您想要根據條件來分派 Job，可以使用 `dispatchIf` 與 `dispatchUnless` 方法：

```php
ProcessPodcast::dispatchIf($accountActive, $podcast);

ProcessPodcast::dispatchUnless($accountSuspended, $podcast);
```

在新的 Laravel 應用程式中，`database` 連線被定義為預設佇列。您可以透過修改應用程式 `.env` 檔案中的 `QUEUE_CONNECTION` 環境變數來指定不同的預設佇列連線。


<a name="delayed-dispatching"></a>
### 延遲分派

如果您想要指定某個 Job 不應立即供佇列 Worker 處理，可以在分派 Job 時使用 `delay` 方法。例如，讓我們指定一個 Job 在分派後 10 分鐘內都不能被處理：

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

在某些情況下，Job 可能已經設定了預設的延遲時間。如果您需要繞過此延遲並立即分派 Job 進行處理，可以使用 `withoutDelay` 方法：

```php
ProcessPodcast::dispatch($podcast)->withoutDelay();
```

> [!WARNING]
> Amazon SQS 佇列服務的最大延遲時間為 15 分鐘。


<a name="synchronous-dispatching"></a>
### 同步分派

若您想要立即（同步地）分派 Job，可以使用 `dispatchSync` 方法。使用此方法時，該 Job 將不會進入佇列，而是會在當前行程(Processes)中立即執行：

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
#### 延後分派

使用延後同步分派，您可以分派一個 Job 在當前行程中處理，但會在 HTTP 回應發送給使用者之後才執行。這讓您可以同步處理「佇列化」的 Job，而不會降低使用者的應用程式體驗。若要延後同步 Job 的執行，請將 Job 分派至 `deferred` 連線：

```php
RecordDelivery::dispatch($order)->onConnection('deferred');
```

`deferred` 連線同時也是預設的[佇列故障移轉](#queue-failover)。

同樣地，`background` 連線會在 HTTP 回應發送給使用者後處理 Job；然而，該 Job 會在單獨派生的 PHP 行程中處理，讓 PHP-FPM / 應用程式 Worker 能空出來處理另一個傳入的 HTTP 請求：

```php
RecordDelivery::dispatch($order)->onConnection('background');
```


<a name="bulk-dispatching"></a>
### 大量分派

若您需要一次分派許多獨立的 Job，且不需要[批次](#job-batching)追蹤或回呼，可以使用 `Bus` Facade 的 `bulk` 方法。Laravel 將會根據其設定的佇列連線與佇列名稱對這些 Job 進行分組，並將每個分組大量推送到適當的佇列中：

```php
use App\Jobs\ProcessUser;
use Illuminate\Support\Facades\Bus;

Bus::bulk(
    $users->map(fn ($user) => new ProcessUser($user))
);
```


<a name="preparing-jobs-before-dispatch"></a>
### 在分派前準備 Job

若 Job 在推送到佇列前需要準備或檢視其狀態，該 Job 可以實作 `Illuminate\Contracts\Queue\PreparesForDispatch` 介面。Laravel 會在分派 Job 之前呼叫該 Job 的 `prepareForDispatch` 方法。如果此方法傳回 `false`，則該 Job 將不會被分派：

```php
<?php

namespace App\Jobs;

use Illuminate\Contracts\Queue\PreparesForDispatch;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;
use Illuminate\Support\Facades\Cache;

class SyncPodcasts implements PreparesForDispatch, ShouldQueue
{
    use Queueable;

    /**
     * Create a new job instance.
     */
    public function __construct(
        public array $podcastIds,
    ) {}

    /**
     * Prepare the job before dispatching.
     */
    public function prepareForDispatch(): bool
    {
        return collect($this->podcastIds)
            ->reject(fn (int $id) => Cache::has("podcast-syncing:{$id}"))
            ->isNotEmpty();
    }
}
```


<a name="jobs-and-database-transactions"></a>
### Job 與資料庫交易

雖然在資料庫交易中分派 Job 是完全沒問題的，但您應該特別注意以確保您的 Job 實際上能夠成功執行。當在交易內分派 Job 時，Worker 可能會在父交易提交（Commit）之前就處理該 Job。當這種情況發生時，您在資料庫交易期間對模型或資料庫紀錄所做的任何更新可能尚未反映在資料庫中。此外，在交易內建立的任何模型或資料庫紀錄可能還不存在於資料庫中。

幸好，Laravel 提供了多種方法來解決這個問題。首先，您可以在佇列連線的設定陣列中設定 `after_commit` 連線選項：

```php
'redis' => [
    'driver' => 'redis',
    // ...
    'after_commit' => true,
],
```

當 `after_commit` 選項為 `true` 時，您可以在資料庫交易內分派 Job；但是，Laravel 會等到所有已開啟的父資料庫交易都提交後，才會真正分派該 Job。當然，如果目前沒有開啟任何資料庫交易，Job 會被立即分派。

如果交易因為在執行過程中發生 Exception 而復原，在該交易期間分派的 Job 將會被丟棄。

> [!NOTE]
> 將 `after_commit` 設定選項設為 `true`，也會使任何佇列化的事件監聽器、Mailable、通知與廣播事件在所有已開啟的資料庫交易提交後才被分派。


<a name="specifying-commit-dispatch-behavior-inline"></a>
#### 行內指定提交分派行為

如果您沒有將 `after_commit` 佇列連線設定選項設為 `true`，您仍可指定特定的 Job 在所有開啟的資料庫交易提交後才被分派。若要達成此目的，可以在您的分派操作上鏈結 `afterCommit` 方法：

```php
use App\Jobs\ProcessPodcast;

ProcessPodcast::dispatch($podcast)->afterCommit();
```

同樣地，如果 `after_commit` 設定選項設為 `true`，您可以指定特定的 Job 立即分派，而不必等待任何開啟的資料庫交易提交：

```php
ProcessPodcast::dispatch($podcast)->beforeCommit();
```

<a name="job-chaining"></a>
### Job 鏈結

Job 鏈結允許您指定一組佇列 Job，使其在主要 Job 成功執行後依序執行。如果序列中的某個 Job 失敗，其餘的 Job 將不會被執行。若要執行佇列化 Job 鏈結，您可以使用 `Bus` Facade 提供的 `chain` 方法。Laravel 的命令 Bus 是一個底層元件，佇列 Job 分派功能就是建立在此元件之上：

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

除了將 Job 類別實例進行鏈結外，您也可以鏈結 Closure：

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
> 在 Job 中使用 `$this->delete()` 方法刪除 Job 並不會阻止鏈結中的後續 Job 被處理。只有當鏈結中的某個 Job 失敗時，鏈結才會停止執行。


<a name="chain-connection-queue"></a>
#### 鏈結連線與佇列

如果您想指定鏈結 Job 應該使用的連線與佇列，可以使用 `onConnection` 與 `onQueue` 方法。除非佇列 Job 被明確指派了不同的連線/佇列，否則這些方法會指定應該使用的佇列連線與佇列名稱：

```php
Bus::chain([
    new ProcessPodcast,
    new OptimizePodcast,
    new ReleasePodcast,
])->onConnection('redis')->onQueue('podcasts')->dispatch();
```


<a name="adding-jobs-to-the-chain"></a>
#### 新增 Job 至鏈結

有時候，您可能需要從鏈結中的某個 Job 內部，將另一個 Job 預先加到 (prepend) 或附加到 (append) 現有的 Job 鏈結中。您可以透過使用 `prependToChain` 與 `appendToChain` 方法來達到此目的：

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

在鏈結 Job 時，您可以使用 `catch` 方法來指定一個 Closure，以便在鏈結中的任何 Job 失敗時呼叫。給定的回呼函式將會接收引發 Job 失敗的 `Throwable` 實例：

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
> 由於鏈結回呼函式會被序列化並由 Laravel 佇列在稍後執行，因此您不應在鏈結回呼函式中使用 `$this` 變數。


<a name="customizing-the-queue-and-connection"></a>
### 自訂佇列與連線


<a name="dispatching-to-a-particular-queue"></a>
#### 分派至特定佇列

透過將 Job 推送到不同的佇列，您可以對佇列化 Job 進行「分類」，甚至可以優先分配多少 Worker 來處理各種佇列。請記住，這並不是將 Job 推送到佇列設定檔中所定義的不同佇列「連線」，而僅是推送到單一連線內的特定佇列。若要指定佇列，請在分派 Job 時使用 `onQueue` 方法：

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

或者，您可以在 Job 的建構子中呼叫 `onQueue` 方法來指定該 Job 的佇列：

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

如果您的應用程式與多個佇列連線進行互動，您可以使用 `onConnection` 方法來指定要將 Job 推送到哪一個連線：

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

您可以將 `onConnection` 與 `onQueue` 方法串接在一起，以指定 Job 的連線與佇列：

```php
ProcessPodcast::dispatch($podcast)
    ->onConnection('sqs')
    ->onQueue('processing');
```

或者，您可以在 Job 的建構子中呼叫 `onConnection` 方法來指定該 Job 的連線：

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


<a name="queue-routing"></a>
#### 佇列路由

您可以使用 `Queue` Facade 的 `route` 方法為特定的 Job 類別定義預設的連線與佇列。當您希望確保特定 Job 始終使用特定的佇列，而無需在 Job 本身指定連線或佇列時，這非常有用。

除了為特定的 Job 類別設定路由之外，您還可以傳遞 Interface、Trait 或父類別給 `route` 方法。當您這樣做時，任何實作該 Interface、使用該 Trait 或繼承該父類別的 Job，都將自動使用設定好的連線與佇列。

通常，您應該在服務提供者的 `boot` 方法中呼叫 `route` 方法：

```php
use App\Concerns\RequiresVideo;
use App\Jobs\ProcessPodcast;
use App\Jobs\ProcessVideo;
use Illuminate\Support\Facades\Queue;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Queue::route(ProcessPodcast::class, connection: 'redis', queue: 'podcasts');
    Queue::route(RequiresVideo::class, queue: 'video');
}
```

當指定了連線但未指定佇列時，Job 將會發送到預設佇列：

```php
Queue::route(ProcessPodcast::class, connection: 'redis');
```

您也可以透過傳遞陣列給 `route` 方法，一次為多個 Job 類別設定路由：

```php
Queue::route([
    ProcessPodcast::class => ['podcasts', 'redis'], // Queue and connection
    ProcessVideo::class => 'videos', // Queue only (uses default connection)
]);
```

> [!NOTE]
> 佇列路由依然可以在各個 Job 內部進行個別覆寫。

<a name="max-job-attempts-and-timeout"></a>
### 指定最大 Job 嘗試次數 / 超時時間


<a name="max-attempts"></a>
#### 最大嘗試次數

Job 嘗試是 Laravel 佇列系統的核心概念，並支援了許多進階功能。雖然一開始可能會讓人感到困惑，但在修改預設設定之前，瞭解它們運作的方式非常重要。

當分派一個 Job 時，它會被推送到佇列中。接著 Worker 會取用它並嘗試執行它。這就是一次 Job 嘗試。

然而，一次嘗試並不一定意味著 Job 的 `handle` 方法已經被執行。嘗試也可能透過以下幾種方式被「消耗」：

<div class="content-list" markdown="1">

- Job 在執行期間遇到未處理的 Exception。
- Job 使用 `$this->release()` 手動釋放回佇列。
- 如 `WithoutOverlapping` 或 `RateLimited` 等中介層無法取得鎖定並釋放了 Job。
- Job 超時。
- Job 的 `handle` 方法正常執行完畢且未拋出任何 Exception。

</div>

您可能不希望無限期地持續嘗試執行一個 Job。因此，Laravel 提供了多種方式來指定一個 Job 可以嘗試的次數或時間長短。

> [!NOTE]
> 預設情況下，Laravel 只會嘗試執行 Job 一次。若您的 Job 使用了如 `WithoutOverlapping` 或 `RateLimited` 等中介層，或者您手動釋放 Job，您可能需要透過 `tries` 選項來增加允許的嘗試次數。

指定 Job 可嘗試的最大次數的其中一種方法，是透過 Artisan 命令列上的 `--tries` 開關。這將適用於該 Worker 處理的所有 Job，除非正在處理的 Job 本身有指定其可嘗試的次數：

```shell
php artisan queue:work --tries=3
```

如果 Job 超過其最大嘗試次數，它將被視為「失敗」的 Job。有關處理失敗 Job 的更多資訊，請參閱[處理失敗 Job 的文件](#dealing-with-failed-jobs)。如果向 `queue:work` 指令傳遞 `--tries=0`，則該 Job 將會無限次重試。

您也可以採用更細粒度的做法，直接在 Job 類別本身使用 `Tries` 屬性來定義最大嘗試次數。如果在 Job 上指定了最大嘗試次數，它將優先於在命令列上提供的 `--tries` 數值：

```php
<?php

namespace App\Jobs;

use Illuminate\Queue\Attributes\Tries;

#[Tries(5)]
class ProcessPodcast implements ShouldQueue
{
    // ...
}
```

如果您需要動態控制特定 Job 的最大嘗試次數，您可以在 Job 上定義一個 `tries` 方法：

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

除了定義 Job 在失敗前可以嘗試多少次之外，您還可以定義一個時間點，超過該時間點後就不再嘗試執行該 Job。這允許 Job 在給定的時間範圍內進行任意次數的嘗試。若要定義 Job 不應再嘗試的時間，請在您的 Job 類別中新增 `retryUntil` 方法。該方法應回傳一個 `DateTime` 實例：

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

如果同時定義了 `retryUntil` 與 `tries`，Laravel 會優先使用 `retryUntil` 方法。

> [!NOTE]
> 您也可以在[佇列化事件監聽器](/docs/{{version}}/events#queued-event-listeners)與[佇列化通知](/docs/{{version}}/notifications#queueing-notifications)上定義 `Tries` 屬性或 `retryUntil` 方法。


<a name="max-exceptions"></a>
#### 最大 Exception 數

有時您可能希望指定 Job 可以嘗試多次，但如果重試是由指定數量的未處理 Exception 所觸發（而非直接透過 `release` 方法釋放），則該 Job 應該直接宣告失敗。若要做到這一點，您可以在 Job 類別上使用 `Tries` 與 `MaxExceptions` 屬性：

```php
<?php

namespace App\Jobs;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;
use Illuminate\Queue\Attributes\MaxExceptions;
use Illuminate\Queue\Attributes\Tries;
use Illuminate\Support\Facades\Redis;

#[Tries(25)]
#[MaxExceptions(3)]
class ProcessPodcast implements ShouldQueue
{
    use Queueable;

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

在此範例中，如果應用程式無法取得 Redis 鎖定，Job 將被釋放 10 秒，並繼續重試最多 25 次。但是，如果該 Job 拋出了 3 次未處理的 Exception，則該 Job 將會失敗。


<a name="stopping-retries-by-exception"></a>
#### 依 Exception 停止重試

有時 Exception 代表佇列中的 Job 應該立即宣告失敗，而不是被釋放以進行另一次嘗試。您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `dontRetry` Exception 方法來設定應該停止 Job 重試的 Exception 型別：

```php
use App\Exceptions\InvalidPodcastSourceException;
use Illuminate\Foundation\Configuration\Exceptions;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->dontRetry([
        InvalidPodcastSourceException::class,
    ]);
})
```

如果您需要更多控制何時應停止重試的邏輯，您可以傳遞一個 Closure 給 `dontRetryWhen` 方法。當 Closure 回傳 `true` 時，該 Job 將被標記為失敗且不會再被重試：

```php
use App\Exceptions\PodcastProcessingException;
use Illuminate\Foundation\Configuration\Exceptions;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->dontRetryWhen(function (PodcastProcessingException $e) {
        return $e->reason() === 'Subscription expired';
    });
})
```


<a name="timeout"></a>
#### 超時

通常，您大致知道佇列 Job 預計需要執行多久。因此，Laravel 允許您指定一個「超時（timeout）」值。預設的超時值為 60 秒。如果 Job 處理的時間超過超時值所指定的秒數，處理該 Job 的 Worker 將會發生錯誤並離開（exit）。通常，Worker 會由[伺服器上設定的行程管理器](#supervisor-configuration)自動重啟。

Job 可運行的最大秒數可以使用 Artisan 命令列上的 `--timeout` 開關來指定：

```shell
php artisan queue:work --timeout=30
```

如果 Job 因不斷超時而超過其最大嘗試次數，它將被標記為失敗。

您也可以在 Job 類別上使用 `Timeout` 屬性來定義該 Job 允許運行的最大秒數。如果在 Job 上指定了超時時間，它將優先於命令列上指定的任何超時時間：

```php
<?php

namespace App\Jobs;

use Illuminate\Queue\Attributes\Timeout;

#[Timeout(120)]
class ProcessPodcast implements ShouldQueue
{
    // ...
}
```

有時，像 Socket 或對外 HTTP 連線這類 I/O 阻塞行程可能不會遵循您指定的超時時間。因此，在使用這些功能時，您也應該始終嘗試使用它們自己的 API 來指定超時時間。例如，在使用 [Guzzle](https://docs.guzzlephp.org) 時，您應該始終指定連線與請求的超時值。

> [!WARNING]
> 必須安裝 [PCNTL](https://www.php.net/manual/en/book.pcntl.php) PHP 擴充套件才能指定 Job 超時時間。此外，Job 的「timeout」值應始終小於其[「retry after」](#job-expiration)值。否則，Job 可能在實際上尚未執行完畢或超時前就被重新嘗試。當使用 `--once` 選項執行 `queue:work` 指令時，`--timeout` 選項沒有作用。


<a name="failing-on-timeout"></a>
#### 超時時標記為失敗

如果您想表示 Job 在超時時應被標記為[失敗](#dealing-with-failed-jobs)，您可以在 Job 類別上使用 `FailOnTimeout` 屬性：

```php
<?php

namespace App\Jobs;

use Illuminate\Queue\Attributes\FailOnTimeout;

#[FailOnTimeout]
class ProcessPodcast implements ShouldQueue
{
    // ...
}
```

> [!NOTE]
> 預設情況下，當 Job 超時時，它會消耗一次嘗試並被釋放回佇列（如果允許重試）。然而，如果您將 Job 設定為超時即失敗，則無論 tries 設定的值為何，它都不會被重試。

<a name="sqs-fifo-and-fair-queues"></a>
### SQS FIFO 與公平佇列

Laravel 支援 [Amazon SQS FIFO (先進先出)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html) 與 [fair (公平)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fair-queues.html) 佇列。FIFO 佇列允許你按照訊息發送的精確順序處理 Job，同時透過訊息去重 (Deduplication) 確保正好處理一次 (Exactly-once processing)。

FIFO 佇列需要一個訊息群組 ID (Message group ID) 來決定哪些 Job 可以同時執行。具有相同群組 ID 的 Job 會按順序處理，而具有不同群組 ID 的訊息則可以同時處理。

Laravel 提供了一個流暢的 `onGroup` 方法，用於在分派 Job 時指定訊息群組 ID：

```php
ProcessOrder::dispatch($order)
    ->onGroup("customer-{$order->customer_id}");
```

SQS FIFO 佇列支援訊息去重，以確保正好處理一次。請在你的 Job 類別中實作 `deduplicationId` 方法來提供自訂的去重 ID：

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


<a name="fair-queues"></a>
#### 公平佇列

如果你使用的是 SQS 標準佇列，設定訊息群組將會啟用公平佇列 (Fair queueing)。換句話說，一旦你指定了群組，SQS 就會使用它們來維護跨租戶/工作負載 (Tenants / workloads) 的公平傳遞。不需要設定額外的 Laravel 設定。

除了在分派時呼叫 `onGroup` 之外，你也可以直接在 Job 上定義 `messageGroup` 方法：

```php
<?php

namespace App\Jobs;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

class ProcessOrder implements ShouldQueue
{
    use Queueable;

    // ...

    /**
     * Get the job's message group.
     */
    public function messageGroup(): string
    {
        return "customer-{$this->order->customer_id}";
    }
}
```


<a name="fifo-listeners-mail-and-notifications"></a>
#### FIFO Listener、郵件與通知

使用 FIFO 佇列時，你還需要為 Listener、郵件 (Mail) 和通知 (Notification) 定義訊息群組。或者，你可以將這些物件的佇列實例分派給非 FIFO 佇列。

若要為[佇列化事件 Listener](/docs/{{version}}/events#queued-event-listeners)定義訊息群組，請在 Listener 上定義 `messageGroup` 方法。你也可以選擇性地定義 `deduplicationId` 方法：

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

發送準備在 FIFO 佇列排隊的[郵件訊息](/docs/{{version}}/mail)時，應在發送通知時呼叫 `onGroup` 方法，並可選擇性地呼叫 `withDeduplicator` 方法：

```php
use App\Mail\InvoicePaid;
use Illuminate\Support\Facades\Mail;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

Mail::to($request->user())->send($invoicePaid);
```

發送準備在 FIFO 佇列排隊的[通知](/docs/{{version}}/notifications)時，應在發送通知時呼叫 `onGroup` 方法，並可選擇性地呼叫 `withDeduplicator` 方法：

```php
use App\Notifications\InvoicePaid;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

$user->notify($invoicePaid);
```


<a name="queue-failover"></a>
### 佇列故障移轉

`failover` 佇列驅動程式在推送 Job 到佇列時提供了自動故障移轉 (Failover) 功能。如果 `failover` 設定的主要佇列連線因任何原因失敗，Laravel 將自動嘗試將 Job 推送到清單中下一個設定好的連線。這對於在佇列可靠性至關重要的正式環境中確保高可用性 (High availability) 特別有用。

要設定故障移轉佇列連線，請指定 `failover` 驅動程式並提供一個按順序嘗試的連線名稱陣列。預設情況下，Laravel 在應用程式的 `config/queue.php` 設定檔中包含了一個範例故障移轉設定：

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

設定好使用 `failover` 驅動程式的連線後，你需要將故障移轉連線設為應用程式 `.env` 檔案中的預設佇列連線，才能使用故障移轉功能：

```ini
QUEUE_CONNECTION=failover
```

接下來，為故障移轉連線清單中的每個連線啟動至少一個 Worker：

```bash
php artisan queue:work redis
php artisan queue:work database
```

> [!NOTE]
> 你不需要為使用 `sync`、`background` 或 `deferred` 佇列驅動程式的連線執行 Worker，因為這些驅動程式會在當前的 PHP 行程中處理 Job。

當佇列連線操作失敗且觸發故障移轉時，Laravel 將分派 `Illuminate\Queue\Events\QueueFailedOver` 事件，允許你回報或記錄佇列連線已失敗。

> [!NOTE]
> 如果你使用 Laravel Horizon，請記住 Horizon 只管理 Redis 佇列。如果你的故障移轉清單包含 `database`，你應該在 Horizon 之外同時執行一般的 `php artisan queue:work database` 行程。

<a name="error-handling"></a>
### 錯誤處理

當處理 Job 拋出 Exception 時，Job 將會自動釋放回佇列以便重新嘗試執行。Job 會持續被釋放，直到達到您應用程式所允許的最大嘗試次數為止。最大嘗試次數定義於 `queue:work` Artisan 指令所使用的 `--tries` 切換開關。或者，也可以在 Job 類別本身定義最大嘗試次數。關於執行佇列 Worker 的更多資訊[可在下方找到](#running-the-queue-worker)。


<a name="manually-releasing-a-job"></a>
#### 手動釋放 Job

有時候，您可能希望手動將 Job 釋放回佇列，以便稍後再次嘗試。您可以透過呼叫 `release` 方法來完成：

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

預設情況下，`release` 方法會將 Job 釋放回佇列以立即進行處理。然而，您可以透過傳遞整數或日期實例給 `release` 方法，指示佇列在經過指定的秒數前，不要讓該 Job 可被處理：

```php
$this->release(10);

$this->release(now()->plus(seconds: 10));
```


<a name="manually-failing-a-job"></a>
#### 手動讓 Job 失敗

偶爾，您可能需要手動將 Job 標記為「失敗」。為此，您可以呼叫 `fail` 方法：

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

如果您想因為擷取到的 Exception 而將 Job 標記為失敗，可以將該 Exception 傳遞給 `fail` 方法。或者為求方便，您也可以傳遞字串錯誤訊息，系統會自動為您將其轉換為 Exception：

```php
$this->fail($exception);

$this->fail('Something went wrong.');
```

> [!NOTE]
> 關於失敗 Job 的更多資訊，請參閱[處理失敗 Job 的文件](#dealing-with-failed-jobs)。


<a name="fail-jobs-on-exceptions"></a>
#### 遇到特定 Exception 時讓 Job 失敗

`FailOnException` [Job 中介層](#job-middleware)允許您在拋出特定 Exception 時跳過後續重試。這允許在發生暫時性 Exception（例如外部 API 錯誤）時進行重試，但在發生持續性 Exception（例如使用者的權限已被撤銷）時讓 Job 永久失敗：

```php
<?php

namespace App\Jobs;

use App\Models\User;
use Illuminate\Auth\Access\AuthorizationException;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;
use Illuminate\Queue\Attributes\Tries;
use Illuminate\Queue\Middleware\FailOnException;
use Illuminate\Support\Facades\Http;

#[Tries(3)]
class SyncChatHistory implements ShouldQueue
{
    use Queueable;

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

Laravel 的 Job 批次處理 (Job Batching) 功能允許您輕鬆地平行執行一組 Job，並在該批次 Job 全部執行完成後執行特定動作。

在開始之前，您應該先建立資料庫 Migration 以建立用來存放 Job 批次元資料 (Meta Information，例如完成百分比) 的資料表。此 Migration 可以使用 `make:queue-batches-table` Artisan 指令來產生：

```shell
php artisan make:queue-batches-table

php artisan migrate
```


<a name="defining-batchable-jobs"></a>
### 定義可批次處理的 Job

要定義可進行批次處理的 Job，您應該像平常一樣[建立可佇列化的 Job](#creating-jobs)；不過，您需要將 `Illuminate\Bus\Batchable` Trait 加入至該 Job 類別中。這個 Trait 提供了存取 `batch` 方法的功能，可用於取得該 Job 目前正在其中執行的批次：

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

要分派一個 Job 批次，您應該使用 `Bus` Facade 的 `batch` 方法。當然，批次處理在結合完成回呼 (Callback) 時特別有用。因此，您可以使用 `then`、`catch` 和 `finally` 方法來為該批次定義完成回呼。當這些回呼被呼叫時，每一個回呼都會接收到一個 `Illuminate\Bus\Batch` 實例。

當執行多個佇列 Worker 時，批次中的 Job 將會平行處理。因此，Job 完成的順序可能與它們被新增至批次中的順序不同。請參閱我們關於 [Job 鏈結與批次](#chains-and-batches) 的文件，以瞭解如何依序執行一系列 Job。

在這個範例中，我們假設要將一個 Job 批次排入佇列，每個 Job 分別處理來自 CSV 檔案的指定行數：

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

批次的 ID（可透過 `$batch->id` 屬性存取）可用於在批次被分派後[查詢 Laravel 命令匯流排 (Command Bus)](#inspecting-batches) 以取得該批次的資訊。

> [!WARNING]
> 由於批次回呼是由 Laravel 佇列進行序列化並在稍後執行，因此您不應在回呼中使用 `$this` 變數。此外，由於批次化的 Job 被包裹在資料庫交易中，因此不應在這些 Job 內部執行會引發隱式提交 (Implicit Commit) 的資料庫語句。


<a name="naming-batches"></a>
#### 命名批次

如果為批次命名，某些工具（例如 [Laravel Horizon](/docs/{{version}}/horizon) 與 [Laravel Telescope](/docs/{{version}}/telescope)）可以為批次提供更友善的除錯資訊。若要為批次指定自訂名稱，可以在定義批次時呼叫 `name` 方法：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import CSV')->dispatch();
```


<a name="batch-connection-queue"></a>
#### 批次連線與佇列

如果您想指定批次 Job 所使用的連線與佇列，可以使用 `onConnection` 與 `onQueue` 方法。所有批次化的 Job 都必須在相同的連線與佇列中執行：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->onConnection('redis')->onQueue('imports')->dispatch();
```


<a name="chains-and-batches"></a>
### 鏈結與批次

您可以在批次中將一組[鏈結 Job](#job-chaining) 放入陣列中來進行定義。例如，我們可以平行執行兩個 Job 鏈結，並在兩個 Job 鏈結都處理完成時執行回呼：

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

相對地，您也可以透過在[鏈結](#job-chaining)內部定義批次來於鏈結中執行 Job 批次。例如，您可以先執行一個 Job 批次來發布多個 Podcast，然後再執行一個 Job 批次來發送發布通知：

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

有時候，從批次內的 Job 為該批次新增額外的 Job 會很有用。當您需要批次處理數千個 Job，且在 Web 請求期間分派可能需要太長時間時，這種模式非常有用。因此，您可以改為分派一個初始的「載入器 (loader)」Job 批次，讓它為該批次填充 (Hydrate) 更多的 Job：

```php
$batch = Bus::batch([
    new LoadImportBatch,
    new LoadImportBatch,
    new LoadImportBatch,
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import Contacts')->dispatch();
```

在此範例中，我們將使用 `LoadImportBatch` Job 來為該批次填充額外 Job。為實現此目的，我們可以使用透過 Job 的 `batch` 方法所取得之批次實例上的 `add` 方法：

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
> 您只能從屬於同一個批次的 Job 內部將 Job 新增至該批次中。

<a name="inspecting-batches"></a>
### 檢視批次

提供給批次完成回調函式的 `Illuminate\Bus\Batch` 實例擁有一系列的屬性與方法，能協助您互動與檢視指定的 Job 批次：

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
#### 從路由傳回批次

所有 `Illuminate\Bus\Batch` 實例皆可序列化為 JSON，這意味著您可以直接從應用程式的路由中傳回它們，以取得包含該批次資訊（包括完成進度）的 JSON 負載。這使得在應用程式的 UI 中顯示批次完成進度的資訊變得非常方便。

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

有時候您可能需要取消特定批次的執行。這可以透過呼叫 `Illuminate\Bus\Batch` 實例上的 `cancel` 方法來完成：

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

正如您在前面的範例中所注意到的，批次處理的 Job 通常應該在繼續執行之前，檢查其對應的批次是否已被取消。然而，為了方便起見，您可以改為將 `SkipIfBatchCancelled` [中介層](#job-middleware) 指派給 Job。顧名思義，如果對應的批次已被取消，此中介層將指示 Laravel 不要處理該 Job：

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

當批次處理中的 Job 失敗時，`catch` 回調函式（如果有設定的話）將會被呼叫。該回調函式僅會在批次中第一個失敗的 Job 發生的時候被呼叫一次。

<a name="allowing-failures"></a>
#### 允許失敗

當批次中的 Job 失敗時，Laravel 會自動將該批次標記為「已取消」。如果您希望的話，可以停用此行為，讓 Job 的失敗不會自動將批次標記為已取消。這可以透過在分派批次時呼叫 `allowFailures` 方法來實現：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->allowFailures()->dispatch();
```

您可以選擇為 `allowFailures` 方法提供一個 Closure，該 Closure 將會在每個 Job 失敗時執行：

```php
$batch = Bus::batch([
    // ...
])->allowFailures(function (Batch $batch, $exception) {
    // Handle individual job failures...
})->dispatch();
```

<a name="retrying-failed-batch-jobs"></a>
#### 重試失敗的批次 Job

為了方便起見，Laravel 提供了一個 `queue:retry-batch` Artisan 指令，讓您可以輕鬆地重試指定批次的所有失敗 Job。此指令接受需要重試失敗 Job 的批次 UUID：

```shell
php artisan queue:retry-batch 32dbc76c-4f82-4749-b610-a639fe0099b5
```

<a name="pruning-batches"></a>
### 修剪批次記錄

如果沒有進行修剪，`job_batches` 資料表會非常快速地累積記錄。為了減輕這種情況，您應該[排程](/docs/{{version}}/scheduling) `queue:prune-batches` Artisan 指令每日執行：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches')->daily();
```

預設情況下，所有超過 24 小時的已完成批次都將被修剪。您可以在呼叫指令時使用 `hours` 選項來決定批次資料要保留多久。例如，以下指令將會刪除所有在 48 小時前完成的批次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48')->daily();
```

有時候，您的 `job_batches` 資料表可能會累積一些從未成功完成的批次記錄，例如批次中有 Job 失敗且該 Job 從未成功重試的情況。您可以透過 `unfinished` 選項指示 `queue:prune-batches` 指令來修剪這些未完成的批次記錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --unfinished=72')->daily();
```

同樣地，您的 `job_batches` 資料表也可能會累積已取消批次的記錄。您可以透過 `cancelled` 選項指示 `queue:prune-batches` 指令來修剪這些已取消的批次記錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --cancelled=72')->daily();
```

<a name="storing-batches-in-dynamodb"></a>
### 將批次儲存於 DynamoDB

Laravel 也支援將批次的 Meta 資訊儲存在 [DynamoDB](https://aws.amazon.com/dynamodb) 中，而不是關聯式資料庫。不過，您需要手動建立一個 DynamoDB 資料表來儲存所有的批次記錄。

通常，此資料表應命名為 `job_batches`，但您應該根據應用程式 `queue` 設定檔中的 `queue.batching.table` 設定值來為資料表命名。


<a name="dynamodb-batch-table-configuration"></a>
#### DynamoDB 批次資料表設定

`job_batches` 資料表應該有一個名為 `application` 的字串型態主分割區鍵 (Primary Partition Key)，以及一個名為 `id` 的字串型態主排序鍵 (Primary Sort Key)。鍵的 `application` 部分將包含您的應用程式名稱（由應用程式 `app` 設定檔中的 `name` 設定值所定義）。由於應用程式名稱是 DynamoDB 資料表鍵的一部分，因此您可以透過同一個資料表來儲存多個 Laravel 應用程式的 Job 批次。

此外，如果您希望利用[自動批次修剪](#pruning-batches-in-dynamodb)功能，可以為您的資料表定義 `ttl` 屬性。


<a name="dynamodb-configuration"></a>
#### DynamoDB 設定

接著，請安裝 AWS SDK，以便您的 Laravel 應用程式可以與 Amazon DynamoDB 進行通訊：

```shell
composer require aws/aws-sdk-php
```

然後，將 `queue.batching.driver` 設定選項的值設為 `dynamodb`。此外，您應該在 `batching` 設定陣列中定義 `key`、`secret` 以及 `region` 設定選項。這些選項將用於向 AWS 進行驗證。當使用 `dynamodb` 驅動時，就不需要設定 `queue.batching.database` 設定選項：

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

當使用 [DynamoDB](https://aws.amazon.com/dynamodb) 來儲存 Job 批次資訊時，用於修剪儲存在關聯式資料庫中批次的傳統修剪指令將無法運作。相反地，您可以利用 [DynamoDB 原生的 TTL 功能](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)來自動刪除舊批次的記錄。

如果您在定義 DynamoDB 資料表時包含了 `ttl` 屬性，則可以定義設定參數來指示 Laravel 如何修剪批次記錄。`queue.batching.ttl_attribute` 設定值定義了保存 TTL 的屬性名稱，而 `queue.batching.ttl` 設定值則定義了自最後一次更新記錄起算，批次記錄可從 DynamoDB 資料表中被移除的秒數：

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
## 佇列化 Closure

除了將 Job 類別分派至佇列外，您也可以分派 Closure。這非常適合用於需要在當前請求週期之外執行的快速、簡單任務。當將 Closure 分派至佇列時，Closure 的程式碼內容會經過加密簽署，因此在傳輸過程中無法被竄改：

```php
use App\Models\Podcast;

$podcast = Podcast::find(1);

dispatch(function () use ($podcast) {
    $podcast->publish();
});
```

若要為排入佇列的 Closure 指定名稱，以便在佇列報告儀表板中使用，或在 `queue:work` 指令中顯示，您可以使用 `name` 方法：

```php
dispatch(function () {
    // ...
})->name('Publish Podcast');
```

使用 `catch` 方法，您可以提供一個 Closure，當排入佇列的 Closure 在耗盡所有佇列的[已設定重試嘗試次數](#max-job-attempts-and-timeout)後仍未能成功完成時執行：

```php
use Throwable;

dispatch(function () use ($podcast) {
    $podcast->publish();
})->catch(function (Throwable $e) {
    // This job has failed...
});
```

> [!WARNING]
> 由於 `catch` 回呼會被序列化並於稍後由 Laravel 佇列執行，您不應該在 `catch` 回呼中使用 `$this` 變數。

<a name="running-the-queue-worker"></a>
## 執行佇列 Worker


<a name="the-queue-work-command"></a>
### `queue:work` 指令

Laravel 包含了一個 Artisan 指令，可以啟動佇列 Worker 並在新的 Job 被推送到佇列時進行處理。您可以透過 `queue:work` Artisan 指令來執行 Worker。請注意，一旦 `queue:work` 指令啟動後，它會持續執行，直到被手動停止或您關閉終端機為止：

```shell
php artisan queue:work
```

> [!NOTE]
> 若要讓 `queue:work` 行程在背景永久持續執行，您應該使用行程監控工具（如 [Supervisor](#supervisor-configuration)）來確保佇列 Worker 不會停止運作。

如果您希望在指令的輸出中包含已處理的 Job ID、連線名稱和佇列名稱，可以在呼叫 `queue:work` 指令時加入 `-v` 旗標：

```shell
php artisan queue:work -v
```

請記住，佇列 Worker 是常駐型的行程 (Long-lived processes)，並會將啟動後的應用程式狀態儲存在記憶體中。因此，它們在啟動後不會察覺到程式碼庫的變更。所以，在您的部署過程中，請務必[重啟您的佇列 Worker](#queue-workers-and-deployment)。此外，請記住，應用程式所建立或修改的任何靜態狀態 (Static state) 不會在 Job 之間自動重設。

或者，您也可以執行 `queue:listen` 指令。當使用 `queue:listen` 指令時，當您想要重新載入更新後的程式碼或重設應用程式狀態時，不需要手動重啟 Worker；然而，這個指令的效率顯著低於 `queue:work` 指令：

```shell
php artisan queue:listen
```


<a name="running-multiple-queue-workers"></a>
#### 執行多個佇列 Worker

要指派多個 Worker 到同一個佇列並同時處理 Job，您只需要啟動多個 `queue:work` 行程即可。這可以透過在終端機開啟多個分頁在機端執行，或者在正式環境中使用行程管理員的設定來完成。[當使用 Supervisor 時](#supervisor-configuration)，您可以使用 `numprocs` 設定值。


<a name="specifying-the-connection-queue"></a>
#### 指定連線與佇列

您也可以指定 Worker 應該使用哪一個佇列連線。傳遞給 `work` 指令的連線名稱應該對應到 `config/queue.php` 設定檔中定義的其中一個連線：

```shell
php artisan queue:work redis
```

預設情況下，`queue:work` 指令只會處理給定連線上的預設佇列。然而，您還可以透過僅處理給定連線上的特定佇列來更進一步自訂您的佇列 Worker。例如，如果您的所有電子郵件都在 `redis` 佇列連線上的 `emails` 佇列中處理，您可以發出以下指令來啟動僅處理該佇列的 Worker：

```shell
php artisan queue:work redis --queue=emails
```


<a name="processing-a-specified-number-of-jobs"></a>
#### 處理指定數量的 Job

`--once` 選項可用於指示 Worker 僅處理佇列中的單個 Job：

```shell
php artisan queue:work --once
```

`--max-jobs` 選項可用於指示 Worker 在處理指定數量的 Job 後退出。當與 [Supervisor](#supervisor-configuration) 結合使用時，這個選項非常有用，這樣您的 Worker 就會在處理完指定數量的 Job 後自動重啟，從而釋放它們可能累積的記憶體：

```shell
php artisan queue:work --max-jobs=1000
```


<a name="processing-all-queued-jobs-then-exiting"></a>
#### 處理所有排隊的 Job 後退出

`--stop-when-empty` 選項可用於指示 Worker 處理所有 Job 後順暢退出。如果您希望在佇列清空後關閉容器，此選項在 Docker 容器內處理 Laravel 佇列時會非常有用：

```shell
php artisan queue:work --stop-when-empty
```


<a name="processing-jobs-for-a-given-number-of-seconds"></a>
#### 處理 Job 指定的秒數

`--max-time` 選項可用於指示 Worker 處理 Job 指定的秒數後退出。當與 [Supervisor](#supervisor-configuration) 結合使用時，這個選項非常有用，這樣您的 Worker 就會在處理 Job 達特定時間後自動重啟，從而釋放它們可能累積的記憶體：

```shell
# Process jobs for one hour and then exit...
php artisan queue:work --max-time=3600
```


<a name="worker-sleep-duration"></a>
#### Worker 睡眠時間

當佇列中有可用 Job 時，Worker 會持續處理 Job，Job 之間不會有延遲。然而，`sleep` 選項決定了如果沒有可用的 Job，Worker 將「睡眠」多少秒。當然，在睡眠期間，Worker 不會處理任何新的 Job：

```shell
php artisan queue:work --sleep=3
```


<a name="maintenance-mode-queues"></a>
#### 維護模式與佇列

當您的應用程式處於[維護模式](/docs/{{version}}/configuration#maintenance-mode)時，不會處理任何佇列中的 Job。一旦應用程式離開維護模式，Job 將會恢復正常處理。

若要強制您的佇列 Worker 即使在啟用維護模式時也處理 Job，您可以使用 `--force` 選項：

```shell
php artisan queue:work --force
```


<a name="resource-considerations"></a>
#### 資源考量

Daemon 佇列 Worker 在處理每個 Job 前不會「重新引導 (Reboot)」框架。因此，您應該在每個 Job 完成後釋放任何重度的資源。例如，如果您使用 [GD 函式庫](https://www.php.net/manual/en/book.image.php)進行[圖片處理](/docs/{{version}}/images)，您應該在處理完圖片後使用 `imagedestroy` 釋放記憶體。


<a name="queue-priorities"></a>
### 佇列優先順序

有時您可能希望優先處理佇列的方式。例如，在您的 `config/queue.php` 設定檔中，您可以將 `redis` 連線的預設 `queue` 設定為 `low`。然而，偶爾您可能希望將 Job 推送到 `high` 優先順序佇列，如下所示：

```php
dispatch((new Job)->onQueue('high'));
```

若要啟動一個在繼續處理 `low` 佇列上的任何 Job 之前驗證是否已處理所有 `high` 佇列 Job 的 Worker，請將逗號分隔的佇列名稱清單傳遞給 `work` 指令：

```shell
php artisan queue:work --queue=high,low
```


<a name="queue-workers-and-deployment"></a>
### 佇列 Worker 與部署

由於佇列 Worker 是常駐型的行程，因此如果不重新啟動，它們就不會察覺到程式碼的變更。因此，部署使用佇列 Worker 的應用程式最簡單方法就是在部署過程中重啟 Worker。您可以透過執行 `queue:restart` 指令來順暢地重啟所有 Worker：

```shell
php artisan queue:restart
```

這個指令會指示所有佇列 Worker 在完成處理當前 Job 後順暢地退出，以確保不會遺失任何現有的 Job。由於佇列 Worker 會在執行 `queue:restart` 指令時退出，因此您應該執行像是 [Supervisor](#supervisor-configuration) 之類的行程管理員來自動重啟佇列 Worker。

> [!NOTE]
> 佇列使用[快取](/docs/{{version}}/cache)來儲存重啟訊號，因此在使用此功能之前，您應該確認應用程式已為其正確設定快取驅動程式。

<a name="reacting-to-worker-signals"></a>
### 回應 Worker 信號

當佇列 Worker 在處理 Job 時收到終止信號（例如 `SIGQUIT`、`SIGTERM` 或 `SIGINT`），Worker 會在退出前完成其當前的 Job。然而，您的 Job 可能需要在行程(Processes)被您的伺服器或容器編排器停止之前回應該信號。例如，一個長時間執行的匯入 Job 可能需要停止拉取新記錄並儲存其當前進度。

若要在 Job 內部回應 Worker 信號，請實作 `Illuminate\Contracts\Queue\Interruptible` 介面並在您的 Job 中定義 `interrupted` 方法。Worker 收到的信號代碼將被傳遞給 `interrupted` 方法：

```php
<?php

namespace App\Jobs;

use App\Models\Import;
use Illuminate\Contracts\Queue\Interruptible;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

class ImportProducts implements ShouldQueue, Interruptible
{
    use Queueable;

    protected bool $shouldStop = false;

    /**
     * Create a new job instance.
     */
    public function __construct(
        public Import $import,
    ) {}

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        foreach ($this->import->pendingRows() as $row) {
            if ($this->shouldStop) {
                break;
            }

            // Import the product row...
        }

        $this->import->saveProgress();
    }

    /**
     * Handle a signal received by the queue worker.
     */
    public function interrupted(int $signal): void
    {
        $this->shouldStop = true;
    }
}
```

`interrupted` 方法僅在 Job 正在執行且 Worker 收到行程信號時才會被調用。它並不是 [超時設定](#worker-timeouts) 或 Job 的 [`failed` 方法](#cleaning-up-after-failed-jobs) 的替代方案。


<a name="job-expirations-and-timeouts"></a>
### Job 過期與超時


<a name="job-expiration"></a>
#### Job 過期

在您的 `config/queue.php` 設定檔中，每個佇列連線都定義了一個 `retry_after` 選項。該選項指定了佇列連線在重試正在處理的 Job 之前應該等待多少秒。例如，如果 `retry_after` 的值設為 `90`，則當 Job 已處理 90 秒而未被釋放或刪除時，該 Job 將被釋放回佇列中。通常，您應該將 `retry_after` 的值設定為您的 Job 合理完成處理所需的最大秒數。

> [!WARNING]
> 唯一不包含 `retry_after` 值的佇列連線是 Amazon SQS。SQS 會根據在 AWS 主控台中管理的 [Default Visibility Timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html) 來重試 Job。


<a name="worker-timeouts"></a>
#### Worker 超時

`queue:work` Artisan 指令提供了一個 `--timeout` 選項。預設情況下，`--timeout` 的值為 60 秒。如果 Job 處理的時間超過超時值指定的秒數，處理該 Job 的 Worker 將會因錯誤而退出。通常，Worker 會由 [伺服器上設定的行程管理器](#supervisor-configuration) 自動重啟：

```shell
php artisan queue:work --timeout=60
```

`retry_after` 設定選項和 `--timeout` CLI 選項是不同的，但它們共同運作以確保 Job 不會遺失，且 Job 只會被成功處理一次。

> [!WARNING]
> `--timeout` 的值應始終比您的 `retry_after` 設定值短至少數秒。這將確保處理凍結 Job 的 Worker 總是在重試該 Job 之前被終止。如果您的 `--timeout` 選項長於 `retry_after` 設定值，您的 Job 可能會被處理兩次。


<a name="pausing-and-resuming-queue-workers"></a>
### 暫停與恢復佇列 Worker

有時候，您可能需要暫時防止佇列 Worker 處理新的 Job，而不需要完全停止 Worker。例如，您可能希望在系統維護期間暫停 Job 處理。Laravel 提供了 `queue:pause` 和 `queue:continue` Artisan 指令來暫停與恢復佇列 Worker。

若要暫停特定的佇列，請提供佇列連線名稱與佇列名稱：

```shell
php artisan queue:pause database:default
```

在這個範例中，`database` 是佇列連線名稱，而 `default` 是佇列名稱。一旦佇列被暫停，處理該佇列中 Job 的任何 Worker 將繼續完成其當前的 Job，但在該佇列恢復之前不會接收任何新的 Job。

若要暫停所有連線上每個佇列的 Job 處理，請使用 `--all` 選項：

```shell
php artisan queue:pause --all
```

若要恢復處理已暫停佇列上的 Job，請使用 `queue:continue` 指令：

```shell
php artisan queue:continue database:default
```

若要恢復所有連線上每個佇列的 Job 處理，請將 `--all` 選項與 `queue:resume` 指令搭配使用：

```shell
php artisan queue:resume --all
```

恢復佇列後，Worker 將立即開始處理該佇列中的新 Job。恢復所有佇列並不會恢復單獨暫停的佇列。請注意，暫停佇列並不會停止 Worker 行程本身——它只會防止 Worker 從指定佇列中處理新的 Job。


<a name="worker-restart-and-pause-signals"></a>
#### Worker 重啟與暫停信號

預設情況下，佇列 Worker 會在每次 Job 迭代時輪詢快取驅動程式以獲取重啟和暫停信號。雖然這種輪詢對於回應 `queue:restart` 和 `queue:pause` 指令不可或缺，但它確實會帶來微小的效能開銷。

如果您需要最佳化效能且不需要這些中斷功能，您可以透過調用 `Queue` Facade 上的 `withoutInterruptionPolling` 方法來全域停用此輪詢。這通常應該在 `AppServiceProvider` 的 `boot` 方法中完成：

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

或者，您可以透過在 `Illuminate\Queue\Worker` 類別上設定靜態 `$restartable` 或 `$pausable` 屬性來分別停用重啟或暫停輪詢：

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

在正式環境 (Production) 中，您需要一種方法來保持 `queue:work` 行程 (Processes) 持續運行。`queue:work` 行程可能會因為各種原因停止運行，例如超出 Worker 的超時時間或執行了 `queue:restart` 指令。

因此，您需要設定一個行程監控工具 (Process Monitor)，用來偵測您的 `queue:work` 行程何時結束並自動重新啟動它們。此外，行程監控工具還可以讓您指定想要同時執行多少個 `queue:work` 行程。Supervisor 是 Linux 環境中常用的行程監控工具，我們將在接下來的文件中討論如何設定它。

<a name="installing-supervisor"></a>
#### 安裝 Supervisor

Supervisor 是適用於 Linux 作業系統的行程監控工具，如果您的 `queue:work` 行程失敗，它會自動重新啟動它們。若要在 Ubuntu 上安裝 Supervisor，您可以使用以下指令：

```shell
sudo apt-get install supervisor
```

> [!NOTE]
> 如果自行設定與管理 Supervisor 聽起來太過繁瑣，可以考慮使用 [Laravel Cloud](https://cloud.laravel.com)，它提供了一個全代管平台來執行 Laravel 佇列 Worker。

<a name="configuring-supervisor"></a>
#### 設定 Supervisor

Supervisor 設定檔通常儲存在 `/etc/supervisor/conf.d` 目錄中。在此目錄下，您可以建立任意數量的設定檔，以指示 Supervisor 應該如何監控您的行程。例如，讓我們建立一個 `laravel-worker.conf` 檔案來啟動並監控 `queue:work` 行程：

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

在此範例中，`numprocs` 設定會指示 Supervisor 執行 8 個 `queue:work` 行程並監控所有這些行程，若它們失敗則自動重新啟動。您應該修改設定中的 `command` 設定值，以反映您想要的佇列連線與 Worker 選項。

> [!WARNING]
> 您應該確保 `stopwaitsecs` 的值大於執行時間最長的 Job 所消耗的秒數。否則，Supervisor 可能會在 Job 尚未處理完成前將其刪除 (Kill)。

<a name="starting-supervisor"></a>
#### 啟動 Supervisor

設定檔建立完成後，您可以使用以下指令更新 Supervisor 設定並啟動行程：

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start "laravel-worker:*"
```

有關 Supervisor 的更多資訊，請參考 [Supervisor 官方文件](http://supervisord.org/index.html)。

<a name="dealing-with-failed-jobs"></a>
## 處理失敗的 Job

有時候您的佇列 Job 會失敗。別擔心，事情並不總是按計劃進行！Laravel 提供了一種便捷的方式來[指定 Job 應嘗試的最大次數](#max-job-attempts-and-timeout)。在非同步 Job 超過此嘗試次數後，它將被插入到 `failed_jobs` 資料庫資料表中。[同步分派的 Job](/docs/{{version}}/queues#synchronous-dispatching) 失敗時不會儲存於此資料表，其 Exception 會立即由應用程式處理。

在新建立的 Laravel 應用程式中，通常已經包含用於建立 `failed_jobs` 資料表的遷移。但是，如果您的應用程式不包含此資料表的遷移，您可以使用 `make:queue-failed-table` Artisan 指令來建立該遷移：

```shell
php artisan make:queue-failed-table

php artisan migrate
```

執行 [佇列 Worker](#running-the-queue-worker) 行程時，您可以使用 `queue:work` 指令上的 `--tries` 開關來指定 Job 應嘗試的最大次數。如果您未指定 `--tries` 選項的值，Job 將僅嘗試一次，或者嘗試由 Job 類別的 `Tries` 屬性所指定的次數：

```shell
php artisan queue:work redis --tries=3
```

使用 `--backoff` 選項，您可以指定 Laravel 在重試遇到 Exception 的 Job 之前應等待多少秒。預設情況下，Job 會立即被釋放回佇列，以便可以再次嘗試：

```shell
php artisan queue:work redis --tries=3 --backoff=3
```

如果您想依據各個 Job 設定 Laravel 在重試遇到 Exception 的 Job 之前應等待多少秒，可以在 Job 類別上使用 `Backoff` 屬性：

```php
<?php

namespace App\Jobs;

use Illuminate\Queue\Attributes\Backoff;

#[Backoff(3)]
class ProcessPodcast implements ShouldQueue
{
    // ...
}
```

如果您需要更複雜的邏輯來決定 Job 的 backoff 時間，可以在 Job 類別上定義一個 `backoff` 方法：

```php
/**
 * Calculate the number of seconds to wait before retrying the job.
 */
public function backoff(): int
{
    return 3;
}
```

您可以透過定義一個 backoff 值陣列來輕鬆設定「指數級 (exponential)」backoff。在此範例中，若還有剩餘嘗試次數，第一次重試的延遲時間為 1 秒，第二次重試為 5 秒，第三次重試為 10 秒，之後每次重試均為 10 秒：

```php
<?php

namespace App\Jobs;

use Illuminate\Queue\Attributes\Backoff;

#[Backoff([1, 5, 10])]
class ProcessPodcast implements ShouldQueue
{
    // ...
}
```


<a name="cleaning-up-after-failed-jobs"></a>
### 在 Job 失敗後清理

當特定 Job 失敗時，您可能希望向使用者發送警報或復原由 Job 部分完成的任何操作。為實現此目的，您可以在 Job 類別上定義一個 `failed` 方法。導致 Job 失敗的 `Throwable` 實例將被傳遞給 `failed` 方法：

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
> 在調用 `failed` 方法之前會實例化該 Job 的新實例；因此，在 `handle` 方法中可能發生的任何類別屬性修改都將遺失。

失敗的 Job 不一定是指遇到未處理 Exception 的 Job。當 Job 耗盡其所有允許的嘗試次數時，也被視為失敗。這些嘗試次數可以透過以下幾種方式被消耗掉：

<div class="content-list" markdown="1">

- Job 超時。
- Job 在執行期間遇到未處理的 Exception。
- Job 手動或由中介層釋放回佇列。

</div>

如果最終嘗試由於 Job 執行期間拋出的 Exception 而失敗，該 Exception 將被傳遞給 Job 的 `failed` 方法。但是，如果 Job 由於達到允許的最大嘗試次數而失敗，則 `$exception` 將是 `Illuminate\Queue\MaxAttemptsExceededException` 的實例。同樣地，如果 Job 由於超過設定的超時時間而失敗，則 `$exception` 將是 `Illuminate\Queue\TimeoutExceededException` 的實例。


<a name="retrying-failed-jobs"></a>
### 重試失敗的 Job

要檢視已插入到 `failed_jobs` 資料庫資料表中的所有失敗 Job，您可以使用 `queue:failed` Artisan 指令：

```shell
php artisan queue:failed
```

`queue:failed` 指令將列出 Job ID、連線、佇列、失敗時間以及有關該 Job 的其他資訊。Job ID 可用於重試失敗的 Job。例如，要重試 ID 為 `ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece` 的失敗 Job，請發布以下指令：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece
```

如有必要，您可以向該指令傳遞多個 ID：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece 91401d2c-0784-4f43-824c-34f94a33c24d
```

您也可以重試特定佇列的所有失敗 Job：

```shell
php artisan queue:retry --queue=name
```

要重試所有失敗的 Job，請執行 `queue:retry` 指令並傳遞 `all` 作為 ID：

```shell
php artisan queue:retry all
```

如果您想刪除失敗的 Job，可以使用 `queue:forget` 指令：

```shell
php artisan queue:forget 91401d2c-0784-4f43-824c-34f94a33c24d
```

> [!NOTE]
> 使用 [Horizon](/docs/{{version}}/horizon) 時，您應該使用 `horizon:forget` 指令來刪除失敗的 Job，而不是 `queue:forget` 指令。

要從 `failed_jobs` 資料表中刪除所有失敗的 Job，您可以使用 `queue:flush` 指令：

```shell
php artisan queue:flush
```

`queue:flush` 指令會從您的佇列中移除所有失敗的 Job 記錄，無論失敗的 Job 有多久歷史。您可以使用 `--hours` 選項僅刪除特定小時數之前或更早失敗的 Job：

```shell
php artisan queue:flush --hours=48
```


<a name="ignoring-missing-models"></a>
### 忽略遺失的模型

將 Eloquent 模型注入到 Job 時，模型會在放入佇列之前自動序列化，並在處理 Job 時從資料庫重新取出。但是，如果在 Job 等待 Worker 處理期間該模型已被刪除，則您的 Job 可能會失敗並出現 `ModelNotFoundException`。

為了方便起見，您可以選擇在 Job 類別上使用 `DeleteWhenMissingModels` 屬性自動刪除模型遺失的 Job。當存在此屬性時，Laravel 將會安靜地捨棄該 Job 而不引發 Exception：

```php
<?php

namespace App\Jobs;

use Illuminate\Queue\Attributes\DeleteWhenMissingModels;

#[DeleteWhenMissingModels]
class ProcessPodcast implements ShouldQueue
{
    // ...
}
```


<a name="pruning-failed-jobs"></a>
### 修剪失敗的 Job

您可以透過調用 `queue:prune-failed` Artisan 指令來修剪應用程式 `failed_jobs` 資料表中的記錄：

```shell
php artisan queue:prune-failed
```

預設情況下，所有超過 24 小時的失敗 Job 記錄都將被修剪。如果您向該指令提供 `--hours` 選項，則僅保留在過去 N 個小時內插入的失敗 Job 記錄。例如，以下指令將刪除所有在 48 小時前插入的失敗 Job 記錄：

```shell
php artisan queue:prune-failed --hours=48
```

<a name="storing-failed-jobs-in-dynamodb"></a>
### 將失敗的 Job 儲存於 DynamoDB

Laravel 也支援將失敗的 Job 紀錄儲存於 [DynamoDB](https://aws.amazon.com/dynamodb)，而非關聯式資料庫資料表。不過，您必須手動建立一個 DynamoDB 資料表來儲存所有失敗的 Job 紀錄。通常這個資料表應命名為 `failed_jobs`，但您應該根據應用程式 `queue` 設定檔中的 `queue.failed.table` 設定值來為資料表命名。

`failed_jobs` 資料表應包含一個名為 `application` 的字串型別主分割鍵 (Primary Partition Key)，以及一個名為 `uuid` 的字串型別主排序鍵 (Primary Sort Key)。鍵值的 `application` 部分將包含您的應用程式名稱（如應用程式的 `app` 設定檔中的 `name` 設定值所定義）。由於應用程式名稱是 DynamoDB 資料表鍵值的一部分，因此您可以使用同一個資料表來儲存多個 Laravel 應用程式的失敗 Job。

此外，請確保您已安裝 AWS SDK，以便您的 Laravel 應用程式能與 Amazon DynamoDB 進行通訊：

```shell
composer require aws/aws-sdk-php
```

接著，將 `queue.failed.driver` 設定選項的值設為 `dynamodb`。此外，您應該在失敗 Job 設定陣列中定義 `key`、`secret` 與 `region` 設定選項。這些選項將用於 AWS 的身份驗證。當使用 `dynamodb` 驅動時，不需要設定 `queue.failed.database` 設定選項：

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
### 停用失敗 Job 的儲存

您可以將 `queue.failed.driver` 設定選項的值設為 `null`，以指示 Laravel 直接丟棄失敗的 Job 而不進行儲存。通常，這可以透過 `QUEUE_FAILED_DRIVER` 環境變數來完成：

```ini
QUEUE_FAILED_DRIVER=null
```


<a name="failed-job-events"></a>
### 失敗 Job 事件

若您想要註冊一個在 Job 失敗時被呼叫的事件監聽器，您可以使用 `Queue` Facade 的 `failing` 方法。例如，我們可以在 Laravel 隨附的 `AppServiceProvider` 的 `boot` 方法中為此事件附加一個 Closure：

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
> 使用 [Horizon](/docs/{{version}}/horizon) 時，您應該使用 `horizon:clear` 指令來清除佇列中的 Job，而不是使用 `queue:clear` 指令。

如果您想刪除預設連線的預設佇列中的所有 Job，可以使用 `queue:clear` Artisan 指令：

```shell
php artisan queue:clear
```

您也可以提供 `connection` 引數和 `queue` 選項，來刪除特定連線與佇列中的 Job：

```shell
php artisan queue:clear redis --queue=emails
```

> [!WARNING]
> 從佇列清除 Job 的功能僅適用於 SQS、Redis 和資料庫佇列驅動。此外，SQS 訊息刪除程序最多需要 60 秒，因此在您清除佇列後的 60 秒內發送到 SQS 佇列的 Job 可能也會被刪除。


<a name="monitoring-your-queues"></a>
## 監控您的佇列

如果您的佇列突然湧入大量 Job，可能會變得不堪重負，導致 Job 需要很長時間才能完成。若您希望，Laravel 可以在您的佇列 Job 數量超過指定門檻值時發出警報。

若要開始使用，您應該排程 `queue:monitor` 指令以[每分鐘執行一次](/docs/{{version}}/scheduling)。該指令接受您希望監控的佇列名稱以及您設定的 Job 數量門檻值：

```shell
php artisan queue:monitor redis:default,redis:deployments --max=100
```

單獨排程此指令不足以觸發通知警報來提醒您佇列過載的情況。當指令遇到 Job 數量超過您門檻值的佇列時，會分派一個 `Illuminate\Queue\Events\QueueBusy` 事件。您可以在應用程式的 `AppServiceProvider` 中監聽此事件，以便向您或您的開發團隊發送通知：

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
                $event->connectionName,
                $event->queue,
                $event->size
            ));
    });
}
```

<a name="testing"></a>
## 測試

當測試會分派 Job 的程式碼時，您可能希望指示 Laravel 不要實際執行 Job 本身，因為 Job 的程式碼可以獨立於分派它的程式碼並直接進行測試。當然，若要測試 Job 本身，您可以在測試中實例化 Job 實例並直接呼叫 `handle` 方法。

您可以使用 `Queue` Facade 的 `fake` 方法來防止佇列化的 Job 被實際推送到佇列中。呼叫 `Queue` Facade 的 `fake` 方法後，您便可以斷言應用程式是否有嘗試將 Job 推送到佇列：

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

    // Assert a job was pushed exactly once...
    Queue::assertPushedOnce(ShipOrder::class);

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

        // Assert a job was pushed exactly once...
        Queue::assertPushedOnce(ShipOrder::class);

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

您可以傳遞 Closure 給 `assertPushed`、`assertNotPushed`、`assertClosurePushed` 或 `assertClosureNotPushed` 方法，以斷言被推送的 Job 是否通過給定的「真值測試 (Truth Test)」。如果至少有一個被推送的 Job 通過了給定的真值測試，則該斷言將會成功：

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
### 模擬部分 Job

若您只需要模擬特定的 Job，同時允許其他 Job 正常執行，您可以將應該被模擬的 Job 類別名稱傳遞給 `fake` 方法：

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

您可以使用 `except` 方法來模擬除了指定 Job 集合之外的所有 Job：

```php
Queue::fake()->except([
    ShipOrder::class,
]);
```


<a name="testing-job-chains"></a>
### 測試 Job 鏈結

若要測試 Job 鏈結，您需要利用 `Bus` Facade 的模擬功能。`Bus` Facade 的 `assertChained` 方法可用於斷言 [Job 鏈結](/docs/{{version}}/queues#job-chaining) 是否已被分派。`assertChained` 方法接受一個鏈結 Job 陣列作為其第一個引數：

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

如上例所示，鏈結 Job 陣列可以是 Job 類別名稱的陣列。然而，您也可以提供實際 Job 實例的陣列。當這麼做時，Laravel 將確保這些 Job 實例與應用程式所分派的鏈結 Job 具有相同的類別以及相同的屬性值：

```php
Bus::assertChained([
    new ShipOrder,
    new RecordShipment,
    new UpdateInventory,
]);
```

您可以使用 `assertDispatchedWithoutChain` 方法來斷言 Job 是否在沒有 Job 鏈結的情況下被推送：

```php
Bus::assertDispatchedWithoutChain(ShipOrder::class);
```


<a name="testing-chain-modifications"></a>
#### 測試鏈結修改

如果鏈結中的 Job [將 Job 前置或附加到現有鏈結](#adding-jobs-to-the-chain)，您可以使用該 Job 的 `assertHasChain` 方法來斷言該 Job 是否具有預期的剩餘 Job 鏈結：

```php
$job = new ProcessPodcast;

$job->handle();

$job->assertHasChain([
    new TranscribePodcast,
    new OptimizePodcast,
    new ReleasePodcast,
]);
```

`assertDoesntHaveChain` 方法可用於斷言 Job 的剩餘鏈結是否為空：

```php
$job->assertDoesntHaveChain();
```


<a name="testing-chained-batches"></a>
#### 測試鏈結的批次

如果您的 Job 鏈結[包含 Job 批次](#chains-and-batches)，您可以透過在鏈結斷言中插入 `Bus::chainedBatch` 定義，來斷言該鏈結的批次符合您的預期：

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

`Bus` Facade 的 `assertBatched` 方法可用於斷言 [Job 批次](/docs/{{version}}/queues#job-batching) 已被分派。傳給 `assertBatched` 方法的 Closure 會接收一個 `Illuminate\Bus\PendingBatch` 實例，可用於檢視批次內的 Job：

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

可以在待處理批次上使用 `hasJobs` 方法，以驗證該批次是否包含預期的 Job。此方法接受 Job 實例、類別名稱或 Closure 的陣列：

```php
Bus::assertBatched(function (PendingBatch $batch) {
    return $batch->hasJobs([
        new ProcessCsvRow(row: 1),
        new ProcessCsvRow(row: 2),
        new ProcessCsvRow(row: 3),
    ]);
});
```

使用 Closure 時，Closure 會接收 Job 實例。預期的 Job 型別將會從 Closure 的型別提示（Type hint）推導出來：

```php
Bus::assertBatched(function (PendingBatch $batch) {
    return $batch->hasJobs([
        fn (ProcessCsvRow $job) => $job->row === 1,
        fn (ProcessCsvRow $job) => $job->row === 2,
        fn (ProcessCsvRow $job) => $job->row === 3,
    ]);
});
```

您可以使用 `assertBatchCount` 方法來斷言已分派了指定數量的批次：

```php
Bus::assertBatchCount(3);
```

您可以使用 `assertNothingBatched` 來斷言沒有分派任何批次：

```php
Bus::assertNothingBatched();
```


<a name="testing-job-batch-interaction"></a>
#### 測試 Job / 批次互動

此外，有時您可能需要測試單一 Job 與其基礎批次之間的互動。例如，您可能需要測試某個 Job 是否取消了該批次後續的處理。若要實現此目的，您需要透過 `withFakeBatch` 方法將一個模擬批次分配給該 Job。`withFakeBatch` 方法會傳回一個包含 Job 實例與模擬批次的元組（Tuple）：

```php
[$job, $batch] = (new ShipOrder)->withFakeBatch();

$job->handle();

$this->assertTrue($batch->cancelled());
$this->assertEmpty($batch->added);
```


<a name="testing-job-queue-interactions"></a>
### 測試 Job / 佇列互動

有時候，您可能需要測試佇列 Job 是否已[將自身釋放回佇列](#manually-releasing-a-job)，或是測試該 Job 是否已刪除自身。您可以透過建立 Job 實例並呼叫 `withFakeQueueInteractions` 方法來測試這些佇列互動。

當 Job 的佇列互動被模擬後，您可以在該 Job 上呼叫 `handle` 方法。呼叫 Job 後，有多種斷言方法可用於驗證 Job 的佇列互動：

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

透過使用 `Queue` [Facade](/docs/{{version}}/facades) 上的 `before` 與 `after` 方法，您可以指定在佇列 Job 處理之前或之後執行的回呼函式。這些回呼函式是進行額外日誌記錄或為儀表板增加統計數據的好時機。通常，您應該在[服務提供者(Service Providers)](/docs/{{version}}/providers) 的 `boot` 方法中呼叫這些方法。例如，我們可以使用 Laravel 內建的 `AppServiceProvider`：

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

透過使用 `Queue` [Facade](/docs/{{version}}/facades) 上的 `looping` 方法，您可以指定在 Worker 嘗試從佇列抓取 Job 之前執行的回呼函式。例如，您可能會註冊一個 Closure 來還原先前失敗 Job 所留下的任何未結束交易：

```php
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Queue;

Queue::looping(function () {
    while (DB::transactionLevel() > 0) {
        DB::rollBack();
    }
});
```

當佇列 Worker 無法從佇列中取得 Job 時，Laravel 也會觸發 `Illuminate\Queue\Events\WorkerIdle` 事件：

```php
use Illuminate\Queue\Events\WorkerIdle;
use Illuminate\Support\Facades\Event;

Event::listen(function (WorkerIdle $event) {
    // $event->connectionName
    // $event->queue
    // $event->workerOptions
});
```