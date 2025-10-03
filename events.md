# 事件

- [介紹](#introduction)
- [生成事件與監聽器](#generating-events-and-listeners)
- [註冊事件與監聽器](#registering-events-and-listeners)
    - [事件探索](#event-discovery)
    - [手動註冊事件](#manually-registering-events)
    - [閉包監聽器](#closure-listeners)
- [定義事件](#defining-events)
- [定義監聽器](#defining-listeners)
- [佇列事件監聽器](#queued-event-listeners)
    - [手動與佇列互動](#manually-interacting-with-the-queue)
    - [佇列事件監聽器與資料庫事務](#queued-event-listeners-and-database-transactions)
    - [佇列監聽器中介層](#queued-listener-middleware)
    - [加密的佇列監聽器](#encrypted-queued-listeners)
    - [處理失敗的工作](#handling-failed-jobs)
- [分派事件](#dispatching-events)
    - [在資料庫事務後分派事件](#dispatching-events-after-database-transactions)
    - [延遲事件](#deferring-events)
- [事件訂閱者](#event-subscribers)
    - [撰寫事件訂閱者](#writing-event-subscribers)
    - [註冊事件訂閱者](#registering-event-subscribers)
- [測試](#testing)
    - [模擬部分事件](#faking-a-subset-of-events)
    - [範圍事件模擬](#scoped-event-fakes)

<a name="introduction"></a>
## 介紹

Laravel 的事件提供了一個簡單的觀察者模式實作，讓您能夠訂閱和監聽應用程式中發生的各種事件。事件類別通常儲存在 `app/Events` 目錄中，而它們的監聽器則儲存在 `app/Listeners` 中。如果您在應用程式中沒有看到這些目錄，請不用擔心，它們會在您使用 Artisan 命令列指令生成事件和監聽器時自動為您創建。

事件是解耦應用程式各個層面的好方法，因為單一事件可以有多個彼此獨立的監聽器。舉例來說，您可能希望在每次訂單出貨時，向您的使用者發送一個 Slack 通知。與其將您的訂單處理程式碼與 Slack 通知程式碼耦合在一起，您可以觸發一個 `App\Events\OrderShipped` 事件，監聽器可以接收此事件並用其分派一個 Slack 通知。

<a name="generating-events-and-listeners"></a>
## 生成事件與監聽器

為了快速生成事件和監聽器，您可以使用 `make:event` 和 `make:listener` Artisan 命令：

```shell
php artisan make:event PodcastProcessed

php artisan make:listener SendPodcastNotification --event=PodcastProcessed
```

為了方便起見，您也可以不帶額外引數來呼叫 `make:event` 和 `make:listener` Artisan 命令。當您這樣做時，Laravel 會自動提示您輸入類別名稱，並且在創建監聽器時，提示您監聽的事件：

```shell
php artisan make:event

php artisan make:listener
```

<a name="registering-events-and-listeners"></a>
## 註冊事件與監聽器

<a name="event-discovery"></a>
### 事件探索

預設情況下，Laravel 會透過掃描應用程式的 `Listeners` 目錄來自動尋找並註冊您的事件監聽器。當 Laravel 找到任何以 `handle` 或 `__invoke` 開頭的監聽器類別方法時，Laravel 會將這些方法註冊為該事件的監聽器，該事件在方法的簽章中進行型別提示：

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

如果您計劃將監聽器儲存在不同的目錄中或多個目錄中，您可以使用應用程式的 `bootstrap/app.php` 檔案中的 `withEvents` 方法指示 Laravel 掃描這些目錄：

```php
->withEvents(discover: [
    __DIR__.'/../app/Domain/Orders/Listeners',
])
```

您可以使用 `*` 字元作為萬用字元，掃描多個相似的目錄中的監聽器：

```php
->withEvents(discover: [
    __DIR__.'/../app/Domain/*/Listeners',
])
```

`event:list` 命令可用來列出應用程式中所有已註冊的監聽器：

```shell
php artisan event:list
```

<a name="event-discovery-in-production"></a>
#### 於正式環境中探索事件

為了加速您的應用程式，您應該使用 `optimize` 或 `event:cache` Artisan 命令快取所有應用程式監聽器的清單。通常，此命令應作為應用程式 [部署流程](/docs/{{version}}/deployment#optimization) 的一部分運行。此清單將被框架用於加快事件註冊流程。`event:clear` 命令可用於銷毀事件快取。

<a name="manually-registering-events"></a>
### 手動註冊事件

使用 `Event` Facade，您可以在應用程式的 `AppServiceProvider` 的 `boot` 方法中手動註冊事件及其對應的監聽器：

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

`event:list` 命令可用來列出應用程式中所有已註冊的監聽器：

```shell
php artisan event:list
```

<a name="closure-listeners"></a>
### 閉包監聽器

通常，監聽器會被定義為類別；不過，您也可以在應用程式的 `AppServiceProvider` 的 `boot` 方法中手動註冊基於閉包的事件監聽器：

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

當註冊基於閉包的事件監聽器時，您可以將監聽器閉包包裹在 `Illuminate\Events\queueable` 函式中，以指示 Laravel 使用 [佇列](/docs/{{version}}/queues) 執行監聽器：

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

與佇列的工作一樣，您可以使用 `onConnection`、`onQueue` 和 `delay` 方法來自訂佇列監聽器的執行方式：

```php
Event::listen(queueable(function (PodcastProcessed $event) {
    // ...
})->onConnection('redis')->onQueue('podcasts')->delay(now()->addSeconds(10)));
```

如果您想處理匿名的佇列監聽器失敗，您可以在定義 `queueable` 監聽器時向 `catch` 方法提供一個閉包。這個閉包將接收事件實例和導致監聽器失敗的 `Throwable` 實例：

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

您也可以使用 `*` 字元作為萬用字元參數來註冊監聽器，讓您可以在同一個監聽器上捕獲多個事件。萬用字元監聽器會將事件名稱作為其第一個引數，並將整個事件資料陣列作為其第二個引數：

```php
Event::listen('event.*', function (string $eventName, array $data) {
    // ...
});
```

<a name="defining-events"></a>
## 定義事件

事件類別本質上是一個資料容器，它包含與事件相關的資訊。舉例來說，假設一個 `App\Events\OrderShipped` 事件接收一個 [Eloquent ORM](/docs/{{version}}/eloquent) 物件：

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

如您所見，這個事件類別不包含任何邏輯。它是一個被購買的 `App\Models\Order` 實例的容器。事件所使用的 `SerializesModels` Trait 將會優雅地序列化任何 Eloquent 模型，如果事件物件是使用 PHP 的 `serialize` 函式進行序列化的話，例如在使用 [佇列事件監聽器](#queued-event-listeners) 時。

<a name="defining-listeners"></a>
## 定義監聽器

接下來，讓我們看看範例事件的監聽器。事件監聽器會在其 `handle` 方法中接收事件實例。當 `make:listener` Artisan 指令搭配 `--event` 選項執行時，它會自動匯入適當的事件類別，並在 `handle` 方法中為事件進行型別提示。在 `handle` 方法中，您可以執行任何必要動作來回應此事件：

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
> 您的事件監聽器也可以在其建構函式中型別提示任何所需的依賴。所有事件監聽器都透過 Laravel [服務容器](/docs/{{version}}/container)解析，因此依賴將會自動注入。

<a name="stopping-the-propagation-of-an-event"></a>
#### 停止事件的傳播

有時，您可能希望停止事件向其他監聽器傳播。您可以透過從監聽器的 `handle` 方法回傳 `false` 來實現。

<a name="queued-event-listeners"></a>
## 佇列事件監聽器

如果您的監聽器將執行緩慢的任務（例如寄送電子郵件或發送 HTTP 請求），則將監聽器排入佇列會很有益處。在使用佇列監聽器之前，請務必[配置您的佇列](/docs/{{version}}/queues)並在伺服器或本地開發環境中啟動佇列 Worker。

要指定監聽器應該被排入佇列，請將 `ShouldQueue` 介面新增到監聽器類別。`make:listener` Artisan 命令所生成的監聽器已經將此介面匯入目前的命名空間，因此您可以立即使用它：

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

就是這樣！現在，當此監聽器處理的事件被分派時，監聽器將自動透過 Laravel 的[佇列系統](/docs/{{version}}/queues)排入佇列。如果在佇列執行監聽器時沒有拋出任何例外，則佇列任務在處理完成後將自動刪除。

<a name="customizing-the-queue-connection-queue-name"></a>
#### 自訂佇列連線、名稱與延遲

如果您想自訂事件監聽器的佇列連線、佇列名稱或佇列延遲時間，您可以在監聽器類別上定義 `$connection`、`$queue` 或 `$delay` 屬性：

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

如果您想在運行時定義監聽器的佇列連線、佇列名稱或延遲，您可以在監聽器上定義 `viaConnection`、`viaQueue` 或 `withDelay` 方法：

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

有時，您可能需要根據運行時可用的某些資料來判斷監聽器是否應該被排入佇列。為此，可以在監聽器中新增 `shouldQueue` 方法，以確定監聽器是否應該被排入佇列。如果 `shouldQueue` 方法返回 `false`，則該監聽器將不會被排入佇列：

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

如果您需要手動存取監聽器底層佇列任務的 `delete` 和 `release` 方法，您可以使用 `Illuminate\Queue\InteractsWithQueue` Trait。此 Trait 在生成的監聽器中預設匯入，並提供對這些方法的存取：

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
### 佇列事件監聽器與資料庫事務

當佇列監聽器在資料庫事務中分派時，它們可能在資料庫事務提交之前被佇列處理。發生這種情況時，您在資料庫事務期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在事務中創建的任何模型或資料庫記錄可能不存在於資料庫中。如果您的監聽器依賴於這些模型，則在處理分派佇列監聽器的任務時可能會發生意外錯誤。

如果您的佇列連線的 `after_commit` 配置選項設定為 `false`，您仍然可以透過在監聽器類別上實作 `ShouldQueueAfterCommit` 介面，來指出特定佇列監聽器應在所有開放資料庫事務提交後分派：

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
> 要了解如何解決這些問題，請查閱有關[佇列任務和資料庫事務](/docs/{{version}}/queues#jobs-and-database-transactions)的文件。

<a name="queued-listener-middleware"></a>
### 佇列監聽器中介層

佇列監聽器也可以利用[任務中介層](/docs/{{version}}/queues#job-middleware)。任務中介層允許您在佇列監聽器的執行周圍包裝自訂邏輯，從而減少監聽器本身的樣板代碼。創建任務中介層後，可以透過從監聽器的 `middleware` 方法返回它們來將它們附加到監聽器：

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

Laravel 允許您透過[加密](/docs/{{version}}/encryption)來確保佇列監聽器資料的隱私和資料完整性。首先，只需將 `ShouldBeEncrypted` 介面新增到監聽器類別。一旦此介面新增到類別，Laravel 會在將您的監聽器推送到佇列之前自動加密它：

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
### 處理失敗的工作

您的佇列事件監聽器有時可能會失敗。如果佇列監聽器超過了由您的佇列 Worker 所定義的最大嘗試次數，則將會在您的監聽器上呼叫 `failed` 方法。`failed` 方法會接收事件實例與導致失敗的 `Throwable` 實例：

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
#### 指定佇列監聽器的最大嘗試次數

如果您的某個佇列監聽器遇到錯誤，您可能不希望它無限期地重試。因此，Laravel 提供了多種方式來指定監聽器可以嘗試的次數或持續時間。

您可以在監聽器類別上定義 `tries` 屬性或方法，以指定監聽器在被視為失敗之前可以嘗試的次數：

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

作為定義監聽器在失敗前可嘗試次數的替代方案，您可以定義一個時間點，在此時間點之後監聽器就不應再被嘗試。這允許監聽器在給定時間範圍內嘗試任意次數。若要定義監聽器不應再被嘗試的時間點，請在您的監聽器類別中新增 `retryUntil` 方法。此方法應回傳 `DateTime` 實例：

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
#### 指定佇列監聽器的退避時間

如果您想設定 Laravel 在重試遇到異常的監聽器之前應等待多少秒，您可以在監聽器類別上定義一個 `backoff` 屬性：

```php
/**
 * The number of seconds to wait before retrying the queued listener.
 *
 * @var int
 */
public $backoff = 3;
```

如果您需要更複雜的邏輯來決定監聽器的退避時間，您可以在監聽器類別上定義一個 `backoff` 方法：

```php
/**
 * Calculate the number of seconds to wait before retrying the queued listener.
 */
public function backoff(OrderShipped $event): int
{
    return 3;
}
```

您可以透過從 `backoff` 方法回傳一個退避值陣列來輕鬆設定「指數型」退避。在此範例中，第一次重試的延遲為 1 秒，第二次為 5 秒，第三次為 10 秒，如果還有更多嘗試次數，則之後每次重試都將延遲 10 秒：

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
#### 指定佇列監聽器的最大異常數

有時，您可能會希望指定佇列監聽器可以嘗試多次，但如果重試是由給定數量的未處理異常觸發，則應失敗（而不是由 `release` 方法直接釋放）。若要達成此目的，您可以在監聽器類別上定義 `maxExceptions` 屬性：

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

在此範例中，監聽器將會重試最多 25 次。然而，如果監聽器拋出三個未處理的異常，監聽器將會失敗。

<a name="specifying-queued-listener-timeout"></a>
#### 指定佇列監聽器的逾時時間

通常，您大約知道您的佇列監聽器預期需要多長時間。因此，Laravel 允許您指定一個「逾時」值。如果監聽器的處理時間超過了逾時值所指定的秒數，處理該監聽器的 Worker 將會因錯誤而退出。您可以在監聽器類別上定義 `timeout` 屬性，以指定監聽器允許執行的最大秒數：

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

如果您想指示監聽器在逾時時應標記為失敗，您可以在監聽器類別上定義 `failOnTimeout` 屬性：

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

要分派事件，您可以呼叫事件上的靜態 `dispatch` 方法。此方法是透過 `Illuminate\Foundation\Events\Dispatchable` trait 提供給事件的。任何傳遞給 `dispatch` 方法的引數都將傳遞給事件的建構式：

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

如果您想有條件地分派事件，可以使用 `dispatchIf` 和 `dispatchUnless` 方法：

```php
OrderShipped::dispatchIf($condition, $order);

OrderShipped::dispatchUnless($condition, $order);
```

> [!NOTE]
> 在測試時，斷言某些事件已分派而無需實際觸發其監聽器可能很有用。Laravel 的[內建測試輔助工具](#testing)讓這變得輕而易舉。

<a name="dispatching-events-after-database-transactions"></a>
### 在資料庫事務後分派事件

有時，您可能希望指示 Laravel 僅在活躍的資料庫事務提交後才分派事件。為此，您可以在事件類別上實作 `ShouldDispatchAfterCommit` 介面。

此介面指示 Laravel 在目前的資料庫事務提交之前不分派事件。如果事務失敗，事件將被捨棄。如果在分派事件時沒有資料庫事務正在進行，則事件將立即分派：

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

延遲事件允許您延遲模型事件的分派和事件監聽器的執行，直到特定程式碼區塊完成後。當您需要確保所有相關紀錄在事件監聽器被觸發之前都已建立時，這特別有用。

要延遲事件，請向 `Event::defer()` 方法提供一個閉包：

```php
use App\Models\User;
use Illuminate\Support\Facades\Event;

Event::defer(function () {
    $user = User::create(['name' => 'Victoria Otwell']);

    $user->posts()->create(['title' => 'My first post!']);
});
```

閉包內觸發的所有事件都將在閉包執行後分派。這確保了事件監聽器能夠存取在延遲執行期間建立的所有相關紀錄。如果閉包內發生例外狀況，則延遲的事件將不會分派。

要僅延遲特定事件，請將事件陣列作為第二個引數傳遞給 `defer` 方法：

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

事件訂閱者是可以在訂閱者類別本身內訂閱多個事件的類別，這允許您在單一類別中定義多個事件處理器。訂閱者應定義一個 `subscribe` 方法，該方法接收一個事件分派器實例。您可以呼叫給定分派器上的 `listen` 方法來註冊事件監聽器：

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

如果您的事件監聽器方法定義在訂閱者本身內，您可能會發現從訂閱者的 `subscribe` 方法返回一個事件和方法名稱的陣列更方便。Laravel 會在註冊事件監聽器時自動判斷訂閱者的類別名稱：

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

在撰寫訂閱者後，如果它們遵循 Laravel 的[事件探索慣例](#event-discovery)，Laravel 會自動註冊訂閱者內的處理器方法。否則，您可以透過 `Event` Facade 的 `subscribe` 方法手動註冊您的訂閱者。通常，這應該在您應用程式 `AppServiceProvider` 的 `boot` 方法中完成：

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

在測試分派事件的程式碼時，你可能希望指示 Laravel 不要實際執行事件的監聽器，因為監聽器程式碼可以直接且獨立於分派對應事件的程式碼進行測試。當然，要測試監聽器本身，你可以直接實例化一個監聽器實例並在測試中呼叫 `handle` 方法。

透過使用 `Event` Facade 的 `fake` 方法，你可以阻止監聽器執行，執行待測試的程式碼，然後使用 `assertDispatched`、`assertNotDispatched` 和 `assertNothingDispatched` 方法斷言應用程式分派了哪些事件：

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

你可以將閉包傳遞給 `assertDispatched` 或 `assertNotDispatched` 方法，以斷言一個事件已被分派且通過了給定的「真值測試」。如果至少有一個事件通過了給定的真值測試，則斷言將會成功：

```php
Event::assertDispatched(function (OrderShipped $event) use ($order) {
    return $event->order->id === $order->id;
});
```

如果你只是想斷言某個事件監聽器正在監聽特定事件，你可以使用 `assertListening` 方法：

```php
Event::assertListening(
    OrderShipped::class,
    SendShipmentNotification::class
);
```

> [!WARNING]
> 呼叫 `Event::fake()` 後，將不會執行任何事件監聽器。因此，如果你的測試使用了依賴事件的模型工廠，例如在模型的 `creating` 事件期間建立 UUID，你應該在使用了模型工廠**之後**再呼叫 `Event::fake()`。


<a name="faking-a-subset-of-events"></a>
### 模擬部分事件

如果你只想為特定事件集模擬事件監聽器，你可以將它們傳遞給 `fake` 或 `fakeFor` 方法：

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

你可以使用 `except` 方法模擬所有事件，但排除一組指定事件：

```php
Event::fake()->except([
    OrderCreated::class,
]);
```


<a name="scoped-event-fakes"></a>
### 範圍事件模擬

如果你只想在測試的一部分中模擬事件監聽器，你可以使用 `fakeFor` 方法：

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