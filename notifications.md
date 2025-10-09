# 通知

- [簡介](#introduction)
- [產生通知](#generating-notifications)
- [傳送通知](#sending-notifications)
    - [使用 Notifiable Trait](#using-the-notifiable-trait)
    - [使用 Notification Facade](#using-the-notification-facade)
    - [指定傳送管道](#specifying-delivery-channels)
    - [將通知排入佇列](#queueing-notifications)
    - [隨需通知](#on-demand-notifications)
- [郵件通知](#mail-notifications)
    - [格式化郵件訊息](#formatting-mail-messages)
    - [自訂寄件者](#customizing-the-sender)
    - [自訂收件者](#customizing-the-recipient)
    - [自訂主旨](#customizing-the-subject)
    - [自訂郵件發送器](#customizing-the-mailer)
    - [自訂模板](#customizing-the-templates)
    - [附件](#mail-attachments)
    - [新增標籤與元資料](#adding-tags-metadata)
    - [自訂 Symfony 訊息](#customizing-the-symfony-message)
    - [使用 Mailables](#using-mailables)
    - [預覽郵件通知](#previewing-mail-notifications)
- [Markdown 郵件通知](#markdown-mail-notifications)
    - [產生訊息](#generating-the-message)
    - [撰寫訊息](#writing-the-message)
    - [自訂元件](#customizing-the-components)
- [資料庫通知](#database-notifications)
    - [必要條件](#database-prerequisites)
    - [格式化資料庫通知](#formatting-database-notifications)
    - [存取通知](#accessing-the-notifications)
    - [將通知標記為已讀](#marking-notifications-as-read)
- [廣播通知](#broadcast-notifications)
    - [必要條件](#broadcast-prerequisites)
    - [格式化廣播通知](#formatting-broadcast-notifications)
    - [監聽通知](#listening-for-notifications)
- [SMS 通知](#sms-notifications)
    - [必要條件](#sms-prerequisites)
    - [格式化 SMS 通知](#formatting-sms-notifications)
    - [自訂「寄件者」號碼](#customizing-the-from-number)
    - [新增客戶參考](#adding-a-client-reference)
    - [路由 SMS 通知](#routing-sms-notifications)
- [Slack 通知](#slack-notifications)
    - [必要條件](#slack-prerequisites)
    - [格式化 Slack 通知](#formatting-slack-notifications)
    - [Slack 互動性](#slack-interactivity)
    - [路由 Slack 通知](#routing-slack-notifications)
    - [通知外部 Slack 工作區](#notifying-external-slack-workspaces)
- [本地化通知](#localizing-notifications)
- [測試](#testing)
- [通知事件](#notification-events)
- [自訂管道](#custom-channels)

<a name="introduction"></a>
## 簡介

除了支援[寄送電子郵件](/docs/{{version}}/mail)外，Laravel 還支援透過多種傳送管道傳送通知，包括電子郵件、SMS (透過 [Vonage](https://www.vonage.com/communications-apis/)，前身為 Nexmo)，以及 [Slack](https://slack.com)。此外，社群也建立了許多[通知管道](https://laravel-notification-channels.com/about/#suggesting-a-new-channel)，可以透過數十種不同的管道傳送通知！通知也可以儲存在資料庫中，以便在您的 Web 介面中顯示。

通常，通知應該是簡短的資訊性訊息，用來告知使用者應用程式中發生了某些事情。例如，如果您正在撰寫一個計費應用程式，您可能會透過電子郵件和 SMS 管道向使用者傳送「發票已支付」通知。

<a name="generating-notifications"></a>
## 產生通知

在 Laravel 中，每個通知都由一個單一類別表示，通常儲存在 `app/Notifications` 目錄中。如果您在應用程式中沒有看到這個目錄，請不用擔心，它會在您執行 `make:notification` Artisan 指令時為您建立：

```shell
php artisan make:notification InvoicePaid
```

這個指令會在您的 `app/Notifications` 目錄中放置一個全新的通知類別。每個通知類別都包含一個 `via` 方法和可變數量的訊息建構方法，例如 `toMail` 或 `toDatabase`，這些方法會將通知轉換為針對特定管道量身定做的訊息。

<a name="sending-notifications"></a>
## 傳送通知

<a name="using-the-notifiable-trait"></a>
### 使用 Notifiable Trait

通知可以透過兩種方式傳送：使用 `Notifiable` trait 的 `notify` 方法，或使用 `Notification` [Facade](/docs/{{version}}/facades)。預設情況下，`Notifiable` trait 已包含在應用程式的 `App\Models\User` 模型中：

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

這個 trait 提供的 `notify` 方法預期會收到一個 notification 實例：

```php
use App\Notifications\InvoicePaid;

$user->notify(new InvoicePaid($invoice));
```

> [!NOTE]
> 請記住，您可以在任何模型上使用 `Notifiable` trait。您不限於僅將它包含在 `User` 模型中。

<a name="using-the-notification-facade"></a>
### 使用 Notification Facade

此外，您也可以透過 `Notification` [Facade](/docs/{{version}}/facades) 來傳送通知。當您需要傳送通知給多個可通知實體 (例如多個使用者) 時，這種方法很有用。若要使用此 Facade 傳送通知，請將所有可通知的實體和 notification 實例傳遞給 `send` 方法：

```php
use Illuminate\Support\Facades\Notification;

Notification::send($users, new InvoicePaid($invoice));
```

您也可以使用 `sendNow` 方法立即傳送通知。即使 notification 實作了 `ShouldQueue` 介面，此方法也會立即傳送通知：

```php
Notification::sendNow($developers, new DeploymentCompleted($deployment));
```

<a name="specifying-delivery-channels"></a>
### 指定傳送管道

每個 notification class 都有一個 `via` 方法，用來決定 notification 將透過哪些 channel 傳送。notification 可以透過 `mail`、`database`、`broadcast`、`vonage` 和 `slack` channel 傳送。

> [!NOTE]
> 如果您想使用其他傳送 channel，例如 Telegram 或 Pusher，請查看社群維護的 [Laravel Notification Channels 網站](http://laravel-notification-channels.com)。

`via` 方法會接收一個 `$notifiable` 實例，這個實例代表了 notification 將被傳送到的 class。您可以使用 `$notifiable` 來決定 notification 應該透過哪些 channel 傳送：

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
> 在將通知排入佇列之前，您應該設定您的佇列並[啟動一個 Worker](/docs/{{version}}/queues#running-the-queue-worker)。

傳送通知可能需要一些時間，特別是當管道需要進行外部 API 呼叫來傳遞通知時。為加速您應用程式的響應時間，您可以透過將 `ShouldQueue` 介面和 `Queueable` trait 加入您的類別，讓您的通知排入佇列。這些介面與 trait 已預先匯入使用 `make:notification` 命令產生的所有通知，因此您可以立即將它們加入您的通知類別：

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

一旦 `ShouldQueue` 介面已加入您的通知，您就可以像平常一樣傳送通知。Laravel 會偵測類別上的 `ShouldQueue` 介面，並自動將通知的傳遞排入佇列：

```php
$user->notify(new InvoicePaid($invoice));
```

當通知排入佇列時，每個收件者和管道組合都會建立一個佇列工作 (queued job)。例如，如果您的通知有三個收件者和兩個管道，將會有六個工作被分派到佇列中。

<a name="delaying-notifications"></a>
#### 延遲通知

如果您想延遲通知的傳送，您可以在通知實例化時，將 `delay` 方法鏈結上去：

```php
$delay = now()->addMinutes(10);

$user->notify((new InvoicePaid($invoice))->delay($delay));
```

您可以將一個陣列傳遞給 `delay` 方法，以指定特定管道的延遲時間：

```php
$user->notify((new InvoicePaid($invoice))->delay([
    'mail' => now()->addMinutes(5),
    'sms' => now()->addMinutes(10),
]));
```

或者，您可以在通知類別本身上定義一個 `withDelay` 方法。該 `withDelay` 方法應回傳一個包含管道名稱和延遲時間的陣列：

```php
/**
 * Determine the notification's delivery delay.
 *
 * @return array<string, \Illuminate\Support\Carbon>
 */
public function withDelay(object $notifiable): array
{
    return [
        'mail' => now()->addMinutes(5),
        'sms' => now()->addMinutes(10),
    ];
}
```

<a name="customizing-the-notification-queue-connection"></a>
#### 自訂通知佇列連線

預設情況下，排入佇列的通知會使用您應用程式的預設佇列連線。如果您想為特定通知指定不同的連線，您可以在通知的建構子中呼叫 `onConnection` 方法：

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

或者，如果您想為通知支援的每個通知管道指定特定的佇列連線，您可以在通知中定義 `viaConnections` 方法。此方法應回傳一個包含管道名稱 / 佇列連線名稱配對的陣列：

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
#### 自訂通知管道佇列

如果您想為通知支援的每個通知管道指定特定的佇列，您可以在通知中定義 `viaQueues` 方法。此方法應回傳一個包含管道名稱 / 佇列名稱配對的陣列：

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
#### 自訂佇列通知工作屬性

您可以透過在通知類別上定義屬性來客製化基礎佇列工作的行為。這些屬性將由傳送通知的佇列工作所繼承：

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

如果您想透過[加密](/docs/{{version}}/encryption)來確保佇列通知資料的隱私與完整性，請將 `ShouldBeEncrypted` 介面新增到您的通知類別：

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

除了直接在通知類別上定義這些屬性之外，您還可以定義 `backoff` 和 `retryUntil` 方法來指定佇列通知工作的退避策略和重試逾時：

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
    return now()->addMinutes(5);
}
```

> [!NOTE]
> 有關這些工作屬性和方法的更多資訊，請查閱[佇列工作](/docs/{{version}}/queues#max-job-attempts-and-timeout)的文檔。

<a name="queued-notification-middleware"></a>
#### 佇列通知中介層

佇列通知可以定義中介層，[就像佇列工作一樣](/docs/{{version}}/queues#job-middleware)。若要開始，請在您的通知類別上定義一個 `middleware` 方法。該 `middleware` 方法會接收 `$notifiable` 和 `$channel` 變數，這讓您可以根據通知的目的地來客製化回傳的中介層：

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
#### 佇列通知與資料庫交易

當佇列通知在資料庫交易中被分派時，它們可能會在資料庫交易提交之前被佇列處理。此時，您在資料庫交易期間對模型或資料庫記錄所做的任何更新，可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫記錄可能尚未存在於資料庫中。如果您的通知依賴這些模型，那麼在處理傳送佇列通知的工作時，可能會發生意外錯誤。

如果您的佇列連線的 `after_commit` 設定選項設為 `false`，您仍然可以透過在傳送通知時呼叫 `afterCommit` 方法，來指出特定佇列通知應該在所有開放資料庫交易提交後才分派：

```php
use App\Notifications\InvoicePaid;

$user->notify((new InvoicePaid($invoice))->afterCommit());
```

或者，您可以從通知的建構子中呼叫 `afterCommit` 方法：

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
> 若要瞭解更多如何解決這些問題的資訊，請查閱有關[佇列工作與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的文檔。

<a name="determining-if-the-queued-notification-should-be-sent"></a>
#### 判斷佇列通知是否應該傳送

當佇列通知已分派至佇列進行背景處理後，它通常會被佇列 Worker 接受並傳送給其預期的收件者。

然而，如果您想在佇列 Worker 處理佇列通知後，最終決定該通知是否應傳送，您可以在通知類別上定義一個 `shouldSend` 方法。如果此方法回傳 `false`，則該通知將不會被傳送：

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
### 隨需通知

有時候，您可能需要向未儲存在應用程式「使用者」中的對象傳送通知。透過 `Notification` 門面的 `route` 方法，您可以在傳送通知前指定隨需通知的路由資訊：

```php
use Illuminate\Broadcasting\Channel;
use Illuminate\Support\Facades\Notification;

Notification::route('mail', 'taylor@example.com')
    ->route('vonage', '5555555555')
    ->route('slack', '#slack-channel')
    ->route('broadcast', [new Channel('channel-name')])
    ->notify(new InvoicePaid($invoice));
```

如果您想在向 `mail` 路由傳送隨需通知時提供收件者的名稱，您可以提供一個陣列，其中包含電子郵件地址作為鍵，名稱作為該陣列第一個元素的值：

```php
Notification::route('mail', [
    'barrett@example.com' => 'Barrett Blair',
])->notify(new InvoicePaid($invoice));
```

使用 `routes` 方法，您可以一次為多個通知管道提供隨需路由資訊：

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

如果通知支援以電子郵件傳送，您應該在通知類別中定義一個 `toMail` 方法。這個方法會接收一個 `$notifiable` 實體，並應該回傳一個 `Illuminate\Notifications\Messages\MailMessage` 實例。

`MailMessage` 類別包含一些簡單的方法，可協助您建構交易性電子郵件訊息。郵件訊息可以包含多行文字以及一個「行動呼籲 (call to action)」。讓我們看看一個 `toMail` 方法的範例：

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
> 請注意，我們在 `toMail` 方法中使用了 `$this->invoice->id`。您可以將通知生成其訊息所需的任何資料傳遞到通知的建構函式中。

在這個範例中，我們註冊了一個問候語、一行文字、一個行動呼籲 (call to action)，然後是另一行文字。`MailMessage` 物件提供的這些方法讓小型交易性電子郵件的格式化變得簡單快速。郵件管道 (mail channel) 會將訊息元件轉換為美觀、響應式的 HTML 電子郵件模板，並附帶一個純文字版本。以下是 `mail` 管道產生電子郵件的範例：

<img src="https://laravel.com/img/docs/notification-example-2.png" alt="一個範例通知電子郵件">

> [!NOTE]
> 傳送郵件通知時，請務必在 `config/app.php` 設定檔中設定 `name` 設定選項。此值將用於郵件通知訊息的頁首和頁尾。

<a name="error-messages"></a>
#### 錯誤訊息

有些通知會告知使用者錯誤，例如發票付款失敗。您可以在建構訊息時呼叫 `error` 方法來指出郵件訊息與錯誤有關。在郵件訊息上使用 `error` 方法時，行動呼籲按鈕將會是紅色而不是黑色：

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
#### 其他郵件通知格式選項

除了在通知類別中定義「行」文字之外，您還可以使用 `view` 方法來指定一個自訂模板，用於渲染通知電子郵件：

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

您可以使用 `view` 方法傳遞一個陣列，將視圖名稱作為陣列的第二個元素，為郵件訊息指定一個純文字視圖：

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

或者，如果您的訊息只有純文字視圖，您可以使用 `text` 方法：

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
### 自訂寄件者

預設情況下，電子郵件的寄件者 / 發件位址是在 `config/mail.php` 設定檔中定義的。然而，您可以使用 `from` 方法為特定通知指定發件位址：

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
### 自訂收件者

當透過 `mail` 管道傳送通知時，通知系統會自動在您的可通知實體上尋找 `email` 屬性。您可以透過在可通知實體上定義 `routeNotificationForMail` 方法，來自訂用於傳送通知的電子郵件位址：

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
### 自訂主旨

預設情況下，電子郵件的主旨是通知類別名稱以「標題大寫 (Title Case)」格式化的結果。因此，如果您的通知類別名稱為 `InvoicePaid`，電子郵件的主旨將是 `Invoice Paid`。如果您想為訊息指定不同的主旨，可以在建構訊息時呼叫 `subject` 方法：

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
### 自訂郵件發送器

預設情況下，電子郵件通知將使用 `config/mail.php` 設定檔中定義的預設郵件發送器 (mailer) 傳送。然而，您可以在建構訊息時呼叫 `mailer` 方法，以在執行時指定不同的郵件發送器：

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
### 自訂模板

您可以透過發佈通知套件的資源來修改郵件通知所使用的 HTML 和純文字模板。執行此命令後，郵件通知模板將位於 `resources/views/vendor/notifications` 目錄中：

```shell
php artisan vendor:publish --tag=laravel-notifications
```

<a name="mail-attachments"></a>
### 附件

若要為電子郵件通知新增附件，請在建構訊息時使用 `attach` 方法。`attach` 方法接受檔案的絕對路徑作為其第一個引數：

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
> 通知郵件訊息提供的 `attach` 方法也接受 [可附加物件](/docs/{{version}}/mail#attachable-objects)。請查閱詳盡的 [可附加物件文件](/docs/{{version}}/mail#attachable-objects) 以了解更多資訊。

當附加檔案到訊息時，您也可以透過將一個 `array` 作為第二個引數傳遞給 `attach` 方法來指定顯示名稱和/或 MIME 類型：

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

與在 Mailable 物件中附加檔案不同，您不能直接使用 `attachFromStorage` 從儲存磁碟附加檔案。您應該使用帶有儲存磁碟上檔案絕對路徑的 `attach` 方法。或者，您可以從 `toMail` 方法中回傳一個 [mailable](/docs/{{version}}/mail#generating-mailables)：

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

必要時，可以使用 `attachMany` 方法將多個檔案附加到訊息：

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

`attachData` 方法可用於將原始位元組字串作為附件附加。呼叫 `attachData` 方法時，您應提供應分配給附件的檔案名稱：

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
### 新增標籤與元資料

一些第三方電子郵件供應商，例如 Mailgun 和 Postmark，支援訊息「標籤 (tags)」和「元資料 (metadata)」，可用於對應用程式傳送的電子郵件進行分組和追蹤。您可以透過 `tag` 和 `metadata` 方法將標籤和元資料新增到電子郵件訊息中：

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

如果您的應用程式使用 Mailgun 驅動程式，您可以查閱 Mailgun 的文件以獲取有關 [標籤](https://documentation.mailgun.com/docs/mailgun/user-manual/tracking-messages/#tags) 和 [元資料](https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/#attaching-metadata-to-messages) 的更多資訊。同樣，也可以查閱 Postmark 文件以獲取有關其對 [標籤](https://postmarkapp.com/blog/tags-support-for-smtp) 和 [元資料](https://postmarkapp.com/support/article/1125-custom-metadata-faq) 支援的更多資訊。

如果您的應用程式使用 Amazon SES 傳送電子郵件，您應該使用 `metadata` 方法將 [SES 「標籤」](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html) 附加到訊息中。

<a name="customizing-the-symfony-message"></a>
### 自訂 Symfony 訊息

`MailMessage` 類別的 `withSymfonyMessage` 方法允許您註冊一個閉包，該閉包將在傳送訊息之前與 Symfony Message 實例一起被呼叫。這讓您有機會在訊息傳送之前深度自訂訊息：

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

如有需要，您可以從通知的 `toMail` 方法中回傳一個完整的 [mailable 物件](/docs/{{version}}/mail)。當回傳 `Mailable` 而非 `MailMessage` 時，您需要使用 mailable 物件的 `to` 方法來指定訊息收件人：

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
#### Mailables 與隨需通知

如果您正在傳送 [隨需通知](#on-demand-notifications)，傳遞給 `toMail` 方法的 `$notifiable` 實例將是 `Illuminate\Notifications\AnonymousNotifiable` 的實例，它提供了一個 `routeNotificationFor` 方法，可用於擷取隨需通知應傳送到的電子郵件地址：

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

在設計郵件通知模板時，方便地在瀏覽器中像典型的 Blade 模板一樣快速預覽渲染後的郵件訊息。為此，Laravel 允許您直接從路由閉包或控制器回傳由郵件通知產生的任何郵件訊息。當回傳 `MailMessage` 時，它將在瀏覽器中渲染並顯示，讓您無需將其發送到實際的電子郵件地址即可快速預覽其設計：

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

Markdown 郵件通知讓您能夠利用預建的郵件通知模板，同時擁有更大的自由度來撰寫更長、更客製化的訊息。由於訊息是以 Markdown 撰寫，Laravel 能夠為這些訊息渲染出美觀、響應式的 HTML 模板，同時也會自動產生純文字版本。

<a name="generating-the-message"></a>
### 產生訊息

若要產生帶有對應 Markdown 模板的通知，您可以使用 `make:notification` Artisan 指令的 `--markdown` 選項：

```shell
php artisan make:notification InvoicePaid --markdown=mail.invoice.paid
```

與所有其他郵件通知一樣，使用 Markdown 模板的通知應在其通知類別中定義 `toMail` 方法。然而，您不應使用 `line` 和 `action` 方法來建構通知，而是使用 `markdown` 方法來指定應使用的 Markdown 模板名稱。您可以將希望提供給模板的資料陣列作為該方法的第二個參數傳遞：

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

Markdown 郵件通知結合了 Blade 元件與 Markdown 語法，讓您能夠輕鬆建構通知，同時利用 Laravel 預先製作的通知元件：

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
> 撰寫 Markdown 郵件時，請勿使用過多的縮排。根據 Markdown 標準，Markdown 解析器會將縮排的內容渲染為程式碼區塊。

<a name="button-component"></a>
#### 按鈕元件

按鈕元件會渲染一個置中的按鈕連結。此元件接受兩個參數：一個 `url` 和一個選用的 `color`。支援的顏色有 `primary`、`green` 和 `red`。您可以在通知中加入任意數量的按鈕元件：

```blade
<x-mail::button :url="$url" color="green">
View Invoice
</x-mail::button>
```

<a name="panel-component"></a>
#### 面板元件

面板元件會將給定的文字區塊渲染在一個面板中，該面板的背景顏色與通知的其他部分略有不同。這讓您可以讓特定的文字區塊受到關注：

```blade
<x-mail::panel>
This is the panel content.
</x-mail::panel>
```

<a name="table-component"></a>
#### 表格元件

表格元件允許您將 Markdown 表格轉換為 HTML 表格。此元件接受 Markdown 表格作為其內容。表格欄位對齊支援使用預設的 Markdown 表格對齊語法：

```blade
<x-mail::table>
| Laravel       | Table         | Example       |
| ------------- | :-----------: | ------------: |
| Col 2 is      | Centered      | $10           |
| Col 3 is      | Right-Aligned | $20           |
</x-mail::table>
```

<a name="customizing-the-components"></a>
### 自訂元件

您可以將所有 Markdown 通知元件匯出到您自己的應用程式中進行客製化。若要匯出這些元件，請使用 `vendor:publish` Artisan 指令來發佈 `laravel-mail` 資產標籤：

```shell
php artisan vendor:publish --tag=laravel-mail
```

此指令將會發佈 Markdown 郵件元件到 `resources/views/vendor/mail` 目錄。`mail` 目錄將會包含 `html` 和 `text` 目錄，每個目錄都包含各自可用的每個元件的表示。您可以隨意客製化這些元件。

<a name="customizing-the-css"></a>
#### 自訂 CSS

匯出元件後，`resources/views/vendor/mail/html/themes` 目錄將會包含一個 `default.css` 檔案。您可以客製化此檔案中的 CSS，您的樣式將會自動內聯到您的 Markdown 通知在 HTML 中的表示。

如果您希望為 Laravel 的 Markdown 元件建構一個全新的主題，您可以將一個 CSS 檔案放置在 `html/themes` 目錄中。在命名並儲存您的 CSS 檔案後，請更新 `mail` 設定檔中的 `theme` 選項，以使其與您的新主題名稱相符。

若要客製化單一通知的主題，您可以在建構通知的郵件訊息時呼叫 `theme` 方法。`theme` 方法接受傳送通知時應使用的主題名稱：

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
### 必要條件

`database` 通知管道將通知資訊儲存到資料庫資料表中。此資料表將包含通知類型以及描述通知的 JSON 資料結構等資訊。

您可以查詢該資料表，以在應用程式的使用者介面中顯示通知。但在執行此動作之前，您需要建立一個資料庫資料表來儲存您的通知。您可以使用 `make:notifications-table` 命令來產生一個包含適當資料表結構的 [migration](/docs/{{version}}/migrations)：

```shell
php artisan make:notifications-table

php artisan migrate
```

> [!NOTE]
> 如果您的 notifiable 模型正在使用 [UUID 或 ULID 主鍵](/docs/{{version}}/eloquent#uuid-and-ulid-keys)，則應該在通知資料表 migration 中將 `morphs` 方法替換為 [uuidMorphs](/docs/{{version}}/migrations#column-method-uuidMorphs) 或 [ulidMorphs](/docs/{{version}}/migrations#column-method-ulidMorphs)。

<a name="formatting-database-notifications"></a>
### 格式化資料庫通知

如果通知支援儲存到資料庫資料表中，您應該在通知類別上定義 `toDatabase` 或 `toArray` 方法。此方法將接收一個 `$notifiable` 實體，並應回傳一個純粹的 PHP 陣列。回傳的陣列將編碼為 JSON，並儲存在 `notifications` 資料表的 `data` 欄位中。讓我們來看一個 `toArray` 方法的範例：

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

當通知儲存在您的應用程式資料庫中時，`type` 欄位預設情況下會設定為通知的類別名稱，而 `read_at` 欄位將為 `null`。然而，您可以透過在通知類別中定義 `databaseType` 和 `initialDatabaseReadAtValue` 方法來客製化此行為：

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

`toArray` 方法也被 `broadcast` 管道用於決定要廣播哪些資料到您的 JavaScript 驅動前端。如果您希望 `database` 和 `broadcast` 管道有兩種不同的陣列表示方式，您應該定義 `toDatabase` 方法而不是 `toArray` 方法。

<a name="accessing-the-notifications"></a>
### 存取通知

一旦通知儲存在資料庫中，您需要一種方便的方式從您的 notifiable 實體存取它們。`Illuminate\Notifications\Notifiable` trait（已包含在 Laravel 預設的 `App\Models\User` 模型中）包含一個 `notifications` [Eloquent 關聯](/docs/{{version}}/eloquent-relationships)，它會回傳該實體的通知。要取得通知，您可以像存取任何其他 Eloquent 關聯一樣存取此方法。預設情況下，通知將依 `created_at` 時間戳記排序，最新的通知位於集合的開頭：

```php
$user = App\Models\User::find(1);

foreach ($user->notifications as $notification) {
    echo $notification->type;
}
```

如果您只想取得「未讀」通知，您可以使用 `unreadNotifications` 關聯。同樣地，這些通知也將依 `created_at` 時間戳記排序，最新的通知位於集合的開頭：

```php
$user = App\Models\User::find(1);

foreach ($user->unreadNotifications as $notification) {
    echo $notification->type;
}
```

如果您只想取得「已讀」通知，您可以使用 `readNotifications` 關聯：

```php
$user = App\Models\User::find(1);

foreach ($user->readNotifications as $notification) {
    echo $notification->type;
}
```

> [!NOTE]
> 要從您的 JavaScript 客戶端存取通知，您應該為您的應用程式定義一個通知控制器，該控制器會回傳 notifiable 實體（例如當前使用者）的通知。然後，您可以從您的 JavaScript 客戶端向該控制器的 URL 發出 HTTP 請求。

<a name="marking-notifications-as-read"></a>
### 將通知標記為已讀

通常，當使用者查看通知時，您會希望將其標記為「已讀」。`Illuminate\Notifications\Notifiable` trait 提供了一個 `markAsRead` 方法，該方法會更新通知資料庫記錄上的 `read_at` 欄位：

```php
$user = App\Models\User::find(1);

foreach ($user->unreadNotifications as $notification) {
    $notification->markAsRead();
}
```

然而，您可以直接在通知集合上使用 `markAsRead` 方法，而不是遍歷每個通知：

```php
$user->unreadNotifications->markAsRead();
```

您也可以使用批量更新查詢來將所有通知標記為已讀，而無需從資料庫中取得它們：

```php
$user = App\Models\User::find(1);

$user->unreadNotifications()->update(['read_at' => now()]);
```

您可以 `delete` 通知以將它們從資料表中完全移除：

```php
$user->notifications()->delete();
```

<a name="broadcast-notifications"></a>
## 廣播通知

<a name="broadcast-prerequisites"></a>
### 必要條件

在廣播通知之前，您應該設定並熟悉 Laravel 的 [事件廣播](/docs/{{version}}/broadcasting) 服務。事件廣播提供了一種從基於 JavaScript 的前端對伺服器端 Laravel 事件作出反應的方式。

<a name="formatting-broadcast-notifications"></a>
### 格式化廣播通知

`broadcast` 管道會使用 Laravel 的 [事件廣播](/docs/{{version}}/broadcasting) 服務來廣播通知，讓您基於 JavaScript 的前端能夠即時捕獲通知。如果通知支援廣播，您可以在通知類別上定義 `toBroadcast` 方法。此方法將接收一個 `$notifiable` 實體，並且應該回傳一個 `BroadcastMessage` 實例。如果 `toBroadcast` 方法不存在，將使用 `toArray` 方法來收集應廣播的資料。回傳的資料將被編碼為 JSON 並廣播到您基於 JavaScript 的前端。讓我們看看一個 `toBroadcast` 方法的範例：

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

所有廣播通知都會排入佇列以供廣播。如果您想設定用於將廣播操作排入佇列的佇列連線或佇列名稱，您可以使用 `BroadcastMessage` 的 `onConnection` 和 `onQueue` 方法：

```php
return (new BroadcastMessage($data))
    ->onConnection('sqs')
    ->onQueue('broadcasts');
```

<a name="customizing-the-notification-type"></a>
#### 自訂通知類型

除了您指定的資料之外，所有廣播通知也都有一個 `type` 欄位，其中包含通知的完整類別名稱。如果您想自訂通知的 `type`，您可以在通知類別上定義一個 `broadcastType` 方法：

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

通知將在一個採用 `{notifiable}.{id}` 慣例格式的私有管道上廣播。因此，如果您正在向 ID 為 `1` 的 `App\Models\User` 實例傳送通知，通知將在 `App.Models.User.1` 私有管道上廣播。當使用 [Laravel Echo](/docs/{{version}}/broadcasting#client-side-installation) 時，您可以輕鬆地使用 `notification` 方法來監聽管道上的通知：

```js
Echo.private('App.Models.User.' + userId)
    .notification((notification) => {
        console.log(notification.type);
    });
```

<a name="using-react-or-vue"></a>
#### 使用 React 或 Vue

Laravel Echo 包含 React 和 Vue 的 Hooks，讓監聽通知變得輕而易舉。首先，呼叫 `useEchoNotification` Hook，這個 Hook 用於監聽通知。當消費元件被卸載時，`useEchoNotification` Hook 將自動離開管道：

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

預設情況下，這個 Hook 會監聽所有通知。若要指定您想要監聽的通知類型，您可以向 `useEchoNotification` 提供一個字串或類型陣列：

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

您也可以指定通知酬載資料的形狀，提供更高的類型安全性與編輯便利性：

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
#### 自訂通知管道

如果您想自訂實體的廣播通知會在哪個管道上廣播，您可以在可通知實體上定義 `receivesBroadcastNotificationsOn` 方法：

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
### 必要條件

在 Laravel 中傳送 SMS 通知是由 [Vonage](https://www.vonage.com/) (前身為 Nexmo) 提供支援。在使用 Vonage 傳送通知之前，您需要安裝 `laravel/vonage-notification-channel` 與 `guzzlehttp/guzzle` 套件：

```shell
composer require laravel/vonage-notification-channel guzzlehttp/guzzle
```

此套件包含一個 [設定檔](https://github.com/laravel/vonage-notification-channel/blob/3.x/config/vonage.php)。然而，您不需要將此設定檔匯出到您的應用程式中。您只需使用 `VONAGE_KEY` 與 `VONAGE_SECRET` 環境變數來定義您的 Vonage 公開與密鑰。

定義您的金鑰後，您應該設定一個 `VONAGE_SMS_FROM` 環境變數，其定義了您的 SMS 訊息預設應從哪個電話號碼傳送。您可以在 Vonage 控制面板中產生此電話號碼：

```ini
VONAGE_SMS_FROM=15556666666
```

<a name="formatting-sms-notifications"></a>
### 格式化 SMS 通知

如果通知支援以 SMS 形式傳送，您應該在通知類別上定義一個 `toVonage` 方法。此方法將接收一個 `$notifiable` 實體，並應回傳一個 `Illuminate\Notifications\Messages\VonageMessage` 實例：

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

如果您的 SMS 訊息將包含 Unicode 字元，您應該在建構 `VonageMessage` 實例時呼叫 `unicode` 方法：

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
### 自訂「寄件者」號碼

如果您想從一個與您的 `VONAGE_SMS_FROM` 環境變數所指定電話號碼不同的號碼傳送通知，您可以在 `VonageMessage` 實例上呼叫 `from` 方法：

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
### 新增客戶參考

如果您想追蹤每個使用者、團隊或客戶的成本，您可以為通知新增一個「客戶參考」。Vonage 將允許您使用此客戶參考來產生報告，以便您能更了解特定客戶的 SMS 使用情況。客戶參考可以是任何最長 40 個字元的字串：

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

若要將 Vonage 通知路由到正確的電話號碼，請在您的可通知實體上定義 `routeNotificationForVonage` 方法：

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
### 必要條件

在傳送 Slack 通知之前，您應該透過 Composer 安裝 Slack 通知管道：

```shell
composer require laravel/slack-notification-channel
```

此外，您必須為您的 Slack 工作區建立一個 [Slack App](https://api.slack.com/apps?new_app=1)。

如果您只需要向建立 App 的同一個 Slack 工作區傳送通知，您應該確保您的 App 擁有 `chat:write`、`chat:write.public` 和 `chat:write.customize` 範圍。這些範圍可以在 Slack 中的「OAuth 與權限」App 管理分頁中新增。

接下來，複製 App 的「機器人使用者 OAuth Token」，並將其放置在您應用程式的 `services.php` 設定檔中的 `slack` 設定陣列內。此 Token 可以在 Slack 中的「OAuth 與權限」分頁中找到：

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

如果您的應用程式將傳送通知給應用程式使用者擁有的外部 Slack 工作區，您將需要透過 Slack「分發」您的 App。App 分發可以在 Slack 中您 App 的「管理分發」分頁中進行管理。一旦您的 App 已分發，您可以使用 [Socialite](/docs/{{version}}/socialite) 代表您的應用程式使用者 [取得 Slack Bot token](/docs/{{version}}/socialite#slack-bot-scopes)。


<a name="formatting-slack-notifications"></a>
### 格式化 Slack 通知

如果通知支援以 Slack 訊息傳送，您應該在通知類別上定義一個 `toSlack` 方法。此方法會接收一個 `$notifiable` 實體，並應回傳一個 `Illuminate\Notifications\Slack\SlackMessage` 實例。您可以使用 [Slack 的 Block Kit API](https://api.slack.com/block-kit) 來建構豐富的通知。以下範例可在 [Slack 的 Block Kit builder](https://app.slack.com/block-kit-builder/T01KWS6K23Z#%7B%22blocks%22:%5B%7B%22type%22:%22header%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22Invoice%20Paid%22%7D%7D,%7B%22type%22:%22context%22,%22elements%22:%5B%7B%22type%22:%22plain_text%22,%22text%22:%22Customer%20%231234%22%7D%5D%7D,%7B%22type%22:%22section%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22An%20invoice%20has%20been%20paid.%22%7D,%22fields%22:%5B%7B%22type%22:%22mrkdwn%22,%22text%22:%22*Invoice%20No:*%5Cn1000%22%7D,%7B%22type%22:%22mrkdwn%22,%22text%22:%22*Invoice%20Recipient:*%5Cntaylor@laravel.com%22%7D%5D%7D,%7B%22type%22:%22divider%22%7D,%7B%22type%22:%22section%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22Congratulations!%22%7D%7D%5D%7D) 中預覽：

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
#### 使用 Slack 的 Block Kit Builder 模板

您可以使用 Slack 的 Block Kit Builder 產生的原始 JSON payload，而不是使用流暢的訊息建構方法來建構您的 Block Kit 訊息，並將其提供給 `usingBlockKitTemplate` 方法：

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

Slack 的 Block Kit 通知系統提供了強大的功能來 [處理使用者互動](https://api.slack.com/interactivity/handling)。為了利用這些功能，您的 Slack App 應啟用「Interactivity」（互動性）並設定一個指向您應用程式所服務的 URL 的「Request URL」（請求 URL）。這些設定可以在 Slack 中您的 App 的「Interactivity & Shortcuts」App 管理分頁中進行管理。

在以下範例中，該範例使用了 `actionsBlock` 方法，Slack 會向您的「Request URL」傳送一個 `POST` 請求，其中包含點擊按鈕的 Slack 使用者、點擊按鈕的 ID 等 Payload。您的應用程式可以根據此 Payload 決定要採取的動作。您還應該 [驗證請求](https://api.slack.com/authentication/verifying-requests-from-slack) 是由 Slack 發出的：

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

如果您希望使用者在執行操作前必須確認，可以在定義按鈕時呼叫 `confirm` 方法。`confirm` 方法接受一個訊息和一個接收 `ConfirmObject` 實例的閉包：

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

如果您想快速檢查您正在建構的區塊，可以在 `SlackMessage` 實例上呼叫 `dd` 方法。`dd` 方法會產生並傾印一個導向 Slack [Block Kit Builder](https://app.slack.com/block-kit-builder/) 的 URL，該工具會在您的瀏覽器中顯示 Payload 和通知的預覽。您可以傳遞 `true` 給 `dd` 方法以傾印原始 Payload：

```php
return (new SlackMessage)
    ->text('One of your invoices has been paid!')
    ->headerBlock('Invoice Paid')
    ->dd();
```

<a name="routing-slack-notifications"></a>
### 路由 Slack 通知

要將 Slack 通知導向適當的 Slack 團隊和頻道，請在您的可通知模型上定義 `routeNotificationForSlack` 方法。此方法可以回傳以下三個值之一：

-   `null` - 這會將路由延遲到通知本身中設定的頻道。您可以在建構 `SlackMessage` 時使用 `to` 方法來設定通知中的頻道。
-   一個字串，指定要傳送通知的 Slack 頻道，例如 `#support-channel`。
-   一個 `SlackRoute` 實例，它允許您指定 OAuth 權杖和頻道名稱，例如 `SlackRoute::make($this->slack_channel, $this->slack_token)`。此方法應用於將通知傳送至外部工作區。

例如，從 `routeNotificationForSlack` 方法回傳 `#support-channel` 會將通知傳送至您的應用程式 `services.php` 設定檔中與 Bot 使用者 OAuth 權杖相關聯的工作區中的 `#support-channel` 頻道：

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
> 在將通知傳送至外部 Slack 工作區之前，您的 Slack App 必須先經過 [發佈](#slack-app-distribution)。

當然，您通常會希望將通知傳送至您的應用程式使用者所擁有的 Slack 工作區。為此，您首先需要為使用者取得一個 Slack OAuth 權杖。幸運的是，[Laravel Socialite](/docs/{{version}}/socialite) 包含了一個 Slack 驅動器，可讓您輕鬆地透過 Slack 驗證您應用程式的使用者並 [取得 Bot 權杖](/docs/{{version}}/socialite#slack-bot-scopes)。

一旦您取得 Bot 權杖並將其儲存在您應用程式的資料庫中，您就可以利用 `SlackRoute::make` 方法將通知路由至使用者的工作區。此外，您的應用程式可能還需要提供一個讓使用者指定通知應傳送至哪個頻道的機會：

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
## 本地化通知

Laravel 允許您以不同於 HTTP 請求當前語系的語系來傳送通知，如果通知被排入佇列，它甚至會記住這個語系。

為此，`Illuminate\Notifications\Notification` 類別提供了 `locale` 方法來設定所需的語言。當通知正在被評估時，應用程式將會切換到這個語系，並在評估完成後切換回先前的語系：

```php
$user->notify((new InvoicePaid($invoice))->locale('es'));
```

多個可通知實體的本地化也可以透過 `Notification` Facade 來實現：

```php
Notification::locale('es')->send(
    $users, new InvoicePaid($invoice)
);
```

<a name="user-preferred-locales"></a>
#### 使用者偏好語系

有時，應用程式會儲存每個使用者的偏好語系。透過在您的可通知模型上實作 `HasLocalePreference` 契約，您可以指示 Laravel 在傳送通知時使用這個儲存的語系：

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

一旦您實作了這個介面，Laravel 在傳送通知和 Mailables 給該模型時，將會自動使用偏好語系。因此，當使用這個介面時，無需呼叫 `locale` 方法：

```php
$user->notify(new InvoicePaid($invoice));
```

<a name="testing"></a>
## 測試

您可以使用 `Notification` Facade 的 `fake` 方法來阻止通知被傳送。通常，傳送通知與您實際測試的程式碼無關。最可能的情況是，僅需斷言 Laravel 被指示傳送了特定的通知就已足夠。

呼叫 `Notification` Facade 的 `fake` 方法後，您可以斷言通知被指示傳送給使用者，甚至檢查通知接收到的資料：

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

您可以將一個閉包傳遞給 `assertSentTo` 或 `assertNotSentTo` 方法，以斷言傳送的通知通過了給定的「真實性測試」。如果至少有一個通知通過了給定的真實性測試，那麼該斷言將會成功：

```php
Notification::assertSentTo(
    $user,
    function (OrderShipped $notification, array $channels) use ($order) {
        return $notification->order->id === $order->id;
    }
);
```

<a name="on-demand-notifications"></a>
#### 隨需通知

如果您測試的程式碼傳送 [隨需通知](#on-demand-notifications)，您可以透過 `assertSentOnDemand` 方法測試該隨需通知是否已傳送：

```php
Notification::assertSentOnDemand(OrderShipped::class);
```

透過將閉包作為第二個參數傳遞給 `assertSentOnDemand` 方法，您可以判斷隨需通知是否已傳送至正確的「路由」位址：

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
#### 通知傳送事件

當通知正在傳送時，`Illuminate\Notifications\Events\NotificationSending` 事件會由通知系統分派。此事件包含「可通知」實體與通知實例本身。您可以在應用程式中為此事件建立 [事件監聽器](/docs/{{version}}/events)：

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

如果 `NotificationSending` 事件的事件監聽器從其 `handle` 方法回傳 `false`，則通知將不會被傳送：

```php
/**
 * Handle the event.
 */
public function handle(NotificationSending $event): bool
{
    return false;
}
```

在事件監聽器中，您可以存取事件上的 `notifiable`、`notification` 和 `channel` 屬性，以了解更多關於通知收件者或通知本身的資訊：

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
#### 通知已傳送事件

當通知已傳送時，`Illuminate\Notifications\Events\NotificationSent` [事件](/docs/{{version}}/events) 會由通知系統分派。此事件包含「可通知」實體與通知實例本身。您可以在應用程式中為此事件建立 [事件監聽器](/docs/{{version}}/events)：

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

在事件監聽器中，您可以存取事件上的 `notifiable`、`notification`、`channel` 和 `response` 屬性，以了解更多關於通知收件者或通知本身的資訊：

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
## 自訂管道

Laravel 內建了多個通知管道，但您可能希望編寫自己的驅動器，透過其他管道傳送通知。Laravel 讓這變得簡單。首先，定義一個包含 `send` 方法的類別。這個方法應該接收兩個引數：一個 `$notifiable` 實體和一個 `$notification` 實例。

在 `send` 方法中，您可以呼叫通知上的方法來取得您的管道所能理解的訊息物件，然後以您希望的任何方式將通知傳送給 `$notifiable` 實例：

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

一旦定義了您的通知管道類別，您可以從任何通知的 `via` 方法回傳該類別名稱。在此範例中，您通知的 `toVoice` 方法可以回傳任何您選擇用來表示語音訊息的物件。例如，您可以定義自己的 `VoiceMessage` 類別來表示這些訊息：

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