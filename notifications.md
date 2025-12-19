# Notifications

- [簡介](#introduction)
- [生成通知](#generating-notifications)
- [發送通知](#sending-notifications)
    - [使用 Notifiable Trait](#using-the-notifiable-trait)
    - [使用 Notification Facade](#using-the-notification-facade)
    - [指定傳送頻道](#specifying-delivery-channels)
    - [將通知排入佇列](#queueing-notifications)
    - [隨選通知](#on-demand-notifications)
- [郵件通知](#mail-notifications)
    - [格式化郵件訊息](#formatting-mail-messages)
    - [自定義寄件者](#customizing-the-sender)
    - [自定義收件者](#customizing-the-recipient)
    - [自定義主旨](#customizing-the-subject)
    - [自定義郵件傳送器](#customizing-the-mailer)
    - [自定義範本](#customizing-the-templates)
    - [附件](#mail-attachments)
    - [新增標籤與中繼資料](#adding-tags-metadata)
    - [自定義 Symfony 訊息](#customizing-the-symfony-message)
    - [使用 Mailables](#using-mailables)
    - [預覽郵件通知](#previewing-mail-notifications)
- [Markdown 郵件通知](#markdown-mail-notifications)
    - [生成訊息](#generating-the-message)
    - [撰寫訊息](#writing-the-message)
    - [自定義組件](#customizing-the-components)
- [資料庫通知](#database-notifications)
    - [先決條件](#database-prerequisites)
    - [格式化資料庫通知](#formatting-database-notifications)
    - [存取通知](#accessing-the-notifications)
    - [將通知標記為已讀](#marking-notifications-as-read)
- [廣播通知](#broadcast-notifications)
    - [先決條件](#broadcast-prerequisites)
    - [格式化廣播通知](#formatting-broadcast-notifications)
    - [監聽通知](#listening-for-notifications)
- [SMS 通知](#sms-notifications)
    - [先決條件](#sms-prerequisites)
    - [格式化 SMS 通知](#formatting-sms-notifications)
    - [自定義「寄件者」號碼](#customizing-the-from-number)
    - [新增用戶端參照](#adding-a-client-reference)
    - [路由 SMS 通知](#routing-sms-notifications)
- [Slack 通知](#slack-notifications)
    - [先決條件](#slack-prerequisites)
    - [格式化 Slack 通知](#formatting-slack-notifications)
    - [Slack 互動性](#slack-interactivity)
    - [路由 Slack 通知](#routing-slack-notifications)
    - [通知外部 Slack 工作區](#notifying-external-slack-workspaces)
- [在地化通知](#localizing-notifications)
- [測試](#testing)
- [通知事件](#notification-events)
- [自定義頻道](#custom-channels)

<a name="introduction"></a>
## 簡介

除了支援 [發送電子郵件](/docs/{{version}}/mail) 之外，Laravel 還支援透過多種傳送頻道發送通知，包括電子郵件、SMS (透過 [Vonage](https://www.vonage.com/communications-apis/)，原名 Nexmo) 以及 [Slack](https://slack.com)。此外，社群也建立了多種 [社群開發的通知頻道](https://laravel-notification-channels.com/about/#suggesting-a-new-channel)，可透過數十個不同的頻道發送通知！通知也可以儲存在資料庫中，以便顯示在您的網頁介面。

通常，通知應該是簡短且具資訊性的訊息，用於告知使用者應用程式中發生的某些事情。例如，如果您正在撰寫一個帳單應用程式，您可能會透過電子郵件和 SMS 頻道向使用者發送「帳單已支付」通知。


<a name="generating-notifications"></a>
## 生成通知

在 Laravel 中，每個通知都由一個類別表示，通常存放在 `app/Notifications` 目錄中。如果在您的應用程式中沒看到這個目錄，請不用擔心 —— 當您執行 `make:notification` Artisan 指令時，它會自動為您建立：

```shell
php artisan make:notification InvoicePaid
```

此指令會在您的 `app/Notifications` 目錄中放置一個新的通知類別。每個通知類別都包含一個 `via` 方法和若干個訊息建構方法，例如 `toMail` 或 `toDatabase`，這些方法會將通知轉換為專為該特定頻道量身定制的訊息。

<a name="sending-notifications"></a>
## 發送通知


<a name="using-the-notifiable-trait"></a>
### 使用 Notifiable Trait

通知可以透過兩種方式發送：使用 `Notifiable` trait 的 `notify` 方法，或是使用 `Notification` [facade](/docs/{{version}}/facades)。預設情況下，`Notifiable` trait 已包含在應用程式的 `App\Models\User` 模型中：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use Notifiable;
}
```

此 trait 提供的 `notify` 方法預期接收一個通知實例：

```php
use App\Notifications\InvoicePaid;

$user->notify(new InvoicePaid($invoice));
```

> [!NOTE]
> 請記住，您可以在任何模型上使用 `Notifiable` trait。您不限於僅將其包含在 `User` 模型中。


<a name="using-the-notification-facade"></a>
### 使用 Notification Facade

或者，您也可以透過 `Notification` [facade](/docs/{{version}}/facades) 發送通知。當您需要向多個可接收通知的實體（例如使用者集合）發送通知時，這種方法非常有用。若要使用 Facade 發送通知，請將所有可接收通知的實體以及通知實例傳遞給 `send` 方法：

```php
use Illuminate\Support\Facades\Notification;

Notification::send($users, new InvoicePaid($invoice));
```

您也可以使用 `sendNow` 方法立即發送通知。即使通知實作了 `ShouldQueue` 介面，此方法也會立即發送通知：

```php
Notification::sendNow($developers, new DeploymentCompleted($deployment));
```


<a name="specifying-delivery-channels"></a>
### 指定傳送頻道

每個通知類別都有一個 `via` 方法，用於決定通知將在哪個頻道傳送。通知可以透過 `mail`、`database`、`broadcast`、`vonage` 與 `slack` 頻道發送。

> [!NOTE]
> 如果您想使用其他傳送頻道（如 Telegram 或 Pusher），請查看社群驅動的 [Laravel Notification Channels 網站](http://laravel-notification-channels.com)。

`via` 方法接收一個 `$notifiable` 實例，該實例是通知發送對象的類別實例。您可以使用 `$notifiable` 來決定通知應該在哪些頻道上傳送：

```php
/**
 * Get the notification's delivery channels.
 *
 * @return array<int, string>
 */
public function via(object $notifiable): array
{
    return $notifiable->prefers_sms ? ['vonage'] : ['mail', 'database'];
}
```

<a name="queueing-notifications"></a>
### 將通知排入佇列

> [!WARNING]
> 在將通知排入佇列之前，你應該先設定你的佇列並[啟動工作者](/docs/{{version}}/queues#running-the-queue-worker)。

發送通知可能需要一些時間，特別是當頻道需要進行外部 API 呼叫來傳送通知時。為了加速應用程式的響應時間，可以透過在類別中加入 `ShouldQueue` 介面和 `Queueable` trait 來讓通知排入佇列。在使用 `make:notification` 命令生成的所有通知中，都已經預先匯入了該介面和 trait，因此你可以立即將它們加入通知類別中：

```php
<?php

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Notification;

class InvoicePaid extends Notification implements ShouldQueue
{
    use Queueable;

    // ...
}
```

一旦將 `ShouldQueue` 介面加入到通知中，你就可以像往常一樣發送通知。Laravel 會偵測類別上的 `ShouldQueue` 介面，並自動將通知的傳送排入佇列：

```php
$user->notify(new InvoicePaid($invoice));
```

將通知排入佇列時，系統會為每個收件者和頻道的組合建立一個佇列任務。例如，如果你的通知有三個收件者和兩個頻道，則會向佇列發送六個任務。


<a name="delaying-notifications"></a>
#### 延遲通知

如果你想延遲通知的傳送，可以在實例化通知時串接 `delay` 方法：

```php
$delay = now()->plus(minutes: 10);

$user->notify((new InvoicePaid($invoice))->delay($delay));
```

你可以向 `delay` 方法傳遞一個陣列，以指定特定頻道的延遲時間：

```php
$user->notify((new InvoicePaid($invoice))->delay([
    'mail' => now()->plus(minutes: 5),
    'sms' => now()->plus(minutes: 10),
]));
```

或者，你可以在通知類別本身定義 `withDelay` 方法。`withDelay` 方法應回傳頻道名稱與延遲值的陣列：

```php
/**
 * Determine the notification's delivery delay.
 *
 * @return array<string, \Illuminate\Support\Carbon>
 */
public function withDelay(object $notifiable): array
{
    return [
        'mail' => now()->plus(minutes: 5),
        'sms' => now()->plus(minutes: 10),
    ];
}
```


<a name="customizing-the-notification-queue-connection"></a>
#### 自定義通知佇列連接

預設情況下，排隊的通知將使用應用程式的預設佇列連接進行排隊。如果你想為特定通知指定不同的連接，可以在通知的建構子中呼叫 `onConnection` 方法：

```php
<?php

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Notification;

class InvoicePaid extends Notification implements ShouldQueue
{
    use Queueable;

    /**
     * Create a new notification instance.
     */
    public function __construct()
    {
        $this->onConnection('redis');
    }
}
```

或者，如果你想為通知支援的每個通知頻道指定特定的佇列連接，可以在通知中定義 `viaConnections` 方法。此方法應回傳頻道名稱 / 佇列連接名稱對的陣列：

```php
/**
 * Determine which connections should be used for each notification channel.
 *
 * @return array<string, string>
 */
public function viaConnections(): array
{
    return [
        'mail' => 'redis',
        'database' => 'sync',
    ];
}
```


<a name="customizing-notification-channel-queues"></a>
#### 自定義通知頻道佇列

如果你想為通知支援的每個通知頻道指定特定的佇列，可以在通知中定義 `viaQueues` 方法。此方法應回傳頻道名稱 / 佇列名稱對的陣列：

```php
/**
 * Determine which queues should be used for each notification channel.
 *
 * @return array<string, string>
 */
public function viaQueues(): array
{
    return [
        'mail' => 'mail-queue',
        'slack' => 'slack-queue',
    ];
}
```


<a name="customizing-queued-notification-job-properties"></a>
#### 自定義排隊通知任務屬性

你可以透過在通知類別中定義屬性來客製化底層佇列任務的行為。這些屬性將由發送通知的佇列任務繼承：

```php
<?php

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Notification;

class InvoicePaid extends Notification implements ShouldQueue
{
    use Queueable;

    /**
     * The number of times the notification may be attempted.
     *
     * @var int
     */
    public $tries = 5;

    /**
     * The number of seconds the notification can run before timing out.
     *
     * @var int
     */
    public $timeout = 120;

    /**
     * The maximum number of unhandled exceptions to allow before failing.
     *
     * @var int
     */
    public $maxExceptions = 3;

    // ...
}
```

如果你想透過[加密](/docs/{{version}}/encryption)來確保排隊通知資料的隱私和完整性，請將 `ShouldBeEncrypted` 介面加入到你的通知類別中：

```php
<?php

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldBeEncrypted;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Notification;

class InvoicePaid extends Notification implements ShouldQueue, ShouldBeEncrypted
{
    use Queueable;

    // ...
}
```

除了直接在通知類別上定義這些屬性外，你還可以定義 `backoff` 和 `retryUntil` 方法，以指定排隊通知任務的退避策略和重試逾時時間：

```php
use DateTime;

/**
 * Calculate the number of seconds to wait before retrying the notification.
 */
public function backoff(): int
{
    return 3;
}

/**
 * Determine the time at which the notification should timeout.
 */
public function retryUntil(): DateTime
{
    return now()->plus(minutes: 5);
}
```

> [!NOTE]
> 有關這些任務屬性和方法的更多資訊，請查看關於[佇列任務](/docs/{{version}}/queues#max-job-attempts-and-timeout)的說明文件。


<a name="queued-notification-middleware"></a>
#### 排隊通知中介層

排隊的通知可以定義中介層，[就像佇列任務一樣](/docs/{{version}}/queues#job-middleware)。首先，在你的通知類別上定義一個 `middleware` 方法。`middleware` 方法將接收 `$notifiable` 和 `$channel` 變數，這讓你可以根據通知的目的地來自定義回傳的中介層：

```php
use Illuminate\Queue\Middleware\RateLimited;

/**
 * Get the middleware the notification job should pass through.
 *
 * @return array<int, object>
 */
public function middleware(object $notifiable, string $channel)
{
    return match ($channel) {
        'mail' => [new RateLimited('postmark')],
        'slack' => [new RateLimited('slack')],
        default => [],
    };
}
```


<a name="queued-notifications-and-database-transactions"></a>
#### 排隊通知與資料庫交易

當排隊的通知在資料庫交易中被分派時，它們可能在資料庫交易提交之前就由佇列處理。發生這種情況時，你在資料庫交易期間對模型或資料庫紀錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫紀錄可能還不存在於資料庫中。如果你的通知依賴這些模型，則在處理發送排隊通知的任务時可能會發生非預期的錯誤。

如果你的佇列連接之 `after_commit` 設定選項設為 `false`，你仍然可以透過在發送通知時呼叫 `afterCommit` 方法，來指示特定排隊通知應在所有開啟的資料庫交易都提交後才分派：

```php
use App\Notifications\InvoicePaid;

$user->notify((new InvoicePaid($invoice))->afterCommit());
```

或者，你可以在通知的建構子中呼叫 `afterCommit` 方法：

```php
<?php

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Notification;

class InvoicePaid extends Notification implements ShouldQueue
{
    use Queueable;

    /**
     * Create a new notification instance.
     */
    public function __construct()
    {
        $this->afterCommit();
    }
}
```

> [!NOTE]
> 若要瞭解更多關於如何解決這些問題的資訊，請查看關於[佇列任務與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的說明文件。


<a name="determining-if-the-queued-notification-should-be-sent"></a>
#### 判斷是否應發送排隊通知

在排隊通知被分派到佇列進行背景處理後，它通常會被佇列工作者接受並發送給預定的收件者。

但是，如果你想在佇列工作者處理通知後，對是否應發送該排隊通知做出最終判斷，可以在通知類別上定義 `shouldSend` 方法。如果此方法回傳 `false`，則通知將不會被發送：

```php
/**
 * Determine if the notification should be sent.
 */
public function shouldSend(object $notifiable, string $channel): bool
{
    return $this->invoice->isPaid();
}
```

<a name="on-demand-notifications"></a>
### 隨選通知

有時您可能需要發送通知給未儲存在應用程式「使用者」中的對象。使用 `Notification` facade 的 `route` 方法，您可以在發送通知前指定臨時的通知路由資訊：

```php
use Illuminate\Broadcasting\Channel;
use Illuminate\Support\Facades\Notification;

Notification::route('mail', 'taylor@example.com')
    ->route('vonage', '5555555555')
    ->route('slack', '#slack-channel')
    ->route('broadcast', [new Channel('channel-name')])
    ->notify(new InvoicePaid($invoice));
```

如果您想在向 `mail` 路由發送隨選通知時提供收件者名稱，您可以提供一個陣列，其中包含電子郵件地址作為鍵名，而名稱則作為該陣列第一個元素的值：

```php
Notification::route('mail', [
    'barrett@example.com' => 'Barrett Blair',
])->notify(new InvoicePaid($invoice));
```

使用 `routes` 方法，您可以一次為多個通知頻道提供臨時的路由資訊：

```php
Notification::routes([
    'mail' => ['barrett@example.com' => 'Barrett Blair'],
    'vonage' => '5555555555',
])->notify(new InvoicePaid($invoice));
```

<a name="mail-notifications"></a>
## 郵件通知


<a name="formatting-mail-messages"></a>
### 格式化郵件訊息

如果通知支援以電子郵件發送，您應該在通知類別中定義 `toMail` 方法。此方法會接收一個 `$notifiable` 實體，並應回傳一個 `Illuminate\Notifications\Messages\MailMessage` 實例。

`MailMessage` 類別包含一些簡單的方法來幫助您建立交易式電子郵件訊息。郵件訊息可以包含文字行以及「行動呼籲 (call to action)」。讓我們看一個 `toMail` 方法的範例：

```php
/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): MailMessage
{
    $url = url('/invoice/'.$this->invoice->id);

    return (new MailMessage)
        ->greeting('Hello!')
        ->line('One of your invoices has been paid!')
        ->lineIf($this->amount > 0, "Amount paid: {$this->amount}")
        ->action('View Invoice', $url)
        ->line('Thank you for using our application!');
}
```

> [!NOTE]
> 請注意，我們在 `toMail` 方法中使用了 `$this->invoice->id`。您可以將通知生成訊息所需的任何資料傳遞到通知的建構子中。

在此範例中，我們註冊了一個問候語、一行文字、一個行動呼籲，然後是另一行文字。`MailMessage` 物件提供的這些方法讓格式化小型交易電子郵件變得既簡單又快速。郵件頻道接著會將訊息組件轉換為一個美觀、響應式的 HTML 郵件範本，並附帶純文字版本。以下是由 `mail` 頻道生成的電子郵件範例：

<img src="https://laravel.com/img/docs/notification-example-2.png">

> [!NOTE]
> 發送郵件通知時，請確保在 `config/app.php` 設定檔中設定了 `name` 選項。此值將用於郵件通知訊息的頁首和頁尾。


<a name="error-messages"></a>
#### 錯誤訊息

某些通知會告知使用者發生了錯誤，例如發票付款失敗。您可以在建立訊息時呼叫 `error` 方法，來表示該郵件訊息是關於錯誤的。在郵件訊息上使用 `error` 方法時，行動呼籲按鈕將會是紅色而非黑色：

```php
/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->error()
        ->subject('Invoice Payment Failed')
        ->line('...');
}
```


<a name="other-mail-notification-formatting-options"></a>
#### 其他郵件通知格式化選項

除了在通知類別中定義文字「行」之外，您還可以使用 `view` 方法來指定用於渲染通知郵件的自定義範本：

```php
/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)->view(
        'mail.invoice.paid', ['invoice' => $this->invoice]
    );
}
```

您可以透過將視圖名稱作為傳遞給 `view` 方法的陣列的第二個元素，來為郵件訊息指定純文字視圖：

```php
/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)->view(
        ['mail.invoice.paid', 'mail.invoice.paid-text'],
        ['invoice' => $this->invoice]
    );
}
```

或者，如果您的訊息只有純文字視圖，則可以使用 `text` 方法：

```php
/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)->text(
        'mail.invoice.paid-text', ['invoice' => $this->invoice]
    );
}
```


<a name="customizing-the-sender"></a>
### 自定義寄件者

預設情況下，電子郵件的寄件者 / 寄件地址定義在 `config/mail.php` 設定檔中。但是，您可以使用 `from` 方法為特定的通知指定寄件地址：

```php
/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->from('barrett@example.com', 'Barrett Blair')
        ->line('...');
}
```


<a name="customizing-the-recipient"></a>
### 自定義收件者

透過 `mail` 頻道發送通知時，通知系統會自動在您的 notifiable 實體上尋找 `email` 屬性。您可以透過在 notifiable 實體上定義 `routeNotificationForMail` 方法，來自定義用於傳送通知的電子郵件地址：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Illuminate\Notifications\Notification;

class User extends Authenticatable
{
    use Notifiable;

    /**
     * Route notifications for the mail channel.
     *
     * @return  array<string, string>|string
     */
    public function routeNotificationForMail(Notification $notification): array|string
    {
        // Return email address only...
        return $this->email_address;

        // Return email address and name...
        return [$this->email_address => $this->name];
    }
}
```


<a name="customizing-the-subject"></a>
### 自定義主旨

預設情況下，電子郵件的主旨是通知的類別名稱格式化為「詞首大寫 (Title Case)」。因此，如果您的通知類別名為 `InvoicePaid`，則電子郵件的主旨將為 `Invoice Paid`。如果您想為訊息指定不同的主旨，可以在建立訊息時呼叫 `subject` 方法：

```php
/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->subject('Notification Subject')
        ->line('...');
}
```


<a name="customizing-the-mailer"></a>
### 自定義郵件傳送器

預設情況下，電子郵件通知將使用 `config/mail.php` 設定檔中定義的預設郵件傳送器 (Mailer) 發送。但是，您可以在運行時透過呼叫 `mailer` 方法來指定不同的郵件傳送器：

```php
/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->mailer('postmark')
        ->line('...');
}
```


<a name="customizing-the-templates"></a>
### 自定義範本

您可以透過發布通知套件的資源來修改郵件通知所使用的 HTML 和純文字範本。執行此命令後，郵件通知範本將位於 `resources/views/vendor/notifications` 目錄中：

```shell
php artisan vendor:publish --tag=laravel-notifications
```

<a name="mail-attachments"></a>
### 附件

要為郵件通知新增附件，請在構建訊息時使用 `attach` 方法。`attach` 方法的第一個參數接受檔案的絕對路徑：

```php
/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->greeting('Hello!')
        ->attach('/path/to/file');
}
```

> [!NOTE]
> 郵件通知訊息提供的 `attach` 方法也接受 [可附加物件 (attachable objects)](/docs/{{version}}/mail#attachable-objects)。請參閱完整的 [可附加物件文件](/docs/{{version}}/mail#attachable-objects) 以了解更多資訊。

在訊息中附加檔案時，您也可以透過傳遞一個 `array` 作為 `attach` 方法的第二個參數，來指定顯示名稱或 MIME 類型：

```php
/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->greeting('Hello!')
        ->attach('/path/to/file', [
            'as' => 'name.pdf',
            'mime' => 'application/pdf',
        ]);
}
```

與在 mailable 物件中附加檔案不同，您不能使用 `attachFromStorage` 直接從儲存磁碟附加檔案。您應該使用 `attach` 方法並配合儲存磁碟上檔案的絕對路徑。或者，您也可以從 `toMail` 方法中回傳一個 [mailable](/docs/{{version}}/mail#generating-mailables)：

```php
use App\Mail\InvoicePaid as InvoicePaidMailable;

/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): Mailable
{
    return (new InvoicePaidMailable($this->invoice))
        ->to($notifiable->email)
        ->attachFromStorage('/path/to/file');
}
```

必要時，可以使用 `attachMany` 方法在訊息中附加多個檔案：

```php
/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->greeting('Hello!')
        ->attachMany([
            '/path/to/forge.svg',
            '/path/to/vapor.svg' => [
                'as' => 'Logo.svg',
                'mime' => 'image/svg+xml',
            ],
        ]);
}
```

<a name="raw-data-attachments"></a>
#### 原始資料附件

`attachData` 方法可用於將原始位元組字串作為附件附加。呼叫 `attachData` 方法時，您應該提供應分配給該附件的檔名：

```php
/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->greeting('Hello!')
        ->attachData($this->pdf, 'name.pdf', [
            'mime' => 'application/pdf',
        ]);
}
```

<a name="adding-tags-metadata"></a>
### 新增標籤與中繼資料

某些第三方郵件供應商（例如 Mailgun 和 Postmark）支援訊息的「標籤 (tags)」與「中繼資料 (metadata)」，這可用於分組和追蹤應用程式發送的郵件。您可以透過 `tag` 和 `metadata` 方法將標籤和中繼資料新增到郵件訊息中：

```php
/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->greeting('Comment Upvoted!')
        ->tag('upvote')
        ->metadata('comment_id', $this->comment->id);
}
```

如果您的應用程式使用 Mailgun 驅動器，您可以查閱 Mailgun 的文件以獲取有關 [標籤](https://documentation.mailgun.com/docs/mailgun/user-manual/tracking-messages/#tags) 和 [中繼資料](https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/#attaching-metadata-to-messages) 的更多資訊。同樣地，也可以查閱 Postmark 文件以獲取有關其對 [標籤](https://postmarkapp.com/blog/tags-support-for-smtp) 和 [中繼資料](https://postmarkapp.com/support/article/1125-custom-metadata-faq) 支援的更多資訊。

如果您的應用程式使用 Amazon SES 發送郵件，您應該使用 `metadata` 方法將 [SES「標籤」](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html) 附加到訊息中。

<a name="customizing-the-symfony-message"></a>
### 自定義 Symfony 訊息

`MailMessage` 類別的 `withSymfonyMessage` 方法允許您註冊一個閉包，該閉包將在發送訊息之前以 Symfony Message 實例作為參數被呼叫。這讓您有機會在訊息傳送前對其進行深度自定義：

```php
use Symfony\Component\Mime\Email;

/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->withSymfonyMessage(function (Email $message) {
            $message->getHeaders()->addTextHeader(
                'Custom-Header', 'Header Value'
            );
        });
}
```

<a name="using-mailables"></a>
### 使用 Mailables

如果需要，您可以從通知的 `toMail` 方法中回傳一個完整的 [mailable 物件](/docs/{{version}}/mail)。當回傳 `Mailable` 而非 `MailMessage` 時，您需要使用 mailable 物件的 `to` 方法指定訊息收件者：

```php
use App\Mail\InvoicePaid as InvoicePaidMailable;
use Illuminate\Mail\Mailable;

/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): Mailable
{
    return (new InvoicePaidMailable($this->invoice))
        ->to($notifiable->email);
}
```

<a name="mailables-and-on-demand-notifications"></a>
#### Mailables 與隨選通知

如果您正在發送 [隨選通知](#on-demand-notifications)，傳遞給 `toMail` 方法的 `$notifiable` 實例將會是 `Illuminate\Notifications\AnonymousNotifiable` 的實例，它提供了一個 `routeNotificationFor` 方法，可用於檢索隨選通知應發送到的電子郵件地址：

```php
use App\Mail\InvoicePaid as InvoicePaidMailable;
use Illuminate\Notifications\AnonymousNotifiable;
use Illuminate\Mail\Mailable;

/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): Mailable
{
    $address = $notifiable instanceof AnonymousNotifiable
        ? $notifiable->routeNotificationFor('mail')
        : $notifiable->email;

    return (new InvoicePaidMailable($this->invoice))
        ->to($address);
}
```

<a name="previewing-mail-notifications"></a>
### 預覽郵件通知

在設計郵件通知範本時，能像一般的 Blade 範本一樣在瀏覽器中快速預覽渲染後的郵件訊息是非常方便的。因此，Laravel 允許您直接從路由閉包或控制器中回傳由郵件通知產生的任何郵件訊息。當回傳 `MailMessage` 時，它將被渲染並顯示在瀏覽器中，讓您能快速預覽其設計，而無需將其發送到實際的電子郵件地址：

```php
use App\Models\Invoice;
use App\Notifications\InvoicePaid;

Route::get('/notification', function () {
    $invoice = Invoice::find(1);

    return (new InvoicePaid($invoice))
        ->toMail($invoice->user);
});
```

<a name="markdown-mail-notifications"></a>
## Markdown 郵件通知

Markdown 郵件通知讓您可以利用郵件通知預建的範本，同時讓您有更多自由來撰寫更長、自定義的訊息。由於訊息是使用 Markdown 撰寫的，Laravel 能夠為訊息渲染出美觀且具備回應式 (Responsive) 的 HTML 範本，同時也會自動生成純文字版本。


<a name="generating-the-message"></a>
### 生成訊息

若要生成帶有對應 Markdown 範本的通知，您可以使用 `make:notification` Artisan 指令的 `--markdown` 選項：

```shell
php artisan make:notification InvoicePaid --markdown=mail.invoice.paid
```

如同所有其他的郵件通知，使用 Markdown 範本的通知應該在通知類別中定義一個 `toMail` 方法。然而，不使用 `line` 和 `action` 方法來建構通知，而是使用 `markdown` 方法來指定應使用的 Markdown 範本名稱。您可以將希望在範本中使用的資料陣列作為該方法的第二個參數傳遞：

```php
/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): MailMessage
{
    $url = url('/invoice/'.$this->invoice->id);

    return (new MailMessage)
        ->subject('Invoice Paid')
        ->markdown('mail.invoice.paid', ['url' => $url]);
}
```


<a name="writing-the-message"></a>
### 撰寫訊息

Markdown 郵件通知結合使用了 Blade 組件與 Markdown 語法，讓您可以輕鬆建構通知，同時利用 Laravel 預先設計的通知組件：

```blade
<x-mail::message>
# Invoice Paid

Your invoice has been paid!

<x-mail::button :url="$url">
View Invoice
</x-mail::button>

Thanks,<br>
{{ config('app.name') }}
</x-mail::message>
```

> [!NOTE]
> 撰寫 Markdown 郵件時請勿使用過多的縮排。根據 Markdown 標準，Markdown 解析器會將縮排的內容渲染為程式碼區塊。


<a name="button-component"></a>
#### 按鈕組件 (Button Component)

按鈕組件會渲染一個置中的按鈕連結。該組件接受兩個參數：`url` 和選填的 `color`。支援的顏色有 `primary`、`green` 和 `red`。您可以根據需要向通知添加任意數量的按鈕組件：

```blade
<x-mail::button :url="$url" color="green">
View Invoice
</x-mail::button>
```


<a name="panel-component"></a>
#### 面板組件 (Panel Component)

面板組件會將指定的文字區塊渲染在一個面板中，該面板的背景顏色與通知的其他部分略有不同。這讓您可以吸引使用者注意特定的文字區塊：

```blade
<x-mail::panel>
This is the panel content.
</x-mail::panel>
```


<a name="table-component"></a>
#### 表格組件 (Table Component)

表格組件允許您將 Markdown 表格轉換為 HTML 表格。該組件接受 Markdown 表格作為其內容。支援使用預設的 Markdown 表格對齊語法來對齊表格欄位：

```blade
<x-mail::table>
| Laravel       | Table         | Example       |
| ------------- | :-----------: | ------------: |
| Col 2 is      | Centered      | $10           |
| Col 3 is      | Right-Aligned | $20           |
</x-mail::table>
```


<a name="customizing-the-components"></a>
### 自定義組件

您可以將所有 Markdown 通知組件匯出到您自己的應用程式中進行自定義。若要匯出組件，請使用 `vendor:publish` Artisan 指令來發佈 `laravel-mail` 資源標籤：

```shell
php artisan vendor:publish --tag=laravel-mail
```

此指令會將 Markdown 郵件組件發佈到 `resources/views/vendor/mail` 目錄。`mail` 目錄下將包含 `html` 和 `text` 目錄，分別包含每個可用組件的對應呈現。您可以隨意自定義這些組件。


<a name="customizing-the-css"></a>
#### 自定義 CSS

匯出組件後，`resources/views/vendor/mail/html/themes` 目錄中會包含一個 `default.css` 檔案。您可以自定義此檔案中的 CSS，您的樣式將自動內嵌 (In-lined) 到 Markdown 通知的 HTML 呈現中。

如果您想為 Laravel 的 Markdown 組件建立一個全新的主題，可以在 `html/themes` 目錄中放置一個 CSS 檔案。命名並儲存 CSS 檔案後，請更新 `mail` 設定檔中的 `theme` 選項，使其與新主題的名稱相符。

若要為個別通知自定義主題，可以在建構通知的郵件訊息時呼叫 `theme` 方法。`theme` 方法接受發送通知時應使用的主題名稱：

```php
/**
 * Get the mail representation of the notification.
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->theme('invoice')
        ->subject('Invoice Paid')
        ->markdown('mail.invoice.paid', ['url' => $url]);
}
```

<a name="database-notifications"></a>
## 資料庫通知

<a name="database-prerequisites"></a>
### 先決條件

`database` 通知頻道將通知資訊存儲在資料庫資料表中。此資料表將包含通知類型以及描述通知的 JSON 資料結構等資訊。

您可以查詢該資料表以在應用程式的使用者介面中顯示通知。但在執行此操作之前，您需要建立一個資料庫資料表來存放您的通知。您可以使用 `make:notifications-table` 指令來生成具有正確資料表結構的 [migration](/docs/{{version}}/migrations)：

```shell
php artisan make:notifications-table

php artisan migrate
```

> [!NOTE]
> 如果您的可通知模型使用的是 [UUID 或 ULID 主鍵](/docs/{{version}}/eloquent#uuid-and-ulid-keys)，您應該在通知資料表的 migration 中將 `morphs` 方法替換為 [uuidMorphs](/docs/{{version}}/migrations#column-method-uuidMorphs) 或 [ulidMorphs](/docs/{{version}}/migrations#column-method-ulidMorphs)。

<a name="formatting-database-notifications"></a>
### 格式化資料庫通知

如果通知支援存儲在資料庫資料表中，您應該在通知類別上定義一個 `toDatabase` 或 `toArray` 方法。此方法將接收一個 `$notifiable` 實體，並應回傳一個純 PHP 陣列。回傳的陣列將被編碼為 JSON 並存儲在 `notifications` 資料表的 `data` 欄位中。讓我們看一個 `toArray` 方法的範例：

```php
/**
 * Get the array representation of the notification.
 *
 * @return array<string, mixed>
 */
public function toArray(object $notifiable): array
{
    return [
        'invoice_id' => $this->invoice->id,
        'amount' => $this->invoice->amount,
    ];
}
```

當通知存儲在應用程式的資料庫中時，預設情況下 `type` 欄位將設定為通知的類別名稱，而 `read_at` 欄位將為 `null`。然而，您可以透過在通知類別中定義 `databaseType` 和 `initialDatabaseReadAtValue` 方法來過自定義此行為：

```php
use Illuminate\Support\Carbon;

/**
 * Get the notification's database type.
 */
public function databaseType(object $notifiable): string
{
    return 'invoice-paid';
}

/**
 * Get the initial value for the "read_at" column.
 */
public function initialDatabaseReadAtValue(): ?Carbon
{
    return null;
}
```

<a name="todatabase-vs-toarray"></a>
#### `toDatabase` vs. `toArray`

`toArray` 方法也被 `broadcast` 頻道用來決定要廣播到您的 JavaScript 驅動前端的資料。如果您希望 `database` 和 `broadcast` 頻道有兩種不同的陣列表示方式，您應該定義一個 `toDatabase` 方法而不是 `toArray` 方法。

<a name="accessing-the-notifications"></a>
### 存取通知

一旦通知存儲在資料庫中，您需要一種方便的方式從您的可通知實體中存取它們。Laravel 預設的 `App\Models\User` 模型中包含的 `Illuminate\Notifications\Notifiable` trait，包含一個 `notifications` [Eloquent 關聯項目](/docs/{{version}}/eloquent-relationships)，該關聯項目會回傳該實體的通知。要取得通知，您可以像存取任何其他 Eloquent 關聯項目一樣存取此方法。預設情況下，通知將按 `created_at` 時間戳記排序，最新的通知位於集合的開頭：

```php
$user = App\Models\User::find(1);

foreach ($user->notifications as $notification) {
    echo $notification->type;
}
```

如果您只想檢索「未讀」通知，可以使用 `unreadNotifications` 關聯項目。同樣地，這些通知將按 `created_at` 時間戳記排序，最新的通知位於集合的開頭：

```php
$user = App\Models\User::find(1);

foreach ($user->unreadNotifications as $notification) {
    echo $notification->type;
}
```

如果您只想檢索「已讀」通知，可以使用 `readNotifications` 關聯項目：

```php
$user = App\Models\User::find(1);

foreach ($user->readNotifications as $notification) {
    echo $notification->type;
}
```

> [!NOTE]
> 要從 JavaScript 客戶端存取您的通知，您應該為您的應用程式定義一個通知控制器，該控制器會回傳可通知實體（例如目前使用者）的通知。然後，您可以從 JavaScript 客戶端向該控制器的 URL 發送 HTTP 請求。

<a name="marking-notifications-as-read"></a>
### 將通知標記為已讀

通常，您會希望在使用者查看通知時將其標記為「已讀」。`Illuminate\Notifications\Notifiable` trait 提供了一個 `markAsRead` 方法，該方法會更新通知資料庫記錄上的 `read_at` 欄位：

```php
$user = App\Models\User::find(1);

foreach ($user->unreadNotifications as $notification) {
    $notification->markAsRead();
}
```

然而，除了循環處理每個通知外，您也可以直接在通知集合上使用 `markAsRead` 方法：

```php
$user->unreadNotifications->markAsRead();
```

您也可以使用大量更新查詢來將所有通知標記為已讀，而無需從資料庫中檢索它們：

```php
$user = App\Models\User::find(1);

$user->unreadNotifications()->update(['read_at' => now()]);
```

您可以 `delete` 這些通知，將它們從資料表中完全移除：

```php
$user->notifications()->delete();
```

<a name="broadcast-notifications"></a>
## 廣播通知


<a name="broadcast-prerequisites"></a>
### 先決條件

在廣播通知之前，您應該先設定並熟悉 Laravel 的[事件廣播](/docs/{{version}}/broadcasting)服務。事件廣播提供了一種從您的 JavaScript 前端對伺服器端 Laravel 事件做出反應的方法。


<a name="formatting-broadcast-notifications"></a>
### 格式化廣播通知

`broadcast` 頻道使用 Laravel 的[事件廣播](/docs/{{version}}/broadcasting)服務來廣播通知，讓您的 JavaScript 前端能即時擷取通知。若通知支援廣播，您可以在通知類別中定義 `toBroadcast` 方法。此方法會接收一個 `$notifiable` 實體，並應回傳一個 `BroadcastMessage` 執行個體。若 `toBroadcast` 方法不存在，則會使用 `toArray` 方法來收集應廣播的資料。回傳的資料將被編碼為 JSON 並廣播到您的 JavaScript 前端。讓我們看一個 `toBroadcast` 方法的範例：

```php
use Illuminate\Notifications\Messages\BroadcastMessage;

/**
 * Get the broadcastable representation of the notification.
 */
public function toBroadcast(object $notifiable): BroadcastMessage
{
    return new BroadcastMessage([
        'invoice_id' => $this->invoice->id,
        'amount' => $this->invoice->amount,
    ]);
}
```


<a name="broadcast-queue-configuration"></a>
#### 廣播佇列設定

所有廣播通知都會被排入佇列以進行廣播。如果您想設定用於佇列廣播操作的佇列連線或佇列名稱，您可以使用 `BroadcastMessage` 的 `onConnection` 與 `onQueue` 方法：

```php
return (new BroadcastMessage($data))
    ->onConnection('sqs')
    ->onQueue('broadcasts');
```


<a name="customizing-the-notification-type"></a>
#### 自定義通知類型

除了您指定的資料外，所有廣播通知還包含一個 `type` 欄位，其中含有該通知的完整類別名稱。如果您想自定義通知的 `type`，您可以在通知類別中定義 `broadcastType` 方法：

```php
/**
 * Get the type of the notification being broadcast.
 */
public function broadcastType(): string
{
    return 'broadcast.message';
}
```


<a name="listening-for-notifications"></a>
### 監聽通知

通知將在一個使用 `{notifiable}.{id}` 慣例格式化的私有頻道上廣播。因此，如果您向 ID 為 `1` 的 `App\Models\User` 執行個體發送通知，該通知將在 `App.Models.User.1` 私有頻道上廣播。使用 [Laravel Echo](/docs/{{version}}/broadcasting#client-side-installation) 時，您可以使用 `notification` 方法輕鬆監聽頻道上的通知：

```js
Echo.private('App.Models.User.' + userId)
    .notification((notification) => {
        console.log(notification.type);
    });
```


<a name="using-react-or-vue"></a>
#### 使用 React 或 Vue

Laravel Echo 包含了 React 與 Vue 的 hook，讓監聽通知變得非常輕鬆。首先，請呼叫用於監聽通知的 `useEchoNotification` hook。`useEchoNotification` hook 會在元件卸載時自動離開頻道：

```js tab=React
import { useEchoNotification } from "@laravel/echo-react";

useEchoNotification(
    `App.Models.User.${userId}`,
    (notification) => {
        console.log(notification.type);
    },
);
```

```vue tab=Vue
<script setup lang="ts">
import { useEchoNotification } from "@laravel/echo-vue";

useEchoNotification(
    `App.Models.User.${userId}`,
    (notification) => {
        console.log(notification.type);
    },
);
</script>
```

預設情況下，該 hook 會監聽所有通知。若要指定您想監聽的通知類型，您可以向 `useEchoNotification` 提供字串或類型陣列：

```js tab=React
import { useEchoNotification } from "@laravel/echo-react";

useEchoNotification(
    `App.Models.User.${userId}`,
    (notification) => {
        console.log(notification.type);
    },
    'App.Notifications.InvoicePaid',
);
```

```vue tab=Vue
<script setup lang="ts">
import { useEchoNotification } from "@laravel/echo-vue";

useEchoNotification(
    `App.Models.User.${userId}`,
    (notification) => {
        console.log(notification.type);
    },
    'App.Notifications.InvoicePaid',
);
</script>
```

您還可以指定通知有效負載 (Payload) 資料的結構，提供更好的型別安全性與開發便利性：

```ts
type InvoicePaidNotification = {
    invoice_id: number;
    created_at: string;
};

useEchoNotification<InvoicePaidNotification>(
    `App.Models.User.${userId}`,
    (notification) => {
        console.log(notification.invoice_id);
        console.log(notification.created_at);
        console.log(notification.type);
    },
    'App.Notifications.InvoicePaid',
);
```


<a name="customizing-the-notification-channel"></a>
#### 自定義通知頻道

如果您想自定義實體的廣播通知在哪個頻道上廣播，您可以在該 Notifiable 實體上定義 `receivesBroadcastNotificationsOn` 方法：

```php
<?php

namespace App\Models;

use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use Notifiable;

    /**
     * The channels the user receives notification broadcasts on.
     */
    public function receivesBroadcastNotificationsOn(): string
    {
        return 'users.'.$this->id;
    }
}
```

<a name="sms-notifications"></a>
## SMS 通知


<a name="sms-prerequisites"></a>
### 先決條件

Laravel 中的 SMS 通知發送功能是由 [Vonage](https://www.vonage.com/) (舊稱為 Nexmo) 提供支援。在您透過 Vonage 發送通知之前，您需要安裝 `laravel/vonage-notification-channel` 與 `guzzlehttp/guzzle` 套件：

```shell
composer require laravel/vonage-notification-channel guzzlehttp/guzzle
```

此套件包含了一個 [設定檔](https://github.com/laravel/vonage-notification-channel/blob/3.x/config/vonage.php)。然而，您並非必須將此設定檔導出到您自己的應用程式中。您可以簡單地使用 `VONAGE_KEY` 與 `VONAGE_SECRET` 環境變數來定義您的 Vonage 公鑰與私鑰。

定義好金鑰後，您應該設定 `VONAGE_SMS_FROM` 環境變數，用以定義預設發送 SMS 訊息的電話號碼。您可以在 Vonage 控制台生成此電話號碼：

```ini
VONAGE_SMS_FROM=15556666666
```


<a name="formatting-sms-notifications"></a>
### 格式化 SMS 通知

如果通知支援以 SMS 發送，您應該在通知類別中定義一個 `toVonage` 方法。此方法將接收一個 `$notifiable` 實體，並應回傳一個 `Illuminate\Notifications\Messages\VonageMessage` 實例：

```php
use Illuminate\Notifications\Messages\VonageMessage;

/**
 * Get the Vonage / SMS representation of the notification.
 */
public function toVonage(object $notifiable): VonageMessage
{
    return (new VonageMessage)
        ->content('Your SMS message content');
}
```


<a name="unicode-content"></a>
#### Unicode 內容

如果您的 SMS 訊息包含 Unicode 字元，您應該在建構 `VonageMessage` 實例時呼叫 `unicode` 方法：

```php
use Illuminate\Notifications\Messages\VonageMessage;

/**
 * Get the Vonage / SMS representation of the notification.
 */
public function toVonage(object $notifiable): VonageMessage
{
    return (new VonageMessage)
        ->content('Your unicode message')
        ->unicode();
}
```


<a name="customizing-the-from-number"></a>
### 自定義「寄件者」號碼

如果您想從一個不同於 `VONAGE_SMS_FROM` 環境變數所指定的電話號碼發送某些通知，您可以在 `VonageMessage` 實例上呼叫 `from` 方法：

```php
use Illuminate\Notifications\Messages\VonageMessage;

/**
 * Get the Vonage / SMS representation of the notification.
 */
public function toVonage(object $notifiable): VonageMessage
{
    return (new VonageMessage)
        ->content('Your SMS message content')
        ->from('15554443333');
}
```


<a name="adding-a-client-reference"></a>
### 新增用戶端參照

如果您想追蹤每個使用者、團隊或用戶端的成本，您可以為通知新增「用戶端參照 (Client Reference)」。Vonage 允許您使用此用戶端參照生成報表，以便您更了解特定客戶的 SMS 使用情況。用戶端參照可以是任何長度不超過 40 個字元的字串：

```php
use Illuminate\Notifications\Messages\VonageMessage;

/**
 * Get the Vonage / SMS representation of the notification.
 */
public function toVonage(object $notifiable): VonageMessage
{
    return (new VonageMessage)
        ->clientReference((string) $notifiable->id)
        ->content('Your SMS message content');
}
```


<a name="routing-sms-notifications"></a>
### 路由 SMS 通知

要將 Vonage 通知路由到正確的電話號碼，請在您的可通知實體上定義一個 `routeNotificationForVonage` 方法：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Illuminate\Notifications\Notification;

class User extends Authenticatable
{
    use Notifiable;

    /**
     * Route notifications for the Vonage channel.
     */
    public function routeNotificationForVonage(Notification $notification): string
    {
        return $this->phone_number;
    }
}
```

<a name="slack-notifications"></a>
## Slack 通知


<a name="slack-prerequisites"></a>
### 先決條件

在發送 Slack 通知之前，您應該透過 Composer 安裝 Slack 通知頻道：

```shell
composer require laravel/slack-notification-channel
```

此外，您必須為您的 Slack 工作區建立一個 [Slack App](https://api.slack.com/apps?new_app=1)。

如果您只需要將通知發送到建立該 App 的同一個 Slack 工作區，您應確保您的 App 具有 `chat:write`、`chat:write.public` 以及 `chat:write.customize` 權限範圍 (Scopes)。這些權限範圍可以從 Slack 內部的「OAuth & Permissions」App 管理分頁中新增。

接著，複製 App 的「Bot User OAuth Token」，並將其放置在應用程式 `services.php` 設定檔中的 `slack` 設定陣列裡。此權杖 (Token) 可以在 Slack 內的「OAuth & Permissions」分頁中找到：

```php
'slack' => [
    'notifications' => [
        'bot_user_oauth_token' => env('SLACK_BOT_USER_OAUTH_TOKEN'),
        'channel' => env('SLACK_BOT_USER_DEFAULT_CHANNEL'),
    ],
],
```


<a name="slack-app-distribution"></a>
#### App 分發

如果您的應用程式將向應用程式使用者擁有的外部 Slack 工作區發送通知，您將需要透過 Slack「分發 (Distribute)」您的 App。App 分發可以在 Slack 內的「Manage Distribution」分頁中進行管理。一旦您的 App 完成分發，您就可以使用 [Socialite](/docs/{{version}}/socialite) 代表您的應用程式使用者[取得 Slack Bot 權杖](/docs/{{version}}/socialite#slack-bot-scopes)。


<a name="formatting-slack-notifications"></a>
### 格式化 Slack 通知

如果通知支援以 Slack 訊息發送，您應該在通知類別中定義一個 `toSlack` 方法。此方法將接收一個 `$notifiable` 實體，並應回傳一個 `Illuminate\Notifications\Slack\SlackMessage` 實體。您可以使用 [Slack 的 Block Kit API](https://api.slack.com/block-kit) 構建豐富的通知。以下範例可以在 [Slack 的 Block Kit builder](https://app.slack.com/block-kit-builder/T01KWS6K23Z#%7B%22blocks%22:%5B%7B%22type%22:%22header%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22Invoice%20Paid%22%7D%7D,%7B%22type%22:%22context%22,%22elements%22:%5B%7B%22type%22:%22plain_text%22,%22text%22:%22Customer%20%231234%22%7D%5D%7D,%7B%22type%22:%22section%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22An%20invoice%20has%20been%20paid.%22%7D,%22fields%22:%5B%7B%22type%22:%22mrkdwn%22,%22text%22:%22*Invoice%20No:*%5Cn1000%22%7D,%7B%22type%22:%22mrkdwn%22,%22text%22:%22*Invoice%20Recipient:*%5Cntaylor@laravel.com%22%7D%5D%7D,%7B%22type%22:%22divider%22%7D,%7B%22type%22:%22section%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22Congratulations!%22%7D%7D%5D%7D) 中進行預覽：

```php
use Illuminate\Notifications\Slack\BlockKit\Blocks\ContextBlock;
use Illuminate\Notifications\Slack\BlockKit\Blocks\SectionBlock;
use Illuminate\Notifications\Slack\SlackMessage;

/**
 * Get the Slack representation of the notification.
 */
public function toSlack(object $notifiable): SlackMessage
{
    return (new SlackMessage)
        ->text('One of your invoices has been paid!')
        ->headerBlock('Invoice Paid')
        ->contextBlock(function (ContextBlock $block) {
            $block->text('Customer #1234');
        })
        ->sectionBlock(function (SectionBlock $block) {
            $block->text('An invoice has been paid.');
            $block->field("*Invoice No:*\n1000")->markdown();
            $block->field("*Invoice Recipient:*\ntaylor@laravel.com")->markdown();
        })
        ->dividerBlock()
        ->sectionBlock(function (SectionBlock $block) {
            $block->text('Congratulations!');
        });
}
```


<a name="using-slacks-block-kit-builder-template"></a>
#### 使用 Slack 的 Block Kit Builder 範本

除了使用流暢的訊息構建器 (Fluent message builder) 方法來構建您的 Block Kit 訊息外，您也可以將 Slack 的 Block Kit Builder 生成的原始 JSON 有效負載 (Payload) 提供給 `usingBlockKitTemplate` 方法：

```php
use Illuminate\Notifications\Slack\SlackMessage;
use Illuminate\Support\Str;

/**
 * Get the Slack representation of the notification.
 */
public function toSlack(object $notifiable): SlackMessage
{
    $template = <<<JSON
        {
          "blocks": [
            {
              "type": "header",
              "text": {
                "type": "plain_text",
                "text": "Team Announcement"
              }
            },
            {
              "type": "section",
              "text": {
                "type": "plain_text",
                "text": "We are hiring!"
              }
            }
          ]
        }
    JSON;

    return (new SlackMessage)
        ->usingBlockKitTemplate($template);
}
```

<a name="slack-interactivity"></a>
### Slack 互動性

Slack 的 Block Kit 通知系統提供了強大的功能來 [處理使用者互動](https://api.slack.com/interactivity/handling)。要使用這些功能，您的 Slack App 應該啟用「Interactivity」並配置一個指向應用程式所提供 URL 的「Request URL」。這些設定可以在 Slack 內的「Interactivity & Shortcuts」App 管理分頁中進行管理。

在以下使用 `actionsBlock` 方法的範例中，Slack 將向您的「Request URL」發送一個 `POST` 請求，其承載內容 (Payload) 包含點擊按鈕的 Slack 使用者、所點擊按鈕的 ID 等資訊。您的應用程式接著可以根據該承載內容決定要執行的動作。您還應該 [驗證請求](https://api.slack.com/authentication/verifying-requests-from-slack) 是否由 Slack 發出：

```php
use Illuminate\Notifications\Slack\BlockKit\Blocks\ActionsBlock;
use Illuminate\Notifications\Slack\BlockKit\Blocks\ContextBlock;
use Illuminate\Notifications\Slack\BlockKit\Blocks\SectionBlock;
use Illuminate\Notifications\Slack\SlackMessage;

/**
 * Get the Slack representation of the notification.
 */
public function toSlack(object $notifiable): SlackMessage
{
    return (new SlackMessage)
        ->text('One of your invoices has been paid!')
        ->headerBlock('Invoice Paid')
        ->contextBlock(function (ContextBlock $block) {
            $block->text('Customer #1234');
        })
        ->sectionBlock(function (SectionBlock $block) {
            $block->text('An invoice has been paid.');
        })
        ->actionsBlock(function (ActionsBlock $block) {
             // ID defaults to "button_acknowledge_invoice"...
            $block->button('Acknowledge Invoice')->primary();

            // Manually configure the ID...
            $block->button('Deny')->danger()->id('deny_invoice');
        });
}
```


<a name="slack-confirmation-modals"></a>
#### 確認視窗

如果您希望使用者在執行動作之前必須進行確認，可以在定義按鈕時調用 `confirm` 方法。`confirm` 方法接受一個訊息和一個接收 `ConfirmObject` 實例的閉包 (Closure)：

```php
use Illuminate\Notifications\Slack\BlockKit\Blocks\ActionsBlock;
use Illuminate\Notifications\Slack\BlockKit\Blocks\ContextBlock;
use Illuminate\Notifications\Slack\BlockKit\Blocks\SectionBlock;
use Illuminate\Notifications\Slack\BlockKit\Composites\ConfirmObject;
use Illuminate\Notifications\Slack\SlackMessage;

/**
 * Get the Slack representation of the notification.
 */
public function toSlack(object $notifiable): SlackMessage
{
    return (new SlackMessage)
        ->text('One of your invoices has been paid!')
        ->headerBlock('Invoice Paid')
        ->contextBlock(function (ContextBlock $block) {
            $block->text('Customer #1234');
        })
        ->sectionBlock(function (SectionBlock $block) {
            $block->text('An invoice has been paid.');
        })
        ->actionsBlock(function (ActionsBlock $block) {
            $block->button('Acknowledge Invoice')
                ->primary()
                ->confirm(
                    'Acknowledge the payment and send a thank you email?',
                    function (ConfirmObject $dialog) {
                        $dialog->confirm('Yes');
                        $dialog->deny('No');
                    }
                );
        });
}
```


<a name="inspecting-slack-blocks"></a>
#### 檢查 Slack 區塊

如果您想快速檢查您建立的區塊，可以在 `SlackMessage` 實例上調用 `dd` 方法。`dd` 方法將生成並傾印 (Dump) 一個指向 Slack [Block Kit Builder](https://app.slack.com/block-kit-builder/) 的 URL，該網頁會在瀏覽器中顯示承載內容與通知的預覽。您可以向 `dd` 方法傳遞 `true` 以傾印原始承載內容：

```php
return (new SlackMessage)
    ->text('One of your invoices has been paid!')
    ->headerBlock('Invoice Paid')
    ->dd();
```


<a name="routing-slack-notifications"></a>
### 路由 Slack 通知

要將 Slack 通知引導至適當的 Slack 團隊和頻道，請在您的可通知模型 (Notifiable Model) 上定義 `routeNotificationForSlack` 方法。此方法可以回傳以下三種值之一：

- `null` - 這會將路由推遲到通知本身中配置的頻道。您可以在建立 `SlackMessage` 時使用 `to` 方法在通知內配置頻道。
- 指定要發送通知的 Slack 頻道的字串，例如 `#support-channel`。
- 一個 `SlackRoute` 實例，這允許您指定一個 OAuth 權杖 (Token) 和頻道名稱，例如 `SlackRoute::make($this->slack_channel, $this->slack_token)`。此方法應當用於向外部工作區發送通知。

例如，從 `routeNotificationForSlack` 方法回傳 `#support-channel` 將會把通知發送到與應用程式 `services.php` 設定檔中的 Bot 使用者 OAuth 權杖關聯的工作區中的 `#support-channel` 頻道：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Illuminate\Notifications\Notification;

class User extends Authenticatable
{
    use Notifiable;

    /**
     * Route notifications for the Slack channel.
     */
    public function routeNotificationForSlack(Notification $notification): mixed
    {
        return '#support-channel';
    }
}
```


<a name="notifying-external-slack-workspaces"></a>
### 通知外部 Slack 工作區

> [!NOTE]
> 在向外部 Slack 工作區發送通知之前，您的 Slack App 必須已經 [發佈](#slack-app-distribution)。

當然，您經常會希望向應用程式使用者擁有的 Slack 工作區發送通知。為此，您首先需要取得該使用者的 Slack OAuth 權杖。幸運的是，[Socialite](/docs/{{version}}/socialite) 包含一個 Slack 驅動程式，可讓您輕鬆地透過 Slack 驗證應用程式的使用者並 [取得 Bot 權杖](/docs/{{version}}/socialite#slack-bot-scopes)。

一旦您取得 Bot 權杖並將其儲存在應用程式的資料庫中，您就可以利用 `SlackRoute::make` 方法將通知路由到使用者的工作區。此外，您的應用程式可能需要提供機會讓使用者指定通知應發送到哪個頻道：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Illuminate\Notifications\Notification;
use Illuminate\Notifications\Slack\SlackRoute;

class User extends Authenticatable
{
    use Notifiable;

    /**
     * Route notifications for the Slack channel.
     */
    public function routeNotificationForSlack(Notification $notification): mixed
    {
        return SlackRoute::make($this->slack_channel, $this->slack_token);
    }
}
```

<a name="localizing-notifications"></a>
## 在地化通知

Laravel 允許您使用 HTTP 請求當前語系以外的語系發送通知，且即便通知被排入佇列，也會記住此語系。

為了達成此目的，`Illuminate\Notifications\Notification` 類別提供了一個 `locale` 方法來設定所需的語言。應用程式會在評估通知時切換到該語系，並在評估完成後切換回之前的語系：

```php
$user->notify((new InvoicePaid($invoice))->locale('es'));
```

多個可通知實體的在地化也可以透過 `Notification` Facade 來實現：

```php
Notification::locale('es')->send(
    $users, new InvoicePaid($invoice)
);
```


<a name="user-preferred-locales"></a>
#### 使用者偏好語系

有時，應用程式會儲存每個使用者的偏好語系。透過在您的可通知模型上實作 `HasLocalePreference` 合約，您可以指示 Laravel 在發送通知時使用此儲存的語系：

```php
use Illuminate\Contracts\Translation\HasLocalePreference;

class User extends Model implements HasLocalePreference
{
    /**
     * Get the user's preferred locale.
     */
    public function preferredLocale(): string
    {
        return $this->locale;
    }
}
```

一旦您實作了該介面，Laravel 在向該模型發送通知和 Mailables 時將自動使用偏好語系。因此，使用此介面時不需要呼叫 `locale` 方法：

```php
$user->notify(new InvoicePaid($invoice));
```


<a name="testing"></a>
## 測試

您可以使用 `Notification` Facade 的 `fake` 方法來防止通知被實際發送。通常，發送通知與您實際測試的程式碼無關。大多數情況下，只需斷言 Laravel 已收到發送給定通知的指令就足夠了。

在呼叫 `Notification` Facade 的 `fake` 方法後，您接著可以斷言通知已被指示發送給使用者，甚至檢查通知收到的資料：

```php tab=Pest
<?php

use App\Notifications\OrderShipped;
use Illuminate\Support\Facades\Notification;

test('orders can be shipped', function () {
    Notification::fake();

    // Perform order shipping...

    // Assert that no notifications were sent...
    Notification::assertNothingSent();

    // Assert a notification was sent to the given users...
    Notification::assertSentTo(
        [$user], OrderShipped::class
    );

    // Assert a notification was not sent...
    Notification::assertNotSentTo(
        [$user], AnotherNotification::class
    );

    // Assert a notification was sent twice...
    Notification::assertSentTimes(WeeklyReminder::class, 2);

    // Assert that a given number of notifications were sent...
    Notification::assertCount(3);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Notifications\OrderShipped;
use Illuminate\Support\Facades\Notification;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_orders_can_be_shipped(): void
    {
        Notification::fake();

        // Perform order shipping...

        // Assert that no notifications were sent...
        Notification::assertNothingSent();

        // Assert a notification was sent to the given users...
        Notification::assertSentTo(
            [$user], OrderShipped::class
        );

        // Assert a notification was not sent...
        Notification::assertNotSentTo(
            [$user], AnotherNotification::class
        );

        // Assert a notification was sent twice...
        Notification::assertSentTimes(WeeklyReminder::class, 2);

        // Assert that a given number of notifications were sent...
        Notification::assertCount(3);
    }
}
```

您可以將閉包傳遞給 `assertSentTo` 或 `assertNotSentTo` 方法，以便斷言發送的通知通過了指定的「真值測試 (Truth Test)」。如果至少發送了一個通過給定真值測試的通知，則斷言將成功：

```php
Notification::assertSentTo(
    $user,
    function (OrderShipped $notification, array $channels) use ($order) {
        return $notification->order->id === $order->id;
    }
);
```


<a name="on-demand-notifications"></a>
#### 隨選通知

如果您測試的程式碼發送了[隨選通知](#on-demand-notifications)，您可以使用 `assertSentOnDemand` 方法測試隨選通知是否已發送：

```php
Notification::assertSentOnDemand(OrderShipped::class);
```

透過將閉包作為第二個參數傳遞給 `assertSentOnDemand` 方法，您可以判斷隨選通知是否發送到正確的「路由」地址：

```php
Notification::assertSentOnDemand(
    OrderShipped::class,
    function (OrderShipped $notification, array $channels, object $notifiable) use ($user) {
        return $notifiable->routes['mail'] === $user->email;
    }
);
```


<a name="notification-events"></a>
## 通知事件


<a name="notification-sending-event"></a>
#### 通知發送中事件

當通知正在發送時，通知系統會派遣 `Illuminate\Notifications\Events\NotificationSending` 事件。這包含「可通知 (notifiable)」實體和通知實體本身。您可以在應用程式中為此事件建立[事件監聽器](/docs/{{version}}/events)：

```php
use Illuminate\Notifications\Events\NotificationSending;

class CheckNotificationStatus
{
    /**
     * Handle the event.
     */
    public function handle(NotificationSending $event): void
    {
        // ...
    }
}
```

如果 `NotificationSending` 事件的事件監聽器從其 `handle` 方法回傳 `false`，則該通知將不會被發送：

```php
/**
 * Handle the event.
 */
public function handle(NotificationSending $event): bool
{
    return false;
}
```

在事件監聽器中，您可以存取事件上的 `notifiable`、`notification` 和 `channel` 屬性，以進一步了解通知收件者或通知本身：

```php
/**
 * Handle the event.
 */
public function handle(NotificationSending $event): void
{
    // $event->channel
    // $event->notifiable
    // $event->notification
}
```


<a name="notification-sent-event"></a>
#### 通知已發送事件

當通知已發送時，通知系統會派遣 `Illuminate\Notifications\Events\NotificationSent` [事件](/docs/{{version}}/events)。這包含「可通知 (notifiable)」實體和通知實體本身。您可以在應用程式中為此事件建立[事件監聽器](/docs/{{version}}/events)：

```php
use Illuminate\Notifications\Events\NotificationSent;

class LogNotification
{
    /**
     * Handle the event.
     */
    public function handle(NotificationSent $event): void
    {
        // ...
    }
}
```

在事件監聽器中，您可以存取事件上的 `notifiable`、`notification`、`channel` 和 `response` 屬性，以進一步了解通知收件者或通知本身：

```php
/**
 * Handle the event.
 */
public function handle(NotificationSent $event): void
{
    // $event->channel
    // $event->notifiable
    // $event->notification
    // $event->response
}
```

<a name="custom-channels"></a>
## 自定義頻道

Laravel 內建了一些通知頻道，但您可能想要編寫自己的驅動器以透過其他頻道傳送通知。Laravel 讓這件事變得簡單。首先，定義一個包含 `send` 方法的類別。該方法應接收兩個參數：`$notifiable` 與 `$notification`。

在 `send` 方法中，您可以呼叫通知上的方法來取得該頻道所能理解的訊息物件，然後依您喜好的方式將通知傳送給 `$notifiable` 實例：

```php
<?php

namespace App\Notifications;

use Illuminate\Notifications\Notification;

class VoiceChannel
{
    /**
     * Send the given notification.
     */
    public function send(object $notifiable, Notification $notification): void
    {
        $message = $notification->toVoice($notifiable);

        // Send notification to the $notifiable instance...
    }
}
```

一旦定義了通知頻道類別，您就可以從任何通知的 `via` 方法回傳該類別名稱。在此範例中，通知的 `toVoice` 方法可以回傳任何您選擇用來表示語音訊息的物件。例如，您可以定義自己的 `VoiceMessage` 類別來表示這些訊息：

```php
<?php

namespace App\Notifications;

use App\Notifications\Messages\VoiceMessage;
use App\Notifications\VoiceChannel;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Notification;

class InvoicePaid extends Notification
{
    use Queueable;

    /**
     * Get the notification channels.
     */
    public function via(object $notifiable): string
    {
        return VoiceChannel::class;
    }

    /**
     * Get the voice representation of the notification.
     */
    public function toVoice(object $notifiable): VoiceMessage
    {
        // ...
    }
}
```