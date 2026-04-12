# 事件

- [簡介](#introduction)
- [產生事件與監聽器](#generating-events-and-listeners)
- [註冊事件與監聽器](#registering-events-and-listeners)
    - [事件探索](#event-discovery)
    - [手動註冊事件](#manually-registering-events)
    - [Closure 監聽器](#closure-listeners)
- [定義事件](#defining-events)
- [定義監聽器](#defining-listeners)
- [佇列事件監聽器](#queued-event-listeners)
    - [手動與佇列互動](#manually-interacting-with-the-queue)
    - [佇列事件監聽器與資料庫交易](#queued-event-listeners-and-database-transactions)
    - [佇列監聽器中介層](#queued-listener-middleware)
    - [加密的佇列監聽器](#encrypted-queued-listeners)
    - [唯一的事件監聽器](#unique-event-listeners)
        - [保持監聽器唯一直到開始處理](#keeping-listeners-unique-until-processing-begins)
        - [唯一的監聽器鎖定](#unique-listener-locks)
    - [處理失敗的工作](#handling-failed-jobs)
- [分派事件](#dispatching-events)
    - [在資料庫交易後分派事件](#dispatching-events-after-database-transactions)
    - [延遲事件](#deferring-events)
- [事件訂閱者](#event-subscribers)
    - [編寫事件訂閱者](#writing-event-subscribers)
    - [註冊事件訂閱者](#registering-event-subscribers)
- [測試](#testing)
    - [模擬部分事件](#faking-a-subset-of-events)
    - [範圍限定的事件模擬](#scoped-event-fakes)

<a name="introduction"></a>
## 簡介

Laravel 的事件提供了一個簡單的觀察者模式 (observer pattern) 實作，允許您訂閱並監聽應用程式中發生的各種事件。事件類別通常儲存在 `app/Events` 目錄中，而其監聽器則儲存在 `app/Listeners`。如果您在應用程式中沒有看到這些目錄，請不用擔心，因為當您使用 Artisan 主控台指令產生事件與監聽器時，它們會自動為您建立。

事件是將應用程式中各種部分解耦 (decouple) 的絕佳方式，因為單一事件可以擁有複數個互不依賴的監聽器。例如，您可能希望在每次訂單出貨時向使用者發送 Slack 通知。與其將訂單處理程式碼與 Slack 通知程式碼耦合在一起，您可以觸發一個 `App\Events\OrderShipped` 事件，由監聽器接收並用來分派 Slack 通知。


<a name="generating-events-and-listeners"></a>
## 產生事件與監聽器

若要快速產生事件與監聽器，您可以使用 `make:event` 和 `make:listener` Artisan 指令：

```shell
php artisan make:event PodcastProcessed

php artisan make:listener SendPodcastNotification --event=PodcastProcessed
```

為了方便起見，您也可以在不需要額外參數的情況下調用 `make:event` 和 `make:listener` Artisan 指令。這樣做時，Laravel 會自動提示您輸入類別名稱，且在建立監聽器時，會提示該監聽器應監聽哪個事件：

```shell
php artisan make:event

php artisan make:listener
```


<a name="registering-events-and-listeners"></a>
## 註冊事件與監聽器


<a name="event-discovery"></a>
### 事件探索

預設情況下，Laravel 會透過掃描應用程式的 `Listeners` 目錄，自動尋找並註冊您的事件監聽器。當 Laravel 發現任何以 `handle` 或 `__invoke` 開頭的監聽器類別方法時，Laravel 會將這些方法註冊為該方法簽名 (signature) 中使用型別提示 (type-hint) 的事件監聽器：

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

您可以使用 PHP 的聯集型別 (union types) 來監聽多個事件：

```php
/**
 * Handle the event.
 */
public function handle(PodcastProcessed|PodcastPublished $event): void
{
    // ...
}
```

如果您打算將監聽器儲存在不同的目錄或多個目錄中，您可以使用應用程式中 `bootstrap/app.php` 檔案裡的 `withEvents` 方法，指示 Laravel 掃描這些目錄：

```php
->withEvents(discover: [
    __DIR__.'/../app/Domain/Orders/Listeners',
])
```

您可以使用 `*` 字元作為萬用字元，來掃描多個類似的目錄：

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
#### 生產環境中的事件探索

為了提升應用程式速度，您應該使用 `optimize` 或 `event:cache` Artisan 指令將所有監聽器的清單快取起來。通常，這個指令應該作為您應用程式 [部署流程](/docs/{{version}}/deployment#optimization) 的一部分來執行。此清單將被框架用於加速事件註冊過程。`event:clear` 指令可用於清除事件快取。


<a name="manually-registering-events"></a>
### 手動註冊事件

使用 `Event` Facade，您可以在應用程式 `AppServiceProvider` 的 `boot` 方法中手動註冊事件及其對應的監聽器：

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

通常監聽器被定義為類別；然而，您也可以在應用程式 `AppServiceProvider` 的 `boot` 方法中手動註冊基於 Closure 的事件監聽器：

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


<a name="queueable-anonymous-event-listeners"></a>
#### 可佇列化的匿名事件監聽器

在註冊基於 Closure 的事件監聽器時，您可以將監聽器 Closure 包裹在 `Illuminate\Events\queueable` 函數中，以指示 Laravel 使用 [佇列](/docs/{{version}}/queues) 來執行該監聽器：

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

就像佇列工作 (queued jobs) 一樣，您可以使用 `onConnection`、`onQueue` 和 `delay` 方法來自定義佇列監聽器的執行方式：

```php
Event::listen(queueable(function (PodcastProcessed $event) {
    // ...
})->onConnection('redis')->onQueue('podcasts')->delay(now()->plus(seconds: 10)));
```

如果您想要處理匿名佇列監聽器的失敗情況，可以在定義 `queueable` 監聽器時，為 `catch` 方法提供一個 Closure。此 Closure 將接收事件實例以及導致監聽器失敗的 `Throwable` 實例：

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

您也可以使用 `*` 字元作為萬用字元參數來註冊監聽器，讓您可以在同一個監聽器中捕捉多個事件。萬用字元監聽器的第一個引數會接收事件名稱，第二個引數則接收完整的事件資料陣列：

```php
Event::listen('event.*', function (string $eventName, array $data) {
    // ...
});
```


<a name="defining-events"></a>
## 定義事件

事件類別在本質上是一個資料容器，用於持有與該事件相關的資訊。例如，假設一個 `App\Events\OrderShipped` 事件接收了一個 [Eloquent ORM](/docs/{{version}}/eloquent) 物件：

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

如您所見，這個事件類別不包含任何邏輯。它是被購買的 `App\Models\Order` 實例的容器。事件所使用的 `SerializesModels` trait，會在事件物件被 PHP 的 `serialize` 函數序列化時（例如使用 [佇列監聽器](#queued-event-listeners) 時），優雅地序列化任何 Eloquent 模型。

<a name="defining-listeners"></a>
## 定義監聽器

接下來，讓我們來看看範例事件的監聽器。事件監聽器會在其 `handle` 方法中接收事件實例。當使用 `--event` 選項執行 `make:listener` Artisan 指令時，它會自動匯入正確的事件類別，並在 `handle` 方法中為該事件加上型別提示。在 `handle` 方法中，您可以執行任何回應該事件所需的動作：

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
> 您的事件監聽器也可以在建構子中為其所需的任何依賴加上型別提示。所有的事件監聽器都透過 Laravel 的 [服務容器](/docs/{{version}}/container) 解析，因此依賴將會被自動注入。


<a name="stopping-the-propagation-of-an-event"></a>
#### 停止事件傳播

有時候，您可能希望停止事件傳播至其他監聽器。您可以透過在監聽器的 `handle` 方法中回傳 `false` 來達成。

<a name="queued-event-listeners"></a>
## 佇列事件監聽器

如果您的監聽器將執行緩慢的任務（例如發送電子郵件或發出 HTTP 請求），將監聽器放入佇列會非常有幫助。在開始使用佇列監聽器之前，請確保您已 [設定您的佇列](/docs/{{version}}/queues) 並在伺服器或本地開發環境中啟動佇列工作者。

若要指定監聽器應該被放入佇列，請在監聽器類別中加入 `ShouldQueue` 介面。透過 `make:listener` Artisan 指令產生的監聽器已經在目前的命名空間中匯入了此介面，因此您可以立即使用：

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

就這麼簡單！現在，當由該監聽器處理的事件被分派時，事件分派器會使用 Laravel 的 [佇列系統](/docs/{{version}}/queues) 自動將監聽器放入佇列。如果監聽器在由佇列執行時沒有拋出任何異常，則佇列工作在處理完成後會自動被刪除。


<a name="customizing-the-queue-connection-queue-name"></a>
#### 自訂佇列連線、名稱與延遲

如果您想要自訂事件監聽器的佇列連線、佇列名稱或佇列延遲時間，您可以在監聽器類別上使用 `Connection`、`Queue` 和 `Delay` 屬性：

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\Attributes\Connection;
use Illuminate\Queue\Attributes\Delay;
use Illuminate\Queue\Attributes\Queue;

#[Connection('sqs')]
#[Queue('listeners')]
#[Delay(60)]
class SendShipmentNotification implements ShouldQueue
{
    // ...
}
```
如果您希望在執行時定義監聽器的佇列連線、佇列名稱或延遲，您可以在監聽器上定義 `viaConnection`、`viaQueue` 或 `withDelay` 方法：

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
#### 有條件地將監聽器放入佇列

有時候，您可能需要根據僅在執行時才可用的某些資料來決定監聽器是否應該被放入佇列。為了實現這一點，可以在監聽器中加入 `shouldQueue` 方法來決定監聽器是否應該被放入佇列。如果 `shouldQueue` 方法回傳 `false`，則監聽器將不會被放入佇列：

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

如果您需要手動存取監聽器底層佇列工作的 `delete` 和 `release` 方法，可以使用 `Illuminate\Queue\InteractsWithQueue` trait。產生的監聽器預設會匯入此 trait 並提供對這些方法的存取：

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
### 佇列事件監聽器與資料庫交易

當佇列監聽器在資料庫交易中被分派時，它們可能會在資料庫交易提交之前就被佇列處理。當這種情況發生時，您在資料庫交易期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫記錄可能還不存在於資料庫中。如果您的監聽器依賴於這些模型，則在處理分派佇列監聽器的工作時，可能會發生非預期的錯誤。

如果您的佇列連線之 `after_commit` 設定選項被設定為 `false`，您仍然可以在監聽器類別上實作 `ShouldQueueAfterCommit` 介面，以指定特定的佇列監聽器應該在所有開啟的資料庫交易提交後才分派：

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
> 若要深入了解如何解決這些問題，請參閱關於 [佇列工作與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions) 的文件。


<a name="queued-listener-middleware"></a>
### 佇列監聽器中介層

佇列監聽器也可以利用 [工作中介層](/docs/{{version}}/queues#job-middleware)。工作中介層允許您在佇列監聽器的執行前後封裝自訂邏輯，從而減少監聽器本身的樣板程式碼。建立工作中介層後，可以透過在監聽器的 `middleware` 方法中回傳它們來將其附加到監聽器上：

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

Laravel 允許您透過 [加密](/docs/{{version}}/encryption) 來確保佇列監聽器資料的私密性與完整性。若要開始使用，只需在監聽器類別中加入 `ShouldBeEncrypted` 介面。一旦將此介面加入類別，Laravel 在將監聽器推送到佇列之前會自動對其進行加密：

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
> 唯一的監聽器需要支援 [鎖定](/docs/{{version}}/cache#atomic-locks) 的快取驅動程式。目前 `memcached`、`redis`、`dynamodb`、`database`、`file` 與 `array` 快取驅動程式支援原子鎖。

有時候，您可能希望確保在任何時間點，佇列中僅存在一個特定監聽器的實例。您可以透過在監聽器類別中實作 `ShouldBeUnique` 介面來達成：

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

在上述範例中，`AcquireProductKey` 監聽器是唯一的。因此，如果另一個監聽器實例已經在佇列中且尚未處理完畢，該監聽器將不會被放入佇列。這確保了每個授權僅會獲取一個產品金鑰，即使該授權在短時間內被多次儲存。

在某些情況下，您可能想要定義一個特定的「金鑰」來讓監聽器變得唯一，或者您可能想要指定一個逾時時間，超過該時間後監聽器將不再保持唯一。為了實現這一點，您可以在監聽器類別中定義 `uniqueId` 與 `uniqueFor` 屬性或方法。這些方法會接收事件實例，讓您可以使用事件資料來建構回傳值：

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

在上述範例中，`AcquireProductKey` 監聽器是根據授權 ID 來決定唯一性的。因此，在現有監聽器處理完畢之前，任何針對同一授權而分派的新監聽器都將被忽略。這可防止為同一授權獲取重複的產品金鑰。此外，如果現有監聽器在一小時內未被處理，唯一鎖定將被釋放，而另一個具有相同唯一金鑰的監聽器可以被放入佇列。

> [!WARNING]
> 如果您的應用程式從多台 Web 伺服器或容器分派事件，您應確保所有伺服器都與同一個中央快取伺服器通訊，以便 Laravel 能準確判斷監聽器是否唯一。


<a name="keeping-listeners-unique-until-processing-begins"></a>
#### 保持監聽器唯一直到開始處理

預設情況下，唯一監聽器會在處理完成或所有重試嘗試均失敗後被「解鎖」。然而，在某些情況下，您可能希望監聽器在開始處理之前立即解鎖。為了實現這一點，您的監聽器應該實作 `ShouldBeUniqueUntilProcessing` 契約而非 `ShouldBeUnique` 契約：

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
#### 唯一的監聽器鎖定

在幕後，當 `ShouldBeUnique` 監聽器被分派時，Laravel 會嘗試獲取一個使用 `uniqueId` 金鑰的 [鎖定](/docs/{{version}}/cache#atomic-locks)。如果鎖定已被占用，則該監聽器不會被分派。此鎖定會在監聽器處理完成或所有重試嘗試均失敗時被釋放。預設情況下，Laravel 將使用預設的快取驅動程式來獲取此鎖定。然而，如果您希望使用另一個驅動程式來獲取鎖定，您可以定義一個 `uniqueVia` 方法來回傳應使用的快取驅動程式：

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
> 如果您只需要限制監聽器的同時執行數量，請改用 [WithoutOverlapping](/docs/{{version}}/queues#preventing-job-overlaps) 工作中介層。

<a name="handling-failed-jobs"></a>
### 處理失敗的工作

有時您的佇列事件監聽器可能會失敗。如果佇列監聽器超過了由佇列工作執行者 (worker) 所定義的最大嘗試次數，您的監聽器將會呼叫 `failed` 方法。`failed` 方法會接收事件實例以及導致失敗的 `Throwable`：

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

如果您的其中一個佇列監聽器遇到錯誤，您可能不希望它無限次地嘗試重新執行。因此，Laravel 提供了多種方式來指定監聽器可以嘗試的次數或持續時間。

您可以在監聽器類別上使用 `Tries` 屬性來指定監聽器在被視為失敗之前可以嘗試多少次：

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\Attributes\Tries;
use Illuminate\Queue\InteractsWithQueue;

#[Tries(5)]
class SendShipmentNotification implements ShouldQueue
{
    use InteractsWithQueue;

    // ...
}
```

除了定義監聽器在失敗前可嘗試的次數外，您也可以定義一個監聽器不再嘗試重新執行的時間點。這允許監聽器在給定的時間範圍內嘗試任意次數。若要定義監聽器不再嘗試的時間，請在監聽器類別中新增 `retryUntil` 方法。此方法應回傳一個 `DateTimeInterface` 實例：

```php
use DateTimeInterface;

/**
 * Determine the time at which the listener should timeout.
 */
public function retryUntil(): DateTimeInterface
{
    return now()->plus(minutes: 5);
}
```

如果同時定義了 `retryUntil` 和 `tries`，Laravel 會優先採用 `retryUntil` 方法。


<a name="specifying-queued-listener-backoff"></a>
#### 指定佇列監聽器的退避時間

如果您想要設定 Laravel 在重新嘗試遇到例外狀況的監聽器之前應等待多少秒，您可以在監聽器類別上使用 `Backoff` 屬性：

```php
<?php

namespace App\Listeners;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\Attributes\Backoff;

#[Backoff(3)]
class SendShipmentNotification implements ShouldQueue
{
    // ...
}
```

如果您需要更複雜的邏輯來決定監聽器的退避時間 (backoff time)，您可以在監聽器類別上定義 `backoff` 方法：

```php
/**
 * Calculate the number of seconds to wait before retrying the queued listener.
 */
public function backoff(OrderShipped $event): int
{
    return 3;
}
```

您可以透過從 `backoff` 方法回傳一個退避值陣列，來輕鬆地設定「指數級」退避。在此範例中，第一次重新嘗試的延遲為 1 秒，第二次為 5 秒，第三次為 10 秒，如果還有剩餘的嘗試次數，之後的每次重新嘗試延遲均為 10 秒：

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
#### 指定佇列監聽器的最大例外次數

有時您可能希望指定佇列監聽器可以嘗試多次，但如果重新嘗試是由於一定數量的未處理例外所觸發（而非由 `release` 方法直接釋放），則應視為失敗。為了達成此目的，您可以在監聽器類別上使用 `Tries` 和 `MaxExceptions` 屬性：

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\Attributes\MaxExceptions;
use Illuminate\Queue\Attributes\Tries;
use Illuminate\Queue\InteractsWithQueue;

#[Tries(25)]
#[MaxExceptions(3)]
class SendShipmentNotification implements ShouldQueue
{
    use InteractsWithQueue;

    /**
     * Handle the event.
     */
    public function handle(OrderShipped $event): void
    {
        // Process the event...
    }
}
```

在此範例中，監聽器將最多被重新嘗試 25 次。然而，如果監聽器拋出三個未處理的例外，則監聽器將失敗。


<a name="specifying-queued-listener-timeout"></a>
#### 指定佇列監聽器的逾時時間

通常，您大致知道佇列監聽器預計花費的時間。因此，Laravel 允許您指定一個「逾時 (timeout)」值。如果監聽器處理的時間超過了逾時值所指定的秒數，處理該監聽器的工作執行者 (worker) 將會以錯誤結束。您可以使用監聽器類別上的 `Timeout` 屬性來定義監聽器允許執行的最大秒數：

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\Attributes\Timeout;

#[Timeout(120)]
class SendShipmentNotification implements ShouldQueue
{
    // ...
}
```

如果您想要指定監聽器在逾時時應被標記為失敗，您可以使用監聽器類別上的 `FailOnTimeout` 屬性：

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\Attributes\FailOnTimeout;

#[FailOnTimeout]
class SendShipmentNotification implements ShouldQueue
{
    // ...
}
```

<a name="dispatching-events"></a>
## 分派事件

要分派一個事件，您可以呼叫該事件上的靜態 `dispatch` 方法。此方法是由 `Illuminate\Foundation\Events\Dispatchable` trait 提供給事件使用的。傳遞給 `dispatch` 方法的任何引數都將被傳遞至事件的建構子：

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

如果您想要根據條件分派事件，可以使用 `dispatchIf` 與 `dispatchUnless` 方法：

```php
OrderShipped::dispatchIf($condition, $order);

OrderShipped::dispatchUnless($condition, $order);
```

> [!NOTE]
> 在測試時，斷言某些事件已被分派而不需要真正觸發其監聽器會非常有幫助。Laravel 內建的 [測試輔助工具](#testing) 讓這件事變得非常簡單。


<a name="dispatching-events-after-database-transactions"></a>
### 在資料庫交易後分派事件

有時，您可能希望指示 Laravel 僅在目前的資料庫交易提交（commit）之後才分派事件。若要達成此目的，您可以在事件類別中實作 `ShouldDispatchAfterCommit` 介面。

此介面會指示 Laravel 在目前的資料庫交易提交之前不要分派該事件。如果交易失敗，該事件將被捨棄。如果分派事件時沒有正在進行的資料庫交易，該事件將立即被分派：

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

延遲事件允許您將模型事件的分派以及事件監聽器的執行，延後到特定程式碼區塊完成之後才進行。當您需要確保所有相關紀錄在觸發事件監聽器之前都已建立時，這尤其有用。

若要延遲事件，請將一個 Closure 傳遞給 `Event::defer()` 方法：

```php
use App\Models\User;
use Illuminate\Support\Facades\Event;

Event::defer(function () {
    $user = User::create(['name' => 'Victoria Otwell']);

    $user->posts()->create(['title' => 'My first post!']);
});
```

在 Closure 內觸發的所有事件都將在 Closure 執行完畢後分派。這能確保事件監聽器可以存取在延遲執行期間建立的所有相關紀錄。如果 Closure 內發生異常，延遲的事件將不會被分派。

若要僅延遲特定事件，請將事件陣列作為 `defer` 方法的第二個引數：

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
### 編寫事件訂閱者

事件訂閱者是可以在訂閱者類別本身內部訂閱多個事件的類別，允許您在單一類別中定義多個事件處理常式。訂閱者應定義一個 `subscribe` 方法，該方法會接收一個事件分派器 (event dispatcher) 實例。您可以呼叫該分派器的 `listen` 方法來註冊事件監聽器：

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

如果您的事件監聽器方法就定義在訂閱者本身之中，您可能會發現從訂閱者的 `subscribe` 方法回傳一個包含事件與方法名稱的陣列會更方便。Laravel 在註冊事件監聽器時會自動判定訂閱者的類別名稱：

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

編寫完訂閱者後，如果其處理方法遵循 Laravel 的 [事件探索慣例](#event-discovery)，Laravel 將會自動註冊訂閱者內的處理方法。否則，您可以使用 `Event` Facade 的 `subscribe` 方法手動註冊您的訂閱者。通常，這應該在應用程式 `AppServiceProvider` 的 `boot` 方法中完成：

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

當測試分派事件的程式碼時，您可能希望指示 Laravel 不要實際執行事件的監聽器，因為監聽器的程式碼可以獨立於分派對應事件的程式碼直接進行測試。當然，若要測試監聽器本身，您可以在測試中實例化一個監聽器實例並直接呼叫 `handle` 方法。

使用 `Event` Facade 的 `fake` 方法，您可以防止監聽器執行，執行被測試的程式碼，然後使用 `assertDispatched`、`assertNotDispatched` 以及 `assertNothingDispatched` 方法來斷言您的應用程式分派了哪些事件：

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

您可以將 Closure 傳遞給 `assertDispatched` 或 `assertNotDispatched` 方法，以斷言分派的事件通過了給定的「真值測試 (truth test)」。如果至少有一個分派的事件通過了給定的真值測試，則斷言將會成功：

```php
Event::assertDispatched(function (OrderShipped $event) use ($order) {
    return $event->order->id === $order->id;
});
```

如果您只是想斷言某個事件監聽器正在監聽給定的事件，可以使用 `assertListening` 方法：

```php
Event::assertListening(
    OrderShipped::class,
    SendShipmentNotification::class
);
```

> [!WARNING]
> 在呼叫 `Event::fake()` 後，不會有任何事件監聽器被執行。因此，如果您的測試使用了依賴事件的模型工廠 (model factories)，例如在模型的 `creating` 事件期間建立 UUID，您應該在使用了工廠 **之後** 再呼叫 `Event::fake()`。


<a name="faking-a-subset-of-events"></a>
### 模擬部分事件

如果您只想針對特定的一組事件模擬事件監聽器，可以將它們傳遞給 `fake` 或 `fakeFor` 方法：

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

您可以使用 `except` 方法來模擬除了指定的一組事件以外的所有事件：

```php
Event::fake()->except([
    OrderCreated::class,
]);
```


<a name="scoped-event-fakes"></a>
### 範圍限定的事件模擬

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