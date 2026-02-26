# Events

- [簡介](#introduction)
- [產生事件與監聽器](#generating-events-and-listeners)
- [註冊事件與監聽器](#registering-events-and-listeners)
    - [事件探索](#event-discovery)
    - [手動註冊事件](#manually-registering-events)
    - [Closure 監聽器](#closure-listeners)
- [定義事件](#defining-events)
- [定義監聽器](#defining-listeners)
- [佇列化事件監聽器](#queued-event-listeners)
    - [手動與佇列互動](#manually-interacting-with-the-queue)
    - [佇列化事件監聽器與資料庫交易](#queued-event-listeners-and-database-transactions)
    - [佇列化監聽器中介層](#queued-listener-middleware)
    - [加密的佇列化監聽器](#encrypted-queued-listeners)
    - [唯一的事件監聽器](#unique-event-listeners)
        - [在開始處理前保持監聽器的唯一性](#keeping-listeners-unique-until-processing-begins)
        - [唯一監聽器鎖定](#unique-listener-locks)
    - [處理失敗的任務 (Jobs)](#handling-failed-jobs)
- [分派事件](#dispatching-events)
    - [在資料庫交易後分派事件](#dispatching-events-after-database-transactions)
    - [延遲事件](#deferring-events)
- [事件訂閱者](#event-subscribers)
    - [撰寫事件訂閱者](#writing-event-subscribers)
    - [註冊事件訂閱者](#registering-event-subscribers)
- [測試](#testing)
    - [模擬部分事件](#faking-a-subset-of-events)
    - [具範圍的事件模擬 (Fakes)](#scoped-event-fakes)

<a name="introduction"></a>
## 簡介

Laravel 的事件提供了一個簡單的觀察者模式實作，允許您訂閱與監聽在應用程式中發生的各種事件。事件類別通常儲存在 `app/Events` 目錄中，而它們的監聽器則儲存在 `app/Listeners`。如果您在應用程式中沒看到這些目錄請不用擔心，當您使用 Artisan 控制台命令產生事件與監聽器時，系統會自動為您建立這些目錄。

事件是解耦應用程式各個面向的好方法，因為一個事件可以有多個互不依賴的監聽器。例如，您可能希望在每次訂單出貨時向使用者發送 Slack 通知。您可以發送一個 `App\Events\OrderShipped` 事件，讓監聽器接收並用來分派 Slack 通知，而不是將訂單處理程式碼與 Slack 通知程式碼耦合在一起。


<a name="generating-events-and-listeners"></a>
## 產生事件與監聽器

要快速產生事件與監聽器，您可以使用 `make:event` 與 `make:listener` Artisan 命令：

```shell
php artisan make:event PodcastProcessed

php artisan make:listener SendPodcastNotification --event=PodcastProcessed
```

為了方便起見，您也可以在不帶額外參數的情況下執行 `make:event` 與 `make:listener` Artisan 命令。當您這樣做時，Laravel 會自動提示您輸入類別名稱，並在建立監聽器時提示其應監聽的事件：

```shell
php artisan make:event

php artisan make:listener
```


<a name="registering-events-and-listeners"></a>
## 註冊事件與監聽器


<a name="event-discovery"></a>
### 事件探索

預設情況下，Laravel 會透過掃描應用程式的 `Listeners` 目錄自動尋找並註冊您的事件監聽器。當 Laravel 發現任何以 `handle` 或 `__invoke` 開頭的監聽器類別方法時，Laravel 會將這些方法註冊為該方法簽章中透過型別提示所指定的事件監聽器：

```php
use App\Events\PodcastProcessed;

class SendPodcastNotification
{
    /**
     * Handle the event.
     */
    public function handle(PodcastProcessed $event): void
    {
        // ...
    }
}
```

您可以使用 PHP 的聯集型別來監聽多個事件：

```php
/**
 * Handle the event.
 */
public function handle(PodcastProcessed|PodcastPublished $event): void
{
    // ...
}
```

如果您打算將監聽器儲存在不同的目錄或多個目錄中，可以使用應用程式 `bootstrap/app.php` 檔案中的 `withEvents` 方法指示 Laravel 掃描這些目錄：

```php
->withEvents(discover: [
    __DIR__.'/../app/Domain/Orders/Listeners',
])
```

您可以使用 `*` 字元作為萬用字元來掃描多個類似的目錄：

```php
->withEvents(discover: [
    __DIR__.'/../app/Domain/*/Listeners',
])
```

`event:list` 命令可用於列出應用程式中註冊的所有監聽器：

```shell
php artisan event:list
```


<a name="event-discovery-in-production"></a>
#### 在正式環境中的事件探索

為了提升應用程式的速度，您應該使用 `optimize` 或 `event:cache` Artisan 命令快取應用程式所有監聽器的清單。通常，此命令應作為應用程式[佈署流程](/docs/{{version}}/deployment#optimization)的一部分執行。框架將使用此清單來加速事件註冊過程。`event:clear` 命令可用於刪除事件快取。


<a name="manually-registering-events"></a>
### 手動註冊事件

使用 `Event` facade，您可以在應用程式 `AppServiceProvider` 的 `boot` 方法中手動註冊事件及其對應的監聽器：

```php
use App\Domain\Orders\Events\PodcastProcessed;
use App\Domain\Orders\Listeners\SendPodcastNotification;
use Illuminate\Support\Facades\Event;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Event::listen(
        PodcastProcessed::class,
        SendPodcastNotification::class,
    );
}
```

`event:list` 命令可用於列出應用程式中註冊的所有監聽器：

```shell
php artisan event:list
```


<a name="closure-listeners"></a>
### Closure 監聽器

通常，監聽器被定義為類別；然而，您也可以在應用程式 `AppServiceProvider` 的 `boot` 方法中手動註冊基於 Closure 的事件監聽器：

```php
use App\Events\PodcastProcessed;
use Illuminate\Support\Facades\Event;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Event::listen(function (PodcastProcessed $event) {
        // ...
    });
}
```


<a name="queuable-anonymous-event-listeners"></a>
#### 可佇列化的匿名事件監聽器

在註冊基於 Closure 的事件監聽器時，您可以將監聽器 Closure 包裝在 `Illuminate\Events\queueable` 函式中，以指示 Laravel 使用[佇列](/docs/{{version}}/queues)執行監聽器：

```php
use App\Events\PodcastProcessed;
use function Illuminate\Events\queueable;
use Illuminate\Support\Facades\Event;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Event::listen(queueable(function (PodcastProcessed $event) {
        // ...
    }));
}
```

與佇列任務一樣，您可以使用 `onConnection`、`onQueue` 和 `delay` 方法來自訂佇列監聽器的執行：

```php
Event::listen(queueable(function (PodcastProcessed $event) {
    // ...
})->onConnection('redis')->onQueue('podcasts')->delay(now()->plus(seconds: 10)));
```

如果您想處理匿名佇列監聽器的失敗，可以在定義 `queueable` 監聽器時為 `catch` 方法提供一個 Closure。此 Closure 將接收事件實例以及導致監聽器失敗的 `Throwable` 實例：

```php
use App\Events\PodcastProcessed;
use function Illuminate\Events\queueable;
use Illuminate\Support\Facades\Event;
use Throwable;

Event::listen(queueable(function (PodcastProcessed $event) {
    // ...
})->catch(function (PodcastProcessed $event, Throwable $e) {
    // The queued listener failed...
}));
```


<a name="wildcard-event-listeners"></a>
#### 萬用字元事件監聽器

您還可以使用 `*` 字元作為萬用字元參數來註冊監聽器，從而允許您在同一個監聽器上捕捉多個事件。萬用字元監聽器接收事件名稱作為其第一個引數，並將整個事件資料陣列作為其第二個引數：

```php
Event::listen('event.*', function (string $eventName, array $data) {
    // ...
});
```


<a name="defining-events"></a>
## 定義事件

事件類別本質上是一個存放與事件相關資訊的資料容器。例如，假設一個 `App\Events\OrderShipped` 事件接收一個 [Eloquent ORM](/docs/{{version}}/eloquent) 物件：

```php
<?php

namespace App\Events;

use App\Models\Order;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class OrderShipped
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    /**
     * Create a new event instance.
     */
    public function __construct(
        public Order $order,
    ) {}
}
```

如您所見，此事件類別不包含任何邏輯。它只是已購買的 `App\Models\Order` 實例的容器。如果使用 PHP 的 `serialize` 函式對事件物件進行序列化（例如在使用[佇列化監聽器](#queued-event-listeners)時），事件所使用的 `SerializesModels` trait 將會優雅地序列化任何 Eloquent 模型。

<a name="defining-listeners"></a>
## 定義監聽器

接下來，讓我們來看看範例事件的監聽器。事件監聽器會在他們的 `handle` 方法中接收事件實例。當使用 `--event` 選項執行 `make:listener` Artisan 指令時，將會自動匯入正確的事件類別，並在 `handle` 方法中對該事件進行型別提示 (Type-hint)。在 `handle` 方法內，您可以執行任何回應事件所需的動作：

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;

class SendShipmentNotification
{
    /**
     * Create the event listener.
     */
    public function __construct() {}

    /**
     * Handle the event.
     */
    public function handle(OrderShipped $event): void
    {
        // Access the order using $event->order...
    }
}
```

> [!NOTE]
> 您的事件監聽器也可以在建構子中對所需的任何依賴進行型別提示。所有的事件監聽器都會透過 Laravel [服務容器](/docs/{{version}}/container) 解析，因此依賴會自動被注入。


<a name="stopping-the-propagation-of-an-event"></a>
#### 停止事件傳遞

有時候，您可能希望停止將事件傳遞給其他監聽器。您可以透過在監聽器的 `handle` 方法中回傳 `false` 來達成此目的。

<a name="queued-event-listeners"></a>
## 佇列化事件監聽器

如果您的監聽器將執行緩慢的任務（例如發送電子郵件或發送 HTTP 請求），則將監聽器加入佇列會很有幫助。在使用佇列化監聽器之前，請確保已[設定您的佇列](/docs/{{version}}/queues)並在伺服器或本地開發環境中啟動佇列工作者 (Queue Worker)。

要指定監聽器應加入佇列，請將 `ShouldQueue` 介面新增到監聽器類別中。由 `make:listener` Artisan 指令產生的監聽器已經將此介面匯入到目前的命名空間中，因此您可以立即使用它：

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldQueue;

class SendShipmentNotification implements ShouldQueue
{
    // ...
}
```

就這樣！現在，當由該監聽器處理的事件被分派時，監聽器將自動由事件分派器使用 Laravel 的[佇列系統](/docs/{{version}}/queues)加入佇列。如果佇列執行監聽器時沒有拋出任何例外，則佇列任務在處理完成後將自動刪除。


<a name="customizing-the-queue-connection-queue-name"></a>
#### 自定義佇列連接、名稱與延遲

如果您想要自定義事件監聽器的佇列連接、佇列名稱或佇列延遲時間，可以在您的監聽器類別中定義 `$connection`、`$queue` 或 `$delay` 屬性：

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldQueue;

class SendShipmentNotification implements ShouldQueue
{
    /**
     * The name of the connection the job should be sent to.
     *
     * @var string|null
     */
    public $connection = 'sqs';

    /**
     * The name of the queue the job should be sent to.
     *
     * @var string|null
     */
    public $queue = 'listeners';

    /**
     * The time (seconds) before the job should be processed.
     *
     * @var int
     */
    public $delay = 60;
}
```

如果您想在執行時定義監聽器的佇列連接、佇列名稱或延遲，可以在監聽器上定義 `viaConnection`、`viaQueue` 或 `withDelay` 方法：

```php
/**
 * Get the name of the listener's queue connection.
 */
public function viaConnection(): string
{
    return 'sqs';
}

/**
 * Get the name of the listener's queue.
 */
public function viaQueue(): string
{
    return 'listeners';
}

/**
 * Get the number of seconds before the job should be processed.
 */
public function withDelay(OrderShipped $event): int
{
    return $event->highPriority ? 0 : 60;
}
```


<a name="conditionally-queueing-listeners"></a>
#### 條件式佇列監聽器

有時，您可能需要根據僅在執行時可用的某些資料來決定是否應將監聽器加入佇列。為此，可以在監聽器中新增 `shouldQueue` 方法來決定監聽器是否應加入佇列。如果 `shouldQueue` 方法回傳 `false`，則該監聽器將不會被加入佇列：

```php
<?php

namespace App\Listeners;

use App\Events\OrderCreated;
use Illuminate\Contracts\Queue\ShouldQueue;

class RewardGiftCard implements ShouldQueue
{
    /**
     * Reward a gift card to the customer.
     */
    public function handle(OrderCreated $event): void
    {
        // ...
    }

    /**
     * Determine whether the listener should be queued.
     */
    public function shouldQueue(OrderCreated $event): bool
    {
        return $event->order->subtotal >= 5000;
    }
}
```


<a name="manually-interacting-with-the-queue"></a>
### 手動與佇列互動

如果您需要手動存取監聽器底層佇列任務的 `delete` 和 `release` 方法，可以使用 `Illuminate\Queue\InteractsWithQueue` trait。此 trait 在產生的監聽器中預設已匯入，並提供對這些方法的存取權限：

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;

class SendShipmentNotification implements ShouldQueue
{
    use InteractsWithQueue;

    /**
     * Handle the event.
     */
    public function handle(OrderShipped $event): void
    {
        if ($condition) {
            $this->release(30);
        }
    }
}
```


<a name="queued-event-listeners-and-database-transactions"></a>
### 佇列化事件監聽器與資料庫交易

當佇列化監聽器在資料庫交易中被分派時，它們可能會在資料庫交易提交前就被佇列處理。發生這種情況時，您在資料庫交易期間對模型或資料庫紀錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫紀錄可能還不存在於資料庫中。如果您的監聽器依賴這些模型，則在處理分派佇列監聽器的任務時可能會發生非預期的錯誤。

如果您的佇列連接的 `after_commit` 設定選項設為 `false`，您仍然可以透過在監聽器類別實作 `ShouldQueueAfterCommit` 介面，來表示特定的佇列化監聽器應在所有開啟的資料庫交易都提交後再分派：

```php
<?php

namespace App\Listeners;

use Illuminate\Contracts\Queue\ShouldQueueAfterCommit;
use Illuminate\Queue\InteractsWithQueue;

class SendShipmentNotification implements ShouldQueueAfterCommit
{
    use InteractsWithQueue;
}
```

> [!NOTE]
> 若要了解更多關於解決這些問題的資訊，請參閱有關[佇列任務與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的說明文件。


<a name="queued-listener-middleware"></a>
### 佇列化監聽器中介層

佇列化監聽器也可以利用[任務中介層](/docs/{{version}}/queues#job-middleware)。任務中介層允許您在佇列監聽器的執行周圍封裝自定義邏輯，減少監聽器本身的樣板程式碼。建立任務中介層後，可以透過監聽器的 `middleware` 方法回傳它們，將其附加到監聽器上：

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use App\Jobs\Middleware\RateLimited;
use Illuminate\Contracts\Queue\ShouldQueue;

class SendShipmentNotification implements ShouldQueue
{
    /**
     * Handle the event.
     */
    public function handle(OrderShipped $event): void
    {
        // Process the event...
    }

    /**
     * Get the middleware the listener should pass through.
     *
     * @return array<int, object>
     */
    public function middleware(OrderShipped $event): array
    {
        return [new RateLimited];
    }
}
```


<a name="encrypted-queued-listeners"></a>
#### 加密的佇列化監聽器

Laravel 允許您透過[加密](/docs/{{version}}/encryption)來確保佇列監聽器資料的隱私與完整性。要開始使用，只需將 `ShouldBeEncrypted` 介面新增到監聽器類別即可。將此介面新增到類別後，Laravel 會在將監聽器推送到佇列之前自動對其進行加密：

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldBeEncrypted;
use Illuminate\Contracts\Queue\ShouldQueue;

class SendShipmentNotification implements ShouldQueue, ShouldBeEncrypted
{
    // ...
}
```

<a name="unique-event-listeners"></a>
### 唯一的事件監聽器

> [!WARNING]
> 唯一的監聽器需要支援 [鎖定 (Locks)](/docs/{{version}}/cache#atomic-locks) 的快取驅動器。目前，`memcached`、`redis`、`dynamodb`、`database`、`file` 與 `array` 快取驅動器皆支援原子鎖定。

有時，你可能想確保在任何時間點，佇列中都只有一個特定監聽器的實例。你可以透過在監聽器類別中實作 `ShouldBeUnique` 介面來達成此目的：

```php
<?php

namespace App\Listeners;

use App\Events\LicenseSaved;
use Illuminate\Contracts\Queue\ShouldBeUnique;
use Illuminate\Contracts\Queue\ShouldQueue;

class AcquireProductKey implements ShouldQueue, ShouldBeUnique
{
    public function __invoke(LicenseSaved $event): void
    {
        // ...
    }
}
```

在上面的範例中，`AcquireProductKey` 監聽器是唯一的。因此，如果佇列中已經存在該監聽器的另一個實例且尚未處理完成，則該監聽器將不會被加入佇列。這確保了每個許可證 (License) 只會取得一個產品金鑰，即使許可證在短時間內被連續儲存多次也是如此。

在某些情況下，你可能想定義一個特定的「Key」來使監聽器具有唯一性，或者你可能想指定一個超時時間，超過該時間後監聽器不再保持唯一。為此，你可以在監聽器類別中定義 `uniqueId` 與 `uniqueFor` 屬性或方法。這些方法會接收事件實例，讓你能夠使用事件資料來建構回傳值：

```php
<?php

namespace App\Listeners;

use App\Events\LicenseSaved;
use Illuminate\Contracts\Queue\ShouldBeUnique;
use Illuminate\Contracts\Queue\ShouldQueue;

class AcquireProductKey implements ShouldQueue, ShouldBeUnique
{
    /**
     * The number of seconds after which the listener's unique lock will be released.
     *
     * @var int
     */
    public $uniqueFor = 3600;

    public function __invoke(LicenseSaved $event): void
    {
        // ...
    }

    /**
     * Get the unique ID for the listener.
     */
    public function uniqueId(LicenseSaved $event): string
    {
        return 'listener:'.$event->license->id;
    }
}
```

在上面的範例中，`AcquireProductKey` 監聽器會根據許可證 ID 保持唯一。因此，對於同一個許可證，任何新的監聽器分派都會被忽略，直到現有的監聽器處理完成。這可以防止為同一個許可證取得重複的產品金鑰。此外，如果現有的監聽器在一小時內沒有被處理，唯一鎖定將被釋放，另一個具有相同唯一 Key 的監聽器就可以被加入佇列。

> [!WARNING]
> 如果你的應用程式從多個網頁伺服器或容器分派事件，你應該確保所有的伺服器都在與同一個中央快取伺服器通訊，以便 Laravel 能準確判斷監聽器是否唯一。

<a name="keeping-listeners-unique-until-processing-begins"></a>
#### 在開始處理前保持監聽器的唯一性

預設情況下，唯一監聽器會在處理完成或所有重試嘗試皆失敗後「解鎖」。然而，在某些情況下，你可能希望監聽器在開始處理之前就立即解鎖。要達成此目的，你的監聽器應該實作 `ShouldBeUniqueUntilProcessing` 契約，而不是 `ShouldBeUnique` 契約：

```php
<?php

namespace App\Listeners;

use App\Events\LicenseSaved;
use Illuminate\Contracts\Queue\ShouldBeUniqueUntilProcessing;
use Illuminate\Contracts\Queue\ShouldQueue;

class AcquireProductKey implements ShouldQueue, ShouldBeUniqueUntilProcessing
{
    // ...
}
```

<a name="unique-listener-locks"></a>
#### 唯一監聽器鎖定

在幕後，當分派一個 `ShouldBeUnique` 監聽器時，Laravel 會嘗試使用 `uniqueId` 作為 Key 來取得 [鎖定 (Lock)](/docs/{{version}}/cache#atomic-locks)。如果鎖定已被佔用，則監聽器不會被分派。當監聽器完成處理或所有重試嘗試皆失敗時，該鎖定就會被釋放。預設情況下，Laravel 會使用預設的快取驅動器來取得此鎖定。但是，如果你希望使用另一個驅動器來取得鎖定，可以定義一個 `uniqueVia` 方法，並回傳應使用的快取驅動器：

```php
<?php

namespace App\Listeners;

use App\Events\LicenseSaved;
use Illuminate\Contracts\Cache\Repository;
use Illuminate\Support\Facades\Cache;

class AcquireProductKey implements ShouldQueue, ShouldBeUnique
{
    // ...

    /**
     * Get the cache driver for the unique listener lock.
     */
    public function uniqueVia(LicenseSaved $event): Repository
    {
        return Cache::driver('redis');
    }
}
```

> [!NOTE]
> 如果你只需要限制監聽器的併發處理，請改用 [WithoutOverlapping](/docs/{{version}}/queues#preventing-job-overlaps) 任務中介層。

<a name="handling-failed-jobs"></a>
### 處理失敗的任務 (Jobs)

有時你的佇列化事件監聽器可能會失敗。如果佇列化監聽器超過了由佇列工作者 (Queue Worker) 定義的最大嘗試次數，則會呼叫監聽器上的 `failed` 方法。`failed` 方法會接收事件實例以及導致失敗的 `Throwable` 實例：

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;
use Throwable;

class SendShipmentNotification implements ShouldQueue
{
    use InteractsWithQueue;

    /**
     * Handle the event.
     */
    public function handle(OrderShipped $event): void
    {
        // ...
    }

    /**
     * Handle a job failure.
     */
    public function failed(OrderShipped $event, Throwable $exception): void
    {
        // ...
    }
}
```


<a name="specifying-queued-listener-maximum-attempts"></a>
#### 指定佇列化監聽器的最大嘗試次數

如果你的其中一個佇列化監聽器遇到錯誤，你可能不希望它無限期地持續重試。因此，Laravel 提供了多種方式來指定監聽器可以嘗試的次數或時間長度。

你可以在監聽器類別中定義一個 `tries` 屬性或方法，以指定在將監聽器視為失敗之前可以嘗試的次數：

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;

class SendShipmentNotification implements ShouldQueue
{
    use InteractsWithQueue;

    /**
     * The number of times the queued listener may be attempted.
     *
     * @var int
     */
    public $tries = 5;
}
```

除了定義監聽器失敗前可以嘗試的次數外，你也可以定義一個不再嘗試監聽器的時間點。這允許監聽器在給定的時間範圍內進行任意次數的嘗試。要定義不再嘗試監聽器的時間點，請在監聽器類別中增加一個 `retryUntil` 方法。此方法應回傳一個 `DateTime` 實例：

```php
use DateTime;

/**
 * Determine the time at which the listener should timeout.
 */
public function retryUntil(): DateTime
{
    return now()->plus(minutes: 5);
}
```

如果同時定義了 `retryUntil` 和 `tries`，Laravel 會優先採用 `retryUntil` 方法。


<a name="specifying-queued-listener-backoff"></a>
#### 指定佇列化監聽器的重試延遲 (Backoff)

如果你想設定 Laravel 在重試遇到例外的監聽器之前應等待多少秒，可以透過在監聽器類別中定義 `backoff` 屬性來達成：

```php
/**
 * The number of seconds to wait before retrying the queued listener.
 *
 * @var int
 */
public $backoff = 3;
```

如果你需要更複雜的邏輯來決定監聽器的重試延遲時間，可以在監聽器類別中定義一個 `backoff` 方法：

```php
/**
 * Calculate the number of seconds to wait before retrying the queued listener.
 */
public function backoff(OrderShipped $event): int
{
    return 3;
}
```

你可以透過在 `backoff` 方法中回傳一個延遲值陣列，來輕鬆地設定「指數型 (Exponential)」重試延遲。在下方的範例中，第一次重試的延遲將為 1 秒，第二次為 5 秒，第三次為 10 秒，如果還有剩餘的嘗試次數，之後的每一次重試都將延遲 10 秒：

```php
/**
 * Calculate the number of seconds to wait before retrying the queued listener.
 *
 * @return list<int>
 */
public function backoff(OrderShipped $event): array
{
    return [1, 5, 10];
}
```


<a name="specifying-queued-listener-max-exceptions"></a>
#### 指定佇列化監聽器的最大例外次數

有時你可能希望指定佇列化監聽器可以嘗試多次，但如果重試是由給定數量的未處理例外觸發的（而不是直接由 `release` 方法釋回），則該監聽器應失敗。要實現這一點，你可以在監聽器類別中定義 `maxExceptions` 屬性：

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;

class SendShipmentNotification implements ShouldQueue
{
    use InteractsWithQueue;

    /**
     * The number of times the queued listener may be attempted.
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
     * Handle the event.
     */
    public function handle(OrderShipped $event): void
    {
        // Process the event...
    }
}
```

在下方的範例中，監聽器最多會重試 25 次。然而，如果監聽器拋出了三個未處理的例外，監聽器就會失敗。


<a name="specifying-queued-listener-timeout"></a>
#### 指定佇列化監聽器的逾時時間

通常情況下，你大致知道你預期佇列化監聽器執行所需的時間。因此，Laravel 允許你指定一個「逾時 (Timeout)」值。如果監聽器的處理時間超過了逾時值指定的秒數，處理該監聽器的工作者將因錯誤而結束。你可以在監聽器類別中定義 `timeout` 屬性，來指定監聽器允許執行的最大秒數：

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldQueue;

class SendShipmentNotification implements ShouldQueue
{
    /**
     * The number of seconds the listener can run before timing out.
     *
     * @var int
     */
    public $timeout = 120;
}
```

如果你希望在逾時後將監聽器標記為失敗，可以在監聽器類別中定義 `failOnTimeout` 屬性：

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldQueue;

class SendShipmentNotification implements ShouldQueue
{
    /**
     * Indicate if the listener should be marked as failed on timeout.
     *
     * @var bool
     */
    public $failOnTimeout = true;
}
```

<a name="dispatching-events"></a>
## 分派事件

要分派一個事件，您可以調用該事件上的靜態 `dispatch` 方法。此方法是由 `Illuminate\Foundation\Events\Dispatchable` trait 提供給事件的。傳遞給 `dispatch` 方法的任何參數都將被傳遞給事件的建構函式：

```php
<?php

namespace App\Http\Controllers;

use App\Events\OrderShipped;
use App\Models\Order;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class OrderShipmentController extends Controller
{
    /**
     * Ship the given order.
     */
    public function store(Request $request): RedirectResponse
    {
        $order = Order::findOrFail($request->order_id);

        // Order shipment logic...

        OrderShipped::dispatch($order);

        return redirect('/orders');
    }
}
```

如果您想根據條件分派事件，可以使用 `dispatchIf` 和 `dispatchUnless` 方法：

```php
OrderShipped::dispatchIf($condition, $order);

OrderShipped::dispatchUnless($condition, $order);
```

> [!NOTE]
> 在測試時，斷言某些事件已分派而實際不觸發其監聽器會很有幫助。Laravel 的[內建測試輔助方法](#testing)讓這件事變得非常簡單。


<a name="dispatching-events-after-database-transactions"></a>
### 在資料庫交易後分派事件

有時，您可能希望指示 Laravel 僅在目前的資料庫交易提交後才分派事件。為此，您可以在事件類別上實作 `ShouldDispatchAfterCommit` 介面。

此介面指示 Laravel 在目前的資料庫交易提交之前不要分派該事件。如果交易失敗，該事件將被捨棄。如果在分派事件時沒有正在進行的資料庫交易，則該事件將立即分派：

```php
<?php

namespace App\Events;

use App\Models\Order;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Events\ShouldDispatchAfterCommit;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class OrderShipped implements ShouldDispatchAfterCommit
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    /**
     * Create a new event instance.
     */
    public function __construct(
        public Order $order,
    ) {}
}
```


<a name="deferring-events"></a>
### 延遲事件

延遲事件允許您將模型事件的分派和事件監聽器的執行延遲到特定程式碼區塊完成之後。當您需要確保在觸發事件監聽器之前已建立所有相關記錄時，這特別有用。

要延遲事件，請提供一個 Closure 給 `Event::defer()` 方法：

```php
use App\Models\User;
use Illuminate\Support\Facades\Event;

Event::defer(function () {
    $user = User::create(['name' => 'Victoria Otwell']);

    $user->posts()->create(['title' => 'My first post!']);
});
```

在 Closure 內觸發的所有事件都將在 Closure 執行後分派。這確保了事件監聽器可以存取在延遲執行期間建立的所有相關記錄。如果 Closure 內發生異常，則不會分派延遲的事件。

若要僅延遲特定事件，請將事件陣列作為第二個參數傳遞給 `defer` 方法：

```php
use App\Models\User;
use Illuminate\Support\Facades\Event;

Event::defer(function () {
    $user = User::create(['name' => 'Victoria Otwell']);

    $user->posts()->create(['title' => 'My first post!']);
}, ['eloquent.created: '.User::class]);
```


<a name="event-subscribers"></a>
## 事件訂閱者


<a name="writing-event-subscribers"></a>
### 撰寫事件訂閱者

事件訂閱者是可以在訂閱者類別本身內訂閱多個事件的類別，允許您在單一類別中定義多個事件處理程序。訂閱者應該定義一個 `subscribe` 方法，該方法接收一個事件分派器實例。您可以調用給定分派器上的 `listen` 方法來註冊事件監聽器：

```php
<?php

namespace App\Listeners;

use Illuminate\Auth\Events\Login;
use Illuminate\Auth\Events\Logout;
use Illuminate\Events\Dispatcher;

class UserEventSubscriber
{
    /**
     * Handle user login events.
     */
    public function handleUserLogin(Login $event): void {}

    /**
     * Handle user logout events.
     */
    public function handleUserLogout(Logout $event): void {}

    /**
     * Register the listeners for the subscriber.
     */
    public function subscribe(Dispatcher $events): void
    {
        $events->listen(
            Login::class,
            [UserEventSubscriber::class, 'handleUserLogin']
        );

        $events->listen(
            Logout::class,
            [UserEventSubscriber::class, 'handleUserLogout']
        );
    }
}
```

如果您的事件監聽器方法定義在訂閱者本身內部，您可能會發現從訂閱者的 `subscribe` 方法返回事件和方法名稱的陣列會更方便。Laravel 在註冊事件監聽器時會自動確定訂閱者的類別名稱：

```php
<?php

namespace App\Listeners;

use Illuminate\Auth\Events\Login;
use Illuminate\Auth\Events\Logout;
use Illuminate\Events\Dispatcher;

class UserEventSubscriber
{
    /**
     * Handle user login events.
     */
    public function handleUserLogin(Login $event): void {}

    /**
     * Handle user logout events.
     */
    public function handleUserLogout(Logout $event): void {}

    /**
     * Register the listeners for the subscriber.
     *
     * @return array<string, string>
     */
    public function subscribe(Dispatcher $events): array
    {
        return [
            Login::class => 'handleUserLogin',
            Logout::class => 'handleUserLogout',
        ];
    }
}
```


<a name="registering-event-subscribers"></a>
### 註冊事件訂閱者

撰寫好訂閱者後，如果它們遵循 Laravel 的[事件探索慣例](#event-discovery)，Laravel 將自動註冊訂閱者中的處理程序方法。否則，您可以使用 `Event` Facade 的 `subscribe` 方法手動註冊您的訂閱者。通常，這應該在應用程式的 `AppServiceProvider` 的 `boot` 方法中完成：

```php
<?php

namespace App\Providers;

use App\Listeners\UserEventSubscriber;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Event::subscribe(UserEventSubscriber::class);
    }
}
```

<a name="testing"></a>
## 測試

當在測試分派事件的程式碼時，您可能希望指示 Laravel 不要實際執行事件的監聽器，因為監聽器的程式碼可以獨立於分派對應事件的程式碼，進行直接且分開的測試。當然，若要測試監聽器本身，您可以實例化一個監聽器實例，並在測試中直接呼叫 `handle` 方法。

使用 `Event` Facade 的 `fake` 方法，您可以防止監聽器執行、執行受測程式碼，並接著使用 `assertDispatched`、`assertNotDispatched` 與 `assertNothingDispatched` 方法來斷言應用程式分派了哪些事件：

```php tab=Pest
<?php

use App\Events\OrderFailedToShip;
use App\Events\OrderShipped;
use Illuminate\Support\Facades\Event;

test('orders can be shipped', function () {
    Event::fake();

    // Perform order shipping...

    // Assert that an event was dispatched...
    Event::assertDispatched(OrderShipped::class);

    // Assert an event was dispatched twice...
    Event::assertDispatched(OrderShipped::class, 2);

    // Assert an event was dispatched once...
    Event::assertDispatchedOnce(OrderShipped::class);

    // Assert an event was not dispatched...
    Event::assertNotDispatched(OrderFailedToShip::class);

    // Assert that no events were dispatched...
    Event::assertNothingDispatched();
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Events\OrderFailedToShip;
use App\Events\OrderShipped;
use Illuminate\Support\Facades\Event;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * Test order shipping.
     */
    public function test_orders_can_be_shipped(): void
    {
        Event::fake();

        // Perform order shipping...

        // Assert that an event was dispatched...
        Event::assertDispatched(OrderShipped::class);

        // Assert an event was dispatched twice...
        Event::assertDispatched(OrderShipped::class, 2);

        // Assert an event was dispatched once...
        Event::assertDispatchedOnce(OrderShipped::class);

        // Assert an event was not dispatched...
        Event::assertNotDispatched(OrderFailedToShip::class);

        // Assert that no events were dispatched...
        Event::assertNothingDispatched();
    }
}
```

您可以將一個 Closure 傳遞給 `assertDispatched` 或 `assertNotDispatched` 方法，以便斷言分派的事件通過了給定的「真實性測試 (Truth Test)」。如果至少分派了一個通過該真實性測試的事件，則斷言將成功：

```php
Event::assertDispatched(function (OrderShipped $event) use ($order) {
    return $event->order->id === $order->id;
});
```

如果您只想斷言某個事件監聽器正在監聽給定的事件，可以使用 `assertListening` 方法：

```php
Event::assertListening(
    OrderShipped::class,
    SendShipmentNotification::class
);
```

> [!WARNING]
> 呼叫 `Event::fake()` 後，將不會執行任何事件監聽器。因此，如果您的測試使用了依賴事件的模型工廠 (Model Factories)，例如在模型的 `creating` 事件期間建立 UUID，則應在 **使用** 工廠之後才呼叫 `Event::fake()`。


<a name="faking-a-subset-of-events"></a>
### 模擬部分事件

如果您只想模擬特定一組事件的事件監聽器，可以將它們傳遞給 `fake` 或 `fakeFor` 方法：

```php tab=Pest
test('orders can be processed', function () {
    Event::fake([
        OrderCreated::class,
    ]);

    $order = Order::factory()->create();

    Event::assertDispatched(OrderCreated::class);

    // Other events are dispatched as normal...
    $order->update([
        // ...
    ]);
});
```

```php tab=PHPUnit
/**
 * Test order process.
 */
public function test_orders_can_be_processed(): void
{
    Event::fake([
        OrderCreated::class,
    ]);

    $order = Order::factory()->create();

    Event::assertDispatched(OrderCreated::class);

    // Other events are dispatched as normal...
    $order->update([
        // ...
    ]);
}
```

您可以使用 `except` 方法來模擬除了指定事件以外的所有事件：

```php
Event::fake()->except([
    OrderCreated::class,
]);
```


<a name="scoped-event-fakes"></a>
### 具範圍的事件模擬 (Fakes)

如果您只想在測試的一小部分中模擬事件監聽器，可以使用 `fakeFor` 方法：

```php tab=Pest
<?php

use App\Events\OrderCreated;
use App\Models\Order;
use Illuminate\Support\Facades\Event;

test('orders can be processed', function () {
    $order = Event::fakeFor(function () {
        $order = Order::factory()->create();

        Event::assertDispatched(OrderCreated::class);

        return $order;
    });

    // Events are dispatched as normal and observers will run...
    $order->update([
        // ...
    ]);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Events\OrderCreated;
use App\Models\Order;
use Illuminate\Support\Facades\Event;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * Test order process.
     */
    public function test_orders_can_be_processed(): void
    {
        $order = Event::fakeFor(function () {
            $order = Order::factory()->create();

            Event::assertDispatched(OrderCreated::class);

            return $order;
        });

        // Events are dispatched as normal and observers will run...
        $order->update([
            // ...
        ]);
    }
}
```