# 事件

- [簡介](#introduction)
- [產生事件與監聽器](#generating-events-and-listeners)
- [註冊事件與監聽器](#registering-events-and-listeners)
    - [事件探索](#event-discovery)
    - [手動註冊事件](#manually-registering-events)
    - [閉包監聽器](#closure-listeners)
- [定義事件](#defining-events)
- [定義監聽器](#defining-listeners)
- [佇列事件監聽器](#queued-event-listeners)
    - [手動與佇列互動](#manually-interacting-with-the-queue)
    - [佇列事件監聽器與資料庫交易](#queued-event-listeners-and-database-transactions)
    - [處理失敗的任務](#handling-failed-jobs)
- [分派事件](#dispatching-events)
    - [在資料庫交易後分派事件](#dispatching-events-after-database-transactions)
- [事件訂閱者](#event-subscribers)
    - [撰寫事件訂閱者](#writing-event-subscribers)
    - [註冊事件訂閱者](#registering-event-subscribers)
- [測試](#testing)
    - [偽造部分事件](#faking-a-subset-of-events)
    - [範圍事件偽造](#scoped-event-fakes)

<a name="introduction"></a>
## 簡介

Laravel 的事件提供了一個簡單的觀察者模式實作，讓您能夠訂閱並監聽應用程式中發生的各種事件。事件類別通常儲存在 `app/Events` 目錄中，而其監聽器則儲存在 `app/Listeners` 中。如果您在應用程式中沒有看到這些目錄，請不用擔心，因為當您使用 Artisan 主控台指令產生事件和監聽器時，它們會自動為您建立。

事件是解耦應用程式各個層面的絕佳方式，因為單一事件可以有多個彼此不依賴的監聽器。例如，您可能希望每次訂單出貨時都向使用者傳送 Slack 通知。與其將訂單處理程式碼與 Slack 通知程式碼耦合，您可以觸發一個 `App\Events\OrderShipped` 事件，然後監聽器可以接收該事件並用來分派 Slack 通知。

<a name="generating-events-and-listeners"></a>
## 產生事件與監聽器

若要快速產生事件與監聽器，您可以使用 `make:event` 和 `make:listener` Artisan 指令：

```shell
php artisan make:event PodcastProcessed

php artisan make:listener SendPodcastNotification --event=PodcastProcessed
```

為了方便起見，您也可以不帶額外引數地呼叫 `make:event` 和 `make:listener` Artisan 指令。當您這樣做時，Laravel 會自動提示您輸入類別名稱，並且在建立監聽器時，會提示您輸入它應該監聽的事件：

```shell
php artisan make:event

php artisan make:listener
```

<a name="registering-events-and-listeners"></a>
## 註冊事件與監聽器

<a name="event-discovery"></a>
### 事件探索

預設情況下，Laravel 會透過掃描應用程式的 `Listeners` 目錄來自動尋找並註冊您的事件監聽器。當 Laravel 找到任何以 `handle` 或 `__invoke` 開頭的監聽器類別方法時，Laravel 會將這些方法註冊為該方法簽章中型別提示的事件的事件監聽器：

    use App\Events\PodcastProcessed;

    class SendPodcastNotification
    {
        /**
         * Handle the given event.
         */
        public function handle(PodcastProcessed $event): void
        {
            // ...
        }
    }

您可以使用 PHP 的聯集型別來監聽多個事件：

    /**
     * Handle the given event.
     */
    public function handle(PodcastProcessed|PodcastPublished $event): void
    {
        // ...
    }

如果您打算將監聽器儲存在不同的目錄或多個目錄中，您可以使用應用程式 `bootstrap/app.php` 檔案中的 `withEvents` 方法來指示 Laravel 掃描這些目錄：

    ->withEvents(discover: [
        __DIR__.'/../app/Domain/Orders/Listeners',
    ])

您可以使用 `*` 字元作為萬用字元來掃描多個類似目錄中的監聽器：

    ->withEvents(discover: [
        __DIR__.'/../app/Domain/*/Listeners',
    ])

`event:list` 指令可用於列出應用程式中所有已註冊的監聽器：

```shell
php artisan event:list
```

<a name="event-discovery-in-production"></a>
#### 生產環境中的事件探索

為了提高應用程式的速度，您應該使用 `optimize` 或 `event:cache` Artisan 指令來快取應用程式所有監聽器的清單。通常，此指令應作為應用程式[部署流程](/docs/{{version}}/deployment#optimization)的一部分執行。此清單將由框架用於加速事件註冊過程。`event:clear` 指令可用於銷毀事件快取。

<a name="manually-registering-events"></a>
### 手動註冊事件

使用 `Event` Facade，您可以在應用程式 `AppServiceProvider` 的 `boot` 方法中手動註冊事件及其對應的監聽器：

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

`event:list` 指令可用於列出應用程式中所有已註冊的監聽器：

```shell
php artisan event:list
```

<a name="closure-listeners"></a>
### 閉包監聽器

通常，監聽器被定義為類別；但是，您也可以在應用程式 `AppServiceProvider` 的 `boot` 方法中手動註冊基於閉包的事件監聽器：

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

<a name="queuable-anonymous-event-listeners"></a>
#### 可佇列的匿名事件監聽器

當註冊基於閉包的事件監聽器時，您可以將監聽器閉包包裝在 `Illuminate\Events\queueable` 函數中，以指示 Laravel 使用[佇列](/docs/{{version}}/queues)執行監聽器：

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

與佇列任務一樣，您可以使用 `onConnection`、`onQueue` 和 `delay` 方法來自訂佇列監聽器的執行：

    Event::listen(queueable(function (PodcastProcessed $event) {
        // ...
    })->onConnection('redis')->onQueue('podcasts')->delay(now()->addSeconds(10)));

如果您想處理匿名佇列監聽器失敗的情況，您可以在定義 `queueable` 監聽器時向 `catch` 方法提供一個閉包。此閉包將接收事件實例和導致監聽器失敗的 `Throwable` 實例：

    use App\Events\PodcastProcessed;
    use function Illuminate\Events\queueable;
    use Illuminate\Support\Facades\Event;
    use Throwable;

    Event::listen(queueable(function (PodcastProcessed $event) {
        // ...
    })->catch(function (PodcastProcessed $event, Throwable $e) {
        // The queued listener failed...
    }));

<a name="wildcard-event-listeners"></a>
#### 萬用字元事件監聽器

您也可以使用 `*` 字元作為萬用字元參數來註冊監聽器，讓您可以在同一個監聽器上捕捉多個事件。萬用字元監聽器會將事件名稱作為第一個引數，並將整個事件資料陣列作為第二個引數：

    Event::listen('event.*', function (string $eventName, array $data) {
        // ...
    });

<a name="defining-events"></a>
## 定義事件

事件類別本質上是一個資料容器，它包含與事件相關的資訊。例如，假設 `App\Events\OrderShipped` 事件接收一個 [Eloquent ORM](/docs/{{version}}/eloquent) 物件：

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

如您所見，此事件類別不包含任何邏輯。它是一個包含已購買的 `App\Models\Order` 實例的容器。事件使用的 `SerializesModels` Trait 會在事件物件使用 PHP 的 `serialize` 函數序列化時（例如在使用[佇列監聽器](#queued-event-listeners)時）優雅地序列化任何 Eloquent 模型。

<a name="defining-listeners"></a>
## 定義監聽器

接下來，讓我們看看範例事件的監聽器。事件監聽器在其 `handle` 方法中接收事件實例。`make:listener` Artisan 指令在帶有 `--event` 選項呼叫時，會自動匯入適當的事件類別並在 `handle` 方法中型別提示事件。在 `handle` 方法中，您可以執行任何必要的動作來回應事件：

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

> [!NOTE]
> 您的事件監聽器也可以在其建構函式上型別提示它們所需的任何依賴。所有事件監聽器都透過 Laravel [服務容器](/docs/{{version}}/container)解析，因此依賴將自動注入。

<a name="stopping-the-propagation-of-an-event"></a>
#### 停止事件傳播

有時，您可能希望停止事件向其他監聽器傳播。您可以透過從監聽器的 `handle` 方法返回 `false` 來實現。

<a name="queued-event-listeners"></a>
## 佇列事件監聽器

如果您的監聽器將執行緩慢的任務，例如傳送電子郵件或發出 HTTP 請求，那麼將監聽器排入佇列會很有益處。在使用佇列監聽器之前，請務必[設定您的佇列](/docs/{{version}}/queues)並在您的伺服器或本地開發環境中啟動一個佇列工作者。

若要指定監聽器應排入佇列，請將 `ShouldQueue` 介面新增到監聽器類別。`make:listener` Artisan 指令產生的監聽器已經將此介面匯入到目前的命名空間中，因此您可以立即使用它：

    <?php

    namespace App\Listeners;

    use App\Events\OrderShipped;
    use Illuminate\Contracts\Queue\ShouldQueue;

    class SendShipmentNotification implements ShouldQueue
    {
        // ...
    }

就這樣！現在，當此監聽器處理的事件被分派時，監聽器將由事件分派器使用 Laravel 的[佇列系統](/docs/{{version}}/queues)自動排入佇列。如果在佇列執行監聽器時沒有拋出任何例外，則佇列任務在處理完成後將自動刪除。

<a name="customizing-the-queue-connection-queue-name"></a>
#### 自訂佇列連線、佇列名稱與延遲

如果您想自訂事件監聽器的佇列連線、佇列名稱或佇列延遲時間，您可以在監聽器類別上定義 `$connection`、`$queue` 或 `$delay` 屬性：

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

如果您想在執行時定義監聽器的佇列連線、佇列名稱或延遲，您可以在監聽器上定義 `viaConnection`、`viaQueue` 或 `withDelay` 方法：

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

<a name="conditionally-queueing-listeners"></a>
#### 有條件地將監聽器排入佇列

有時，您可能需要根據某些僅在執行時可用的資料來判斷監聽器是否應排入佇列。為此，可以在監聽器中新增 `shouldQueue` 方法來判斷監聽器是否應排入佇列。如果 `shouldQueue` 方法返回 `false`，則監聽器將不會排入佇列：

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

<a name="manually-interacting-with-the-queue"></a>
### 手動與佇列互動

如果您需要手動存取監聽器底層佇列任務的 `delete` 和 `release` 方法，您可以使用 `Illuminate\Queue\InteractsWithQueue` Trait。此 Trait 預設會匯入到產生的監聽器中，並提供對這些方法的存取：

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

<a name="queued-event-listeners-and-database-transactions"></a>
### 佇列事件監聽器與資料庫交易

當佇列監聽器在資料庫交易中分派時，它們可能會在資料庫交易提交之前由佇列處理。當這種情況發生時，您在資料庫交易期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫記錄可能不存在於資料庫中。如果您的監聽器依賴這些模型，則在處理分派佇列監聽器的任務時可能會發生意外錯誤。

如果您的佇列連線的 `after_commit` 設定選項設定為 `false`，您仍然可以透過在監聽器類別上實作 `ShouldQueueAfterCommit` 介面來指示特定佇列監聽器應在所有開啟的資料庫交易提交後分派：

    <?php

    namespace App\Listeners;

    use Illuminate\Contracts\Queue\ShouldQueueAfterCommit;
    use Illuminate\Queue\InteractsWithQueue;

    class SendShipmentNotification implements ShouldQueueAfterCommit
    {
        use InteractsWithQueue;
    }

> [!NOTE]
> 若要了解如何解決這些問題，請查閱有關[佇列任務與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的文件。

<a name="handling-failed-jobs"></a>
### 處理失敗的任務

有時您的佇列事件監聽器可能會失敗。如果佇列監聽器超過佇列工作者定義的最大嘗試次數，則會呼叫監聽器上的 `failed` 方法。`failed` 方法接收事件實例和導致失敗的 `Throwable`：

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

<a name="specifying-queued-listener-maximum-attempts"></a>
#### 指定佇列監聽器最大嘗試次數

如果您的其中一個佇列監聽器遇到錯誤，您可能不希望它無限期地重試。因此，Laravel 提供了多種方式來指定監聽器可以嘗試的次數或時間長度。

您可以在監聽器類別上定義 `$tries` 屬性，以指定監聽器在被視為失敗之前可以嘗試的次數：

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

作為定義監聽器在失敗之前可以嘗試多少次的一種替代方法，您可以定義監聽器不應再嘗試的時間。這允許監聽器在給定時間範圍內嘗試任意次數。若要定義監聽器不應再嘗試的時間，請在監聽器類別中新增 `retryUntil` 方法。此方法應返回一個 `DateTime` 實例：

    use DateTime;

    /**
     * Determine the time at which the listener should timeout.
     */
    public function retryUntil(): DateTime
    {
        return now()->addMinutes(5);
    }

<a name="specifying-queued-listener-backoff"></a>
#### 指定佇列監聽器退避時間

如果您想設定 Laravel 在重試遇到例外狀況的監聽器之前應等待的秒數，您可以透過在監聽器類別上定義 `backoff` 屬性來實現：

    /**
     * The number of seconds to wait before retrying the queued listener.
     *
     * @var int
     */
    public $backoff = 3;

如果您需要更複雜的邏輯來決定監聽器的退避時間，您可以在監聽器類別上定義 `backoff` 方法：

    /**
     * Calculate the number of seconds to wait before retrying the queued listener.
     */
    public function backoff(): int
    {
        return 3;
    }

您可以透過從 `backoff` 方法返回一個退避值陣列來輕鬆設定「指數」退避。在此範例中，如果還有更多嘗試次數，則第一次重試的延遲為 1 秒，第二次重試為 5 秒，第三次重試為 10 秒，之後每次重試均為 10 秒：

    /**
     * Calculate the number of seconds to wait before retrying the queued listener.
     *
     * @return array<int, int>
     */
    public function backoff(): array
    {
        return [1, 5, 10];
    }

<a name="dispatching-events"></a>
## 分派事件

若要分派事件，您可以呼叫事件上的靜態 `dispatch` 方法。此方法透過 `Illuminate\Foundation\Events\Dispatchable` Trait 在事件上可用。傳遞給 `dispatch` 方法的任何引數都將傳遞給事件的建構函式：

    <?php

    namespace App\Http\Controllers;

    use App\Events\OrderShipped;
    use App\Http\Controllers\Controller;
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

如果您想有條件地分派事件，您可以使用 `dispatchIf` 和 `dispatchUnless` 方法：

    OrderShipped::dispatchIf($condition, $order);

    OrderShipped::dispatchUnless($condition, $order);

> [!NOTE]
> 在測試時，斷言某些事件已分派而無需實際觸發其監聽器會很有幫助。Laravel 的[內建測試輔助工具](#testing)使其變得輕而易舉。

<a name="dispatching-events-after-database-transactions"></a>
### 在資料庫交易後分派事件

有時，您可能希望指示 Laravel 僅在活動資料庫交易提交後才分派事件。為此，您可以在事件類別上實作 `ShouldDispatchAfterCommit` 介面。

此介面指示 Laravel 在目前資料庫交易提交之前不分派事件。如果交易失敗，事件將被丟棄。如果在分派事件時沒有進行中的資料庫交易，事件將立即分派：

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

<a name="event-subscribers"></a>
## 事件訂閱者

<a name="writing-event-subscribers"></a>
### 撰寫事件訂閱者

事件訂閱者是可以在訂閱者類別本身內部訂閱多個事件的類別，讓您可以在單一類別中定義多個事件處理器。訂閱者應定義一個 `subscribe` 方法，該方法將傳入一個事件分派器實例。您可以呼叫給定分派器上的 `listen` 方法來註冊事件監聽器：

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

如果您的事件監聽器方法定義在訂閱者本身內部，您可能會發現從訂閱者的 `subscribe` 方法返回一個事件和方法名稱的陣列更方便。Laravel 在註冊事件監聽器時會自動判斷訂閱者的類別名稱：

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

<a name="registering-event-subscribers"></a>
### 註冊事件訂閱者

撰寫訂閱者後，如果它們遵循 Laravel 的[事件探索慣例](#event-discovery)，Laravel 將自動註冊訂閱者中的處理器方法。否則，您可以使用 `Event` Facade 的 `subscribe` 方法手動註冊您的訂閱者。通常，這應該在應用程式 `AppServiceProvider` 的 `boot` 方法中完成：

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

<a name="testing"></a>
## 測試

在測試分派事件的程式碼時，您可能希望指示 Laravel 不要實際執行事件的監聽器，因為監聽器的程式碼可以直接且獨立於分派相應事件的程式碼進行測試。當然，若要測試監聽器本身，您可以實例化一個監聽器實例並直接在測試中呼叫 `handle` 方法。

使用 `Event` Facade 的 `fake` 方法，您可以阻止監聽器執行，執行受測程式碼，然後使用 `assertDispatched`、`assertNotDispatched` 和 `assertNothingDispatched` 方法斷言應用程式分派了哪些事件：

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

        // Assert an event was not dispatched...
        Event::assertNotDispatched(OrderFailedToShip::class);

        // Assert that no events were dispatched...
        Event::assertNothingDispatched();
    }
}
```

您可以將閉包傳遞給 `assertDispatched` 或 `assertNotDispatched` 方法，以斷言分派的事件通過給定的「真實性測試」。如果至少有一個事件通過給定的真實性測試，則斷言將成功：

    Event::assertDispatched(function (OrderShipped $event) use ($order) {
        return $event->order->id === $order->id;
    });

如果您只想斷言事件監聽器正在監聽給定事件，您可以使用 `assertListening` 方法：

    Event::assertListening(
        OrderShipped::class,
        SendShipmentNotification::class
    );

> [!WARNING]
> 呼叫 `Event::fake()` 後，不會執行任何事件監聽器。因此，如果您的測試使用依賴事件的模型工廠，例如在模型的 `creating` 事件期間建立 UUID，您應該在使用工廠**之後**呼叫 `Event::fake()`。

<a name="faking-a-subset-of-events"></a>
### 偽造部分事件

如果您只想偽造特定事件集的事件監聽器，您可以將它們傳遞給 `fake` 或 `fakeFor` 方法：

```php tab=Pest
test('orders can be processed', function () {
    Event::fake([
        OrderCreated::class,
    ]);

    $order = Order::factory()->create();

    Event::assertDispatched(OrderCreated::class);

    // Other events are dispatched as normal...
    $order->update([...]);
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
    $order->update([...]);
}
```

您可以使用 `except` 方法偽造所有事件，除了指定的一組事件：

    Event::fake()->except([
        OrderCreated::class,
    ]);

<a name="scoped-event-fakes"></a>
### 範圍事件偽造

如果您只想在測試的一部分中偽造事件監聽器，您可以使用 `fakeFor` 方法：

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

    // Events are dispatched as normal and observers will run ...
    $order->update([...]);
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

        // Events are dispatched as normal and observers will run ...
        $order->update([...]);
    }
}
```
