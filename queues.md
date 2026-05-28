# 佇列 (Queues)

- [簡介](#introduction)
    - [連線 vs. 佇列](#connections-vs-queues)
    - [驅動程式說明與事前準備](#driver-prerequisites)
- [建立任務](#creating-jobs)
    - [生成任務類別](#generating-job-classes)
    - [類別結構](#class-structure)
    - [唯一任務](#unique-jobs)
    - [防抖任務](#debounced-jobs)
    - [加密任務](#encrypted-jobs)
- [任務中介層](#job-middleware)
    - [速率限制](#rate-limiting)
    - [防止任務重疊](#preventing-job-overlaps)
    - [節流例外狀況](#throttling-exceptions)
    - [跳過任務](#skipping-jobs)
- [分派任務](#dispatching-jobs)
    - [延遲分派](#delayed-dispatching)
    - [同步分派](#synchronous-dispatching)
    - [在分派前準備任務](#preparing-jobs-before-dispatch)
    - [任務與資料庫交易](#jobs-and-database-transactions)
    - [任務鏈結](#job-chaining)
    - [自訂佇列與連線](#customizing-the-queue-and-connection)
    - [指定最大任務嘗試次數與逾時值](#max-job-attempts-and-timeout)
    - [SQS FIFO 與公平佇列](#sqs-fifo-and-fair-queues)
    - [佇列容錯移轉](#queue-failover)
    - [錯誤處理](#error-handling)
- [任務批次](#job-batching)
    - [定義可批次處理的任務](#defining-batchable-jobs)
    - [分派批次任務](#dispatching-batches)
    - [鏈結與批次](#chains-and-batches)
    - [向批次新增任務](#adding-jobs-to-batches)
    - [檢查批次](#inspecting-batches)
    - [取消批次](#cancelling-batches)
    - [批次失敗](#batch-failures)
    - [修剪批次](#pruning-batches)
    - [將批次儲存在 DynamoDB](#storing-batches-in-dynamodb)
- [佇列化閉包](#queueing-closures)
- [執行佇列工作者](#running-the-queue-worker)
    - [The `queue:work` Command](#the-queue-work-command)
    - [佇列優先權](#queue-priorities)
    - [佇列工作者與部署](#queue-workers-and-deployment)
    - [對工作者訊號做出反應](#reacting-to-worker-signals)
    - [任務到期與逾時](#job-expirations-and-timeouts)
    - [暫停與恢復佇列工作者](#pausing-and-resuming-queue-workers)
- [Supervisor 設定](#supervisor-configuration)
- [處理失敗的任務](#dealing-with-failed-jobs)
    - [任務失敗後的清理](#cleaning-up-after-failed-jobs)
    - [重試失敗的任務](#retrying-failed-jobs)
    - [忽略遺失的模型](#ignoring-missing-models)
    - [修剪失敗的任務](#pruning-failed-jobs)
    - [將失敗的任務儲存在 DynamoDB](#storing-failed-jobs-in-dynamodb)
    - [停用失敗任務的儲存](#disabling-failed-job-storage)
    - [失敗任務事件](#failed-job-events)
- [清除佇列中的任務](#clearing-jobs-from-queues)
- [監控您的佇列](#monitoring-your-queues)
- [測試](#testing)
    - [模擬部分任務](#faking-a-subset-of-jobs)
    - [測試任務鏈結](#testing-job-chains)
    - [測試任務批次](#testing-job-batches)
    - [測試任務與佇列的互動](#testing-job-queue-interactions)
- [任務事件](#job-events)

<a name="introduction"></a>
## 簡介

在建構網頁應用程式時，你可能會遇到一些任務（例如解析和儲存上傳的 CSV 檔案）在一般的網頁請求中需要花費太長時間來執行。幸好，Laravel 允許你輕鬆建立可在背景處理的佇列任務 (Queued jobs)。透過將耗時的任務移至佇列，你的應用程式便能以極快的速度回應網頁請求，並為你的客戶提供更好的使用者體驗。

Laravel 佇列為各種不同的佇列後端提供了一致的佇列 API，例如 [Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io)，甚至是關聯式資料庫。

Laravel 的佇列設定選項儲存在應用程式的 `config/queue.php` 設定檔中。在這個檔案中，你會發現框架內建的每個佇列驅動程式的連線設定，包括 database、[Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 和 [Beanstalkd](https://beanstalkd.github.io/) 驅動程式，以及會立即執行任務的同步 (Synchronous) 驅動程式（用於開發或測試）。另外也包含了一個會直接丟棄佇列任務的 `null` 佇列驅動程式。

> [!NOTE]
> Laravel Horizon 是專為 Redis 驅動的佇列所設計的美觀儀表板與設定系統。請參閱完整的 [Horizon 說明文件](/docs/{{version}}/horizon)以取得更多資訊。


<a name="connections-vs-queues"></a>
### 連線 vs. 佇列

在開始使用 Laravel 佇列之前，了解「連線 (Connections)」與「佇列 (Queues)」之間的區別非常重要。在你的 `config/queue.php` 設定檔中，有一個 `connections` 設定陣列。此選項定義了與後端佇列服務（如 Amazon SQS、Beanstalk 或 Redis）的連線。然而，任何給定的佇列連線都可能有多個「佇列」，這些佇列可以被視為佇列任務的不同堆疊或堆積。

請注意，`queue` 設定檔中的每個連線設定範例都包含一個 `queue` 屬性。這是當任務被發送到給定連線時，預設會被分派到的佇列。換句話說，如果你在分派任務時沒有明確定義應該分派到哪個佇列，該任務將會被放置在連線設定的 `queue` 屬性中所定義的佇列中：

```php
use App\Jobs\ProcessPodcast;

// This job is sent to the default connection's default queue...
ProcessPodcast::dispatch();

// This job is sent to the default connection's "emails" queue...
ProcessPodcast::dispatch()->onQueue('emails');
```

某些應用程式可能永遠不需要將任務推送到多個佇列，而是偏好使用單一簡單的佇列。然而，將任務推送到多個佇列對於希望為任務處理方式進行優先級排序或分類的應用程式特別有用，因為 Laravel 佇列工作者 (Queue worker) 允許你指定應該依優先權處理哪些佇列。例如，如果你將任務推送到 `high` 佇列，你可以執行一個給予它們更高處理優先權的工作者：

```shell
php artisan queue:work --queue=high,default
```


<a name="driver-prerequisites"></a>
### 驅動程式說明與事前準備


<a name="database"></a>
#### 資料庫

為了使用 `database` 佇列驅動程式，你需要一個資料庫資料表來存放任務。通常，這已包含在 Laravel 預設的 `0001_01_01_000002_create_jobs_table.php` [資料庫遷移](/docs/{{version}}/migrations)中；不過，如果你的應用程式不包含此遷移，你可以使用 `make:queue-table` Artisan 指令來建立它：

```shell
php artisan make:queue-table

php artisan migrate
```


<a name="redis"></a>
#### Redis

為了使用 `redis` 佇列驅動程式，你應該在 `config/database.php` 設定檔中設定 Redis 資料庫連線。

> [!WARNING]
> `redis` 佇列驅動程式不支援 `serializer` 和 `compression` 這兩個 Redis 選項。


<a name="redis-cluster"></a>
##### Redis 叢集 (Redis Cluster)

如果你的 Redis 佇列連線使用 [Redis 叢集 (Redis Cluster)](https://redis.io/docs/latest/operate/rs/databases/durability-ha/clustering)，你的佇列名稱必須包含 [key hash tag](https://redis.io/docs/latest/develop/using-commands/keyspace/#hashtags)。這是必要的，以確保給定佇列的所有 Redis 鍵 (Keys) 都被放置在同一個雜湊槽 (Hash slot) 中：

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

使用 Redis 佇列時，你可以使用 `block_for` 設定選項來指定驅動程式在循環工作者迴圈並重新輪詢 Redis 資料庫之前，應該等待任務可用多長時間。

根據你的佇列負載調整此值，會比持續輪詢 Redis 資料庫以取得新任務更有效率。例如，你可以將該值設為 `5`，表示驅動程式在等待任務可用時應該阻塞五秒：

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
> 將 `block_for` 設為 `0` 將導致佇列工作者無限期阻塞，直到任務可用為止。這也會阻止處理如 `SIGTERM` 的訊號，直到下一個任務處理完畢。


<a name="sqs-overflow-storage"></a>
#### SQS 溢位儲存 (SQS Overflow Storage)

Amazon SQS 限制了佇列訊息承載資料 (Payload) 的最大大小。如果你需要分派承載資料可能超過此限制的任務，你可以設定 Laravel 將過大的 SQS 承載資料儲存在快取存放區中，並改為透過 SQS 發送指標 (Pointer)。要啟用此功能，請在你的 SQS 佇列連線設定中新增 `overflow` 陣列：

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

啟用溢位儲存時，Laravel 會將至少 1 MB 的承載資料儲存在設定的快取存放區中。如果 `always` 選項為 `true`，則不論大小，每個 SQS 承載資料都將儲存在快取存放區中。由於佇列任務在處理時需要從快取存放區中檢索其承載資料，因此你應該選擇一個能夠保留承載資料直到你的工作者處理它們的存放區。預設情況下，儲存的承載資料會在任務成功處理並從 SQS 刪除後一併刪除。

如果 `flush_on_clear` 選項為 `true`，當 `queue:clear` 指令清除 SQS 佇列時，已設定的溢位快取存放區將會被排空 (Flushed)。由於排空快取存放區可能會移除該存放區中的所有項目，因此在啟用此選項時，你應該將 SQS 溢位儲存設定為使用專用的快取存放區。


<a name="other-driver-prerequisites"></a>
#### 其他驅動程式事前準備

以下是列出的佇列驅動程式所需的依賴套件。這些依賴套件可以透過 Composer 套件管理器安裝：

<div class="content-list" markdown="1">

- Amazon SQS: `aws/aws-sdk-php ~3.0`
- Beanstalkd: `pda/pheanstalk ~5.0`
- Redis: `predis/predis ~2.0` or phpredis PHP extension
- [MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/queues/): `mongodb/laravel-mongodb`

</div>

<a name="creating-jobs"></a>
## 建立任務

<a name="generating-job-classes"></a>
### 生成任務類別

預設情況下，應用程式中所有可佇列化的任務都儲存在 `app/Jobs` 目錄中。如果 `app/Jobs` 目錄不存在，當您執行 `make:job` Artisan 指令時會自動建立：

```shell
php artisan make:job ProcessPodcast
```

生成的類別將會實作 `Illuminate\Contracts\Queue\ShouldQueue` 介面，這會向 Laravel 指示該任務應該被推送到佇列中並以非同步方式執行。

> [!NOTE]
> 任務 Stub 可以使用 [Stub 發布](/docs/{{version}}/artisan#stub-customization) 進行自訂。

<a name="class-structure"></a>
### 類別結構

任務類別非常簡單，通常只包含一個 `handle` 方法，該方法會在佇列處理任務時被呼叫。首先，讓我們來看一個任務類別的範例。在這個範例中，我們假設我們管理一個 Podcast 發布服務，並且需要在上傳的 Podcast 檔案發布前對其進行處理：

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

在這個範例中，請注意我們可以將 [Eloquent 模型](/docs/{{version}}/eloquent) 直接傳入佇列任務的建構子中。由於該任務使用了 `Queueable` trait，在任務處理時，Eloquent 模型及其已載入的關聯將會被優雅地序列化與反序列化。

如果您的佇列任務在建構子中接受 Eloquent 模型，則只有該模型的識別碼（Identifier）會被序列化到佇列中。當任務實際被處理時，佇列系統會自動從資料庫重新取得完整的模型實例及其已載入的關聯。這種模型序列化的方式可以讓傳送到佇列驅動程式的任務承載資料（Payload）變得更小。

<a name="handle-method-dependency-injection"></a>
#### `handle` 方法的依賴注入

當佇列處理任務時，就會呼叫 `handle` 方法。請注意，我們可以在任務的 `handle` 方法上對依賴關係進行型別提示（Type-hint）。Laravel 的 [服務容器](/docs/{{version}}/container) 會自動注入這些依賴項目。

如果您想完全控制容器如何將依賴項目注入到 `handle` 方法中，可以使用容器的 `bindMethod` 方法。`bindMethod` 方法接受一個回呼（Callback），該回呼會接收任務和容器。在回呼中，您可以自由地以任何您想要的方式呼叫 `handle` 方法。通常，您應該在 `App\Providers\AppServiceProvider` [服務提供者(Service Providers)](/docs/{{version}}/providers) 的 `boot` 方法中呼叫此方法：

```php
use App\Jobs\ProcessPodcast;
use App\Services\AudioProcessor;
use Illuminate\Contracts\Foundation\Application;

$this->app->bindMethod([ProcessPodcast::class, 'handle'], function (ProcessPodcast $job, Application $app) {
    return $job->handle($app->make(AudioProcessor::class));
});
```

> [!WARNING]
> 二進位資料（例如原始圖片內容）在傳遞給佇列任務之前，應該先經過 `base64_encode` 函數處理。否則，該任務在放入佇列時可能無法正確序列化為 JSON。

<a name="handling-relationships"></a>
#### 佇列化關聯

因為當任務進入佇列時，所有已載入的 Eloquent 模型關聯也會被序列化，這有時會使序列化後的任務字串變得非常龐大。此外，當任務被反序列化且模型關聯從資料庫重新取得時，它們將會被完整地載入。在任務佇列化過程中、模型被序列化之前所套用的任何關聯條件限制，在任務反序列化時都不會被套用。因此，如果您只想處理特定關聯的子集，您應該在佇列任務中重新對該關聯進行條件限制。

或者，為了防止關聯被序列化，您可以在設定屬性值時對模型呼叫 `withoutRelations` 方法。此方法將會傳回一個不包含已載入關聯的模型實例：

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

如果您使用的是 [PHP 建構子屬性升級 (Constructor Property Promotion)](https://www.php.net/manual/en/language.oop5.decon.php#language.oop5.decon.constructor.promotion)，且希望指示 Eloquent 模型的關聯不應該被序列化，您可以使用 `WithoutRelations` 屬性（Attribute）：

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

為了方便起見，如果您希望將所有模型在沒有關聯的情況下進行序列化，您可以將 `WithoutRelations` 屬性套用到整個類別，而不是套用到每個模型上：

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

如果任務接收的是 Eloquent 模型的集合（Collection）或陣列，而不是單一模型，那麼在任務反序列化並執行時，該集合內模型的關聯將不會被還原。這是為了防止在處理大量模型的任務中消耗過多的系統資源。

<a name="unique-jobs"></a>
### 唯一任務

> [!WARNING]
> 唯一任務需要支援[鎖定](/docs/{{version}}/cache#atomic-locks)的快取驅動程式。目前，`memcached`、`redis`、`dynamodb`、`database`、`file` 與 `array` 快取驅動程式支援不可分割鎖定。

> [!WARNING]
> 唯一任務的限制不適用於批次 (Batches) 內部的任務。

有時候，您可能希望確保在任何時間點，佇列中都只有一個特定任務的執行個體。您可以透過在任務類別中實作 `ShouldBeUnique` 介面來達到此目的。此介面不需要您在類別中定義任何額外的方法：

```php
<?php

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Queue\ShouldBeUnique;

class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    // ...
}
```

在上述範例中，`UpdateSearchIndex` 任務是唯一的。因此，如果該任務的另一個執行個體已經在佇列中且尚未完成處理，則該任務將不會被分派。

在某些情況下，您可能希望定義一個使任務具有唯一性的特定「鍵 (Key)」，或者您可能希望指定一個逾時時間，超過該時間後任務將不再保持唯一。為了解決這個問題，您可以使用 `UniqueFor` 屬性並在您的任務類別中定義一個 `uniqueId` 方法：

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
在上述範例中，`UpdateSearchIndex` 任務是根據產品 ID 來確保唯一性的。因此，在現有任務完成處理之前，任何使用相同產品 ID 新分派的任務都將被忽略。此外，如果現有任務在一個小時內未被處理，唯一鎖定將被釋放，並且可以向佇列分派另一個具有相同唯一鍵的任務。

> [!WARNING]
> 如果您的應用程式從多個網頁伺服器或容器分派任務，您應該確保所有伺服器都與同一個中央快取伺服器進行通訊，以便 Laravel 能夠精確判斷任務是否唯一。


<a name="keeping-jobs-unique-until-processing-begins"></a>
#### 保持任務唯一直到開始處理

預設情況下，唯一任務會在任務完成處理或所有重試嘗試皆失敗後被「解除鎖定」。然而，在某些情況下，您可能希望任務在開始處理前立即解除鎖定。為了實現這一點，您的任務應該實作 `ShouldBeUniqueUntilProcessing` 契約而非 `ShouldBeUnique` 契約：

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
#### 唯一任務鎖定

在幕後，當分派 `ShouldBeUnique` 任務時，Laravel 會嘗試使用 `uniqueId` 鍵值來獲取[鎖定](/docs/{{version}}/cache#atomic-locks)。如果鎖定已被持有，則不會分派該任務。此鎖定會在任務完成處理或所有重試嘗試皆失敗時釋放。預設情況下，Laravel 將使用預設的快取驅動程式來獲取此鎖定。然而，如果您希望使用另一個驅動程式來獲取鎖定，您可以定義一個 `uniqueVia` 方法，該方法會返回應使用的快取驅動程式：

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
> 如果您只需要限制任務的同時執行，請改用 [WithoutOverlapping](/docs/{{version}}/queues#preventing-job-overlaps) 任務中介層。


<a name="debounced-jobs"></a>
### 防抖任務

有時候，您可能希望確保在短暫的時間內多次分派相同的任務時，只有最後一次分派實際執行。您可以透過在任務中加入 `DebounceFor` 屬性來達到此目的：

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

在上述範例中，在 `30` 秒內重複為同一個產品分派 `UpdateSearchIndex` 將會對該任務進行防抖處理，從而只有最後一次分派才會執行。

如果您想限制被頻繁重新分派的任務最多可以被延遲多久，您可以將 `maxWait` 引數傳遞給 `DebounceFor` 屬性：

```php
#[DebounceFor(30, maxWait: 120)]
class UpdateSearchIndex implements ShouldQueue
{
    use Queueable;

    // ...
}
```

您可以透過在任務中定義 `debounceVia` 方法來自訂用於防抖追蹤的快取儲存：

```php
use Illuminate\Contracts\Cache\Repository;
use Illuminate\Support\Facades\Cache;

public function debounceVia(): Repository
{
    return Cache::driver('redis');
}
```

如果某個防抖任務被較新的分派所取代，Laravel 將會分派 `Illuminate\Queue\Events\JobDebounced` 事件，並將被取代的任務自佇列中移除。

> [!WARNING]
> 防抖任務與唯一任務是互斥的。使用 `DebounceFor` 屬性的任務不應實作 `ShouldBeUnique`。

> [!WARNING]
> 如果您的應用程式從多個網頁伺服器或容器分派防抖任務，您應該確保所有伺服器都與同一個中央快取伺服器進行通訊。


<a name="encrypted-jobs"></a>
### 加密任務

Laravel 允許您透過[加密](/docs/{{version}}/encryption)來確保任務資料的隱私與完整性。要開始使用，只需將 `ShouldBeEncrypted` 介面新增到任務類別即可。一旦將此介面新增到類別中，Laravel 將會在把您的任務推送到佇列之前，自動對其進行加密：

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
## 任務中介層

任務中介層 (Job middleware) 允許您在執行佇列任務時包裝自訂邏輯，減少任務類別本身的樣板程式碼。例如，請參考以下使用 Laravel 的 Redis 速率限制功能，限制每五秒只能處理一個任務的 `handle` method：

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

雖然這段程式碼可以正常運作，但因為混雜了 Redis 速率限制邏輯，使得 `handle` 方法的實作顯得臃腫。此外，我們想要限制速率的其他任務也都必須重複這段速率限制邏輯。與其在 handle 方法中進行速率限制，我們不如定義一個處理速率限制的任務中介層：

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

如您所見，就像[路由中介層](/docs/{{version}}/middleware)一樣，任務中介層接收正在處理的任務，以及一個用來繼續處理該任務的回呼函式 (callback)。

您可以使用 `make:job-middleware` Artisan 命令來生成新的任務中介層類別。建立任務中介層後，可以透過在任務的 `middleware` 方法中將其返回，來將其附加到任務上。這個方法在透過 `make:job` Artisan 命令生成的任務中預設並不存在，因此您需要手動將其新增到您的任務類別中：

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
> 任務中介層也可以分配給[佇列化事件監聽器](/docs/{{version}}/events#queued-event-listeners)、[寄送郵件類別 (mailables)](/docs/{{version}}/mail#queueing-mail)與[通知](/docs/{{version}}/notifications#queueing-notifications)。


<a name="rate-limiting"></a>
### 速率限制

雖然我們剛剛示範了如何編寫自訂的速率限制任務中介層，但 Laravel 實際上已經內建了速率限制中介層，您可以直接使用它來限制任務速率。就像[路由速率限制器](/docs/{{version}}/routing#defining-rate-limiters)一樣，任務速率限制器是使用 `RateLimiter` Facade 的 `for` 方法來定義的。

例如，您可能希望允許使用者每小時備份一次資料，而對尊榮客戶 (premium customers) 則不設此限制。要達成此目的，您可以在 `AppServiceProvider` 的 `boot` 方法中定義一個 `RateLimiter`：

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

在上述範例中，我們定義了每小時的速率限制；不過，您也可以使用 `perMinute` 方法輕鬆定義基於分鐘的速率限制。此外，您可以將任何您想要的數值傳遞給速率限制的 `by` 方法；不過，這個數值最常用於依客戶區分速率限制：

```php
return Limit::perMinute(50)->by($job->user->id);
```

定義好速率限制後，您可以使用 `Illuminate\Queue\Middleware\RateLimited` 中介層將速率限制器附加到您的任務上。每當任務超過速率限制時，該中介層會根據速率限制的持續時間，以適當的延遲將任務釋放回佇列中：

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

將受速率限制的任務釋放回佇列中仍會增加該任務的總嘗試次數（`attempts`）。因此，您可能需要相應地調整任務類別中的 `Tries` 和 `MaxExceptions` 屬性。或者，您也可以使用 [retryUntil 方法](#time-based-attempts)來定義該任務在多久時間內不應再被嘗試。

使用 `releaseAfter` 方法，您也可以指定釋放後的任務在再次嘗試之前必須經過的秒數：

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

如果您不希望任務在受速率限制時被重試，您可以使用 `dontRelease` 方法：

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

如果您使用的是 Redis，您可以使用 `Illuminate\Queue\Middleware\RateLimitedWithRedis` 中介層，該中介層針對 Redis 進行了最佳化，比基礎的速率限制中介層更有效率：

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
### 防止任務重疊

Laravel 包含了一個 `Illuminate\Queue\Middleware\WithoutOverlapping` 中介層，允許您根據任意鍵值來防止任務重疊。當佇列任務正在修改某個一次只能由一個任務修改的資源時，這會非常有用。

例如，假設您有一個更新使用者信用評分的佇列任務，並且您希望防止同一個使用者 ID 的信用評分更新任務重疊。為了實現這一點，您可以從任務的 `middleware` 方法中返回 `WithoutOverlapping` 中介層：

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

將重疊的任務釋放回佇列中仍然會增加該任務的總嘗試次數。您可能需要相應地調整您任務類別上的 `Tries` 與 `MaxExceptions` 屬性。例如，將 `Tries` 保持預設值 1，將會防止任何重疊的任務在稍後進行重試。

任何相同類型的重疊任務都將被釋放回佇列。您還可以指定在再次嘗試執行該已釋放任務之前，必須經過的秒數：

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

如果您希望立即刪除任何重疊的任務，使其不再重試，您可以使用 `dontRelease` 方法：

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

`WithoutOverlapping` 中介層是由 Laravel 的原子鎖（atomic lock）功能所支援。有時候，您的任務可能會因意外失敗或逾時，導致鎖沒有被釋放。因此，您可以使用 `expireAfter` 方法明確定義鎖的過期時間。例如，下方的範例將指示 Laravel 在任務開始處理三分鐘後，釋放 `WithoutOverlapping` 的鎖：

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
> `WithoutOverlapping` 中介層需要支援[鎖(locks)](/docs/{{version}}/cache#atomic-locks)的快取驅動程式。目前，`memcached`、`redis`、`dynamodb`、`database`、`file` 以及 `array` 快取驅動程式皆支援原子鎖。

<a name="sharing-lock-keys"></a>
#### 跨任務類別共享鎖的鍵值

預設情況下，`WithoutOverlapping` 中介層只會防止相同類別的任務重疊。因此，即使兩個不同的任務類別使用相同的鎖鍵值，它們也不會被阻止重疊。然而，您可以使用 `shared` 方法指示 Laravel 將此鍵值應用於跨任務類別：

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

Laravel 包含了一個 `Illuminate\Queue\Middleware\ThrottlesExceptions` 中介層，可讓您節流例外狀況。一旦任務拋出指定數量的例外狀況，所有後續執行該任務的嘗試都將被延遲，直到指定的間隔時間過後為止。此中介層對於需要與不穩定第三方服務互動的任務特別有用。

例如，假設有一個與第三方 API 互動的佇列化任務開始拋出例外狀況。若要節流例外狀況，您可以在任務的 `middleware` 方法中返回 `ThrottlesExceptions` 中介層。通常，此中介層應該與實作了[基於時間的嘗試](#time-based-attempts)的任務搭配使用：

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

該中介層接受的第一個建構子引數是任務在被節流前可以拋出的例外狀況數量，而第二個建構子引數則是任務被節流後，需要經過多少秒才能再次嘗試。在上述程式碼範例中，如果任務連續拋出 10 次例外狀況，我們將等待 5 分鐘後才再次嘗試該任務，並受限於 30 分鐘的時間限制。

當任務拋出例外狀況但尚未達到例外狀況閾值時，該任務通常會立即重試。然而，您可以在將中介層附加到任務時，透過呼叫 `backoff` 方法來指定此類任務應延遲的分鐘數：

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

而 `backoff` 方法也接受一個接收所拋出例外狀況的閉包，允許動態決定延遲時間：

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

在內部，此中介層使用 Laravel 的快取系統來實作速率限制，並以任務的類別名稱作為快取的「鍵值 (key)」。您可以在將中介層附加到任務時，透過呼叫 `by` 方法來覆寫此鍵值。如果您有多個任務與同一個第三方服務進行互動，且希望它們共享同一個節流「桶子 (bucket)」以確保它們遵循單一共享限制時，這會非常有用：

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

預設情況下，此中介層會對所有例外狀況進行節流。您可以在將中介層附加到任務時，透過呼叫 `when` 方法來修改此行為。如此一來，只有在提供給 `when` 方法的閉包返回 `true` 時，該例外狀況才會被節流：

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

與將任務釋放回佇列或拋出例外狀況的 `when` 方法不同， `deleteWhen` 方法允許您在發生特定例外狀況時完全刪除該任務：

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

如果您希望將被節流的例外狀況回報給應用程式的例外狀況處理常式 (Exception handler)，可以在將中介層附加到任務時，透過呼叫 `report` 方法來實現。您也可以選擇提供一個閉包給 `report` 方法，如此一來，只有在該閉包返回 `true` 時才會回報例外狀況：

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

如果您使用的是 Redis，可以使用 `Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis` 中介層，該中介層針對 Redis 進行了最佳化，比基本的例外狀況節流中介層更有效率：

```php
use Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis;

public function middleware(): array
{
    return [new ThrottlesExceptionsWithRedis(10, 10 * 60)];
}
```

可以使用 `connection` 方法來指定中介層應該使用哪個 Redis 連線：

```php
return [(new ThrottlesExceptionsWithRedis(10, 10 * 60))->connection('limiter')];
```

<a name="skipping-jobs"></a>
### 跳過任務

`Skip` 中介層允許您指定應跳過或刪除任務，而無需修改任務本身的邏輯。如果指定的條件評估為 `true`，`Skip::when` 方法將會刪除該任務；而如果條件評估為 `false`，`Skip::unless` 方法則會刪除該任務：

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

您也可以將一個 `Closure`（閉包）傳遞給 `when` 與 `unless` 方法，以進行更複雜的條件評估：

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
## 分派任務

一旦撰寫好您的任務類別，您就可以使用任務本身的 `dispatch` 方法來分派它。傳遞給 `dispatch` 方法的引數將會被傳入該任務的建構子：

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

如果您想依據條件來分派任務，可以使用 `dispatchIf` 和 `dispatchUnless` 方法：

```php
ProcessPodcast::dispatchIf($accountActive, $podcast);

ProcessPodcast::dispatchUnless($accountSuspended, $podcast);
```

在新的 Laravel 應用程式中，`database` 連線被定義為預設的佇列。您可以透過修改應用程式 `.env` 檔案中的 `QUEUE_CONNECTION` 環境變數來指定不同的預設佇列連線。


<a name="delayed-dispatching"></a>
### 延遲分派

如果您想要指定某個任務不應立即讓佇列工作者處理，可以在分派任務時使用 `delay` 方法。例如，讓我們指定一個任務在分派後的 10 分鐘內都無法被處理：

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

在某些情況下，任務可能會設定預設的延遲。如果您需要繞過這個延遲並立即處理該任務，可以使用 `withoutDelay` 方法：

```php
ProcessPodcast::dispatch($podcast)->withoutDelay();
```

> [!WARNING]
> Amazon SQS 佇列服務的最大延遲時間為 15 分鐘。


<a name="synchronous-dispatching"></a>
### 同步分派

如果您想要立即（同步）分派任務，可以使用 `dispatchSync` 方法。使用此方法時，任務將不會被放入佇列，而是會在目前的行程中立即執行：

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

使用延遲同步分派，您可以將任務分派在目前的行程中處理，但在 HTTP 回應發送給使用者之後才執行。這讓您可以同步處理「佇列化」的任務，而不會降低使用者在使用應用程式時的體驗。若要延遲同步任務的執行，請將任務分派至 `deferred` 連線：

```php
RecordDelivery::dispatch($order)->onConnection('deferred');
```

`deferred` 連線也作為預設的 [佇列容錯移轉](#queue-failover)。

同樣地，`background` 連線會在 HTTP 回應發送給使用者之後處理任務；然而，該任務是在獨立衍生的 PHP 行程中處理的，這能讓 PHP-FPM / 應用程式工作者空出資源來處理其他傳入的 HTTP 請求：

```php
RecordDelivery::dispatch($order)->onConnection('background');
```


<a name="preparing-jobs-before-dispatch"></a>
### 在分派前準備任務

如果某個任務在被推送到佇列之前需要準備或檢查其狀態，該任務可以實作 `Illuminate\Contracts\Queue\PreparesForDispatch` 介面。Laravel 會在分派任務之前調用該任務的 `prepareForDispatch` 方法。如果此方法回傳 `false`，則該任務將不會被分派：

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
### 任務與資料庫交易

雖然在資料庫交易中分派任務完全沒有問題，但您必須特別注意，以確保您的任務實際上能夠成功執行。在交易內分派任務時，該任務有可能在父交易提交（Commit）之前就被工作者處理。當這種情況發生時，您在資料庫交易期間對模型或資料庫紀錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫紀錄可能也尚未存在於資料庫中。

幸運的是，Laravel 提供了幾種方法來解決這個問題。首先，您可以在佇列連線的設定陣列中將 `after_commit` 連線選項設為 true：

```php
'redis' => [
    'driver' => 'redis',
    // ...
    'after_commit' => true,
],
```

當 `after_commit` 選項為 `true` 時，您可以在資料庫交易中分派任務；然而，Laravel 會等到開啟的父資料庫交易提交後，才會實際分派該任務。當然，如果目前沒有開啟任何資料庫交易，任務將會立即分派。

如果交易因交易期間發生的例外狀況而回復（Rollback），則在該交易期間分派的任務將會被丟棄。

> [!NOTE]
> 將 `after_commit` 設定選項設為 `true` 也會導致任何佇列化事件監聽器、Mailable、通知及廣播事件，在所有開啟的資料庫交易提交之後才被分派。


<a name="specifying-commit-dispatch-behavior-inline"></a>
#### 線上指定提交分派行為

如果您未將 `after_commit` 佇列連線設定選項設為 `true`，您仍然可以指示特定任務在所有開啟的資料庫交易都提交後才被分派。若要達成此目的，您可以將 `afterCommit` 方法鏈結至您的分派操作上：

```php
use App\Jobs\ProcessPodcast;

ProcessPodcast::dispatch($podcast)->afterCommit();
```

同樣地，如果 `after_commit` 設定選項被設為 `true`，您可以指示特定任務應立即分派，而不需要等待任何開啟的資料庫交易提交：

```php
ProcessPodcast::dispatch($podcast)->beforeCommit();
```

<a name="job-chaining"></a>
### 任務鏈結

任務鏈結 (Job chaining) 允許你指定一連串的佇列任務，在主要任務成功執行後依序執行。如果序列中的某個任務失敗，剩餘的任務將不會被執行。若要執行佇列任務鏈，你可以使用 `Bus` Facade 提供的 `chain` 方法。Laravel 的 command bus 是一個底層組件，佇列任務的分派功能就是建立在它之上的：

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

除了鏈結任務類別的實例之外，你也可以鏈結閉包：

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
> 在任務內使用 `$this->delete()` 方法刪除任務並不會阻止鏈結任務被處理。只有在鏈結中的某個任務失敗時，該鏈結才會停止執行。


<a name="chain-connection-queue"></a>
#### 鏈結的連線與佇列

如果你想指定鏈結任務所使用的連線與佇列，可以使用 `onConnection` 和 `onQueue` 方法。除非佇列任務被明確指派不同的連線或佇列，否則這些方法會指定該使用的佇列連線與佇列名稱：

```php
Bus::chain([
    new ProcessPodcast,
    new OptimizePodcast,
    new ReleasePodcast,
])->onConnection('redis')->onQueue('podcasts')->dispatch();
```


<a name="adding-jobs-to-the-chain"></a>
#### 向鏈結中新增任務

有時候，你可能需要從鏈結中的某個任務內部，向現有的任務鏈結最前面或最後面追加任務。你可以使用 `prependToChain` 和 `appendToChain` 方法來達成此目的：

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

鏈結任務時，你可以使用 `catch` 方法來指定一個閉包，當鏈結中的某個任務失敗時就會調用該閉包。此回呼將會接收導致任務失敗的 `Throwable` 實例：

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
> 由於鏈結的回呼會被序列化並在稍後由 Laravel 佇列執行，因此你不應該在鏈結回呼中使用 `$this` 變數。


<a name="customizing-the-queue-and-connection"></a>
### 自訂佇列與連線


<a name="dispatching-to-a-particular-queue"></a>
#### 分派到特定佇列

透過將任務推送到不同的佇列，你可以對佇列任務進行「分類」，甚至可以為不同的佇列指派不同數量的處理程序 (Worker) 來排定優先權。請記住，這並不是將任務推送到佇列設定檔中定義的不同佇列「連線」，而僅僅是推送到單一連線中的特定佇列。若要指定佇列，請在分派任務時使用 `onQueue` 方法：

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

或者，你也可以在任務的建構子中調用 `onQueue` 方法來指定任務的佇列：

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

如果你的應用程式與多個佇列連線進行互動，你可以使用 `onConnection` 方法來指定要將任務推送到哪個連線：

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

你可以將 `onConnection` 和 `onQueue` 方法鏈結在一起，以指定任務的連線與佇列：

```php
ProcessPodcast::dispatch($podcast)
    ->onConnection('sqs')
    ->onQueue('processing');
```

或者，你也可以在任務的建構子中調用 `onConnection` 方法來指定任務的連線：

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

你可以使用 `Queue` Facade 的 `route` 方法來為特定的任務類別定義預設的連線與佇列。這在你想要確保某些任務始終使用特定佇列，而不需要在任務本身指定連線或佇列時非常有用。

除了對特定的任務類別進行路由外，你也可以將介面、特徵 (Trait) 或父類別傳遞給 `route` 方法。當你這樣做時，任何實作該介面、使用該特徵或繼承該父類別的任務，都將自動使用所設定的連線與佇列。

通常，你應該在服務提供者 (Service Provider) 的 `boot` 方法中調用 `route` 方法：

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

當指定了連線但沒有指定佇列時，任務將會被發送到預設佇列：

```php
Queue::route(ProcessPodcast::class, connection: 'redis');
```

你也可以藉由向 `route` 方法傳遞一個陣列，一次對多個任務類別進行路由：

```php
Queue::route([
    ProcessPodcast::class => ['podcasts', 'redis'], // Queue and connection
    ProcessVideo::class => 'videos', // Queue only (uses default connection)
]);
```

> [!NOTE]
> 佇列路由仍然可以在單一任務中被覆寫。

<a name="max-job-attempts-and-timeout"></a>
### 指定最大任務嘗試次數與逾時值

<a name="max-attempts"></a>
#### 最大嘗試次數

任務嘗試次數是 Laravel 佇列系統的核心概念，並驅動了許多進階功能。雖然它們起初可能看起來令人困惑，但在修改預設設定之前，理解它們的工作原理非常重要。

當任務被分派時，它會被推送到佇列中。接著，工作者會取出它並嘗試執行它。這就是一次任務嘗試。

然而，一次嘗試並不一定意味著任務的 `handle` 方法已被執行。嘗試次數也可能透過以下幾種方式被「消耗」：

<div class="content-list" markdown="1">

- 任務在執行期間遇到未處理的例外狀況。
- 任務使用 `$this->release()` 手動釋放回佇列。
- 中介層（例如 `WithoutOverlapping` 或 `RateLimited`）未能取得鎖定並釋放了該任務。
- 任務逾時。
- 任務的 `handle` 方法執行並完成，且未拋出例外狀況。

</div>

您可能不想無限制地一直嘗試執行任務。因此，Laravel 提供了多種方式來指定任務可以嘗試的次數或時間。

> [!NOTE]
> 預設情況下，Laravel 只會嘗試執行任務一次。如果您的任務使用像是 `WithoutOverlapping` 或 `RateLimited` 的中介層，或者您手動釋放任務，您可能需要透過 `tries` 選項來增加允許的嘗試次數。

指定任務最大嘗試次數的一種方法是透過 Artisan 命令列上的 `--tries` 選項。這將適用於該工作者處理的所有任務，除非正在處理的任務本身有指定其可嘗試的次數：

```shell
php artisan queue:work --tries=3
```

如果任務超過其最大嘗試次數，它將被視為「失敗」的任務。有關處理失敗任務的更多資訊，請參閱[處理失敗的任務文件](#dealing-with-failed-jobs)。如果向 `queue:work` 命令提供了 `--tries=0`，任務將會無限期重試。

您可以採取更細粒度的方法，使用 `Tries` 屬性直接在任務類別本身定義任務的最大嘗試次數。如果在任務上指定了最大嘗試次數，它將優先於命令列上提供的 `--tries` 值：

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

If you need dynamic control over a particular job's maximum attempts, you may define a `tries` method on the job:

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

除了定義任務在失敗前可以嘗試多少次之外，您也可以定義一個時間點，超過該時間後就不再嘗試執行該任務。這允許任務在給定的時間範圍內進行任意次數的嘗試。要定義不再嘗試任務的時間，請在您的任務類別中新增 `retryUntil` 方法。此方法應返回一個 `DateTime` 實例：

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
> 您也可以在 [佇列化事件監聽器](/docs/{{version}}/events#queued-event-listeners) 與 [佇列化通知](/docs/{{version}}/notifications#queueing-notifications) 上定義 `Tries` 屬性或 `retryUntil` 方法。

<a name="max-exceptions"></a>
#### 最大例外狀況次數

有時您可能希望指定任務可以嘗試多次，但如果重試是由特定次數的未處理例外狀況所觸發（而不是直接由 `release` 方法釋放），則該任務應直接宣告失敗。若要實現此目的，您可以在您的任務類別上使用 `Tries` 與 `MaxExceptions` 屬性：

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

在此範例中，如果應用程式無法取得 Redis 鎖定，任務將會被釋放十秒，並將繼續重試最多 25 次。然而，如果任務拋出了三次未處理的例外狀況，該任務將會失敗。

<a name="timeout"></a>
#### 逾時

通常，您大約知道您預期佇列任務需要執行多久。因此，Laravel 允許您指定一個「逾時 (timeout)」值。預設情況下，逾時值為 60 秒。如果任務的處理時間超過逾時值指定的秒數，處理該任務的工作者將會因錯誤而結束。通常，工作者會由您[伺服器上設定的行程管理器](#supervisor-configuration)自動重啟。

任務可以執行的最大秒數可以使用 Artisan 命令列上的 `--timeout` 選項來指定：

```shell
php artisan queue:work --timeout=30
```

如果任務因持續逾時而超過其最大嘗試次數，它將被標記為失敗。

您也可以使用任務類別上的 `Timeout` 屬性來定義允許任務執行的最大秒數。如果在任務上指定了逾時時間，它將優先於命令列上指定的任何逾時時間：

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

有時，I/O 阻塞的程序（例如 socket 或連外 HTTP 連線）可能不會遵守您指定的逾時時間。因此，在使用這些功能時，您應該始終嘗試同時使用它們的 API 來指定逾時時間。例如，使用 [Guzzle](https://docs.guzzlephp.org) 時，您應該始終指定連線和請求的逾時值。

> [!WARNING]
> 必須安裝 [PCNTL](https://www.php.net/manual/en/book.pcntl.php) PHP 擴充功能才能指定任務逾時。此外，任務的「逾時 (timeout)」值應始終小於其 [「重新嘗試時間 (retry_after)」](#job-expiration) 值。否則，任務可能會在實際完成執行或逾時之前就被重新嘗試執行。

<a name="failing-on-timeout"></a>
#### 逾時時判定為失敗

如果您想指出任務在逾時時應被標記為[失敗](#dealing-with-failed-jobs)，您可以在任務類別上使用 `FailOnTimeout` 屬性：

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
> 預設情況下，當任務逾時時，它會消耗一次嘗試次數，並被釋放回佇列中（如果允許重試）。然而，如果您將任務設定為逾時時失敗，則無論 tries 的值設定為何，它都不會被重試。

<a name="sqs-fifo-and-fair-queues"></a>
### SQS FIFO 與公平佇列

Laravel 支援 [Amazon SQS FIFO (First-In-First-Out)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html) 與 [公平 (fair)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fair-queues.html) 佇列。FIFO 佇列允許你按照發送的確切順序處理任務，同時透過訊息去重 (message deduplication) 確保僅處理一次。

FIFO 佇列需要訊息群組 ID 來決定哪些任務可以平行處理。具有相同群組 ID 的任務會按順序處理，而具有不同群組 ID 的訊息則可以同時處理。

Laravel 提供了一個流暢的 `onGroup` 方法，用於在分派任務時指定訊息群組 ID：

```php
ProcessOrder::dispatch($order)
    ->onGroup("customer-{$order->customer_id}");
```

SQS FIFO 佇列支援訊息去重，以確保僅處理一次。請在你的任務類別中實作 `deduplicationId` 方法，以提供自訂的去重 ID：

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

如果你使用的是 SQS 標準佇列，設定訊息群組將啟用公平佇列。換句話說，一旦你指定了群組，SQS 就會使用它們來維持多租戶或工作負載之間的公平傳遞。不需要額外的 Laravel 設定。

除了在分派時呼叫 `onGroup` 之外，你也可以直接在任務上定義 `messageGroup` 方法：

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
#### FIFO 監聽器、郵件與通知

使用 FIFO 佇列時，你還需要為監聽器、郵件和通知定義訊息群組。或者，你也可以將這些物件的佇列化實例分派到非 FIFO 佇列。

要為 [佇列事件監聽器](/docs/{{version}}/events#queued-event-listeners) 定義訊息群組，請在監聽器上定義 `messageGroup` 方法。你也可以選擇定義 `deduplicationId` 方法：

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

當發送即將在 FIFO 佇列排程的 [郵件訊息](/docs/{{version}}/mail) 時，你應該在發送通知時呼叫 `onGroup` 方法以及選用的 `withDeduplicator` 方法：

```php
use App\Mail\InvoicePaid;
use Illuminate\Support\Facades\Mail;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

Mail::to($request->user())->send($invoicePaid);
```

當發送即將在 FIFO 佇列排程的 [通知](/docs/{{version}}/notifications) 時，你應該在發送通知時呼叫 `onGroup` 方法以及選用的 `withDeduplicator` 方法：

```php
use App\Notifications\InvoicePaid;

$invoicePaid = (new InvoicePaid($invoice))
    ->onGroup('invoices')
    ->withDeduplicator(fn () => 'invoices-'.$invoice->id);

$user->notify($invoicePaid);
```


<a name="queue-failover"></a>
### 佇列容錯移轉

`failover` 佇列驅動程式在將任務推送到佇列時提供自動容錯移轉功能。如果 `failover` 設定中的主要佇列連線因任何原因失敗，Laravel 將會自動嘗試將任務推送到清單中下一個設定的連線。這在確保正式環境中的高可用性特別有用，因為在這些環境中佇列的可靠性至關重要。

若要設定容錯移轉佇列連線，請指定 `failover` 驅動程式，並提供要依序嘗試的連線名稱陣列。預設情況下，Laravel 已在應用程式的 `config/queue.php` 設定檔中包含了一個範例容錯移轉設定：

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

設定好使用 `failover` 驅動程式的連線後，你需要將應用程式 `.env` 檔案中的預設佇列連線設為該容錯移轉連線，以便使用容錯移轉功能：

```ini
QUEUE_CONNECTION=failover
```

Next, start at least one worker for each connection in your failover connection list:

```bash
php artisan queue:work redis
php artisan queue:work database
```

> [!NOTE]
> 你不需要為使用 `sync`、`background` 或 `deferred` 佇列驅動程式的連線執行工作者，因為這些驅動程式會在當前的 PHP 行程中處理任務。

當佇列連線操作失敗且啟動容錯移轉時，Laravel 會分派 `Illuminate\Queue\Events\QueueFailedOver` 事件，讓你可以回報或記錄佇列連線已失敗。

> [!NOTE]
> 如果你使用 Laravel Horizon，請記住 Horizon 僅管理 Redis 佇列。如果你的容錯移轉清單包含 `database`，你應該在執行 Horizon 的同時，執行一般的 `php artisan queue:work database` 行程。

<a name="error-handling"></a>
### 錯誤處理

如果在處理任務時拋出了例外狀況，該任務將會自動被釋放回佇列中，以便再次嘗試。該任務將會持續被釋放，直到達到您的應用程式所允許的最大嘗試次數為止。最大嘗試次數是由 `queue:work` Artisan 命令所使用的 `--tries` 選項來定義的。或者，也可以在任務類別本身定義最大嘗試次數。有關執行佇列工作者的更多資訊[可以在下方找到](#running-the-queue-worker)。


<a name="manually-releasing-a-job"></a>
#### 手動釋放任務

有時候您可能會希望手動將任務釋放回佇列，以便稍後再次嘗試。您可以透過呼叫 `release` 方法來達成此目的：

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

預設情況下，`release` 方法會將任務釋放回佇列以進行立即處理。然而，您可以透過傳遞一個整數或日期實例給 `release` 方法，來指示佇列在經過指定的秒數之前不要處理該任務：

```php
$this->release(10);

$this->release(now()->plus(seconds: 10));
```


<a name="manually-failing-a-job"></a>
#### 手動將任務標記為失敗

有時候您可能需要手動將任務標記為「失敗」。若要這樣做，您可以呼叫 `fail` 方法：

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

如果您想因為捕獲的例外狀況而將任務標記為失敗，可以將該例外狀況傳遞給 `fail` 方法。或者，為了方便起見，您也可以傳遞一個字串錯誤訊息，系統會自動為您將其轉換為例外狀況：

```php
$this->fail($exception);

$this->fail('Something went wrong.');
```

> [!NOTE]
> 如需更多關於失敗任務的資訊，請參考[處理失敗任務的說明文件](#dealing-with-failed-jobs)。


<a name="fail-jobs-on-exceptions"></a>
#### 在特定例外狀況下使任務失敗

`FailOnException` [任務中介層](#job-middleware)允許您在拋出特定例外狀況時直接中斷重試。這讓您可以在遇到暫時性的例外狀況（例如外部 API 錯誤）時進行重試，但在遇到持續性的例外狀況（例如使用者的權限被撤銷）時永久使任務失敗：

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
## 任務批次

Laravel 的任務批次功能讓您能輕鬆地平行執行一組任務，並在該批次任務執行完成後執行特定動作。

在開始之前，您應該建立一個資料庫遷移來建立一個資料表，該資料表將包含有關任務批次的元數據 (Meta Information)，例如其完成百分比。可以使用 `make:queue-batches-table` Artisan 指令來生成此遷移：

```shell
php artisan make:queue-batches-table

php artisan migrate
```


<a name="defining-batchable-jobs"></a>
### 定義可批次處理的任務

要定義可批次的任務，您應該像平常一樣[建立可佇列的任務](#creating-jobs)；然而，您必須將 `Illuminate\Bus\Batchable` trait 新增至該任務類別中。此 trait 提供了 `batch` 方法的存取權限，可用於取得該任務目前正在其中執行的批次：

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
### 分派批次任務

要分派批次任務，您應該使用 `Bus` facade 的 `batch` 方法。當然，批次處理在與完成回呼 (Callback) 結合使用時最為有用。因此，您可以使用 `then`、`catch` 和 `finally` 方法來定義批次的完成回呼。這些回呼在被叫用時，各自都會接收一個 `Illuminate\Bus\Batch` 實例。

當執行多個佇列工作者時，批次中的任務將會平行處理。因此，任務完成的順序可能與它們被新增到批次中的順序不同。請參閱我們關於[鏈結與批次](#chains-and-batches)的說明文件，以瞭解如何依序執行一系列任務。

在此範例中，我們假設我們正在將一個批次任務排入佇列，其中每個任務分別處理 CSV 檔案中指定數量的資料列：

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

在分派批次之後，可以使用 `$batch->id` 屬性取得該批次的 ID，並用來[查詢 Laravel 命令匯流排](#inspecting-batches)以獲取有關該批次的資訊。

> [!WARNING]
> 由於批次回呼會被序列化，並在稍後由 Laravel 佇列執行，因此您不應在回呼中使用 `$this` 變數。此外，由於批次任務會被封裝在資料庫交易中，因此不應在任務中執行會觸發隱式提交 (Implicit Commit) 的資料庫語句。


<a name="naming-batches"></a>
#### 命名批次

如果為批次命名，某些工具（例如 [Laravel Horizon](/docs/{{version}}/horizon) 和 [Laravel Telescope](/docs/{{version}}/telescope)）可以為批次提供更易讀的偵錯資訊。若要為批次指定任意名稱，您可以在定義批次時呼叫 `name` 方法：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import CSV')->dispatch();
```


<a name="batch-connection-queue"></a>
#### 批次連線與佇列

如果您想指定批次任務應使用的連線與佇列，可以使用 `onConnection` 和 `onQueue` 方法。所有批次任務必須在同一個連線和佇列中執行：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->onConnection('redis')->onQueue('imports')->dispatch();
```


<a name="chains-and-batches"></a>
### 鏈結與批次

您可以透過將鏈結任務放在陣列中，在一個批次中定義一組[鏈結任務](#job-chaining)。例如，我們可以平行執行兩個任務鏈結，並在兩個任務鏈結都處理完成時執行回呼：

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

相反地，您也可以透過在鏈結中定義批次，在[鏈結](#job-chaining)中執行多個批次任務。例如，您可以先執行一個批次任務來發布多個 Podcast，接著再執行另一個批次任務來發送發布通知：

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
### 向批次新增任務

有時候，從批次任務內部向該批次新增額外的任務會非常有用。當您需要批次處理數千個任務時，這種模式會很有幫助，因為在單次 Web 請求期間分派這麼多任務可能會花費太長時間。因此，相反地，您可能希望分派一個初始的「載入器 (Loader)」任務批次，由它來將更多任務注入 (Hydrate) 到該批次中：

```php
$batch = Bus::batch([
    new LoadImportBatch,
    new LoadImportBatch,
    new LoadImportBatch,
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->name('Import Contacts')->dispatch();
```

在此範例中，我們將使用 `LoadImportBatch` 任務來將額外任務注入批次中。為了實現這一點，我們可以使用任務的 `batch` 方法所取得的批次實例上的 `add` 方法：

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
> 您只能在屬於同一個批次的任務內部，向該批次新增任務。


<a name="inspecting-batches"></a>
### 檢查批次

提供給批次完成回呼的 `Illuminate\Bus\Batch` 實例擁有多種屬性與方法，可協助您與指定的任務批次進行互動與檢查：

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

所有 `Illuminate\Bus\Batch` 實例都可以被序列化為 JSON，這代表您可以直接從應用程式的路由中返回它們，以取得包含該批次資訊（包括其完成進度）的 JSON 承載資料 (Payload)。這使得在應用程式的使用者介面 (UI) 中顯示批次完成進度變得非常便利。

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

有時候您可能需要取消特定批次的執行。這可以藉由呼叫 `Illuminate\Bus\Batch` 實例上的 `cancel` 方法來達成：

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

如同您在先前的範例中所注意到的，批次處理的任務通常應該在繼續執行之前，先判斷其對應的批次是否已被取消。然而，為了方便起見，您也可以將 `SkipIfBatchCancelled` [中介層](#job-middleware)指派給該任務。顧名思義，此中介層會指示 Laravel 在其對應的批次已被取消時，不要處理該任務：

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

當批次任務失敗時，系統會調用 `catch` 回呼（若有指派）。此回呼僅會針對該批次中第一個失敗的任務進行調用。

<a name="allowing-failures"></a>
#### 允許失敗

當批次中的某個任務失敗時，Laravel 會自動將該批次標記為「已取消 (cancelled)」。如果您希望，可以停用此行為，讓個別任務的失敗不會自動將整批任務標記為取消。這可以藉由在分派批次時呼叫 `allowFailures` 方法來達成：

```php
$batch = Bus::batch([
    // ...
])->then(function (Batch $batch) {
    // All jobs completed successfully...
})->allowFailures()->dispatch();
```

您也可以選擇向 `allowFailures` 方法傳遞一個閉包，該閉包將在每次任務失敗時執行：

```php
$batch = Bus::batch([
    // ...
])->allowFailures(function (Batch $batch, $exception) {
    // Handle individual job failures...
})->dispatch();
```

<a name="retrying-failed-batch-jobs"></a>
#### 重試失敗的批次任務

為了方便起見，Laravel 提供了 `queue:retry-batch` Artisan 命令，讓您可以輕鬆地重試指定批次中的所有失敗任務。此命令接受要重試其失敗任務的批次 UUID：

```shell
php artisan queue:retry-batch 32dbc76c-4f82-4749-b610-a639fe0099b5
```

<a name="pruning-batches"></a>
### 修剪批次

若不進行修剪，`job_batches` 資料表會非常快速地累積紀錄。為了減輕此問題，您應該[排程](/docs/{{version}}/scheduling)讓 `queue:prune-batches` Artisan 命令每天執行：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches')->daily();
```

預設情況下，所有已完成且超過 24 小時的批次都將被修剪。您可以在呼叫該命令時使用 `hours` 選項，來決定要保留批次資料多久。例如，以下命令將刪除所有在 48 小時前就已完成的批次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48')->daily();
```

有時候，您的 `job_batches` 資料表可能會累積那些從未成功完成的批次紀錄，例如其中某個任務失敗且從未成功重試的批次。您可以使用 `unfinished` 選項來指示 `queue:prune-batches` 命令修剪這些未完成的批次紀錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --unfinished=72')->daily();
```

同樣地，您的 `job_batches` 資料表也可能累積已取消批次的批次紀錄。您可以使用 `cancelled` 選項來指示 `queue:prune-batches` 命令修剪這些已取消的批次紀錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-batches --hours=48 --cancelled=72')->daily();
```

<a name="storing-batches-in-dynamodb"></a>
### 將批次儲存在 DynamoDB

Laravel 也支援將批次的中繼資訊 (Meta Information) 儲存在 [DynamoDB](https://aws.amazon.com/dynamodb)，而不是關聯式資料庫。但是，您需要手動建立一個 DynamoDB 資料表來儲存所有的批次紀錄。

通常，此資料表應命名為 `job_batches`，但您應該根據應用程式的 `queue` 設定檔中 `queue.batching.table` 的設定值來命名該資料表。

<a name="dynamodb-batch-table-configuration"></a>
#### DynamoDB 批次資料表設定

`job_batches` 資料表應該有一個名為 `application` 的字串型別主分割鍵 (Primary Partition Key)，以及一個名為 `id` 的字串型別主排序鍵 (Primary Sort Key)。鍵值的 `application` 部分將包含您的應用程式名稱，這是在應用程式的 `app` 設定檔中由 `name` 設定值所定義的。由於應用程式名稱是 DynamoDB 資料表鍵的一部分，因此您可以使用同一個資料表來儲存多個 Laravel 應用程式的任務批次。

此外，如果您想利用 [自動批次修剪](#pruning-batches-in-dynamodb) 功能，可以為您的資料表定義 `ttl` 屬性。

<a name="dynamodb-configuration"></a>
#### DynamoDB 設定

接下來，安裝 AWS SDK，以便您的 Laravel 應用程式可以與 Amazon DynamoDB 進行通訊：

```shell
composer require aws/aws-sdk-php
```

然後，將 `queue.batching.driver` 設定選項的值設為 `dynamodb`。此外，您應該在 `batching` 設定陣列中定義 `key`、`secret` 與 `region` 設定選項。這些選項將用於向 AWS 進行認證。當使用 `dynamodb` 驅動程式時，就不需要 `queue.batching.database` 設定選項：

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

當使用 [DynamoDB](https://aws.amazon.com/dynamodb) 來儲存任務批次資訊時，用於修剪關聯式資料庫中所儲存批次的典型修剪命令將無法運作。相反地，您可以利用 [DynamoDB 的原生 TTL 功能](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html) 來自動移除舊批次的紀錄。

如果您為 DynamoDB 資料表定義了 `ttl` 屬性，則可以定義設定參數來指示 Laravel 如何修剪批次紀錄。`queue.batching.ttl_attribute` 設定值定義了存放 TTL 的屬性名稱，而 `queue.batching.ttl` 設定值則定義了在最後一次更新紀錄後，經過多少秒就可以將批次紀錄從 DynamoDB 資料表中移除：

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

除了將任務類別分派到佇列之外，您也可以分派閉包。這非常適合用於需要在目前請求週期之外執行的快速、簡單任務。當將閉包分派到佇列時，閉包的程式碼內容會經過加密簽章，以確保其在傳輸過程中不會被修改：

```php
use App\Models\Podcast;

$podcast = Podcast::find(1);

dispatch(function () use ($podcast) {
    $podcast->publish();
});
```

若要為佇列化閉包指定一個名稱，以便讓佇列報告儀表板使用以及在 `queue:work` 指令中顯示，您可以使用 `name` 方法：

```php
dispatch(function () {
    // ...
})->name('Publish Podcast');
```

使用 `catch` 方法，您可以提供一個閉包，當佇列化閉包在耗盡您佇列所有[設定的重試嘗試次數](#max-job-attempts-and-timeout)後仍無法成功完成時，就會執行該閉包：

```php
use Throwable;

dispatch(function () use ($podcast) {
    $podcast->publish();
})->catch(function (Throwable $e) {
    // This job has failed...
});
```

> [!WARNING]
> 由於 `catch` 回呼會被序列化並在稍後由 Laravel 佇列執行，因此您不應該在 `catch` 回呼中使用 `$this` 變數。

<a name="running-the-queue-worker"></a>
## 執行佇列工作者


<a name="the-queue-work-command"></a>
### The `queue:work` Command

Laravel 包含了一個 Artisan 指令，可用於啟動佇列工作者，並在有新任務被推送到佇列時進行處理。您可以使用 `queue:work` Artisan 指令來執行工作者。請注意，一旦 `queue:work` 指令啟動後，它將會持續執行，直到被手動停止或您關閉終端機為止：

```shell
php artisan queue:work
```

> [!NOTE]
> 若要讓 `queue:work` 行程永久在背景執行，您應該使用如 [Supervisor](#supervisor-configuration) 的行程監控器，以確保佇列工作者不會停止執行。

如果您希望指令的輸出內容包含已處理的任務 ID、連線名稱及佇列名稱，可以在呼叫 `queue:work` 指令時加入 `-v` 旗標：

```shell
php artisan queue:work -v
```

請記住，佇列工作者是常駐型（long-lived）行程，會將已啟動的應用程式狀態儲存在記憶體中。因此，在工作者啟動後，它們將不會注意到您程式碼庫中的任何變更。所以在部署過程中，請務必 [重啟您的佇列工作者](#queue-workers-and-deployment)。此外，請記住，您的應用程式所建立或修改的任何靜態狀態（static state）都不會在不同任務之間自動重設。

另一種選擇是，您可以執行 `queue:listen` 指令。當使用 `queue:listen` 指令時，每當您想要重新載入更新後的程式碼或重設應用程式狀態，您不需要手動重啟工作者；然而，此指令的執行效率明顯低於 `queue:work` 指令：

```shell
php artisan queue:listen
```


<a name="running-multiple-queue-workers"></a>
#### 執行多個佇列工作者

要為佇列配置多個工作者以同時處理任務，您只需要啟動多個 `queue:work` 行程即可。這可以在本機上透過多個終端機分頁來完成，或在正式環境中使用行程管理器的設定來達成。[當使用 Supervisor 時](#supervisor-configuration)，您可以使用 `numprocs` 設定值。


<a name="specifying-the-connection-queue"></a>
#### 指定連線與佇列

您也可以指定工作者應該使用哪一個佇列連線。傳遞給 `work` 指令的連線名稱，應對應於您的 `config/queue.php` 設定檔中所定義的其中一個連線：

```shell
php artisan queue:work redis
```

預設情況下，`queue:work` 指令僅會處理給定連線上預設佇列的任務。然而，您可以透過僅處理特定連線上的特定佇列，來進一步自訂您的佇列工作者。例如，若您所有的電子郵件都在 `redis` 佇列連線的 `emails` 佇列中處理，您可以執行以下指令來啟動一個僅處理該佇列的工作者：

```shell
php artisan queue:work redis --queue=emails
```


<a name="processing-a-specified-number-of-jobs"></a>
#### 處理指定數量的任務

`--once` 選項可用於指示工作者僅處理佇列中的單一任務：

```shell
php artisan queue:work --once
```

`--max-jobs` 選項可用於指示工作者處理指定數量的任務後隨即退出。當此選項與 [Supervisor](#supervisor-configuration) 搭配使用時非常有用，這能讓您的工作者在處理完指定數量的任務後自動重啟，從而釋放它們可能已累積的任何記憶體：

```shell
php artisan queue:work --max-jobs=1000
```


<a name="processing-all-queued-jobs-then-exiting"></a>
#### 處理所有佇列中的任務後退出

`--stop-when-empty` 選項可用於指示工作者處理完所有任務後優雅地退出。當您在 Docker 容器內處理 Laravel 佇列時，如果您希望在佇列清空後關閉容器，此選項會非常實用：

```shell
php artisan queue:work --stop-when-empty
```


<a name="processing-jobs-for-a-given-number-of-seconds"></a>
#### 處理指定秒數的任務

`--max-time` 選項可用於指示工作者執行指定的秒數後退出。當此選項與 [Supervisor](#supervisor-configuration) 搭配使用時非常有用，這樣一來，工作者在執行指定的時間後會自動重啟，釋放任何可能已累積的記憶體：

```shell
# Process jobs for one hour and then exit...
php artisan queue:work --max-time=3600
```


<a name="worker-sleep-duration"></a>
#### 工作者睡眠時間

當佇列中有可用的任務時，工作者會持續處理任務，任務之間沒有任何延遲。然而，`sleep` 選項決定了當沒有可用任務時，工作者將「睡眠」多少秒。當然，在睡眠期間，工作者將不會處理任何新任務：

```shell
php artisan queue:work --sleep=3
```


<a name="maintenance-mode-queues"></a>
#### 維護模式與佇列

當您的應用程式處於 [維護模式](/docs/{{version}}/configuration#maintenance-mode) 時，系統將不會處理任何已佇列的任務。一旦應用程式解除維護模式，這些任務將會繼續照常處理。

若要強制您的佇列工作者即使在啟用維護模式時仍處理任務，您可以使用 `--force` 選項：

```shell
php artisan queue:work --force
```


<a name="resource-considerations"></a>
#### 資源考量

守護行程（Daemon）佇列工作者在處理每個任務之前並不會「重啟」框架。因此，您應該在每個任務完成後釋放任何龐大的資源。例如，如果您正在使用 [GD 函式庫](https://www.php.net/manual/en/book.image.php) 進行圖片處理，則應在處理完圖片後使用 `imagedestroy` 釋放記憶體。


<a name="queue-priorities"></a>
### 佇列優先權

有時您可能希望為佇列的處理方式指定優先權。例如，在您的 `config/queue.php` 設定檔中，您可以將 `redis` 連線的預設 `queue` 設定為 `low`。然而，有時您可能希望將任務推送到 `high` 優先權的佇列，如下所示：

```php
dispatch((new Job)->onQueue('high'));
```

若要啟動一個工作者，以確保在繼續處理 `low` 佇列中的任何任務之前，先處理完所有 `high` 佇列中的任務，請向 `work` 指令傳遞一個以逗號分隔的佇列名稱列表：

```shell
php artisan queue:work --queue=high,low
```


<a name="queue-workers-and-deployment"></a>
### 佇列工作者與部署

由於佇列工作者是常駐型行程，如果不重啟，它們將不會注意到程式碼的變更。因此，部署使用佇列工作者的應用程式最簡單的方法，就是在部署過程中重啟工作者。您可以透過執行 `queue:restart` 指令來優雅地重啟所有工作者：

```shell
php artisan queue:restart
```

此指令將指示所有佇列工作者在處理完目前任務後優雅地退出，以確保不遺失任何現有任務。由於在執行 `queue:restart` 指令時，佇列工作者會結束執行，因此您應該執行如 [Supervisor](#supervisor-configuration) 的行程管理器來自動重新啟動佇列工作者。

> [!NOTE]
> 佇列會使用 [快取](/docs/{{version}}/cache) 來儲存重啟訊號，因此在使用此功能之前，您應該先確認您的應用程式已正確設定快取驅動程式。

<a name="reacting-to-worker-signals"></a>
### 對工作者訊號做出反應

當佇列工作者在處理任務時收到終止訊號（例如 `SIGQUIT`、`SIGTERM` 或 `SIGINT`），工作者會在退出之前完成其目前的任務。然而，您的任務可能需要在行程被您的伺服器或容器協調器（container orchestrator）停止之前，先對該訊號做出反應。例如，一個執行時間較長的匯入任務可能需要停止提取新記錄並儲存其目前的進度。

若要在任務中對工作者訊號做出反應，請實作 `Illuminate\Contracts\Queue\Interruptible` 介面並在您的任務中定義一個 `interrupted` 方法。工作者收到的訊號編號將會被傳遞給 `interrupted` 方法：

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

`interrupted` 方法只有在任務執行期間，工作者收到行程訊號時才會被調用。它並不能取代 [工作者逾時](#worker-timeouts) 或任務的 [`failed` 方法](#cleaning-up-after-failed-jobs)。


<a name="job-expirations-and-timeouts"></a>
### 任務到期與逾時


<a name="job-expiration"></a>
#### 任務到期

在您的 `config/queue.php` 設定檔中，每個佇列連線都定義了一個 `retry_after` 選項。此選項指定了佇列連線在重試正在處理的任務之前，應該等待多少秒。例如，如果 `retry_after` 的值設為 `90`，且任務在處理了 90 秒後仍未被釋放或刪除，該任務將會被重新釋放回佇列中。通常，您應該將 `retry_after` 的值設為您的任務合理完成處理所需的最大秒數。

> [!WARNING]
> 唯一不包含 `retry_after` 值的佇列連線是 Amazon SQS。SQS 將會根據在 AWS 主控台中管理的 [Default Visibility Timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html)（預設可見性逾時）來重試任務。


<a name="worker-timeouts"></a>
#### 工作者逾時

`queue:work` Artisan 指令提供了一個 `--timeout` 選項。預設情況下，`--timeout` 的值為 60 秒。如果任務的處理時間超過了逾時值指定的秒數，處理該任務的工作者將會因錯誤而結束。通常，工作者將會由[伺服器上設定的行程管理器](#supervisor-configuration)自動重啟：

```shell
php artisan queue:work --timeout=60
```

`retry_after` 設定選項與 `--timeout` 命令列（CLI）選項不同，但它們相輔相成，共同確保任務不會遺失，且任務只會被成功處理一次。

> [!WARNING]
> `--timeout` 的值應該要比您的 `retry_after` 設定值至少短幾秒。這能確保處理卡死任務的工作者，在任務被重試之前就已經被終止。如果您的 `--timeout` 選項長於您的 `retry_after` 設定值，您的任務可能會被重複處理兩次。


<a name="pausing-and-resuming-queue-workers"></a>
### 暫停與恢復佇列工作者

有時您可能需要暫時阻止佇列工作者處理新任務，而不需要完全停止該工作者。例如，您可能希望在系統維護期間暫停任務處理。Laravel 提供了 `queue:pause` 與 `queue:continue` Artisan 指令來暫停與恢復佇列工作者。

若要暫停特定的佇列，請提供佇列連線名稱與佇列名稱：

```shell
php artisan queue:pause database:default
```

在這個範例中，`database` 是佇列連線名稱，而 `default` 是佇列名稱。一旦佇列被暫停，任何從該佇列處理任務的工作者都會繼續完成他們目前的工作，但在佇列恢復之前將不會取得任何新任務。

若要在已暫停的佇列上恢復處理任務，請使用 `queue:continue` 指令：

```shell
php artisan queue:continue database:default
```

恢復佇列後，工作者將會立即開始處理該佇列中的新任務。請注意，暫停佇列並不會停止工作者行程本身，它只是防止工作者從指定的佇列中處理新任務。


<a name="worker-restart-and-pause-signals"></a>
#### 工作者重啟與暫停訊號

預設情況下，佇列工作者會在每次任務反覆執行（iteration）時向快取驅動程式輪詢重啟與暫停訊號。雖然這種輪詢對於響應 `queue:restart` 和 `queue:pause` 指令至關重要，但它確實會帶來一點效能開銷。

如果您需要最佳化效能，並且不需要這些中斷功能，您可以透過呼叫 `Queue` facade 的 `withoutInterruptionPolling` 方法來全域停用此輪詢。這通常應該在您 `AppServiceProvider` 的 `boot` 方法中進行：

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

或者，您可以透過在 `Illuminate\Queue\Worker` 類別上設定靜態屬性 `$restartable` 或 `$pausable`，來個別停用重啟或暫停的輪詢：

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
> 當中斷輪詢被停用時，工作者將不會對 `queue:restart` 或 `queue:pause` 指令做出回應（取決於停用了哪些功能）。

<a name="supervisor-configuration"></a>
## Supervisor 設定

在正式環境中，您需要一種方法來保持 `queue:work` 行程(Processes)持續執行。`queue:work` 行程(Processes)可能會因為各種原因停止運作，例如超過了工作者逾時時間，或是執行了 `queue:restart` 命令。

因此，您需要設定一個行程監控器，用以偵測您的 `queue:work` 行程(Processes)何時結束並自動重新啟動它們。此外，行程監控器還能讓您指定要同時執行多少個 `queue:work` 行程(Processes)。Supervisor 是 Linux 環境中常用的行程監控器，我們將在後續的文件中討論如何設定它。

<a name="installing-supervisor"></a>
#### 安裝 Supervisor

Supervisor 是適用於 Linux 作業系統的行程監控器，如果您的 `queue:work` 行程(Processes)失敗，它會自動將其重新啟動。要在 Ubuntu 上安裝 Supervisor，您可以使用以下命令：

```shell
sudo apt-get install supervisor
```

> [!NOTE]
> If configuring and managing Supervisor yourself sounds overwhelming, consider using [Laravel Cloud](https://cloud.laravel.com), which provides a fully-managed platform for running Laravel queue workers.

<a name="configuring-supervisor"></a>
#### 設定 Supervisor

Supervisor 的設定檔通常儲存在 `/etc/supervisor/conf.d` 目錄中。在此目錄下，您可以建立任意數量的設定檔，以指示 Supervisor 應該如何監控您的行程(Processes)。例如，讓我們建立一個 `laravel-worker.conf` 檔案，來啟動並監控 `queue:work` 行程(Processes)：

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

在此範例中，`numprocs` 指令將指示 Supervisor 執行八個 `queue:work` 行程(Processes)並監控所有行程，如果它們失敗則會自動重新啟動。您應該修改設定檔中的 `command` 指令，以反映您所需的佇列連線與工作者選項。

> [!WARNING]
> You should ensure that the value of `stopwaitsecs` is greater than the number of seconds consumed by your longest running job. Otherwise, Supervisor may kill the job before it is finished processing.

<a name="starting-supervisor"></a>
#### 啟動 Supervisor

設定檔建立完成後，您可以使用以下命令更新 Supervisor 設定並啟動行程(Processes)：

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start "laravel-worker:*"
```

For more information on Supervisor, consult the [Supervisor documentation](http://supervisord.org/index.html).

<a name="dealing-with-failed-jobs"></a>
## 處理失敗的任務

有時候，您放入佇列的任務會失敗。別擔心，事情並不總是按計劃進行！Laravel 提供了一種便利的方式來[指定任務的最大嘗試次數](#max-job-attempts-and-timeout)。當非同步任務超過這個嘗試次數後，它會被寫入到 `failed_jobs` 資料庫資料表中。失敗的[同步分派任務](/docs/{{version}}/queues#synchronous-dispatching)不會儲存在此資料表中，其例外狀況會立即由應用程式處理。

在新的 Laravel 應用程式中，通常已經包含了建立 `failed_jobs` 資料表的遷移檔。然而，如果您的應用程式不包含此資料表的遷移檔，您可以使用 `make:queue-failed-table` 指令來建立該遷移：

```shell
php artisan make:queue-failed-table

php artisan migrate
```

執行 [佇列工作者](#running-the-queue-worker) 行程時，您可以使用 `queue:work` 指令的 `--tries` 選項來指定任務的最大嘗試次數。如果您沒有指定 `--tries` 選項的值，任務將只會被嘗試一次，或者嘗試該任務類別之 `Tries` 屬性所指定的次數：

```shell
php artisan queue:work redis --tries=3
```

使用 `--backoff` 選項，您可以指定當任務遇到例外狀況時，Laravel 在重試前應該等待多少秒。預設情況下，任務會立即被釋放回佇列中以便再次嘗試：

```shell
php artisan queue:work redis --tries=3 --backoff=3
```

如果您想針對個別任務，設定 Laravel 在遇到例外狀況後重試前應等待的秒數，您可以在任務類別上使用 `Backoff` 屬性：

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

如果您需要更複雜的邏輯來決定任務的退避時間，您可以在任務類別中定義一個 `backoff` 方法：

```php
/**
 * Calculate the number of seconds to wait before retrying the job.
 */
public function backoff(): int
{
    return 3;
}
```

您可以藉由定義退避值陣列，輕鬆地設定「指數型 (exponential)」退避。在這個範例中，如果還有剩餘的嘗試次數，第一次重試的延遲將為 1 秒，第二次為 5 秒，第三次為 10 秒，此後每次重試皆為 10 秒：

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
### 任務失敗後的清理

當特定任務失敗時，您可能希望向使用者傳送警示，或還原該任務已部分完成的任何操作。為了實現這一點，您可以在任務類別中定義一個 `failed` 方法。導致任務失敗的 `Throwable` 實例將會被傳遞給 `failed` 方法：

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
> A new instance of the job is instantiated before invoking the `failed` method; therefore, any class property modifications that may have occurred within the `handle` method will be lost.

失敗的任務並不一定代表它遇到了未處理的例外狀況。當任務用盡了所有允許的嘗試次數時，也會被視為失敗。這些嘗試次數可能會以下列幾種方式被消耗：

<div class="content-list" markdown="1">

- 任務逾時。
- 任務在執行期間遇到未處理的例外狀況。
- 任務被手動或透過中介層釋放回佇列中。

</div>

如果最後一次嘗試是由於任務執行期間拋出例外狀況而失敗，該例外狀況將會被傳遞給任務的 `failed` 方法。然而，如果任務是因為達到最大允許嘗試次數而失敗，則 `$exception` 將會是 `Illuminate\Queue\MaxAttemptsExceededException` 的實例。同樣地，如果任務是因為超過設定的逾時時間而失敗，則 `$exception` 將會是 `Illuminate\Queue\TimeoutExceededException` 的實例。


<a name="retrying-failed-jobs"></a>
### 重試失敗的任務

若要檢視已寫入 `failed_jobs` 資料庫資料表中的所有失敗任務，您可以使用 `queue:failed` Artisan 指令：

```shell
php artisan queue:failed
```

`queue:failed` 指令將會列出任務 ID、連線、佇列、失敗時間以及關於該任務的其他資訊。任務 ID 可用於重試失敗的任務。例如，若要重試 ID 為 `ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece` 的失敗任務，請執行以下指令：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece
```

如有必要，您您可以向該指令傳遞多個 ID：

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece 91401d2c-0784-4f43-824c-34f94a33c24d
```

您也可以重試特定佇列的所有失敗任務：

```shell
php artisan queue:retry --queue=name
```

若要重試所有失敗的任務，請執行 `queue:retry` 指令並傳遞 `all` 作為 ID：

```shell
php artisan queue:retry all
```

如果您想刪除某個失敗的任務，您可以使用 `queue:forget` 指令：

```shell
php artisan queue:forget 91401d2c-0784-4f43-824c-34f94a33c24d
```

> [!NOTE]
> When using [Horizon](/docs/{{version}}/horizon), you should use the `horizon:forget` command to delete a failed job instead of the `queue:forget` command.

若要從 `failed_jobs` 資料表中刪除所有失敗的任務，您可以使用 `queue:flush` 指令：

```shell
php artisan queue:flush
```

無論失敗的任務有多久，`queue:flush` 指令都會從您的佇列中移除所有失敗任務的紀錄。您可以使用 `--hours` 選項，僅刪除在指定小時數之前失敗的任務：

```shell
php artisan queue:flush --hours=48
```


<a name="ignoring-missing-models"></a>
### 忽略遺失的模型

將 Eloquent 模型注入到任務中時，該模型會在被放入佇列之前自動序列化，並在處理任務時重新從資料庫中檢索。然而，如果在任務等待工作者處理的過程中該模型已被刪除，您的任務可能會因 `ModelNotFoundException` 而失敗。

為了方便起見，您可以在任務類別上使用 `DeleteWhenMissingModels` 屬性，來選擇自動刪除遺失模型的任務。當此屬性存在時，Laravel 會默默地丟棄該任務而不引發例外狀況：

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
### 修剪失敗的任務

您可以藉由執行 `queue:prune-failed` Artisan 指令，來修剪應用程式 `failed_jobs` 資料表中的紀錄：

```shell
php artisan queue:prune-failed
```

預設情況下，所有超過 24 小時的失敗任務紀錄都會被修剪。如果您為該指令提供 `--hours` 選項，則只會保留在過去 N 個小時內寫入的失敗任務紀錄。例如，以下指令將會刪除所有寫入時間超過 48 小時的失敗任務紀錄：

```shell
php artisan queue:prune-failed --hours=48
```

<a name="storing-failed-jobs-in-dynamodb"></a>
### 將失敗的任務儲存在 DynamoDB

Laravel 也支援將您失敗的任務記錄儲存在 [DynamoDB](https://aws.amazon.com/dynamodb)，而不是關聯式資料庫的資料表中。但是，您必須手動建立一個 DynamoDB 資料表來儲存所有失敗的任務記錄。通常，此資料表應命名為 `failed_jobs`，但您應該根據應用程式的 `queue` 設定檔中 `queue.failed.table` 設定值來命名該資料表。

`failed_jobs` 資料表應具有一個名為 `application` 的字串型別主分割鍵 (Primary Partition Key) 以及一個名為 `uuid` 的字串型別主排序鍵 (Primary Sort Key)。鍵值的 `application` 部分將包含您的應用程式名稱，該名稱定義於應用程式 `app` 設定檔中的 `name` 設定值。由於應用程式名稱是 DynamoDB 資料表鍵值的一部分，因此您可以使用同一個資料表來儲存多個 Laravel 應用程式的失敗任務。

此外，請確保您已安裝 AWS SDK，以便您的 Laravel 應用程式可以與 Amazon DynamoDB 進行通訊：

```shell
composer require aws/aws-sdk-php
```

接下來，將 `queue.failed.driver` 設定選項的值設為 `dynamodb`。此外，您應該在失敗任務設定陣列中定義 `key`、`secret` 和 `region` 設定選項。這些選項將用於向 AWS 進行認證。當使用 `dynamodb` 驅動程式時，`queue.failed.database` 設定選項是不需要的：

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
### 停用失敗任務的儲存

您可以透過將 `queue.failed.driver` 設定選項的值設置為 `null`，來指示 Laravel 直接捨棄失敗的任務而不進行儲存。通常，這可以透過 `QUEUE_FAILED_DRIVER` 環境變數來完成：

```ini
QUEUE_FAILED_DRIVER=null
```


<a name="failed-job-events"></a>
### 失敗任務事件

如果您想要註冊一個在任務失敗時會被調用的事件監聽器，可以使用 `Queue` Facade 的 `failing` 方法。例如，我們可以在 Laravel 內建的 `AppServiceProvider` 中的 `boot` 方法裡，為此事件綁定一個閉包：

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
## 清除佇列中的任務

> [!NOTE]
> 使用 [Horizon](/docs/{{version}}/horizon) 時，您應該使用 `horizon:clear` 命令來清除佇列中的任務，而不是使用 `queue:clear` 命令。

如果您想從預設連線的預設佇列中刪除所有任務，可以使用 `queue:clear` Artisan 命令：

```shell
php artisan queue:clear
```

您也可以提供 `connection` 引數和 `queue` 選項，以刪除特定連線和佇列中的任務：

```shell
php artisan queue:clear redis --queue=emails
```

> [!WARNING]
> 清除佇列中的任務僅適用於 SQS、Redis 和 database 佇列驅動程式。此外，SQS 的訊息刪除程序最多需要 60 秒，因此在您清除佇列後的 60 秒內發送到 SQS 佇列的任務也可能隨之被刪除。


<a name="monitoring-your-queues"></a>
## 監控您的佇列

如果您的佇列突然湧入大量任務，它可能會變得不堪重負，導致任務完成的等待時間變長。如果您需要，Laravel 可以在您的佇列任務數量超過指定閾值時向您發出警報。

若要開始使用，您應該排程 `queue:monitor` 命令以[每分鐘執行一次](/docs/{{version}}/scheduling)。該命令接受您想要監控的佇列名稱，以及您期望的任務數量閾值：

```shell
php artisan queue:monitor redis:default,redis:deployments --max=100
```

單獨排程此命令並不足以觸發警報通知您佇列已滿載。當該命令發現某個佇列的任務數量超過您的閾值時，將會分派一個 `Illuminate\Queue\Events\QueueBusy` 事件。您可以在應用程式的 `AppServiceProvider` 中監聽此事件，以便向您或您的開發團隊發送通知：

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

在測試會分派任務的程式碼時，您可能會希望指示 Laravel 不要實際執行任務本身，因為任務的程式碼可以直接與分派它的程式碼分開進行測試。當然，如果要測試任務本身，您可以在測試中直接將該任務實例化並呼叫 `handle` 方法。

您可以使用 `Queue` Facade 的 `fake` 方法來防止佇列任務被實際推送到佇列中。呼叫 `Queue` Facade 的 `fake` 方法後，您就可以斷言應用程式是否有嘗試將任務推送到佇列中：

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

您可以傳遞一個閉包給 `assertPushed`、`assertNotPushed`、`assertClosurePushed` 或 `assertClosureNotPushed` 方法，以斷言推送到佇列的任務是否通過給定的「真值測試 (Truth Test)」。如果至少有一個被推送的任務通過了給定的真值測試，則斷言將會成功：

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
### 模擬部分任務

如果您只需要模擬特定的任務，同時允許其他任務正常執行，可以將要模擬的任務類別名稱傳遞給 `fake` 方法：

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

您可以使用 `except` 方法，來模擬除了指定的一組任務之外的所有任務：

```php
Queue::fake()->except([
    ShipOrder::class,
]);
```


<a name="testing-job-chains"></a>
### 測試任務鏈結

要測試任務鏈結，您需要利用 `Bus` Facade 的模擬功能。`Bus` Facade 的 `assertChained` 方法可用於斷言是否分派了[任務鏈結](/docs/{{version}}/queues#job-chaining)。`assertChained` 方法接受一個鏈結任務陣列作為其第一個引數：

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

如上例所示，鏈結任務陣列可以是任務類別名稱的陣列。不過，您也可以提供實際的任務實例陣列。這樣做時，Laravel 將確保這些任務實例與您的應用程式所分派的鏈結任務屬於相同的類別，且具有相同的屬性值：

```php
Bus::assertChained([
    new ShipOrder,
    new RecordShipment,
    new UpdateInventory,
]);
```

您可以使用 `assertDispatchedWithoutChain` 方法來斷言某個任務在被推送到佇列時，並未伴隨任何任務鏈結：

```php
Bus::assertDispatchedWithoutChain(ShipOrder::class);
```


<a name="testing-chain-modifications"></a>
#### 測試鏈結修改

如果鏈結任務[將任務前置或附加至現有鏈結中](#adding-jobs-to-the-chain)，您可以使用該任務的 `assertHasChain` 方法，來斷言該任務是否擁有預期的剩餘任務鏈結：

```php
$job = new ProcessPodcast;

$job->handle();

$job->assertHasChain([
    new TranscribePodcast,
    new OptimizePodcast,
    new ReleasePodcast,
]);
```

`assertDoesntHaveChain` 方法可用於斷言該任務的剩餘鏈結是空的：

```php
$job->assertDoesntHaveChain();
```


<a name="testing-chained-batches"></a>
#### 測試鏈結批次

如果您的任務鏈結[包含批次任務](#chains-and-batches)，您可以透過在鏈結斷言中插入 `Bus::chainedBatch` 定義，來斷言該鏈結批次是否符合您的預期：

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
### 測試任務批次

可以使用 `Bus` Facade 的 `assertBatched` 方法來斷言一個[任務批次](/docs/{{version}}/queues#job-batching)已被分派。傳給 `assertBatched` 方法的閉包會接收一個 `Illuminate\Bus\PendingBatch` 的實例，可以用來檢查該批次內的任務：

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

可以在等待處理的批次上使用 `hasJobs` 方法，以驗證該批次是否包含預期的任務。此方法接受任務實例、類別名稱或閉包組成的陣列：

```php
Bus::assertBatched(function (PendingBatch $batch) {
    return $batch->hasJobs([
        new ProcessCsvRow(row: 1),
        new ProcessCsvRow(row: 2),
        new ProcessCsvRow(row: 3),
    ]);
});
```

使用閉包時，閉包將會接收任務實例。預期的任務型別將會從閉包的型別提示中推導出來：

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

您可以使用 `assertNothingBatched` 來斷言沒有任何批次被分派：

```php
Bus::assertNothingBatched();
```


<a name="testing-job-batch-interaction"></a>
#### 測試任務與批次的互動

此外，您有時可能需要測試單個任務與其所屬批次之間的互動。例如，您可能需要測試某個任務是否取消了其批次的後續處理。為了實現這一點，您需要透過 `withFakeBatch` 方法為該任務指派一個模擬的批次。`withFakeBatch` 方法會返回一個包含任務實例與模擬批次的元組：

```php
[$job, $batch] = (new ShipOrder)->withFakeBatch();

$job->handle();

$this->assertTrue($batch->cancelled());
$this->assertEmpty($batch->added);
```


<a name="testing-job-queue-interactions"></a>
### 測試任務與佇列的互動

有時，您可能需要測試佇列任務是否[將自身釋放回佇列](#manually-releasing-a-job)，或者測試該任務是否已刪除自身。您可以透過實例化該任務並呼叫 `withFakeQueueInteractions` 方法來測試這些佇列互動。

一旦模擬了任務的佇列互動，您就可以在任務上呼叫 `handle` 方法。在執行該任務之後，有各種斷言方法可用於驗證任務的佇列互動：

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

透過在 `Queue` [facade](/docs/{{version}}/facades) 上使用 `before` 和 `after` 方法，您可以指定在處理佇列任務之前或之後執行的回呼。這些回呼是進行額外記錄或為儀表板遞增統計數據的絕佳機會。通常，您應該在[服務提供者(Service Providers)](/docs/{{version}}/providers) 的 `boot` 方法中呼叫這些方法。例如，我們可以使用 Laravel 內建的 `AppServiceProvider`：

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

透過在 `Queue` [facade](/docs/{{version}}/facades) 上使用 `looping` 方法，您可以指定在工作者嘗試從佇列獲取任務之前執行的回呼。例如，您可以註冊一個閉包，用於回復由先前失敗任務所遺留且未提交的任何交易：

```php
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Queue;

Queue::looping(function () {
    while (DB::transactionLevel() > 0) {
        DB::rollBack();
    }
});
```

當佇列工作者無法從佇列中檢索到任務時，Laravel 也會分派一個 `Illuminate\Queue\Events\WorkerIdle` 事件：

```php
use Illuminate\Queue\Events\WorkerIdle;
use Illuminate\Support\Facades\Event;

Event::listen(function (WorkerIdle $event) {
    // $event->connectionName
    // $event->queue
    // $event->workerOptions
});
```