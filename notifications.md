# 通知

- [簡介](#introduction)
- [產生通知](#generating-notifications)
- [發送通知](#sending-notifications)
    - [使用 Notifiable Trait](#using-the-notifiable-trait)
    - [使用 Notification Facade](#using-the-notification-facade)
    - [指定傳遞通道](#specifying-delivery-channels)
    - [通知佇列化](#queueing-notifications)
    - [隨需通知](#on-demand-notifications)
- [郵件通知](#mail-notifications)
    - [格式化郵件訊息](#formatting-mail-messages)
    - [自訂寄件者](#customizing-the-sender)
    - [自訂收件者](#customizing-the-recipient)
    - [自訂主旨](#customizing-the-subject)
    - [自訂郵件驅動 (Mailer)](#customizing-the-mailer)
    - [自訂模板](#customizing-the-templates)
    - [附件](#mail-attachments)
    - [新增標記與元數據 (Metadata)](#adding-tags-metadata)
    - [自訂 Symfony 訊息](#customizing-the-symfony-message)
    - [使用 Mailables](#using-mailables)
    - [預覽郵件通知](#previewing-mail-notifications)
- [Markdown 郵件通知](#markdown-mail-notifications)
    - [產生訊息](#generating-the-message)
    - [撰寫訊息](#writing-the-message)
    - [自訂組件](#customizing-the-components)
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
    - [自訂「寄件」號碼](#customizing-the-from-number)
    - [新增客戶參考資料 (Client Reference)](#adding-a-client-reference)
    - [路由 SMS 通知](#routing-sms-notifications)
- [Slack 通知](#slack-notifications)
    - [先決條件](#slack-prerequisites)
    - [格式化 Slack 通知](#formatting-slack-notifications)
    - [Slack 互動性](#slack-interactivity)
    - [路由 Slack 通知](#routing-slack-notifications)
    - [通知外部 Slack 工作區](#notifying-external-slack-workspaces)
- [通知在地化](#localizing-notifications)
- [測試](#testing)
- [通知事件](#notification-events)
- [自訂通道](#custom-channels)

<a name="introduction"></a>
## 簡介

除了支援 [發送郵件](/docs/{{version}}/mail) 之外，Laravel 還支援透過多種傳遞通道發送通知，包括電子郵件、SMS（透過 [Vonage](https://www.vonage.com/communications-apis/)，原名為 Nexmo）以及 [Slack](https://slack.com)。此外，社群還建立了許多 [社群打造的通知通道](https://laravel-notification-channels.com/about/#suggesting-a-new-channel)，可以用來透過數十種不同的通道發送通知！通知也可以儲存在資料庫中，以便在您的網頁介面中顯示。

通常，通知應該是簡短且具資訊性的訊息，用來告知使用者應用程式中發生了某些事情。例如，如果您正在編寫一個帳單應用程式，您可能會透過電子郵件和 SMS 通道向使用者發送「發票已支付 (Invoice Paid)」通知。


<a name="generating-notifications"></a>
## 產生通知

在 Laravel 中，每個通知都由一個類別表示，通常儲存在 `app/Notifications` 目錄中。如果您在應用程式中沒有看到這個目錄，請不用擔心 — 當您執行 `make:notification` Artisan 指令時，系統會為您建立它：

```shell
php artisan make:notification InvoicePaid
```

此指令會在您的 `app/Notifications` 目錄中放置一個全新的通知類別。每個通知類別都包含一個 `via` 方法以及若干個訊息構建方法（例如 `toMail` 或 `toDatabase`），用來將通知轉換為針對該特定通道量身打造的訊息。

<a name="sending-notifications"></a>
## 發送通知


<a name="using-the-notifiable-trait"></a>
### 使用 Notifiable Trait

通知可以透過兩種方式發送：使用 `Notifiable` trait 的 `notify` 方法，或是使用 `Notification` [Facade](/docs/{{version}}/facades)。您的應用程式在 `App\Models\User` 模型中預設就包含了 `Notifiable` trait：

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

此 trait 提供的 `notify` 方法預期會接收一個通知實例：

```php
use App\Notifications\InvoicePaid;

$user->notify(new InvoicePaid($invoice));
```

> [!NOTE]
> 請記得，您可以在任何模型上使用 `Notifiable` trait，並不限於僅在 `User` 模型中使用。


<a name="using-the-notification-facade"></a>
### 使用 Notification Facade

或者，您也可以透過 `Notification` [Facade](/docs/{{version}}/facades) 發送通知。當您需要向多個可通知實體（例如一組使用者集合）發送通知時，這種方法非常有用。若要使用 Facade 發送通知，請將所有可通知實體與通知實例傳遞給 `send` 方法：

```php
use Illuminate\Support\Facades\Notification;

Notification::send($users, new InvoicePaid($invoice));
```

您也可以使用 `sendNow` 方法立即發送通知。即使該通知實作了 `ShouldQueue` 介面，此方法仍會立即發送通知：

```php
Notification::sendNow($developers, new DeploymentCompleted($deployment));
```


<a name="specifying-delivery-channels"></a>
### 指定傳遞通道

每個通知類別都有一個 `via` 方法，用於決定通知將透過哪些通道傳遞。通知可以透過 `mail`、`database`、`broadcast`、`vonage` 和 `slack` 通道發送。

> [!NOTE]
> 如果您想使用其他傳遞通道（例如 Telegram 或 Pusher），請參考社群驅動的 [Laravel Notification Channels 網站](http://laravel-notification-channels.com)。

`via` 方法會接收一個 `$notifiable` 實例，該實例即為接收通知的類別實例。您可以使用 `$notifiable` 來決定通知應該透過哪些通道傳遞：

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
### 通知佇列化

> [!WARNING]
> 在將通知佇列化之前，您應該先配置佇列並[啟動工作者](/docs/{{version}}/queues#running-the-queue-worker)。

發送通知可能需要一些時間，尤其是當傳遞通道需要呼叫外部 API 才能遞送通知時。為了提高應用程式的響應速度，您可以透過在類別中加入 `ShouldQueue` 介面與 `Queueable` trait 來將通知佇列化。所有使用 `make:notification` 指令產生的通知都已經匯入了該介面與 trait，因此您可以直接將其加入到通知類別中：

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

一旦 `ShouldQueue` 介面被加入到通知中，您就可以像平常一樣發送通知。Laravel 會偵測類別上的 `ShouldQueue` 介面並自動將通知的遞送放入佇列中：

```php
$user->notify(new InvoicePaid($invoice));
```

在將通知佇列化時，系統會為每個收件者與通道的組合建立一個佇列工作 (queued job)。例如，如果您的通知有三個收件者和兩個通道，將會有六個工作被發送到佇列中。


<a name="delaying-notifications"></a>
#### 延遲通知

如果您想延遲通知的遞送，可以在實例化通知時鏈接 `delay` 方法：

```php
$delay = now()->plus(minutes: 10);

$user->notify((new InvoicePaid($invoice))->delay($delay));
```

您可以將陣列傳遞給 `delay` 方法，以指定特定通道的延遲量：

```php
$user->notify((new InvoicePaid($invoice))->delay([
    'mail' => now()->plus(minutes: 5),
    'sms' => now()->plus(minutes: 10),
]));
```

或者，您也可以在通知類別本身定義 `withDelay` 方法。`withDelay` 方法應回傳一個包含通道名稱與延遲值的陣列：

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
#### 自訂通知佇列連線

預設情況下，佇列通知將使用應用程式的預設佇列連線。如果您想為特定通知指定不同的連線，可以在通知的建構子中呼叫 `onConnection` 方法：

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

或者，如果您想為通知所支援的每個通知通道指定特定的佇列連線，可以在通知中定義 `viaConnections` 方法。此方法應回傳通道名稱與佇列連線名稱配對的陣列：

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
#### 自訂通知通道佇列

如果您想為通知所支援的每個通知通道指定特定的佇列，可以在通知中定義 `viaQueues` 方法。此方法應回傳通道名稱與佇列名稱配對的陣列：

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

您可以在通知類別上定義佇列屬性，藉此自訂底層佇列工作的行為。這些屬性將被發送通知的佇列工作繼承：

```php
<?php

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Notification;
use Illuminate\Queue\Attributes\MaxExceptions;
use Illuminate\Queue\Attributes\Timeout;
use Illuminate\Queue\Attributes\Tries;

#[Tries(5)]
#[Timeout(120)]
#[MaxExceptions(3)]
class InvoicePaid extends Notification implements ShouldQueue
{
    use Queueable;

    // ...
}
```

如果您想透過[加密](/docs/{{version}}/encryption)來確保佇列通知數據的隱私與完整性，請在通知類別中加入 `ShouldBeEncrypted` 介面：

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

除了直接在通知類別上定義這些屬性外，您還可以定義 `backoff` 與 `retryUntil` 方法，以指定佇列通知工作的退避策略 (backoff strategy) 與重試逾時：

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
> 如需更多關於這些工作屬性與方法的資訊，請參閱[佇列工作](/docs/{{version}}/queues#max-job-attempts-and-timeout)的說明文件。


<a name="queued-notification-middleware"></a>
#### 佇列通知中介層

佇列通知可以像[佇列工作](/docs/{{version}}/queues#job-middleware)一樣定義中介層。首先，在通知類別中定義 `middleware` 方法。`middleware` 方法會接收 `$notifiable` 與 `$channel` 變數，讓您能根據通知的目的地自訂回傳的中介層：

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

當佇列通知在資料庫交易中被發送時，它們可能會在資料庫交易提交之前就被佇列處理。當這種情況發生時，您在資料庫交易期間對模型或資料庫紀錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫紀錄可能還不存在於資料庫中。如果您的通知依賴於這些模型，當處理發送佇列通知的工作時，可能會發生非預期的錯誤。

如果您的佇列連線 `after_commit` 配置選項設定為 `false`，您仍然可以在發送通知時呼叫 `afterCommit` 方法，來指定特定的佇列通知應在所有開啟的資料庫交易提交後才發送：

```php
use App\Notifications\InvoicePaid;

$user->notify((new InvoicePaid($invoice))->afterCommit());
```

或者，您也可以在通知的建構子中呼叫 `afterCommit` 方法：

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
> 若要深入了解如何解決這些問題，請參閱關於[佇列工作與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的說明文件。


<a name="determining-if-the-queued-notification-should-be-sent"></a>
#### 決定佇列通知是否應發送

當佇列通知被發送到佇列進行背景處理後，通常會由佇列工作者接收並發送給預定的收件者。

然而，如果您想在佇列工作者處理通知後，才做出是否發送該佇列通知的最終決定，您可以在通知類別上定義 `shouldSend` 方法。如果此方法回傳 `false`，則通知將不會被發送：

```php
/**
 * Determine if the notification should be sent.
 */
public function shouldSend(object $notifiable, string $channel): bool
{
    return $this->invoice->isPaid();
}
```


<a name="after-sending-notifications"></a>
#### 發送通知之後

如果您想在通知發送後執行程式碼，可以在通知類別上定義 `afterSending` 方法。此方法會接收可通知實體 (notifiable entity)、通道名稱以及來自該通道的響應：

```php
/**
 * Handle the notification after it has been sent.
 */
public function afterSending(object $notifiable, string $channel, mixed $response): void
{
    // ...
}
```

<a name="on-demand-notifications"></a>
### 隨需通知

有時候您可能需要向不被儲存為應用程式「使用者」的人發送通知。透過使用 `Notification` Facade 的 `route` 方法，您可以在發送通知前指定臨時的通知路由資訊：

```php
use Illuminate\Broadcasting\Channel;
use Illuminate\Support\Facades\Notification;

Notification::route('mail', 'taylor@example.com')
    ->route('vonage', '5555555555')
    ->route('slack', '#slack-channel')
    ->route('broadcast', [new Channel('channel-name')])
    ->notify(new InvoicePaid($invoice));
```

如果您在向 `mail` 路由發送隨需通知時想要提供收件者的姓名，您可以提供一個陣列，將電子郵件地址作為鍵 (key)，姓名作為該陣列第一個元素的值 (value)：

```php
Notification::route('mail', [
    'barrett@example.com' => 'Barrett Blair',
])->notify(new InvoicePaid($invoice));
```

使用 `routes` 方法，您可以一次提供多個通知通道的臨時路由資訊：

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

如果通知支援以電子郵件發送，您應該在通知類別中定義一個 `toMail` 方法。此方法將接收一個 `$notifiable` 實體，並應回傳一個 `Illuminate\Notifications\Messages\MailMessage` 實例。

`MailMessage` 類別包含一些簡單的方法來協助您建構交易式電子郵件訊息。郵件訊息可以包含多行文字以及一個「行動呼籲 (call to action)」。讓我們來看看 `toMail` 方法的範例：

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
> 請注意，我們在 `toMail` 方法中使用了 `$this->invoice->id`。您可以將通知產生訊息所需的任何資料傳遞到通知的建構子中。

在這個範例中，我們註冊了一個問候語、一行文字、一個行動呼籲，接著又是另一行文字。`MailMessage` 物件提供的這些方法讓格式化小型交易式電子郵件變得簡單且快速。郵件通道隨後會將這些訊息組件轉換為美觀且響應式的 HTML 電子郵件模板，並附帶一個純文字版本。以下是透過 `mail` 通道產生的電子郵件範例：

<img src="https://laravel.com/img/docs/notification-example-2.png">

> [!NOTE]
> 發送郵件通知時，請務必在 `config/app.php` 設定檔中設定 `name` 設定選項。此值將用於郵件通知訊息的頁首和頁尾。


<a name="error-messages"></a>
#### 錯誤訊息

某些通知是用於告知使用者錯誤資訊的，例如發票付款失敗。您可以在建構訊息時呼叫 `error` 方法，來表示該郵件訊息與錯誤相關。當在郵件訊息中使用 `error` 方法時，行動呼籲按鈕將顯示為紅色而非黑色：

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

您可以選擇使用 `view` 方法來指定一個自訂模板來渲染通知郵件，而不是在通知類別中定義多行文字：

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

您可以透過將視圖名稱作為傳遞給 `view` 方法之陣列的第二個元素，來為郵件訊息指定一個純文字視圖：

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

或者，如果您的訊息僅有純文字視圖，可以使用 `text` 方法：

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

預設情況下，電子郵件的寄件者 / 寄件地址定義在 `config/mail.php` 設定檔中。不過，您可以使用 `from` 方法為特定的通知指定寄件地址：

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

透過 `mail` 通道發送通知時，通知系統會自動在您的可通知實體上尋找 `email` 屬性。您可以透過在可通知實體上定義 `routeNotificationForMail` 方法，來自訂用於遞送通知的電子郵件地址：

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

預設情況下，電子郵件的主旨是通知類別名稱的「首字母大寫 (Title Case)」格式。因此，如果您的通知類別名稱為 `InvoicePaid`，電子郵件的主旨將會是 `Invoice Paid`。如果您想為訊息指定不同的主旨，可以在建構訊息時呼叫 `subject` 方法：

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
### 自訂郵件驅動 (Mailer)

預設情況下，電子郵件通知將使用 `config/mail.php` 設定檔中定義的預設郵件驅動。不過，您可以在執行時透過呼叫 `mailer` 方法來指定不同的郵件驅動：

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

您可以透過發布通知套件的資源來修改郵件通知所使用的 HTML 和純文字模板。執行此指令後，郵件通知模板將位於 `resources/views/vendor/notifications` 目錄中：

```shell
php artisan vendor:publish --tag=laravel-notifications
```

<a name="mail-attachments"></a>
### 附件

要在郵件通知中新增附件，請在建構訊息時使用 `attach` 方法。`attach` 方法的第一個引數為檔案的絕對路徑：

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
> 郵件通知訊息提供的 `attach` 方法也接受 [可附加物件 (attachable objects)](/docs/{{version}}/mail#attachable-objects)。請參閱詳盡的 [可附加物件文件](/docs/{{version}}/mail#attachable-objects) 以了解更多資訊。

在為訊息附加檔案時，您也可以透過將 `array` 作為 `attach` 方法的第二個引數，來指定顯示名稱和/或 MIME 類型：

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

與在 Mailable 物件中附加檔案不同，您不能使用 `attachFromStorage` 直接從儲存磁碟附加檔案。您應該使用 `attach` 方法並提供儲存磁碟上檔案的絕對路徑。或者，您可以從 `toMail` 方法回傳一個 [mailable](/docs/{{version}}/mail#generating-mailables)：

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

必要時，可以使用 `attachMany` 方法為訊息附加多個檔案：

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

`attachData` 方法可用於將原始位元組字串作為附件附加。呼叫 `attachData` 方法時，您應該提供要分配給該附件的檔案名稱：

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
### 新增標記與元數據 (Metadata)

某些第三方郵件提供者（如 Mailgun 和 Postmark）支援訊息「標記 (tags)」和「元數據 (metadata)」，可用於對應用程式發送的郵件進行分組和追蹤。您可以使用 `tag` 和 `metadata` 方法將標記和元數據新增至郵件訊息中：

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

如果您的應用程式使用 Mailgun 驅動，您可以參考 Mailgun 的文件以獲取更多關於 [標記 (tags)](https://documentation.mailgun.com/docs/mailgun/user-manual/tracking-messages/#tags) 和 [元數據 (metadata)](https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/#attaching-metadata-to-messages) 的資訊。同樣地，也可以參考 Postmark 的文件以獲取更多關於其對 [標記 (tags)](https://postmarkapp.com/blog/tags-support-for-smtp) 和 [元數據 (metadata)](https://postmarkapp.com/support/article/1125-custom-metadata-faq) 支援的資訊。

如果您的應用程式使用 Amazon SES 發送郵件，您應該使用 `metadata` 方法將 [SES 「標記 (tags)」](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html) 附加到訊息中。


<a name="customizing-the-symfony-message"></a>
### 自訂 Symfony 訊息

`MailMessage` 類別的 `withSymfonyMessage` 方法允許您註冊一個閉包 (closure)，該閉包會在發送訊息之前，帶著 Symfony Message 實例被呼叫。這讓您有機會在訊息送達前對其進行深度自訂：

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

如果需要，您可以從通知的 `toMail` 方法回傳一個完整的 [Mailable 物件](/docs/{{version}}/mail)。當回傳 `Mailable` 而非 `MailMessage` 時，您需要使用 Mailable 物件的 `to` 方法來指定訊息收件者：

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

如果您正在發送 [隨需通知](#on-demand-notifications)，傳遞給 `toMail` 方法的 `$notifiable` 實例將會是 `Illuminate\Notifications\AnonymousNotifiable` 的實例，它提供了一個 `routeNotificationFor` 方法，可用於獲取隨需通知應發送的電子郵件地址：

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

在設計郵件通知模板時，像一般的 Blade 模板一樣，在瀏覽器中快速預覽渲染後的郵件訊息會非常方便。因此，Laravel 允許您直接從路由閉包或控制器回傳由郵件通知產生的任何郵件訊息。當回傳 `MailMessage` 時，它將被渲染並顯示在瀏覽器中，讓您無需將其發送到實際的電子郵件地址即可快速預覽其設計：

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

Markdown 郵件通知讓您能利用郵件通知的預建模板，同時在撰寫較長且自訂的訊息時擁有更多自由度。由於訊息是以 Markdown 撰寫，Laravel 能夠為這些訊息渲染出精美且具響應式的 HTML 模板，並自動產生對應的純文字版本。


<a name="generating-the-message"></a>
### 產生訊息

若要產生一個帶有對應 Markdown 模板的通知，您可以使用 `make:notification` Artisan 命令的 `--markdown` 選項：

```shell
php artisan make:notification InvoicePaid --markdown=mail.invoice.paid
```

與所有其他郵件通知一樣，使用 Markdown 模板的通知應該在其通知類別中定義 `toMail` 方法。然而，請使用 `markdown` 方法來指定要使用的 Markdown 模板名稱，而不是使用 `line` 和 `action` 方法來建構通知。您可以將想要提供給模板的資料陣列作為該方法的第二個引數傳入：

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

Markdown 郵件通知結合了 Blade 組件與 Markdown 語法，讓您在利用 Laravel 預先製作的通知組件時，能輕鬆地建構通知：

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
> 在撰寫 Markdown 郵件時，請勿使用過多的縮排。根據 Markdown 標準，Markdown 解析器會將縮排的內容渲染為程式碼區塊。


<a name="button-component"></a>
#### 按鈕組件

按鈕組件會渲染一個置中的按鈕連結。該組件接受兩個參數：`url` 以及一個選填的 `color`。支援的顏色包括 `primary`、`green` 和 `red`。您可以根據需求在通知中加入任意數量的按鈕組件：

```blade
<x-mail::button :url="$url" color="green">
View Invoice
</x-mail::button>
```


<a name="panel-component"></a>
#### 面板組件

面板組件會將給定的文字區塊渲染在一個背景顏色與通知其餘部分略有不同的面板中。這讓您能夠吸引讀者對特定文字區塊的注意：

```blade
<x-mail::panel>
This is the panel content.
</x-mail::panel>
```


<a name="table-component"></a>
#### 表格組件

表格組件允許您將 Markdown 表格轉換為 HTML 表格。該組件將 Markdown 表格作為其內容。表格欄位的對齊方式支援使用預設的 Markdown 表格對齊語法：

```blade
<x-mail::table>
| Laravel       | Table         | Example       |
| ------------- | :-----------: | ------------: |
| Col 2 is      | Centered      | $10           |
| Col 3 is      | Right-Aligned | $20           |
</x-mail::table>
```


<a name="customizing-the-components"></a>
### 自訂組件

您可以將所有 Markdown 通知組件匯出到自己的應用程式中以便自訂。若要匯出組件，請使用 `vendor:publish` Artisan 命令來發布 `laravel-mail` 資產標記：

```shell
php artisan vendor:publish --tag=laravel-mail
```

此命令會將 Markdown 郵件組件發布到 `resources/views/vendor/mail` 目錄。`mail` 目錄將包含 `html` 和 `text` 兩個子目錄，每個目錄分別包含所有可用組件的對應表示形式。您可以隨意自訂這些組件。


<a name="customizing-the-css"></a>
#### 自訂 CSS

匯出組件後，`resources/views/vendor/mail/html/themes` 目錄將包含一個 `default.css` 檔案。您可以自訂此檔案中的 CSS，您的樣式將自動內嵌 (in-lined) 在 Markdown 通知之 HTML 表示形式中。

如果您想為 Laravel 的 Markdown 組件打造一套全新的主題，可以在 `html/themes` 目錄中放置一個 CSS 檔案。命名並儲存 CSS 檔案後，請更新 `mail` 設定檔中的 `theme` 選項，使其與新主題的名稱一致。

若要為單一通知自訂主題，您可以在建構通知的郵件訊息時呼叫 `theme` 方法。`theme` 方法接受發送通知時應使用的主題名稱：

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

`database` 通知通道將通知資訊儲存在資料庫表中。此表將包含通知類型以及描述該通知的 JSON 資料結構等資訊。

您可以查詢該表以在應用程式的使用者介面中顯示通知。但在執行此操作之前，您需要建立一個資料庫表來存放通知。您可以使用 `make:notifications-table` 指令來產生一個具有正確表結構的 [migration](/docs/{{version}}/migrations)：

```shell
php artisan make:notifications-table

php artisan migrate
```

> [!NOTE]
> 如果您的可通知模型使用 [UUID 或 ULID 主鍵](/docs/{{version}}/eloquent#uuid-and-ulid-keys)，您應該在通知表的 migration 中將 `morphs` 方法替換為 [uuidMorphs](/docs/{{version}}/migrations#column-method-uuidMorphs) 或 [ulidMorphs](/docs/{{version}}/migrations#column-method-ulidMorphs)。


<a name="formatting-database-notifications"></a>
### 格式化資料庫通知

如果通知支援儲存在資料庫表中，您應該在通知類別中定義 `toDatabase` 或 `toArray` 方法。此方法將接收一個 `$notifiable` 實體，並應回傳一個純 PHP 陣列。回傳的陣列將被編碼為 JSON 並儲存在 `notifications` 表的 `data` 欄位中。讓我們來看看一個 `toArray` 方法的範例：

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

當通知儲存在應用程式的資料庫中時，`type` 欄位預設會被設定為通知的類別名稱，而 `read_at` 欄位將為 `null`。然而，您可以通过在通知類別中定義 `databaseType` 和 `initialDatabaseReadAtValue` 方法來自訂此行為：

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

`toArray` 方法也被 `broadcast` 通道用來決定要廣播哪些資料到您的 JavaScript 前端。如果您希望 `database` 和 `broadcast` 通道具有兩種不同的陣列表示形式，您應該定義 `toDatabase` 方法而非 `toArray` 方法。


<a name="accessing-the-notifications"></a>
### 存取通知

一旦通知儲存在資料庫中，您需要一種方便的方式從可通知實體存取它們。包含在 Laravel 預設 `App\Models\User` 模型中的 `Illuminate\Notifications\Notifiable` trait，提供了一個 `notifications` [Eloquent 關聯](/docs/{{version}}/eloquent-relationships)，可用於回傳該實體的通知。要獲取通知，您可以像存取任何其他 Eloquent 關聯一樣存取此方法。預設情況下，通知將根據 `created_at` 時間戳記進行排序，最新的通知會位於集合的開頭：

```php
$user = App\Models\User::find(1);

foreach ($user->notifications as $notification) {
    echo $notification->type;
}
```

如果您只想檢索「未讀」通知，可以使用 `unreadNotifications` 關聯。同樣地，這些通知將根據 `created_at` 時間戳記進行排序，最新的通知會位於集合的開頭：

```php
$user = App\Models\User::find(1);

foreach ($user->unreadNotifications as $notification) {
    echo $notification->type;
}
```

如果您只想檢索「已讀」通知，可以使用 `readNotifications` 關聯：

```php
$user = App\Models\User::find(1);

foreach ($user->readNotifications as $notification) {
    echo $notification->type;
}
```

> [!NOTE]
> 要從 JavaScript 客戶端存取通知，您應該為應用程式定義一個通知控制器，用以回傳可通知實體（例如當前使用者）的通知。然後，您可以從 JavaScript 客戶端向該控制器的 URL 發送 HTTP 請求。


<a name="marking-notifications-as-read"></a>
### 將通知標記為已讀

通常，您會在使用者查看通知時將其標記為「已讀」。`Illuminate\Notifications\Notifiable` trait 提供了一個 `markAsRead` 方法，可用於更新通知資料庫記錄中的 `read_at` 欄位：

```php
$user = App\Models\User::find(1);

foreach ($user->unreadNotifications as $notification) {
    $notification->markAsRead();
}
```

然而，您不必迴圈處理每個通知，可以直接在通知集合上使用 `markAsRead` 方法：

```php
$user->unreadNotifications->markAsRead();
```

您也可以使用大量更新查詢來將所有通知標記為已讀，而無需從資料庫中檢索它們：

```php
$user = App\Models\User::find(1);

$user->unreadNotifications()->update(['read_at' => now()]);
```

您可以使用 `delete` 將通知從表中完全移除：

```php
$user->notifications()->delete();
```

<a name="broadcast-notifications"></a>
## 廣播通知


<a name="broadcast-prerequisites"></a>
### 先決條件

在進行廣播通知之前，您應該配置並熟悉 Laravel 的 [事件廣播](/docs/{{version}}/broadcasting) 服務。事件廣播提供了一種方式，讓您的 JavaScript 前端能夠對伺服器端的 Laravel 事件做出反應。


<a name="formatting-broadcast-notifications"></a>
### 格式化廣播通知

`broadcast` 通道使用 Laravel 的 [事件廣播](/docs/{{version}}/broadcasting) 服務來廣播通知，讓您的 JavaScript 前端能即時接收到通知。如果通知支援廣播，您可以在通知類別中定義 `toBroadcast` 方法。此方法將接收一個 `$notifiable` 實體，並應回傳一個 `BroadcastMessage` 實例。如果 `toBroadcast` 方法不存在，則會使用 `toArray` 方法來收集應廣播的資料。回傳的資料將被編碼為 JSON 並廣播到您的 JavaScript 前端。讓我們來看看 `toBroadcast` 方法的範例：

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
#### 廣播佇列配置

所有廣播通知都會被放入佇列中進行廣播。如果您想要配置用於廣播操作的佇列連線或佇列名稱，可以使用 `BroadcastMessage` 的 `onConnection` 與 `onQueue` 方法：

```php
return (new BroadcastMessage($data))
    ->onConnection('sqs')
    ->onQueue('broadcasts');
```


<a name="customizing-the-notification-type"></a>
#### 自訂通知類型

除了您指定的資料外，所有廣播通知還包含一個 `type` 欄位，其中包含通知的完整類別名稱。如果您想要自訂通知的 `type`，可以在通知類別中定義 `broadcastType` 方法：

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

通知將在一個使用 `{notifiable}.{id}` 慣例格式化的私有通道上廣播。因此，如果您將通知發送給 ID 為 `1` 的 `App\Models\User` 實例，通知將在 `App.Models.User.1` 私有通道上廣播。當使用 [Laravel Echo](/docs/{{version}}/broadcasting#client-side-installation) 時，您可以使用 `notification` 方法輕鬆地監聽通道上的通知：

```js
Echo.private('App.Models.User.' + userId)
    .notification((notification) => {
        console.log(notification.type);
    });
```


<a name="using-react-or-vue"></a>
#### 使用 React, Vue 或 Svelte

Laravel Echo 包含了 React、Vue 與 Svelte 的 hooks，讓監聽通知變得非常簡單。要開始使用，請呼叫 `useEchoNotification` hook 來監聽通知。當使用該 hook 的組件被卸載 (unmounted) 時，`useEchoNotification` hook 會自動離開通道：

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

```svelte tab=Svelte
<script>
import { useEchoNotification } from "@laravel/echo-svelte";

useEchoNotification(
    `App.Models.User.${userId}`,
    (notification) => {
        console.log(notification.type);
    },
);
</script>
```

預設情況下，此 hook 會監聽所有通知。若要指定您想要監聽的通知類型，您可以向 `useEchoNotification` 提供一個字串或類型陣列：

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

```svelte tab=Svelte
<script>
import { useEchoNotification } from "@laravel/echo-svelte";

useEchoNotification(
    `App.Models.User.${userId}`,
    (notification) => {
        console.log(notification.type);
    },
    'App.Notifications.InvoicePaid',
);
</script>
```

您也可以指定通知有效載荷 (payload) 資料的形狀，以提供更好的型別安全與編輯便利性：

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
#### 自訂通知通道

如果您想要自訂實體的廣播通知在哪个通道上廣播，可以在可通知實體上定義 `receivesBroadcastNotificationsOn` 方法：

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

Laravel 的 SMS 通知功能是由 [Vonage](https://www.vonage.com/) (原名 Nexmo) 所驅動。在透過 Vonage 發送通知之前，您需要安裝 `laravel/vonage-notification-channel` 與 `guzzlehttp/guzzle` 套件：

```shell
composer require laravel/vonage-notification-channel guzzlehttp/guzzle
```

此套件包含一個 [設定檔](https://github.com/laravel/vonage-notification-channel/blob/3.x/config/vonage.php)。不過，您不需要將此設定檔匯出到您的應用程式中。您只需使用 `VONAGE_KEY` 與 `VONAGE_SECRET` 環境變數來定義您的 Vonage 公鑰與私鑰即可。

定義金鑰後，您應該設定一個 `VONAGE_SMS_FROM` 環境變數，用以定義您的 SMS 訊息預設的寄件電話號碼。您可以在 Vonage 控制面板中產生此電話號碼：

```ini
VONAGE_SMS_FROM=15556666666
```


<a name="formatting-sms-notifications"></a>
### 格式化 SMS 通知

如果一個通知支援以 SMS 形式發送，您應該在通知類別中定義一個 `toVonage` 方法。此方法將接收一個 `$notifiable` 實體，並應回傳一個 `Illuminate\Notifications\Messages\VonageMessage` 實例：

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
### 自訂「寄件」號碼

如果您希望從一個與 `VONAGE_SMS_FROM` 環境變數中指定的號碼不同的電話號碼發送某些通知，您可以在 `VonageMessage` 實例上呼叫 `from` 方法：

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
### 新增客戶參考資料 (Client Reference)

如果您想要追蹤每個使用者、團隊或客戶的費用，您可以為通知新增「客戶參考資料 (client reference)」。Vonage 將允許您使用此客戶參考資料來產生報表，以便您更了解特定客戶的 SMS 使用情況。客戶參考資料可以是任何長度最多 40 個字元的字串：

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
### 先決條件

在發送 Slack 通知之前，您應該透過 Composer 安裝 Slack 通知通道：

```shell
composer require laravel/slack-notification-channel
```

此外，您必須為您的 Slack 工作區建立一個 [Slack App](https://api.slack.com/apps?new_app=1)。

如果您只需要將通知發送到建立該 App 的同一個 Slack 工作區，請確保您的 App 具有 `chat:write`、`chat:write.public` 和 `chat:write.customize` 權限範圍 (scopes)。這些權限範圍可以從 Slack 內 App 管理分頁的 "OAuth & Permissions" 中新增。

接下來，複製 App 的 "Bot User OAuth Token"，並將其放置在應用程式 `services.php` 設定檔的 `slack` 設定陣列中。此令牌可以在 Slack 的 "OAuth & Permissions" 分頁中找到：

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

如果您的應用程式將發送通知到由應用程式使用者所擁有的外部 Slack 工作區，您將需要透過 Slack 「分發 (distribute)」您的 App。App 分發可以從 Slack 內 App 的 "Manage Distribution" 分頁中進行管理。一旦您的 App 已分發，您可以使用 [Socialite](/docs/{{version}}/socialite) 代表您的應用程式使用者 [獲取 Slack Bot 令牌](/docs/{{version}}/socialite#slack-bot-scopes)。


<a name="formatting-slack-notifications"></a>
### 格式化 Slack 通知

如果通知支援以 Slack 訊息發送，您應該在通知類別中定義一個 `toSlack` 方法。此方法將接收一個 `$notifiable` 實體，並應回傳一個 `Illuminate\Notifications\Slack\SlackMessage` 執行個體。您可以使用 [Slack's Block Kit API](https://api.slack.com/block-kit) 來構建豐富的通知。以下範例可以在 [Slack's Block Kit builder](https://app.slack.com/block-kit-builder/T01KWS6K23Z#%7B%22blocks%22:%5B%7B%22type%22:%22header%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22Invoice%20Paid%22%7D%7D,%7B%22type%22:%22context%22,%22elements%22:%5B%7B%22type%22:%22plain_text%22,%22text%22:%22Customer%20%231234%22%7D%5D%7D,%7B%22type%22:%22section%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22An%20invoice%20has%20been%20paid.%22%7D,%22fields%22:%5B%7B%22type%22:%22mrkdwn%22,%22text%22:%22*Invoice%20No:*%5Cn1000%22%7D,%7B%22type%22:%22mrkdwn%22,%22text%22:%22*Invoice%20Recipient:*%5Cntaylor@laravel.com%22%7D%5D%7D,%7B%22type%22:%22divider%22%7D,%7B%22type%22:%22section%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22Congratulations!%22%7D%7D%5D%7D) 中預覽：

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
#### 使用 Slack's Block Kit Builder 模板

與其使用流暢的訊息構建方法來建構您的 Block Kit 訊息，您也可以將 Slack Block Kit Builder 產生的原始 JSON 負載 (payload) 提供給 `usingBlockKitTemplate` 方法：

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

Slack 的 Block Kit 通知系統提供了強大的功能來 [處理使用者互動](https://api.slack.com/interactivity/handling)。要利用這些功能，您的 Slack App 應該啟用 「Interactivity」 並設定一個指向您應用程式提供之 URL 的 「Request URL」。這些設定可以在 Slack 的 「Interactivity & Shortcuts」 App 管理分頁中進行管理。

在以下利用 `actionsBlock` 方法的範例中，Slack 會發送一個 `POST` 請求到您的 「Request URL」，其內容（payload）包含點擊按鈕的 Slack 使用者、被點擊按鈕的 ID 等資訊。您的應用程式接著可以根據該內容來決定要執行的動作。您還應該 [驗證該請求](https://api.slack.com/authentication/verifying-requests-from-slack) 是由 Slack 發出的：

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
#### 確認模態視窗 (Confirmation Modals)

如果您希望使用者在執行動作之前必須先確認，您可以在定義按鈕時呼叫 `confirm` 方法。`confirm` 方法接受一個訊息以及一個接收 `ConfirmObject` 實例的閉包：

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

如果您想快速檢查您所建構的區塊，可以在 `SlackMessage` 實例上呼叫 `dd` 方法。`dd` 方法會產生並印出一個指向 Slack [Block Kit Builder](https://app.slack.com/block-kit-builder/) 的 URL，讓您可以在瀏覽器中預覽內容（payload）與通知。您可以向 `dd` 方法傳遞 `true` 來印出原始的內容：

```php
return (new SlackMessage)
    ->text('One of your invoices has been paid!')
    ->headerBlock('Invoice Paid')
    ->dd();
```

<a name="routing-slack-notifications"></a>
### 路由 Slack 通知

若要將 Slack 通知導向適當的 Slack 團隊與通道，請在您的可通知模型 (notifiable model) 上定義 `routeNotificationForSlack` 方法。此方法可以回傳以下三種值之一：

- `null`：將路由延遲至在通知本身中設定的通道。您可以在建構 `SlackMessage` 時使用 `to` 方法來設定通知內的通道。
- 指定要發送通知之 Slack 通道的字串，例如 `#support-channel`。
- `SlackRoute` 實例，允許您指定 OAuth 令牌與通道名稱，例如 `SlackRoute::make($this->slack_channel, $this->slack_token)`。此方法應被用於發送通知到外部工作區。

例如，從 `routeNotificationForSlack` 方法回傳 `#support-channel` 將會把通知發送到與您應用程式 `services.php` 設定檔中 Bot User OAuth 令牌相關聯的工作區之 `#support-channel` 通道：

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
> 在發送通知到外部 Slack 工作區之前，您的 Slack App 必須先經過 [分發(distributed)](#slack-app-distribution)。

當然，您通常會想將通知發送到由您應用程式使用者所擁有的 Slack 工作區。為此，您首先需要為該使用者獲取一個 Slack OAuth 令牌。幸運的是，[Laravel Socialite](/docs/{{version}}/socialite) 包含了一個 Slack 驅動程式，讓您可以輕鬆地使用 Slack 認證您的應用程式使用者並 [獲取 Bot 令牌](/docs/{{version}}/socialite#slack-bot-scopes)。

一旦您獲取了 Bot 令牌並將其儲存在應用程式的資料庫中，您可以使用 `SlackRoute::make` 方法將通知路由到該使用者的工作區。此外，您的應用程式可能需要提供讓使用者指定通知應發送到哪個通道的機會：

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
## 通知在地化

Laravel 允許您以與目前 HTTP 請求不同的在地化語言 (locale) 發送通知，且即使通知被放入佇列中，它也會記得這個在地化語言。

為了實現這一點，`Illuminate\Notifications\Notification` 類別提供了一個 `locale` 方法來設定所需的語言。當通知被評估時，應用程式會切換到該在地化語言，並在評估完成後恢復到之前的語言：

```php
$user->notify((new InvoicePaid($invoice))->locale('es'));
```

您也可以透過 `Notification` Facade 來為多個可通知實體進行在地化：

```php
Notification::locale('es')->send(
    $users, new InvoicePaid($invoice)
);
```


<a name="user-preferred-locales"></a>
#### 使用者偏好在地化語言

有時候，應用程式會儲存每位使用者偏好的在地化語言。透過在您的可通知模型上實作 `HasLocalePreference` 契約，您可以指示 Laravel 在發送通知時使用此儲存的在地化語言：

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

一旦您實作了該介面，Laravel 在向該模型發送通知和 Mailables 時會自動使用偏好的在地化語言。因此，使用此介面時不需要呼叫 `locale` 方法：

```php
$user->notify(new InvoicePaid($invoice));
```


<a name="testing"></a>
## 測試

您可以使用 `Notification` Facade 的 `fake` 方法來防止通知被發送。通常，發送通知與您實際測試的程式碼無關。在大多數情況下，只需斷言 (assert) Laravel 已被指示發送特定的通知就足夠了。

在呼叫 `Notification` Facade 的 `fake` 方法後，您可以斷言通知已被指示發送給使用者，甚至可以檢查通知接收到的資料：

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

您可以將閉包 (closure) 傳遞給 `assertSentTo` 或 `assertNotSentTo` 方法，以斷言發送的通知是否通過特定的「真實性測試」。如果至少有一個發送的通知通過了該真實性測試，則斷言成功：

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

如果您測試的程式碼會發送 [隨需通知](#on-demand-notifications)，您可以使用 `assertSentOnDemand` 方法來測試該隨需通知是否已發送：

```php
Notification::assertSentOnDemand(OrderShipped::class);
```

透過將閉包作為 `assertSentOnDemand` 方法的第二個引數，您可以判斷隨需通知是否發送到了正確的「路由」地址：

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
#### 通知發送中事件 (Notification Sending Event)

當通知正在發送時，通知系統會發出 `Illuminate\Notifications\Events\NotificationSending` 事件。這包含「可通知 (notifiable)」實體以及通知實例本身。您可以在應用程式中為此事件建立 [事件監聽器](/docs/{{version}}/events)：

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

如果 `NotificationSending` 事件的監聽器在 `handle` 方法中返回 `false`，則該通知將不會被發送：

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
#### 通知已發送事件 (Notification Sent Event)

當通知已發送後，通知系統會發出 `Illuminate\Notifications\Events\NotificationSent` [事件](/docs/{{version}}/events)。這包含「可通知 (notifiable)」實體以及通知實例本身。您可以在應用程式中為此事件建立 [事件監聽器](/docs/{{version}}/events)：

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
## 自訂通道

Laravel 內建了幾個通知通道，但您可能想要編寫自己的驅動程式，以便透過其他通道傳遞通知。Laravel 讓這件事變得簡單。首先，請定義一個包含 `send` 方法的類別。該方法應接收兩個引數：`$notifiable` 與 `$notification`。

在 `send` 方法中，您可以呼叫通知上的方法來取得您的通道可以識別的訊息物件，然後依照您的需求將通知發送至 `$notifiable` 實例：

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

一旦定義好通知通道類別後，您可以在任何通知的 `via` 方法中回傳該類別名稱。在此範例中，通知的 `toVoice` 方法可以回傳任何您選擇用來代表語音訊息的物件。例如，您可以定義自己的 `VoiceMessage` 類別來代表這些訊息：

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