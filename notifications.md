# 通知

- [簡介](#introduction)
- [產生通知](#generating-notifications)
- [發送通知](#sending-notifications)
    - [使用 Notifiable Trait](#using-the-notifiable-trait)
    - [使用 Notification Facade](#using-the-notification-facade)
    - [指定發送頻道](#specifying-delivery-channels)
    - [將通知排入佇列](#queueing-notifications)
    - [隨需通知](#on-demand-notifications)
- [郵件通知](#mail-notifications)
    - [格式化郵件訊息](#formatting-mail-messages)
    - [自訂寄件者](#customizing-the-sender)
    - [自訂收件者](#customizing-the-recipient)
    - [自訂主旨](#customizing-the-subject)
    - [自訂郵件服務](#customizing-the-mailer)
    - [自訂範本](#customizing-the-templates)
    - [附件](#mail-attachments)
    - [新增標籤與中繼資料](#adding-tags-metadata)
    - [自訂 Symfony Message](#customizing-the-symfony-message)
    - [使用 Mailables](#using-mailables)
    - [預覽郵件通知](#previewing-mail-notifications)
- [Markdown 郵件通知](#markdown-mail-notifications)
    - [產生訊息](#generating-the-message)
    - [編寫訊息](#writing-the-message)
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
- [簡訊通知](#sms-notifications)
    - [先決條件](#sms-prerequisites)
    - [格式化簡訊通知](#formatting-sms-notifications)
    - [自訂「發送者」號碼](#customizing-the-from-number)
    - [新增客戶參考](#adding-a-client-reference)
    - [路由簡訊通知](#routing-sms-notifications)
- [Slack 通知](#slack-notifications)
    - [先決條件](#slack-prerequisites)
    - [格式化 Slack 通知](#formatting-slack-notifications)
    - [Slack 互動性](#slack-interactivity)
    - [路由 Slack 通知](#routing-slack-notifications)
    - [通知外部 Slack 工作區](#notifying-external-slack-workspaces)
- [在地化通知](#localizing-notifications)
- [測試](#testing)
- [通知事件](#notification-events)
- [自訂頻道](#custom-channels)

<a name="introduction"></a>
## 簡介

除了支援 [發送電子郵件](/docs/{{version}}/mail) 外，Laravel 還支援透過多種發送頻道傳送通知，包含電子郵件、簡訊 (透過 [Vonage](https://www.vonage.com/communications-apis/)，前身為 Nexmo)，以及 [Slack](https://slack.com)。此外，還有許多由社群建立的 [通知頻道](https://laravel-notification-channels.com/about/#suggesting-a-new-channel)，可以透過數十種不同的頻道發送通知！通知也可以儲存在資料庫中，以便在你的 Web 介面中顯示。

通常，通知應該是簡短且提供資訊的訊息，用於通知使用者應用程式中發生的事件。舉例來說，如果你正在編寫一個帳務應用程式，你可能會透過電子郵件和簡訊頻道向使用者發送「發票已支付」通知。

<a name="generating-notifications"></a>
## 產生通知

在 Laravel 中，每個通知都由一個單獨的類別來表示，通常儲存在 `app/Notifications` 目錄中。如果你的應用程式中沒有這個目錄，請不用擔心，當你執行 `make:notification` Artisan 命令時，它會為你建立：

```shell
php artisan make:notification InvoicePaid
```

這個命令會將一個全新的通知類別放置到你的 `app/Notifications` 目錄中。每個通知類別都包含一個 `via` 方法，以及數量不定的訊息建構方法，例如 `toMail` 或 `toDatabase`，這些方法會將通知轉換為適用於特定頻道的訊息。

<a name="sending-notifications"></a>
## 發送通知

<a name="using-the-notifiable-trait"></a>
### 使用 Notifiable Trait

通知可透過兩種方式發送：使用 `Notifiable` trait 的 `notify` 方法，或是使用 `Notification` [Facade](/docs/{{version}}/facades)。`Notifiable` trait 預設包含在應用程式的 `App\Models\User` 模型中：

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

此 trait 所提供的 `notify` 方法預期接收一個通知實例：

```php
use App\Notifications\InvoicePaid;

$user->notify(new InvoicePaid($invoice));
```

> [!NOTE]
> 請記住，您可以在任何模型上使用 `Notifiable` trait。您不限於只能將其包含在 `User` 模型中。

<a name="using-the-notification-facade"></a>
### 使用 Notification Facade

或者，您可以透過 `Notification` [Facade](/docs/{{version}}/facades) 發送通知。當您需要向多個可通知實體 (例如使用者集合) 發送通知時，這種方法很有用。若要使用 Facade 發送通知，請將所有可通知實體和通知實例傳遞給 `send` 方法：

```php
use Illuminate\Support\Facades\Notification;

Notification::send($users, new InvoicePaid($invoice));
```

您也可以使用 `sendNow` 方法立即發送通知。即使通知實作了 `ShouldQueue` 介面，此方法也會立即發送通知：

```php
Notification::sendNow($developers, new DeploymentCompleted($deployment));
```

<a name="specifying-delivery-channels"></a>
### 指定發送頻道

每個通知類別都有一個 `via` 方法，用於決定通知將透過哪些頻道發送。通知可透過 `mail`、`database`、`broadcast`、`vonage` 和 `slack` 頻道發送。

> [!NOTE]
> 如果您想使用其他發送頻道，例如 Telegram 或 Pusher，請查看社群驅動的 [Laravel Notification Channels 網站](http://laravel-notification-channels.com)。

`via` 方法接收一個 `$notifiable` 實例，該實例將是通知要發送到的類別實例。您可以使用 `$notifiable` 來決定通知應該透過哪些頻道發送：

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
> 在將通知排入佇列之前，你應該配置你的佇列並[啟動一個 worker](/docs/{{version}}/queues#running-the-queue-worker)。

發送通知可能需要時間，特別是當頻道需要進行外部 API 呼叫來遞送通知時。為了加快應用程式的回應時間，你可以將通知排入佇列，方法是在你的類別中加入 `ShouldQueue` 介面和 `Queueable` Trait。所有透過 `make:notification` Artisan 指令產生的通知都已匯入此介面和 Trait，所以你可以立即將它們加入你的通知類別中：

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

一旦 `ShouldQueue` 介面已新增至你的通知，你就可以像往常一樣發送通知。Laravel 將會偵測到類別上的 `ShouldQueue` 介面，並自動將通知的遞送排入佇列：

```php
$user->notify(new InvoicePaid($invoice));
```

將通知排入佇列時，將會為每個收件者與頻道組合建立一個排入佇列的 Job。例如，如果你的通知有三個收件者和兩個頻道，則將會分派六個 Job 到佇列中。

<a name="delaying-notifications"></a>
#### 延遲通知

如果你想延遲通知的遞送，你可以將 `delay` 方法鏈接到你的通知實例上：

```php
$delay = now()->addMinutes(10);

$user->notify((new InvoicePaid($invoice))->delay($delay));
```

你可以傳遞一個陣列給 `delay` 方法來指定特定頻道的延遲時間：

```php
$user->notify((new InvoicePaid($invoice))->delay([
    'mail' => now()->addMinutes(5),
    'sms' => now()->addMinutes(10),
]));
```

此外，你可以在通知類別本身定義一個 `withDelay` 方法。`withDelay` 方法應該回傳一個包含頻道名稱和延遲值的陣列：

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

預設情況下，排入佇列的通知將使用你應用程式的預設佇列連線。如果你想為特定通知指定不同的連線，你可以在通知的建構式中呼叫 `onConnection` 方法：

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

或者，如果你想為通知支援的每個通知頻道指定特定的佇列連線，你可以在通知中定義一個 `viaConnections` 方法。此方法應該回傳一個包含頻道名稱 / 佇列連線名稱配對的陣列：

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
#### 自訂通知頻道佇列

如果你想為通知支援的每個通知頻道指定特定的佇列，你可以在通知中定義一個 `viaQueues` 方法。此方法應該回傳一個包含頻道名稱 / 佇列名稱配對的陣列：

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

<a name="queued-notification-middleware"></a>
#### 排入佇列的通知中介層

排入佇列的通知可以定義中介層，[就像排入佇列的 Job 一樣](/docs/{{version}}/queues#job-middleware)。要開始使用，請在你的通知類別上定義一個 `middleware` 方法。`middleware` 方法將會收到 `$notifiable` 和 `$channel` 變數，這允許你根據通知的目的地自訂回傳的中介層：

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
#### 排入佇列的通知與資料庫交易

當排入佇列的通知在資料庫交易中被分派時，它們可能在資料庫交易提交之前就被佇列處理。當這種情況發生時，你在資料庫交易期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫記錄可能不存在於資料庫中。如果你的通知依賴這些模型，在處理發送排入佇列通知的 Job 時，可能會發生意外錯誤。

如果你的佇列連線的 `after_commit` 配置選項設定為 `false`，你仍然可以指出某個特定的排入佇列通知應在所有開啟的資料庫交易提交後才分派，方法是在發送通知時呼叫 `afterCommit` 方法：

```php
use App\Notifications\InvoicePaid;

$user->notify((new InvoicePaid($invoice))->afterCommit());
```

或者，你可以在通知的建構式中呼叫 `afterCommit` 方法：

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
> 要了解更多關於如何解決這些問題，請查閱關於 [排入佇列的 Job 與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions) 的文件。

<a name="determining-if-the-queued-notification-should-be-sent"></a>
#### 判斷排入佇列的通知是否應發送

當排入佇列的通知被分派到佇列中進行背景處理後，它通常會被一個佇列 worker 接受並發送給預期的收件者。

然而，如果你想在佇列 worker 處理後，最終決定是否應該發送排入佇列的通知，你可以在通知類別上定義一個 `shouldSend` 方法。如果此方法回傳 `false`，則通知將不會發送：

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

有時，您可能需要向未儲存為應用程式「使用者」的人發送通知。透過 `Notification` Facade 的 `route` 方法，您可以在發送通知前指定隨需通知路由資訊：

```php
use Illuminate\Broadcasting\Channel;
use Illuminate\Support\Facades\Notification;

Notification::route('mail', 'taylor@example.com')
    ->route('vonage', '5555555555')
    ->route('slack', '#slack-channel')
    ->route('broadcast', [new Channel('channel-name')])
    ->notify(new InvoicePaid($invoice));
```

如果您希望在發送隨需通知到 `mail` 路由時提供收件者的姓名，您可以提供一個陣列，其中包含電子郵件地址作為鍵，以及姓名作為該陣列第一個元素的值：

```php
Notification::route('mail', [
    'barrett@example.com' => 'Barrett Blair',
])->notify(new InvoicePaid($invoice));
```

使用 `routes` 方法，您可以同時為多個通知頻道提供隨需路由資訊：

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

如果一個通知支援以電子郵件發送，您應該在通知類別上定義一個 `toMail` 方法。這個方法會接收一個 `$notifiable` 實體，並應回傳一個 `Illuminate\Notifications\Messages\MailMessage` 實例。

`MailMessage` 類別包含一些簡單的方法，可協助您建立交易性電子郵件訊息。郵件訊息可以包含文字行以及「行動呼籲」。讓我們看一個 `toMail` 方法的範例：

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
> 請注意，我們在 `toMail` 方法中使用了 `$this->invoice->id`。您可以將通知生成訊息所需的任何資料傳遞給通知的建構子。

在此範例中，我們註冊一個問候語、一行文字、一個行動呼籲，然後是另一行文字。`MailMessage` 物件提供的這些方法使格式化小型交易性電子郵件變得簡單快速。郵件頻道隨後會將訊息組件轉換成美觀、回應式的 HTML 電子郵件範本，並附帶純文字版本。以下是透過 `mail` 頻道產生的電子郵件範例：

![一個範例圖片](https://laravel.com/img/docs/notification-example-2.png)

> [!NOTE]
> 發送郵件通知時，請務必在 `config/app.php` 設定檔中設定 `name` 設定選項。此值將用於郵件通知訊息的頁首和頁尾。

<a name="error-messages"></a>
#### 錯誤訊息

有些通知會告知使用者錯誤，例如發票付款失敗。您可以透過在建立訊息時呼叫 `error` 方法來指示郵件訊息與錯誤有關。當在郵件訊息上使用 `error` 方法時，行動呼籲按鈕將會是紅色而非黑色：

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

除了在通知類別中定義文字「行」之外，您還可以使用 `view` 方法來指定應用於渲染通知電子郵件的自訂範本：

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

您可以透過將視圖名稱作為陣列的第二個元素傳遞給 `view` 方法，來為郵件訊息指定一個純文字視圖：

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

預設情況下，電子郵件的寄件者 / 發送位址定義在 `config/mail.php` 設定檔中。不過，您可以透過使用 `from` 方法來為特定通知指定發送位址：

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

當透過 `mail` 頻道發送通知時，通知系統會自動在您的可通知實體上尋找 `email` 屬性。您可以透過在可通知實體上定義 `routeNotificationForMail` 方法，來自訂用於遞送通知的電子郵件位址：

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

預設情況下，電子郵件的主旨是通知的類別名稱，格式化為「標題大寫格式」。因此，如果您的通知類別名為 `InvoicePaid`，電子郵件的主旨將是 `Invoice Paid`。如果您想為訊息指定不同的主旨，可以在建立訊息時呼叫 `subject` 方法：

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
### 自訂郵件服務

預設情況下，電子郵件通知將使用 `config/mail.php` 設定檔中定義的預設郵件服務來發送。不過，您可以在建立訊息時呼叫 `mailer` 方法，在執行時期指定不同的郵件服務：

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
### 自訂範本

您可以透過發佈通知套件的資源來修改郵件通知使用的 HTML 和純文字範本。執行此指令後，郵件通知範本將位於 `resources/views/vendor/notifications` 目錄中：

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
> `attach` 方法由通知郵件訊息提供，它也接受 [attachable 物件](/docs/{{version}}/mail#attachable-objects)。請查閱完整的 [attachable 物件文件](/docs/{{version}}/mail#attachable-objects) 以了解更多資訊。

為訊息附加檔案時，您也可以透過傳遞一個 `array` 作為 `attach` 方法的第二個引數來指定顯示名稱和/或 MIME 類型：

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

不同於在 mailable 物件中附加檔案，您無法使用 `attachFromStorage` 直接從儲存磁碟附加檔案。您應該改用 `attach` 方法並提供檔案在儲存磁碟上的絕對路徑。此外，您也可以從 `toMail` 方法回傳一個 [mailable](/docs/{{version}}/mail#generating-mailables) 物件：

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

如有必要，您可以使用 `attachMany` 方法為訊息附加多個檔案：

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

`attachData` 方法可用於將原始位元組字串作為附件附加。呼叫 `attachData` 方法時，您應該提供應分配給附件的檔名：

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

某些第三方電子郵件供應商，例如 Mailgun 和 Postmark，支援訊息「標籤 (tags)」和「中繼資料 (metadata)」，這些可用於對應用程式發送的電子郵件進行分組和追蹤。您可以透過 `tag` 和 `metadata` 方法為電子郵件訊息新增標籤與中繼資料：

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

如果您的應用程式使用 Mailgun 驅動，您可以查閱 Mailgun 的文件，了解更多關於 [標籤 (tags)](https://documentation.mailgun.com/docs/mailgun/user-manual/tracking-messages/#tags) 和 [中繼資料 (metadata)](https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/#attaching-metadata-to-messages) 的資訊。同樣地，也可以查閱 Postmark 的文件，了解更多關於其支援的 [標籤 (tags)](https://postmarkapp.com/blog/tags-support-for-smtp) 和 [中繼資料 (metadata)](https://postmarkapp.com/support/article/1125-custom-metadata-faq) 資訊。

如果您的應用程式使用 Amazon SES 來發送電子郵件，您應該使用 `metadata` 方法來為訊息附加 [SES 「標籤 (tags)」](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html)。

<a name="customizing-the-symfony-message"></a>
### 自訂 Symfony Message

`MailMessage` 類別的 `withSymfonyMessage` 方法允許您註冊一個閉包 (closure)，該閉包會在發送訊息之前，與 Symfony Message 實例一起被調用。這讓您有機會在訊息遞送之前對其進行深度自訂：

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

如有需要，您可以從通知的 `toMail` 方法回傳一個完整的 [mailable 物件](/docs/{{version}}/mail)。當回傳 `Mailable` 而非 `MailMessage` 時，您需要使用 mailable 物件的 `to` 方法來指定訊息收件者：

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

如果您正在發送 [隨需通知](#on-demand-notifications)，傳遞給 `toMail` 方法的 `$notifiable` 實例將會是 `Illuminate\Notifications\AnonymousNotifiable` 的實例，它提供了一個 `routeNotificationFor` 方法，可用於擷取隨需通知應發送到的電子郵件位址：

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

在設計郵件通知範本時，能夠像典型的 Blade 範本一樣在瀏覽器中快速預覽渲染後的郵件訊息會很方便。為此，Laravel 允許您直接從路由閉包 (closure) 或控制器中回傳由郵件通知生成的任何 `MailMessage`。當回傳 MailMessage 時，它將在瀏覽器中渲染並顯示，讓您可以快速預覽其設計，而無需將其發送到實際的電子郵件位址：

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

Markdown 郵件通知讓您能夠利用郵件通知的預建範本，同時賦予您更多彈性撰寫較長且客製化的訊息。由於訊息是以 Markdown 撰寫，Laravel 能夠渲染出美觀且響應式的 HTML 範本，同時自動生成純文字版本。

<a name="generating-the-message"></a>
### 產生訊息

若要產生具有對應 Markdown 範本的通知，您可以使用 `make:notification` Artisan 命令的 `--markdown` 選項：

```shell
php artisan make:notification InvoicePaid --markdown=mail.invoice.paid
```

就像所有其他郵件通知一樣，使用 Markdown 範本的通知應在其通知類別中定義 `toMail` 方法。然而，不是使用 `line` 和 `action` 方法來建構通知，而是使用 `markdown` 方法來指定應使用的 Markdown 範本名稱。您希望在範本中可用的資料陣列可以作為該方法的第二個引數傳入：

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
### 編寫訊息

Markdown 郵件通知結合了 Blade 組件和 Markdown 語法，讓您可以輕鬆地建構通知，同時利用 Laravel 預先製作的通知組件：

```blade
<x-mail::message>

# Invoice Paid

您的帳單已支付！

<x-mail::button :url="$url">
檢視帳單
</x-mail::button>

感謝您,<br>
{{ config('app.name') }}
</x-mail::message>

> [!NOTE]
> 撰寫 Markdown 郵件時，請勿使用過多縮排。根據 Markdown 標準，縮排的內容將被 Markdown 解析器渲染為程式碼區塊。


<a name="button-component"></a>
#### 按鈕組件

按鈕組件會渲染一個置中的按鈕連結。此組件接受兩個引數，一個是 `url`，另一個是選用的 `color`。支援的顏色有 `primary`、`green` 和 `red`。您可以在通知中加入任意數量的按鈕組件：

```blade
<x-mail::button :url="$url" color="green">
View Invoice
</x-mail::button>
```


<a name="panel-component"></a>
#### 面板組件

面板組件會將給定的文字區塊渲染在一個面板中，該面板的背景顏色與通知的其餘部分略有不同。這可讓您將注意力吸引到特定的文字區塊：

```blade
<x-mail::panel>
This is the panel content.
</x-mail::panel>
```


<a name="table-component"></a>
#### 表格組件

表格組件可讓您將 Markdown 表格轉換為 HTML 表格。此組件接受 Markdown 表格作為其內容。表格欄位對齊支援使用預設的 Markdown 表格對齊語法：

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

您可以將所有 Markdown 通知組件匯出到您自己的應用程式中進行自訂。若要匯出這些組件，請使用 `vendor:publish` Artisan 命令來發佈 `laravel-mail` 資源標籤：

```shell
php artisan vendor:publish --tag=laravel-mail
```

此命令會將 Markdown 郵件組件發佈到 `resources/views/vendor/mail` 目錄。`mail` 目錄將包含 `html` 和 `text` 目錄，每個目錄都包含每個可用組件的各自表示法。您可以隨意自訂這些組件。


<a name="customizing-the-css"></a>
#### 自訂 CSS

匯出組件後，`resources/views/vendor/mail/html/themes` 目錄將包含一個 `default.css` 檔案。您可以自訂此檔案中的 CSS，您的樣式將會自動內嵌在 Markdown 通知中 HTML 表示法。

如果您想為 Laravel 的 Markdown 組件建立一個全新的主題，您可以將 CSS 檔案放在 `html/themes` 目錄中。在命名並儲存您的 CSS 檔案後，請更新 `mail` 設定檔中的 `theme` 選項，使其與您的新主題名稱相符。

若要自訂個別通知的主題，您可以在建構通知的郵件訊息時呼叫 `theme` 方法。`theme` 方法接受發送通知時應使用的主題名稱：

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

`database` 通知頻道會將通知資訊儲存在資料庫表格中。此表格會包含諸如通知類型以及描述通知的 JSON 資料結構等資訊。

您可以查詢該表格以在應用程式的使用者介面中顯示通知。但是，在此之前，您需要建立一個資料庫表格來存放您的通知。您可以使用 `make:notifications-table` Artisan 指令來產生一個包含正確表格結構的[遷移檔](/docs/{{version}}/migrations)：

```shell
php artisan make:notifications-table

php artisan migrate
```

> [!NOTE]
> 如果您的可通知模型使用的是 [UUID 或 ULID 主鍵](/docs/{{version}}/eloquent#uuid-and-ulid-keys)，則應在通知表格遷移檔中將 `morphs` 方法替換為 [uuidMorphs](/docs/{{version}}/migrations#column-method-uuidMorphs) 或 [ulidMorphs](/docs/{{version}}/migrations#column-method-ulidMorphs)。

<a name="formatting-database-notifications"></a>
### 格式化資料庫通知

如果通知支援儲存在資料庫表格中，您應該在通知類別上定義一個 `toDatabase` 或 `toArray` 方法。此方法將接收一個 `$notifiable` 實體，並應回傳一個純 PHP 陣列。回傳的陣列將被編碼為 JSON，並儲存在 `notifications` 表格的 `data` 欄位中。讓我們來看一個 `toArray` 方法的範例：

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

當通知儲存在應用程式的資料庫中時，`type` 欄位預設會設定為通知的類別名稱，`read_at` 欄位則為 `null`。然而，您可以透過在通知類別中定義 `databaseType` 和 `initialDatabaseReadAtValue` 方法來自訂此行為：

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
#### `toDatabase` 與 `toArray`

`toArray` 方法也由 `broadcast` 頻道使用，以決定要廣播到您由 JavaScript 驅動的前端資料。如果您想為 `database` 和 `broadcast` 頻道提供兩種不同的陣列表示，您應該定義一個 `toDatabase` 方法而不是 `toArray` 方法。

<a name="accessing-the-notifications"></a>
### 存取通知

一旦通知儲存在資料庫中，您需要一種便捷的方式從可通知實體存取它們。`Illuminate\Notifications\Notifiable` trait（預設包含在 Laravel 的 `App\Models\User` 模型中）包含一個 `notifications` [Eloquent 關聯](/docs/{{version}}/eloquent-relationships)，它會回傳該實體的通知。要取得通知，您可以像存取任何其他 Eloquent 關聯一樣存取此方法。預設情況下，通知會依 `created_at` 時間戳記排序，最新的通知會排在集合的開頭：

```php
$user = App\Models\User::find(1);

foreach ($user->notifications as $notification) {
    echo $notification->type;
}
```

如果您只想取得「未讀」通知，您可以使用 `unreadNotifications` 關聯。同樣地，這些通知也會依 `created_at` 時間戳記排序，最新的通知會排在集合的開頭：

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
> 若要從 JavaScript 用戶端存取您的通知，您應該為您的應用程式定義一個通知控制器，該控制器會回傳給可通知實體 (例如目前使用者) 的通知。然後，您可以從 JavaScript 用戶端向該控制器的 URL 發出 HTTP 請求。

<a name="marking-notifications-as-read"></a>
### 將通知標記為已讀

通常，當使用者查看通知時，您會希望將其標記為「已讀」。`Illuminate\Notifications\Notifiable` trait 提供了一個 `markAsRead` 方法，該方法會更新通知資料庫記錄上的 `read_at` 欄位：

```php
$user = App\Models\User::find(1);

foreach ($user->unreadNotifications as $notification) {
    $notification->markAsRead();
}
```

然而，您也可以直接在通知集合上使用 `markAsRead` 方法，而無需循環遍歷每個通知：

```php
$user->unreadNotifications->markAsRead();
```

您還可以使用批次更新查詢來將所有通知標記為已讀，而無需從資料庫中取得它們：

```php
$user = App\Models\User::find(1);

$user->unreadNotifications()->update(['read_at' => now()]);
```

您可以 `delete` 通知以將它們從表格中完全刪除：

```php
$user->notifications()->delete();
```

<a name="broadcast-notifications"></a>
## 廣播通知


<a name="broadcast-prerequisites"></a>
### 先決條件

在廣播通知之前，您應該配置並熟悉 Laravel 的 [event broadcasting](/docs/{{version}}/broadcasting) 服務。事件廣播提供了一種從您由 JavaScript 驅動的前端對伺服器端 Laravel 事件做出反應的方式。


<a name="formatting-broadcast-notifications"></a>
### 格式化廣播通知

`broadcast` 頻道使用 Laravel 的 [event broadcasting](/docs/{{version}}/broadcasting) 服務廣播通知，讓您由 JavaScript 驅動的前端能夠即時捕捉通知。如果通知支援廣播，您可以在通知類別上定義一個 `toBroadcast` 方法。此方法將接收一個 `$notifiable` 實體，並應回傳一個 `BroadcastMessage` 實例。如果 `toBroadcast` 方法不存在，將使用 `toArray` 方法來收集應廣播的資料。回傳的資料將被編碼為 JSON 並廣播到您由 JavaScript 驅動的前端。讓我們看看一個 `toBroadcast` 方法的範例：

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

所有廣播通知都已排入佇列等待廣播。如果您想配置用於佇列廣播操作的佇列連線或佇列名稱，您可以使用 `BroadcastMessage` 的 `onConnection` 和 `onQueue` 方法：

```php
return (new BroadcastMessage($data))
    ->onConnection('sqs')
    ->onQueue('broadcasts');
```


<a name="customizing-the-notification-type"></a>
#### 自訂通知類型

除了您指定的資料外，所有廣播通知還帶有一個 `type` 欄位，其中包含通知的完整類別名稱。如果您想自訂通知 `type`，您可以在通知類別上定義一個 `broadcastType` 方法：

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

通知將會廣播到一個使用 `{notifiable}.{id}` 慣例格式化的私有頻道。因此，如果您正在向一個 ID 為 `1` 的 `App\Models\User` 實例發送通知，該通知將會廣播到 `App.Models.User.1` 私有頻道。當使用 [Laravel Echo](/docs/{{version}}/broadcasting#client-side-installation) 時，您可以使用 `notification` 方法輕鬆監聽頻道上的通知：

```js
Echo.private('App.Models.User.' + userId)
    .notification((notification) => {
        console.log(notification.type);
    });
```


<a name="using-react-or-vue"></a>
#### 使用 React 或 Vue

Laravel Echo 包含了 React 和 Vue 的 Hooks，使監聽通知變得輕而易舉。要開始使用，請調用 `useEchoNotification` Hook，它用於監聽通知。當消耗組件被卸載時，`useEchoNotification` Hook 將自動離開頻道：

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

默認情況下，該 Hook 會監聽所有通知。要指定您想監聽的通知類型，您可以向 `useEchoNotification` 提供一個字串或類型陣列：

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

您也可以指定通知負載資料的形狀，提供更強的類型安全和編輯便利性：

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
#### 自訂通知頻道

如果您想自訂實體廣播通知的頻道，您可以在可通知實體上定義一個 `receivesBroadcastNotificationsOn` 方法：

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
## 簡訊通知

<a name="sms-prerequisites"></a>
### 先決條件

Laravel 中的簡訊通知是由 [Vonage](https://www.vonage.com/) (前身為 Nexmo) 提供技術支援。在使用 Vonage 發送通知之前，你需要安裝 `laravel/vonage-notification-channel` 和 `guzzlehttp/guzzle` 套件：

```shell
composer require laravel/vonage-notification-channel guzzlehttp/guzzle
```

該套件包含一個[設定檔](https://github.com/laravel/vonage-notification-channel/blob/3.x/config/vonage.php)。但是，你不必將此設定檔匯出到你的應用程式中。你只需使用 `VONAGE_KEY` 和 `VONAGE_SECRET` 環境變數來定義你的 Vonage 公開金鑰和密鑰。

定義金鑰後，你應設定一個 `VONAGE_SMS_FROM` 環境變數，此變數定義簡訊預設的發送電話號碼。你可以在 Vonage 控制面板中產生此電話號碼：

```ini
VONAGE_SMS_FROM=15556666666
```

<a name="formatting-sms-notifications"></a>
### 格式化簡訊通知

如果通知支援作為簡訊發送，你應在通知類別上定義一個 `toVonage` 方法。此方法會接收一個 `$notifiable` 實體，並應回傳一個 `Illuminate\Notifications\Messages\VonageMessage` 實例：

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

如果你的簡訊訊息將包含 Unicode 字元，你應在建構 `VonageMessage` 實例時呼叫 `unicode` 方法：

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
### 自訂「發送者」號碼

如果你想從一個與 `VONAGE_SMS_FROM` 環境變數中指定的電話號碼不同的電話號碼發送通知，你可以在 `VonageMessage` 實例上呼叫 `from` 方法：

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

如果你想追蹤每個使用者、團隊或客戶的費用，你可以為通知新增一個「客戶參考」。Vonage 將允許你使用此客戶參考來產生報告，以便你更好地了解特定客戶的簡訊使用情況。客戶參考可以是任何長度不超過 40 個字元的字串：

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
### 路由簡訊通知

要將 Vonage 通知路由到正確的電話號碼，請在你的可通知實體上定義 `routeNotificationForVonage` 方法：

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

如果您只需要將通知發送到 App 所在的 Slack 工作區，您應該確保您的 App 具有 `chat:write`、`chat:write.public` 和 `chat:write.customize` 範圍。這些範圍可以在 Slack 中的「OAuth 與權限」App 管理標籤中新增。

接下來，將 App 的「Bot 使用者 OAuth 權杖」複製並放置在您的應用程式 `services.php` 設定檔中的 `slack` 設定陣列內。此權杖可以在 Slack 中的「OAuth 與權限」標籤中找到：

```php
'slack' => [
    'notifications' => [
        'bot_user_oauth_token' => env('SLACK_BOT_USER_OAUTH_TOKEN'),
        'channel' => env('SLACK_BOT_USER_DEFAULT_CHANNEL'),
    ],
],
```

<a name="slack-app-distribution"></a>
#### App 發佈

如果您的應用程式將向由應用程式使用者擁有的外部 Slack 工作區發送通知，您需要透過 Slack 「發佈」您的 App。App 發佈可以在您的 App 在 Slack 中的「管理發佈」標籤中進行管理。一旦您的 App 已發佈，您可以使用 [Socialite](/docs/{{version}}/socialite) 為您的應用程式使用者 [取得 Slack Bot 權杖](/docs/{{version}}/socialite#slack-bot-scopes)。

<a name="formatting-slack-notifications"></a>
### 格式化 Slack 通知

如果通知支援作為 Slack 訊息發送，您應該在通知類別中定義一個 `toSlack` 方法。此方法將接收一個 `$notifiable` 實體，並應返回一個 `Illuminate\Notifications\Slack\SlackMessage` 實例。您可以使用 [Slack 的 Block Kit API](https://api.slack.com/block-kit) 來建構豐富的通知。以下範例可以在 [Slack 的 Block Kit Builder](https://app.slack.com/block-kit-builder/T01KWS6K23Z#%7B%22blocks%22:%5B%7B%22type%22:%22header%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22Invoice%20Paid%22%7D%7D,%7B%22type%22:%22context%22,%22elements%22:%5B%7B%22type%22:%22plain_text%22,%22text%22:%22Customer%20%231234%22%7D%5D%7D,%7B%22type%22:%22section%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22An%20invoice%20has%20been%20paid.%22%7D,%22fields%22:%5B%7B%22type%22:%22mrkdwn%22,%22text%22:%22*Invoice%20No:*%5Cn1000%22%7D,%7B%22type%22:%22mrkdwn%22,%22text%22:%22*Invoice%20Recipient:*%5Cntaylor@laravel.com%22%7D%5D%7D,%7B%22type%22:%22divider%22%7D,%7B%22type%22:%22section%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22Congratulations!%22%7D%7D%5D%7D) 中預覽：

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

除了使用流暢的訊息建構器方法來建構您的 Block Kit 訊息之外，您還可以將 Slack 的 Block Kit Builder 生成的原始 JSON 負載提供給 `usingBlockKitTemplate` 方法：

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

Slack 的 Block Kit 通知系統提供了強大的功能來 [處理使用者互動](https://api.slack.com/interactivity/handling)。要利用這些功能，您的 Slack App 應該啟用「互動性」並設定一個「請求 URL」指向您的應用程式提供的 URL。這些設定可以在 Slack 中的「互動性與捷徑」App 管理標籤中進行管理。

在以下使用 `actionsBlock` 方法的範例中，Slack 將向您的「請求 URL」發送一個 `POST` 請求，其中包含點擊按鈕的 Slack 使用者、點擊按鈕的 ID 等負載。您的應用程式隨後可以根據負載決定要採取的動作。您還應該 [驗證請求](https://api.slack.com/authentication/verifying-requests-from-slack) 是由 Slack 發出的：

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
#### 確認彈出視窗

如果您希望使用者在執行操作之前必須確認，您可以在定義按鈕時調用 `confirm` 方法。`confirm` 方法接受一條訊息和一個閉包，該閉包接收一個 `ConfirmObject` 實例：

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

如果您想快速檢查您正在建構的區塊，您可以在 `SlackMessage` 實例上調用 `dd` 方法。`dd` 方法將生成並轉儲一個指向 Slack 的 [Block Kit Builder](https://app.slack.com/block-kit-builder/) 的 URL，該 URL 會在您的瀏覽器中顯示負載和通知的預覽。您可以向 `dd` 方法傳遞 `true` 以轉儲原始負載：

```php
return (new SlackMessage)
    ->text('One of your invoices has been paid!')
    ->headerBlock('Invoice Paid')
    ->dd();
```

<a name="routing-slack-notifications"></a>
### 路由 Slack 通知

要將 Slack 通知導向適當的 Slack 團隊和頻道，請在您的可通知模型上定義一個 `routeNotificationForSlack` 方法。此方法可以返回三個值之一：

-   `null` - 將路由推遲到通知本身中配置的頻道。您可以在建構 `SlackMessage` 時使用 `to` 方法在通知內配置頻道。
-   一個指定要發送通知的 Slack 頻道字串，例如 `#support-channel`。
-   一個 `SlackRoute` 實例，它允許您指定 OAuth 權杖和頻道名稱，例如 `SlackRoute::make($this->slack_channel, $this->slack_token)`。此方法應用於向外部工作區發送通知。

例如，從 `routeNotificationForSlack` 方法返回 `#support-channel` 將把通知發送到您的應用程式 `services.php` 設定檔中與 Bot 使用者 OAuth 權杖相關聯的工作區中的 `#support-channel` 頻道：

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
> 在向外部 Slack 工作區發送通知之前，您的 Slack App 必須已 [發佈](#slack-app-distribution)。

當然，您通常會希望向應用程式使用者擁有的 Slack 工作區發送通知。為此，您首先需要為使用者取得一個 Slack OAuth 權杖。幸運的是，[Laravel Socialite](/docs/{{version}}/socialite) 包含一個 Slack 驅動，可讓您輕鬆地使用 Slack 驗證應用程式使用者並 [取得 bot 權杖](/docs/{{version}}/socialite#slack-bot-scopes)。

一旦您取得了 bot 權杖並將其儲存在應用程式的資料庫中，您就可以使用 `SlackRoute::make` 方法將通知路由到使用者的工作區。此外，您的應用程式可能需要提供機會讓使用者指定應將通知發送到哪個頻道：

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

Laravel 允許您以不同於 HTTP 請求當前語系的方式發送通知，即使通知已排入佇列，它也會記住該語系。

為了實現這一點，`Illuminate\Notifications\Notification` 類別提供了 `locale` 方法來設定期望的語言。在評估通知時，應用程式會切換到此語系，並在評估完成後恢復為先前的語系：

```php
$user->notify((new InvoicePaid($invoice))->locale('es'));
```

也可透過 `Notification` Facade 來實現多個可通知實體的在地化：

```php
Notification::locale('es')->send(
    $users, new InvoicePaid($invoice)
);
```

<a name="user-preferred-locales"></a>
#### 使用者偏好的語系

有時，應用程式會儲存每個使用者偏好的語系。透過在您的可通知模型上實作 `HasLocalePreference` 契約，您可以指示 Laravel 在發送通知時使用此儲存的語系：

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

一旦您實作了此介面，Laravel 在向模型發送通知和 Mailables 時，將自動使用偏好的語系。因此，在使用此介面時，無需呼叫 `locale` 方法：

```php
$user->notify(new InvoicePaid($invoice));
```

<a name="testing"></a>
## 測試

您可以使用 `Notification` Facade 的 `fake` 方法來阻止通知被發送。通常，發送通知與您實際測試的程式碼無關。最可能的是，只需斷言 Laravel 已被指示發送特定通知就足夠了。

呼叫 `Notification` Facade 的 `fake` 方法後，您可以斷言通知已被指示發送給使用者，甚至檢查通知收到的資料：

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

您可以傳遞一個閉包給 `assertSentTo` 或 `assertNotSentTo` 方法，以斷言已發送的通知通過了給定的「真實性測試」。如果至少有一個通知通過了給定的真實性測試，則該斷言將會成功：

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

如果您測試的程式碼發送 [隨需通知](#on-demand-notifications)，您可以透過 `assertSentOnDemand` 方法測試隨需通知是否已發送：

```php
Notification::assertSentOnDemand(OrderShipped::class);
```

透過傳遞閉包作為 `assertSentOnDemand` 方法的第二個參數，您可以判斷隨需通知是否已發送到正確的「路由」位址：

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
#### 通知發送事件

當通知正在發送時，通知系統會分發 `Illuminate\Notifications\Events\NotificationSending` 事件。此事件包含「可通知的」實體和通知實例本身。您可以在應用程式中為此事件建立 [事件監聽器](/docs/{{version}}/events)：

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

在事件監聽器中，您可以存取事件上的 `notifiable`、`notification` 和 `channel` 屬性，以了解更多關於通知接收者或通知本身的資訊：

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

當通知已發送時，通知系統會分發 `Illuminate\Notifications\Events\NotificationSent` [事件](/docs/{{version}}/events)。此事件包含「可通知的」實體和通知實例本身。您可以在應用程式中為此事件建立 [事件監聽器](/docs/{{version}}/events)：

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

在事件監聽器中，您可以存取事件上的 `notifiable`、`notification`、`channel` 和 `response` 屬性，以了解更多關於通知接收者或通知本身的資訊：

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
## 自訂頻道

Laravel 附帶了少量的通知頻道，但您可能希望編寫自己的驅動程式，透過其他頻道來發送通知。Laravel 讓這變得簡單。首先，定義一個包含 `send` 方法的類別。該方法應接收兩個引數：一個 `$notifiable` 和一個 `$notification`。

在 `send` 方法中，您可以呼叫通知上的方法，以取得頻道理解的訊息物件，然後將通知以您希望的任何方式發送給 `$notifiable` 實例：

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

一旦您的通知頻道類別已定義，您可以從通知的 `via` 方法中回傳該類別名稱。在此範例中，通知的 `toVoice` 方法可以回傳您選擇的任何物件來表示語音訊息。例如，您可以定義自己的 `VoiceMessage` 類別來表示這些訊息：

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