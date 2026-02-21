# 事件

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
    - [處理失敗的任務](#handling-failed-jobs)
- [分派事件](#dispatching-events)
    - [在資料庫交易後分派事件](#dispatching-events-after-database-transactions)
    - [延遲事件](#deferring-events)
- [事件訂閱者](#event-subscribers)
    - [撰寫事件訂閱者](#writing-event-subscribers)
    - [註冊事件訂閱者](#registering-event-subscribers)
- [測試](#testing)
    - [模擬部分事件](#faking-a-subset-of-events)
    - [限縮範圍的事件模擬](#scoped-event-fakes)

<a name="introduction"></a>
## 簡介

Laravel 的事件提供了一個簡單的觀察者模式 (Observer Pattern) 實作，讓你能夠訂閱並監聽應用程式中發生的各種事件。事件類別通常儲存在 `app/Events` 目錄，而監聽器則儲存在 `app/Listeners`。如果你在應用程式中沒看到這些目錄，請不用擔心，當你使用 Artisan 主控台指令產生事件與監聽器時，它們會自動為你建立。

事件是解耦應用程式各個層面的絕佳方式，因為單一事件可以有多個互不相依的監聽器。例如，你可能希望在每次訂單出貨時向使用者發送 Slack 通知。與其將訂單處理程式碼與 Slack 通知程式碼耦合在一起，你可以發出一個 `App\Events\OrderShipped` 事件，讓監聽器接收並用來發送 Slack 通知。


<a name="generating-events-and-listeners"></a>
## 產生事件與監聽器

若要快速產生事件與監聽器，你可以使用 `make:event` 與 `make:listener` Artisan 指令：

```shell
php artisan make:event PodcastProcessed

php artisan make:listener SendPodcastNotification --event=PodcastProcessed
```

為了方便起見，你也可以在不帶額外參數的情況下調用 `make:event` 和 `make:listener` Artisan 指令。當你這樣做時，Laravel 會自動提示你輸入類別名稱，以及在建立監聽器時，它應該監聽的事件：

```shell
php artisan make:event

php artisan make:listener
```


<a name="registering-events-and-listeners"></a>
## 註冊事件與監聽器


<a name="event-discovery"></a>
### 事件探索

預設情況下，Laravel 會透過掃描應用程式的 `Listeners` 目錄來自動尋找並註冊你的事件監聽器。當 Laravel 找到任何以 `handle` 或 `__invoke` 開頭的監聽器類別方法時，Laravel 會將這些方法註冊為該方法簽章中型別提示 (Type-hinted) 事件的事件監聽器：

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

你可以使用 PHP 的聯集型別 (Union Types) 來監聽多個事件：

```php
/**
 * Handle the event.
 */
public function handle(PodcastProcessed|PodcastPublished $event): void
{
    // ...
}
```

如果你計畫將監聽器儲存在不同的目錄或多個目錄中，你可以在應用程式的 `bootstrap/app.php` 檔案中使用 `withEvents` 方法指示 Laravel 掃描這些目錄：

```php
->withEvents(discover: [
    __DIR__.'/../app/Domain/Orders/Listeners',
])
```

你可以使用 `*` 字元作為萬用字元來掃描多個相似目錄中的監聽器：

```php
->withEvents(discover: [
    __DIR__.'/../app/Domain/*/Listeners',
])
```

`event:list` 指令可用於列出應用程式中註冊的所有監聽器：

```shell
php artisan event:list
```


<a name="event-discovery-in-production"></a>
#### 在正式環境中使用事件探索

為了提升應用程式的速度，你應該使用 `optimize` 或 `event:cache` Artisan 指令快取應用程式所有監聽器的清單 (Manifest)。通常，此指令應作為應用程式[部署程序](/docs/{{version}}/deployment#optimization)的一部分執行。框架將使用此清單來加快事件註冊程序。`event:clear` 指令可用於清除事件快取。


<a name="manually-registering-events"></a>
### 手動註冊事件

使用 `Event` Facade，你可以在應用程式 `AppServiceProvider` 的 `boot` 方法中手動註冊事件及其對應的監聽器：

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

`event:list` 指令可用於列出應用程式中註冊的所有監聽器：

```shell
php artisan event:list
```


<a name="closure-listeners"></a>
### Closure 監聽器

通常，監聽器被定義為類別；然而，你也可以在應用程式 `AppServiceProvider` 的 `boot` 方法中手動註冊基於 Closure 的事件監聽器：

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

註冊基於 Closure 的事件監聽器時，你可以將監聽器 Closure 包裝在 `Illuminate\Events\queueable` 函式中，以指示 Laravel 使用[佇列](/docs/{{version}}/queues)執行監聽器：

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

與佇列任務一樣，你可以使用 `onConnection`、`onQueue` 和 `delay` 方法來自訂佇列監聽器的執行：

```php
Event::listen(queueable(function (PodcastProcessed $event) {
    // ...
})->onConnection('redis')->onQueue('podcasts')->delay(now()->plus(seconds: 10)));
```

如果你想處理匿名佇列監聽器的失敗，你可以在定義 `queueable` 監聽器時向 `catch` 方法提供一個 Closure。此 Closure 將接收事件實例以及導致監聽器失敗的 `Throwable` 實例：

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

你也可以使用 `*` 字元作為萬用字元參數來註冊監聽器，讓你在同一個監聽器上捕捉多個事件。萬用字元監聽器接收事件名稱作為其第一個引數，並接收整個事件資料陣列作為其第二個引數：

```php
Event::listen('event.*', function (string $eventName, array $data) {
    // ...
});
```


<a name="defining-events"></a>
## 定義事件

事件類別本質上是一個資料容器，保存與事件相關的資訊。例如，假設 `App\Events\OrderShipped` 事件接收一個 [Eloquent ORM](/docs/{{version}}/eloquent) 物件：

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

如你所見，此事件類別不包含任何邏輯。它是已購買的 `App\Models\Order` 實例的容器。如果事件物件使用 PHP 的 `serialize` 函式進行序列化（例如在使用[佇列化監聽器](#queued-event-listeners)時），則事件使用的 `SerializesModels` trait 將優雅地序列化任何 Eloquent 模型。

<a name="defining-listeners"></a>
## 定義監聽器

接下來，讓我們看看範例事件的監聽器。事件監聽器會在其 `handle` 方法中接收事件實例。`make:listener` Artisan 指令在配合 `--event` 選項使用時，會自動匯入正確的事件類別，並在 `handle` 方法中對該事件進行型別提示 (Type-hint)。在 `handle` 方法中，您可以執行任何必要的操作來回應事件：

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
> 您的事件監聽器也可以在建構子中對所需的任何依賴項進行型別提示。所有事件監聽器都經由 Laravel [服務容器](/docs/{{version}}/container) 解析，因此依賴項將會自動被注入。


<a name="stopping-the-propagation-of-an-event"></a>
#### 停止事件傳播

有時，您可能希望停止將事件傳播給其他監聽器。您可以透過從監聽器的 `handle` 方法回傳 `false` 來達成。

<a name="queued-event-listeners"></a>
## 佇列化事件監聽器

如果你的監聽器要執行耗時的任務（如發送電子郵件或發出 HTTP 請求），將監聽器放入佇列會很有幫助。在使用佇列化監聽器之前，請確保已[配置你的佇列](/docs/{{version}}/queues)並在你的伺服器或本地開發環境中啟動一個佇列工作者 (worker)。

要指定一個監聽器應該被佇列化，請將 `ShouldQueue` 介面加入監聽器類別中。由 `make:listener` Artisan 命令產生的監聽器通常已經將此介面導入到目前的命名空間中，因此你可以立即使用它：

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

就這樣！現在，當由該監聽器處理的事件被分派時，事件分派器會使用 Laravel 的[佇列系統](/docs/{{version}}/queues)自動將該監聽器排入佇列。如果在佇列執行監聽器時沒有拋出任何例外，則該佇列任務在處理完成後將自動被刪除。


<a name="customizing-the-queue-connection-queue-name"></a>
#### 自定義佇列連線、名稱與延遲

如果你想要自定義事件監聽器的佇列連線、佇列名稱或佇列延遲時間，你可以在你的監聽器類別中定義 `$connection`、`$queue` 或 `$delay` 屬性：

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

如果你想要在執行時 (runtime) 定義監聽器的佇列連線、佇列名稱或延遲時間，可以在監聽器中定義 `viaConnection`、`viaQueue` 或 `withDelay` 方法：

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
#### 條件式佇列化監聽器

有時，你可能需要根據僅在執行時可用的某些數據來判斷監聽器是否應進入佇列。為了實現這一點，可以在監聽器中添加一個 `shouldQueue` 方法來判斷該監聽器是否應進入佇列。如果 `shouldQueue` 方法返回 `false`，該監聽器將不會進入佇列：

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

如果你需要手動訪問監聽器底層佇列任務的 `delete` 和 `release` 方法，你可以使用 `Illuminate\Queue\InteractsWithQueue` trait。此 trait 在產生的監聽器中預設已導入，並提供了對這些方法的訪問：

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

當佇列化監聽器在資料庫交易中被分派時，它們可能會在資料庫交易提交之前就由佇列處理。發生這種情況時，你在資料庫交易期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫記錄也可能不存在於資料庫中。如果你的監聽器依賴這些模型，則在處理分派佇列化監聽器的任務時可能會發生意外錯誤。

如果你的佇列連線中的 `after_commit` 設定項目被設為 `false`，你仍然可以透過在監聽器類別上實作 `ShouldQueueAfterCommit` 介面，來表示特定的佇列化監聽器應在所有開啟的資料庫交易提交後才被分派：

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
> 若要瞭解更多關於解決這些問題的方法，請參閱關於[佇列任務與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的說明文件。


<a name="queued-listener-middleware"></a>
### 佇列化監聽器中介層

佇列化監聽器也可以利用[任務中介層](/docs/{{version}}/queues#job-middleware)。任務中介層允許你在佇列化監聽器的執行邏輯周圍封裝自定義邏輯，從而減少監聽器本身的重複程式碼。在建立任務中介層之後，可以透過監聽器的 `middleware` 方法回傳它們，進而將其掛載到監聽器上：

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

Laravel 允許你透過[加密](/docs/{{version}}/encryption)來確保佇列化監聽器資料的隱私與完整性。要開始使用，只需將 `ShouldBeEncrypted` 介面加入監聽器類別即可。一旦將此介面加入類別，Laravel 就會在將你的監聽器推送到佇列之前自動對其進行加密：

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
> 唯一的監聽器需要支援[鎖定](/docs/{{version}}/cache#atomic-locks)的快取驅動。目前 `memcached`、`redis`、`dynamodb`、`database`、`file` 與 `array` 快取驅動皆支援原子鎖。

有時，您可能希望確保在任何時間點，佇列中只有一個特定監聽器的執行個體。您可以透過在監聽器類別實作 `ShouldBeUnique` 介面來達成此目的：

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

在上述範例中，`AcquireProductKey` 監聽器是唯一的。因此，如果佇列中已經存在另一個該監聽器的執行個體且尚未處理完成，則此監聽器將不會被放入佇列。這能確保每個授權只會取得一個產品金鑰，即使該授權在短時間內被多次儲存也是如此。

在某些情況下，您可能希望定義一個特定的「鍵值 (Key)」來讓監聽器變得唯一，或者您可能希望指定一個逾時時間，超過該時間後監聽器將不再保持唯一。為此，您可以在監聽器類別中定義 `uniqueId` 與 `uniqueFor` 屬性或方法。這些方法會接收事件執行個體，讓您可以使用事件資料來建構回傳值：

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

在上述範例中，`AcquireProductKey` 監聽器會根據授權 ID 保持唯一。因此，對於同一個授權，在現有的監聽器處理完成之前，任何新的監聽器分派都將被忽略。這可以防止同一個授權取得重複的產品金鑰。此外，如果現有的監聽器在一小時內未被處理，則唯一鎖將被釋放，且具有相同唯一鍵值的另一個監聽器可以被放入佇列。

> [!WARNING]
> 如果您的應用程式是從多個網頁伺服器或容器分派事件，則應確保所有伺服器都與同一個中央快取伺服器通訊，以便 Laravel 能夠精確判斷監聽器是否唯一。

預設情況下，Laravel 會使用預設的快取驅動來取得唯一鎖。但是，如果您希望使用另一個驅動來取得鎖定，可以定義一個 `uniqueVia` 方法，並回傳應使用的快取驅動：

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

<a name="handling-failed-jobs"></a>
### 處理失敗的任務

有時您的佇列化事件監聽器可能會失敗。如果佇列化監聽器的重試次數超過了佇列工作者定義的最大嘗試次數，則會呼叫監聽器中的 `failed` 方法。`failed` 方法會接收事件實例以及導致失敗的 `Throwable` 實例：

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

如果您的某個佇列化監聽器遇到錯誤，您可能不希望它無限期地持續重試。因此，Laravel 提供了多種方式來指定監聽器可以嘗試的次數或時長。

您可以在監聽器類別中定義 `tries` 屬性或方法，以指定監聽器在被視為失敗之前可以嘗試的次數：

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

除了定義監聽器失敗前的重試次數外，您也可以定義一個監聽器不再重試的時間點。這允許監聽器在給定的時間範圍內嘗試任意次數。若要定義監聽器不再重試的時間，請在監聽器類別中加入 `retryUntil` 方法。此方法應回傳一個 `DateTime` 實例：

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
#### 指定佇列化監聽器的延遲時間 (Backoff)

如果您想設定 Laravel 在重試遇到異常的監聽器之前應等待多少秒，可以透過在監聽器類別中定義 `backoff` 屬性來實現：

```php
/**
 * The number of seconds to wait before retrying the queued listener.
 *
 * @var int
 */
public $backoff = 3;
```

如果您需要更複雜的邏輯來決定監聽器的延遲時間，可以在監聽器類別中定義 `backoff` 方法：

```php
/**
 * Calculate the number of seconds to wait before retrying the queued listener.
 */
public function backoff(OrderShipped $event): int
{
    return 3;
}
```

您可以透過從 `backoff` 方法回傳一個延遲值陣列來輕鬆設定「指數型 (exponential)」延遲。在此範例中，第一次重試的延遲時間為 1 秒，第二次重試為 5 秒，第三次重試為 10 秒，如果還有剩餘的嘗試次數，則之後的每次重試皆為 10 秒：

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

有時您可能希望指定一個佇列化監聽器可以嘗試多次，但如果重試是由特定次數的未處理例外所觸發（而非直接由 `release` 方法釋放），則該監聽器應被視為失敗。為了實現這一點，您可以在監聽器類別中定義 `maxExceptions` 屬性：

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

在此範例中，監聽器最多會重試 25 次。然而，如果監聽器拋出三個未處理的例外，則該監聽器將會失敗。


<a name="specifying-queued-listener-timeout"></a>
#### 指定佇列化監聽器的逾時時間

通常，您大致知道您預期佇列化監聽器需要執行多久。因此，Laravel 允許您指定一個「逾時 (timeout)」值。如果監聽器的處理時間超過了逾時值指定的秒數，處理該監聽器的工作者將會因錯誤而結束執行。您可以透過在監聽器類別中定義 `timeout` 屬性來指定監聽器允許運行的最大秒數：

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

如果您想指出監聽器在逾時後應被標記為失敗，您可以在監聽器類別中定義 `failOnTimeout` 屬性：

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

要分派事件，您可以呼叫事件上的靜態 `dispatch` 方法。此方法透過 `Illuminate\Foundation\Events\Dispatchable` trait 在事件中提供。傳遞給 `dispatch` 方法的任何參數都將傳遞給事件的建構子：

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
> 在測試時，斷言某些事件已分派而不需要實際觸發其監聽器會很有幫助。Laravel [內建的測試輔助函式](#testing) 讓這件事變得輕而易舉。


<a name="dispatching-events-after-database-transactions"></a>
### 在資料庫交易後分派事件

有時，您可能希望指示 Laravel 僅在當前資料庫交易提交 (Commit) 後才分派事件。為此，您可以在事件類別上實作 `ShouldDispatchAfterCommit` 介面。

此介面指示 Laravel 在當前資料庫交易提交之前不要分派事件。如果交易失敗，該事件將被捨棄。如果在分派事件時沒有正在進行中的資料庫交易，該事件將立即被分派：

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

延遲事件允許您將模型事件的分派與事件監聽器的執行延遲到特定程式碼區塊完成之後。當您需要確保在觸發事件監聽器之前已建立所有相關記錄時，這特別有用。

要延遲事件，請提供一個 Closure 給 `Event::defer()` 方法：

```php
use App\Models\User;
use Illuminate\Support\Facades\Event;

Event::defer(function () {
    $user = User::create(['name' => 'Victoria Otwell']);

    $user->posts()->create(['title' => 'My first post!']);
});
```

在 Closure 內觸發的所有事件都將在 Closure 執行後分派。這確保了事件監聽器可以存取在延遲執行期間建立的所有相關記錄。如果 Closure 內發生例外狀況，則不會分派延遲事件。

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

事件訂閱者是可以在訂閱者類別本身中訂閱多個事件的類別，允許您在單個類別中定義多個事件處理常式。訂閱者應定義一個 `subscribe` 方法，該方法接收一個事件分派器實例。您可以呼叫該分派器上的 `listen` 方法來註冊事件監聽器：

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

如果您在訂閱者本身中定義了事件監聽器方法，您可能會發現從訂閱者的 `subscribe` 方法返回一個事件與方法名稱的陣列會更方便。Laravel 在註冊事件監聽器時會自動判斷訂閱者的類別名稱：

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

撰寫完訂閱者後，如果訂閱者符合 Laravel 的 [事件探索慣例](#event-discovery)，Laravel 將自動註冊訂閱者中的處理方法。否則，您可以使用 `Event` facade 的 `subscribe` 方法手動註冊您的訂閱者。通常，這應該在應用程式的 `AppServiceProvider` 的 `boot` 方法中完成：

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

測試會分派事件的程式碼時，您可能希望指示 Laravel 不要實際執行事件的監聽器，因為監聽器的程式碼可以單獨於分派對應事件的程式碼之外進行直接測試。當然，若要測試監聽器本身，您可以在測試中實例化一個監聽器實例並直接呼叫 `handle` 方法。

使用 `Event` Facade 的 `fake` 方法，您可以防止監聽器執行，執行待測程式碼，然後使用 `assertDispatched`、`assertNotDispatched` 和 `assertNothingDispatched` 方法來斷言您的應用程式分派了哪些事件：

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

您可以將一個 Closure 傳遞給 `assertDispatched` 或 `assertNotDispatched` 方法，以斷言分派的事件是否通過指定的「真值測試 (Truth Test)」。如果至少有一個分派的事件通過了給定的真值測試，則斷言成功：

```php
Event::assertDispatched(function (OrderShipped $event) use ($order) {
    return $event->order->id === $order->id;
});
```

如果您只是想斷言某個事件監聽器正在監聽特定的事件，可以使用 `assertListening` 方法：

```php
Event::assertListening(
    OrderShipped::class,
    SendShipmentNotification::class
);
```

> [!WARNING]
> 呼叫 `Event::fake()` 後，將不會執行任何事件監聽器。因此，如果您的測試使用的模型工廠 (Model Factories) 依賴於事件（例如在模型的 `creating` 事件期間建立 UUID），則應在呼叫工廠**之後**再呼叫 `Event::fake()`。


<a name="faking-a-subset-of-events"></a>
### 模擬部分事件

如果您只想針對特定的事件集合模擬事件監聽器，可以將它們傳遞給 `fake` 或 `fakeFor` 方法：

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

您可以使用 `except` 方法來模擬除了指定事件集合之外的所有事件：

```php
Event::fake()->except([
    OrderCreated::class,
]);
```


<a name="scoped-event-fakes"></a>
### 限縮範圍的事件模擬

如果您只想在測試的一部分中模擬事件監聽器，可以使用 `fakeFor` 方法：

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