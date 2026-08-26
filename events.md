# 事件 (Events)

- [簡介](#introduction)
- [產生事件與監聽器](#generating-events-and-listeners)
- [註冊事件與監聽器](#registering-events-and-listeners)
    - [事件自動發掘](#event-discovery)
    - [手動註冊事件](#manually-registering-events)
    - [閉包監聽器](#closure-listeners)
- [定義事件](#defining-events)
- [定義監聽器](#defining-listeners)
- [佇列化事件監聽器](#queued-event-listeners)
    - [手動與佇列互動](#manually-interacting-with-the-queue)
    - [佇列化事件監聽器與資料庫交易](#queued-event-listeners-and-database-transactions)
    - [佇列化監聽器中介層](#queued-listener-middleware)
    - [加密的佇列化監聽器](#encrypted-queued-listeners)
    - [唯一事件監聽器](#unique-event-listeners)
        - [保持監聽器唯一直到開始處理](#keeping-listeners-unique-until-processing-begins)
        - [唯一監聽器鎖定](#unique-listener-locks)
    - [防抖動事件監聽器](#debounced-event-listeners)
    - [處理失敗的任務](#handling-failed-jobs)
- [派發事件](#dispatching-events)
    - [在資料庫交易完成後派發事件](#dispatching-events-after-database-transactions)
    - [延遲事件](#deferring-events)
- [事件訂閱者](#event-subscribers)
    - [撰寫事件訂閱者](#writing-event-subscribers)
    - [註冊事件訂閱者](#registering-event-subscribers)
- [測試](#testing)
    - [偽造部分事件](#faking-a-subset-of-events)
    - [限定作用域的事件偽造](#scoped-event-fakes)

<a name="introduction"></a>
## 簡介

Laravel 的事件機制提供了一套簡單的觀察者模式（Observer Pattern）實作，讓你能訂閱與監聽應用程式中發生的各種事件。事件類別通常儲存在 `app/Events` 目錄中，而它們的監聽器則儲存在 `app/Listeners` 中。如果在應用程式中沒看到這些目錄也不用擔心，因為當你使用 Artisan 主控台指令產生事件與監聽器時，系統會自動幫你建立。

事件是解耦（Decouple）應用程式各個環節的好方法，因為單一事件可以有多個互不依賴的監聽器。例如，你可能希望每次訂單出貨時發送 Slack 通知給使用者。與其將訂單處理程式碼與 Slack 通知程式碼強行耦合，不如觸發一個 `App\Events\OrderShipped` 事件，讓專門的監聽器接收該事件並用來發送 Slack 通知。


<a name="generating-events-and-listeners"></a>
## 產生事件與監聽器

若要快速產生事件與監聽器，你可以使用 `make:event` 與 `make:listener` Artisan 指令：

```shell
php artisan make:event PodcastProcessed

php artisan make:listener SendPodcastNotification --event=PodcastProcessed
```

為了方便起見，你也可以在執行 `make:event` 與 `make:listener` Artisan 指令時不加任何引數。當你這麼做時，Laravel 會自動提示你輸入類別名稱，以及在建立監聽器時提示它應該監聽哪一個事件：

```shell
php artisan make:event

php artisan make:listener
```


<a name="registering-events-and-listeners"></a>
## 註冊事件與監聽器


<a name="event-discovery"></a>
### 事件自動發掘

預設情況下，Laravel 會透過掃描應用程式的 `Listeners` 目錄自動尋找並註冊你的事件監聽器。當 Laravel 發現任何以 `handle` 或 `__invoke` 開頭的監聽器類別方法時，Laravel 會將這些方法註冊為事件監聽器，並監聽該方法簽名中型別提示（Type-hinted）的事件：

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

你可以使用 PHP 的聯集型別（Union types）來監聽多個事件：

```php
/**
 * Handle the event.
 */
public function handle(PodcastProcessed|PodcastPublished $event): void
{
    // ...
}
```

如果你預計將監聽器儲存在不同的目錄或多個目錄中，可以使用應用程式 `bootstrap/app.php` 檔案中的 `withEvents` 方法來指示 Laravel 掃描這些目錄：

```php
->withEvents(discover: [
    __DIR__.'/../app/Domain/Orders/Listeners',
])
```

你可以使用 `*` 字元作為萬用字元，來掃描多個相似目錄中的監聽器：

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
#### 正式環境中的事件自動發掘

為了提升應用程式的執行速度，你應該使用 `optimize` 或 `event:cache` Artisan 指令快取應用程式中所有監聽器的清單（Manifest）。通常，這個指令應該作為應用程式[部署流程](/docs/{{version}}/deployment#optimization)的一部分來執行。此清單將被框架用來加速事件註冊過程。`event:clear` 指令可用於銷毀事件快取。


<a name="dynamic-event-discovery"></a>
#### 動態事件自動發掘

若要動態控制是否發掘特定的監聽器，可以在監聽器類別上實作 `ShouldBeDiscovered` 介面，並定義一個傳回布林值的 `shouldBeDiscovered` 方法。如果該方法傳回 `false`，則在事件發掘期間不會註冊該監聽器：

```php
use Illuminate\Contracts\Events\ShouldBeDiscovered;

class SendPodcastNotification implements ShouldBeDiscovered
{
    /**
     * Handle the event.
     */
    public function handle(PodcastProcessed $event): void
    {
        // ...
    }

    /**
     * Determine if the listener should be discovered.
     */
    public static function shouldBeDiscovered(): bool
    {
        return app()->environment('production');
    }
}
```


<a name="manually-registering-events"></a>
### 手動註冊事件

使用 `Event` Facade，你可以在應用程式 `AppServiceProvider` 的 `boot` 方法中，手動註冊事件及其對應的監聽器：

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
### 閉包監聽器

通常監聽器會被定義為類別；不過，你也可以在應用程式 `AppServiceProvider` 的 `boot` 方法中手動註冊基於閉包的事件監聽器：

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

註冊基於閉包的事件監聽器時，你可以將監聽器閉包包裹在 `Illuminate\Events\queueable` 函式中，以指示 Laravel 使用[佇列](/docs/{{version}}/queues)來執行該監聽器：

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

如同佇列任務一樣，你可以使用 `onConnection`、`onQueue` 與 `delay` 方法來自訂佇列化監聽器的執行方式：

```php
Event::listen(queueable(function (PodcastProcessed $event) {
    // ...
})->onConnection('redis')->onQueue('podcasts')->delay(now()->plus(seconds: 10)));
```

如果你想處理匿名佇列化監聽器的失敗情況，可以在定義 `queueable` 監聽器時傳入一個閉包給 `catch` 方法。該閉包將接收事件實例以及導致監聽器失敗的 `Throwable` 實例：

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

你也可以使用 `*` 字元作為萬用字元參數來註冊監聽器，讓你能在同一個監聽器中捕捉多個事件。萬用字元監聽器會接收事件名稱作為第一個引數，並接收整個事件資料陣列作為第二個引數：

```php
Event::listen('event.*', function (string $eventName, array $data) {
    // ...
});
```

<a name="defining-events"></a>
## 定義事件

事件類別本質上是一個資料容器，用於保存與該事件相關的資訊。例如，假設我們的 `App\Events\OrderShipped` 事件會接收一個 [Eloquent ORM](/docs/{{version}}/eloquent) 物件：

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

如你所見，這個事件類別不包含任何邏輯。它純粹是購買訂單的 `App\Models\Order` 實例容器。當事件物件被使用 PHP 的 `serialize` 函式序列化時（例如使用[佇列化監聽器](#queued-event-listeners)時），事件所使用的 `SerializesModels` trait 將會優雅地序列化任何 Eloquent 模型。


<a name="defining-listeners"></a>
## 定義監聽器

接下來，讓我們來看看範例事件的監聽器。事件監聽器會在它們的 `handle` 方法中接收事件實例。當執行 `make:listener` Artisan 指令並附帶 `--event` 選項時，將會自動匯入適當的事件類別，並在 `handle` 方法中對該事件進行型別提示。在 `handle` 方法內，你可以執行任何回應事件所需的動作：

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
> 你的事件監聽器也可以在建構子中對其所需的任何依賴進行型別提示。所有事件監聽器都會透過 Laravel 的[服務容器](/docs/{{version}}/container)來解析，因此依賴項將會自動被注入。


<a name="stopping-the-propagation-of-an-event"></a>
#### 停止事件繼續傳播

有時，你可能希望停止將事件傳播給其他監聽器。你可以透過在監聽器的 `handle` 方法中回傳 `false` 來達到這個目的。

<a name="queued-event-listeners"></a>
## 佇列化事件監聽器

如果您的監聽器即將執行較慢的任務（例如傳送電子郵件或發送 HTTP 請求），將監聽器佇列化會相當有幫助。在使用佇列化監聽器之前，請確保已[設定您的佇列](/docs/{{version}}/queues)，並在您的伺服器或本機開發環境中啟動佇列 Worker。

若要指定監聽器應進入佇列，請將 `ShouldQueue` 介面新增至監聽器類別。由 `make:listener` Artisan 指令產生的監聽器已經將此介面匯入到目前的命名空間中，因此您可以立即使用：

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

就是這樣！現在，當由這個監聽器處理的事件被派發時，事件派發器會使用 Laravel 的[佇列系統](/docs/{{version}}/queues)自動將該監聽器放入佇列中。如果在佇列執行監聽器時沒有拋出任何例外，佇列任務將會在處理完成後自動被刪除。

<a name="customizing-the-queue-connection-queue-name"></a>
#### 自訂佇列連線、名稱與延遲

如果您想自訂事件監聽器的佇列連線、佇列名稱或佇列延遲時間，您可以在監聽器類別上使用 `Connection`、`Queue` 和 `Delay` 屬性（Attributes）：

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
如果您想要在執行階段（Runtime）動態定義監聽器的佇列連線、佇列名稱或延遲時間，您可以在監聽器上定義 `viaConnection`、`viaQueue` 或 `withDelay` 方法：

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

有時候，您可能需要根據僅在執行階段才能取得的資料來判斷監聽器是否應該放入佇列。為此，可以在監聽器中新增 `shouldQueue` 方法來判斷是否該將監聽器佇列化。如果 `shouldQueue` 方法傳回 `false`，該監聽器就不會被放入佇列：

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

如果您需要手動存取監聽器底層佇列任務的 `delete` 和 `release` 方法，您可以使用 `Illuminate\Queue\InteractsWithQueue` Trait。在產生的監聽器中預設已匯入此 Trait，並提供對這些方法的存取：

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

當佇列化監聽器在資料庫交易（Database Transactions）內被派發時，它們可能會在資料庫交易提交（Commit）之前就被佇列處理。發生這種情況時，您在資料庫交易期間對 Model 或資料庫紀錄所做的任何更新，可能都尚未反映在資料庫中。此外，在交易內建立的任何 Model 或資料庫紀錄也可能還不存在於資料庫中。如果您的監聽器依賴這些 Model，則當處理派發該佇列化監聽器的任務時，可能會發生意外錯誤。

如果您的佇列連線設定選項 `after_commit` 設定為 `false`，您仍可透過在監聽器類別上實作 `ShouldQueueAfterCommit` 介面，來指定特定的佇列化監聽器應在所有開啟的資料庫交易提交後才派發：

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
> 若要瞭解更多解決這些問題的詳細資訊，請參閱關於[佇列任務與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的說明文件。

<a name="queued-listener-middleware"></a>
### 佇列化監聽器中介層

佇列化監聽器也可以使用[任務中介層](/docs/{{version}}/queues#job-middleware)。任務中介層允許您在佇列化監聽器的執行前後包覆自訂邏輯，從而減少監聽器內部的重複程式碼。建立任務中介層後，可以透過在監聽器的 `middleware` 方法中傳回它們，將其附加到監聽器上：

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

Laravel 允許您透過[加密](/docs/{{version}}/encryption)來確保佇列化監聽器資料的隱私性與完整性。若要開始使用，只需將 `ShouldBeEncrypted` 介面新增至監聽器類別即可。一旦將此介面新增至類別，Laravel 就會在將您的監聽器推送到佇列之前自動對其進行加密：

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
### 唯一事件監聽器

> [!WARNING]
> 唯一監聽器需要支援[鎖定](/docs/{{version}}/cache#atomic-locks)的快取驅動。目前，`memcached`、`redis`、`dynamodb`、`database`、`file` 與 `array` 快取驅動皆支援原子鎖。

有時，您可能希望確保在任何時間點，佇列中都只有一個特定監聽器的實例。您可以透過在監聽器類別上實作 `ShouldBeUnique` 介面來做到這一點：

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

在上述範例中，`AcquireProductKey` 監聽器是唯一的。因此，如果佇列中已經存在另一個尚未處理完成的監聽器實例，則不會將新的監聽器放入佇列。這可以確保每個授權只會取得一個產品金鑰，即使授權在短時間內被連續儲存多次也是如此。

在某些情況下，您可能希望定義一個特定的「鍵值 (key)」讓監聽器保持唯一，或者您可能希望指定一個逾時時間，超過該時間後監聽器就不再保持唯一。若要實現此目的，您可以在監聽器類別上定義 `uniqueId` 和 `uniqueFor` 屬性或方法。這些方法會接收事件實例，讓您能夠使用事件資料來建立回傳值：

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

在上述範例中，`AcquireProductKey` 監聽器是以授權 ID 作為唯一的依據。因此，在現有的監聽器處理完成之前，針對相同授權所派發的任何新監聽器都會被忽略。這可以防止為相同的授權重複取得產品金鑰。此外，如果現有的監聽器在一小時之內未被處理，唯一鎖定將會被釋放，並且可以將具有相同唯一鍵值的另一個監聽器放入佇列。

> [!WARNING]
> 如果您的應用程式是從多台 Web 伺服器或容器派發事件，您應該確保所有伺服器都與同一個中央快取伺服器進行通訊，以便 Laravel 能夠準確判斷監聽器是否唯一。


<a name="keeping-listeners-unique-until-processing-begins"></a>
#### 保持監聽器唯一直到開始處理

預設情況下，唯一監聽器會在處理完成或所有重試嘗試均失敗後才會「解鎖」。然而，在某些情況下，您可能希望監聽器在開始處理前立即解鎖。若要實現此目的，您的監聽器應該實作 `ShouldBeUniqueUntilProcessing` 契約，而不是 `ShouldBeUnique` 契約：

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

在幕後，當派發實作 `ShouldBeUnique` 的監聽器時，Laravel 會嘗試取得帶有 `uniqueId` 鍵值的[鎖定](/docs/{{version}}/cache#atomic-locks)。如果鎖定已被佔用，則不會派發該監聽器。當監聽器完成處理或所有重試嘗試均失敗時，此鎖定就會被釋放。預設情況下，Laravel 會使用預設的快取驅動來取得此鎖定。但是，如果您希望使用另一個驅動來取得鎖定，您可以定義一個 `uniqueVia` 方法，並回傳應該使用的快取驅動：

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
> 如果您只需要限制監聽器的同時處理數量，請改用 [WithoutOverlapping](/docs/{{version}}/queues#preventing-job-overlaps) 任務中介層。


<a name="debounced-event-listeners"></a>
### 防抖動事件監聽器

有時，您可能只想處理在短時間內重複派發的事件中的最新實例。您可以透過在佇列化監聽器上加入 `DebounceFor` 屬性來做到這一點：

```php
<?php

namespace App\Listeners;

use App\Events\ProductUpdated;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\Attributes\DebounceFor;

#[DebounceFor(30)]
class UpdateProductSearchIndex implements ShouldQueue
{
    /**
     * Handle the event.
     */
    public function handle(ProductUpdated $event): void
    {
        // Update the product's search index...
    }

    /**
     * Get the debounce ID for the listener.
     */
    public function debounceId(ProductUpdated $event): string
    {
        return (string) $event->product->getKey();
    }
}
```

在上述範例中，如果在 `30` 秒內針對同一個產品重複派發 `ProductUpdated` 事件，將會防抖動該監聽器，從而只處理最新的事件。不同的防抖動 ID 是獨立處理的。

如果您想限制頻繁派發的事件延遲監聽器的最長時間，您可以為 `DebounceFor` 屬性提供 `maxWait` 引數：

```php
#[DebounceFor(30, maxWait: 120)]
class UpdateProductSearchIndex implements ShouldQueue
{
    // ...
}
```

您可以透過在監聽器上定義 `debounceVia` 方法來自訂用於防抖動追蹤的快取儲存。該方法接收事件實例，並應回傳一個快取儲存庫 (Cache Repository)：

```php
use Illuminate\Contracts\Cache\Repository;
use Illuminate\Support\Facades\Cache;

public function debounceVia(ProductUpdated $event): Repository
{
    return Cache::driver('redis');
}
```

防抖動監聽器與唯一監聽器是互斥的。使用 `DebounceFor` 屬性的監聽器不應該實作 `ShouldBeUnique`。

> [!WARNING]
> 如果您的應用程式是從多台 Web 伺服器或容器派發事件，您應該確保所有伺服器都與同一個中央快取伺服器進行通訊。

<a name="handling-failed-jobs"></a>
### 處理失敗的任務

有時您的佇列化事件監聽器可能會失敗。若佇列化監聽器超過了佇列 Worker 所定義的最大嘗試次數，系統將會呼叫您監聽器上的 `failed` 方法。`failed` 方法會接收事件實例以及導致失敗的 `Throwable` 實例：

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

當您的佇列化監聽器遇到錯誤時，您可能不希望它無限制地持續重試。因此，Laravel 提供了多種方式來指定監聽器可以嘗試的次數或時間長度。

您可以在監聽器類別上使用 `Tries` 屬性，以指定在被視為失敗之前，該監聽器可以嘗試的次數：

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

除了定義監聽器在失敗前可以嘗試的次數之外，您也可以定義一個時間點，超過該時間點後就不再嘗試執行該監聽器。這允許監聽器在給定的時間範圍內進行任意次數的嘗試。若要定義監聽器不再嘗試的時間點，請在您的監聽器類別中新增 `retryUntil` 方法。該方法應回傳一個 `DateTimeInterface` 實例：

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

若同時定義了 `retryUntil` 與 `tries`，Laravel 會優先採用 `retryUntil` 方法。


<a name="specifying-queued-listener-backoff"></a>
#### 指定佇列化監聽器的重試延遲時間

如果您想設定當監聽器遇到例外時，Laravel 在重試前應該等待幾秒鐘，可以在監聽器類別上使用 `Backoff` 屬性：

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

如果您需要更複雜的邏輯來決定監聽器的重試延遲時間，可以在監聽器類別中定義 `backoff` 方法：

```php
/**
 * Calculate the number of seconds to wait before retrying the queued listener.
 */
public function backoff(OrderShipped $event): int
{
    return 3;
}
```

您也可以從 `backoff` 方法回傳一個延遲數值陣列，輕鬆設定「指數退避 (exponential backoff)」時間。在此範例中，若是還有剩餘的嘗試次數，第一次重試的延遲時間為 1 秒，第二次重試為 5 秒，第三次重試為 10 秒，之後的每次重試皆為 10 秒：

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

有時您可能希望佇列化監聽器可以嘗試許多次，但若重試是由特定次數未處理的例外所觸發（相對於直接透過 `release` 方法釋放），則該監聽器應該直接失敗。若要達成此目的，您可以在監聽器類別上使用 `Tries` 與 `MaxExceptions` 屬性：

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

在此範例中，監聽器最多會重試 25 次。然而，如果監聽器拋出了 3 次未處理的例外，監聽器就會標記為失敗。


<a name="specifying-queued-listener-timeout"></a>
#### 指定佇列化監聽器的逾時時間

通常，您會大致了解佇列化監聽器預期需要執行多久。因此，Laravel 允許您指定一個「逾時 (timeout)」值。如果監聽器的處理時間超過了逾時值所設定的秒數，處理該監聽器的 Worker 將會發生錯誤並結束執行。您可以使用監聽器類別上的 `Timeout` 屬性來定義允許監聽器執行的最大秒數：

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

如果您想表明監聽器在逾時時應被標記為失敗，可以在監聽器類別上使用 `FailOnTimeout` 屬性：

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
## 派發事件

要派發事件，你可以對事件呼叫靜態的 `dispatch` 方法。這個方法是由 `Illuminate\Foundation\Events\Dispatchable` trait 提供給事件的。任何傳遞給 `dispatch` 方法的引數都會傳遞給事件的建構函式：

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

如果你想在滿足特定條件時才派發事件，可以使用 `dispatchIf` 和 `dispatchUnless` 方法：

```php
OrderShipped::dispatchIf($condition, $order);

OrderShipped::dispatchUnless($condition, $order);
```

> [!NOTE]
> 在測試時，如果只想斷言特定事件是否已被派發，而不實際觸發其監聽器，這會非常有幫助。Laravel 的[內建測試輔助函式](#testing)讓這件事變得輕而易舉。


<a name="dispatching-events-after-database-transactions"></a>
### 在資料庫交易完成後派發事件

有時，你可能希望指示 Laravel 僅在當前的資料庫交易提交（Commit）後才派發事件。為此，你可以在事件類別上實作 `ShouldDispatchAfterCommit` 介面。

這個介面指示 Laravel 在當前資料庫交易提交之前不要派發該事件。如果交易失敗，該事件將被丟棄。如果派發事件時沒有正在進行中的資料庫交易，則會立即派發該事件：

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

延遲事件允許你將 Model 事件的派發與事件監聽器的執行延後，直到特定程式區塊執行完成為止。當你需要確保所有相關紀錄都已建立，才觸發事件監聽器時，這點特別有用。

要延遲事件，請傳遞一個閉包給 `Event::defer()` 方法：

```php
use App\Models\User;
use Illuminate\Support\Facades\Event;

Event::defer(function () {
    $user = User::create(['name' => 'Victoria Otwell']);

    $user->posts()->create(['title' => 'My first post!']);
});
```

在閉包內觸發的所有事件都將在閉包執行完畢後派發。這能確保事件監聽器可以存取到在延遲執行期間建立的所有相關紀錄。如果閉包內發生例外，延遲的事件將不會被派發。

若要僅延遲特定事件，請將事件陣列作為第二個引數傳遞給 `defer` 方法：

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

事件訂閱者是在訂閱者類別本身內部訂閱多個事件的類別，允許你在單一類別中定義多個事件處理常式。訂閱者應該定義一個 `subscribe` 方法，該方法會接收一個事件派發器實例。你可以對給定的派發器呼叫 `listen` 方法來註冊事件監聽器：

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

如果你的事件監聽器方法定義在訂閱者本身之中，你可能會發現直接從訂閱者的 `subscribe` 方法回傳一個包含事件與方法名稱的陣列會更加方便。Laravel 在註冊事件監聽器時會自動判定訂閱者的類別名稱：

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

撰寫完訂閱者後，如果訂閱者內部的處理常式方法符合 Laravel 的[事件自動發掘慣例](#event-discovery)，Laravel 將會自動註冊這些方法。否則，你可以使用 `Event` Facade 的 `subscribe` 方法手動註冊訂閱者。通常，這應該在應用程式的 `AppServiceProvider` 中的 `boot` 方法內完成：

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

在測試會派發事件的程式碼時，您可能希望指示 Laravel 不要實際執行該事件的監聽器，因為監聽器的程式碼可以獨立於派發事件的程式碼之外直接進行測試。當然，若要測試監聽器本身，您可以在測試中實例化監聽器物件並直接呼叫 `handle` 方法。

使用 `Event` Facade 的 `fake` 方法，您可以阻止監聽器執行、執行受測程式碼，然後使用 `assertDispatched`、`assertNotDispatched` 及 `assertNothingDispatched` 方法來斷言應用程式派發了哪些事件：

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

您可以傳遞一個閉包給 `assertDispatched` 或 `assertNotDispatched` 方法，以斷言派發的事件是否通過給定的「真值測試 (truth test)」。若至少有一個被派發的事件通過了給定的真值測試，斷言就會成功：

```php
Event::assertDispatched(function (OrderShipped $event) use ($order) {
    return $event->order->id === $order->id;
});
```

如果您只是想斷言某個事件監聽器有在監聽特定的事件，可以使用 `assertListening` 方法：

```php
Event::assertListening(
    OrderShipped::class,
    SendShipmentNotification::class
);
```

> [!WARNING]
> 呼叫 `Event::fake()` 之後，將不會執行任何事件監聽器。因此，如果您的測試使用了依賴事件的模型工廠（例如在模型的 `creating` 事件期間建立 UUID），您應該在使用了工廠**之後**才呼叫 `Event::fake()`。


<a name="faking-a-subset-of-events"></a>
### 偽造部分事件

如果您只想偽造特定一組事件的事件監聽器，可以將它們傳遞給 `fake` 或 `fakeFor` 方法：

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

您可以使用 `except` 方法來偽造除了指定的一組事件之外的所有事件：

```php
Event::fake()->except([
    OrderCreated::class,
]);
```


<a name="scoped-event-fakes"></a>
### 限定作用域的事件偽造

如果您只想在測試的其中一部分偽造事件監聽器，可以使用 `fakeFor` 方法：

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