# 事件

- [簡介](#introduction)
- [生成事件與監聽器](#generating-events-and-listeners)
- [註冊事件與監聽器](#registering-events-and-listeners)
    - [事件探索](#event-discovery)
    - [手動註冊事件](#manually-registering-events)
    - [閉包監聽器](#closure-listeners)
- [定義事件](#defining-events)
- [定義監聽器](#defining-listeners)
- [佇列事件監聽器](#queued-event-listeners)
    - [手動與佇列互動](#manually-interacting-with-the-queue)
    - [佇列事件監聽器與資料庫交易](#queued-event-listeners-and-database-transactions)
    - [佇列監聽器 Middleware](#queued-listener-middleware)
    - [加密的佇列監聽器](#encrypted-queued-listeners)
    - [處理失敗的任務](#handling-failed-jobs)
- [分派事件](#dispatching-events)
    - [在資料庫交易後分派事件](#dispatching-events-after-database-transactions)
    - [延遲事件](#deferring-events)
- [事件訂閱者](#event-subscribers)
    - [撰寫事件訂閱者](#writing-event-subscribers)
    - [註冊事件訂閱者](#registering-event-subscribers)
- [測試](#testing)
    - [模擬部分事件](#faking-a-subset-of-events)
    - [範圍事件模擬](#scoped-event-fakes)

<a name="introduction"></a>
## 簡介

Laravel 的事件提供了一個簡單的觀察者模式實作，讓你可以訂閱並監聽應用程式中發生的各種事件。事件類別通常儲存在 `app/Events` 目錄中，而其監聽器則儲存在 `app/Listeners` 中。如果你的應用程式中沒有這些目錄，請不用擔心，當你使用 Artisan 命令生成事件和監聽器時，它們會自動為你建立。

事件是解耦應用程式各個方面的好方法，因為一個單一事件可以有多個互不依賴的監聽器。例如，你可能希望每次訂單出貨時都向使用者發送 Slack 通知。與其將訂單處理程式碼與 Slack 通知程式碼耦合，你可以觸發一個 `App\Events\OrderShipped` 事件，然後監聽器可以接收並使用它來分派 Slack 通知。

<a name="generating-events-and-listeners"></a>
## 生成事件與監聽器

為了快速生成事件和監聽器，你可以使用 `make:event` 和 `make:listener` Artisan 命令：

```shell
php artisan make:event PodcastProcessed

php artisan make:listener SendPodcastNotification --event=PodcastProcessed
```

為了方便起見，你也可以不帶額外參數地呼叫 `make:event` 和 `make:listener` Artisan 命令。當你這樣做時，Laravel 會自動提示你輸入類別名稱，並且在建立監聽器時，會提示它應該監聽的事件：

```shell
php artisan make:event

php artisan make:listener
```

<a name="registering-events-and-listeners"></a>
## 註冊事件與監聽器

<a name="event-discovery"></a>
### 事件探索

預設情況下，Laravel 會透過掃描應用程式的 `Listeners` 目錄來自動尋找並註冊你的事件監聽器。當 Laravel 找到任何以 `handle` 或 `__invoke` 開頭的監聽器類別方法時，Laravel 會將這些方法註冊為該方法簽章中型別提示的事件的事件監聽器：

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

你可以使用 PHP 的聯集型別來監聽多個事件：

```php
/**
 * Handle the event.
 */
public function handle(PodcastProcessed|PodcastPublished $event): void
{
    // ...
}
```

如果你打算將監聽器儲存在不同的目錄或多個目錄中，你可以指示 Laravel 使用應用程式 `bootstrap/app.php` 檔案中的 `withEvents` 方法來掃描這些目錄：

```php
->withEvents(discover: [
    __DIR__.'/../app/Domain/Orders/Listeners',
])
```

你可以使用 `*` 字元作為萬用字元來掃描多個類似的目錄：

```php
->withEvents(discover: [
    __DIR__.'/../app/Domain/*/Listeners',
])
```

`event:list` 命令可以用來列出應用程式中所有已註冊的監聽器：

```shell
php artisan event:list
```

<a name="event-discovery-in-production"></a>
#### 生產環境中的事件探索

為了提高應用程式的速度，你應該使用 `optimize` 或 `event:cache` Artisan 命令快取應用程式所有監聽器的清單。通常，此命令應作為應用程式[部署流程](/docs/{{version}}/deployment#optimization)的一部分執行。此清單將由框架使用，以加快事件註冊過程。`event:clear` 命令可用於清除事件快取。

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

`event:list` 命令可以用來列出應用程式中所有已註冊的監聽器：

```shell
php artisan event:list
```

<a name="closure-listeners"></a>
### 閉包監聽器

通常，監聽器被定義為類別；但是，你也可以在應用程式 `AppServiceProvider` 的 `boot` 方法中手動註冊基於閉包的事件監聽器：

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
#### 可佇列的匿名事件監聽器

註冊基於閉包的事件監聽器時，你可以將監聽器閉包包裝在 `Illuminate\Events\queueable` 函數中，以指示 Laravel 使用[佇列](/docs/{{version}}/queues)執行監聽器：

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
})->onConnection('redis')->onQueue('podcasts')->delay(now()->addSeconds(10)));
```

如果你想處理匿名佇列監聽器失敗，你可以在定義 `queueable` 監聽器時向 `catch` 方法提供一個閉包。此閉包將接收事件實例和導致監聽器失敗的 `Throwable` 實例：

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

你也可以使用 `*` 字元作為萬用字元參數來註冊監聽器，允許你在同一個監聽器上捕獲多個事件。萬用字元監聽器將事件名稱作為第一個參數，並將整個事件資料陣列作為第二個參數：

```php
Event::listen('event.*', function (string $eventName, array $data) {
    // ...
});
```

<a name="defining-events"></a>
## 定義事件

事件類別本質上是一個資料容器，它包含與事件相關的資訊。例如，假設 `App\Events\OrderShipped` 事件接收一個 [Eloquent ORM](/docs/{{version}}/eloquent) 物件：

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

如你所見，這個事件類別不包含任何邏輯。它是一個包含已購買的 `App\Models\Order` 實例的容器。事件使用的 `SerializesModels` Trait 將在事件物件使用 PHP 的 `serialize` 函數序列化時（例如在使用[佇列監聽器](#queued-event-listeners)時）優雅地序列化任何 Eloquent 模型。

<a name="defining-listeners"></a>
## 定義監聽器

接下來，讓我們看看我們範例事件的監聽器。事件監聽器在其 `handle` 方法中接收事件實例。`make:listener` Artisan 命令在帶有 `--event` 選項呼叫時，會自動匯入正確的事件類別並在 `handle` 方法中型別提示事件。在 `handle` 方法中，你可以執行任何必要的動作來回應事件：

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
> 你的事件監聽器也可以在其建構函式上型別提示它們所需的任何依賴。所有事件監聽器都透過 Laravel [服務容器](/docs/{{version}}/container)解析，因此依賴將自動注入。

<a name="stopping-the-propagation-of-an-event"></a>
#### 停止事件的傳播

有時，你可能希望停止事件向其他監聽器傳播。你可以透過從監聽器的 `handle` 方法返回 `false` 來實現。

<a name="queued-event-listeners"></a>
## 佇列事件監聽器

如果你的監聽器將執行諸如發送電子郵件或發出 HTTP 請求等耗時任務，那麼將監聽器排入佇列會很有益處。在使用佇列監聽器之前，請確保[配置你的佇列](/docs/{{version}}/queues)並在你的伺服器或本地開發環境中啟動一個佇列工作者。

要指定監聽器應該排入佇列，請將 `ShouldQueue` 介面新增到監聽器類別。`make:listener` Artisan 命令生成的監聽器已經將此介面匯入到當前命名空間中，因此你可以立即使用它：

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

就是這樣！現在，當此監聽器處理的事件被分派時，監聽器將自動由事件分派器使用 Laravel 的[佇列系統](/docs/{{version}}/queues)排入佇列。如果在佇列執行監聽器時沒有拋出異常，則佇列任務將在處理完成後自動刪除。

<a name="customizing-the-queue-connection-queue-name"></a>
#### 自訂佇列連線、佇列名稱和延遲

如果你想自訂事件監聽器的佇列連線、佇列名稱或佇列延遲時間，你可以在監聽器類別上定義 `$connection`、`$queue` 或 `$delay` 屬性：

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

如果你想在執行時定義監聽器的佇列連線、佇列名稱或延遲，你可以在監聽器上定義 `viaConnection`、`viaQueue` 或 `withDelay` 方法：

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
#### 有條件地將監聽器排入佇列

有時，你可能需要根據執行時才可用的某些資料來判斷監聽器是否應該排入佇列。為此，可以向監聽器新增 `shouldQueue` 方法，以判斷監聽器是否應該排入佇列。如果 `shouldQueue` 方法返回 `false`，則監聽器將不會排入佇列：

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

如果你需要手動存取監聽器底層佇列任務的 `delete` 和 `release` 方法，你可以使用 `Illuminate\Queue\InteractsWithQueue` Trait。此 Trait 預設匯入到生成的監聽器中，並提供對這些方法的存取：

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
        if (true) {
            $this->release(30);
        }
    }
}
```

<a name="queued-event-listeners-and-database-transactions"></a>
### 佇列事件監聽器與資料庫交易

當佇列監聽器在資料庫交易中分派時，它們可能會在資料庫交易提交之前由佇列處理。發生這種情況時，你在資料庫交易期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫記錄可能不存在於資料庫中。如果你的監聽器依賴於這些模型，則在處理分派佇列監聽器的任務時可能會發生意外錯誤。

如果你的佇列連線的 `after_commit` 配置選項設定為 `false`，你仍然可以透過在監聽器類別上實作 `ShouldQueueAfterCommit` 介面來指示特定佇列監聽器應在所有開啟的資料庫交易提交後分派：

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
> 要了解有關解決這些問題的更多資訊，請查閱有關[佇列任務和資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的文件。

<a name="queued-listener-middleware"></a>
### 佇列監聽器 Middleware

佇列監聽器也可以使用[任務 Middleware](/docs/{{version}}/queues#job-middleware)。任務 Middleware 允許你將自訂邏輯包裝在佇列監聽器的執行周圍，減少監聽器本身的樣板程式碼。建立任務 Middleware 後，可以透過從監聽器的 `middleware` 方法返回它們來將它們附加到監聽器：

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
#### 加密的佇列監聽器

Laravel 允許你透過[加密](/docs/{{version}}/encryption)來確保佇列監聽器資料的隱私和完整性。要開始使用，只需將 `ShouldBeEncrypted` 介面新增到監聽器類別。一旦此介面新增到類別中，Laravel 將在將你的監聽器推送到佇列之前自動加密它：

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

<a name="handling-failed-jobs"></a>
### 處理失敗的任務

有時你的佇列事件監聽器可能會失敗。如果佇列監聽器超過了佇列工作者定義的最大嘗試次數，則將在你的監聽器上呼叫 `failed` 方法。`failed` 方法接收事件實例和導致失敗的 `Throwable`：

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
#### 指定佇列監聽器最大嘗試次數

如果你的佇列監聽器遇到錯誤，你可能不希望它無限期地重試。因此，Laravel 提供了多種方法來指定監聽器可以嘗試的次數或時間。

你可以在監聽器類別上定義 `tries` 屬性，以指定監聽器在被視為失敗之前可以嘗試的次數：

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

作為定義監聽器在失敗之前可以嘗試的次數的替代方法，你可以定義監聽器不應再嘗試的時間。這允許監聽器在給定時間範圍內嘗試任意次數。要定義監聽器不應再嘗試的時間，請向監聽器類別新增 `retryUntil` 方法。此方法應返回一個 `DateTime` 實例：

```php
use DateTime;

/**
 * Determine the time at which the listener should timeout.
 */
public function retryUntil(): DateTime
{
    return now()->addMinutes(5);
}
```

如果同時定義了 `retryUntil` 和 `tries`，Laravel 會優先使用 `retryUntil` 方法。

<a name="specifying-queued-listener-backoff"></a>
#### 指定佇列監聽器退避時間

如果你想配置 Laravel 在重試遇到異常的監聽器之前應該等待多少秒，你可以在監聽器類別上定義 `backoff` 屬性：

```php
/**
 * The number of seconds to wait before retrying the queued listener.
 *
 * @var int
 */
public $backoff = 3;
```

如果你需要更複雜的邏輯來確定監聽器的退避時間，你可以在監聽器類別上定義 `backoff` 方法：

```php
/**
 * Calculate the number of seconds to wait before retrying the queued listener.
 */
public function backoff(OrderShipped $event): int
{
    return 3;
}
```

你可以透過從 `backoff` 方法返回退避值陣列來輕鬆配置「指數」退避。在此範例中，第一次重試的重試延遲為 1 秒，第二次重試為 5 秒，第三次重試為 10 秒，如果還有更多嘗試，則每次後續重試為 10 秒：

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
#### 指定佇列監聽器最大異常數

有時你可能希望指定佇列監聽器可以嘗試多次，但如果重試是由給定數量的未處理異常觸發的（而不是直接由 `release` 方法釋放的），則應失敗。為此，你可以在監聽器類別上定義 `maxExceptions` 屬性：

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

在此範例中，監聽器將重試最多 25 次。但是，如果監聽器拋出三個未處理的異常，則監聽器將失敗。

<a name="specifying-queued-listener-timeout"></a>
#### 指定佇列監聽器逾時

通常，你大致知道你的佇列監聽器預計需要多長時間。因此，Laravel 允許你指定一個「逾時」值。如果監聽器處理時間超過逾時值指定的秒數，則處理監聽器的工作者將以錯誤退出。你可以透過在監聽器類別上定義 `timeout` 屬性來定義監聽器允許執行的最大秒數：

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

如果你想指示監聽器在逾時時應標記為失敗，你可以在監聽器類別上定義 `failOnTimeout` 屬性：

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

要分派事件，你可以呼叫事件上的靜態 `dispatch` 方法。此方法透過 `Illuminate\Foundation\Events\Dispatchable` Trait 在事件上可用。傳遞給 `dispatch` 方法的任何參數都將傳遞給事件的建構函式：

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

如果你想有條件地分派事件，你可以使用 `dispatchIf` 和 `dispatchUnless` 方法：

```php
OrderShipped::dispatchIf($condition, $order);

OrderShipped::dispatchUnless($condition, $order);
```

> [!NOTE]
> 測試時，斷言某些事件已分派而無需實際觸發其監聽器會很有幫助。Laravel 的[內建測試輔助函式](#testing)使其變得輕而易舉。

<a name="dispatching-events-after-database-transactions"></a>
### 在資料庫交易後分派事件

有時，你可能希望指示 Laravel 僅在活動資料庫交易提交後才分派事件。為此，你可以在事件類別上實作 `ShouldDispatchAfterCommit` 介面。

此介面指示 Laravel 在當前資料庫交易提交之前不分派事件。如果交易失敗，事件將被丟棄。如果在分派事件時沒有進行中的資料庫交易，則事件將立即分派：

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

延遲事件允許你延遲模型事件的分派和事件監聽器的執行，直到特定程式碼區塊完成後。這在你需要確保所有相關記錄在事件監聽器觸發之前都已建立時特別有用。

要延遲事件，請向 `Event::defer()` 方法提供一個閉包：

```php
use App\Models\User;
use Illuminate\Support\Facades\Event;

Event::defer(function () {
    $user = User::create(['name' => 'Victoria Otwell']);

    $user->posts()->create(['title' => 'My first post!']);
});
```

閉包中觸發的所有事件都將在閉包執行後分派。這確保事件監聽器可以存取在延遲執行期間建立的所有相關記錄。如果在閉包中發生異常，則延遲事件將不會分派。

要僅延遲特定事件，請將事件陣列作為第二個參數傳遞給 `defer` 方法：

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

事件訂閱者是可以在訂閱者類別本身中訂閱多個事件的類別，允許你在單一類別中定義多個事件處理器。訂閱者應該定義一個 `subscribe` 方法，該方法接收一個事件分派器實例。你可以呼叫給定分派器上的 `listen` 方法來註冊事件監聽器：

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

如果你的事件監聽器方法定義在訂閱者本身中，你可能會發現從訂閱者的 `subscribe` 方法返回事件和方法名稱的陣列更方便。Laravel 將在註冊事件監聽器時自動確定訂閱者的類別名稱：

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

撰寫訂閱者後，如果它們遵循 Laravel 的[事件探索慣例](#event-discovery)，Laravel 將自動註冊訂閱者中的處理器方法。否則，你可以使用 `Event` Facade 的 `subscribe` 方法手動註冊你的訂閱者。通常，這應該在應用程式 `AppServiceProvider` 的 `boot` 方法中完成：

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

測試分派事件的程式碼時，你可能希望指示 Laravel 不要實際執行事件的監聽器，因為監聽器的程式碼可以直接且獨立於分派相應事件的程式碼進行測試。當然，要測試監聽器本身，你可以實例化一個監聽器實例並直接在測試中呼叫 `handle` 方法。

使用 `Event` Facade 的 `fake` 方法，你可以阻止監聽器執行，執行被測試的程式碼，然後使用 `assertDispatched`、`assertNotDispatched` 和 `assertNothingDispatched` 方法斷言應用程式分派了哪些事件：

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

你可以向 `assertDispatched` 或 `assertNotDispatched` 方法傳遞一個閉包，以斷言分派的事件通過了給定的「真實性測試」。如果至少有一個事件通過了給定的真實性測試，則斷言將成功：

```php
Event::assertDispatched(function (OrderShipped $event) use ($order) {
    return $event->order->id === $order->id;
});
```

如果你只想斷言事件監聽器正在監聽給定事件，你可以使用 `assertListening` 方法：

```php
Event::assertListening(
    OrderShipped::class,
    SendShipmentNotification::class
);
```

> [!WARNING]
> 呼叫 `Event::fake()` 後，不會執行任何事件監聽器。因此，如果你的測試使用依賴事件的模型工廠，例如在模型的 `creating` 事件期間建立 UUID，你應該在使用工廠**之後**呼叫 `Event::fake()`。

<a name="faking-a-subset-of-events"></a>
### 模擬部分事件

如果你只想模擬特定事件集的事件監聽器，你可以將它們傳遞給 `fake` 或 `fakeFor` 方法：

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

你可以使用 `except` 方法模擬所有事件，除了指定的一組事件：

```php
Event::fake()->except([
    OrderCreated::class,
]);
```

<a name="scoped-event-fakes"></a>
### 範圍事件模擬

如果你只想模擬測試的一部分事件監聽器，你可以使用 `fakeFor` 方法：

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

