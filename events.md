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
    - [處理失敗的任務](#handling-failed-jobs)
- [分派事件](#dispatching-events)
    - [在資料庫交易後分派事件](#dispatching-events-after-database-transactions)
- [事件訂閱者](#event-subscribers)
    - [撰寫事件訂閱者](#writing-event-subscribers)
    - [註冊事件訂閱者](#registering-event-subscribers)
- [測試](#testing)
    - [偽造部分事件](#faking-a-subset-of-events)
    - [範圍化的事件偽造](#scoped-event-fakes)

<a name="introduction"></a>
## 簡介

Laravel 的事件提供了一個簡單的觀察者模式實作，讓你可以訂閱與監聽應用程式中發生的各種事件。事件類別通常儲存在 `app/Events` 目錄中，而其監聽器則儲存在 `app/Listeners` 中。如果你的應用程式中沒有這些目錄，請別擔心，當你使用 Artisan 主控台命令生成事件和監聽器時，它們將會自動為你建立。

事件是解耦應用程式各個層面的一個好方法，因為單一事件可以有多個互不依賴的監聽器。例如，你可能希望在每次訂單出貨時向使用者發送 Slack 通知。與其將訂單處理程式碼與 Slack 通知程式碼耦合在一起，不如觸發一個 `App\Events\OrderShipped` 事件，監聽器可以接收此事件並用它來分派 Slack 通知。

<a name="generating-events-and-listeners"></a>
## 生成事件與監聽器

為了快速生成事件和監聽器，你可以使用 `make:event` 和 `make:listener` Artisan 命令：

```shell
php artisan make:event PodcastProcessed

php artisan make:listener SendPodcastNotification --event=PodcastProcessed
```

為了方便，你也可以在沒有額外引數的情況下呼叫 `make:event` 和 `make:listener` Artisan 命令。當你這樣做時，Laravel 將自動提示你輸入類別名稱，並在建立監聽器時，提示你輸入它應該監聽的事件：

```shell
php artisan make:event

php artisan make:listener
```

<a name="registering-events-and-listeners"></a>
## 註冊事件與監聽器

<a name="event-discovery"></a>
### 事件探索

預設情況下，Laravel 會自動掃描應用程式的 `Listeners` 目錄來尋找並註冊你的事件監聽器。當 Laravel 找到任何以 `handle` 或 `__invoke` 開頭的監聽器類別方法時，Laravel 會將這些方法註冊為事件監聽器，以處理該方法簽名中類型提示的事件：

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

你可以使用 PHP 的聯合型別來監聽多個事件：

    /**
     * Handle the given event.
     */
    public function handle(PodcastProcessed|PodcastPublished $event): void
    {
        // ...
    }

如果你打算將監聽器儲存在不同的目錄或多個目錄中，你可以指示 Laravel 使用應用程式 `bootstrap/app.php` 檔案中的 `withEvents` 方法掃描這些目錄：

    ->withEvents(discover: [
        __DIR__.'/../app/Domain/Orders/Listeners',
    ])

你可以使用 `*` 字元作為萬用字元，在多個類似的目錄中掃描監聽器：

    ->withEvents(discover: [
        __DIR__.'/../app/Domain/*/Listeners',
    ])

`event:list` 命令可以用來列出應用程式中所有已註冊的監聽器：

```shell
php artisan event:list
```

<a name="event-discovery-in-production"></a>
#### 生產環境中的事件探索

為了讓你的應用程式獲得速度提升，你應該使用 `optimize` 或 `event:cache` Artisan 命令快取應用程式中所有監聽器的清單。通常，這個命令應該作為你的應用程式[部署流程](/docs/{{version}}/deployment#optimization)的一部分來執行。這個清單將被框架用於加速事件註冊過程。`event:clear` 命令可以用來銷毀事件快取。

<a name="manually-registering-events"></a>
### 手動註冊事件

使用 `Event` facade，你可以在應用程式 `AppServiceProvider` 的 `boot` 方法中手動註冊事件及其對應的監聽器：

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

`event:list` 命令可以用來列出應用程式中所有已註冊的監聽器：

```shell
php artisan event:list
```

<a name="closure-listeners"></a>
### 閉包監聽器

通常，監聽器是定義為類別；但是，你也可以在應用程式 `AppServiceProvider` 的 `boot` 方法中手動註冊基於閉包的事件監聽器：

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

當註冊基於閉包的事件監聽器時，你可以將監聽器閉包包裹在 `Illuminate\Events\queueable` 函數中，以指示 Laravel 使用[佇列](/docs/{{version}}/queues)執行監聽器：

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

就像佇列任務一樣，你可以使用 `onConnection`、`onQueue` 和 `delay` 方法來自訂佇列監聽器的執行：

    Event::listen(queueable(function (PodcastProcessed $event) {
        // ...
    })->onConnection('redis')->onQueue('podcasts')->delay(now()->addSeconds(10)));

如果你想要處理匿名佇列監聽器失敗的情況，你可以在定義 `queueable` 監聽器時向 `catch` 方法提供一個閉包。這個閉包將接收事件實例和導致監聽器失敗的 `Throwable` 實例：

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

你也可以使用 `*` 字元作為萬用字元參數來註冊監聽器，這讓你可以在同一個監聽器上捕捉多個事件。萬用字元監聽器會將事件名稱作為其第一個引數，並將整個事件資料陣列作為其第二個引數：

    Event::listen('event.*', function (string $eventName, array $data) {
        // ...
    });

<a name="defining-events"></a>
## 定義事件

事件類別本質上是一個資料容器，它儲存著與事件相關的資訊。例如，假設一個 `App\Events\OrderShipped` 事件接收一個 [Eloquent ORM](/docs/{{version}}/eloquent) 物件：

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

如你所見，這個事件類別不包含任何邏輯。它是一個用於儲存所購買的 `App\Models\Order` 實例的容器。事件所使用的 `SerializesModels` trait 會優雅地序列化任何 Eloquent 模型，如果事件物件使用 PHP 的 `serialize` 函數序列化時，例如在使用[佇列監聽器](#queued-event-listeners)時。

<a name="defining-listeners"></a>
## 定義監聽器

接下來，我們來看看範例事件的監聽器。事件監聽器在其 `handle` 方法中接收事件實例。當 `make:listener` Artisan 命令在使用 `--event` 選項呼叫時，會自動引入正確的事件類別，並在 `handle` 方法中為事件進行型別提示。在 `handle` 方法中，您可以執行任何必要的動作以回應事件：

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
> 您的事件監聽器也可在其建構子中為所需的任何依賴進行型別提示。所有事件監聽器都是透過 Laravel 的 [服務容器](/docs/{{version}}/container) 解析，因此依賴將會自動被注入。

<a name="stopping-the-propagation-of-an-event"></a>
#### 停止事件傳播

有時，您可能希望停止事件傳播到其他監聽器。您可以透過從監聽器的 `handle` 方法回傳 `false` 來實現。

<a name="queued-event-listeners"></a>
## 佇列事件監聽器

如果您的監聽器將執行耗時任務，例如寄送電子郵件或發送 HTTP 請求，那麼佇列監聽器會很有幫助。在使用佇列監聽器之前，請務必[設定您的佇列](/docs/{{version}}/queues)並在您的伺服器或本地開發環境中啟動一個佇列工作者。

若要指定監聽器應進入佇列，請在監聽器類別中新增 `ShouldQueue` 介面。透過 `make:listener` Artisan 命令生成的監聽器已經將此介面導入到目前的命名空間中，因此您可以立即使用：

    <?php

    namespace App\Listeners;

    use App\Events\OrderShipped;
    use Illuminate\Contracts\Queue\ShouldQueue;

    class SendShipmentNotification implements ShouldQueue
    {
        // ...
    }

就這樣！現在，當此監聽器處理的事件被分派時，監聽器將會透過 Laravel 的[佇列系統](/docs/{{version}}/queues)自動進入佇列。如果當監聽器由佇列執行時沒有拋出例外，則該佇列任務在處理完成後將自動刪除。

<a name="customizing-the-queue-connection-queue-name"></a>
#### 自訂佇列連線、名稱與延遲

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

如果您想在運行時定義監聽器的佇列連線、佇列名稱或延遲，您可以在監聽器上定義 `viaConnection`、`viaQueue` 或 `withDelay` 方法：

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
#### 條件式佇列監聽器

有時，您可能需要根據僅在運行時可用的資料來判斷監聽器是否應該進入佇列。為此，可以在監聽器中新增一個 `shouldQueue` 方法來判斷監聽器是否應該進入佇列。如果 `shouldQueue` 方法返回 `false`，則監聽器將不會進入佇列：

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

如果您需要手動存取監聽器底層佇列任務的 `delete` 和 `release` 方法，您可以使用 `Illuminate\Queue\InteractsWithQueue` trait。這個 trait 在生成的監聽器上預設導入，並提供對這些方法的存取：

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

當佇列監聽器在資料庫交易中分派時，它們可能在資料庫交易提交之前由佇列處理。發生這種情況時，您在資料庫交易期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫記錄可能不存在於資料庫中。如果您的監聽器依賴這些模型，那麼當分派佇列監聽器的任務被處理時，可能會發生意想不到的錯誤。

如果您的佇列連線的 `after_commit` 設定選項設為 `false`，您仍然可以透過在監聽器類別上實作 `ShouldQueueAfterCommit` 介面，指出特定的佇列監聽器應在所有開啟的資料庫交易提交後分派：

    <?php

    namespace App\Listeners;

    use Illuminate\Contracts\Queue\ShouldQueueAfterCommit;
    use Illuminate\Queue\InteractsWithQueue;

    class SendShipmentNotification implements ShouldQueueAfterCommit
    {
        use InteractsWithQueue;
    }

> [!NOTE]  
> 要了解更多關於解決這些問題的方法，請查閱有關[佇列任務與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的說明文件。

<a name="handling-failed-jobs"></a>
### 處理失敗的任務

有時候你的佇列事件監聽器可能會失敗。如果佇列監聽器超過了由你的佇列 worker 所定義的最大嘗試次數，那麼 `failed` 方法將會在你的監聽器上被呼叫。`failed` 方法會接收事件實例以及導致失敗的 `Throwable` 實例：

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
#### 指定佇列監聽器的最大嘗試次數

如果你的佇列監聽器遇到錯誤，你可能不希望它無限期地重試。因此，Laravel 提供了多種方式來指定監聽器可以嘗試的次數或時間長度。

你可以在你的監聽器類別上定義 `$tries` 屬性，以指定在被視為失敗之前，該監聽器可以嘗試的次數：

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

作為定義監聽器在失敗前可嘗試次數的替代方案，你可以定義一個監聽器不應再被嘗試的時間點。這允許監聽器在給定的時間範圍內嘗試任意次數。要定義監聽器不應再被嘗試的時間點，請在你的監聽器類別中新增一個 `retryUntil` 方法。此方法應回傳一個 `DateTime` 實例：

    use DateTime;

    /**
     * Determine the time at which the listener should timeout.
     */
    public function retryUntil(): DateTime
    {
        return now()->addMinutes(5);
    }

<a name="specifying-queued-listener-backoff"></a>
#### 指定佇列監聽器的退避策略

如果你想設定 Laravel 在重試遇到例外狀況的監聽器之前應該等待多少秒，你可以在監聽器類別上定義 `backoff` 屬性：

    /**
     * The number of seconds to wait before retrying the queued listener.
     *
     * @var int
     */
    public $backoff = 3;

如果你需要更複雜的邏輯來決定監聽器的退避時間，你可以在你的監聽器類別上定義一個 `backoff` 方法：

    /**
     * Calculate the number of seconds to wait before retrying the queued listener.
     */
    public function backoff(): int
    {
        return 3;
    }

你可以透過從 `backoff` 方法回傳一個退避值陣列來輕鬆設定「指數型」退避。在此範例中，第一次重試的延遲為 1 秒，第二次重試為 5 秒，第三次重試為 10 秒，如果還有更多嘗試次數，則後續每次重試都為 10 秒：

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

要分派事件，您可以呼叫事件上的靜態 `dispatch` 方法。此方法是透過 `Illuminate\Foundation\Events\Dispatchable` trait 提供給事件的。任何傳遞給 `dispatch` 方法的參數都將傳遞給事件的建構函式：

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

如果您想有條件地分派事件，可以使用 `dispatchIf` 和 `dispatchUnless` 方法：

    OrderShipped::dispatchIf($condition, $order);

    OrderShipped::dispatchUnless($condition, $order);

> [!NOTE]  
> 在測試時，如果無需實際觸發事件的監聽器，而只斷言特定事件已被分派，這會很有幫助。Laravel 內建的[測試輔助工具](#testing)能輕易實現此功能。

<a name="dispatching-events-after-database-transactions"></a>
### 在資料庫交易後分派事件

有時，您可能希望指示 Laravel 僅在活躍的資料庫交易提交後才分派事件。為此，您可以在事件類別上實作 `ShouldDispatchAfterCommit` 介面。

此介面指示 Laravel 在目前的資料庫交易提交之前不要分派事件。如果交易失敗，事件將會被捨棄。如果在事件分派時沒有進行中的資料庫交易，事件將會立即分派：

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

事件訂閱者是指能夠在訂閱者類別本身內訂閱多個事件的類別，這允許您在單一類別中定義多個事件處理器。訂閱者應該定義一個 `subscribe` 方法，該方法將會傳入一個事件分派器 (event dispatcher) 實例。您可以呼叫給定分派器上的 `listen` 方法來註冊事件監聽器：

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

如果您的事件監聽器方法定義在訂閱者本身內部，您可能會發現從訂閱者的 `subscribe` 方法中回傳一個事件和方法名稱的陣列會更方便。Laravel 在註冊事件監聽器時會自動判斷訂閱者的類別名稱：

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

在撰寫完訂閱者後，如果它們遵循 Laravel 的[事件探索慣例](#event-discovery)，Laravel 會自動註冊訂閱者內的處理器方法。否則，您可以透過 `Event` Facade 的 `subscribe` 方法手動註冊您的訂閱者。通常，這應該在您應用程式的 `AppServiceProvider` 的 `boot` 方法中完成：

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

當測試分派事件的程式碼時，您可能希望指示 Laravel 不要實際執行事件的監聽器，因為監聽器的程式碼可以直接且獨立地測試，與分派相應事件的程式碼分開。當然，若要測試監聽器本身，您可以實例化一個監聽器實例，並直接在測試中呼叫其 `handle` 方法。

使用 `Event` Facade 的 `fake` 方法，您可以阻止監聽器執行，執行受測程式碼，然後使用 `assertDispatched`、`assertNotDispatched` 和 `assertNothingDispatched` 方法來斷言您的應用程式分派了哪些事件：

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

您可以將閉包傳遞給 `assertDispatched` 或 `assertNotDispatched` 方法，以斷言一個通過特定「真值測試」的事件已被分派。如果至少一個事件通過了給定的真值測試，則該斷言將會成功：

    Event::assertDispatched(function (OrderShipped $event) use ($order) {
        return $event->order->id === $order->id;
    });

如果您只是想斷言一個事件監聽器正在監聽某個特定事件，您可以使用 `assertListening` 方法：

    Event::assertListening(
        OrderShipped::class,
        SendShipmentNotification::class
    );

> [!WARNING]  
> 呼叫 `Event::fake()` 後，將不會執行任何事件監聽器。因此，如果您的測試使用依賴事件的模型工廠 (model factories)，例如在模型的 `creating` 事件中建立 UUID，您應該在使用了工廠**之後**再呼叫 `Event::fake()`。

<a name="faking-a-subset-of-events"></a>
### 偽造部分事件

如果您只想偽造特定事件的事件監聽器，您可以將其傳遞給 `fake` 或 `fakeFor` 方法：

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

您可以偽造所有事件，但排除一組指定的事件，使用 `except` 方法：

    Event::fake()->except([
        OrderCreated::class,
    ]);

<a name="scoped-event-fakes"></a>
### 範圍化的事件偽造

如果您只想在測試的部分範圍內偽造事件監聽器，您可以使用 `fakeFor` 方法：

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